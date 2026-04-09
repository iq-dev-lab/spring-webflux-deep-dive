# 테스트 — StepVerifier와 WebTestClient

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `StepVerifier`로 `Mono`/`Flux`의 신호를 어떻게 단계별로 검증하는가?
- `VirtualTimeScheduler`로 시간 기반 연산자(`delayElements`, `interval`)를 어떻게 빠르게 테스트하는가?
- `TestPublisher`로 커스텀 에러 시나리오를 어떻게 만드는가?
- `WebTestClient`로 WebFlux 컨트롤러를 어떻게 통합 테스트하는가?
- Reactive 코드에서 `.block()`을 테스트에서 쓰면 왜 안 좋은가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

WebFlux 코드를 `block()`으로 테스트하는 것은 가장 흔한 안티패턴입니다. 동작은 하지만 Backpressure, 에러 신호 타이밍, 취소 동작 등 Reactive 파이프라인의 핵심 특성을 전혀 검증하지 못합니다. `StepVerifier`는 신호 수준에서 파이프라인을 검증하는 도구로, WebFlux 코드의 정확한 테스트를 위한 필수 도구입니다.

---

## 😱 흔한 실수 (Before — 테스트를 잘못 작성할 때)

```
실수 1: block()으로 테스트

  @Test
  void test() {
      String result = mono.block();  // 단순히 값만 꺼냄
      assertThat(result).isEqualTo("expected");
  }
  
  문제:
    onError 신호는 예외로 변환되지만 타입이 다를 수 있음
    onComplete 타이밍 검증 불가
    Backpressure 동작 검증 불가
    취소(cancel) 동작 테스트 불가

실수 2: 시간 기반 연산자 테스트에 실제 시간 사용

  @Test
  void testRetry() {
      mono.retryWhen(Retry.backoff(3, Duration.ofSeconds(5)));
      // 실제 15초 대기 → CI 타임아웃
  }

실수 3: 비동기 완료를 기다리지 않음

  @Test
  void test() {
      flux.subscribe(item -> items.add(item));
      assertThat(items).hasSize(3);  // subscribe()가 비동기면 items가 비어있을 수 있음
  }
```

---

## ✨ 올바른 접근 (After — StepVerifier 사용)

```
StepVerifier 기본 패턴:

  StepVerifier.create(publisher)
      .expectSubscription()           // onSubscribe 확인
      .expectNext(값1)                // onNext 확인
      .expectNextMatches(predicate)   // 조건으로 onNext 확인
      .expectError(ExceptionType.class) // onError 확인
      .expectComplete()               // onComplete 확인
      .verify();                      // 실행 및 검증

시간 기반 테스트:
  StepVerifier.withVirtualTime(() -> flux.delayElements(Duration.ofHours(1)))
      .expectSubscription()
      .thenAwait(Duration.ofHours(1))  // 가상 시간 1시간 전진
      .expectNextCount(1)
      .verifyComplete();
  // 실제 시간 소요: ~밀리초
```

---

## 🔬 내부 동작 원리

### 1. StepVerifier 동작 원리

```
StepVerifier.create(publisher)은:
  1. 내부 Subscriber를 생성
  2. publisher를 구독
  3. 예상 신호(expectNext 등)를 큐에 등록
  4. verify() 호출 시 실제 신호와 예상 신호를 순서대로 비교

내부 신호 검증 흐름:
  publisher → onNext("a") → StepVerifier 수신
             기대: expectNext("a") → 일치 ✓
  publisher → onNext("b") → StepVerifier 수신
             기대: expectComplete() → 불일치! → AssertionError

verify() 변형:
  verify():             타임아웃 없음 (무한 대기 가능)
  verify(Duration):     타임아웃 설정 (추천)
  verifyComplete():     expectComplete() + verify() 단축어
  verifyError():        expectError() + verify() 단축어
  verifyErrorMessage(): 에러 메시지 검증 + verify()
```

### 2. StepVerifier 주요 expect 메서드

```
값 검증:
  .expectNext(T... values):     정확한 값 순서대로 검증
  .expectNextMatches(Predicate): 조건으로 검증
  .expectNextCount(n):          n개 수신 확인 (값 무관)
  .expectNextSequence(Iterable): Iterable과 순서대로 비교

신호 검증:
  .expectSubscription():        onSubscribe 확인
  .expectComplete():            onComplete 확인
  .expectError():               onError (타입 무관)
  .expectError(Class):          특정 타입 onError
  .expectErrorMessage(String):  에러 메시지 검증
  .expectErrorMatches(Predicate): 에러 조건 검증

부수 효과:
  .doOnNext(Consumer):          값 수신 시 추가 동작
  .recordWith(Supplier<Collection>): 수신한 값을 컬렉션에 기록
  .consumeNextWith(Consumer):   다음 값을 소비하며 검증

흐름 제어:
  .thenRequest(n):              n개 추가 request (Backpressure 테스트)
  .thenCancel():                구독 취소
  .thenConsumeWhile(Predicate): 조건이 참인 동안 소비
```

