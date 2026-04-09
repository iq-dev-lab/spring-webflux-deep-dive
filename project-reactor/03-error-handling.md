# 에러 처리 — Reactive에서 try-catch가 없는 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Reactive 파이프라인에서 `try-catch`를 쓸 수 없는 이유는 무엇인가?
- `onErrorReturn`, `onErrorResume`, `onErrorMap`, `doOnError`는 어떤 상황에서 각각 사용하는가?
- `retry`와 `retryWhen`(지수 백오프)의 내부 동작은 어떻게 다른가?
- `onError` 신호가 파이프라인을 타고 전파되는 원리는 무엇인가?
- 에러를 격리하여 스트림이 계속 진행되게 하려면 어떻게 해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

동기 코드에서는 `try-catch`로 예외를 잡고 처리합니다. 그런데 WebFlux 파이프라인에서 `try-catch`를 쓰면 비동기 실행 시점에 발생하는 예외를 잡을 수 없습니다. Reactive에서는 예외가 `onError` 신호로 변환되어 파이프라인을 타고 흘러내려갑니다.

에러 처리를 제대로 이해하지 못하면 예외가 로그에만 남고 HTTP 응답이 없는 상태, 재시도 로직이 외부 API를 폭격하는 상태, 일부 항목 에러로 전체 스트림이 종료되는 상태를 만나게 됩니다.

---

## 😱 흔한 실수 (Before — 에러 처리를 잘못 이해할 때)

```
실수 1: 파이프라인 밖에서 try-catch 사용

  @GetMapping("/user/{id}")
  public Mono<User> getUser(@PathVariable Long id) {
      try {
          return userRepository.findById(id);  // 비동기 실행!
          // findById가 실제 실행되는 건 subscribe() 시점
          // 그 시점에 발생한 예외는 여기서 잡히지 않음
      } catch (Exception e) {
          return Mono.just(User.empty());  // 절대 실행 안 됨
      }
  }

실수 2: map 안의 예외를 놓침

  flux.map(item -> {
      if (item.isInvalid()) throw new IllegalArgumentException();
      return item.process();
  })
  // → map은 예외를 onError로 변환하지만
  // 이후에 onError 핸들러가 없으면 스트림 종료 + 에러 로그만

실수 3: retry를 무한정 또는 너무 자주

  webClient.get().uri("/flaky-api").retrieve()
      .bodyToMono(String.class)
      .retry();  // 영구 재시도 → 외부 API 폭격 위험
```

---

## ✨ 올바른 접근 (After — 에러를 신호로 처리)

```
에러 처리 연산자 선택 기준:

로깅만 필요               → doOnError (스트림 계속)
기본값으로 대체            → onErrorReturn
다른 Publisher로 대체     → onErrorResume
에러 타입 변환             → onErrorMap
재시도 (고정 횟수)         → retry(n)
재시도 (지수 백오프)       → retryWhen(Retry.backoff(...))
항목별 에러 무시           → flatMap 내부에서 onErrorResume

핵심 원칙:
  onError = 스트림 종료 신호
  → onError 처리 후 다른 값/Publisher 반환 가능
  → 처리 없으면 Subscriber의 onError 콜백 호출
```

---

## 🔬 내부 동작 원리

### 1. 에러 신호 전파 원리

```
파이프라인에서 에러 발생 흐름:

Flux.range(1, 5)
    .map(i -> {
        if (i == 3) throw new RuntimeException("에러!");
        return i * 10;
    })
    .filter(i -> i > 0)
    .subscribe(
        item  -> log.info("수신: {}", item),
        error -> log.error("에러: {}", error.getMessage()),
        ()    -> log.info("완료")
    );

실행 흐름:
  onNext(1) → map(10) → filter → subscribe.onNext(10)
  onNext(2) → map(20) → filter → subscribe.onNext(20)
  onNext(3) → map 예외 발생!
              → map이 upstream.cancel() 호출 (range 중단)
              → filter.onError(RuntimeException)
              → subscribe.onError("에러!")
  // onNext(4), onNext(5) 없음 — 스트림 종료

핵심:
  map 내부 예외 → onError 신호로 자동 변환
  onError는 체인 끝까지 전파
  중간에 onError 처리 연산자 없으면 최종 Subscriber까지 전달
  onError 이후 onNext 없음 (스펙 보장)
```

### 2. 에러 처리 연산자 4종 상세

