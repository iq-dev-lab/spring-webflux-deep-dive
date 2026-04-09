# 운영 중 발생하는 문제 패턴 — BlockHound와 메모리 누수

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `BlockHound`가 감지하지 못하는 블로킹 패턴은 무엇인가?
- 구독 해제가 안 된 `Flux`로 인한 메모리 누수를 어떻게 진단하는가?
- `Schedulers.parallel()`에서 블로킹 코드를 실행하면 어떤 증상이 나타나는가?
- Reactor의 `checkpoint()` 연산자는 디버깅에 어떻게 도움이 되는가?
- 운영 환경에서 Reactive 앱의 건강 상태를 어떻게 모니터링하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

WebFlux 앱을 운영하다 보면 MVC에서 보기 힘든 독특한 문제들이 나타납니다. 특정 시간 이후 처리량이 점진적으로 감소하는 메모리 누수, EventLoop를 가끔 블로킹하는 코드로 인한 간헐적 지연, 스택 트레이스가 분산되어 원인을 찾기 어려운 에러 — 이런 문제들을 진단하고 해결하는 방법을 알아야 진정한 WebFlux 운영을 할 수 있습니다.

---

## 😱 흔한 실수 (Before — 운영 문제를 놓치는 경우)

```
실수 1: BlockHound를 개발 환경에서만 사용

  // 개발 환경: BlockHound 설치 → 즉시 발견
  BlockHound.install();
  
  // 운영 환경: BlockHound 미설치 → 증상만 보임
  // "가끔 특정 요청이 느림" → 원인 불명
  // → CI/CD 파이프라인에서 BlockHound 테스트 필수

실수 2: Flux.subscribe()로 구독 후 Disposable 관리 안 함

  // 초기화 코드에서
  hotPublisher.subscribe(item -> processItem(item));
  // Disposable 반환값 무시
  // → 앱이 종료되어도 구독 유지
  // → 메모리 누수, 리소스 누수

실수 3: 에러 스택 트레이스를 보고 원인을 알 수 없음

  reactor.core.Exceptions$ErrorCallbackNotImplemented: java.lang.NullPointerException
    at reactor.core.publisher.Operators.onErrorDropped(...)
    // "어디서 NPE가 났는지 모름"
  → Hooks.onOperatorDebug() 또는 checkpoint() 사용
```

---

## ✨ 올바른 접근 (After — 운영 문제 진단 체계)

```
운영 문제 진단 체계:

1. 예방 — 개발/CI 단계에서 감지
   BlockHound 테스트 (CI 필수)
   Reactor Scheduler 올바른 선택
   Disposable 관리 패턴

2. 발견 — 증상으로 원인 유추
   처리량 점진 감소 → 메모리 누수 or EventLoop 블로킹
   특정 요청 패턴에서 지연 → 블로킹 I/O
   스택 트레이스 불명확 → checkpoint() 추가

3. 진단 — 도구 활용
   JVM Heap Dump + MAT(Memory Analyzer)
   Micrometer EventLoop 메트릭
   Reactor 디버그 모드 (checkpoint, Hooks)

4. 해결 — 근본 원인 수정
   블로킹 코드 → boundedElastic 오프로딩 or 논블로킹 대체
   메모리 누수 → Disposable 관리 또는 Hot Publisher 정리
```

---

## 🔬 내부 동작 원리

### 1. BlockHound가 감지하는 것과 못하는 것

```
BlockHound 감지 원리:
  ByteBuddy로 블로킹 메서드들을 인터셉트
  현재 스레드가 NonBlockingThread(Reactor 표시)인지 확인
  NonBlocking 스레드에서 블로킹 메서드 호출 시 예외

감지할 수 있는 블로킹:
  Thread.sleep()           → 직접 감지
  Object.wait()            → 감지
  Socket.read()            → 감지 (JDBC 포함)
  Files.readAllBytes()     → 감지
  synchronized 블록 대기   → 부분적 감지

감지하지 못하는 블로킹:
  LockSupport.parkNanos() (일부 내부 사용)
  Java NIO 비동기 API의 내부 동기화
  제3자 라이브러리의 커스텀 대기 메커니즘
  CompletableFuture.get() (감지 가능하지만 설정 필요)

false positive 처리:
  BlockHound.install(builder -> builder
      .allowBlockingCallsInside(
          "org.hibernate.validator", "validate"
          // Bean Validation은 짧은 동기 작업 허용
      )
      .allowBlockingCallsInside(
          "com.example.Cache", "get"
          // 인메모리 캐시 동기 접근 허용
      )
  );
```

