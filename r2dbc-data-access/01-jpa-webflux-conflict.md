# JPA와 WebFlux의 충돌 — 블로킹 JDBC 문제

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JPA는 왜 내부적으로 JDBC를 사용하고, 이것이 왜 EventLoop를 블로킹하는가?
- `@Transactional` + JPA는 WebFlux에서 왜 제대로 동작하지 않는가?
- `Schedulers.boundedElastic()`로 JPA를 오프로딩하는 방식의 한계는 무엇인가?
- JPA를 유지할 것인가 R2DBC로 전환할 것인가를 결정하는 기준은 무엇인가?
- MVC와 WebFlux를 혼합 사용(부분 전환)하는 아키텍처는 어떻게 구성하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"WebFlux로 마이그레이션했는데 왜 성능이 오히려 나쁘지?"의 가장 흔한 원인이 JPA입니다. JPA + WebFlux 조합은 서버에 JPA 기술 스택을 그대로 두고 프레임워크만 바꾼 것입니다. EventLoop 스레드를 보호하기 위해 `boundedElastic`으로 감싸면 WebFlux의 이점이 크게 줄어듭니다. 이 문서를 읽고 나면 현재 프로젝트에서 JPA를 유지해야 할지, R2DBC로 전환해야 할지 근거 있게 판단할 수 있습니다.

---

## 😱 흔한 실수 (Before — JPA를 WebFlux에서 그대로 사용)

```
실수 1: JPA Repository를 WebFlux 컨트롤러에서 직접 사용

  @GetMapping("/users")
  public Flux<User> getUsers() {
      return Flux.fromIterable(
          userRepository.findAll()  // 블로킹 JDBC!
          // → EventLoop 스레드에서 JDBC 소켓 대기
          // → 이 EventLoop가 담당하는 모든 요청 지연
      );
  }

실수 2: @Transactional이 Reactive 컨텍스트에서 동작한다고 기대

  @Transactional
  public Mono<Void> createOrder(OrderRequest req) {
      return Mono.fromRunnable(() -> {
          Order order = orderRepository.save(Order.from(req));   // 블로킹
          paymentRepository.save(Payment.from(order));           // 블로킹
          // @Transactional이 ThreadLocal 기반으로 작동
          // Reactive 스케줄러로 스레드 전환 시 트랜잭션 컨텍스트 유실
      });
      // → 트랜잭션이 보장되지 않음
  }

실수 3: Mono.fromCallable + JPA를 subscribeOn 없이 사용

  public Mono<User> findUser(Long id) {
      return Mono.fromCallable(() -> userRepository.findById(id).orElseThrow());
      // subscribeOn(boundedElastic) 없음
      // → EventLoop에서 블로킹 실행
      // BlockHound: "Blocking call!"
  }
```

---

## ✨ 올바른 접근 (After — 충돌 원인 이해 후 전략 선택)

```
현실적 대안 3가지:

1. boundedElastic 오프로딩 (단기 해결):
   Mono.fromCallable(() -> jpaRepo.findById(id).orElseThrow())
       .subscribeOn(Schedulers.boundedElastic())
   → 당장은 동작하지만 boundedElastic 스레드 수가 한계
   → 복잡한 @Transactional 처리 어려움

2. MVC + WebFlux 혼합 (점진적 전환):
   기존 JPA 서비스: Spring MVC 유지
   신규 I/O 집약 서비스: WebFlux + R2DBC로 분리
   → 안정적이지만 두 스택 관리 부담

3. R2DBC 전환 (근본 해결):
   완전 논블로킹 → WebFlux 이점 극대화
   → JPA 기능(지연 로딩, 복잡한 조인) 포기 필요
   → 마이그레이션 비용 있음
```

---

## 🔬 내부 동작 원리

### 1. JPA의 JDBC 의존성 — 논블로킹 불가능한 이유

