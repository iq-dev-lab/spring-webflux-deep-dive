# 마이크로서비스에서 WebFlux — 서비스 간 호출 체인

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 서비스 간 `WebClient` 호출 체인에서 에러는 어떻게 전파되고 처리하는가?
- Resilience4j Reactive Circuit Breaker는 WebClient와 어떻게 통합하는가?
- Micrometer Tracing과 Reactor Context를 이용해 TraceId를 어떻게 전파하는가?
- Reactive 기반 Event-Driven 아키텍처에서 서비스 간 비동기 통신은 어떻게 구현하는가?
- 서비스 간 타임아웃과 폴백은 어떻게 계층적으로 설정하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

MSA에서 하나의 요청이 여러 서비스를 거치는 호출 체인에서 장애는 연쇄적으로 전파됩니다. 결제 서비스가 느려지면 주문 서비스가 기다리고, 주문 서비스가 느려지면 API 게이트웨이가 막힙니다. Circuit Breaker, 타임아웃, 폴백의 세 가지 방어막을 Reactive 파이프라인에 올바르게 적용하면 하나의 서비스 장애가 전체 시스템으로 전파되는 것을 막을 수 있습니다.

---

## 😱 흔한 실수 (Before — MSA 연쇄 장애 방치)

```
실수 1: 타임아웃 없는 서비스 간 호출

  return orderWebClient.get()
      .uri("/orders/{id}", orderId)
      .retrieve()
      .bodyToMono(Order.class);
  // 타임아웃 없음 → orderService 무한 대기
  // orderService 연결 풀 고갈 → apiGateway도 막힘

실수 2: 에러 전파 없이 삼키기

  return orderService.getOrder(orderId)
      .onErrorResume(e -> Mono.empty());  // 에러 무시
  // 상위 서비스가 주문 정보 없이 처리 → 데이터 불일치

실수 3: TraceId가 서비스 경계에서 소실

  // serviceA에서 생성된 traceId가 serviceB로 전달 안 됨
  webClient.get().uri("/api").retrieve()...
  // → 분산 추적 불가, 서비스 B의 로그와 매핑 불가능
```

---

## ✨ 올바른 접근 (After — 탄력적 서비스 간 호출)

```
서비스 간 호출 방어 레이어:

1. Timeout: 최대 응답 대기 시간 제한
   .timeout(Duration.ofSeconds(5))

2. Retry: 일시적 장애 자동 복구
   .retryWhen(Retry.backoff(2, Duration.ofMillis(500)))

3. Circuit Breaker: 반복 장애 시 즉시 실패
   circuitBreakerFactory.create("order-service")
       .run(apiCall, throwable -> fallback)

4. Fallback: 서비스 불가 시 대안
   .onErrorReturn(Order.empty())

계층 적용 순서:
  cb.run(
      apiCall.timeout(3s).retryWhen(backoff),
      throwable -> fallback
  )
```

---

## 🔬 내부 동작 원리

### 1. 서비스 간 호출 체인 에러 전파

```
호출 체인 구조:
  API Gateway → Order Service → Payment Service
                             → Inventory Service

에러 전파 흐름:
  Payment Service: 503 → WebClientResponseException.ServiceUnavailable
  Order Service: payment 에러 수신
    → onError → Circuit Breaker
    → Circuit Open → Fallback (빈 결제 응답)
    → Order 생성 시 결제 없이 PENDING 상태로 처리
  API Gateway: Order 응답 정상 수신

에러 변환 패턴:
  // 하위 서비스 에러를 도메인 예외로 변환
  return paymentWebClient.post()
      .uri("/charge")
      .bodyValue(order)
      .retrieve()
      .onStatus(status -> status.is5xxServerError(),
          response -> response.createException()
              .flatMap(e -> Mono.error(
                  new PaymentServiceException("결제 서비스 불가", e)
              ))
      )
      .bodyToMono(PaymentResult.class)
      .onErrorMap(WebClientResponseException.NotFound.class,
          e -> new PaymentNotFoundException("결제 정보 없음")
      );
```