### 2. 메모리 누수 패턴과 진단

```
메모리 누수 패턴 1: 무한 Hot Publisher 구독 해제 안 함

  // 문제 코드
  public void initialize() {
      hotPublisher.subscribe(item -> {
          processItem(item);  // Disposable 무시
      });
      // 앱이 재시작되어도 이 구독은 GC되지 않음
      // (hotPublisher가 이 Subscriber 참조를 가짐)
  }

  // 올바른 코드
  private Disposable subscription;

  @PostConstruct
  public void initialize() {
      subscription = hotPublisher.subscribe(item -> processItem(item));
  }

  @PreDestroy
  public void destroy() {
      if (subscription != null && !subscription.isDisposed()) {
          subscription.dispose();
      }
  }

메모리 누수 패턴 2: 취소되지 않은 SSE 연결

  // Flux가 취소되지 않으면 Hot Publisher에 구독이 쌓임
  @GetMapping(value = "/events", produces = TEXT_EVENT_STREAM_VALUE)
  public Flux<Event> streamEvents() {
      return hotPublisher.asFlux()
          .doOnCancel(() -> log.info("구독 취소 확인"));
          // 클라이언트 연결 끊기 → Netty 감지 → Flux cancel
          // cancel이 전파되지 않으면 hotPublisher 구독 유지
  }

진단 방법:
  1. JVM 메트릭: Heap 사용량이 시간에 따라 선형 증가
  2. GC 로그: Old Gen이 가득 차고 GC 후에도 해제 안 됨
  3. Heap Dump: 특정 Subscriber 클래스 인스턴스 수 증가
     → MAT에서 "Leak Suspects" 분석

  Micrometer 메트릭:
    jvm.memory.used (heap): 시간에 따른 증가 추이
    jvm.gc.pause: GC 빈도/시간 증가
```

### 3. Schedulers.parallel()에서 블로킹 — 증상과 해결

```
잘못된 스케줄러 선택:

  Flux.range(1, 1000)
      .publishOn(Schedulers.parallel())  // CPU 집약용 스케줄러
      .flatMap(i ->
          Mono.fromCallable(() -> jdbcTemplate.query(...))
          // parallel 스케줄러에서 JDBC 블로킹!
      )
      .subscribe();

증상:
  parallel 스케줄러 스레드 수 = CPU 코어 수 (예: 8개)
  8개 스레드 모두 JDBC 블로킹 → 다른 CPU 집약 작업도 지연
  CPU 사용률 낮은데 처리량 낮음 (스레드가 I/O 대기)

BlockHound 로그:
  BlockingOperationError: Blocking call! java.sql.Connection.prepareStatement
    at com.example.service.UserService.lambda$findById$0(UserService.java:42)
    at reactor.core.scheduler.SchedulerTask.run(...)
  Scheduler: parallel-1  ← parallel 스케줄러임을 확인

해결:
  Mono.fromCallable(() -> jdbcTemplate.query(...))
      .subscribeOn(Schedulers.boundedElastic())  // 블로킹용 스케줄러
  // parallel 스케줄러는 CPU 집약 작업 전용으로 남겨둠
```

### 4. Reactor 디버그 — checkpoint와 Hooks

