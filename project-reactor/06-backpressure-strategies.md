# Backpressure 전략 — BUFFER / DROP / LATEST / ERROR

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 생산자가 소비자보다 빠를 때 Backpressure 없이 어떤 일이 일어나는가?
- `BUFFER`, `DROP`, `LATEST`, `ERROR` 전략은 각각 어떤 상황에서 적합한가?
- Reactor의 기본 prefetch 크기(256)는 어떻게 동작하는가?
- Hot Publisher에서 Backpressure를 어떻게 적용하는가?
- `limitRate(n)`과 `onBackpressureBuffer(n)`의 차이는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Backpressure 없는 Reactive 스트림은 폭탄에 뇌관을 달지 않은 것과 같습니다. 평상시에는 잘 동작하는 것처럼 보이지만, 생산자가 소비자보다 조금이라도 빠른 순간 메모리가 폭발합니다. Kafka에서 데이터를 소비하거나, 실시간 IoT 데이터를 처리하거나, 대용량 파일을 스트리밍할 때 Backpressure 전략 선택이 서비스 안정성을 결정합니다.

---

## 😱 흔한 실수 (Before — Backpressure 없이 스트림 처리)

```
실수 1: 빠른 생산자 + 느린 소비자 (OOM)

  Flux.interval(Duration.ofMillis(1))   // 초당 1,000개 생산
      .flatMap(i ->
          Mono.delay(Duration.ofMillis(10))  // 소비는 100ms
              .map(d -> processData(i))
      )
      .subscribe();
  
  flatMap 내부 큐에 데이터 무한 적재
  → 수 초 내에 OutOfMemoryError

실수 2: Hot Publisher에서 Backpressure 무시

  hotEventStream.subscribe(event -> {
      slowProcess(event);  // 처리 느림
  });
  // Hot Publisher는 소비자 속도 무시하고 계속 발행
  // 내부 큐 → 메모리 폭발

실수 3: subscribe() 람다에서 느린 처리

  flux.subscribe(item -> {
      Thread.sleep(100);  // 느린 처리
      // Backpressure 없음 → 내부 버퍼에 데이터 쌓임
  });
```

---

## ✨ 올바른 접근 (After — 명시적 Backpressure 전략)

```
전략 선택 기준:

데이터 소실 불가, 처리 지연 허용  → BUFFER (크기 제한 필수)
일부 데이터 소실 허용, 최신만 필요 → LATEST
일부 데이터 소실 허용, 순서 유지   → DROP
소비자 지연 = 시스템 에러         → ERROR

실무 기본:
  Hot Publisher + 소비자 처리 →
    onBackpressureBuffer(1000).onOverflowDrop()
    또는 onBackpressureLatest()

  Kafka Consumer → flatMap(fn, prefetch)으로 처리량 제어

  파일 스트리밍 → limitRate(n)으로 읽기 속도 제어
```

---

## 🔬 내부 동작 원리

### 1. Backpressure가 없을 때 — 메모리 폭발 원리

```
Hot Publisher + 느린 소비자 시나리오:

생산자 속도: 초당 10,000개
소비자 속도: 초당 100개

Backpressure 없음:
  t=0:   생산 10,000개 → 소비 100개 → 버퍼 9,900개
  t=1:   생산 10,000개 → 소비 100개 → 버퍼 19,800개
  t=2:   생산 10,000개 → 소비 100개 → 버퍼 29,700개
  ...
  t=N:   버퍼 무한 증가 → OutOfMemoryError

내부 버퍼 구조:
  Reactor는 연산자 간에 고정 크기 SpscLinkedArrayQueue 사용
  기본 크기: Queues.SMALL_BUFFER_SIZE = 256개
  → 256개 초과 시 MissingBackpressureException 발생

subscribe() 람다는 request(Long.MAX_VALUE):
  구독 시 자동으로 unbounded request
  → 생산자는 최대 속도로 발행
  → 소비자가 느리면 내부 큐 폭발
```

### 2. BUFFER 전략 — 버퍼에 저장 후 처리