### 2. Resilience4j Reactive Circuit Breaker

```java
// 설정
application.yml:
  resilience4j:
    circuitbreaker:
      instances:
        payment-service:
          sliding-window-size: 10
          failure-rate-threshold: 50    # 50% 실패율 시 OPEN
          wait-duration-in-open-state: 10s
          permitted-calls-in-half-open: 3
          slow-call-rate-threshold: 80  # 80%가 느리면 OPEN
          slow-call-duration-threshold: 3s  # 3초 초과 = 느린 호출

// 서비스 통합
@Service
@RequiredArgsConstructor
public class OrderService {
    private final ReactiveCircuitBreakerFactory cbFactory;
    private final WebClient paymentWebClient;

    public Mono<PaymentResult> chargePayment(Order order) {
        ReactiveCircuitBreaker cb = cbFactory.create("payment-service");

        return cb.run(
            // 실제 호출 (서킷이 닫혀있을 때)
            paymentWebClient.post()
                .uri("/charge")
                .bodyValue(order)
                .retrieve()
                .bodyToMono(PaymentResult.class)
                .timeout(Duration.ofSeconds(3)),

            // Fallback (서킷이 열렸거나 에러 시)
            throwable -> {
                log.warn("결제 서비스 불가, 폴백 실행: {}", throwable.getMessage());
                return Mono.just(PaymentResult.pending(order.getId()));
            }
        );
    }
}

// 서킷 상태 모니터링
@Component
public class CircuitBreakerMonitor {
    private final CircuitBreakerRegistry registry;

    @PostConstruct
    public void registerListeners() {
        registry.circuitBreaker("payment-service")
            .getEventPublisher()
            .onStateTransition(event ->
                log.warn("CB 상태 변경: {} → {}",
                    event.getStateTransition().getFromState(),
                    event.getStateTransition().getToState())
            )
            .onCallNotPermitted(event ->
                log.warn("CB OPEN: 요청 차단됨")
            );
    }
}
```

### 3. Micrometer Tracing — TraceId 전파

```java
// Spring Boot 3 + Micrometer Tracing 자동 설정
// WebClient에 자동으로 TraceId 헤더 주입 (설정 없이)

// application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% 트레이싱
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans

// 수동 TraceId 전파 (자동 설정이 안 되는 경우)
@Configuration
public class TracingWebClientConfig {

    @Bean
    public WebClient tracedWebClient(
            WebClient.Builder builder,
            Tracer tracer) {
        return builder
            .filter(ExchangeFilterFunction.ofRequestProcessor(request -> {
                Span currentSpan = tracer.currentSpan();
                if (currentSpan != null) {
                    return Mono.just(ClientRequest.from(request)
                        .header("X-B3-TraceId",
                            currentSpan.context().traceId())
                        .header("X-B3-SpanId",
                            currentSpan.context().spanId())
                        .build());
                }
                return Mono.just(request);
            }))
            .build();
    }
}

// Reactor Context를 통한 TraceId 전파
@Component
@Order(-50)
public class TraceContextFilter implements WebFilter {
    private final Tracer tracer;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        Span span = tracer.nextSpan()
            .name("http:" + exchange.getRequest().getPath())
            .start();

        return chain.filter(exchange)
            .contextWrite(ctx ->
                ctx.put("currentSpan", span)
                   .put("traceId", span.context().traceId())
            )
            .doFinally(signal -> span.end());
    }
}
```

### 4. 계층적 타임아웃 설정

