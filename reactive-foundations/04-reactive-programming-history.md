# Reactive Programming의 탄생 — 콜백에서 스트림으로

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 비동기 콜백(Callback) 방식은 어떤 문제를 만들어 내는가?
- `Future`와 `CompletableFuture`가 콜백보다 나은 점과 여전히 부족한 점은 무엇인가?
- Reactive Streams 스펙이 표준화한 핵심 계약(Contract)은 무엇인가?
- `Publisher`, `Subscriber`, `Subscription`, `Processor` 4개의 인터페이스는 어떻게 협력하는가?
- Java 9의 `Flow` API와 Project Reactor의 `Mono`/`Flux`는 어떤 관계인가?

---

## 🔍 왜 이 개념이 WebFlux에서 중요한가

Spring WebFlux는 Project Reactor를 기반으로 하고, Project Reactor는 Reactive Streams 스펙을 구현합니다. WebFlux 코드에서 매일 마주치는 `Mono`, `Flux`, `subscribe()`, `onNext()`, `request(n)` — 이 모든 것이 Reactive Streams 스펙에서 나왔습니다.

이 스펙이 왜 탄생했는지, 기존 방식의 어떤 문제를 해결하려 했는지를 모르면 `Backpressure`가 왜 필요한지, `subscribe()` 전에 왜 아무것도 실행되지 않는지, `Flux`가 `Iterable`과 어떻게 다른지를 표면적으로만 암기하게 됩니다.

---

## 😱 흔한 실수 (Before — 콜백과 Future의 한계를 모를 때)

```
콜백 지옥(Callback Hell) 예시:

// 사용자 조회 → 주문 조회 → 결제 확인 → 배송 조회
userService.findById(userId, user -> {
    if (user == null) {
        handleError(new UserNotFoundException());
        return;
    }
    orderService.findByUser(user, order -> {
        if (order == null) {
            handleError(new OrderNotFoundException());
            return;
        }
        paymentService.verify(order, payment -> {
            if (!payment.isVerified()) {
                handleError(new PaymentException());
                return;
            }
            shippingService.track(order, tracking -> {
                // 드디어 결과
                respond(new OrderStatus(order, payment, tracking));
                // 에러 처리 누락 가능성
                // 이 중첩 안에서 예외 발생 시 어디서 잡나?
            });
        });
    });
});

문제:
  ① 들여쓰기가 깊어져 코드 가독성 심각 저하
  ② 에러 처리를 각 콜백마다 별도로 해야 함 (누락 위험)
  ③ 실행 순서를 코드 읽기로 파악하기 어려움
  ④ 취소(Cancel) 메커니즘 없음 — 중간에 멈출 방법 없음
  ⑤ Backpressure 없음 — 생산자가 소비자보다 빠르면 메모리 폭발

CompletableFuture로 개선했지만:
  userService.findById(userId)
      .thenCompose(user -> orderService.findByUser(user))
      .thenCompose(order -> paymentService.verify(order))
      .thenCompose(payment -> shippingService.track(payment))
      .exceptionally(e -> handleError(e));

  여전히 부족한 것:
  ① 0개 또는 1개 값만 처리 (단일 결과)
  ② 여러 값의 스트림 처리 불가
  ③ Backpressure 없음
  ④ 여러 비동기 라이브러리가 자체 Future 구현 → 호환 불가
```

---

## ✨ 올바른 접근 (After — Reactive Streams 스펙 기반)

```
Reactive Streams 스펙이 해결한 것:

1. 스트림 처리 (0개, 1개, 다수 값 모두 지원)
   Publisher<T>: 0~N개 값을 비동기로 발행

2. Backpressure (소비자가 생산자 속도를 제어)
   Subscription.request(n): "n개만 주세요"
   → 생산자는 요청받은 수만큼만 발행

3. 에러 처리 표준화
   onError(Throwable): 스트림에서 에러 발생 시 단일 지점 처리

4. 완료 신호
   onComplete(): 스트림 완료

5. 라이브러리 간 호환
   RxJava, Reactor, Akka Streams, Vert.x 모두 동일 스펙 구현
   → 라이브러리 교체해도 파이프라인 로직 재사용 가능

Project Reactor 적용:
  // 콜백 지옥 대신:
  userService.findById(userId)         // Mono<User>
      .flatMap(orderService::findByUser)   // Mono<Order>
      .flatMap(paymentService::verify)     // Mono<Payment>
      .flatMap(shippingService::track)     // Mono<Tracking>
      .subscribe(
          result -> respond(result),
          error  -> handleError(error)  // 에러 처리 한 곳
      );

  ① 체인이 선형으로 읽힘
  ② 에러 처리 단일 지점
  ③ Backpressure 내장
  ④ 취소(Disposable.dispose()) 지원
```