```
onBackpressureBuffer(maxSize):
  소비자가 처리하지 못한 데이터를 버퍼에 저장
  maxSize 초과 시 → 에러 또는 정책에 따라 처리

  Flux.interval(Duration.ofMillis(1))
      .onBackpressureBuffer(1000)  // 최대 1,000개 버퍼
      .subscribe(item -> slowProcess(item));

  마블 다이어그램:
    생산: ─ 1 ─ 2 ─ 3 ─ 4 ─ 5 ─ ...
    버퍼: [1,2,3,4,5...]
    소비: ────────── 1 ────────── 2 ─ ...

onBackpressureBuffer(maxSize, onOverflow):
  버퍼 초과 시 전략 지정:
  DROP_OLDEST:  가장 오래된 것 버림
  DROP_LATEST:  가장 최신 것 버림 (= 새 데이터 거부)
  ERROR:        OverflowException 발생 (기본)

  Flux.interval(Duration.ofMillis(1))
      .onBackpressureBuffer(1000,
          dropped -> log.warn("버퍼 초과, 버림: {}", dropped),
          BufferOverflowStrategy.DROP_OLDEST)
      .subscribe(item -> slowProcess(item));
```

### 3. DROP 전략 — 초과 데이터 버림

```
onBackpressureDrop():
  소비자가 처리 중일 때 새로 도착한 데이터를 즉시 버림
  소비자가 처리 가능한 데이터만 받음

  Flux.interval(Duration.ofMillis(1))
      .onBackpressureDrop(dropped ->
          log.debug("드롭: {}", dropped))
      .subscribe(item -> slowProcess(item));

  마블 다이어그램:
    생산: ─ 1 ─ 2 ─ 3 ─ 4 ─ 5 ─ 6 ─ ...
          소비 중...       소비 중...
    소비: ─ 1 ────────── 4 ────────── 7 ─ ...
          2,3 드롭     5,6 드롭

  적합한 경우:
    - 실시간 센서 데이터 (최신 값이 중요, 과거 누락 허용)
    - 로그 수집 (일부 로그 누락 허용)
    - UI 갱신 (중간 상태 스킵 허용)
```

### 4. LATEST 전략 — 최신 값만 유지

```
onBackpressureLatest():
  소비자가 준비될 때까지 가장 최신 값 하나만 유지
  이전 미처리 값은 최신 값으로 교체

  Flux.interval(Duration.ofMillis(1))
      .onBackpressureLatest()
      .subscribe(item -> slowProcess(item));

  마블 다이어그램:
    생산: ─ 1 ─ 2 ─ 3 ─ 4 ─ 5 ─ 6 ─ ...
    홀딩: [1→2→3→4→5→6...]  (최신으로 계속 교체)
    소비: ─ 1 ────────── 6 ──────── ...
          (소비 준비 시 그 시점의 최신값)

  DROP vs LATEST 차이:
    DROP:   소비 중 도착한 데이터를 즉시 버림
    LATEST: 소비 완료 시 그때까지 온 것 중 최신 하나 처리

  적합한 경우:
    - 주식 현재가 (항상 가장 최신 가격만 필요)
    - 게임 플레이어 위치 업데이트
    - 실시간 대시보드 지표
```

### 5. ERROR 전략 — 소비자 지연을 에러로 처리

```
onBackpressureError():
  소비자가 처리하지 못하면 MissingBackpressureException 발생

  Flux.interval(Duration.ofMillis(1))
      .onBackpressureError()
      .subscribe(
          item -> slowProcess(item),
          err -> log.error("Backpressure 초과!", err)
      );

  마블 다이어그램:
    생산: ─ 1 ─ 2 ─ 3 ─ ... (소비자가 못 따라옴)
    에러: ─────────── MissingBackpressureException!

  사용 이유:
    - 소비자 지연이 시스템 에러임을 명시적으로 표현
    - 버퍼/드롭 없이 문제를 즉시 발견
    - 알람 → 소비자 스케일 업 또는 생산자 속도 제한
```

### 6. limitRate(n) — 소비자가 생산 속도 제어

```
limitRate(n):
  소비자가 n개씩 request() → 생산자 속도 직접 제어
  Backpressure의 가장 정통한 방식

  Flux.range(1, 1_000_000)
      .limitRate(100)  // 100개씩 요청
      .subscribe(item -> processItem(item));

  내부 동작:
    subscribe() → request(100)
    100개 수신 후 → request(75) (75% 소진 시 다음 요청, 기본 prefetch)
    → 생산자는 최대 100개까지만 앞서 생산

  limitRate(highTide, lowTide):
    highTide: 최초 request 크기
    lowTide:  리필 시 request 크기 (기본: highTide의 75%)

  Reactor 기본 prefetch (256):
    flatMap, publishOn 등의 내부 기본 prefetch = 256
    → 상위 Publisher에게 256개 선요청
    → 75% (192개) 소진 시 다음 256개 요청
```

---

## 💻 실전 코드

### 실험 1: OOM 재현 및 Backpressure 적용