```
타임아웃 계층:
  클라이언트 (브라우저): 30초
  API Gateway: 20초
  Order Service: 10초
  Payment Service: 5초

  → 하위 서비스 타임아웃 < 상위 서비스 타임아웃
  → 하위가 먼저 타임아웃 → 상위가 적절히 처리 가능

Order Service 설정:
  @Service
  public class OrderService {

      // Payment 호출: 5초 타임아웃 + 2회 재시도
      private Mono<PaymentResult> callPayment(Order order) {
          return paymentWebClient.post()
              .uri("/charge")
              .bodyValue(order)
              .retrieve()
              .bodyToMono(PaymentResult.class)
              .timeout(Duration.ofSeconds(5))         // ← 하위 서비스 타임아웃
              .retryWhen(Retry.backoff(2, Duration.ofMillis(300))
                  .filter(e -> !(e instanceof TimeoutException))
              )
              .onErrorReturn(PaymentResult.pending(order.getId())); // 폴백
      }

      // Order 전체 처리: 10초 타임아웃
      @GetMapping("/orders/{id}")
      public Mono<OrderDetail> getOrderDetail(@PathVariable Long id) {
          return Mono.zip(
              orderRepo.findById(id),
              callPayment(Order.of(id))  // 내부 5초 타임아웃 포함
          )
          .map(tuple -> OrderDetail.of(tuple.getT1(), tuple.getT2()))
          .timeout(Duration.ofSeconds(10));  // ← 전체 처리 타임아웃
      }
  }
```

### 5. Event-Driven 비동기 서비스 간 통신

```java
// Kafka + Reactor 기반 이벤트 발행/구독

// 이벤트 발행 (Order Service)
@Service
@RequiredArgsConstructor
public class OrderEventPublisher {
    private final ReactiveKafkaProducerTemplate<String, OrderEvent> producer;

    public Mono<Void> publish(OrderEvent event) {
        return producer.send("orders.topic",
            event.orderId().toString(), event)
            .doOnSuccess(r ->
                log.info("이벤트 발행: {} offset={}", event.type(),
                    r.recordMetadata().offset())
            )
            .then();
    }
}

// 이벤트 구독 (Payment Service)
@Service
@RequiredArgsConstructor
public class PaymentEventConsumer implements DisposableBean {

    private final ReactiveKafkaConsumerTemplate<String, OrderEvent> consumer;
    private Disposable subscription;

    @PostConstruct
    public void consume() {
        subscription = consumer.receiveAutoAck()
            .flatMap(record ->
                processPayment(record.value())
                    .onErrorResume(e -> {
                        log.error("결제 처리 실패: {}", record.value(), e);
                        return Mono.empty();  // 개별 이벤트 실패는 계속 진행
                    }),
                5  // 동시 처리 5개
            )
            .subscribe();
    }

    @Override
    public void destroy() {
        if (subscription != null) subscription.dispose();
    }
}
```

---

## 💻 실전 코드

### 실험 1: 탄력적 서비스 호출 완전 패턴

```java
@Service
@RequiredArgsConstructor
public class ResilientOrderService {

    private final ReactiveCircuitBreakerFactory cbFactory;
    private final WebClient inventoryWebClient;
    private final WebClient paymentWebClient;

    public Mono<OrderResult> createOrder(CreateOrderRequest req) {
        ReactiveCircuitBreaker inventoryCb = cbFactory.create("inventory");
        ReactiveCircuitBreaker paymentCb = cbFactory.create("payment");

        // 1. 재고 확인 (서킷 브레이커 + 타임아웃)
        Mono<InventoryResult> inventory = inventoryCb.run(
            inventoryWebClient.post()
                .uri("/check")
                .bodyValue(req.items())
                .retrieve()
                .bodyToMono(InventoryResult.class)
                .timeout(Duration.ofSeconds(3)),
            e -> Mono.just(InventoryResult.available())  // 재고 확인 실패 시 낙관적 처리
        );

        // 2. 주문 생성 (재고 확인 후)
        return inventory
            .filter(InventoryResult::isAvailable)
            .switchIfEmpty(Mono.error(new InsufficientStockException()))
            .flatMap(inv -> orderRepository.save(Order.from(req)))
            .flatMap(order -> {
                // 3. 결제 (서킷 브레이커 + 재시도)
                return paymentCb.run(
                    paymentWebClient.post()
                        .uri("/charge")
                        .bodyValue(PaymentRequest.from(order))
                        .retrieve()
                        .bodyToMono(PaymentResult.class)
                        .timeout(Duration.ofSeconds(5))
                        .retryWhen(Retry.backoff(1, Duration.ofMillis(200))),
                    e -> Mono.just(PaymentResult.pending(order.getId()))
                ).map(payment -> OrderResult.of(order, payment));
            })
            .timeout(Duration.ofSeconds(10));  // 전체 타임아웃
    }
}
```

