# EventLoop에서 블로킹 코드의 위험 — 전체 채널이 멈추는 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- EventLoop 스레드에서 블로킹 코드가 실행되면 정확히 어떤 일이 벌어지는가?
- 동일한 EventLoop에 바인딩된 다른 채널들이 왜 같이 멈추는가?
- `BlockHound`는 어떻게 블로킹 호출을 탐지하는가?
- 블로킹 코드를 `Schedulers.boundedElastic()`으로 오프로딩하는 올바른 패턴은?
- WebFlux에서 어떤 코드가 블로킹으로 분류되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"WebFlux를 도입했는데 왜 요청이 더 느리지?"의 가장 흔한 원인이 바로 EventLoop 블로킹입니다. MVC에서 WebFlux로 마이그레이션할 때 `JdbcTemplate`, `RestTemplate`, 파일 I/O 같은 블로킹 코드를 그대로 두면 EventLoop를 점령하여 처리량이 오히려 감소합니다. 이 원리를 명확히 이해하면 WebFlux 코드 리뷰에서 위험한 패턴을 즉시 발견할 수 있습니다.

---

## 😱 흔한 실수 (Before — EventLoop에서 블로킹 실행)

```
실수 1: JDBC를 EventLoop에서 직접 호출

  @GetMapping("/users")
  public Flux<User> getUsers() {
      return Flux.fromIterable(
          jdbcTemplate.query("SELECT * FROM users", rowMapper)
          // → EventLoop 스레드에서 JDBC 실행
          // → JDBC: 드라이버 → TCP → DB 쿼리 → 결과 대기
          // → EventLoop 점령 시간: DB 응답 시간 (수십~수백 ms)
          // → 같은 EventLoop의 다른 모든 요청 지연
      );
  }

실수 2: RestTemplate을 WebFlux에서 사용

  @GetMapping("/data")
  public Mono<String> getData() {
      return Mono.fromCallable(() ->
          restTemplate.getForObject("http://external-api/data", String.class)
          // RestTemplate = 블로킹 HTTP 클라이언트
          // EventLoop에서 실행 시 HTTP 응답 대기 동안 블로킹
      );
      // subscribeOn(boundedElastic) 없이 사용 시 위험
  }

실수 3: Thread.sleep() / synchronized로 EventLoop 블로킹

  @GetMapping("/slow")
  public Mono<String> slow() {
      return Mono.fromCallable(() -> {
          Thread.sleep(1000);  // EventLoop 1초 블로킹
          return "done";
      });
      // → 이 EventLoop의 모든 채널이 1초 동안 응답 못 함
  }
```

---

## ✨ 올바른 접근 (After — 블로킹 코드 오프로딩)

```
오프로딩 기본 패턴:

블로킹 코드를 Mono.fromCallable()로 래핑 +
subscribeOn(Schedulers.boundedElastic())으로 오프로딩

  Mono.fromCallable(() -> jdbcTemplate.query(...))
      .subscribeOn(Schedulers.boundedElastic())
  // JDBC 실행: boundedElastic 스레드 (블로킹 허용)
  // EventLoop: 자유롭게 다른 채널 처리

  Flux.fromCallable → flatMapMany로 변환:
  Mono.fromCallable(() -> jdbcTemplate.query(...))
      .subscribeOn(Schedulers.boundedElastic())
      .flatMapMany(Flux::fromIterable)

논블로킹 대체제:
  JDBC → R2DBC (완전 논블로킹 DB 드라이버)
  RestTemplate → WebClient (논블로킹 HTTP 클라이언트)
  파일 I/O → Flux.using + Path 비동기 API
```

---

## 🔬 내부 동작 원리

### 1. EventLoop 블로킹의 연쇄 효과 — 단계별

