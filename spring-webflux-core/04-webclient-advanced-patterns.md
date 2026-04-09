# WebClient 고급 패턴 — 병렬 호출과 서킷 브레이커

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Mono.zip`으로 여러 외부 API를 병렬 호출할 때 처리 시간은 어떻게 계산되는가?
- `Flux.merge`와 `Flux.concat`은 외부 API 호출에서 어떻게 다르게 동작하는가?
- Resilience4j의 Reactive 서킷 브레이커는 어떻게 WebClient에 통합되는가?
- `WebClient` 인스턴스를 Bean으로 관리할 때 `mutate()`를 어떻게 활용하는가?
- 순차 호출과 병렬 호출을 언제 선택해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

주문 상세 페이지를 보여주기 위해 주문 정보, 결제 정보, 배송 정보를 각각 외부 API로 조회한다고 합시다. 순차 호출하면 300ms + 200ms + 150ms = 650ms입니다. `Mono.zip`으로 병렬 호출하면 max(300, 200, 150) = 300ms입니다. 코드 몇 줄의 차이가 응답 시간 2배 차이를 만듭니다. 서킷 브레이커 없이 외부 API 장애가 발생하면 연결 풀 고갈로 내 서비스도 함께 장애가 납니다.

---

## 😱 흔한 실수 (Before — 순차 호출과 서킷 브레이커 없이)

```
실수 1: 독립적인 API를 순차적으로 호출

  public Mono<OrderDetail> getOrderDetail(Long orderId) {
      return orderClient.getOrder(orderId)        // 300ms
          .flatMap(order ->
              paymentClient.getPayment(orderId)   // 200ms (order 완료 후 시작)
                  .flatMap(payment ->
                      shippingClient.getShipping(orderId)  // 150ms (payment 완료 후)
                          .map(shipping ->
                              OrderDetail.of(order, payment, shipping))
                  )
          );
      // 총 처리 시간: 300 + 200 + 150 = 650ms (독립적인데 순차로!)
  }

실수 2: 서킷 브레이커 없이 외부 API 장애 허용

  // 결제 서비스 장애 시 → 50개 연결 모두 결제 서비스 대기
  // → ConnectionPool Exhaustion → 다른 서비스 요청도 대기
  // → 연쇄 장애 (Cascading Failure)

실수 3: WebClient를 Bean으로 공유하는데 특정 요청에만 헤더 추가

  @Service
  public class UserService {
      private final WebClient webClient;

      public Mono<User> findUser(Long id, String traceId) {
          webClient.mutate()  // 잘못된 사용: mutate()는 빌더 반환
              .defaultHeader("X-Trace-ID", traceId);  // 원본 변경 안 됨!
          return webClient.get().uri("/users/{id}", id)...
          // traceId가 전달 안 됨
      }
  }
```

---

## ✨ 올바른 접근 (After — 병렬 호출과 서킷 브레이커 적용)

```
병렬 호출:
  독립적인 API → Mono.zip으로 병렬 실행
  결과가 순서 중요하면 Mono.zip (동시 실행 + 합산)
  결과 순서 무관하면 Flux.merge (도착 순)

서킷 브레이커:
  외부 API 호출마다 CircuitBreaker 적용
  실패율 50% 이상 → 서킷 OPEN → 즉시 fallback
  대기 없이 즉시 실패 → 연결 풀 보호

WebClient mutate():
  공유 WebClient 기반으로 요청별 추가 설정 적용
  원본 WebClient 불변 유지
  webClient.mutate().defaultHeader(...).build()로 새 인스턴스 생성
```

---

## 🔬 내부 동작 원리

### 1. 병렬 호출 — Mono.zip

```
Mono.zip 처리 흐름:

  Mono<Order>    orderMono    = orderClient.getOrder(id);    // 300ms
  Mono<Payment>  paymentMono  = paymentClient.getPayment(id); // 200ms
  Mono<Shipping> shippingMono = shippingClient.getShipping(id);// 150ms

  Mono.zip(orderMono, paymentMono, shippingMono)
      .map(tuple -> OrderDetail.of(
          tuple.getT1(), tuple.getT2(), tuple.getT3()
      ));