---

## 🔬 내부 동작 원리

### 1. 비동기 처리 역사 — 콜백에서 Reactive까지

```
1세대: 콜백 (Callback)

  비동기 결과를 함수 인자로 전달
  
  문제: Callback Hell, 에러 처리 분산, 취소 불가

──────────────────────────────────────

2세대: Future / Promise (Java 1.5)

  Future<String> future = executor.submit(() -> fetch());
  String result = future.get();  // ← 블로킹!
  
  문제: get()이 블로킹 → 논블로킹의 이점 소멸

──────────────────────────────────────

3세대: CompletableFuture (Java 8)

  CompletableFuture.supplyAsync(() -> fetch())
      .thenApply(s -> transform(s))
      .thenAccept(s -> respond(s));
  
  개선: 체이닝 가능, 비블로킹 합성
  한계: 단일 값만, Backpressure 없음, 라이브러리 파편화

──────────────────────────────────────

4세대: Reactive Streams 스펙 (2013~, Java 9에 Flow API로 표준화)

  Publisher<T> → Subscriber<T> 표준 계약
  + Backpressure (Subscription.request(n))
  + 에러/완료 신호 표준화
  + 다수 값(스트림) 처리

──────────────────────────────────────

구현체:
  RxJava (Netflix, 2013)     → Observable, Single, Flowable
  Project Reactor (Pivotal)  → Mono, Flux (Spring WebFlux 기반)
  Akka Streams (Lightbend)   → Source, Flow, Sink
  SmallRye Mutiny (Quarkus)  → Uni, Multi
  → 모두 Reactive Streams 스펙 구현 → 상호 호환
```

### 2. Reactive Streams 4개 인터페이스 완전 분해

```java
// Reactive Streams 스펙 (java.util.concurrent.Flow — Java 9)

// 1. Publisher<T>: 데이터 생산자
// "Subscriber가 구독하면 데이터를 발행하겠다"
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> subscriber);
    // subscribe() 호출 시:
    //   1. Subscription 생성
    //   2. subscriber.onSubscribe(subscription) 호출
    //   3. 이후 subscriber.request(n)이 와야만 데이터 발행 시작
}

// 2. Subscriber<T>: 데이터 소비자
// "Publisher로부터 데이터를 받겠다"
public interface Subscriber<T> {
    void onSubscribe(Subscription s);
    // 구독 시작 시 호출
    // 이 시점에 s.request(n)을 호출해야 데이터 흐름 시작
    // → "n개 보내줘"

    void onNext(T t);
    // 데이터 하나 수신 시 호출
    // onSubscribe 이후, request(n)에 응답으로만 호출됨
    // → n번 호출 후 추가 request 없으면 발행 중단

    void onError(Throwable t);
    // 에러 발생 시 호출 (이후 onNext, onComplete 호출 없음)

    void onComplete();
    // 스트림 완료 시 호출 (이후 onNext 호출 없음)
}

// 3. Subscription: Publisher ↔ Subscriber 계약
// "이 구독 관계에서 데이터 흐름을 제어한다"
public interface Subscription {
    void request(long n);
    // Subscriber가 "n개 더 줘"라고 요청
    // n개 이하의 onNext()가 이후 호출됨
    // → Backpressure의 핵심!

    void cancel();
    // 구독 취소 — Publisher는 데이터 발행 중단
}

// 4. Processor<T, R>: Publisher + Subscriber 동시 구현
// 중간 처리 단계 (map, filter 등의 연산자)
public interface Processor<T, R>
    extends Subscriber<T>, Publisher<R> {}
```

### 3. Reactive Streams 신호 흐름 — 전체 라이프사이클

