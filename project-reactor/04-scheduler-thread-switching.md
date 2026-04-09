# 스케줄러(Scheduler) — 스레드 전환이 일어나는 시점

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `subscribeOn`은 어느 시점에 스레드를 전환하고, 그 영향은 파이프라인 어디까지인가?
- `publishOn`은 어느 시점에 스레드를 전환하고, 여러 개를 쓰면 어떻게 되는가?
- `Schedulers.boundedElastic()`과 `Schedulers.parallel()`은 언제 각각 선택하는가?
- EventLoop 스레드에서 블로킹 코드를 실행하면 정확히 무슨 일이 일어나는가?
- 동일 파이프라인에서 `subscribeOn`과 `publishOn`을 함께 쓰면 어떤 순서로 적용되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

WebFlux에서 가장 이해하기 어려운 개념 중 하나가 스레드 전환입니다. "이 코드가 어느 스레드에서 실행되는가?"를 모르면 EventLoop를 블로킹하는 버그, ThreadLocal 유실, 테스트에서만 발생하는 데드락 등을 만나게 됩니다.

`subscribeOn`과 `publishOn`은 비슷해 보이지만 영향 범위가 완전히 다릅니다. 이 차이를 정확히 이해하면 "JDBC를 써야 하는데 WebFlux에서 어떻게 하지?"라는 질문에 자신 있게 답할 수 있습니다.

---

## 😱 흔한 실수 (Before — 스레드 전환을 모를 때)

```
실수 1: EventLoop 스레드에서 블로킹 코드 실행

  @GetMapping("/data")
  public Mono<String> getData() {
      return Mono.fromCallable(() -> {
          return jdbcTemplate.queryForObject(...);  // 블로킹!
      });
      // → fromCallable이 EventLoop 스레드에서 실행
      // → jdbcTemplate이 EventLoop를 블로킹
      // → 이 EventLoop가 담당하는 모든 연결 지연

실수 2: subscribeOn 위치가 잘못됨

  Mono.fromCallable(() -> blockingWork())
      .map(s -> s.toUpperCase())
      .subscribeOn(Schedulers.boundedElastic())
      // subscribeOn은 어디 있든 전체 파이프라인의 시작 스레드를 결정
      // → 이 코드는 올바름, 위치는 상관없음 (하지만 관례상 뒤에 붙이면 혼란)
      .subscribe();

실수 3: publishOn 이전 코드도 전환될 것을 기대

  Mono.fromCallable(() -> {
      log.info("스레드: {}", Thread.currentThread().getName());
      return "data";
  })
  .publishOn(Schedulers.parallel())  // 여기 이후만 전환
  .map(s -> {
      log.info("스레드: {}", Thread.currentThread().getName());
      return s.toUpperCase();
  })
  .subscribe();
  // fromCallable: EventLoop 스레드 (publishOn 이전)
  // map: parallel 스레드 (publishOn 이후)
  // 이전 코드도 바꾸려면 subscribeOn 사용
```

---

## ✨ 올바른 접근 (After — 스레드 전환 원칙 이해)

```
핵심 원칙:

subscribeOn(scheduler):
  파이프라인 시작(subscribe) 시 사용할 스레드 지정
  위치에 관계없이 전체 파이프라인에 영향 (첫 번째 subscribeOn만 유효)
  블로킹 소스 래핑의 표준 패턴

publishOn(scheduler):
  해당 연산자 이후의 코드가 실행될 스레드 전환
  여러 번 사용 가능 → 파이프라인 중간에 스레드 변경
  CPU 집약 작업과 I/O를 분리할 때 유용

블로킹 코드 오프로딩 표준 패턴:
  Mono.fromCallable(() -> jdbcTemplate.queryForObject(...))
      .subscribeOn(Schedulers.boundedElastic())
  // JDBC 작업이 boundedElastic 스레드에서 실행
  // EventLoop 스레드는 자유롭게 다른 이벤트 처리
```

---

## 🔬 내부 동작 원리

### 1. Schedulers 종류와 특성

```
Schedulers.immediate():
  현재 스레드에서 실행 (전환 없음)
  테스트 또는 이미 적절한 스레드에서 실행 중일 때

Schedulers.single():
  단일 재사용 스레드
  순서 보장이 필요한 경량 작업에 사용
  주의: 이 스레드가 막히면 모든 작업 지연

Schedulers.parallel():
  CPU 코어 수만큼 스레드 (기본: Runtime.getRuntime().availableProcessors())
  CPU 집약 작업 전용 (블로킹 코드 사용 금지)
  Netty EventLoop와 유사한 구조

Schedulers.boundedElastic():
  동적으로 스레드 생성, 최대 10 × CPU 코어 수 스레드
  유휴 60초 후 스레드 반환
  블로킹 코드 실행 전용 (JDBC, 파일 I/O 등)
  무한정 스레드 생성 방지 (bounded)

Schedulers.fromExecutorService(executor):
  커스텀 ExecutorService 사용
  특정 스레드 풀 정책이 필요할 때
```

