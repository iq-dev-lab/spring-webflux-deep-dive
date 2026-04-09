# Cold vs Hot Publisher — 구독 시점의 차이

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Cold Publisher와 Hot Publisher는 구독 시점에 어떻게 다르게 동작하는가?
- `publish().refCount()`와 `publish().autoConnect()`는 무엇이 다른가?
- `Sinks.Many`와 `Sinks.One`은 언제 사용하는가?
- 실시간 이벤트 스트림(SSE, WebSocket 브로드캐스트)에서 Hot Publisher가 왜 필요한가?
- `share()`는 내부적으로 어떻게 동작하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

모든 `Mono`와 `Flux`가 Cold Publisher라고 생각하면 함정에 빠집니다. SSE 엔드포인트에서 여러 클라이언트가 같은 실시간 데이터를 받아야 할 때, 각 구독자마다 독립 데이터 소스를 만들면 외부 API나 DB에 N배의 부하가 걸립니다. Hot Publisher로 하나의 소스를 여러 구독자가 공유하면 이 문제가 해결됩니다.

반대로 Cold Publisher가 필요한 곳에 Hot Publisher를 쓰면 늦게 구독한 클라이언트가 초기 데이터를 받지 못하는 문제가 생깁니다.

---

## 😱 흔한 실수 (Before — Cold/Hot 차이를 모를 때)

```
실수 1: SSE에서 Cold Publisher로 각 구독자마다 독립 소스

  @GetMapping(value = "/stream/prices", produces = TEXT_EVENT_STREAM_VALUE)
  public Flux<StockPrice> streamPrices() {
      return stockPriceService.getPriceStream();
      // getPriceStream() = Flux.interval + 외부 API 호출 (Cold)
      // 클라이언트 100명 → getPriceStream() 100번 호출
      // → 외부 API 100개 연결, 100배 부하
  }

실수 2: 동일 Mono를 여러 번 구독

  Mono<Config> configMono = Mono.fromCallable(() -> loadConfigFromDB());
  configMono.subscribe(c -> useForA(c));   // DB 쿼리 1
  configMono.subscribe(c -> useForB(c));   // DB 쿼리 2 (Cold Publisher!)
  configMono.subscribe(c -> useForC(c));   // DB 쿼리 3

  // 원하는 것: DB 1번 조회 후 3곳에 공유
  // 해결: configMono.cache() 또는 share()

실수 3: Hot Publisher 구독 타이밍 문제

  Flux<String> hot = hotSource.asFlux();
  // ... 몇 가지 이벤트 발생 ...
  hot.subscribe(s -> log.info("나중에 구독: {}", s));
  // 구독 전에 발생한 이벤트 수신 불가
  // 이를 예상하지 못하면 "데이터가 빠졌다" 버그 발생
```

---

## ✨ 올바른 접근 (After — 목적에 맞는 Publisher 선택)

```
Cold Publisher가 적합한 경우:
  - 각 구독자가 처음부터 모든 데이터 필요 (API 응답, DB 쿼리 결과)
  - 구독자마다 독립적인 데이터 필요
  - 기본값: Mono.just(), Flux.range(), webClient... 등

Hot Publisher가 적합한 경우:
  - 여러 구독자가 동일한 실시간 스트림 공유 (주식 시세, 알림)
  - SSE/WebSocket 브로드캐스트
  - 이벤트 버스 (내부 이벤트 전파)

Cold → Hot 변환 방법:
  flux.publish().refCount(1):  1명 이상 구독 시 시작, 0명 되면 중단
  flux.publish().autoConnect(): 영구적으로 시작 (구독자 없어도 유지)
  flux.share():                 = publish().refCount(1)의 단축어
  flux.cache():                 Cold → 재사용 가능한 캐시 (완료 시까지)
```

---

## 🔬 내부 동작 원리

### 1. Cold Publisher 상세 동작

```
Cold Publisher 특징:
  - subscribe()마다 새로운 독립 데이터 스트림 생성
  - 각 구독자가 처음부터 끝까지 동일한 데이터 수신
  - 데이터 소스가 "구독 시 생성"됨

예시:
  Flux<Integer> cold = Flux.range(1, 5);

  구독자 A: subscribe() → 1, 2, 3, 4, 5 수신
  구독자 B: subscribe() → 1, 2, 3, 4, 5 수신 (독립적)

  내부 구조:
    Flux.range(1, 5)가 subscribe() 수신 시 RangeSubscription 생성
    각 subscribe() → 새로운 RangeSubscription → 독립 카운터
    → 두 구독자는 서로 영향 없음

WebClient Mono도 Cold:
  Mono<Response> api = webClient.get().uri("/api").retrieve()
      .bodyToMono(Response.class);
  api.subscribe();  // HTTP 요청 1번
  api.subscribe();  // HTTP 요청 또 1번 (독립!)
```