```
① onErrorReturn(fallback) — 기본값으로 대체 후 정상 완료

  Mono<User> user = userRepository.findById(id)
      .onErrorReturn(User.anonymous());
  // DB 에러 시 → anonymous User 반환 → onComplete
  // 에러가 없었던 것처럼 정상 완료

  타입별 조건 추가:
  .onErrorReturn(DataAccessException.class, User.anonymous())
  // DataAccessException 계열만 처리, 그 외는 전파

────────────────────────────────────────

② onErrorResume(fallbackFn) — 다른 Publisher로 대체

  Mono<User> user = userRepository.findById(id)
      .onErrorResume(e -> {
          log.warn("DB 조회 실패, 캐시에서 조회: {}", e.getMessage());
          return userCacheRepository.findById(id);  // fallback Publisher
      });
  // DB 에러 시 → 캐시에서 재시도
  // 캐시도 실패 시 그 에러가 전파

  onErrorReturn과 차이:
    onErrorReturn: 고정 값 하나 반환
    onErrorResume: 다른 Publisher 실행 가능 (추가 I/O, 비동기 처리)

────────────────────────────────────────

③ onErrorMap(mapFn) — 에러 타입 변환

  Mono<User> user = userRepository.findById(id)
      .onErrorMap(DataAccessException.class,
          e -> new ServiceException("데이터 조회 실패", e));
  // DataAccessException → ServiceException으로 래핑
  // 스트림은 여전히 에러 상태 (종료)
  // 하위 계층 예외를 상위 계층 예외로 변환할 때 사용

────────────────────────────────────────

④ doOnError(handler) — 부수 효과(로깅), 스트림 계속

  Mono<User> user = userRepository.findById(id)
      .doOnError(e -> log.error("사용자 조회 실패: id={}", id, e));
  // 에러 로깅 후 onError 신호 계속 전파
  // 에러를 처리하지 않음 — 로깅만 담당
  // doOnError는 스트림을 변경하지 않음
```

### 3. retry와 retryWhen — 재시도 전략

```
retry(n) — 고정 횟수 즉시 재시도:

  webClient.get().uri("/api").retrieve()
      .bodyToMono(String.class)
      .retry(3);  // 실패 시 최대 3번 즉시 재시도

  마블 다이어그램:
    원본: ─ error ─|
    retry(2):
    시도 1: ─ error ─|
    시도 2: ─ error ─|
    시도 3: ─ success ─|
    결과: ─────────── success ─|

  문제: 즉시 재시도 → 외부 API가 일시적 과부하일 때 더 악화

────────────────────────────────────────

retryWhen(Retry.backoff) — 지수 백오프:

  webClient.get().uri("/api").retrieve()
      .bodyToMono(String.class)
      .retryWhen(Retry.backoff(3, Duration.ofSeconds(1))
          .maxBackoff(Duration.ofSeconds(10))
          .jitter(0.5)
          .filter(e -> e instanceof WebClientResponseException)
      );
  // 최초 실패: 1초 대기
  // 2번째 실패: 2초 대기
  // 3번째 실패: 4초 대기 (최대 10초)
  // jitter: ±50% 랜덤 — 동시 재시도 충돌 방지
  // filter: 특정 예외만 재시도 (4xx는 재시도 불필요)

지수 백오프 계산:
  대기 시간 = min(초기값 × 2^시도횟수, 최대값) × (1 ± jitter)
  1초, 2초, 4초, 8초, 10초(최대), ...

retryWhen 내부 동작:
  1. onError 신호 수신
  2. RetrySpec이 재시도 여부와 대기 시간 결정
  3. 대기 후 전체 파이프라인 처음부터 재구독
  4. n번 실패 후 RetryExhaustedException으로 종료
```

### 4. 스트림 계속 진행 — 항목별 에러 격리

```
Flux에서 일부 항목 에러가 전체 스트림 종료하는 문제:

// 문제 상황: 하나라도 실패하면 전체 중단
orders.flatMap(order -> paymentClient.charge(order))
    .subscribe(...);
// order-3 결제 실패 → 전체 Flux 종료

// 해결: 내부에서 에러 격리
orders.flatMap(order ->
    paymentClient.charge(order)
        .onErrorResume(e -> {
            log.warn("결제 실패: {}", order.getId());
            return Mono.just(PaymentResult.failed(order.getId()));
        })
)
// 각 주문의 에러가 독립적으로 처리됨 → Flux 계속 진행

// onErrorContinue (스트림 레벨, 사용 주의)
orders.flatMap(order -> paymentClient.charge(order))
    .onErrorContinue((err, item) ->
        log.error("처리 실패, 건너뜀: {}", item)
    );
// onErrorContinue는 동작이 직관적이지 않아 공식 문서에서도 주의 권장
// flatMap 내부의 onErrorResume이 더 명확하고 안전
```

