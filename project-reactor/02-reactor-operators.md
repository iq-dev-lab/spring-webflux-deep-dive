# Reactor 연산자 완전 분해 — map vs flatMap vs concatMap

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `map`과 `flatMap`은 어떤 상황에서 각각 사용하는가?
- `flatMap`은 왜 순서를 보장하지 않고, `concatMap`은 왜 느린가?
- `switchMap`은 어떤 시나리오에서 적합한가?
- `zip`, `merge`, `concat`의 차이는 무엇인가?
- `flatMap`의 `concurrency` 파라미터는 어떻게 동시 처리 수를 제어하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

WebFlux 코드에서 가장 자주 쓰는 연산자는 `map`과 `flatMap`입니다. 그런데 이 두 연산자를 잘못 선택하면 정렬이 깨지거나, 성능이 예상보다 훨씬 나쁘거나, 외부 API를 필요 이상으로 많이 호출하게 됩니다. `concatMap`과 `flatMap`의 차이를 모르면 "왜 순서가 뒤바뀌었지?"라는 버그를 디버깅하는 데 한참을 쓰게 됩니다.

---

## 😱 흔한 실수 (Before — 연산자를 잘못 선택할 때)

```
실수 1: 비동기 변환에 map 사용

  Flux<Order> orders = orderRepository.findAll();
  Flux<PaymentStatus> statuses = orders.map(order ->
      paymentClient.getStatus(order.getId())  // Mono<PaymentStatus> 반환!
  );
  // 결과 타입: Flux<Mono<PaymentStatus>> — 원하는 게 아님
  // 올바른 선택: flatMap

실수 2: 순서가 중요한데 flatMap 사용

  Flux<Integer> results = Flux.range(1, 5)
      .flatMap(i -> slowApiCall(i));  // 각 호출이 랜덤 시간 소요
  // 결과: 2, 4, 1, 5, 3 (순서 보장 없음)
  // 순서 필요 시: concatMap

실수 3: flatMap으로 무제한 동시 호출

  // 주문 10,000개에 대해 외부 결제 API 호출
  orders.flatMap(order -> paymentClient.charge(order));
  // → 10,000개 동시 HTTP 요청 → 외부 API 장애 유발 or 503
  // 올바른 선택: flatMap(fn, 10)  // 동시 10개로 제한
```

---

## ✨ 올바른 접근 (After — 연산자 특성 이해)

```
연산자 선택 기준:

동기 변환 (1:1, 블로킹 없음)    → map
비동기 변환 (Mono/Flux 반환)    → flatMap (순서 무관)
비동기 변환 + 순서 보장         → concatMap (성능 희생)
최신 결과만 필요                → switchMap
동시 처리 수 제한               → flatMap(fn, n)

결합 연산자:
  두 Mono를 병렬 실행 후 합산    → Mono.zip()
  여러 Flux를 병합 (도착 순)    → Flux.merge()
  여러 Flux를 순서대로 이어붙임  → Flux.concat()
```

---

## 🔬 내부 동작 원리

### 1. map — 동기 변환, 1:1

```
마블 다이어그램 (텍스트):
  입력:  ─── 1 ─── 2 ─── 3 ─── 4 ─|
  map(x → x*10)
  출력:  ─── 10 ── 20 ── 30 ── 40 ─|

특징:
  - 동기 함수: T → R
  - 블로킹 코드 금지 (EventLoop에서 실행)
  - 입력 1개 → 출력 1개 (정확히 1:1)
  - 순서 보장

내부 동작:
  onNext(item) 수신 → mapper.apply(item) 즉시 실행 → 결과 onNext()
  예외 발생 시 → onError() 신호로 변환

적합한 사용:
  - 필드 추출: .map(User::getName)
  - 타입 변환: .map(dto -> mapper.toEntity(dto))
  - null 안전 변환: .map(s -> s.toUpperCase())
```

### 2. flatMap — 비동기 변환, 동시 처리