내부 동작:
  subscribe() 시:
    orderMono 구독 시작 → Netty: HTTP 요청 1 전송
    paymentMono 구독 시작 → Netty: HTTP 요청 2 전송 (동시)
    shippingMono 구독 시작 → Netty: HTTP 요청 3 전송 (동시)
    
    t=0:   세 요청 동시 전송
    t=150: shipping 응답 도착 → zip 내부 버퍼에 저장 (대기)
    t=200: payment 응답 도착 → zip 내부 버퍼에 저장 (대기)
    t=300: order 응답 도착 → 세 값 모두 도착 → tuple 생성 → map 실행
    
    총 처리 시간: max(300, 200, 150) = 300ms (순차 650ms의 46%)

하나라도 에러 시:
  → 전체 Mono.zip이 onError
  → 나머지 진행 중 요청도 취소 (cancel 신호)
  → fallback 처리 필요

  Mono.zip(orderMono, paymentMono.onErrorReturn(Payment.empty()), shippingMono)
  // payment 에러 시 기본값으로 진행 (partial 결과 허용)
```

### 2. 순차 vs 병렬 선택 기준

```
병렬 호출이 적합한 경우:
  - 각 API 호출이 독립적 (앞 결과에 의존 없음)
  - 응답 시간이 중요한 사용자 요청
  - 조회 API (읽기 전용)

  주문 상세 = zip(주문 조회, 결제 조회, 배송 조회)
  대시보드 = zip(매출 통계, 주문 통계, 사용자 통계)

순차 호출이 적합한 경우:
  - 앞 결과가 다음 호출의 파라미터로 사용됨
  - 트랜잭션 의미가 있음 (앞 실패 시 다음 불필요)
  - 쓰기 작업의 순서 보장 필요

  주문 생성 = 재고 확인 → 주문 생성 → 결제 처리 → 배송 요청
  (각 단계가 앞 단계의 성공을 전제)

혼합 패턴:
  // 주문 조회 후, 결제/배송을 병렬로
  orderClient.getOrder(id)
      .flatMap(order -> Mono.zip(
          paymentClient.getPayment(order.getPaymentId()),
          shippingClient.getShipping(order.getShippingId())
      ).map(tuple -> OrderDetail.of(order, tuple.getT1(), tuple.getT2())))
```

### 3. Flux.merge vs Flux.concat — 여러 소스 병합

```
시나리오: 여러 소스에서 사용자 목록 조회

Flux.merge (병렬, 도착 순):
  Flux<User> usersA = serviceA.getUsers();  // 300ms
  Flux<User> usersB = serviceB.getUsers();  // 200ms

  Flux.merge(usersA, usersB)
  // A, B 동시 구독
  // B가 먼저 완료 → B 결과 먼저 방출
  // A 완료 → A 결과 방출
  // 순서: B 결과, A 결과 (도착 순)
  // 총 시간: max(300, 200) = 300ms

Flux.concat (순차):
  Flux.concat(usersA, usersB)
  // A 완료 후 B 시작
  // 순서: A 결과, B 결과 (소스 순서 보장)
  // 총 시간: 300 + 200 = 500ms

mergeDelayError (에러 지연):
  Flux.mergeDelayError(1, usersA, usersB)
  // 한 소스에서 에러가 나도 나머지 계속 처리
  // 모든 소스 완료 후 에러 전파
  // 부분 결과라도 수집할 때 유용
```

### 4. Resilience4j Reactive 서킷 브레이커

```
서킷 브레이커 3가지 상태:
  CLOSED:   정상 (요청 통과)
  OPEN:     장애 감지 (요청 즉시 실패, fallback 실행)
  HALF_OPEN: 복구 확인 (일부 요청 통과, 성공률 확인)