### 2. subscribeOn 동작 원리

```
subscribeOn은 subscribe() 호출 시 사용할 스레드를 지정:

Mono.fromCallable(() -> {
    log.info("실행 스레드: {}", Thread.currentThread().getName());
    return blockingWork();
})
.subscribeOn(Schedulers.boundedElastic())
.map(s -> {
    log.info("map 스레드: {}", Thread.currentThread().getName());
    return s.toUpperCase();
})
.subscribe();

실행 흐름:
  subscribe() 호출 → subscribeOn이 boundedElastic 스레드로 전환
  boundedElastic 스레드에서 fromCallable 실행
  map도 동일 boundedElastic 스레드에서 실행

로그:
  실행 스레드: boundedElastic-1
  map 스레드:  boundedElastic-1

중요한 특성:
  subscribeOn은 어디에 위치해도 동일 효과
  여러 subscribeOn 중 첫 번째만 유효
  전체 상위(upstream) 파이프라인 실행 스레드 결정
```

### 3. publishOn 동작 원리

```
publishOn은 해당 연산자 이후의 실행 스레드를 전환:

Mono.fromCallable(() -> {
    log.info("실행 스레드: {}", Thread.currentThread().getName());
    return "data";
})
.publishOn(Schedulers.parallel())  // 여기서 스레드 전환
.map(s -> {
    log.info("map 스레드: {}", Thread.currentThread().getName());
    return s.toUpperCase();
})
.publishOn(Schedulers.boundedElastic())  // 또 전환
.subscribe(result -> {
    log.info("subscribe 스레드: {}", Thread.currentThread().getName());
});

로그:
  실행 스레드: reactor-http-epoll-1  (EventLoop)
  map 스레드:  parallel-1            (publishOn 이후)
  subscribe:   boundedElastic-1      (두 번째 publishOn 이후)

publishOn 내부 동작:
  상위에서 onNext 신호 수신
  → 신호를 큐에 저장
  → 지정된 스케줄러 스레드에서 큐를 드레인
  → 하위 Subscriber.onNext() 호출
  → 스레드 전환 비용: 큐 접근 + 스케줄러 오버헤드
```

### 4. subscribeOn vs publishOn — 핵심 차이

```
비유:
  subscribeOn: "이 공장(파이프라인) 전체를 A 공장 노동자가 운영"
  publishOn:   "이 컨베이어 벨트 이후로는 B 팀이 처리"

파이프라인 다이어그램:

  [Source] → [op1] → [op2] → publishOn(P) → [op3] → [op4]
                                                ↑
                               여기 이후: P 스케줄러

  subscribeOn(S):
  [Source] → [op1] → [op2] → [op3] → [op4]
      ↑
  전체: S 스케줄러 (subscribeOn 위치 무관)

실무 패턴:

  패턴 1: 블로킹 소스 오프로딩
    Mono.fromCallable(() -> jdbcTemplate.query(...))
        .subscribeOn(Schedulers.boundedElastic())
    → 블로킹 코드를 EventLoop에서 분리

  패턴 2: CPU 작업 후 I/O
    dataFlux
        .publishOn(Schedulers.parallel())  // CPU 집약 작업
        .map(item -> heavyCompute(item))
        .publishOn(Schedulers.boundedElastic())  // I/O 작업
        .flatMap(item -> fileService.write(item))
```

### 5. EventLoop 블로킹 — 왜 치명적인가

```
Netty EventLoop 스레드에서 블로킹 발생 시:

정상 상태 (논블로킹):
  EventLoop-1: Channel-A 처리 → Channel-B 이벤트 처리 → Channel-C 이벤트 처리
  → 수천 개 채널을 빠르게 순환

블로킹 발생 시:
  EventLoop-1: Channel-A 처리 중...
    JDBC.execute()  ← 200ms 블로킹 시작
    [200ms 동안 EventLoop-1 점유]
    Channel-B, C, D, ... 이벤트 처리 대기
  → 200ms 동안 Channel-B, C, D의 모든 요청 지연

검출: BlockHound
  BlockHound.install();
  // EventLoop 스레드에서 블로킹 호출 발생 시:
  // BlockingOperationError: Blocking call!
  //   at java.io.FileInputStream.read(...)
  //   at com.example.Service.loadFile(...)

해결:
  Mono.fromCallable(() -> blockingWork())
      .subscribeOn(Schedulers.boundedElastic())
  // boundedElastic 스레드에서 블로킹 → EventLoop는 자유
```