### 3. VirtualTimeScheduler — 시간 기반 테스트

```
실제 시간을 소비하는 연산자:
  delayElements, delaySubscription, interval,
  timeout, retryWhen(backoff), cache(Duration)

VirtualTimeScheduler 적용:

  StepVerifier.withVirtualTime(
      () -> Flux.interval(Duration.ofSeconds(1)).take(3)
  )
  .expectSubscription()
  .expectNoEvent(Duration.ofSeconds(1))   // 1초 동안 아무 이벤트 없음 확인 후 전진
  .expectNext(0L)
  .thenAwait(Duration.ofSeconds(1))       // 가상 시간 1초 전진
  .expectNext(1L)
  .thenAwait(Duration.ofSeconds(1))
  .expectNext(2L)
  .verifyComplete();
  // 실제 실행 시간: ~밀리초

주의사항:
  withVirtualTime()의 Supplier 안에서 Flux 생성
  Supplier 밖에서 미리 생성하면 VirtualTimeScheduler와 연결 안 됨

  // 잘못된 사용
  Flux<Long> interval = Flux.interval(Duration.ofSeconds(1));  // 이미 생성됨
  StepVerifier.withVirtualTime(() -> interval)  // 실제 시간 사용!

  // 올바른 사용
  StepVerifier.withVirtualTime(() ->
      Flux.interval(Duration.ofSeconds(1))  // withVirtualTime 안에서 생성
  )
```

### 4. TestPublisher — 커스텀 시나리오

```
TestPublisher<T>: 외부에서 신호를 직접 제어하는 Publisher

  TestPublisher<String> testPublisher = TestPublisher.create();
  Flux<String> flux = testPublisher.flux();

  StepVerifier.create(flux)
      .then(() -> testPublisher.next("a", "b"))  // 신호 발행
      .expectNext("a", "b")
      .then(() -> testPublisher.error(new RuntimeException("실패")))
      .expectError(RuntimeException.class)
      .verify();

비준수(Non-Compliant) Publisher 테스트:
  TestPublisher.createNoncompliant(
      TestPublisher.Violation.ALLOW_NULL  // null 발행 허용 (스펙 위반)
  );
  // 파이프라인이 비정상 Publisher를 어떻게 처리하는지 테스트

활용 시나리오:
  - 특정 타이밍에 에러 발생 테스트
  - request(n) 신호에 맞춰 데이터 발행 테스트
  - 취소(cancel) 후 동작 테스트
```

### 5. WebTestClient — HTTP 레이어 통합 테스트

```
WebTestClient 설정 옵션:

1. 특정 컨트롤러만 테스트 (단위 테스트 수준)
   WebTestClient client = WebTestClient
       .bindToController(new UserController(mockUserService))
       .build();

2. RouterFunction 테스트
   WebTestClient client = WebTestClient
       .bindToRouterFunction(routerFunction)
       .build();

3. 전체 애플리케이션 컨텍스트 (통합 테스트)
   @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
   @AutoConfigureWebTestClient

4. 실제 서버 대상
   WebTestClient client = WebTestClient
       .bindToServer()
       .baseUrl("http://localhost:8080")
       .build();

HTTP 검증 DSL:
  client.get().uri("/users/{id}", 1)
      .header("Authorization", "Bearer token")
      .exchange()
      .expectStatus().isOk()
      .expectHeader().contentType(MediaType.APPLICATION_JSON)
      .expectBody(UserResponse.class)
      .value(user -> assertThat(user.getName()).isEqualTo("Alice"));

  // 여러 항목 (Flux)
  client.get().uri("/users")
      .exchange()
      .expectStatus().isOk()
      .expectBodyList(UserResponse.class)
      .hasSize(3)
      .contains(expectedUser);
```

---

## 💻 실전 코드

### 실험 1: StepVerifier 기본 시나리오