```
JPA 실행 경로:

  userRepository.findById(id)   ← Spring Data JPA
    → EntityManager.find()      ← Hibernate
      → Session.get()           ← Hibernate Session
        → JDBC Connection.prepareStatement()  ← 블로킹 소켓 연결
        → PreparedStatement.executeQuery()    ← 블로킹 쿼리 실행
        → ResultSet.next()                    ← 블로킹 결과 읽기

JDBC 소켓 대기 (블로킹의 실체):
  executeQuery() 내부:
    OutputStream.write(queryBytes);  // 쿼리 전송
    InputStream.read(resultBytes);   // 결과 대기 ← 여기서 블로킹!
    // read()는 DB가 응답할 때까지 현재 스레드를 점유

EventLoop 스레드에서 발생 시:
  EventLoop-1이 JDBC 결과를 기다리는 동안
  → Channel-A, B, C, D의 모든 이벤트 처리 중단
  → 해당 채널의 요청들이 JDBC 쿼리 시간만큼 지연

근본 이유:
  JDBC는 1999년 설계 → 비동기/논블로킹 개념 없음
  Java NIO의 소켓 채널을 사용하지 않고 블로킹 소켓 사용
  JPA/Hibernate는 JDBC 위에 구축 → 마찬가지로 블로킹
  → JPA를 진짜 논블로킹으로 만드는 것은 불가능
```

### 2. @Transactional과 ThreadLocal — WebFlux에서의 실패

```
Spring @Transactional의 동작 원리 (MVC):

  Thread-1: 서비스 메서드 진입
  Spring AOP 프록시: 트랜잭션 시작
    → Connection 획득 → ThreadLocal에 저장
  Thread-1: Repository 호출 → ThreadLocal에서 Connection 꺼냄 (같은 스레드!)
  Thread-1: 추가 Repository 호출 → 동일 Connection 사용 (트랜잭션 공유!)
  Thread-1: 메서드 종료 → 커밋 or 롤백

WebFlux에서의 문제:

  EventLoop-1: 서비스 Mono 구독 시작
  Spring AOP: 트랜잭션 시작 → Connection → ThreadLocal(EventLoop-1)에 저장
  
  .subscribeOn(Schedulers.boundedElastic()) 발생 시:
  boundedElastic-1: Repository 호출
  ThreadLocal(boundedElastic-1): Connection 없음! (EventLoop-1에 있음)
  → TransactionSynchronizationManager: 트랜잭션 없음으로 인식
  → 새 Connection으로 별도 트랜잭션 시작
  → 원래 트랜잭션과 분리됨 → 원자성 보장 안 됨

결론:
  JPA + @Transactional + WebFlux(스레드 전환) = 트랜잭션 안전 보장 없음
  → R2DBC의 ReactiveTransactionManager는 Reactor Context 기반
  → 스레드 전환에 무관하게 트랜잭션 유지 (Chapter 5-03 참고)
```

### 3. boundedElastic 오프로딩의 한계

```
오프로딩 방식:

  @Repository
  public class UserService {
      private final JpaUserRepository jpaRepo;

      public Mono<User> findById(Long id) {
          return Mono.fromCallable(() ->
              jpaRepo.findById(id).orElseThrow()
          ).subscribeOn(Schedulers.boundedElastic());
      }
  }

겉보기엔 동작하지만 문제가 있음:

  한계 1: 스레드 수 제한
    boundedElastic 최대 스레드: 10 × CPU 코어 수 (8코어 = 80개)
    JDBC 연결 풀 크기: 예) HikariCP 기본 10개
    80개 스레드가 10개 연결을 기다림 → 병목
    초당 100개 이상의 DB 요청 → 스레드 큐 대기 → 레이턴시 증가

  한계 2: Thread-per-Request의 부활
    boundedElastic이 JDBC 스레드 역할 수행
    → 결국 MVC의 스레드 모델과 동일
    → WebFlux를 쓰는 이유 소멸 (복잡성만 증가)

  한계 3: @Transactional 미동작
    boundedElastic으로 스레드 전환 시 ThreadLocal 트랜잭션 컨텍스트 유실
    여러 JPA 작업을 하나의 트랜잭션으로 묶는 것이 매우 복잡

  결론: boundedElastic 오프로딩은 임시방편
    → 간단한 조회, 마이그레이션 기간, 레거시 유지에만 사용
    → 복잡한 비즈니스 로직은 R2DBC로 전환 권장
```

### 4. 전환 판단 기준

