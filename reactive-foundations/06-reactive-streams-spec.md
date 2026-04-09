# Reactive Streams 스펙 완전 분해 — push vs pull

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Reactive Streams의 신호 전파 방향은 구독(subscribe)할 때와 데이터가 흐를 때 어떻게 다른가?
- `onNext()`는 언제, 무슨 조건에서 호출되는가? 무작정 호출될 수 있는가?
- Backpressure의 실체는 무엇이고, `request(n)` 신호가 어떻게 생산자를 제어하는가?
- Reactor에서 `Flux.subscribe()`의 람다 호출 시 내부에서 어떤 신호 흐름이 발생하는가?
- `onError`와 `onComplete` 이후 `onNext`가 호출되면 스펙 위반인가?

---

## 🔍 왜 이 개념이 WebFlux에서 중요한가

"Reactive Streams 스펙을 안다"는 것은 단순히 4개의 인터페이스 이름을 외우는 것이 아닙니다. WebFlux 코드를 작성하다 마주치는 대부분의 버그와 혼란은 이 스펙의 계약을 이해하지 못해서 발생합니다.

`subscribe()`를 호출했는데 왜 코드가 실행되지 않을까? `flatMap` 안에서 왜 에러가 파이프라인 밖으로 전파될까? `request(Long.MAX_VALUE)`가 왜 기본값일까? — 이 모든 것이 Reactive Streams 스펙의 계약에서 비롯됩니다. 이 문서는 스펙을 신호 흐름 관점에서 완전히 분해합니다.

---

## 😱 흔한 실수 (Before — 스펙 계약을 모를 때)

```
실수 1: subscribe() 없이 파이프라인 실행을 기대
  Mono<String> result = webClient.get()
      .uri("/api/data")
      .retrieve()
      .bodyToMono(String.class)
      .map(s -> s.toUpperCase());
  
  log.info("요청 완료!");  ← 실제로 요청이 전혀 안 보내짐
  
  이유: subscribe()가 없으면 request(n) 신호가 생산자에게 전달되지 않음
        → 데이터 흐름 자체가 시작되지 않음
        → Cold Publisher의 핵심 특성

실수 2: onError 이후 onNext를 받을 것을 기대
  flux.subscribe(
      item -> process(item),
      error -> log.error("에러", error),
      () -> log.info("완료")
  );
  // onError 발생 후 process()가 더 호출될 것을 기대 → 절대 없음
  // 스펙: onError 또는 onComplete 이후 onNext 없음

실수 3: request(n) 없이 onNext가 올 것을 기대
  // 커스텀 Subscriber 구현 시
  @Override
  public void onSubscribe(Subscription s) {
      // s.request(n) 호출 안 함
      // → onNext()가 절대 호출되지 않음
      // → 파이프라인 영원히 대기
  }

실수 4: onNext 안에서 예외 발생 시 동작 오해
  flux.subscribe(item -> {
      throw new RuntimeException("처리 실패");  // 여기서 예외 발생
  });
  // → onError로 라우팅되지 않고 예외가 그냥 전파됨 (Reactor 기본 동작)
  // 올바른 패턴: map/flatMap 등 연산자 안에서 예외 → onError 신호로 변환
```

---

## ✨ 올바른 접근 (After — 스펙 계약을 이해한 코드)

```
핵심 계약 이해 후 올바른 패턴:

1. subscribe()가 항상 필요
   // 반환값이 있는 컨트롤러는 WebFlux가 내부적으로 subscribe() 호출
   @GetMapping("/data")
   public Mono<String> getData() {
       return webClient.get().uri("/api").retrieve().bodyToMono(String.class);
       // WebFlux DispatcherHandler가 Mono를 구독 → 결과를 HTTP 응답으로 변환
   }
   
   // Service 레이어에서 직접 구독이 필요한 경우
   public void fireAndForget(Event event) {
       eventBus.publish(event).subscribe(
           null,    // onNext 무시
           error -> log.error("발행 실패", error)
       );
       // subscribe() 없으면 이벤트 발행 안 됨!
   }

2. 에러 처리는 파이프라인에서
   webClient.get().uri("/api").retrieve()
       .bodyToMono(String.class)
       .map(s -> {
           if (s.isEmpty()) throw new IllegalArgumentException("empty");
           return s.toUpperCase();
       })
       // map 안의 예외 → onError 신호로 자동 변환
       .onErrorReturn("기본값")
       .subscribe(result -> log.info("결과: {}", result));
```