Resilience4j + WebFlux 통합:

  의존성: resilience4j-spring-boot3 + resilience4j-reactor

  application.yml:
    resilience4j:
      circuitbreaker:
        instances:
          paymentService:
            slidingWindowSize: 10       # 최근 10번 호출 기준
            failureRateThreshold: 50    # 50% 실패율 시 OPEN
            waitDurationInOpenState: 10s # OPEN 유지 10초
            permittedNumberOfCallsInHalfOpenState: 3

  서비스에서 사용:
    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public Mono<PaymentResult> charge(Order order) {
        return paymentWebClient.post()
            .uri("/charge")
            .bodyValue(order)
            .retrieve()
            .bodyToMono(PaymentResult.class);
    }
    
    public Mono<PaymentResult> paymentFallback(Order order, Throwable t) {
        log.warn("결제 서비스 서킷 오픈, fallback 실행: {}", t.getMessage());
        return Mono.just(PaymentResult.pending(order.getId()));
    }

  프로그래밍 방식:
    ReactiveCircuitBreaker cb = circuitBreakerFactory
        .create("paymentService");

    return cb.run(
        paymentWebClient.post().uri("/charge")
            .bodyValue(order)
            .retrieve()
            .bodyToMono(PaymentResult.class),
        throwable -> Mono.just(PaymentResult.pending(order.getId()))
    );
```

### 5. WebClient mutate() — 공유 인스턴스 커스터마이징

```
WebClient.mutate():
  기존 WebClient 설정을 기반으로 새 WebClient 생성
  원본 불변 유지

  @Service
  public class UserService {
      private final WebClient webClient;  // 공유 인스턴스

      public Mono<User> findUser(Long id) {
          // 기본 WebClient 사용
          return webClient.get()
              .uri("/users/{id}", id)
              .retrieve()
              .bodyToMono(User.class);
      }

      public Mono<User> findUserWithTrace(Long id, String traceId) {
          // 요청별 추가 헤더를 가진 새 WebClient 생성
          WebClient traced = webClient.mutate()
              .defaultHeader("X-Trace-ID", traceId)
              .build();  // 원본 webClient 변경 없음

          return traced.get()
              .uri("/users/{id}", id)
              .retrieve()
              .bodyToMono(User.class);
      }

      public Mono<User> findUserWithAuth(Long id, String token) {
          // mutate는 새 WebClient 반환 — 원본 안전
          return webClient.mutate()
              .defaultHeader(HttpHeaders.AUTHORIZATION, "Bearer " + token)
              .build()
              .get()
              .uri("/users/{id}", id)
              .retrieve()
              .bodyToMono(User.class);
      }
  }

ExchangeFilterFunction으로 동적 헤더:
  WebClient traced = webClient.mutate()
      .filter(ExchangeFilterFunction.ofRequestProcessor(request ->
          Mono.deferContextual(ctx ->
              Mono.just(ClientRequest.from(request)
                  .header("X-Trace-ID", ctx.getOrDefault("traceId", ""))
                  .build())
          )
      ))
      .build();
  // Reactor Context의 traceId를 자동으로 헤더에 주입
```

---

## 💻 실전 코드

### 실험 1: 주문 상세 병렬 조회

```java
@Service
@RequiredArgsConstructor
public class OrderDetailService {
    private final OrderClient orderClient;
    private final PaymentClient paymentClient;
    private final ShippingClient shippingClient;
    private final ReactiveCircuitBreakerFactory cbFactory;

    public Mono<OrderDetail> getOrderDetail(Long orderId) {
        ReactiveCircuitBreaker cb = cbFactory.create("orderDetail");

        Mono<Order> order = orderClient.findById(orderId)
            .timeout(Duration.ofSeconds(3));

        Mono<Payment> payment = paymentClient.findByOrderId(orderId)
            .timeout(Duration.ofSeconds(3))
            .onErrorReturn(Payment.empty());  // 결제 실패 시 빈 결과로

        Mono<Shipping> shipping = shippingClient.findByOrderId(orderId)
            .timeout(Duration.ofSeconds(3))
            .onErrorReturn(Shipping.empty());  // 배송 실패 시 빈 결과로

        return cb.run(
            Mono.zip(order, payment, shipping)
                .map(tuple -> OrderDetail.of(
                    tuple.getT1(), tuple.getT2(), tuple.getT3()
                )),
            throwable -> {
                log.error("주문 상세 조회 실패: {}", orderId, throwable);
                return Mono.error(new OrderDetailException(orderId));
            }
        );
    }
}
```

### 실험 2: 대량 주문 병렬 처리 (동시 수 제한)

```java
public Flux<ProcessResult> processOrders(List<Long> orderIds) {
    return Flux.fromIterable(orderIds)
        .flatMap(
            id -> processOrder(id)
                .onErrorResume(e -> {
                    log.error("주문 처리 실패: {}", id, e);
                    return Mono.just(ProcessResult.failed(id));
                }),
            5  // 최대 5개 동시 처리
        );
}