### 실험 2: 분산 추적 확인

```java
// 로그 패턴 예시
@GetMapping("/orders/{id}")
public Mono<OrderDetail> getOrder(@PathVariable Long id) {
    return ReactiveSecurityContextHolder.getContext()
        .flatMap(ctx -> {
            String userId = ctx.getAuthentication().getName();
            return Mono.deferContextual(context -> {
                String traceId = context.getOrDefault("traceId", "no-trace");
                log.info("[traceId={}] 주문 조회: orderId={}, userId={}",
                    traceId, id, userId);
                return getOrderDetail(id);
            });
        });
}
// 로그: [traceId=abc123] 주문 조회: orderId=1, userId=42
// 동일 traceId가 payment-service, inventory-service 로그에도 존재
// → Zipkin에서 전체 호출 체인 시각화
```

---

## 📊 성능 비교

```
서비스 간 호출 패턴별 처리 시간:

순차 호출 (기존 MVC 패턴):
  order → payment → inventory → shipping
  시간: 100 + 100 + 100 + 100 = 400ms

WebFlux 병렬 호출:
  order → zip(payment, inventory, shipping)
  시간: 100 + max(100, 100, 100) = 200ms (50% 단축)

Circuit Breaker OPEN (폴백):
  order → cb.run(payment, fallback)
  OPEN 상태: fallback 즉시 실행 ~1ms
  → 전체 요청 지연 없이 처리

재시도 비용:
  1회 실패 + 300ms 대기 + 2회 성공:
  총 시간: 100ms + 300ms + 100ms = 500ms
  → 재시도 횟수 × 대기 시간이 전체 레이턴시 결정
  → 상위 타임아웃 < 재시도 총 시간 이면 재시도 의미 없음

TraceId 전파 오버헤드:
  헤더 추가: ~수십 ns
  Zipkin 전송 (비동기): ~수 ms (별도 스레드)
  → 요청 처리에 거의 영향 없음
```

---

## ⚖️ 트레이드오프

```
Circuit Breaker 설정 트레이드오프:
  너무 민감: 일시적 장애에도 OPEN → 정상 요청 차단
  너무 둔감: 장애 서비스에 오래 요청 → 연결 풀 고갈

  권장: 실측 기반 설정
    - 정상 실패율: 1~5%
    - threshold: 정상 실패율의 5~10배 (10~50%)

타임아웃 계층:
  너무 짧음: 정상 응답도 타임아웃
  너무 긺: 장애 서비스에 오래 대기
  권장: SLA × 0.7 (SLA 10초 → 타임아웃 7초)

폴백 데이터 품질:
  빈 응답: 안전하지만 기능 저하
  캐시된 이전 응답: 오래된 데이터 가능
  → 서비스 특성에 따라 선택

Event-Driven vs Sync:
  Sync 호출: 즉시 응답, 결합도 높음
  Async 이벤트: 결합도 낮음, Eventually Consistent
  → 강한 일관성 필요: Sync
  → 독립성 중요: Event-Driven
```

---

## 📌 핵심 정리

```
MSA WebFlux 핵심:

방어 레이어:
  Timeout: 최대 응답 대기 시간 (.timeout(Duration))
  Retry: 일시적 장애 자동 복구 (.retryWhen(Retry.backoff))
  Circuit Breaker: 반복 장애 차단 (Resilience4j)
  Fallback: 대안 응답 (onErrorReturn, cb 폴백)

타임아웃 계층:
  하위 서비스 타임아웃 < 상위 서비스 타임아웃
  전체 처리 타임아웃은 각 단계 합보다 작게

분산 추적:
  Spring Boot 3 + Micrometer Tracing 자동 전파
  WebClient 요청에 TraceId 헤더 자동 첨부
  Zipkin/Jaeger로 호출 체인 시각화

이벤트 드리븐:
  Kafka + ReactiveKafkaProducerTemplate/ConsumerTemplate
  DisposableBean으로 구독 생명주기 관리
  flatMap(fn, 동시수)로 처리량 제어
```