```
시나리오:
  EventLoop-1이 Channel-A, B, C, D 담당 (각 100ms마다 데이터 처리)
  Channel-A의 요청에서 jdbcTemplate.query() 호출 (실행 시간 200ms)

정상 상태 (논블로킹):
  t=0:   EventLoop-1: Channel-A 읽기 → 비동기 처리 시작 → Channel-B 읽기
  t=100: EventLoop-1: Channel-B 처리 → Channel-C 읽기
  t=200: Channel-A 비동기 결과 도착 → EventLoop-1: 응답 쓰기
  → 모든 채널이 정상 처리

블로킹 발생 시:
  t=0:   EventLoop-1: Channel-A 읽기 → jdbcTemplate.query() 실행 시작
         ← 이 순간 EventLoop-1 스레드가 JDBC 소켓 응답 대기로 점령됨
  t=0~200: Channel-B, C, D의 새로운 데이터 도착
           → EventLoop-1이 점령 중 → select() 루프 실행 불가
           → Channel-B, C, D 읽기 지연
  t=200: JDBC 응답 도착 → EventLoop-1 스레드 반환
         → Channel-B, C, D 뒤늦게 처리 (200ms 지연)

결과:
  Channel-A: JDBC 응답 시간 (200ms)
  Channel-B, C, D: JDBC 응답 시간 + 자기 처리 시간 (200ms + N ms)
  → EventLoop-1이 담당하는 모든 채널 영향
  → EventLoop-2, 3은 무관 (다른 채널 담당)
```

### 2. 블로킹 호출의 종류

```
블로킹으로 분류되는 코드:

① 네트워크 I/O 대기:
  JDBC (java.sql.*)       — TCP로 DB에 연결, 쿼리 대기
  RestTemplate            — HTTP 응답 대기
  HttpURLConnection       — HTTP 응답 대기
  Socket.read()           — 소켓 읽기 대기

② 파일 I/O:
  FileInputStream.read()  — 디스크 읽기 대기
  FileOutputStream.write() — 디스크 쓰기 대기
  Files.readAllBytes()    — 전체 파일 읽기

③ 스레드 대기:
  Thread.sleep()          — 스레드 일시 중단
  Object.wait()           — 모니터 대기
  CountDownLatch.await()  — 카운트 대기
  Future.get()            — Future 완료 대기

④ synchronized:
  synchronized 블록이 Lock을 기다리는 경우
  → 다른 스레드가 Lock 해제할 때까지 대기

논블로킹으로 안전한 코드:
  순수 계산 (< 1ms)
  메모리 접근
  Mono/Flux 파이프라인 구성 (실행은 나중에)
  WebClient 호출 (subscribe 전까지 비동기)
```

### 3. BlockHound — 블로킹 탐지 도구

```
BlockHound:
  Reactor 에코시스템의 블로킹 호출 탐지 라이브러리
  ByteBuddy로 Java 메서드를 런타임에 인터셉트
  EventLoop(=논블로킹 스레드)에서 블로킹 호출 감지 시 예외 발생

설치:
  의존성: io.projectreactor.tools:blockhound:1.x.x
  
  BlockHound.install();  // 애플리케이션 시작 시 한 번
  또는
  @SpringBootTest에서 자동 활성화 (BlockHound JUnit extension)

동작 원리:
  1. BlockHound.install() 시 Java Agent처럼 바이트코드 변환
  2. 블로킹 메서드 목록을 내부적으로 유지 (Thread.sleep 등)
  3. 각 블로킹 메서드 진입 시 현재 스레드 확인
  4. NonBlockingThread 표시가 있는 스레드라면 예외 발생
     → Netty의 FastThreadLocalThread, Reactor의 NonBlocking 태그 스레드

탐지 예시:
  EventLoop 스레드에서 Thread.sleep(100) 호출 시:
  BlockingOperationError: Blocking call! java.lang.Thread.sleep
    at com.example.MyService.fetchData(MyService.java:42)
    at ...

허용 목록 설정 (false positive 처리):
  BlockHound.install(b -> b
      .allowBlockingCallsInside(
          "com.example.AllowedService", "specificMethod"
      )
      .disallowBlockingCallsInside(
          "io.r2dbc", "execute"
      )
  );
```

### 4. 올바른 오프로딩 패턴 상세