```
JPA 유지 판단 기준:
  ✓ 복잡한 JPA 쿼리 (JPQL, QueryDSL 비중 높음)
  ✓ JPA의 캐싱(2차 캐시), Dirty Checking에 의존
  ✓ 레거시 코드베이스, 팀의 JPA 전문성 높음
  ✓ 동시 접속자 수가 적어 블로킹 모델로 충분
  → 선택: Spring MVC 유지, 또는 WebFlux + boundedElastic 오프로딩

R2DBC 전환 판단 기준:
  ✓ 높은 동시 접속 처리 필요 (수천~수만 concurrent)
  ✓ I/O 집약적 서비스 (DB 대기 시간이 길고 요청이 많음)
  ✓ 신규 서비스 (레거시 없음)
  ✓ JPA 기능 포기 가능 (단순 CRUD 중심)
  → 선택: WebFlux + Spring Data R2DBC

혼합 아키텍처 (가장 현실적):
  레거시 서비스 → Spring MVC + JPA (변경 없음)
  신규 API 게이트웨이 → WebFlux (라우팅, 인증)
  신규 I/O 집약 서비스 → WebFlux + R2DBC
  → 서비스별로 독립 선택, 장점만 취함
```

### 5. MVC + WebFlux 혼합 아키텍처

```
동일 JVM에서 MVC와 WebFlux 혼합:
  불가능 → Spring은 하나의 컨텍스트에서 MVC or WebFlux만 활성화

MSA 기반 혼합 (권장):
  user-service     → Spring MVC + JPA (복잡한 비즈니스 로직)
  api-gateway      → Spring WebFlux (라우팅, 인증, 집계)
  notification-svc → Spring WebFlux + R2DBC (실시간, I/O 집약)

모듈 기반 혼합 (단일 JVM):
  spring.main.web-application-type: reactive  # WebFlux 메인
  # 단, JPA를 쓰는 모듈에서는 boundedElastic 오프로딩 필수

마이그레이션 순서 (점진적):
  1단계: WebFlux 도입, JPA + boundedElastic (빠른 전환)
  2단계: 신규 기능은 R2DBC로 작성
  3단계: 조회가 많은 핵심 서비스부터 R2DBC 전환
  4단계: 복잡한 비즈니스 서비스는 마지막에 전환 또는 MVC 유지
```

---

## 💻 실전 코드

### 실험 1: JPA + boundedElastic 오프로딩 (임시 해결)

```java
@Service
@RequiredArgsConstructor
public class LegacyUserService {
    private final JpaUserRepository jpaRepo;  // Spring Data JPA

    // 단건 조회 (간단한 케이스)
    public Mono<User> findById(Long id) {
        return Mono.fromCallable(() ->
            jpaRepo.findById(id)
                .orElseThrow(() -> new UserNotFoundException(id))
        )
        .subscribeOn(Schedulers.boundedElastic())
        .onErrorMap(UserNotFoundException.class, e -> e);
    }

    // 목록 조회
    public Flux<User> findAll() {
        return Mono.fromCallable(jpaRepo::findAll)
            .subscribeOn(Schedulers.boundedElastic())
            .flatMapMany(Flux::fromIterable);
    }

    // 트랜잭션이 필요한 경우 — 복잡하고 안전하지 않음
    // → R2DBC TransactionalOperator 사용 권장
    public Mono<Order> createOrder(OrderRequest req) {
        return Mono.fromCallable(() -> {
            // 이 블록 전체를 하나의 Callable로 감싸야 트랜잭션 유지 가능
            // @Transactional이 아닌 TransactionTemplate 사용
            return transactionTemplate.execute(status -> {
                Order order = orderRepo.save(Order.from(req));
                paymentRepo.save(Payment.from(order));
                return order;
            });
        })
        .subscribeOn(Schedulers.boundedElastic());
    }
}
```

### 실험 2: BlockHound로 JPA 블로킹 감지

```java
@SpringBootTest
class JpaBlockingDetectionTest {

    @BeforeAll
    static void installBlockHound() {
        BlockHound.install();
    }

    @Autowired UserRepository userRepository;

    @Test
    void jpaInEventLoop_ShouldBeBlocked() {
        // JPA를 subscribeOn 없이 사용 → BlockingOperationError
        Mono<User> badMono = Mono.fromCallable(() ->
            userRepository.findById(1L).orElseThrow()
        );

        assertThatThrownBy(() ->
            StepVerifier.create(badMono)
                .expectNextCount(1)
                .verifyComplete()
        ).hasCauseInstanceOf(BlockingOperationError.class);
    }

    @Test
    void jpaWithBoundedElastic_ShouldWork() {
        Mono<User> goodMono = Mono.fromCallable(() ->
            userRepository.findById(1L).orElseThrow()
        ).subscribeOn(Schedulers.boundedElastic());

        StepVerifier.create(goodMono)
            .assertNext(user -> assertThat(user).isNotNull())
            .verifyComplete();
    }
}
```