---

## 🤔 생각해볼 문제

**Q1.** Circuit Breaker OPEN 상태에서 폴백이 항상 Mono.just()로 즉시 반환하지 않고, Redis에서 캐시된 데이터를 조회한다면 어떤 문제가 생길 수 있나요?

<details>
<summary>해설 보기</summary>

폴백 내에서 Redis 조회가 실패하면 폴백도 에러가 됩니다. 이중 장애(원래 서비스 + Redis)가 동시에 발생하면 요청이 에러로 반환됩니다.

또한 Circuit Breaker의 목적이 "빠른 실패"(fail fast)인데, 폴백에서 Redis를 조회하면 지연이 생겨 목적이 퇴색됩니다.

안전한 폴백 전략:
```java
// Redis 조회 폴백 (에러 처리 필수)
throwable -> {
    return redisTemplate.opsForValue()
        .get("cache:payment:" + orderId)
        .switchIfEmpty(Mono.just(PaymentResult.pending(orderId)))
        .onErrorReturn(PaymentResult.pending(orderId));  // Redis도 실패하면 기본값
}
```

폴백은 가능한 한 단순하게 (인메모리 기본값)로 유지하고, 복잡한 로직은 서킷이 닫혔을 때만 실행하는 것이 권장됩니다.

</details>

---

**Q2.** 서비스 A → 서비스 B → 서비스 C 호출 체인에서, B가 C에게 3초 타임아웃으로 호출하고 A는 B에게 2초 타임아웃으로 호출하면 어떤 일이 발생하나요?

<details>
<summary>해설 보기</summary>

A의 타임아웃(2초)이 B의 C 호출 타임아웃(3초)보다 짧습니다.

시나리오:
1. A → B 호출 (2초 타임아웃 시작)
2. B → C 호출 (3초 타임아웃 시작)
3. t=2초: A의 타임아웃 → A가 B와의 연결 종료
4. B는 여전히 C의 응답 대기 중 (3초까지)
5. t=3초: B의 C 호출 타임아웃 → B도 에러

문제:
- B는 A가 이미 끊었는데도 1초를 더 기다림 → B의 자원 낭비
- B가 C의 응답을 받아도 A에게 전달할 곳이 없음

해결 원칙: 하위 서비스 타임아웃 < 상위 서비스 타임아웃

```
올바른 설정:
  C: 1초 타임아웃
  B: C(1초) + 처리 시간 < 3초 타임아웃
  A: B(3초) + 처리 시간 < 10초 타임아웃
```

</details>

---

**Q3.** Reactive 서비스에서 요청 취소(cancel)가 서비스 간 호출 체인에 어떻게 전파되나요?

<details>
<summary>해설 보기</summary>

Reactive의 취소 신호는 파이프라인을 역방향으로 전파됩니다.

```
클라이언트 연결 종료
  → Netty 감지 → SSE/HTTP 스트림 cancel
  → WebFlux 파이프라인 cancel
  → 현재 진행 중인 WebClient 요청 cancel
  → Netty가 HTTP 요청 취소 (Connection 닫기)
  → 외부 서비스(B)가 연결 종료 감지
  → B도 진행 중인 C 호출 취소 (Reactor 파이프라인 cancel 전파)
```

이것이 Reactive의 큰 장점입니다. 클라이언트가 떠나면 불필요한 작업이 자동으로 취소됩니다. MVC에서는 클라이언트가 떠나도 서버에서 작업이 계속됩니다.

단, 취소 전파가 되려면 각 단계가 진정한 Reactive (Mono/Flux)여야 합니다. `block()`이나 블로킹 코드가 중간에 있으면 취소가 전파되지 않을 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Reactive Caching](./03-reactive-caching.md)** | **[홈으로 🏠](../README.md)** | **[다음: 언제 WebFlux를 쓰지 말아야 하는가 ➡️](./05-when-not-to-use-webflux.md)**

</div>