```java
class UserServiceTest {

    @Mock UserRepository userRepository;
    UserService userService;

    @BeforeEach void setUp() {
        userService = new UserService(userRepository);
    }

    @Test
    void findById_존재하는_사용자() {
        User user = new User(1L, "Alice");
        when(userRepository.findById(1L)).thenReturn(Mono.just(user));

        StepVerifier.create(userService.findById(1L))
            .expectNextMatches(u ->
                u.getId().equals(1L) && u.getName().equals("Alice"))
            .expectComplete()
            .verify(Duration.ofSeconds(5));
    }

    @Test
    void findById_존재하지_않는_사용자() {
        when(userRepository.findById(999L)).thenReturn(Mono.empty());

        StepVerifier.create(userService.findById(999L))
            .expectError(UserNotFoundException.class)
            .verify(Duration.ofSeconds(5));
    }

    @Test
    void findAll_Backpressure_테스트() {
        List<User> users = List.of(
            new User(1L, "A"), new User(2L, "B"), new User(3L, "C")
        );
        when(userRepository.findAll()).thenReturn(Flux.fromIterable(users));

        StepVerifier.create(userService.findAll(), 1)  // 처음 1개만 request
            .expectNext(users.get(0))
            .thenRequest(2)                            // 2개 더 request
            .expectNext(users.get(1), users.get(2))
            .verifyComplete();
    }
}
```

### 실험 2: VirtualTimeScheduler로 재시도 테스트

```java
@Test
void retryWithExponentialBackoff_성공까지_재시도() {
    AtomicInteger attempts = new AtomicInteger(0);

    Mono<String> flakyMono = Mono.defer(() -> {
        if (attempts.incrementAndGet() < 3) {
            return Mono.error(new RuntimeException("일시적 장애"));
        }
        return Mono.just("성공");
    })
    .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)));

    StepVerifier.withVirtualTime(() -> flakyMono)
        .expectSubscription()
        .thenAwait(Duration.ofSeconds(1))   // 첫 번째 backoff 대기
        .thenAwait(Duration.ofSeconds(2))   // 두 번째 backoff 대기
        .expectNext("성공")
        .verifyComplete();

    assertThat(attempts.get()).isEqualTo(3);
}
```

### 실험 3: WebTestClient 통합 테스트

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@AutoConfigureWebTestClient
class UserControllerIntegrationTest {

    @Autowired WebTestClient webTestClient;
    @Autowired UserRepository userRepository;

    @BeforeEach
    void setUp() {
        userRepository.deleteAll().block();
        userRepository.saveAll(List.of(
            new User("Alice"), new User("Bob")
        )).blockLast();
    }

    @Test
    void GET_users_모든_사용자_반환() {
        webTestClient.get().uri("/users")
            .exchange()
            .expectStatus().isOk()
            .expectBodyList(UserResponse.class)
            .hasSize(2);
    }

    @Test
    void POST_users_사용자_생성() {
        webTestClient.post().uri("/users")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(new CreateUserRequest("Charlie"))
            .exchange()
            .expectStatus().isCreated()
            .expectBody(UserResponse.class)
            .value(user -> assertThat(user.getName()).isEqualTo("Charlie"));
    }

    @Test
    void GET_users_stream_SSE_테스트() {
        webTestClient.get().uri("/users/stream")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .exchange()
            .expectStatus().isOk()
            .returnResult(UserResponse.class)
            .getResponseBody()
            .take(2)  // 2개만 받고 종료
            .as(StepVerifier::create)
            .expectNextCount(2)
            .verifyComplete();
    }
}
```

---

## 📊 성능 비교

```
테스트 방식별 특성 비교:

방식                 | 신호 검증 | Backpressure | 시간 제어 | 실행 속도
────────────────────┼─────────┼────────────┼─────────┼───────
block() 기반         | 아니오   | 아니오       | 실제 시간 | 빠름
StepVerifier         | 예       | 예           | 실제 시간 | 빠름
StepVerifier+VTime   | 예       | 예           | 가상 시간 | 매우 빠름
WebTestClient        | HTTP 수준 | 아니오      | 실제 시간 | 느림(서버 포함)

VirtualTimeScheduler 효과:
  delayElements(1시간) 테스트
  실제 시간: 3,600,000ms
  VirtualTime: ~10ms
  → 3,600,00배 빠름

WebTestClient vs MockMvc:
  MockMvc: 서블릿 기반 MVC 전용
  WebTestClient: WebFlux + 실제 HTTP 서버 가능
  성능: MockMvc > WebTestClient (서버 오버헤드)
  정확도: WebTestClient > MockMvc (실제 Netty 동작)
```

---

## ⚖️ 트레이드오프

```
테스트 전략 선택:

StepVerifier (단위/통합 테스트):
  장점: 신호 레벨 정밀 검증, Backpressure 테스트, 빠름
  단점: HTTP 레이어 테스트 불가

WebTestClient (통합 테스트):
  장점: 실제 HTTP 요청/응답, 헤더, 상태 코드 검증
  단점: 서버 기동 필요, 느림, 신호 레벨 검증 어려움

병행 사용 권장:
  Service/Repository 레이어: StepVerifier
  Controller/Router: WebTestClient (bindToController)
  End-to-End: WebTestClient + SpringBootTest

block() 사용 제한:
  테스트 setUp/tearDown에서 DB 초기화: 허용 (block())
  실제 테스트 로직: StepVerifier 사용
  프로덕션 코드: 절대 금지