### 2. Hot Publisher 상세 동작

```
Hot Publisher 특징:
  - 구독 여부와 관계없이 데이터 발행 (외부 이벤트 소스처럼)
  - 구독 시점 이후의 데이터만 수신 가능
  - 여러 구독자가 동일한 스트림 공유

Sinks.Many — Reactor 3.4+의 Hot Publisher 표준:

  // 멀티캐스트 (여러 구독자에게 동일 데이터)
  Sinks.Many<String> sink = Sinks.many().multicast()
      .onBackpressureBuffer();

  Flux<String> hot = sink.asFlux();

  // 구독자 A 등록
  hot.subscribe(s -> log.info("A: {}", s));

  // 이벤트 발행
  sink.tryEmitNext("event-1");   // A: event-1

  // 구독자 B 등록 (이후부터만 수신)
  hot.subscribe(s -> log.info("B: {}", s));

  sink.tryEmitNext("event-2");   // A: event-2, B: event-2
  sink.tryEmitNext("event-3");   // A: event-3, B: event-3

  // A는 event-1, 2, 3 수신
  // B는 event-2, 3만 수신 (event-1 누락)
```

### 3. publish().refCount() — Cold를 Hot으로 변환

```
publish(): Cold Flux를 ConnectableFlux로 변환
  ConnectableFlux<T>: connect() 전까지 발행 시작 안 함

refCount(n): n명 구독 시 자동 connect(), 0명 되면 disconnect

예시:
  Flux<Long> sharedTimer = Flux.interval(Duration.ofSeconds(1))
      .publish()
      .refCount(1);  // 1명 이상 구독 시 타이머 시작

  // 구독자 없음: 타이머 아직 시작 안 함
  sharedTimer.subscribe(t -> log.info("A: {}", t));
  // 1명 → connect() → 타이머 시작
  // A: 0, A: 1, A: 2 ...

  Thread.sleep(3000);
  sharedTimer.subscribe(t -> log.info("B: {}", t));
  // B: 3, 4, 5 ... (A와 동일 타이머 공유)

  // A 구독 취소 → 구독자 1명 (B만 남음)
  // B 구독 취소 → 구독자 0명 → disconnect → 타이머 중단!
  // 다시 구독 시 → 새 타이머 시작 (처음부터)

autoConnect(n):
  n명 구독 시 connect(), 이후 구독자 0명 되어도 유지
  refCount vs autoConnect:
    refCount: 구독자 없으면 소스 중단 (자원 절약)
    autoConnect: 한번 시작하면 영원히 유지 (이벤트 유실 없음)
```

### 4. share() — 가장 간단한 Hot 변환

```
share() = publish().refCount(1)의 단축어

Flux<StockPrice> priceStream = stockApiClient.getPrices()
    .share();  // 첫 구독 시 시작, 0명 되면 중단

// SSE 엔드포인트에서 공유
@GetMapping(value = "/stream/prices", produces = TEXT_EVENT_STREAM_VALUE)
public Flux<StockPrice> streamPrices() {
    return priceStream;  // 모든 클라이언트가 동일 스트림 공유
    // 외부 API는 1개 연결만 유지
}
```

### 5. Mono.cache() — 결과 캐싱

```
cache(): 첫 구독 결과를 캐싱 후 이후 구독에 재사용

Mono<Config> cachedConfig = Mono.fromCallable(() -> loadConfigFromDB())
    .subscribeOn(Schedulers.boundedElastic())
    .cache();  // 첫 구독 후 결과 캐싱

cachedConfig.subscribe(c -> useForA(c));  // DB 쿼리 1번
cachedConfig.subscribe(c -> useForB(c));  // 캐시 반환 (DB 쿼리 없음)
cachedConfig.subscribe(c -> useForC(c));  // 캐시 반환

TTL 지정:
  Mono.fromCallable(() -> loadFromDB())
      .cache(Duration.ofMinutes(5));  // 5분 TTL
```

---

## 💻 실전 코드

### 실험 1: SSE 브로드캐스트 — Hot Publisher 적용

```java
@Service
public class StockPriceService {
    private final Flux<StockPrice> priceStream;

    public StockPriceService(WebClient stockApiClient) {
        // Cold Flux → share()로 Hot 변환
        this.priceStream = stockApiClient.get()
            .uri("/prices/stream")
            .retrieve()
            .bodyToFlux(StockPrice.class)
            .share();  // 모든 구독자가 동일 스트림 공유
    }

    public Flux<StockPrice> getPrices() {
        return priceStream;
    }
}

@RestController
public class StockController {

    @GetMapping(value = "/stream/prices",
                produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<StockPrice> streamPrices() {
        return stockPriceService.getPrices();
        // 100명 접속 → 외부 API는 1개 연결만
        // share() 없이 Cold였다면 100개 연결
    }
}
```