```java
// OOM 유발 (실험용, 운영 사용 금지)
@Test
void withoutBackpressure() {
    // 내부적으로 MissingBackpressureException 발생
    assertThatThrownBy(() -> {
        Flux.interval(Duration.ofMillis(1))
            .map(i -> new byte[1024])  // 각 항목 1KB
            .take(Duration.ofSeconds(5))
            .blockLast();
    }).hasCauseInstanceOf(MissingBackpressureException.class);
}

// Backpressure 적용 버전
@Test
void withBackpressureBuffer() {
    AtomicInteger processed = new AtomicInteger(0);
    AtomicInteger dropped = new AtomicInteger(0);

    Flux.interval(Duration.ofMillis(1))
        .onBackpressureBuffer(500,
            item -> dropped.incrementAndGet(),
            BufferOverflowStrategy.DROP_OLDEST)
        .take(Duration.ofSeconds(5))
        .subscribe(item -> {
            Thread.sleep(10);  // 10ms 처리 시간 시뮬레이션
            processed.incrementAndGet();
        });

    log.info("처리: {}, 드롭: {}", processed.get(), dropped.get());
    // 처리: ~500, 드롭: ~4000
}
```

### 실험 2: Kafka 소비 패턴 — flatMap + limitRate

```java
// Kafka에서 메시지를 소비하며 처리량 제어
@Component
public class KafkaMessageProcessor {

    public Flux<ProcessResult> processMessages(Flux<KafkaMessage> messages) {
        return messages
            .limitRate(50)          // 50개씩 요청 (생산자 속도 제어)
            .flatMap(
                msg -> processMessage(msg)
                    .onErrorResume(e -> {
                        log.error("처리 실패: {}", msg.key(), e);
                        return Mono.just(ProcessResult.failed(msg));
                    }),
                10                  // 동시 처리 10개
            )
            .doOnNext(result ->
                log.info("완료: {} ({})", result.key(), result.status())
            );
    }
}
```

### 실험 3: 실시간 센서 데이터 — LATEST 전략

```java
@RestController
public class SensorController {

    @GetMapping(value = "/sensor/{id}/live",
                produces = TEXT_EVENT_STREAM_VALUE)
    public Flux<SensorReading> streamSensor(@PathVariable String id) {
        return sensorHub.getReadings(id)  // 초당 100개 발행 가능
            .onBackpressureLatest()       // 최신 값만 유지
            .sample(Duration.ofMillis(100))  // 100ms마다 한 번씩 전송
            // → 클라이언트는 초당 10개 수신 (최신 값)
            // → 중간 값은 sample()이 자동으로 최신만 선택
            ;
    }
}
```

---

## 📊 성능 비교

```
Backpressure 전략별 특성 비교:

전략              | 메모리 사용    | 데이터 손실  | CPU 오버헤드
─────────────────┼─────────────┼──────────┼──────────
BUFFER           | 버퍼 크기까지  | 없음       | 버퍼 관리 비용
DROP             | 최소          | 있음       | 드롭 결정 비용 최소
LATEST           | O(1)          | 있음       | 비교/교체 비용
ERROR            | 없음          | 없음       | 예외 처리 비용

생산자 초당 10,000개, 소비자 초당 100개 (100배 차이):

BUFFER(1000):    ~10초 후 버퍼 소진 → DROP_OLDEST 시작
DROP:            초당 9,900개 드롭, 100개 처리
LATEST:          초당 9,999개 무시, 최신 1개씩 처리
ERROR:           즉시 MissingBackpressureException

limitRate 효과:
  limitRate(100): 생산자에게 100개만 요청 → 생산자도 느려짐
  → 메모리 안전, 데이터 손실 없음, 처리량은 소비자 속도
```

---

## ⚖️ 트레이드오프

```
Backpressure 전략 선택 트레이드오프:

데이터 정확성 vs 메모리 안전 vs 처리량:
  BUFFER:  정확성 ↑, 메모리 위험 있음 (크기 제한 필수)
  DROP:    메모리 ↑, 정확성 ↓ (누락 허용)
  LATEST:  메모리 ↑, 정확성 ↓ (항상 최신)
  ERROR:   정확성 신호 ↑, 처리량 ↓ (문제 즉시 노출)

limitRate vs onBackpressure*:
  limitRate: 생산자 속도를 소비자에 맞춤 → 가장 안전
  onBackpressure*: 생산자는 빠르게, 소비자 전에 전략 적용

Hot Publisher 필수 고려사항:
  Hot Publisher는 request(n) 신호 무시 → 소비자 제어 불가
  → onBackpressure* 연산자로 Hot과 소비자 사이에 버퍼/드롭 적용 필요
  Cold Publisher라면 limitRate가 더 효율적
```

---

