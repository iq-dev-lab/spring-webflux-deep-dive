# Mono와 Flux — 왜 subscribe() 전에 실행되지 않는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Mono`와 `Flux`는 선언 시점에 왜 아무 것도 실행하지 않는가?
- Cold Publisher의 지연 평가(Lazy Evaluation) 원리는 무엇인가?
- `subscribe()` 호출 시 내부에서 정확히 어떤 순서로 무슨 일이 일어나는가?
- `Mono`(0~1개)와 `Flux`(0~N개)의 내부 구현 차이는 무엇인가?
- WebFlux 컨트롤러에서 `subscribe()`를 직접 호출하지 않아도 요청이 처리되는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

WebFlux에서 가장 흔한 버그 유형은 "파이프라인을 만들었는데 실행이 안 된다"입니다. 원인은 단 하나 — `subscribe()`가 없어서입니다. 하지만 "왜 subscribe() 없이는 실행이 안 되는가"를 정확히 모르면, 다음에 비슷한 상황을 만났을 때 또 같은 실수를 반복합니다.

또한 `subscribe()`를 남용하면 WebFlux의 이벤트 루프 모델이 깨집니다. 언제 WebFlux가 자동으로 구독을 처리하고, 언제 직접 호출해야 하는지를 구분하는 것이 WebFlux 숙달의 첫 번째 관문입니다.

---

## 😱 흔한 실수 (Before — 지연 평가를 모를 때)

```
실수 1: subscribe() 없이 실행을 기대

  Mono<Void> saveResult = userRepository.save(newUser);
  log.info("저장 완료!");  // ← 실제로 save()가 호출되지 않음!

  이유: Mono<Void>는 파이프라인 설명서일 뿐
        subscribe() 없이는 실행 자체가 없음
        R2DBC save()도 호출 안 됨

실수 2: 동일 Mono를 두 번 subscribe → 중복 실행

  Mono<User> findUser = userRepository.findById(id);
  findUser.subscribe(u -> log.info("로그: {}", u));  // DB 쿼리 1번
  return findUser;  // WebFlux가 구독 → DB 쿼리 또 1번!

실수 3: Service에서 subscribe() 후 반환

  public Mono<String> process() {
      Mono<String> result = webClient.get().uri("/api").retrieve()
          .bodyToMono(String.class);
      result.subscribe();  // 여기서 구독 (실행 시작)
      return result;       // WebFlux가 또 구독 → 두 번 실행
  }
```

---

## ✨ 올바른 접근 (After — 지연 평가를 이해한 패턴)

```
핵심 원칙:
  Mono/Flux = 데이터 흐름의 "설계도"
  subscribe() = "설계도대로 실제 실행 시작"
  WebFlux 컨트롤러 = Mono/Flux 반환 시 WebFlux가 구독

올바른 패턴:

1. 컨트롤러: 항상 Mono/Flux 반환 (직접 subscribe 불필요)
   @GetMapping("/user/{id}")
   public Mono<User> getUser(@PathVariable Long id) {
       return userRepository.findById(id);  // WebFlux가 구독
   }

2. "fire and forget" 시나리오
   notificationService.send(event)
       .subscribeOn(Schedulers.boundedElastic())
       .subscribe(null, err -> log.error("알림 실패", err));
   return Mono.empty();

3. 부수 효과는 doOn* 연산자로
   return userRepository.findById(id)
       .doOnNext(user -> log.info("조회: {}", user.getId()))
       .doOnError(e -> log.error("조회 실패", e));
   // subscribe는 WebFlux가 한 번만 호출
```

---

## 🔬 내부 동작 원리

### 1. Mono와 Flux의 정체 — 파이프라인 설계도