---

## 🔬 내부 동작 원리

### 1. 신호 전파의 두 방향 — 구독 설정(아래→위)과 데이터 흐름(위→아래)

```
파이프라인 예시:
  Flux.range(1, 5)
      .filter(i -> i % 2 == 0)
      .map(i -> i * 10)
      .subscribe(result -> log.info("결과: {}", result));

물리적 구조 (위→아래가 데이터 방향):
  Flux.range(1,5)   ← Publisher (데이터 생산자)
       │
  filter(i % 2)     ← Processor (Subscriber이자 Publisher)
       │
  map(i * 10)       ← Processor
       │
  subscribe(...)    ← Subscriber (데이터 소비자)

─────────────────────────────────────────

1단계: 구독 설정 (아래 → 위 방향)
  subscribe() 호출
    ↑
  map이 filter를 subscribe
    ↑
  filter가 range를 subscribe
    ↑
  range: Subscription 생성 → filter의 onSubscribe(sub) 호출
    ↓
  filter: Subscription 래핑 → map의 onSubscribe(filteredSub) 호출
    ↓
  map: Subscription 래핑 → 최종 Subscriber의 onSubscribe(mappedSub) 호출

결과: 구독 체인 완성 (각 단계가 자신의 Subscription 보유)

─────────────────────────────────────────

2단계: request(n) 신호 (아래 → 위 방향)
  최종 Subscriber: subscription.request(Long.MAX_VALUE)
    ↑
  map: 상위 subscription.request(Long.MAX_VALUE)
    ↑
  filter: 상위 subscription.request(Long.MAX_VALUE)
    ↑
  range: "Long.MAX_VALUE개 달라는 요청 수신 → 발행 시작"

─────────────────────────────────────────

3단계: 데이터 흐름 (위 → 아래 방향)
  range: onNext(1) → filter
  filter: 1 % 2 ≠ 0 → 건너뜀 (다음 request 불필요, 상위에 다시 요청)
  range: onNext(2) → filter
  filter: 2 % 2 = 0 → map.onNext(2)
  map: 2 * 10 = 20 → subscriber.onNext(20)
  subscriber: log.info("결과: 20")
  ...
  range: onComplete() → filter → map → subscriber.onComplete()
```

### 2. Backpressure 내부 동작 — request(n)의 실체

```
Backpressure 시나리오:
  느린 소비자가 빠른 생산자를 제어

Flux.range(1, 100)
    .subscribe(new BaseSubscriber<Integer>() {
        int processed = 0;
        
        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            log.info("구독 완료 — 5개 요청");
            request(5);  // "처음엔 5개만 줘"
        }
        
        @Override
        protected void hookOnNext(Integer value) {
            log.info("수신: {}", value);
            processed++;
            slowConsumer(value);  // 느린 처리
            
            if (processed % 5 == 0) {
                log.info("5개 처리 완료 — 5개 더 요청");
                request(5);  // "5개 더 줘"
            }
        }
    });

신호 흐름:
  subscribe() → onSubscribe() → request(5)
  range: onNext(1), onNext(2), onNext(3), onNext(4), onNext(5)
  range: 5개 발행 후 대기 (추가 request 없으므로)
  소비자: 5개 처리 완료 → request(5)
  range: onNext(6), onNext(7), ... onNext(10)
  ... 반복 ...
  range: onNext(100) → onComplete()

메모리 사용:
  생산자가 100개를 모두 만들지 않음
  각 request(5)에 응답으로 5개씩만 생성
  → 동시에 메모리에 올라오는 데이터: 최대 5개
  → O(request 크기) 메모리

Reactor의 기본 Backpressure:
  람다로 subscribe() 호출 시: request(Long.MAX_VALUE) 자동 호출
  → 무제한 요청 (unbounded)
  → 생산자가 최대 속도로 발행
  → 소비자가 충분히 빠르면 문제 없음
  → 소비자가 느리면 내부 버퍼로 처리 (onBackpressureBuffer)
```

### 3. Reactor 신호 종류와 의미