private Mono<ProcessResult> processOrder(Long orderId) {
    return orderClient.findById(orderId)
        .flatMap(order -> paymentClient.charge(order))
        .flatMap(payment -> shippingClient.request(payment))
        .map(shipping -> ProcessResult.success(orderId));
}
```

### 실험 3: ExchangeFilterFunction으로 공통 처리

```java
// 로깅 필터 + 인증 필터 조합
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder
        .baseUrl("https://api.example.com")
        .filter(loggingFilter())
        .filter(authFilter())
        .build();
}

private ExchangeFilterFunction loggingFilter() {
    return ExchangeFilterFunction.ofRequestProcessor(request -> {
        log.debug("요청: {} {}", request.method(), request.url());
        return Mono.just(request);
    });
}

private ExchangeFilterFunction authFilter() {
    return ExchangeFilterFunction.ofRequestProcessor(request ->
        tokenService.getAccessToken()
            .map(token -> ClientRequest.from(request)
                .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
                .build())
    );
}
```

---

## 📊 성능 비교

```
순차 vs 병렬 처리 시간 비교:

3개 독립 API (각각 300ms, 200ms, 150ms):

  순차 flatMap 체인:    650ms (300+200+150)
  Mono.zip 병렬:         300ms (max값)
  개선율:               54% 감소

10개 독립 API (평균 200ms):
  순차:   2,000ms
  병렬:   ~250ms (가장 느린 것 + 약간의 오버헤드)
  개선율: 87% 감소

서킷 브레이커 효과 (외부 API 장애 시):
  서킷 브레이커 없음: 모든 요청이 타임아웃(10초) 대기
    → 연결 풀 고갈 → 연쇄 장애
  서킷 브레이커 있음:
    OPEN 시: fallback 즉시 실행 (~1ms)
    → 연결 풀 보호 → 내 서비스 정상 운영

연결 풀 보호 효과:
  maxConnections=50, 타임아웃 10초, 초당 100 요청
  서킷 없음: 50개 연결 10초 점유 → 950개 큐 대기 → 전체 지연
  서킷 있음: 장애 감지 즉시 fallback → 연결 풀 여유 → 다른 서비스 정상
```

---

## ⚖️ 트레이드오프

```
병렬 호출 트레이드오프:
  장점: 응답 시간 대폭 감소
  단점: 하나 실패 시 전체 실패 (zip의 특성)
  해결: 실패 허용하는 API는 onErrorReturn으로 기본값

서킷 브레이커 트레이드오프:
  장점: 연쇄 장애 방지, 빠른 실패(fail fast)
  단점: 설정 복잡 (slidingWindowSize, failureRateThreshold 튜닝 필요)
        OPEN 상태에서 실제 서비스가 복구되어도 대기 필요

mutate() vs 새 WebClient 생성:
  mutate(): 기존 설정 상속, 연결 풀 공유 (효율적)
  새 create(): 독립적 설정, 독립적 연결 풀 (격리되지만 비효율)
  → mutate() 권장 (연결 풀 재사용)

Resilience4j vs Reactive 직접 구현:
  Resilience4j: 표준 패턴, Actuator 통합, 설정 파일 기반
  직접 구현 (retryWhen + onErrorResume): 단순한 경우 충분
  → 복잡한 정책 필요 시 Resilience4j
```

---

## 📌 핵심 정리

```
WebClient 고급 패턴 핵심:

병렬 호출:
  독립 API → Mono.zip(a, b, c) → 총 시간 = max(각 시간)
  하나 실패 허용 → onErrorReturn으로 기본값
  순서 무관 스트림 → Flux.merge (병렬)
  순서 보장 스트림 → Flux.concat (순차)

서킷 브레이커:
  Resilience4j Reactive + @CircuitBreaker or ReactiveCircuitBreaker
  OPEN 시 fallback 즉시 실행 → 연결 풀 보호
  slidingWindowSize, failureRateThreshold 튜닝 필수

mutate():
  공유 WebClient 기반으로 요청별 설정 변경
  원본 불변 → 연결 풀 공유 유지
  ExchangeFilterFunction으로 동적 처리