---

## 💻 실전 코드

### 실험 1: 스레드 전환 흐름 로깅

```java
@Test
void threadSwitchingDemo() {
    Mono.fromCallable(() -> {
        log.info("fromCallable: {}", Thread.currentThread().getName());
        return "data";
    })
    .subscribeOn(Schedulers.boundedElastic())
    .map(s -> {
        log.info("map1: {}", Thread.currentThread().getName());
        return s + "-mapped";
    })
    .publishOn(Schedulers.parallel())
    .map(s -> {
        log.info("map2: {}", Thread.currentThread().getName());
        return s.toUpperCase();
    })
    .block();

    // 출력:
    // fromCallable: boundedElastic-1  (subscribeOn 영향)
    // map1:         boundedElastic-1  (subscribeOn 영향, publishOn 이전)
    // map2:         parallel-1        (publishOn 이후)
}
```

### 실험 2: 블로킹 JDBC를 안전하게 오프로딩

```java
@Service
public class UserService {

    private final JdbcTemplate jdbcTemplate;

    // 올바른 패턴: JDBC 블로킹을 boundedElastic으로 오프로딩
    public Mono<User> findById(Long id) {
        return Mono.fromCallable(() ->
            jdbcTemplate.queryForObject(
                "SELECT * FROM users WHERE id = ?",
                userRowMapper,
                id
            )
        )
        .subscribeOn(Schedulers.boundedElastic());
        // JDBC 실행: boundedElastic 스레드
        // 결과 전달: 기존 스레드 (WebFlux가 처리)
    }

    // 목록 조회
    public Flux<User> findAll() {
        return Mono.fromCallable(() ->
            jdbcTemplate.query("SELECT * FROM users", userRowMapper)
        )
        .subscribeOn(Schedulers.boundedElastic())
        .flatMapMany(Flux::fromIterable);
    }
}
```

### 실험 3: BlockHound로 블로킹 감지

```java
@SpringBootTest
class BlockingDetectionTest {

    @BeforeAll
    static void setup() {
        BlockHound.install();  // 테스트 환경 블로킹 감지 활성화
    }

    @Test
    void detectsBlockingInEventLoop() {
        // 블로킹 코드가 EventLoop에서 실행되면 예외 발생
        assertThatThrownBy(() -> {
            Mono.fromCallable(() -> {
                Thread.sleep(100);  // 블로킹!
                return "done";
            }).block();
        }).hasCauseInstanceOf(BlockingOperationError.class);
    }

    @Test
    void noBlockingWithBoundedElastic() {
        // boundedElastic에서는 블로킹 허용
        String result = Mono.fromCallable(() -> {
            Thread.sleep(100);  // OK — boundedElastic 스레드
            return "done";
        })
        .subscribeOn(Schedulers.boundedElastic())
        .block();

        assertThat(result).isEqualTo("done");
    }
}
```

---

## 📊 성능 비교

```
스케줄러별 특성 비교:

스케줄러          | 스레드 수           | 블로킹 허용 | 적합한 작업
─────────────────┼───────────────────┼────────────┼────────────────
immediate()      | 현재 스레드         | 상황에 따름 | 전환 불필요
single()         | 1개 고정           | 아니오      | 순서 보장 단순 작업
parallel()       | CPU 코어 수         | 아니오      | CPU 집약 작업
boundedElastic() | 최대 10×CPU (동적) | 예          | JDBC, 파일 I/O

publishOn 스레드 전환 비용:
  onNext → 큐 삽입 → 스케줄러 스레드 wake → 큐 드레인 → 하위 onNext
  비용: ~수 μs (컨텍스트 스위치 포함)
  → 불필요한 publishOn 남발 시 오버헤드 축적

Schedulers.boundedElastic() 한계:
  기본 최대 스레드: 10 × availableProcessors (8코어 → 80개)
  초과 시: 큐에서 대기 (최대 100,000개 큐잉 가능)
  → JDBC 집약 서비스에서 boundedElastic도 부족할 수 있음
  → 이 경우 완전한 R2DBC 전환 필요
```

---

## ⚖️ 트레이드오프

```
스레드 전환 트레이드오프:

subscribeOn 사용:
  장점: 블로킹 코드를 EventLoop에서 안전하게 분리
  단점: boundedElastic 스레드가 블로킹 대기 (Thread-per-Request와 유사)
        → JDBC 집약 서비스라면 MVC가 더 단순

publishOn 사용:
  장점: 파이프라인 중간에 스레드 전환으로 작업 분리
  단점: 컨텍스트 스위치 오버헤드, ThreadLocal 전파 불가

이상적인 설계:
  논블로킹 드라이버(R2DBC, Reactive Redis)를 사용하면
  subscribeOn/publishOn 없이 EventLoop만으로 처리
  → 스케줄러는 어쩔 수 없이 블로킹 코드를 써야 할 때의 차선책
```