```
Reactive Streams 6가지 신호:

1. subscribe()
   - Subscriber → Publisher 방향
   - "나는 이 Publisher를 구독하겠다"
   - → Publisher는 새 Subscription 생성

2. onSubscribe(Subscription s)
   - Publisher → Subscriber 방향
   - "구독 받았다, 이 Subscription으로 제어해라"
   - Subscriber는 여기서 request(n) 첫 호출

3. request(n)
   - Subscriber → Publisher 방향 (Subscription 통해)
   - "n개의 onNext를 보내도 된다"
   - n = Long.MAX_VALUE: 제한 없이 보내도 됨
   - Publisher는 n개 이하의 onNext만 발행 가능

4. onNext(T item)
   - Publisher → Subscriber 방향
   - "요청받은 데이터 하나 전달"
   - request(n) 없이는 절대 호출 불가 (스펙 위반)
   - 누적 호출 수 ≤ 누적 request(n) 합

5. onError(Throwable t)
   - Publisher → Subscriber 방향
   - "에러 발생 — 스트림 종료"
   - 이후 onNext, onComplete 호출 없음 (스펙 보장)
   - 취소(cancel)가 없어도 스트림 종료

6. onComplete()
   - Publisher → Subscriber 방향
   - "모든 데이터 발행 완료"
   - 이후 onNext, onError 호출 없음 (스펙 보장)

추가: cancel() (Subscription)
   - Subscriber → Publisher 방향 (Subscription 통해)
   - "더 이상 데이터 필요 없다"
   - Publisher는 합리적인 시간 내에 발행 중단
   - 즉시 중단 보장 없음 (스펙)

Reactor에서 이를 표현하는 신호:
  onSubscribe → doOnSubscribe()
  onNext      → doOnNext()
  onError     → doOnError()
  onComplete  → doOnComplete()
  cancel()    → doOnCancel()
  request(n)  → doOnRequest()
```

### 4. map 연산자 내부에서 신호 변환

```
map(Function<T, R> mapper)이 내부적으로 하는 일:

// 개념적 구현 (실제 Reactor 코드와 단순화 차이 있음)
class MapSubscriber<T, R> implements Subscriber<T>, Subscription {
    private final Subscriber<R> actual;  // 하위 Subscriber
    private final Function<T, R> mapper;
    private Subscription upstream;       // 상위 Subscription

    // 상위 Publisher가 onSubscribe 호출 시
    @Override
    public void onSubscribe(Subscription s) {
        this.upstream = s;
        // 하위 Subscriber에게 자신을 Subscription으로 전달
        actual.onSubscribe(this);
    }

    // 하위 Subscriber가 request(n) 호출 시
    @Override
    public void request(long n) {
        // 그대로 상위 Publisher에 전달
        upstream.request(n);
    }

    // 상위 Publisher가 onNext 호출 시
    @Override
    public void onNext(T item) {
        R result;
        try {
            result = mapper.apply(item);  // 변환 함수 적용
        } catch (Exception e) {
            // mapper에서 예외 발생 → onError로 변환
            upstream.cancel();           // 상위 취소
            actual.onError(e);           // 하위에 에러 신호
            return;
        }
        actual.onNext(result);  // 변환된 값 하위 전달
    }

    @Override
    public void onError(Throwable t) { actual.onError(t); }

    @Override
    public void onComplete() { actual.onComplete(); }
}

핵심 확인:
  mapper 안의 예외 → onError 신호로 자동 변환
  request(n) → 상위로 그대로 전달 (map은 1:1 변환)
  filter: 조건 불만족 시 upstream.request(1) 추가 호출 (소비자 요청 충족 위해)
```

### 5. Cold Publisher vs Hot Publisher — request(n) 관점

```
Cold Publisher (Reactor 기본):
  subscribe()마다 새로운 데이터 스트림 생성
  각 Subscriber가 처음부터 모든 데이터를 받음

  Flux<Integer> cold = Flux.range(1, 3);
  cold.subscribe(i -> log.info("A: {}", i));  // A: 1, 2, 3
  cold.subscribe(i -> log.info("B: {}", i));  // B: 1, 2, 3
  // 독립적인 두 스트림 — 각자 처음부터

Hot Publisher:
  구독 시점과 무관하게 데이터 발행
  늦게 구독한 Subscriber는 이전 데이터를 못 받음

  Sinks.Many<Integer> sink = Sinks.many().multicast().onBackpressureBuffer();
  Flux<Integer> hot = sink.asFlux();

  hot.subscribe(i -> log.info("A: {}", i));
  sink.tryEmitNext(1);  // A: 1
  sink.tryEmitNext(2);  // A: 2

  hot.subscribe(i -> log.info("B: {}", i));  // B는 이 이후부터
  sink.tryEmitNext(3);  // A: 3, B: 3
  // B는 1, 2를 못 받음 — Hot Publisher 특성

WebFlux에서 Hot Publisher 활용:
  실시간 이벤트 스트림 (주식 시세, 알림)
  Server-Sent Events (SSE) 브로드캐스트
  WebSocket 메시지 브로드캐스트

Hot Publisher + Backpressure:
  Hot Publisher는 소비자 요청과 무관하게 발행
  소비자가 느리면 → onBackpressureBuffer/Drop/Latest 전략 적용
```