### 실험 2: Sinks로 내부 이벤트 버스 구현

```java
@Component
public class EventBus {
    private final Sinks.Many<ApplicationEvent> sink =
        Sinks.many().multicast().onBackpressureBuffer();

    public void publish(ApplicationEvent event) {
        sink.tryEmitNext(event);
    }

    public Flux<ApplicationEvent> subscribe() {
        return sink.asFlux();
    }

    public <T extends ApplicationEvent> Flux<T> subscribe(Class<T> type) {
        return sink.asFlux()
            .filter(type::isInstance)
            .cast(type);
    }
}

// 사용
@Service
public class OrderService {
    private final EventBus eventBus;

    public Mono<Order> createOrder(OrderRequest req) {
        return orderRepository.save(Order.from(req))
            .doOnNext(order ->
                eventBus.publish(new OrderCreatedEvent(order))
            );
    }
}

@Component
public class NotificationHandler {
    @PostConstruct
    public void init() {
        eventBus.subscribe(OrderCreatedEvent.class)
            .flatMap(event -> notificationService.notify(event))
            .subscribe(null, err -> log.error("알림 실패", err));
    }
}
```

### 실험 3: cache()로 초기화 비용 절감

```java
@Service
public class FeatureFlagService {
    // 애플리케이션 시작 시 한 번만 DB 조회 후 캐싱
    private final Mono<Map<String, Boolean>> flags = Mono.fromCallable(
            () -> flagRepository.findAll()
                .stream()
                .collect(Collectors.toMap(Flag::getName, Flag::isEnabled))
        )
        .subscribeOn(Schedulers.boundedElastic())
        .cache();  // 결과 영구 캐싱

    public Mono<Boolean> isEnabled(String flagName) {
        return flags.map(map -> map.getOrDefault(flagName, false));
        // 매번 DB 쿼리 없이 캐시에서 반환
    }
}
```

---

## 📊 성능 비교

```
Cold vs Hot Publisher 구독자 100명 비교:

시나리오: 외부 API 실시간 스트리밍 (초당 10개 이벤트)

Cold Publisher (기본):
  구독자 100명 → 외부 API 연결 100개
  외부 API 요청: 100 req/s (각 구독자마다 독립)
  메모리: 100개 독립 스트림

Hot Publisher (share()):
  구독자 100명 → 외부 API 연결 1개
  외부 API 요청: 1 req/s
  메모리: 1개 스트림 + 100개 구독자 큐
  → 외부 API 부하 100배 감소

Sinks.Many 멀티캐스트 버퍼 설정:
  Sinks.many().multicast().onBackpressureBuffer()
  → 기본 버퍼: Queues.SMALL_BUFFER_SIZE (256개)
  → 구독자가 느릴 때 버퍼에 쌓임
  → 버퍼 초과 시 FAIL 에러 발생
  
  구독자가 느릴 경우 전략:
    개별 구독자에게 onBackpressureDrop()
    또는 버퍼 크기 증가
```

---

## ⚖️ 트레이드오프

```
Cold vs Hot 트레이드오프:

Cold Publisher:
  장점: 각 구독자 독립, 구독 시점 무관하게 처음부터 수신
  단점: 동일 작업 N번 반복 (비효율적)

Hot Publisher:
  장점: 하나의 소스를 다수 구독자 공유 (효율적)
  단점: 늦게 구독하면 이전 데이터 못 받음

share() vs cache():
  share() = publish().refCount(1):
    구독자 0명 되면 소스 중단 (재구독 시 새로 시작)
    스트리밍 데이터에 적합

  cache():
    첫 구독 결과를 영구 저장
    설정값, 초기화 데이터에 적합
    주의: 완료되지 않는 Flux에 cache() 사용 시 메모리 누수

replay():
  늦게 구독한 구독자에게 이전 N개 데이터 재전송
  WebSocket에서 재연결 시 유용
  메모리 비용: 재전송할 데이터 수만큼
```

---

## 📌 핵심 정리