---

## 💻 실전 코드

### 실험 1: 에러 처리 연산자 체인

```java
@GetMapping("/user/{id}")
public Mono<UserResponse> getUser(@PathVariable Long id) {
    return userRepository.findById(id)
        .doOnError(e -> log.error("DB 조회 실패: id={}", id, e))  // 로깅
        .onErrorResume(DataAccessException.class,                   // DB 에러 시
            e -> userCacheRepository.findById(id))                  // 캐시 fallback
        .map(UserResponse::from)
        .onErrorReturn(UserNotFoundException.class,                 // 없으면
            UserResponse.notFound())                                // 기본 응답
        .onErrorMap(e -> new ServiceException("사용자 조회 실패", e)); // 그 외 에러 변환
}
```

### 실험 2: 지수 백오프 재시도

```java
// 외부 결제 API 호출 — 일시적 장애 시 재시도
public Mono<PaymentResult> chargeWithRetry(Order order) {
    return webClient.post()
        .uri("/payment/charge")
        .bodyValue(order)
        .retrieve()
        .onStatus(
            status -> status.is5xxServerError(),
            response -> Mono.error(new PaymentServiceException("결제 서버 오류"))
        )
        .bodyToMono(PaymentResult.class)
        .retryWhen(Retry.backoff(3, Duration.ofMillis(500))
            .maxBackoff(Duration.ofSeconds(5))
            .jitter(0.3)
            .filter(e -> e instanceof PaymentServiceException)  // 5xx만 재시도
            .doBeforeRetry(signal ->
                log.warn("결제 재시도 #{}: {}",
                    signal.totalRetries() + 1,
                    signal.failure().getMessage())
            )
        )
        .onErrorMap(RetryExhaustedException.class,
            e -> new PaymentFailedException("재시도 초과", e));
}
```

### 실험 3: StepVerifier로 에러 시나리오 테스트

```java
@Test
void testErrorHandling() {
    Mono<String> mono = Mono.<String>error(new RuntimeException("DB 오류"))
        .onErrorReturn("기본값");

    StepVerifier.create(mono)
        .expectNext("기본값")
        .expectComplete()
        .verify();
}

@Test
void testRetry() {
    AtomicInteger attempts = new AtomicInteger(0);

    Mono<String> flaky = Mono.fromCallable(() -> {
        if (attempts.incrementAndGet() < 3) {
            throw new RuntimeException("일시적 실패");
        }
        return "성공";
    }).retry(2);  // 최대 2번 재시도 (총 3번 시도)

    StepVerifier.create(flaky)
        .expectNext("성공")
        .expectComplete()
        .verify();

    assertThat(attempts.get()).isEqualTo(3);
}
```

---

## 📊 성능 비교

```
재시도 전략별 외부 API 부하:

전략               | 1번 실패 후  | 외부 API 요청 패턴      | 적합한 상황
──────────────────┼────────────┼───────────────────────┼──────────────
retry(3)          | 즉시 3번    | t, t, t, t            | 일시적 네트워크 문제
retry + delay     | 일정 간격   | t, t+1s, t+2s, t+3s   | 서버 재시작 대기
retryWhen.backoff | 지수 증가   | t, t+1s, t+3s, t+7s   | 외부 API 과부하
retryWhen+jitter  | 랜덤 분산   | t, t+0.7s, t+2.3s ... | 다수 클라이언트 동시 재시도

에러 처리 연산자 오버헤드:
  doOnError: 에러 발생 시에만 실행 → 정상 경로 오버헤드 없음
  onErrorReturn: 단순 값 반환 → 최소 오버헤드
  onErrorResume: 새 Publisher 구독 → 추가 I/O 비용 가능
```

---

## ⚖️ 트레이드오프

```
에러 처리 전략 트레이드오프:

onErrorReturn:
  장점: 단순, 안전 (에러 후 정상 완료)
  단점: 에러 원인을 숨길 수 있음 (로깅 필수)

onErrorResume:
  장점: 유연한 fallback (캐시, 기본값, 재시도)
  단점: fallback도 실패할 수 있음 → 에러 중첩

retry:
  장점: 일시적 장애 자동 복구
  단점: 즉시 재시도는 장애 서버에 추가 부하
  → 항상 retryWhen(backoff) 권장

onErrorContinue:
  장점: 개별 에러 무시하고 스트림 계속
  단점: 내부 동작이 직관적이지 않음, 일부 연산자와 비호환
  → flatMap 내부 onErrorResume 권장
```