---

## 💻 실전 코드

### 실험 1: 신호 흐름 전체 로깅으로 확인

```java
// Reactor의 log() 연산자로 모든 신호 로깅
Flux.range(1, 5)
    .log("range")       // range Publisher의 신호 로깅
    .filter(i -> i % 2 == 0)
    .log("filter")      // filter 이후 신호 로깅
    .map(i -> i * 10)
    .log("map")         // map 이후 신호 로깅
    .subscribe(
        item -> log.info("최종: {}", item),
        error -> log.error("에러", error),
        () -> log.info("완료")
    );

// 출력 (로그 레벨 DEBUG):
// range  | onSubscribe([synchronous Fuseable])
// filter | onSubscribe([Fuseable Conditional])
// map    | onSubscribe([Fuseable Conditional])
// map    | request(unbounded)         ← subscribe()에서 request(MAX) 전달
// filter | request(unbounded)         ← 상위로 전달
// range  | request(unbounded)         ← 최상위까지 전달
// range  | onNext(1)                  ← 데이터 시작
// filter | onNext(1)  [조건 불만족, 건너뜀]
// range  | onNext(2)
// filter | onNext(2)  [조건 만족]
// map    | onNext(2)
// map    | onNext(20)                 ← 변환 후 하위로
// 최종: 20
// range  | onNext(3) [건너뜀]
// range  | onNext(4) → filter → map → 최종: 40
// range  | onNext(5) [건너뜀]
// range  | onComplete()
// filter | onComplete()
// map    | onComplete()
// 완료
```

### 실험 2: Backpressure 신호 흐름 추적

```java
// Backpressure를 명시적으로 제어하는 커스텀 Subscriber
Flux.range(1, 20)
    .log("source")
    .subscribe(new BaseSubscriber<Integer>() {
        int count = 0;

        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            log.info("구독 — 3개 요청");
            request(3);
        }

        @Override
        protected void hookOnNext(Integer value) {
            log.info("수신: {}", value);
            count++;
            if (count % 3 == 0) {
                log.info("3개 처리 완료 — 3개 더 요청");
                request(3);
            }
        }

        @Override
        protected void hookOnComplete() {
            log.info("전체 완료");
        }
    });

// 로그 출력:
// 구독 — 3개 요청
// source | request(3)           ← request 신호 상위 전달
// source | onNext(1)
// 수신: 1
// source | onNext(2)
// 수신: 2
// source | onNext(3)
// 수신: 3
// 3개 처리 완료 — 3개 더 요청
// source | request(3)           ← 다음 3개 요청
// source | onNext(4)
// ...
```

### 실험 3: StepVerifier로 신호 검증

```java
// StepVerifier — Reactive Streams 신호 단계별 검증
@Test
void testSignalFlow() {
    Flux<Integer> flux = Flux.range(1, 5)
                             .filter(i -> i % 2 == 0)
                             .map(i -> i * 10);

    StepVerifier.create(flux)
        .expectSubscription()          // onSubscribe 신호 검증
        .expectNext(20)                // 첫 번째 onNext(20)
        .expectNext(40)                // 두 번째 onNext(40)
        .expectComplete()             // onComplete 신호 검증
        .verify();
}

// 에러 신호 검증
@Test
void testErrorSignal() {
    Flux<Integer> flux = Flux.range(1, 3)
        .map(i -> {
            if (i == 2) throw new RuntimeException("에러!");
            return i;
        });

    StepVerifier.create(flux)
        .expectNext(1)                 // onNext(1)
        .expectErrorMessage("에러!")   // onError 신호 검증
        .verify();
    // onError 이후 onNext 없음 — 스펙 보장
}

// Backpressure 검증
@Test
void testBackpressure() {
    StepVerifier.create(Flux.range(1, 10), 3)  // 처음 3개만 request
        .expectNext(1, 2, 3)
        .thenRequest(5)               // 추가 5개 요청
        .expectNext(4, 5, 6, 7, 8)
        .thenCancel()                 // 취소
        .verify();
}
```

---