```
Reactive 스택 트레이스 문제:
  동기 코드: NPE → 정확한 라인 번호
  Reactive: NPE → 내부 Reactor 라인 번호 (원인 불명)

해결책 1: checkpoint() 연산자
  Mono.fromCallable(() -> service.process())
      .checkpoint("service.process() after")  // 스택 트레이스에 표시
      .flatMap(result -> downstream(result))
      .checkpoint("downstream after");
  
  에러 시 스택 트레이스:
    Caused by: java.lang.NullPointerException
    at ... (실제 에러 위치)
    Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException:
    Assembly trace from producer [MonoFlatMap]:
      checkpoint("service.process() after")  ← 위치 표시
      checkpoint("downstream after")

해결책 2: Hooks.onOperatorDebug() (개발 환경)
  Hooks.onOperatorDebug();  // 애플리케이션 시작 시
  // 모든 연산자에 디버그 정보 추가 → 성능 오버헤드 있음
  // → 개발/테스트 환경에서만 사용

해결책 3: ReactorDebugAgent (운영 환경 가능)
  Java Agent: -javaagent:reactor-tools.jar
  // 바이트코드 변환으로 스택 트레이스 개선
  // 성능 오버헤드 낮음 (~5%)
  // 운영 환경에서 사용 가능
```

### 5. EventLoop 상태 모니터링

```
Netty EventLoop 메트릭 (Micrometer):

  // EventLoop 큐 크기 모니터링
  reactor.netty.eventloop.pending.tasks    # 대기 중 태스크 수
  reactor.netty.bytebuf.allocator.active.heap.memory  # 버퍼 메모리

  // 커스텀 메트릭
  @Bean
  public NettyServerCustomizer nettyMetricsCustomizer(MeterRegistry registry) {
      return server -> server
          .metrics(true, Function.identity());  // Netty 자체 메트릭 활성화
  }

경보 설정:
  # EventLoop 블로킹 감지 (간접)
  - alert: SlowEventLoopRespone
    expr: histogram_quantile(0.99, http_request_duration_seconds_bucket) > 1
    # 99th percentile이 1초 초과 → EventLoop 블로킹 의심

  # 메모리 누수 감지
  - alert: HeapMemoryLeaking
    expr: increase(jvm_memory_used_bytes{area="heap"}[30m]) > 100_000_000
    # 30분에 100MB 이상 증가 → 누수 의심

  # Reactor 스케줄러 큐 적체
  - alert: SchedulerQueueBuild up
    expr: executor_queued_tasks{executor="boundedElastic"} > 500
    # boundedElastic 큐 500개 이상 → 블로킹 병목
```

---

## 💻 실전 코드

### 실험 1: BlockHound CI 통합

```java
// 모든 통합 테스트에서 BlockHound 활성화
@TestConfiguration
public class BlockHoundTestConfig {

    @BeforeAll
    public static void installBlockHound() {
        BlockHound.install(builder -> builder
            // 검증 라이브러리는 허용 (짧은 동기 작업)
            .allowBlockingCallsInside(
                "jakarta.validation.Validator", "validate")
            // 로깅은 허용
            .allowBlockingCallsInside(
                "ch.qos.logback.core.OutputStreamAppender", "append")
        );
    }
}

// CI 파이프라인에서 실행
@SpringBootTest
@AutoConfigureWebTestClient
class IntegrationTestWithBlockHound extends BlockHoundTestConfig {

    @Test
    void allEndpoints_ShouldBeNonBlocking() {
        // 모든 엔드포인트를 실제 요청으로 테스트
        // → BlockHound가 블로킹 호출 감지 시 테스트 실패
        webTestClient.get().uri("/api/users/1")
            .exchange().expectStatus().isOk();

        webTestClient.get().uri("/api/orders")
            .exchange().expectStatus().isOk();
    }
}
```

### 실험 2: Disposable 안전한 관리 패턴

```java
@Component
@RequiredArgsConstructor
public class EventProcessor implements DisposableBean {

    private final EventFluxPublisher publisher;
    private final CompositeDisposable disposables = new CompositeDisposable();

    @PostConstruct
    public void init() {
        // 구독을 CompositeDisposable로 관리
        disposables.add(
            publisher.getOrderEvents()
                .doOnError(e -> log.error("주문 이벤트 처리 오류", e))
                .retry(3)
                .subscribe(this::processOrderEvent)
        );

        disposables.add(
            publisher.getPaymentEvents()
                .subscribe(this::processPaymentEvent)
        );
    }

    @Override
    public void destroy() {
        // 앱 종료 시 모든 구독 해제
        disposables.dispose();
        log.info("모든 이벤트 구독 해제 완료");
    }
}
```