```
Mono<T>와 Flux<T>는 추상 클래스:

  public abstract class Mono<T> implements Publisher<T> {
      // 실제 데이터나 실행 코드가 없음
      // "어떻게 데이터를 생산할지"에 대한 명세만 존재
  }

구체 구현체 예시:
  Mono.just("hello")       → MonoJust     (값 캡처, 즉시 평가)
  Mono.fromCallable(fn)    → MonoCallable (함수 저장, 지연 평가)
  webClient.get().uri(...) → 내부 체인    (HTTP 설정만 저장)
  Flux.range(1, 100)       → FluxRange    (범위 기반 생성 설계도)

핵심:
  MonoJust("hello"):  "hello"가 이미 메모리에 있음
  MonoCallable(fn):   fn 자체만 저장, 아직 실행 안 함
  HTTP Mono:          요청 설정만 저장, 요청 안 보냄

  → subscribe() 전까지 어떤 I/O, DB 쿼리, 계산도 없음
  → 이것이 지연 평가(Lazy Evaluation)의 실체
```

### 2. subscribe() 호출 시 발생하는 일 — 단계별 분해

```
코드 예시:
  Flux.range(1, 3)
      .map(i -> i * 10)
      .subscribe(item -> log.info("수신: {}", item));

subscribe() 내부 실행 순서:

Step 1: 람다 → LambdaSubscriber 래핑

Step 2: 구독 체인 연결 (아래 → 위)
  LambdaSubscriber → MapSubscriber 생성
  MapSubscriber → FluxRange.subscribe(MapSubscriber)

Step 3: FluxRange가 RangeSubscription 생성
  → MapSubscriber.onSubscribe(rangeSubscription)

Step 4: onSubscribe 체인 (위 → 아래)
  MapSubscriber → LambdaSubscriber.onSubscribe(mappedSub)

Step 5: LambdaSubscriber.hookOnSubscribe
  → request(Long.MAX_VALUE) 자동 호출

Step 6: request 역방향 전파 (아래 → 위)
  LambdaSubscriber → MapSubscriber → FluxRange

Step 7: FluxRange 실행 시작
  onNext(1) → map → 10 → log.info("수신: 10")
  onNext(2) → map → 20 → log.info("수신: 20")
  onNext(3) → map → 30 → log.info("수신: 30")
  onComplete()
```

### 3. Mono vs Flux — 선택 기준

```
Mono<T> (0 또는 1개 값):
  단일 결과, 경량, Fusion 최적화 가능

Flux<T> (0~N개 값):
  다수 값 스트림, 무한 스트림 표현 가능
  Backpressure 처리 포함

변환:
  Mono → Flux: mono.flux(), mono.flatMapMany(...)
  Flux → Mono: flux.next(), flux.collectList(), flux.reduce()

WebFlux 컨트롤러 선택:
  단일 객체    → Mono<User>
  컬렉션       → Flux<User> 또는 Mono<List<User>>
  스트리밍     → Flux<ServerSentEvent>
  빈 응답      → Mono<Void>
```

### 4. WebFlux 컨트롤러에서의 자동 구독

```
DispatcherHandler 처리 흐름:

@GetMapping("/user/{id}")
public Mono<User> getUser(@PathVariable Long id) {
    return userRepository.findById(id);
}

1. HTTP 요청 수신 (Netty EventLoop)
2. RouterFunction으로 핸들러 메서드 찾기
3. 핸들러 메서드 호출 → Mono<User> 수신 (아직 실행 안 됨)
4. HttpMessageWriter로 JSON 직렬화 파이프라인 연결
5. 내부적으로 subscribe() 호출 → 파이프라인 실행 시작
6. User 수신 → JSON 직렬화 → HTTP 응답

핵심:
  컨트롤러 개발자는 Mono/Flux만 반환
  구독과 응답 변환은 WebFlux가 처리
  컨트롤러에서 subscribe() 직접 호출 = 불필요하거나 위험
```

### 5. Mono.just vs Mono.fromCallable vs Mono.defer

```
Mono.just(value):
  즉시 값 캡처 — 선언 시점에 평가
  Mono<LocalDateTime> wrong = Mono.just(LocalDateTime.now());
  // "지금"이 고정됨, subscribe() 늦어도 같은 시간

Mono.fromCallable(fn):
  subscribe() 시점에 fn 실행 — 진정한 지연 평가
  Mono<LocalDateTime> right = Mono.fromCallable(LocalDateTime::now);
  // subscribe() 시점의 현재 시간 반환

Mono.defer(supplier):
  subscribe()마다 새로운 Mono 생성
  Mono<User> deferred = Mono.defer(() ->
      Mono.just(userCache.getCurrentUser())
  );
  // subscribe()마다 userCache 재호출 → 항상 최신 값

실무 선택:
  고정 값:            Mono.just()
  블로킹 코드 래핑:    Mono.fromCallable() + subscribeOn(boundedElastic())
  subscribe마다 재실행: Mono.defer()
  조건부 Mono 생성:    Mono.defer(() -> condition ? Mono.just(a) : Mono.just(b))
```