```
마블 다이어그램 (텍스트):
  입력:  ─── 1 ────── 2 ────── 3 ─|
  flatMap(x → asyncApi(x))
           asyncApi(1): ─── A1 ─|
           asyncApi(2): ─ A2 ─|        (2가 먼저 완료)
           asyncApi(3): ──── A3 ─|
  출력:  ─── A2 ── A1 ── A3 ─|   (순서 무보장)

특징:
  - 비동기 함수: T → Publisher<R>
  - 각 항목에 대해 내부 Publisher 생성
  - 내부 Publisher들을 동시에 구독
  - 완료된 순서대로 merge → 입력 순서 보장 없음

내부 동작:
  onNext(1) 수신 → asyncApi(1) 구독 시작
  onNext(2) 수신 → asyncApi(2) 구독 시작 (1이 아직 진행 중)
  onNext(3) 수신 → asyncApi(3) 구독 시작
  → 세 개의 내부 Publisher가 동시 진행
  → 가장 먼저 완료된 것부터 onNext 방출

concurrency 파라미터:
  flatMap(fn):      Integer.MAX_VALUE개 동시 (사실상 무제한)
  flatMap(fn, 10):  최대 10개 동시 (11번째는 앞 것 완료 후 시작)
  
  실무 권장: 외부 API 호출 시 flatMap(fn, 10~50)으로 제한
```

### 3. concatMap — 비동기 변환, 순서 보장

```
마블 다이어그램 (텍스트):
  입력:  ─── 1 ────── 2 ────── 3 ─|
  concatMap(x → asyncApi(x))
           asyncApi(1): ─── A1 ─|
                                  asyncApi(2): ─ A2 ─|    (1 완료 후 시작)
                                                       asyncApi(3): ── A3 ─|
  출력:  ───────── A1 ──── A2 ─── A3 ─|   (순서 보장)

특징:
  - 비동기 함수: T → Publisher<R>
  - 한 번에 하나씩 순차 처리
  - 이전 내부 Publisher가 완료된 후 다음 시작
  - 입력 순서 = 출력 순서

flatMap vs concatMap 성능 비교:
  각 asyncApi() 호출 시간: 200ms, 항목 5개

  flatMap:     최대 5개 동시 → 총 ~200ms (가장 오래 걸리는 것)
  concatMap:   순차 처리 → 총 ~1000ms (5 × 200ms)

적합한 사용:
  - 순서가 중요한 이벤트 처리 (메시지 발행 순서 유지)
  - 앞 결과가 다음 입력에 영향 (의존성 있는 순차 호출)
  - DB 트랜잭션 순서 보장 필요
```

### 4. switchMap — 최신 결과만 유지

```
마블 다이어그램 (텍스트):
  입력:  ─ 1 ─ 2 ─── 3 ──────────|
  switchMap(x → asyncApi(x))
           asyncApi(1): ──── [취소됨]
           asyncApi(2): ──── [취소됨]
           asyncApi(3): ────── A3 ─|
  출력:  ──────────────── A3 ─|

특징:
  - 새 입력 수신 시 이전 내부 Publisher 취소
  - 항상 최신 입력의 결과만 방출
  - 중간 결과는 버려짐

실무 활용:
  - 검색창 실시간 검색 (사용자가 빠르게 입력할 때 이전 검색 취소)
  - SSE 구독 갱신 (최신 구독만 유지)
  - 연속 버튼 클릭 시 마지막 클릭만 처리
```

### 5. 결합 연산자 — zip, merge, concat

```
Mono.zip / Flux.zip:
  여러 Publisher를 병렬 실행 → 모두 완료 시 튜플로 합산
  
  마블:
    Mono A: ─── a ─|
    Mono B: ─── b ─|
    zip:    ─────── (a,b) ─|

  Mono.zip(paymentMono, shippingMono)
      .map(tuple -> new OrderResult(tuple.getT1(), tuple.getT2()))
  // 두 API 병렬 호출, 둘 다 완료 시 합산 → 총 시간 = 더 느린 것

Flux.merge:
  여러 Flux를 동시 구독 → 도착 순서대로 방출

  마블:
    Flux A: ─ a1 ──── a2 ─|
    Flux B: ──── b1 ─ b2 ─|
    merge:  ─ a1 ─ b1 ─ a2 ─ b2 ─|

Flux.concat:
  여러 Flux를 순서대로 이어붙임 (이전 완료 후 다음 시작)

  마블:
    Flux A: ─ a1 ─ a2 ─|
    Flux B:              ─ b1 ─ b2 ─|
    concat: ─ a1 ─ a2 ─ b1 ─ b2 ─|

선택 기준:
  병렬 실행 + 합산: Mono.zip
  병렬 실행 + 순서 무관: Flux.merge
  순차 실행 + 이어붙임: Flux.concat
```

---

## 💻 실전 코드

### 실험 1: map vs flatMap 타입 차이