```
Cold vs Hot Publisher 핵심:

Cold Publisher (Reactor 기본):
  subscribe()마다 새로운 스트림 → 각 구독자 독립
  Mono.just(), Flux.range(), webClient 등 모두 Cold

Hot Publisher:
  구독 시점과 무관하게 진행 중인 스트림
  Sinks.Many/One으로 직접 생성

Cold → Hot 변환:
  share():              구독자 1명 이상이면 유지
  publish().refCount(): 구독자 수 기반 자동 연결/해제
  publish().autoConnect(): 최초 구독 후 영구 유지
  cache():              완료된 결과를 영구 캐싱

선택 기준:
  각 구독자마다 독립 데이터 → Cold (기본)
  다수 구독자가 동일 스트림 공유 → Hot (share/Sinks)
  계산 결과 재사용 → Mono.cache()
```

---

## 🤔 생각해볼 문제

**Q1.** `share()`는 구독자가 0명이 되면 소스를 중단합니다. 이 때 발생할 수 있는 문제와 해결책은 무엇인가요?

<details>
<summary>해설 보기</summary>

`share()` = `publish().refCount(1)`이므로, 모든 구독자가 취소하면 소스가 중단됩니다. 이후 새 구독자가 들어오면 소스가 **처음부터 재시작**됩니다.

문제 시나리오:
- 주식 가격 SSE: 마지막 구독자가 연결을 끊음 → 외부 API 연결 종료
- 새 구독자 연결 → 외부 API 다시 연결 → 연결 중 지연 발생

해결책:
```java
// autoConnect(1) — 한 번 시작하면 구독자 없어도 유지
Flux<Price> prices = stockApi.stream()
    .publish()
    .autoConnect(1);  // 첫 구독 시 시작, 이후 구독자 수 무관

// 또는 애플리케이션 시작 시 즉시 connect
ConnectableFlux<Price> connectable = stockApi.stream().publish();
connectable.connect();  // 구독자 없어도 즉시 시작
Flux<Price> shared = connectable;
```

`autoConnect`는 구독자가 없어도 소스를 계속 실행하므로 메모리와 I/O 자원을 계속 사용합니다. 트레이드오프를 고려해 선택해야 합니다.

</details>

---

**Q2.** `Sinks.many().unicast()`와 `Sinks.many().multicast()`의 차이는 무엇인가요?

<details>
<summary>해설 보기</summary>

**unicast()**: 구독자가 **단 1명**만 허용됩니다. 두 번째 subscribe 시 `IllegalStateException` 발생.
- 단일 소비자 파이프라인 보장이 필요할 때
- 큐 기반으로 Backpressure 완전 지원

**multicast()**: 여러 구독자에게 동일 데이터 전달.
- SSE, WebSocket 브로드캐스트
- `onBackpressureBuffer()`: 느린 구독자를 위한 버퍼 사용
- 구독자가 없을 때 발행된 데이터는 소실 (버퍼링 없음)

```java
// unicast — 단일 소비자
Sinks.Many<String> unicast = Sinks.many().unicast().onBackpressureBuffer();

// multicast — 여러 소비자
Sinks.Many<String> multicast = Sinks.many().multicast().onBackpressureBuffer();

// replay — 늦은 구독자에게 이전 n개 재전송
Sinks.Many<String> replay = Sinks.many().replay().limit(10);
// 새 구독자는 최근 10개 이벤트를 즉시 수신
```

</details>

---

**Q3.** `Flux.interval()`은 Cold인가요, Hot인가요?

<details>
<summary>해설 보기</summary>

`Flux.interval()`은 **Cold Publisher**입니다. `subscribe()`마다 새로운 독립 타이머가 시작됩니다.

```java
Flux<Long> interval = Flux.interval(Duration.ofSeconds(1));

interval.subscribe(t -> log.info("A: {}", t));  // 타이머 A 시작
Thread.sleep(3000);
interval.subscribe(t -> log.info("B: {}", t));  // 타이머 B 시작 (0부터)

// A: 0, 1, 2, 3, 4...
// B: 0, 1, 2, 3...  (A와 독립된 새 타이머)
```

`Flux.interval()`을 여러 구독자가 공유하려면 `share()`나 `publish().refCount()`로 Hot으로 변환해야 합니다:

```java
Flux<Long> sharedInterval = Flux.interval(Duration.ofSeconds(1)).share();

sharedInterval.subscribe(t -> log.info("A: {}", t));
Thread.sleep(3000);
sharedInterval.subscribe(t -> log.info("B: {}", t));

// A: 0, 1, 2, 3, 4...
// B: 3, 4...  (같은 타이머 공유, B는 3부터 수신)
```

</details>

---

<div align="center">

**[⬅️ 이전: 스케줄러](./04-scheduler-thread-switching.md)** | **[홈으로 🏠](../README.md)** | **[다음: Backpressure 전략 ➡️](./06-backpressure-strategies.md)**

</div>