```
subscribe() 호출부터 완료까지의 신호 흐름:

Publisher                    Subscription                 Subscriber
    │                             │                            │
    │◄─────────── subscribe() ────────────────────────────────│
    │                             │                            │
    │ ──── onSubscribe(sub) ──────────────────────────────────►│
    │                             │                            │
    │                             │◄──── request(3) ──────────│
    │ ◄─────────── request(3) ────│                            │
    │                             │                            │
    │ ──────── onNext(item1) ─────────────────────────────────►│
    │ ──────── onNext(item2) ─────────────────────────────────►│
    │ ──────── onNext(item3) ─────────────────────────────────►│
    │  (3개 발행 후 대기 — 추가 request 없으면 멈춤)              │
    │                             │                            │
    │                             │◄──── request(2) ──────────│
    │ ◄─────────── request(2) ────│                            │
    │                             │                            │
    │ ──────── onNext(item4) ─────────────────────────────────►│
    │ ──────── onNext(item5) ─────────────────────────────────►│
    │                             │                            │
    │ ──────── onComplete() ──────────────────────────────────►│
    │  (더 이상 발행할 데이터 없음)                              │

핵심 규칙 (Reactive Streams 스펙):
  1. onSubscribe()는 반드시 1번만 호출
  2. onNext()는 request(n)에 응답으로만, n번 이하만 호출
  3. onError() 또는 onComplete() 이후 onNext() 절대 없음
  4. cancel() 후 Publisher는 발행 중단 (즉시 보장 없음)
  5. request(n)의 n은 Long.MAX_VALUE 가능 (unbounded)

Backpressure 실체:
  소비자가 처리할 수 있는 만큼만 request(n)으로 요청
  생산자는 n개 이하만 발행
  → 소비자가 생산자를 제어 (pull 방식의 느낌)
  → 메모리 폭발 방지
```

### 4. Iterable(Pull) vs Reactive Streams(Push + Pull 혼합)

```
전통적 Iterable (동기, Pull 방식):

  List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
  for (Integer item : list) {
      // 소비자가 iterator.next()로 하나씩 당김 (Pull)
      // 소비자가 준비될 때 가져옴 → 생산자는 기다림
      process(item);
  }
  
  특징:
    소비자가 생산 속도를 자연스럽게 제어
    하지만 모든 데이터가 메모리에 있어야 함
    비동기 불가 (동기 블로킹)

────────────────────────────────────────

Reactive Streams (비동기, Push + Pull 혼합):

  // Publisher가 준비되는 대로 push
  // 하지만 request(n)으로 소비자가 속도 제어 (Pull 측면)
  
  Flux.range(1, 1_000_000)  // 백만 개지만 메모리에 미리 올리지 않음
      .map(i -> process(i))
      .subscribe(new BaseSubscriber<Integer>() {
          @Override
          protected void hookOnSubscribe(Subscription subscription) {
              request(10);  // 처음에 10개만 요청 (Pull)
          }
          
          @Override
          protected void hookOnNext(Integer value) {
              doSomething(value);
              // 처리 완료 후 추가 요청
              if (shouldContinue()) {
                  request(10);  // 10개 더 요청
              }
          }
      });
  
  특징:
    무한 스트림도 처리 가능 (메모리 O(10) 수준 유지)
    비동기 처리 (이벤트 루프와 통합)
    생산자 속도 ≠ 소비자 속도 → Backpressure로 조율

────────────────────────────────────────

결론:
  Iterable: "내가 필요할 때 가져오겠다" (소비자 주도, 동기)
  Reactive:  "생산자가 준비되면 주되, 내가 요청한 수만큼만" (협력, 비동기)
```

### 5. Java 9 Flow API vs Project Reactor

```
Java 9 Flow API (java.util.concurrent.Flow):
  표준 인터페이스 정의만 제공
  Flow.Publisher, Flow.Subscriber, Flow.Subscription, Flow.Processor
  → 실제 구현 없음, 스펙만 있음

Project Reactor:
  Reactive Streams 스펙 + 실제 구현 + 연산자 풍부
  Mono<T>: 0~1개 값 (Flow.Publisher 구현)
  Flux<T>: 0~N개 값 (Flow.Publisher 구현)
  
  + 수백 개의 연산자 (map, flatMap, filter, zip, merge ...)
  + 스케줄러 통합 (Schedulers.parallel(), boundedElastic() ...)
  + 테스트 지원 (StepVerifier, VirtualTimeScheduler)
  + Netty 통합 (Spring WebFlux)

호환성:
  Reactor Flux → RxJava Flowable 변환 가능 (호환 브릿지 존재)
  모두 Reactive Streams 스펙을 구현하므로 상호 변환 가능

Spring WebFlux의 선택:
  Project Reactor를 기본 구현체로 채택
  → Spring Boot, Spring Data R2DBC, Spring Security Reactive 모두 Reactor 기반
  → RxJava도 지원하지만 Reactor가 Spring 에코시스템의 표준
```