```java
Flux<Order> orders = orderRepository.findAll();

// 잘못된 사용 — Flux<Mono<Status>>가 됨
Flux<Mono<PaymentStatus>> wrong = orders.map(order ->
    paymentClient.getStatus(order.getId())  // Mono<PaymentStatus> 반환
);

// 올바른 사용 — flatMap이 내부 Mono를 구독하여 값 추출
Flux<PaymentStatus> correct = orders.flatMap(order ->
    paymentClient.getStatus(order.getId())
);
```

### 실험 2: flatMap vs concatMap 순서 차이 실험

```java
// 각 항목마다 랜덤한 시간이 걸리는 API 시뮬레이션
Flux.range(1, 5)
    .flatMap(i -> Mono.just(i)
        .delayElement(Duration.ofMillis(new Random().nextInt(300)))
        .map(n -> "flatMap-" + n))
    .subscribe(s -> log.info(s));
// 출력: flatMap-3, flatMap-1, flatMap-5, flatMap-2, flatMap-4 (랜덤)

Flux.range(1, 5)
    .concatMap(i -> Mono.just(i)
        .delayElement(Duration.ofMillis(new Random().nextInt(300)))
        .map(n -> "concatMap-" + n))
    .subscribe(s -> log.info(s));
// 출력: concatMap-1, concatMap-2, concatMap-3, concatMap-4, concatMap-5 (항상 순서 보장)
```

### 실험 3: flatMap으로 동시 외부 API 호출 제어

```java
// 주문 목록으로 결제 API 호출 — 동시 호출 수 제한
Flux<Order> orders = orderRepository.findAll();

Flux<PaymentResult> results = orders
    .flatMap(
        order -> paymentClient.charge(order),  // 각 주문 결제
        10  // concurrency: 최대 10개 동시 호출
    )
    .onErrorContinue((err, order) ->
        log.error("결제 실패: {}", order, err)
    );
// 10개씩 배치 처리 → 외부 API 부하 제어
```

### 실험 4: Mono.zip으로 병렬 API 호출

```java
@GetMapping("/order/{id}/summary")
public Mono<OrderSummary> getOrderSummary(@PathVariable Long id) {
    Mono<Order> order = orderRepository.findById(id);
    Mono<PaymentInfo> payment = paymentClient.getInfo(id);
    Mono<ShippingInfo> shipping = shippingClient.getInfo(id);

    // 세 개 동시 실행 — 가장 느린 것이 전체 시간 결정
    return Mono.zip(order, payment, shipping)
        .map(tuple -> OrderSummary.of(
            tuple.getT1(), tuple.getT2(), tuple.getT3()
        ));
    // 각 300ms라면 순차: 900ms, 병렬: ~300ms
}
```

---

## 📊 성능 비교

```
연산자별 동시성과 순서 보장 특성:

연산자        | 동시 처리 | 순서 보장 | 내부 Publisher 취소
─────────────┼──────────┼─────────┼────────────────
map           | -         | 예       | -
flatMap       | MAX_INT   | 아니오   | 없음
flatMap(fn,n) | n개       | 아니오   | 없음
concatMap     | 1개       | 예       | 없음
switchMap     | 1개(최신) | -        | 새 입력 시 이전 취소

처리 시간 비교 (항목 5개, 각 200ms 지연):

flatMap:    ~200ms  (모두 동시)
concatMap:  ~1000ms (순차, 5 × 200ms)
flatMap(,2): ~600ms (2개씩 처리, 3라운드)

결합 연산자:
  Mono.zip(A, B):   병렬 실행, max(A, B) 시간
  Flux.merge(A, B): 병렬 실행, 도착 순
  Flux.concat(A, B): 순차 실행, A + B 시간
```

---

## ⚖️ 트레이드오프

```
연산자 선택 트레이드오프:

flatMap:
  장점: 빠른 처리 (동시 실행)
  단점: 순서 무보장, 동시 수 제한 없으면 외부 API 과부하 위험

concatMap:
  장점: 순서 보장
  단점: 순차 처리 → 전체 처리 시간 = 항목 수 × 단일 처리 시간

switchMap:
  장점: 최신 결과만 처리 → 불필요한 작업 취소
  단점: 중간 결과 소실 → 중간 결과가 필요한 경우 부적합

flatMap(fn, n) 동시성 제한:
  너무 낮은 n: 처리량 저하
  너무 높은 n: 외부 서비스 부하, 내 서버 메모리 압박
  실무 권장: 외부 API는 10~50, 내부 DB는 20~100
```