선택 기준:
  독립 API = 병렬 (Mono.zip)
  의존 API = 순차 (flatMap 체인)
  외부 서비스 = 서킷 브레이커 필수
```

---

## 🤔 생각해볼 문제

**Q1.** `Mono.zip`에서 하나의 Mono가 에러를 반환했을 때, 이미 완료된 다른 Mono의 결과는 어떻게 되나요?

<details>
<summary>해설 보기</summary>

기본 `Mono.zip`은 하나라도 에러가 발생하면 전체가 에러가 됩니다. 이미 완료된 다른 Mono의 결과는 버려지고, 아직 완료되지 않은 Mono는 취소(cancel) 신호를 받습니다.

```java
Mono.zip(
    Mono.just("OK"),                          // 즉시 완료
    Mono.error(new RuntimeException("실패")), // 에러
    Mono.delay(Duration.ofSeconds(1)).map(d -> "delayed")  // 아직 진행 중
)
// → "OK"의 결과는 버려짐
// → "delayed"는 취소됨
// → RuntimeException("실패")이 zip 전체의 onError
```

부분 성공을 허용하려면:
```java
Mono.zip(
    monoA,
    monoB.onErrorReturn("B-default"),  // B 실패 시 기본값으로
    monoC.onErrorReturn("C-default")   // C 실패 시 기본값으로
)
// A가 에러면 전체 실패
// B나 C가 에러면 기본값으로 zip 진행
```

</details>

---

**Q2.** Resilience4j 서킷 브레이커에서 `slidingWindowType`을 `COUNT_BASED`와 `TIME_BASED` 중 어떻게 선택하나요?

<details>
<summary>해설 보기</summary>

**COUNT_BASED** (기본): 최근 N번의 호출을 기준으로 실패율 계산.
- 호출 빈도가 일정하거나 높을 때 적합
- 트래픽이 낮을 때 N번 호출이 오래 걸릴 수 있음

**TIME_BASED**: 최근 N초의 호출을 기준으로 실패율 계산.
- 트래픽이 불규칙할 때 적합 (야간 트래픽 감소 등)
- 시간 기반으로 더 직관적

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        # 최근 10번 호출 기준
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10

      reportService:
        # 최근 60초 기준 (트래픽이 낮은 서비스)
        slidingWindowType: TIME_BASED
        slidingWindowSize: 60
```

MSA에서 일반적으로 핵심 API는 `COUNT_BASED`, 배치성/주기적 서비스는 `TIME_BASED`를 권장합니다.

</details>

---

**Q3.** `Flux.merge`와 `Flux.mergeDelayError`는 에러 처리에서 어떻게 다른가요?

<details>
<summary>해설 보기</summary>

**`Flux.merge`**: 어느 소스에서든 에러가 발생하면 즉시 전체 스트림에 에러 전파, 나머지 소스 취소.

**`Flux.mergeDelayError(prefetch, sources...)`**: 에러가 발생해도 다른 소스는 계속 진행, 모든 소스가 완료된 후 수집된 에러를 전파.

```java
Flux<String> sourceA = Flux.just("A1", "A2")
    .concatWith(Flux.error(new RuntimeException("A 실패")));
Flux<String> sourceB = Flux.just("B1", "B2", "B3");

// Flux.merge: A 에러 발생 즉시 B도 중단
Flux.merge(sourceA, sourceB)
    .subscribe(
        s -> log.info(s),
        e -> log.error("에러: {}", e.getMessage())
    );
// 출력: A1, A2, B1(순서 불확실), 에러: A 실패 (B2, B3 누락 가능)

// Flux.mergeDelayError: A 에러 지연, B 끝까지 처리
Flux.mergeDelayError(1, sourceA, sourceB)
    .subscribe(
        s -> log.info(s),
        e -> log.error("에러: {}", e.getMessage())
    );
// 출력: A1, A2, B1, B2, B3 (B 완료 보장), 이후 에러: A 실패
```

여러 소스 중 일부 실패해도 나머지 결과는 수집하고 싶을 때 `mergeDelayError`가 유용합니다.

</details>

---

<div align="center">

**[⬅️ 이전: WebClient 완전 분해](./03-webclient-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: SSE와 WebSocket ➡️](./05-streaming-sse-websocket.md)**

</div>