---

## 💻 실전 코드

### 실험 1: Reactive Streams 스펙 직접 구현으로 이해

```java
// Reactive Streams 인터페이스를 직접 구현하여 내부 동작 이해
public class NumberPublisher implements Publisher<Integer> {
    private final int count;

    public NumberPublisher(int count) {
        this.count = count;
    }

    @Override
    public void subscribe(Subscriber<? super Integer> subscriber) {
        subscriber.onSubscribe(new NumberSubscription(subscriber, count));
    }
}

public class NumberSubscription implements Subscription {
    private final Subscriber<? super Integer> subscriber;
    private final int max;
    private int current = 0;
    private boolean cancelled = false;

    // ...

    @Override
    public void request(long n) {
        if (cancelled) return;
        long toEmit = Math.min(n, max - current);
        for (long i = 0; i < toEmit; i++) {
            if (cancelled) return;
            subscriber.onNext(++current);
        }
        if (current >= max) {
            subscriber.onComplete();
        }
    }

    @Override
    public void cancel() {
        cancelled = true;
    }
}

// 사용
Publisher<Integer> publisher = new NumberPublisher(10);
publisher.subscribe(new Subscriber<Integer>() {
    private Subscription sub;

    @Override
    public void onSubscribe(Subscription s) {
        this.sub = s;
        s.request(3);  // 처음에 3개만 요청
    }

    @Override
    public void onNext(Integer item) {
        log.info("수신: {}", item);
        if (item == 3) sub.request(7);  // 3 받은 후 7개 더 요청
    }

    @Override
    public void onError(Throwable t) { log.error("에러", t); }

    @Override
    public void onComplete() { log.info("완료"); }
});
// 출력: 수신: 1, 수신: 2, 수신: 3, (잠시 후) 수신: 4 ~ 10, 완료
```

### 실험 2: Backpressure 없을 때 vs 있을 때 비교

```java
// Backpressure 없음 — 생산자가 소비자보다 훨씬 빠를 때
Flux.range(1, 1_000_000)
    .doOnNext(i -> {
        if (i % 100_000 == 0) log.info("생산: {}", i);
    })
    .publishOn(Schedulers.boundedElastic())  // 소비자를 다른 스레드로
    .subscribe(i -> {
        // 소비자가 느림 (처리 시간 가정)
        if (i % 100_000 == 0) log.info("소비: {}", i);
    });
// → 내부 버퍼에 최대 256개(기본 prefetch)까지 쌓임
// → 기본 BUFFER 전략으로 일정 수준까지 완충

// Backpressure 명시적 제어
Flux.range(1, 1_000_000)
    .log()  // 신호 로깅 (request/onNext/onComplete)
    .subscribe(new BaseSubscriber<Integer>() {
        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            log.info("구독 시작 — 10개 요청");
            request(10);  // 처음에 10개만 요청
        }

        @Override
        protected void hookOnNext(Integer value) {
            log.info("처리: {}", value);
            // 처리 후 1개씩 추가 요청 (완전한 Backpressure 제어)
            request(1);
        }

        @Override
        protected void hookOnComplete() {
            log.info("완료");
        }
    });
// 로그에서 request(10), onNext×10, request(1)×990000... 확인 가능
```

### 실험 3: 라이브러리 간 Reactive Streams 호환

```java
// Project Reactor Flux → RxJava Flowable 변환 (호환 확인)
Flux<Integer> reactorFlux = Flux.range(1, 5);

// RxJava Flowable로 변환 (동일 Reactive Streams 스펙 구현)
Flowable<Integer> rxFlowable = Flowable.fromPublisher(reactorFlux);
rxFlowable.subscribe(i -> log.info("RxJava 수신: {}", i));

// RxJava → Reactor 역방향 변환
Flowable<String> rxSource = Flowable.just("a", "b", "c");
Flux<String> reactorFromRx = Flux.from(rxSource);
reactorFromRx.subscribe(s -> log.info("Reactor 수신: {}", s));

// → 동일 스펙 구현이므로 변환 비용 최소 (래퍼만 추가)
```

---

## 📊 성능 비교