---

## 📌 핵심 정리

```
Reactor 연산자 핵심:

map:      동기 1:1 변환 → T → R
flatMap:  비동기 변환 + 동시 실행 → T → Publisher<R>, 순서 무관
concatMap: 비동기 변환 + 순차 실행 → T → Publisher<R>, 순서 보장
switchMap: 비동기 변환 + 최신만 유지 → 이전 것 취소

결합:
  zip:    병렬 실행 + 튜플 합산 (모두 완료 대기)
  merge:  병렬 구독 + 도착 순 방출
  concat: 순차 구독 + 이어붙임

실무 선택 공식:
  Mono 반환 함수 → flatMap
  순서 중요       → concatMap
  최신만 필요     → switchMap
  병렬 + 합산    → zip
  동시 수 제한    → flatMap(fn, n)
```

---

## 🤔 생각해볼 문제

**Q1.** `flatMap` 내부에서 에러가 발생하면 전체 Flux가 종료되나요?

<details>
<summary>해설 보기</summary>

기본적으로 내부 Publisher에서 `onError`가 발생하면 전체 외부 Flux도 `onError`로 종료됩니다.

개별 항목의 에러를 무시하고 계속 처리하려면:

```java
orders.flatMap(order ->
    paymentClient.charge(order)
        .onErrorResume(e -> {
            log.error("결제 실패: {}", order.getId(), e);
            return Mono.empty();  // 이 항목은 건너뜀
        })
)

// 또는 onErrorContinue (스트림 수준)
orders.flatMap(order -> paymentClient.charge(order))
    .onErrorContinue((err, item) ->
        log.error("처리 실패, 계속 진행: {}", item)
    );
```

`onErrorResume`은 각 내부 Publisher 수준에서 에러를 격리하므로 더 명확하고 권장됩니다.

</details>

---

**Q2.** `Flux.zip(fluxA, fluxB)`와 `Mono.zip(monoA, monoB)`의 내부 동작은 어떻게 다른가요?

<details>
<summary>해설 보기</summary>

`Mono.zip(monoA, monoB)`:
- 두 Mono를 동시 구독
- 둘 다 `onNext`가 도착하면 튜플 생성 후 `onComplete`
- 하나라도 `onError` → 전체 onError

`Flux.zip(fluxA, fluxB)`:
- 두 Flux를 동시 구독
- **인덱스 기준으로 쌍을 맞춤**: fluxA의 첫 번째 + fluxB의 첫 번째, 두 번째 + 두 번째...
- 어느 한 쪽이 먼저 완료되면 전체 완료
- 처리량이 다른 Flux를 같은 속도로 맞출 때 유용

```java
Flux<String> names = Flux.just("Alice", "Bob", "Charlie");
Flux<Integer> scores = Flux.just(90, 85, 92);

Flux.zip(names, scores)
    .map(t -> t.getT1() + ": " + t.getT2())
    .subscribe(log::info);
// Alice: 90, Bob: 85, Charlie: 92
```

</details>

---

**Q3.** `flatMapSequential`은 `flatMap`과 `concatMap`의 어떤 특성을 각각 갖나요?

<details>
<summary>해설 보기</summary>

`flatMapSequential`은 두 가지 특성을 조합합니다.

- `flatMap`처럼: 내부 Publisher를 **동시에** 구독 (성능)
- `concatMap`처럼: 입력 순서대로 결과를 **재정렬** 후 방출 (순서 보장)

```java
// 동시에 실행되지만 결과는 입력 순서 유지
Flux.range(1, 5)
    .flatMapSequential(i -> Mono.just(i)
        .delayElement(Duration.ofMillis(new Random().nextInt(300)))
    )
    .subscribe(i -> log.info("{}", i));
// 출력: 1, 2, 3, 4, 5 (순서 보장, 하지만 내부에서는 병렬 실행)
```

내부적으로는 빠르게 완료된 내부 Publisher의 결과를 임시 버퍼에 저장했다가, 입력 순서에 맞춰 방출합니다. 순서와 성능 모두 필요할 때 사용합니다. 단, 버퍼 사용으로 메모리가 `flatMap`보다 더 필요할 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Mono와 Flux 지연 평가](./01-mono-flux-lazy-evaluation.md)** | **[홈으로 🏠](../README.md)** | **[다음: 에러 처리 ➡️](./03-error-handling.md)**

</div>