---

## 📌 핵심 정리

```
Scheduler와 스레드 전환 핵심:

subscribeOn(scheduler):
  파이프라인 시작 스레드 지정
  위치 무관, 전체 상위(upstream) 영향
  블로킹 소스 오프로딩의 표준 패턴

publishOn(scheduler):
  해당 연산자 이후 실행 스레드 전환
  여러 번 사용 가능 (각각의 이후부터 적용)
  CPU↔I/O 작업 분리 시 사용

Schedulers:
  boundedElastic: 블로킹 허용, 동적 스레드 (JDBC, 파일 I/O)
  parallel:       CPU 집약, 코어 수 스레드 (블로킹 금지)

EventLoop 블로킹:
  → 해당 EventLoop 담당 모든 채널 지연
  → BlockHound로 개발 시 감지
  → boundedElastic 오프로딩으로 해결
```

---

## 🤔 생각해볼 문제

**Q1.** 동일 파이프라인에 `subscribeOn(A)`와 `subscribeOn(B)`가 모두 있으면 어떻게 되나요?

<details>
<summary>해설 보기</summary>

첫 번째로 만나는 `subscribeOn`만 유효합니다. 파이프라인을 아래에서 위로 타고 올라가는 subscribe 과정에서 첫 번째 `subscribeOn`이 스케줄러를 지정하면, 그 이상 올라가도 다시 변경되지 않습니다.

```java
Mono.fromCallable(() -> "data")
    .subscribeOn(Schedulers.boundedElastic())   // 이것이 유효
    .map(s -> s.toUpperCase())
    .subscribeOn(Schedulers.parallel())         // 무시됨
    .subscribe();

// fromCallable과 map 모두 boundedElastic에서 실행
```

`publishOn`은 이와 다르게 여러 번 사용할 수 있고 각 위치에서 이후 파이프라인의 스레드를 변경합니다.

</details>

---

**Q2.** `Schedulers.boundedElastic()`의 "bounded"는 무엇을 의미하나요? `Schedulers.elastic()`과의 차이는?

<details>
<summary>해설 보기</summary>

`elastic()`은 Reactor 3.5에서 deprecated되었고, `boundedElastic()`으로 대체되었습니다.

**차이점:**
- `elastic()`: 스레드 수 무제한 → OOM 위험
- `boundedElastic()`: 최대 `10 × availableProcessors` 스레드, 초과 작업은 큐에서 대기

"bounded"는 스레드 수에 상한이 있다는 의미입니다:

```
기본값:
  최대 스레드 수: 10 × Runtime.getRuntime().availableProcessors()
  큐 용량: 100,000개
  유휴 타임아웃: 60초 (이후 스레드 반환)
```

커스터마이징:
```java
Schedulers.newBoundedElastic(
    50,         // 최대 스레드 수
    10_000,     // 큐 용량
    "my-pool",  // 스레드 이름 접두사
    60          // 유휴 타임아웃 (초)
);
```

</details>

---

**Q3.** WebFlux에서 Spring Security의 `SecurityContext`는 어떻게 스레드가 바뀌어도 전달되나요?

<details>
<summary>해설 보기</summary>

전통적인 `ThreadLocal` 기반의 `SecurityContextHolder`는 스레드가 바뀌면 컨텍스트가 유실됩니다. WebFlux에서는 Reactor `Context`를 사용합니다.

`ReactiveSecurityContextHolder`는 Reactor Context에 `SecurityContext`를 저장합니다:

```java
// WebFlux 필터에서 컨텍스트 설정
return chain.filter(exchange)
    .contextWrite(ReactiveSecurityContextHolder
        .withSecurityContext(Mono.just(securityContext)));

// 파이프라인 어디서든 꺼내기
ReactiveSecurityContextHolder.getContext()
    .map(ctx -> ctx.getAuthentication().getName())
    .subscribe(username -> log.info("사용자: {}", username));
```

Reactor Context는 파이프라인을 타고 자동으로 전파되며, 스레드가 `publishOn`으로 바뀌어도 Context는 유지됩니다. 이것이 ThreadLocal 대신 Reactor Context를 사용하는 핵심 이유입니다. (자세한 내용은 07-reactor-context.md 참고)

</details>

---

<div align="center">

**[⬅️ 이전: 에러 처리](./03-error-handling.md)** | **[홈으로 🏠](../README.md)** | **[다음: Cold vs Hot Publisher ➡️](./05-cold-vs-hot-publisher.md)**

</div>