```
비동기 처리 방식별 특성 비교:

방식                  | 스트림 처리 | Backpressure | 에러처리 | 취소   | 라이브러리 호환
─────────────────────┼───────────┼─────────────┼─────────┼───────┼─────────────
Callback             | 불가       | 없음         | 분산     | 없음   | -
CompletableFuture    | 불가(1개)  | 없음         | exceptionally | thenApply 체인 중단 | 없음
Reactive Streams     | 가능(0~N개)| 내장         | onError  | cancel() | 있음(표준)

Backpressure 효과 (생산 100만/초, 소비 10만/초):

방식               | 메모리 사용       | 결과
──────────────────┼─────────────────┼─────────────────
Backpressure 없음  | 급격히 증가 → OOM | OutOfMemoryError 위험
Backpressure 있음  | O(request 크기)   | 소비자 속도에 맞게 조율
  (request(100))   | 최대 수백 KB     | 안정적 처리

Project Reactor 연산자별 Backpressure 처리:
  Flux.create() → onBackpressureBuffer() 기본
  Flux.generate() → Backpressure 완전 지원
  Flux.range() → Backpressure 지원
  Flux.interval() → BUFFER 기본 (시간 기반, 소비자가 따라오지 못할 수 있음)
```

---

## ⚖️ 트레이드오프

```
Reactive Streams 기반 Reactive Programming:

장점:
  ① 표준 스펙으로 라이브러리 간 호환
  ② Backpressure 내장으로 메모리 안전
  ③ 에러 처리 단일 지점 (onError)
  ④ 취소 지원 (cancel() → 자원 정리)
  ⑤ 무한 스트림 처리 가능

단점:
  ① 학습 곡선 가파름
     - 새로운 사고방식 필요 (데이터 흐름 vs 명령형 순서)
     - 연산자 수백 개 (map vs flatMap vs concatMap ...)
  ② 디버깅 어려움
     - 스택 트레이스가 Reactor 내부 클래스로 가득
     - 실제 오류 지점 파악 어려움
     - 해결: Hooks.onOperatorDebug() 또는 checkpoint()
  ③ 모든 라이브러리가 Reactive 지원하지 않음
     - JDBC, JPA → R2DBC로 교체 필요
     - Reactive 미지원 라이브러리 → Schedulers.boundedElastic() 래핑

언제 선택하는가:
  I/O 집약적, 높은 동시성, 스트리밍 데이터 → Reactive 적합
  단순 CRUD, 팀 학습 비용 > 장점, 블로킹 라이브러리 의존 → MVC 선호
```

---

## 📌 핵심 정리

```
Reactive Streams 탄생 배경과 스펙:

탄생 이유:
  콜백 지옥 → CompletableFuture(단일 값, Backpressure 없음) → 부족
  → 표준 스트림 처리 + Backpressure + 에러 처리 스펙 필요

4개 인터페이스:
  Publisher<T>:     데이터 생산자 (subscribe() 제공)
  Subscriber<T>:    데이터 소비자 (onSubscribe/onNext/onError/onComplete)
  Subscription:     생산-소비 계약 (request(n)/cancel())
  Processor<T,R>:   중간 처리 단계 (Publisher + Subscriber)

핵심 계약:
  subscribe() 호출 → onSubscribe() → request(n) → onNext()×n 이하
  데이터 흐름은 request(n)으로 시작 → Backpressure의 실체

Project Reactor:
  Mono<T>:  0~1개 값 (Reactive Streams Publisher 구현)
  Flux<T>:  0~N개 값 (Reactive Streams Publisher 구현)
  subscribe() 전까지 파이프라인 실행 없음 (Cold Publisher)
  → WebFlux의 기반
```

---

## 🤔 생각해볼 문제

**Q1.** `Flux.create()`와 `Flux.generate()`의 차이는 Backpressure 관점에서 무엇인가요?

<details>
<summary>해설 보기</summary>

**`Flux.create()`:**
- 콜백에서 자유롭게 `sink.next()`를 호출할 수 있음
- Backpressure를 자동으로 처리하지 않음 (호출자가 `sink.onBackpressureBuffer()` 등을 명시해야 함)
- 외부 이벤트 소스(리스너, 콜백 기반 API)를 Flux로 래핑할 때 적합

```java
Flux.create(sink -> {
    eventEmitter.on("data", data -> sink.next(data));
    eventEmitter.on("end", () -> sink.complete());
    eventEmitter.on("error", e -> sink.error(e));
});
```