## 📌 핵심 정리

```
Backpressure 전략 핵심:

왜 필요한가:
  생산자 > 소비자 속도 → 내부 큐 무한 증가 → OOM
  Backpressure = 소비자가 감당할 수 있는 속도로 제어

4가지 전략:
  BUFFER:  소실 없음, 메모리 위험 (크기 제한 필수)
  DROP:    초과 데이터 즉시 버림 (실시간, 누락 허용)
  LATEST:  소비 시점의 최신 값만 (주식 현재가)
  ERROR:   초과 즉시 에러 (문제를 명시적으로)

소스 속도 제어:
  limitRate(n): Cold Publisher의 생산 속도 직접 제어
  flatMap(fn, n): 동시 처리 수 제한

Reactor 기본:
  subscribe() 람다 → request(Long.MAX_VALUE) → unbounded
  연산자 내부 prefetch → 256 (약 75%마다 리필)
```

---

## 🤔 생각해볼 문제

**Q1.** `flatMap(fn)`의 기본 동시성이 `Integer.MAX_VALUE`인데, 실제로 무한에 가깝게 동시 호출이 일어나나요?

<details>
<summary>해설 보기</summary>

`Integer.MAX_VALUE`지만 실제로 무한히 동시 호출이 일어나지는 않습니다. `flatMap` 내부에는 prefetch 버퍼가 있어, 상위 Publisher에게 prefetch(기본 256)개만 요청합니다.

동시성 제한이 없다는 의미는 **prefetch된 항목은 모두 동시에 내부 Publisher로 시작**된다는 것입니다. 256개가 prefetch되면 256개가 동시 실행됩니다.

실무에서 외부 API 호출 시:
```java
// 위험: 256개 동시 HTTP 요청 → 외부 API 연결 풀 초과
orders.flatMap(order -> paymentApi.charge(order))

// 안전: 최대 10개 동시
orders.flatMap(order -> paymentApi.charge(order), 10)
// → 내부적으로 prefetch도 10으로 제한
```

</details>

---

**Q2.** `onBackpressureBuffer()` 버퍼가 가득 찼을 때 기본 동작은 무엇이고, 어떻게 변경하나요?

<details>
<summary>해설 보기</summary>

기본 동작은 `OverflowException`을 발생시켜 스트림을 종료합니다.

```java
// 기본: 버퍼 초과 시 OverflowException
flux.onBackpressureBuffer(100)

// 커스텀: 초과 항목에 대해 콜백 실행 후 ERROR
flux.onBackpressureBuffer(100,
    item -> log.warn("버퍼 초과 항목 드롭: {}", item))
// → 초과 항목을 드롭하고 onError 발생 (스트림 종료)

// 완전한 제어: 버퍼 크기 + 콜백 + 오버플로우 전략
flux.onBackpressureBuffer(100,
    item -> log.warn("드롭: {}", item),
    BufferOverflowStrategy.DROP_OLDEST)  // 가장 오래된 것 제거, 계속 진행
```

`DROP_OLDEST`와 `DROP_LATEST`는 스트림을 종료하지 않고 계속 진행합니다. 실시간 데이터 처리에서 버퍼 오버플로우를 무시하고 싶을 때 사용합니다.

</details>

---

**Q3.** R2DBC로 DB에서 대용량 데이터를 스트리밍할 때 Backpressure는 어떻게 동작하나요?

<details>
<summary>해설 보기</summary>

R2DBC는 Reactive Streams 스펙을 완전히 지원합니다. `request(n)` 신호가 DB 드라이버까지 전달되어, **DB에서 실제로 n행만 fetch**합니다.

```java
// R2DBC 스트리밍
userRepository.findAll()  // Flux<User>
    .limitRate(100)        // 100행씩 DB에서 fetch
    .flatMap(user -> process(user), 10)
    .subscribe();
```

내부 동작:
1. `limitRate(100)` → `request(100)` → R2DBC → DB: `FETCH 100 ROWS`
2. 100행 처리 → `request(75)` → R2DBC → DB: `FETCH 75 ROWS`
3. 반복

JDBC와의 차이:
- JDBC: `findAll()` → DB에서 전체 결과를 메모리로 → OOM 위험
- R2DBC: `findAll()` → `limitRate(100)` → DB에서 100행씩 → 메모리 O(100)

이것이 R2DBC의 Backpressure 지원이 대용량 데이터 처리에서 특히 중요한 이유입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Cold vs Hot Publisher](./05-cold-vs-hot-publisher.md)** | **[홈으로 🏠](../README.md)** | **[다음: Context ➡️](./07-reactor-context.md)**

</div>