## 📊 성능 비교

```
Backpressure 전략별 특성 비교:

전략                 | 메모리 사용     | 데이터 손실 | 소비자 지연 시 동작
────────────────────┼──────────────┼──────────┼─────────────────
request(n) 명시 제어  | O(n)          | 없음      | 자연스럽게 조율
onBackpressureBuffer | 버퍼 크기까지  | 없음      | 버퍼 초과 시 OOM 위험
onBackpressureDrop   | 최소           | 있음      | 초과 항목 버림
onBackpressureLatest | 최소           | 있음      | 최신 항목만 유지
onBackpressureError  | 최소           | 없음      | OverflowException 발생

신호 처리 비용:
  onNext() 호출 자체: 나노초 수준 (연산자 체인 통과)
  연산자 체인 깊이: 각 연산자마다 Subscriber 래핑 → 메모리 오버헤드
  → 연산자를 합칠 수 있으면 합치는 것이 유리
    (map 여러 개 vs map 하나에 합성 함수)

Reactor의 Fusion 최적화:
  인접한 연산자를 합쳐 신호 전달 오버헤드 감소
  Fuseable 인터페이스를 구현한 연산자끼리 자동으로 합성
  → 로그에 [Fuseable] 표시되는 연산자가 Fusion 적용됨
```

---

## ⚖️ 트레이드오프

```
Reactive Streams 스펙 준수의 트레이드오프:

엄격한 계약(request(n)이 있어야 onNext())의 장점:
  ① 메모리 안전: 소비자가 처리할 수 있는 만큼만 생산
  ② 취소 가능: cancel()로 불필요한 자원 해제
  ③ 표준화: 라이브러리 간 호환

단점:
  ① 구현 복잡도: Subscriber 직접 구현 시 스펙 엄수 필요
  ② 직관성 부족: "subscribe() 전까지 아무것도 실행 안 됨"
     → 초보자 가장 많이 실수하는 부분
  ③ 디버깅 어려움: 신호 흐름이 보이지 않음 (log() 연산자 활용 필요)

실무 권장:
  Subscriber 직접 구현 지양 → Reactor 제공 연산자 활용
  Backpressure 제어 필요 시 → BaseSubscriber 상속
  테스트: StepVerifier로 신호 흐름 검증 필수
  디버깅: Hooks.onOperatorDebug() 또는 checkpoint() 활용
```

---

## 📌 핵심 정리

```
Reactive Streams 스펙 신호 흐름:

구독 설정 (아래 → 위):
  subscribe() → 상위 Publisher 연쇄 subscribe
  → 각 단계 onSubscribe() 호출 → 최종 Subscriber까지

데이터 흐름 시작 (아래 → 위):
  Subscriber: subscription.request(n)
  → 연산자 체인 거슬러 올라가 최상위 Publisher에 전달
  → Publisher: request(n)개 이하의 onNext() 발행 시작

데이터 흐름 (위 → 아래):
  Publisher: onNext(item) → 연산자 체인 거쳐 → 최종 Subscriber

종료 신호 (위 → 아래):
  onComplete() 또는 onError() → 체인 거쳐 최종 Subscriber
  이후 onNext() 없음 (스펙 보장)

핵심 계약:
  request(n) 없으면 onNext() 없음 → subscribe() 필수
  onError/onComplete 이후 onNext 없음 → 스트림 종료
  map 안의 예외 → onError 신호로 변환 (try-catch 불필요)
```

---

## 🤔 생각해볼 문제

**Q1.** 동일한 Mono를 여러 번 `subscribe()`하면 어떻게 되나요? HTTP 요청이 여러 번 발생할까요?

<details>
<summary>해설 보기</summary>

`Mono`는 Cold Publisher이므로, `subscribe()`를 호출할 때마다 완전히 새로운 스트림이 시작됩니다.

```java
Mono<String> httpRequest = webClient.get()
    .uri("/api/data")
    .retrieve()
    .bodyToMono(String.class);

httpRequest.subscribe(s -> log.info("구독자 A: {}", s));  // HTTP 요청 1번
httpRequest.subscribe(s -> log.info("구독자 B: {}", s));  // HTTP 요청 또 1번!
```

이것이 실무에서 중복 HTTP 요청이 발생하는 흔한 원인입니다. 해결책은 `Mono.cache()`를 사용하여 첫 번째 결과를 캐싱하거나, `share()`로 Hot Publisher로 변환하는 것입니다.