---

## 💻 실전 코드

### 실험 1: subscribe() 없이 실행되지 않음 증명

```java
@Test
void lazyEvaluationProof() {
    AtomicBoolean executed = new AtomicBoolean(false);

    Mono<String> mono = Mono.fromCallable(() -> {
        executed.set(true);
        return "result";
    });

    assertThat(executed.get()).isFalse();  // 아직 실행 안 됨

    mono.subscribe();

    assertThat(executed.get()).isTrue();   // 이제 실행됨
}
```

### 실험 2: Cold Publisher — subscribe마다 독립 실행

```java
@Test
void coldPublisherIndependentExecution() {
    AtomicInteger callCount = new AtomicInteger(0);

    Mono<String> coldMono = Mono.fromCallable(() -> {
        int count = callCount.incrementAndGet();
        return "result-" + count;
    });

    coldMono.subscribe(r -> log.info("A: {}", r));  // result-1
    coldMono.subscribe(r -> log.info("B: {}", r));  // result-2

    assertThat(callCount.get()).isEqualTo(2);
    // subscribe마다 완전히 새로운 실행
}
```

### 실험 3: doOnNext로 부수 효과 분리

```java
// 잘못된 방식 — 두 번 구독
Mono<User> user = userRepository.findById(id);
user.subscribe(u -> log.info("로그: {}", u));  // DB 쿼리 1
return user;  // DB 쿼리 2

// 올바른 방식 — doOnNext로 부수 효과
return userRepository.findById(id)
    .doOnNext(u -> log.info("로그: {}", u));  // DB 쿼리 1번, 로그 포함
```

### 실험 4: Mono.defer 활용

```java
// 매번 최신 baseUrl을 반영하는 파이프라인
public Mono<Response> callExternalApi(String path) {
    return Mono.defer(() -> {
        String baseUrl = configService.getBaseUrl();  // subscribe 시점 최신값
        return webClient.mutate().baseUrl(baseUrl).build()
            .get().uri(path)
            .retrieve()
            .bodyToMono(Response.class);
    });
}
```

---

## 📊 성능 비교

```
Mono 생성 방식별 지연 평가 특성:

생성 방식              | 평가 시점       | 매번 재실행
─────────────────────┼──────────────┼──────────
Mono.just(value)     | 즉시 (선언 시) | 아니오
Mono.fromCallable()  | subscribe 시  | 예
Mono.defer()         | subscribe 시  | 예 (Mono 재생성)
Mono.fromFuture()    | 즉시 시작      | 아니오
Mono.create()        | subscribe 시  | 예

Mono vs Flux 메모리 비교 (1만 건 조회):
  Mono<List<User>>: 1만 건 전체를 메모리에 → JVM Heap 사용량 큼
  Flux<User>:       항목 단위 스트리밍 → Heap 사용량 최소
  → 대용량: Flux 필수, 소규모: 차이 없음
```

---

## ⚖️ 트레이드오프

```
지연 평가(Lazy Evaluation):

장점:
  ① 필요할 때만 실행 → 불필요한 I/O 방지
  ② 파이프라인 합성 용이 → 실행 없이 Mono/Flux 조합
  ③ 취소 지원 → subscribe 안 하면 자원 소비 없음

단점:
  ① 초보자 실수 유발 → "subscribe() 빼먹음" 버그
  ② 실행 시점 예측 어려움
  ③ 부수 효과 관리 복잡 (subscribe 없이는 절대 실행 안 됨)

컨트롤러 vs Service 구독 규칙:
  컨트롤러: 반환만 → WebFlux 자동 구독
  Service:  Mono/Flux 반환 → 상위에서 구독
  fire-and-forget: subscribe() 명시 (에러 핸들러 필수)
```

---

## 📌 핵심 정리