### 실험 3: 성능 비교 — JPA 오프로딩 vs R2DBC

```java
// 성능 측정 (비교용 코드)
@RestController
public class BenchmarkController {

    private final LegacyUserService jpaService;    // JPA + boundedElastic
    private final ReactiveUserService r2dbcService; // R2DBC

    @GetMapping("/benchmark/jpa/{id}")
    public Mono<User> jpaUser(@PathVariable Long id) {
        return jpaService.findById(id);
        // boundedElastic 스레드 사용 → 스레드 수 한계
        // 1000 동시: 920개 큐 대기 (스레드 80개 한계)
    }

    @GetMapping("/benchmark/r2dbc/{id}")
    public Mono<User> r2dbcUser(@PathVariable Long id) {
        return r2dbcService.findById(id);
        // EventLoop에서 논블로킹 처리 → 스레드 16개로 1000 동시 처리
    }
}
```

---

## 📊 성능 비교

```
JPA 오프로딩 vs R2DBC 처리량 비교:
(DB 쿼리 50ms, 8코어 서버, 동시 요청 1000개)

Spring MVC + JPA (200 스레드):
  동시 처리: 200개 (스레드 수)
  대기 큐: 800개
  초당 처리: 200 / 0.05s = 4,000 req/s (이론)
  실제: ~2,000 req/s (컨텍스트 스위치 등 고려)

WebFlux + JPA + boundedElastic (80 스레드):
  동시 처리: 80개 (boundedElastic 스레드 수)
  대기 큐: 920개
  초당 처리: 80 / 0.05s = 1,600 req/s
  실제: MVC보다 낮음! (복잡성만 증가)

WebFlux + R2DBC (16 EventLoop):
  동시 처리: 논블로킹 (스레드 수 제한 없음)
  실질적 한계: DB 커넥션 풀 크기 (예: 10개)
  초당 처리: DB 처리 속도에 따름 (~5,000-10,000 req/s)
  → 진정한 고처리량

결론:
  WebFlux + JPA 오프로딩 < Spring MVC + JPA (성능 저하!)
  WebFlux + R2DBC >> Spring MVC + JPA (진정한 이점)
```

---

## ⚖️ 트레이드오프

```
JPA 유지 vs R2DBC 전환 트레이드오프:

JPA 유지 + boundedElastic:
  장점: 기존 코드 재사용, 빠른 WebFlux 도입
  단점: WebFlux 이점 반감, 트랜잭션 복잡, 코드 장황

JPA 유지 + Spring MVC:
  장점: 단순, 검증된 패턴, 팀 친숙
  단점: 고동시 처리 한계 (스레드 수가 병목)

R2DBC 전환:
  장점: 완전 논블로킹, 고처리량, WebFlux 이점 극대화
  단점: JPA 기능 포기(지연 로딩, QueryDSL, 복잡한 연관관계)
        학습 비용, 마이그레이션 비용

현실적 권장:
  신규 서비스: WebFlux + R2DBC (처음부터 논블로킹)
  레거시 마이그레이션: MVC 유지 → 점진적 R2DBC 전환
  복잡한 도메인: MVC + JPA (포기 불가한 JPA 기능 있을 때)
```

---

## 📌 핵심 정리

```
JPA-WebFlux 충돌 핵심:

충돌 원인:
  JPA → JDBC → 블로킹 소켓 I/O
  WebFlux EventLoop → 블로킹 금지
  → JPA를 EventLoop에서 실행 = 모든 채널 지연

@Transactional 문제:
  ThreadLocal 기반 → 스레드 전환 시 트랜잭션 컨텍스트 유실
  Reactive에서 안전한 트랜잭션: R2DBC + ReactiveTransactionManager

boundedElastic 오프로딩:
  임시 해결책, 스레드 수 한계 (80개 기본)
  복잡한 트랜잭션 처리 어려움
  MVC 대비 성능 저하 가능

전환 결정 기준:
  높은 동시성, I/O 집약 → R2DBC
  복잡한 도메인, 레거시 → MVC + JPA
  점진적 전환 → boundedElastic 오프로딩 → R2DBC
```