### 실험 3: Reactor 디버그 모드 설정

```java
@Configuration
public class ReactorDebugConfig {

    @Value("${app.reactor.debug:false}")
    private boolean debugEnabled;

    @PostConstruct
    public void configureReactorDebug() {
        if (debugEnabled) {
            // 개발/스테이징 환경: 전체 디버그 (성능 오버헤드 있음)
            Hooks.onOperatorDebug();
            log.warn("Reactor Operator Debug Mode 활성화 — 성능 오버헤드 발생");
        }
        // 운영 환경: ReactorDebugAgent (-javaagent:reactor-tools.jar)로 대체
    }
}

// checkpoint 사용 패턴
@Service
public class OrderService {

    public Mono<Order> createOrder(CreateOrderRequest req) {
        return validateRequest(req)
            .checkpoint("validateRequest")   // 개발 시 추가, 운영 시 제거
            .flatMap(valid -> orderRepo.save(Order.from(valid)))
            .checkpoint("orderRepo.save")
            .flatMap(order -> paymentService.charge(order))
            .checkpoint("paymentService.charge")
            .doOnError(e -> log.error("주문 생성 실패: {}", req, e));
    }
}
```

---

## 📊 성능 비교

```
BlockHound 오버헤드:
  설치 안 함: 0% 오버헤드
  설치:       ~3~8% CPU 오버헤드
  → 개발/테스트 환경: 무조건 설치
  → 운영 환경: 설치하지 않음

ReactorDebugAgent 오버헤드:
  Hooks.onOperatorDebug(): ~20~40% 오버헤드 (개발만)
  ReactorDebugAgent (-javaagent): ~3~8% 오버헤드 (운영 가능)
  checkpoint(): 해당 연산자에만 적용 → 최소 오버헤드

Heap Dump 분석 시간:
  1GB Heap Dump: 수십 초 ~ 수분
  MAT 분석: 누수 의심 객체 자동 추출
  → 운영 중 메모리 누수 진단 시 OOM 이전에 주기적 Heap Dump 권장

메모리 누수 발견 소요 시간 (Heap 증가 속도):
  초당 1MB 누수: 약 8시간 후 8GB → OOM
  진단 권장: 2시간마다 Heap 사용량 추이 모니터링
```

---

## ⚖️ 트레이드오프

```
BlockHound:
  개발 환경: 설치 필수 (CI 통합)
  운영 환경: 성능 오버헤드로 사용 어려움
  → 대안: 운영 환경에서 EventLoop CPU 100% 모니터링

checkpoint():
  장점: 정확한 에러 위치, 선택적 적용
  단점: 코드 분산, 운영에서 제거 필요
  → 프로파일로 관리 (dev 환경만 활성화)

Hooks.onOperatorDebug():
  장점: 전체 파이프라인 디버그
  단점: 큰 성능 오버헤드 → 개발/스테이징만
  → ReactorDebugAgent로 운영 대체

메모리 누수 예방:
  Disposable 항상 관리 (CompositeDisposable)
  Hot Publisher 구독 수 모니터링
  SSE 연결 수 모니터링 (연결 누수)
```

---

## 📌 핵심 정리

```
운영 문제 패턴 핵심:

BlockHound:
  CI에서 반드시 실행 → 블로킹 코드 조기 발견
  운영: 불가 → EventLoop 메트릭으로 간접 감지

메모리 누수:
  Disposable 관리 필수 (CompositeDisposable)
  Hot Publisher 구독 생명주기 관리
  @PreDestroy에서 dispose() 호출
  Heap 모니터링 + MAT 분석

Schedulers 선택:
  boundedElastic: 블로킹 코드 전용
  parallel: CPU 집약 코드만 (블로킹 금지)
  혼용 시 블로킹이 parallel을 점령 → CPU 낭비

디버그:
  개발: Hooks.onOperatorDebug()
  운영: ReactorDebugAgent (-javaagent)
  선택적: checkpoint() (에러 위치 표시)
```