```
패턴 1: 단순 블로킹 래핑

  // JDBC 단건 조회
  public Mono<User> findById(Long id) {
      return Mono.fromCallable(() ->
          jdbcTemplate.queryForObject(
              "SELECT * FROM users WHERE id = ?",
              userRowMapper, id
          )
      )
      .subscribeOn(Schedulers.boundedElastic());
  }

패턴 2: 블로킹 목록 → Flux

  // JDBC 목록 조회
  public Flux<User> findAll() {
      return Mono.fromCallable(() ->
          jdbcTemplate.query("SELECT * FROM users", userRowMapper)
      )
      .subscribeOn(Schedulers.boundedElastic())
      .flatMapMany(Flux::fromIterable);
  }

패턴 3: 파일 I/O

  public Mono<byte[]> readFile(Path path) {
      return Mono.fromCallable(() -> Files.readAllBytes(path))
          .subscribeOn(Schedulers.boundedElastic());
  }

패턴 4: 기존 Future/CompletableFuture 래핑

  // 이미 비동기인 CompletableFuture → Mono 변환
  // (CompletableFuture는 별도 스레드에서 실행 중)
  public Mono<String> fetchAsync() {
      return Mono.fromFuture(
          CompletableFuture.supplyAsync(this::blockingFetch,
              Executors.newCachedThreadPool())
      );
      // fromFuture: CompletableFuture가 완료되면 EventLoop에 결과 전달
  }

패턴 5: 근본적 해결 — 논블로킹 드라이버 교체

  // JDBC → R2DBC (Spring Data R2DBC)
  public Flux<User> findAll() {
      return userReactiveRepository.findAll();
      // R2DBC: 완전 논블로킹 DB I/O
      // subscribeOn 불필요
  }
```

---

## 💻 실전 코드

### 실험 1: BlockHound로 블로킹 감지

```java
@SpringBootTest
class BlockingDetectionTest {

    @BeforeAll
    static void installBlockHound() {
        BlockHound.install();
    }

    @Test
    void jdbc_In_EventLoop_Should_Be_Detected() {
        // 블로킹 JDBC를 EventLoop에서 실행하면 예외
        Mono<List<User>> badMono = Mono.fromCallable(() ->
            jdbcTemplate.query("SELECT * FROM users", rowMapper)
        );
        // subscribeOn 없으면 EventLoop에서 실행 → BlockingOperationError
        assertThatThrownBy(() ->
            StepVerifier.create(badMono)
                .expectNextCount(1)
                .verifyComplete()
        ).hasCauseInstanceOf(BlockingOperationError.class);
    }

    @Test
    void jdbc_With_BoundedElastic_Should_Work() {
        Mono<List<User>> goodMono = Mono.fromCallable(() ->
            jdbcTemplate.query("SELECT * FROM users", rowMapper)
        ).subscribeOn(Schedulers.boundedElastic());

        StepVerifier.create(goodMono)
            .assertNext(users -> assertThat(users).isNotEmpty())
            .verifyComplete();
    }
}
```

### 실험 2: 블로킹 서비스 안전한 래핑

```java
@Service
public class SafeUserService {
    private final JdbcTemplate jdbc;

    public Mono<User> findById(Long id) {
        return Mono.fromCallable(() ->
            jdbc.queryForObject(
                "SELECT * FROM users WHERE id = ?",
                userRowMapper, id
            )
        )
        .subscribeOn(Schedulers.boundedElastic())
        .onErrorMap(EmptyResultDataAccessException.class,
            e -> new UserNotFoundException("사용자 없음: " + id));
    }

    public Flux<User> findByDepartment(String dept) {
        return Mono.fromCallable(() ->
            jdbc.query(
                "SELECT * FROM users WHERE dept = ?",
                userRowMapper, dept
            )
        )
        .subscribeOn(Schedulers.boundedElastic())
        .flatMapMany(Flux::fromIterable)
        .filter(user -> user.isActive());
    }
}
```

### 실험 3: 블로킹 감지 메트릭 수집