---

## 📌 핵심 정리

```
Reactive 에러 처리 핵심:

왜 try-catch 불가:
  비동기 파이프라인에서 예외는 subscribe() 시점에 발생
  → 선언 시점의 try-catch로는 잡을 수 없음
  → 예외 → onError 신호로 변환 → 파이프라인 전파

연산자 선택:
  doOnError:      로깅만 (스트림 계속 에러 상태)
  onErrorReturn:  고정 값으로 대체 후 완료
  onErrorResume:  다른 Publisher로 대체
  onErrorMap:     에러 타입 변환 (여전히 에러 상태)

재시도:
  retry(n):       즉시 재시도 n회 (일시적 문제에만)
  retryWhen(backoff): 지수 백오프 (외부 API 표준 패턴)
  jitter 필수:    다수 클라이언트 동시 재시도 충돌 방지

항목별 에러 격리:
  flatMap 내부에서 onErrorResume → 개별 에러 처리
  전체 Flux 에러 전파 방지
```

---

## 🤔 생각해볼 문제

**Q1.** `retry(3)`와 `retryWhen(Retry.max(3))`은 동일한가요?

<details>
<summary>해설 보기</summary>

기능적으로 유사하지만 `retryWhen(Retry.max(3))`이 더 유연합니다.

`retry(3)`은 3번 즉시 재시도합니다. `retryWhen(Retry.max(3))`은 동일한 재시도 횟수이지만, `retryWhen`의 인자는 `Retry` 객체로 추가 설정이 가능합니다.

실무에서는 `retry(n)` 대신 항상 백오프를 포함한 `retryWhen`을 권장합니다:

```java
// 권장 패턴
.retryWhen(Retry.backoff(3, Duration.ofMillis(100))
    .filter(e -> !(e instanceof IllegalArgumentException))  // 4xx는 재시도 안 함
    .onRetryExhaustedThrow((spec, signal) ->
        new ServiceUnavailableException("재시도 초과", signal.failure())
    )
)
```

기본 `RetryExhaustedException` 대신 의미 있는 예외로 교체하면 에러 로그 분석이 쉬워집니다.

</details>

---

**Q2.** `onErrorReturn`이 적용된 Mono가 구독될 때, 에러가 없는 정상 경우에도 성능 오버헤드가 있나요?

<details>
<summary>해설 보기</summary>

거의 없습니다. `onErrorReturn`은 내부적으로 `onError` 신호가 도착했을 때만 fallback 값을 방출하는 Subscriber를 추가합니다. 정상 경로에서는 단순히 `onNext`와 `onComplete`를 하위로 전달하므로 오버헤드가 무시 가능한 수준입니다.

그러나 연산자 체인의 깊이 자체는 각 단계마다 Subscriber 래핑을 추가합니다. 수십 개의 연산자가 체인되면 메모리 오버헤드가 쌓일 수 있습니다. 실무에서는 과도한 연산자 중첩 없이 논리적으로 필요한 에러 처리만 추가하는 것이 좋습니다.

`doOnError`도 마찬가지로 에러 발생 시에만 핸들러가 실행되므로 정상 경로의 오버헤드는 없습니다.

</details>

---

**Q3.** `retryWhen`에서 재시도 중 새로운 에러가 발생하면 어떻게 되나요?

<details>
<summary>해설 보기</summary>

`retryWhen`에서 재시도 중 새로운 에러가 발생하면, 해당 에러도 재시도 가능 횟수에 포함됩니다. 재시도 횟수를 소진하면 `RetryExhaustedException`으로 감싸져 최종 `onError`로 전파됩니다.

재시도 중 **다른 종류의 에러**가 발생하면 `filter` 조건에 따라 처리가 달라집니다:

```java
.retryWhen(Retry.backoff(3, Duration.ofSeconds(1))
    .filter(e -> e instanceof TransientException)
    // TransientException만 재시도
    // 재시도 중 PermanentException 발생 → filter 불통과 → 즉시 onError
)
```

`filter`를 사용하면 재시도할 에러와 즉시 실패할 에러를 구분할 수 있습니다. 이것이 `retry(n)` 대신 `retryWhen`을 사용하는 핵심 이유 중 하나입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Reactor 연산자 완전 분해](./02-reactor-operators.md)** | **[홈으로 🏠](../README.md)** | **[다음: 스케줄러 ➡️](./04-scheduler-thread-switching.md)**

</div>