```

---

## 📌 핵심 정리

```
Reactive 테스트 핵심:

StepVerifier:
  publisher의 신호(onNext, onError, onComplete)를 순서대로 검증
  .create(publisher).expectNext(...).expectComplete().verify()
  verify(Duration)으로 타임아웃 설정 권장

VirtualTimeScheduler:
  withVirtualTime(() -> publisher) → thenAwait(duration)
  시간 기반 연산자를 실제 시간 소모 없이 테스트

TestPublisher:
  외부에서 신호를 직접 제어
  특정 타이밍의 에러, 취소 시나리오 테스트에 활용

WebTestClient:
  HTTP 수준 통합 테스트
  bindToController: 단위 수준
  @SpringBootTest + @AutoConfigureWebTestClient: 통합 수준

block() 테스트 사용:
  setUp/tearDown의 초기화 코드에만 제한적 허용
  검증 로직에서는 StepVerifier 필수
```

---

## 🤔 생각해볼 문제

**Q1.** `StepVerifier.create(flux).expectNextCount(1000).verifyComplete()`는 Backpressure를 테스트하는 좋은 방법인가요?

<details>
<summary>해설 보기</summary>

아닙니다. `expectNextCount(1000)`은 내부적으로 `request(Long.MAX_VALUE)`를 사용하므로 Backpressure가 전혀 적용되지 않습니다.

Backpressure를 제대로 테스트하려면:

```java
StepVerifier.create(flux, 10)  // 처음 10개만 request
    .expectNextCount(10)
    .thenRequest(10)            // 10개 더 request
    .expectNextCount(10)
    .thenRequest(Long.MAX_VALUE)
    .expectNextCount(980)
    .verifyComplete();
```

두 번째 인자(10)가 초기 request 크기를 지정합니다. 이렇게 해야 생산자가 Backpressure를 올바르게 처리하는지 검증할 수 있습니다. `expectNextCount(1000)`만으로는 생산자가 request(n) 없이 발행해도 테스트가 통과합니다.

</details>

---

**Q2.** `StepVerifier`에서 `assertNext`와 `expectNextMatches`의 차이는 무엇인가요?

<details>
<summary>해설 보기</summary>

**`expectNextMatches(Predicate<T>)`:**
- Predicate가 `false`를 반환하면 `AssertionError`
- 간결한 조건 검증에 적합

**`assertNext(Consumer<T>)`:**
- Consumer 내부에서 AssertJ 등의 assertion 라이브러리 직접 사용 가능
- Consumer가 예외를 던지면 검증 실패
- 복잡한 객체 검증에 적합

```java
// expectNextMatches — 단순 조건
.expectNextMatches(user -> user.getName().equals("Alice"))

// assertNext — 복잡한 검증 (AssertJ 활용)
.assertNext(user -> {
    assertThat(user.getName()).isEqualTo("Alice");
    assertThat(user.getAge()).isGreaterThan(18);
    assertThat(user.getRoles()).contains("ADMIN");
})
```

실무에서는 단순한 조건은 `expectNextMatches`, 여러 필드를 검증하거나 AssertJ를 활용할 때는 `assertNext`를 사용합니다.

</details>

---

**Q3.** `WebTestClient`로 SSE 스트림 전체를 테스트할 때 무한 스트림이 완료되지 않으면 어떻게 처리하나요?

<details>
<summary>해설 보기</summary>

무한 SSE 스트림은 `onComplete` 신호가 오지 않으므로 `verifyComplete()`을 쓸 수 없습니다. 대신 `.take(n)`으로 일부만 받거나 타임아웃을 설정합니다.

```java
// 방법 1: take(n)으로 일부만 수신
webTestClient.get()
    .uri("/events/stream")
    .accept(TEXT_EVENT_STREAM)
    .exchange()
    .expectStatus().isOk()
    .returnResult(ServerSentEvent.class)
    .getResponseBody()
    .take(5)                          // 5개만 받고 취소
    .as(StepVerifier::create)
    .expectNextCount(5)
    .verifyComplete();                // take 완료 시 complete

// 방법 2: thenCancel()로 명시적 취소
StepVerifier.create(
    webTestClient.get().uri("/events/stream")
        .accept(TEXT_EVENT_STREAM)
        .exchange()
        .returnResult(String.class)
        .getResponseBody()
)
.expectNextCount(3)
.thenCancel()                        // 3개 받은 후 취소
.verify(Duration.ofSeconds(10));
```

두 방법 모두 실제로는 HTTP 연결이 `take` 또는 `cancel` 이후 닫힙니다. WebFlux 서버는 클라이언트 연결 종료를 감지하고 SSE 생산을 중단합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Context](./07-reactor-context.md)** | **[홈으로 🏠](../README.md)**

</div>