```java
// 처리 시간이 임계값 초과 시 로깅 (간접적 블로킹 감지)
@Component
public class SlowOperationDetector {

    @Around("@annotation(Reactive)")
    public Object detectSlowOps(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        Object result = pjp.proceed();
        long elapsed = System.nanoTime() - start;

        // Reactive 메서드가 즉시 반환되지 않으면 (Mono/Flux 반환 전에 블로킹)
        if (elapsed > 10_000_000) {  // 10ms
            log.warn("메서드 반환이 느림 (블로킹 의심): {} {}ms",
                pjp.getSignature(),
                elapsed / 1_000_000);
        }
        return result;
    }
}
```

---

## 📊 성능 비교

```
EventLoop 블로킹 영향 (8코어, Worker 16개 기준):

시나리오: JDBC 쿼리 200ms, 초당 1000 요청

블로킹 코드 (잘못된 방식):
  각 요청이 EventLoop를 200ms 점령
  EventLoop 1개 → 초당 5개 요청 처리 (1000ms / 200ms)
  16개 EventLoop → 초당 80개 요청 처리
  실제 TPS: 80 (요청 1000개 중 920개 큐 대기)

올바른 오프로딩 (boundedElastic):
  EventLoop: 요청 수신 → 즉시 다음 요청
  boundedElastic: 별도 JDBC 스레드에서 처리
  기본 80개 스레드 (10 × 8코어)
  초당 처리: 80개 JDBC 쿼리 동시 실행 가능
  실제 TPS: ~400 (지연 고려)

이상적인 논블로킹 (R2DBC):
  EventLoop: 요청 → R2DBC 비동기 → 결과 수신
  R2DBC는 EventLoop에서 논블로킹 TCP로 DB 통신
  실제 TPS: 수천 (DB 성능이 한계)

결론:
  블로킹 → 오프로딩: 5배 성능 향상
  오프로딩 → R2DBC: 추가 수십 배 향상 가능
```

---

## ⚖️ 트레이드오프

```
오프로딩 vs 논블로킹 드라이버 전환:

boundedElastic 오프로딩:
  장점: 기존 JDBC 코드 재사용, 전환 비용 낮음
  단점: boundedElastic 스레드 수가 병목 (기본 80개)
        JDBC 연결 풀 크기와 맞춰야 함
        R2DBC보다 처리량 낮음

R2DBC 전환:
  장점: 완전 논블로킹, EventLoop에서 직접 처리, 최고 처리량
  단점: JPA 미지원 (Spring Data R2DBC는 CRUD 수준)
        복잡한 쿼리는 직접 작성
        학습 비용, 마이그레이션 비용

실무 권장:
  신규 서비스, 높은 동시 처리 필요: R2DBC
  기존 서비스 마이그레이션: boundedElastic 오프로딩 → 점진적 R2DBC 전환
  복잡한 쿼리/보고서: JDBC + boundedElastic (R2DBC의 한계)
```

---

## 📌 핵심 정리

```
EventLoop 블로킹 핵심:

왜 위험한가:
  EventLoop 스레드 = 해당 EventLoop 담당 모든 채널의 유일한 처리 스레드
  블로킹 시 = 모든 채널이 대기
  8코어 서버, 16 EventLoop → 1개 EventLoop 블로킹 = TPS 6% 감소

블로킹 코드 종류:
  JDBC, RestTemplate, 파일 I/O, Thread.sleep(), synchronized 대기

탐지:
  BlockHound.install() → NonBlocking 스레드에서 블로킹 호출 시 예외
  개발/테스트 환경에서 반드시 활성화

해결 방법:
  1. Mono.fromCallable() + subscribeOn(boundedElastic()): 임시/빠른 해결
  2. 논블로킹 드라이버 (R2DBC, WebClient): 근본적 해결

오프로딩 패턴:
  Mono.fromCallable(() -> blockingWork())
      .subscribeOn(Schedulers.boundedElastic())
```

---

## 🤔 생각해볼 문제