```java
// 캐싱: 첫 구독 결과를 재사용
Mono<String> cached = webClient.get()
    .uri("/api/data")
    .retrieve()
    .bodyToMono(String.class)
    .cache();  // 한 번만 HTTP 요청

cached.subscribe(s -> log.info("A: {}", s));  // HTTP 요청 발생
cached.subscribe(s -> log.info("B: {}", s));  // 캐시된 결과 사용
```

WebFlux 컨트롤러에서 같은 Mono를 두 번 `subscribe()`하는 경우가 없도록 주의해야 합니다. `Mono.zip()`이나 `flatMap()` 체인으로 한 번만 구독하도록 파이프라인을 구성하는 것이 올바른 패턴입니다.

</details>

---

**Q2.** `flatMap` 내부에서 `RuntimeException`이 발생하면 Reactive Streams 스펙상 어떻게 처리되나요? `map`과 다른가요?

<details>
<summary>해설 보기</summary>

`map`과 `flatMap` 모두 내부 예외를 `onError` 신호로 변환합니다. 하지만 차이가 있습니다.

**`map`의 경우:**
```java
Flux.range(1, 3)
    .map(i -> {
        if (i == 2) throw new RuntimeException("map 에러");
        return i;
    })
    .subscribe(
        i -> log.info("값: {}", i),
        e -> log.error("에러: {}", e.getMessage())
    );
// 출력: 값: 1, 에러: map 에러
// i == 2에서 예외 → onError → 스트림 종료 → i == 3은 처리 안 됨
```

**`flatMap`의 경우:**
- `flatMap`의 **함수 자체**에서 예외: `map`과 동일하게 onError
- `flatMap`이 반환한 **내부 Publisher에서** onError: 기본적으로 외부 스트림의 onError로 전파

```java
Flux.range(1, 3)
    .flatMap(i -> {
        if (i == 2) return Mono.error(new RuntimeException("내부 에러"));
        return Mono.just(i);
    })
    .subscribe(
        v -> log.info("값: {}", v),
        e -> log.error("에러: {}", e.getMessage())
    );
// 출력: 값: 1, 에러: 내부 에러
// 내부 Publisher의 onError → 외부 Flux의 onError로 전파
```

에러를 격리하고 싶으면 `flatMap` 내에서 `onErrorReturn`으로 처리:
```java
.flatMap(i -> callApi(i).onErrorReturn("기본값"))
// 개별 호출 에러가 전체 스트림을 종료하지 않음
```

</details>

---

**Q3.** Reactor의 `checkpoint()`는 어떤 원리로 디버깅을 돕나요? `Hooks.onOperatorDebug()`와의 차이는 무엇인가요?

<details>
<summary>해설 보기</summary>

Reactor의 기본 스택 트레이스에는 실제 오류 발생 지점이 보이지 않습니다. 연산자 체인이 람다와 익명 클래스로 구성되어 있어 스택 트레이스가 Reactor 내부 클래스로 가득 찹니다.

**`checkpoint()`:**
```java
Flux.range(1, 3)
    .map(i -> i / 0)
    .checkpoint("division-checkpoint")  // 여기서 에러 발생 시 표시
    .subscribe(System.out::println, System.err::println);
// 에러 스택에: Assembly trace from checkpoint [division-checkpoint]
// → 어느 연산자에서 문제 발생했는지 알 수 있음
```
- 특정 지점에 마커를 추가
- 해당 체크포인트를 통과하는 에러에 위치 정보 추가
- **성능 영향 없음** (에러 발생 시에만 스택 캡처)

**`Hooks.onOperatorDebug()`:**
```java
Hooks.onOperatorDebug();  // 앱 시작 시 한 번 설정
// 이후 모든 연산자에 스택 캡처 추가
```
- 모든 연산자 생성 시점의 스택 트레이스 캡처
- 에러 발생 시 전체 조립 과정 추적 가능
- **성능 영향 있음** (모든 연산자마다 스택 캡처) → 개발/테스트 환경에서만

**실용적 선택:**
- 개발 환경: `Hooks.onOperatorDebug()` 전역 활성화
- 프로덕션: `checkpoint("설명")` 으로 중요 지점만 표시
- Spring Boot: `ReactorDebugAgent.init()` (Java Agent 방식, 성능 영향 최소)

</details>

---

<div align="center">

**[⬅️ 이전: WebFlux vs Spring MVC](./05-webflux-vs-mvc.md)** | **[홈으로 🏠](../README.md)**

</div>