---

## 🤔 생각해볼 문제

**Q1.** `Mono.just(value).subscribe()`처럼 Cold Publisher에서 subscribe()를 호출해도 Disposable 관리가 필요한가요?

<details>
<summary>해설 보기</summary>

`Mono.just(value)`는 즉시 완료(`onComplete`)하므로 구독이 자동으로 종료됩니다. 별도 `Disposable` 관리가 필요 없습니다.

관리가 필요한 경우는:
- **무한 스트림** (`Flux.interval`, Hot Publisher)
- **장기 실행 작업** (타임아웃 없는 외부 API 호출)
- **리소스를 계속 보유하는 구독** (DB 커서, 파일 스트림)

```java
// 관리 불필요: 즉시 완료
Mono.just("hello").subscribe(s -> log.info(s));  // onComplete 후 자동 정리

// 관리 필요: 무한 스트림
Disposable subscription = Flux.interval(Duration.ofSeconds(1))
    .subscribe(t -> log.info("tick: {}", t));
// 나중에 반드시 subscription.dispose()

// 판단 기준: 스트림이 언제 완료되는가?
// 명확한 완료 시점 있음 → 관리 불필요
// 무한하거나 외부 이벤트로 완료 → 관리 필요
```

</details>

---

**Q2.** Reactor 파이프라인에서 예외가 `onError`가 아닌 실제 예외로 던져지는 경우는 어떤 상황인가요?

<details>
<summary>해설 보기</summary>

`subscribe()` 콜백 내에서 예외가 발생하면 `onError`로 처리되지 않고 실제 예외로 던져질 수 있습니다:

```java
flux.subscribe(
    item -> {
        throw new RuntimeException("여기서 발생");
        // → Operators.onErrorDropped()로 전달
        // → 기본적으로 로깅 후 무시 (silent drop!)
    },
    error -> log.error("에러", error)
);
// 위의 RuntimeException은 error 콜백에서 잡히지 않음!
```

Reactor에서 `subscribe()` 이후 콜백의 예외는 `Operators.onErrorDropped()`로 전달되어 기본적으로 로깅되고 무시됩니다.

안전한 패턴:
```java
flux
    .doOnNext(item -> {
        if (problem(item)) throw new RuntimeException();
    })
    .onErrorResume(e -> {
        log.error("에러 처리", e);
        return Mono.empty();
    })
    .subscribe();
// doOnNext의 예외 → onError → onErrorResume에서 처리
```

파이프라인 연산자 내부의 예외는 `onError`로 변환되지만, `subscribe()` 콜백 내 예외는 다르게 처리됩니다.

</details>

---

**Q3.** 운영 환경에서 `Hooks.onOperatorDebug()` 없이 에러 원인을 빠르게 찾는 방법은?

<details>
<summary>해설 보기</summary>

**1. ReactorDebugAgent 사용** (가장 권장):
```
java -javaagent:reactor-tools.jar -jar app.jar
```
바이트코드 계측으로 스택 트레이스를 개선합니다. 성능 오버헤드 3~8%로 운영에서 사용 가능합니다.

**2. 로깅 강화**:
```java
.doOnError(e -> log.error("at step: user-service, op: findById, params: id={}", id, e))
```
에러 발생 위치와 파라미터를 로그에 남기면 스택 트레이스 없이도 원인 추론 가능.

**3. Micrometer + Zipkin 분산 추적**:
각 Reactive 파이프라인의 스팬(Span)을 추적하면 어떤 연산에서 지연/에러가 발생했는지 시각화 가능합니다.

**4. 선택적 checkpoint()**:
에러가 자주 발생하는 구간에만 `checkpoint()`를 추가하여 오버헤드 최소화:
```java
.checkpoint("after-payment", true)  // 성능 정보 포함
```

</details>

---

<div align="center">

**[⬅️ 이전: MVC vs WebFlux 성능 비교](./01-mvc-vs-webflux-benchmark.md)** | **[홈으로 🏠](../README.md)** | **[다음: Reactive Caching ➡️](./03-reactive-caching.md)**

</div>