**Q1.** `Mono.fromCallable()`로 래핑한 코드가 항상 `subscribeOn`에 지정된 스케줄러에서만 실행되나요?

<details>
<summary>해설 보기</summary>

`subscribeOn`이 없으면 `fromCallable`은 `subscribe()`를 호출한 스레드에서 실행됩니다. WebFlux 컨트롤러 반환 시 subscribe를 호출하는 것은 Netty EventLoop 스레드입니다.

```java
// 위험: subscribeOn 없음 → EventLoop에서 블로킹
Mono.fromCallable(() -> jdbcTemplate.query(...))
    .subscribe();  // WebFlux가 EventLoop에서 subscribe

// 안전: boundedElastic에서 블로킹
Mono.fromCallable(() -> jdbcTemplate.query(...))
    .subscribeOn(Schedulers.boundedElastic())
    .subscribe();  // subscribe는 EventLoop, 실행은 boundedElastic
```

`subscribeOn`의 위치는 관계없지만, **반드시 있어야** EventLoop에서 분리됩니다. `publishOn`은 `fromCallable` 실행 스레드에 영향을 주지 않습니다.

</details>

---

**Q2.** `blockingGet()`을 사용하는 Retrofit 같은 라이브러리를 WebFlux에서 쓰려면 어떻게 해야 하나요?

<details>
<summary>해설 보기</summary>

`Mono.fromCallable()` + `subscribeOn(boundedElastic())`으로 래핑하거나, Retrofit의 비동기 `enqueue()` API를 `Mono.create()`로 래핑합니다.

```java
// 방법 1: 블로킹 메서드 래핑 (간단하지만 스레드 사용)
public Mono<User> fetchUser(Long id) {
    return Mono.fromCallable(() ->
        retrofitApi.getUser(id).execute().body()  // 블로킹
    ).subscribeOn(Schedulers.boundedElastic());
}

// 방법 2: 비동기 콜백을 Mono.create로 래핑 (더 효율적)
public Mono<User> fetchUserAsync(Long id) {
    return Mono.create(sink ->
        retrofitApi.getUser(id).enqueue(new Callback<User>() {
            @Override
            public void onResponse(Call<User> call, Response<User> response) {
                sink.success(response.body());
            }
            @Override
            public void onFailure(Call<User> call, Throwable t) {
                sink.error(t);
            }
        })
    );
    // enqueue: 비동기, EventLoop 블로킹 없음
}
```

가능하면 방법 2처럼 콜백 기반 비동기 API를 `Mono.create()`로 변환하는 것이 EventLoop 스레드를 전혀 사용하지 않아 더 효율적입니다.

</details>

---

**Q3.** `BlockHound`를 프로덕션에서 활성화해도 되나요?

<details>
<summary>해설 보기</summary>

일반적으로 **프로덕션에서는 권장하지 않습니다.** 이유는:

**성능 오버헤드**: ByteBuddy로 메서드를 인터셉트하므로, 블로킹 가능성이 있는 모든 메서드 호출에서 스레드 체크 비용이 발생합니다.

**예상치 못한 예외**: 프로덕션에서 처음 발견된 블로킹 호출이 서비스 장애로 이어질 수 있습니다.

권장 사용:

- **개발/CI 환경**: 반드시 활성화 — 블로킹 버그를 조기 발견
- **스테이징**: 선택적으로 활성화 — 프로덕션 유사 환경에서 검증
- **프로덕션**: 비활성화 — 성능 우선

대신 프로덕션에서는:
- Micrometer로 요청 처리 시간 모니터링
- EventLoop 스레드 CPU 사용률 모니터링 (높으면 블로킹 의심)
- `reactor.netty.http.server.accessLog=DEBUG`로 요청별 처리 시간 로깅

</details>

---

<div align="center">

**[⬅️ 이전: ChannelPipeline과 Handler](./03-channel-pipeline-handler.md)** | **[홈으로 🏠](../README.md)** | **[다음: 연결 관리 ➡️](./05-connection-management.md)**

</div>