---

## 🤔 생각해볼 문제

**Q1.** Spring Boot 프로젝트에 `spring-boot-starter-data-jpa`와 `spring-boot-starter-webflux`를 동시에 넣으면 어떻게 되나요?

<details>
<summary>해설 보기</summary>

두 의존성이 공존할 수 있지만, Spring Boot는 `webflux`가 있을 때 자동으로 Reactive 웹 서버(Netty)를 사용합니다.

JPA는 정상적으로 빈으로 등록되고 사용할 수 있습니다. 단, JPA를 호출하는 코드가 EventLoop 스레드에서 실행될 경우 블로킹 문제가 발생합니다.

```yaml
# 명시적으로 MVC 강제 (JPA와 안전하게 사용하려면)
spring:
  main:
    web-application-type: servlet  # MVC 강제 (기본은 reactive)
```

`webflux` + `web(MVC)` 동시 사용은 충돌합니다. 둘 중 하나만 사용해야 하며, `webflux`만 있으면 Netty, `web`만 있으면 Tomcat이 기동됩니다. JPA는 두 경우 모두 사용 가능하지만 WebFlux에서는 블로킹 오프로딩이 필요합니다.

</details>

---

**Q2.** `Mono.fromCallable(() -> jpaRepo.findById(id))` + `subscribeOn(boundedElastic())`은 `@Transactional`과 함께 안전하게 사용할 수 있나요?

<details>
<summary>해설 보기</summary>

위험합니다. `@Transactional`은 ThreadLocal 기반이므로, `subscribeOn(boundedElastic())`으로 스레드가 전환되면 트랜잭션 컨텍스트가 유실됩니다.

안전한 패턴은 `TransactionTemplate`을 `fromCallable` 안에서 사용하는 것입니다:

```java
public Mono<Order> createOrder(OrderRequest req) {
    return Mono.fromCallable(() ->
        // 트랜잭션 전체를 하나의 Callable 내에서 처리
        transactionTemplate.execute(status -> {
            Order order = orderRepo.save(Order.from(req));
            paymentRepo.save(Payment.from(order));
            inventoryRepo.decrease(req.getItemId(), req.getQuantity());
            return order;
        })
    )
    .subscribeOn(Schedulers.boundedElastic());
    // 모든 작업이 같은 boundedElastic 스레드에서 실행
    // → ThreadLocal 트랜잭션 컨텍스트 유지
}
```

이 패턴은 동작하지만 코드가 장황하고, 여러 Mono를 조합하는 복잡한 Reactive 파이프라인에서는 적용이 어렵습니다. 복잡한 트랜잭션은 R2DBC + `@Transactional`(Reactive 버전)이 훨씬 안전합니다.

</details>

---

**Q3.** HikariCP(JDBC 연결 풀)과 r2dbc-pool의 최적 연결 수 설정은 어떻게 다른가요?

<details>
<summary>해설 보기</summary>

**HikariCP (JDBC, 블로킹)**:
- 각 스레드가 연결을 점유하는 동안 대기
- `maximumPoolSize` = 스레드 수와 맞추기 (MVC 기본 200 → 연결도 200 근처)
- 공식: `connections = (core_count × 2) + effective_spindle_count` (실제론 부하에 따라)

**r2dbc-pool (논블로킹)**:
- 연결이 쿼리 실행 중에만 점유, 쿼리 완료 즉시 반환
- 훨씬 적은 연결로 많은 요청 처리 가능
- 권장: CPU 코어 수와 같거나 약간 많게 (8코어 → 10~20개)

```yaml
spring:
  r2dbc:
    pool:
      initial-size: 5
      max-size: 20      # HikariCP의 200과 달리 20개로 충분
      max-idle-time: 30m
```

HikariCP를 200으로 설정하고 r2dbc-pool을 동일하게 설정하면 오히려 성능이 저하됩니다. 논블로킹에서는 연결 수가 많다고 좋은 게 아니라 적정 수준이 최적입니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: R2DBC 완전 분해 ➡️](./02-r2dbc-internals.md)**

</div>