**`Flux.generate()`:**
- 상태 기반 동기 생성기, 한 번에 하나씩 발행
- `request(n)`이 들어올 때마다 n번 호출 → 완전한 Backpressure 지원
- Pull 방식에 가장 적합

```java
Flux.generate(
    () -> 0,  // 초기 상태
    (state, sink) -> {
        sink.next(state);
        if (state == 100) sink.complete();
        return state + 1;  // 다음 상태
    }
);
```

Backpressure 관점:
- `generate()`는 소비자가 `request(1)`할 때마다 하나씩 생성 → 완전 제어
- `create()`는 외부 소스 속도에 따라 발행 → 별도 Backpressure 전략 필요

</details>

---

**Q2.** `Long.MAX_VALUE`를 `request()`에 전달하면 Backpressure가 사실상 없어지는 건가요? Reactor에서 이를 언제 허용하나요?

<details>
<summary>해설 보기</summary>

`request(Long.MAX_VALUE)`는 Reactive Streams 스펙에서 "unbounded request"로 정의됩니다. 소비자가 생산자에게 "제한 없이 보내도 됩니다"라고 말하는 것입니다.

Backpressure 관점에서는 사실상 비활성화와 같습니다. 하지만 Reactor에서 `subscribe()`를 람다로 호출할 때 기본적으로 `Long.MAX_VALUE`를 request합니다:

```java
flux.subscribe(item -> process(item));
// 내부적으로 Long.MAX_VALUE request → unbounded

// Backpressure 제어가 필요하면:
flux.subscribe(new BaseSubscriber<Integer>() {
    @Override
    protected void hookOnSubscribe(Subscription subscription) {
        request(10);  // 명시적 제어
    }
});
```

`Long.MAX_VALUE`가 적절한 경우:
- 소비자가 생산자보다 항상 빠른 경우 (CPU 처리가 빠를 때)
- `Flux.just()`, `Flux.range()` 같이 메모리에 이미 있는 데이터
- 처리량이 충분히 제어되는 파이프라인 (`publishOn`으로 스레드 분리 후 큐 제한)

`Long.MAX_VALUE`가 위험한 경우:
- 외부 소스가 초당 수십만 건 발행 가능한 경우
- 소비자 처리가 느린 경우 → 내부 버퍼가 무한 성장 → OOM

실무에서 대부분의 WebFlux 코드는 `Long.MAX_VALUE`를 사용하되, 느린 소비자 시나리오에서는 `onBackpressureBuffer(maxSize)` 또는 `onBackpressureDrop()`으로 보호합니다.

</details>

---

**Q3.** `cancel()`을 호출하면 Publisher는 즉시 발행을 멈추는 것이 보장되나요?

<details>
<summary>해설 보기</summary>

Reactive Streams 스펙에서 `cancel()`은 즉시 중단을 보장하지 않습니다. 스펙에 따르면:

> "Publisher MAY emit further items after `cancel()` has been called" (cancel() 이후에도 Publisher가 추가 항목을 발행할 수 있음)

단, 합리적인 시간 내에 중단해야 한다는 암묵적 기대가 있습니다.

실제 동작:
- `Flux.range()`나 `Flux.just()`: `cancel()` 시 루프 조건 확인 → 즉시에 가깝게 중단
- 외부 I/O 기반 Publisher: 진행 중인 I/O는 완료 후 중단 (즉시 불가)
- `Flux.interval()`: 타이머 취소 후 중단

WebFlux에서의 취소:
```java
Disposable subscription = flux.subscribe(item -> process(item));
// 나중에 취소
subscription.dispose();  // cancel() 내부 호출

// 또는 HTTP 요청이 클라이언트 측에서 중단되면:
// WebFlux가 자동으로 파이프라인 cancel() 호출
// → 불필요한 DB 쿼리나 외부 API 호출 중단 가능 (R2DBC, WebClient 지원)
```

이것이 Reactive의 자원 효율성의 또 다른 측면입니다: 클라이언트가 연결을 끊으면, 진행 중인 파이프라인도 취소되어 불필요한 자원 사용을 방지할 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: 논블로킹 I/O의 원리](./03-nonblocking-io-epoll.md)** | **[홈으로 🏠](../README.md)** | **[다음: WebFlux vs Spring MVC ➡️](./05-webflux-vs-mvc.md)**

</div>