```
Mono/Flux 지연 평가 핵심:

정의:
  Mono/Flux = 데이터 흐름 설계도 (Cold Publisher)
  subscribe() = 설계도대로 실제 실행 시작

subscribe() 시 순서:
  1. Subscriber 래핑
  2. 구독 체인 연결 (아래→위)
  3. onSubscribe 전파 (위→아래)
  4. request(n) 전파 (아래→위)
  5. 데이터 흐름 시작 (위→아래)

Mono vs Flux:
  Mono: 0~1개, 경량, 단일 결과
  Flux: 0~N개, 스트림, 대용량/스트리밍

WebFlux 구독 규칙:
  컨트롤러: 반환 → WebFlux가 구독 (한 번)
  동일 Mono 두 번 구독: 금지 → doOnNext 사용
  fire-and-forget: subscribe() 명시 + 에러 핸들러
```

---

## 🤔 생각해볼 문제

**Q1.** `Mono.fromFuture(CompletableFuture.supplyAsync(() -> fetch()))`는 지연 평가인가요?

<details>
<summary>해설 보기</summary>

아닙니다. `CompletableFuture.supplyAsync()`는 `Mono.fromFuture()`가 호출되는 순간 이미 시작됩니다.

```java
// 즉시 평가 — CompletableFuture가 선언 시점에 시작됨
Mono<String> eager = Mono.fromFuture(
    CompletableFuture.supplyAsync(() -> fetch())  // 이미 실행 중!
);

// 지연 평가로 만들려면 Mono.defer() 사용
Mono<String> lazy = Mono.defer(() ->
    Mono.fromFuture(CompletableFuture.supplyAsync(() -> fetch()))
);
// subscribe() 시에만 CompletableFuture 생성
```

서비스 메서드에서 `Mono.fromFuture()`를 재사용할 때 이 차이가 중요합니다. 지연 평가가 필요하다면 반드시 `Mono.defer()`로 감싸야 합니다.

</details>

---

**Q2.** `Mono<Void>`를 반환하는 WebFlux 컨트롤러에서 예외가 발생하면 어떻게 처리되나요?

<details>
<summary>해설 보기</summary>

`Mono<Void>` 파이프라인에서 `onError` 신호가 발생하면 WebFlux는 기본적으로 HTTP 500을 응답합니다.

```java
@DeleteMapping("/user/{id}")
public Mono<Void> deleteUser(@PathVariable Long id) {
    return userRepository.deleteById(id)
        .then(cacheService.evict(id));  // 여기서 예외 → HTTP 500
}

// 명시적 에러 처리:
@DeleteMapping("/user/{id}")
public Mono<ResponseEntity<Void>> deleteUser(@PathVariable Long id) {
    return userRepository.deleteById(id)
        .then(cacheService.evict(id))
        .then(Mono.just(ResponseEntity.<Void>noContent().build()))
        .onErrorReturn(CacheException.class,
            ResponseEntity.status(207).build());
}
```

중요한 작업에서는 명시적인 에러 처리를 포함하는 것을 권장합니다.

</details>

---

**Q3.** `flux.collectList()`와 `flux.buffer(n)` 중 대용량 데이터에서 어떤 것이 안전한가요?

<details>
<summary>해설 보기</summary>

`collectList()`는 모든 항목을 메모리에 수집 후 `Mono<List<T>>`를 반환합니다. 대용량 시 OOM 위험이 있습니다.

`buffer(n)`은 n개씩 묶어 `Flux<List<T>>`를 반환, 전체를 올리지 않아 안전합니다.

```java
// 위험: 100만 건 전부 메모리에
Mono<List<User>> all = userRepository.findAll().collectList();

// 안전: 1,000건씩 배치 처리
Flux<List<User>> batches = userRepository.findAll().buffer(1000);
batches.flatMap(batch -> processBatch(batch)).subscribe();

// 가장 효율: 항목 단위 스트리밍
userRepository.findAll()
    .flatMap(user -> processUser(user), 10)  // 동시 10개
    .subscribe();
```

`Flux<User>`를 `Mono<List<User>>`로 바꾸는 순간 Flux의 메모리 효율 이점이 사라집니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Reactor 연산자 완전 분해 ➡️](./02-reactor-operators.md)**

</div>
