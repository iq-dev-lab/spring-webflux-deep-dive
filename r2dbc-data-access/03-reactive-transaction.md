# Reactive Transaction — @Transactional이 Reactive 컨텍스트에서 동작하는 방식

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Transactional`이 Reactive 환경에서 `ThreadLocal` 대신 Reactor `Context`를 사용하는 이유는?
- `TransactionalOperator`와 `@Transactional`은 Reactive 코드에서 어떻게 다르게 동작하는가?
- Reactive 파이프라인에서 롤백이 어떻게 트리거되는가?
- `flatMap` 내부의 에러는 외부 트랜잭션에 어떻게 전파되는가?
- `Propagation.REQUIRES_NEW`는 Reactive 트랜잭션에서 어떻게 구현되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"R2DBC에서 `@Transactional`을 붙였는데 롤백이 안 된다"는 버그는 Reactive 트랜잭션의 전파 방식을 모를 때 발생합니다. JPA에서 ThreadLocal에 연결을 바인딩하는 방식을 그대로 기대하면 Reactor Context 기반의 Reactive 트랜잭션을 이해하기 어렵습니다. 이 문서를 읽고 나면 `@Transactional`을 WebFlux/R2DBC에서 안전하게 사용하고, 필요 시 `TransactionalOperator`로 프로그래밍 방식 트랜잭션을 구현할 수 있습니다.

---

## 😱 흔한 실수 (Before — Reactive 트랜잭션을 잘못 이해할 때)

```
실수 1: subscribe() 후 트랜잭션 경계 이탈

  @Transactional
  public Mono<Void> process() {
      orderRepo.save(order).subscribe();    // 내부 subscribe!
      paymentRepo.save(payment).subscribe(); // 다른 subscribe!
      return Mono.empty();
      // 각 subscribe()가 독립적 트랜잭션으로 실행
      // → 하나 실패해도 다른 것 롤백 안 됨
  }

실수 2: @Transactional 메서드에서 Mono 반환 없이 블로킹

  @Transactional
  public Mono<Order> createOrder(OrderRequest req) {
      Order order = orderRepo.findById(1L).block();  // 블로킹!
      // block() → 새 스레드 → ThreadLocal 트랜잭션 유실 가능
      // Reactive 트랜잭션과 block() 혼용 = 위험
      return Mono.just(processedOrder);
  }

실수 3: 트랜잭션 내에서 예외를 onErrorResume으로 무시

  @Transactional
  public Mono<Void> process() {
      return orderRepo.save(order)
          .then(paymentRepo.save(payment))
          .onErrorResume(e -> {
              log.error("에러 무시", e);
              return Mono.empty();  // 에러를 정상으로 처리
          });
      // onErrorResume으로 에러를 삼키면 트랜잭션은 롤백 안 됨
      // → 부분 커밋 가능
  }
```

---

## ✨ 올바른 접근 (After — Reactive 트랜잭션 원칙)

```
핵심 원칙:

1. 트랜잭션 경계 = 하나의 Reactive 파이프라인
   @Transactional 메서드 → Mono/Flux 반환 필수
   파이프라인 내 모든 R2DBC 작업이 동일 트랜잭션 참여

2. 롤백 = onError 신호
   파이프라인에서 에러 발생 → 트랜잭션 롤백
   onErrorResume으로 에러를 삼키면 롤백 안 됨

3. 복잡한 트랜잭션은 TransactionalOperator
   프로그래밍 방식으로 트랜잭션 범위 명시적 제어

@Transactional 올바른 사용:
  @Transactional
  public Mono<Order> createOrder(OrderRequest req) {
      return orderRepo.save(Order.from(req))   // 트랜잭션 내
          .flatMap(order ->
              paymentRepo.save(Payment.from(order))  // 동일 트랜잭션
              .thenReturn(order)
          );
      // 에러 발생 → onError → 트랜잭션 롤백
  }
```

---

## 🔬 내부 동작 원리

### 1. Reactor Context 기반 트랜잭션 바인딩

```
MVC 트랜잭션 (ThreadLocal 방식):
  Thread-1: @Transactional AOP 진입
  → Connection 획득 → ThreadLocal["connection"] = conn
  Thread-1: orderRepo.save() → ThreadLocal에서 conn 꺼냄 → 동일 트랜잭션
  Thread-1: @Transactional AOP 종료 → 커밋/롤백

Reactive 트랜잭션 (Reactor Context 방식):
  subscribe() 시 ReactiveTransactionManager가 개입
  → Connection 획득
  → contextWrite(Context.of("transaction-sync", transactionSync))
  → 파이프라인 전체에 Context 전파 (스레드 전환과 무관)

  orderRepo.save() 내부:
    → Mono.deferContextual(ctx -> {
          TransactionSynchronization sync = ctx.get("transaction-sync");
          // sync에서 Connection 꺼냄 → 동일 트랜잭션
      })

  핵심: Context가 파이프라인을 따라 자동 전파
  publishOn, subscribeOn으로 스레드 전환해도 Context 유지
  → MVC와 달리 스레드 독립적 트랜잭션 바인딩
```

### 2. @Transactional의 Reactive 처리 흐름

```
@Transactional 메서드 처리 (Spring AOP 프록시):

1. 프록시 메서드 진입
2. ReactiveTransactionManager.getReactiveTransaction(txDef) 호출
   → Mono<ReactiveTransaction> 반환
3. 원본 메서드 실행 → Mono<Order> 파이프라인 구성
4. 트랜잭션 컨텍스트 주입:
   원본Mono.contextWrite(txContext)
   → 파이프라인 전체에 트랜잭션 Context 전파
5. 파이프라인 결과에 따라:
   onComplete: 커밋
   onError: 롤백

구체적인 내부 코드 (단순화):

  // AOP 프록시가 하는 일 (의사 코드)
  public Mono<Order> createOrderProxy(OrderRequest req) {
      return transactionManager.getReactiveTransaction(txDef)
          .flatMap(status -> {
              return createOrderOriginal(req)
                  .flatMap(result ->
                      transactionManager.commit(status)
                          .thenReturn(result)
                  )
                  .onErrorResume(ex ->
                      transactionManager.rollback(status)
                          .then(Mono.error(ex))  // 에러 재방출
                  );
          });
  }
```

### 3. TransactionalOperator — 프로그래밍 방식 트랜잭션

```
TransactionalOperator:
  @Transactional 없이 코드로 트랜잭션 범위 명시
  복잡한 조건부 트랜잭션에 유용

  @Autowired TransactionalOperator transactionalOperator;

  public Mono<Void> complexOperation() {
      Mono<Void> operation = orderRepo.save(order)
          .then(paymentRepo.save(payment))
          .then(inventoryRepo.decrease(itemId, qty));

      return transactionalOperator.transactional(operation);
      // operation 전체를 트랜잭션으로 감쌈
  }

  // 또는 as() 사용:
  return orderRepo.save(order)
      .then(paymentRepo.save(payment))
      .as(transactionalOperator::transactional);

@Transactional vs TransactionalOperator:
  @Transactional: 선언적, AOP 기반, 간결
    → 메서드 전체가 트랜잭션 범위
    → 메서드가 Mono/Flux 반환 필수
  TransactionalOperator: 프로그래밍 방식, 유연
    → 원하는 Mono/Flux 부분만 트랜잭션 적용 가능
    → 조건부 트랜잭션, 중첩 트랜잭션에 유용
```

### 4. 롤백 — onError 신호의 역할

```
롤백 발생 조건:
  파이프라인에서 onError 신호 발생 → 트랜잭션 롤백

롤백 시나리오:

  @Transactional
  public Mono<Order> createOrder(OrderRequest req) {
      return orderRepo.save(order)      // 성공: onNext(order)
          .flatMap(savedOrder ->
              paymentRepo.save(payment)  // 실패: onError(PaymentException)
          );
      // → orderRepo.save는 커밋 안 됨 (트랜잭션 롤백)
  }

롤백이 일어나지 않는 함정:
  // 에러를 삼키면 롤백 안 됨
  .onErrorResume(e -> {
      log.error("무시", e);
      return Mono.empty();  // 에러 → 정상으로 처리
  })
  // → onComplete → 커밋 → order는 저장, payment는 없음

올바른 에러 처리 (롤백 보장):
  .onErrorResume(PaymentException.class, e -> {
      // 에러 변환 (여전히 에러 상태)
      return Mono.error(new OrderCreationException("결제 실패", e));
  })
  // → 여전히 onError → 롤백 발생

@Transactional(rollbackFor) 지정:
  @Transactional(rollbackFor = BusinessException.class)
  // 기본: RuntimeException, Error만 롤백
  // Checked Exception은 @rollbackFor 명시 필요
```

### 5. 트랜잭션 전파 (Propagation)

```
REQUIRED (기본):
  외부 트랜잭션 있으면 참여, 없으면 새로 시작
  Reactive에서 Reactor Context의 트랜잭션 정보 확인

  @Transactional  // REQUIRED
  public Mono<Void> outerMethod() {
      return orderService.createOrder()  // @Transactional(REQUIRED)
          .then(paymentService.charge()); // 동일 트랜잭션 참여
  }

REQUIRES_NEW:
  항상 새 트랜잭션 시작 (외부 트랜잭션 일시 중단)
  Reactive에서: 새 Connection 획득, 별도 Context로 격리

  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public Mono<Void> auditLog(String action) {
      // 외부 트랜잭션과 무관하게 독립 커밋
      return auditRepo.save(new AuditLog(action));
  }

  실제 사용:
    @Transactional
    public Mono<Order> createOrder(OrderRequest req) {
        return orderRepo.save(order)
            .flatMap(saved ->
                auditService.log("ORDER_CREATED")  // REQUIRES_NEW
                    .thenReturn(saved)
            );
        // 주문 저장 롤백 시 감사 로그는 유지 (별도 트랜잭션)
    }

NESTED:
  R2DBC에서 완전 지원하지 않음 → REQUIRES_NEW로 대안
  Savepoint 기반이라 드라이버 지원 필요

Reactive 전파 주의사항:
  flatMap 내부에서 @Transactional 서비스 호출 시
  → Context 전파로 외부 트랜잭션에 자동 참여 (REQUIRED 기본)
  → subscribe()를 내부에서 호출하면 Context 분리 → 참여 실패
```

---

## 💻 실전 코드

### 실험 1: @Transactional 기본 사용

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepo;
    private final PaymentRepository paymentRepo;
    private final InventoryRepository inventoryRepo;

    @Transactional
    public Mono<OrderResult> createOrder(CreateOrderRequest req) {
        return orderRepo.save(Order.from(req))
            .flatMap(order ->
                inventoryRepo.decrease(req.itemId(), req.quantity())
                    .filter(affected -> affected > 0)
                    .switchIfEmpty(Mono.error(new InsufficientStockException()))
                    .thenReturn(order)
            )
            .flatMap(order ->
                paymentRepo.save(Payment.from(order))
                    .map(payment -> OrderResult.of(order, payment))
            );
        // 어느 단계에서든 에러 → 전체 롤백
        // inventoryRepo, orderRepo 모두 롤백
    }
}
```

### 실험 2: TransactionalOperator로 복잡한 트랜잭션

```java
@Service
@RequiredArgsConstructor
public class BatchOrderService {
    private final OrderRepository orderRepo;
    private final TransactionalOperator txOp;

    // 조건부 트랜잭션: 100개 이상이면 트랜잭션, 아니면 그냥 저장
    public Flux<Order> saveBatch(List<CreateOrderRequest> requests) {
        Flux<Order> saveFlux = Flux.fromIterable(requests)
            .flatMap(req -> orderRepo.save(Order.from(req)));

        if (requests.size() >= 100) {
            // 대량 배치: 트랜잭션으로 원자성 보장
            return saveFlux.as(txOp::transactional);
        } else {
            // 소량: 각각 독립 저장 (성능 우선)
            return saveFlux;
        }
    }

    // REQUIRES_NEW로 감사 로그 독립 커밋
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public Mono<Void> saveAuditLog(String event, Long orderId) {
        return auditRepo.save(new AuditLog(event, orderId)).then();
    }
}
```

### 실험 3: 트랜잭션 테스트

```java
@DataR2dbcTest
class OrderServiceTest {

    @Autowired OrderService orderService;
    @Autowired OrderRepository orderRepo;
    @Autowired InventoryRepository inventoryRepo;

    @Test
    void createOrder_paymentFail_shouldRollback() {
        // 재고 1개만 있는 상태
        inventoryRepo.save(new Inventory(itemId, 1)).block();

        // 2개 주문 → 재고 부족 → 롤백
        StepVerifier.create(
            orderService.createOrder(new CreateOrderRequest(itemId, 2))
        )
        .expectError(InsufficientStockException.class)
        .verify();

        // 롤백 검증: 주문이 저장되지 않아야 함
        StepVerifier.create(orderRepo.count())
            .expectNext(0L)  // 롤백으로 0개 확인
            .verifyComplete();
    }

    @Test
    @Transactional  // 테스트 후 롤백
    void createOrder_success_shouldCommit() {
        StepVerifier.create(
            orderService.createOrder(validRequest)
        )
        .expectNextMatches(result -> result.order().id() != null)
        .verifyComplete();
    }
}
```

---

## 📊 성능 비교

```
트랜잭션 방식별 오버헤드:

@Transactional (R2DBC):
  AOP 프록시: ~수μs
  Context 설정 및 전파: ~수십 ns
  총 오버헤드: < 1ms (DB 쿼리 시간 대비 무시 가능)

TransactionalOperator:
  직접 호출로 AOP 없음 → 약간 더 빠름
  실질적 차이: 무시 가능

REQUIRES_NEW (새 연결 획득):
  새 DB 연결 획득 시간: ~수 ms (풀에서 즉시) or ~수십 ms (연결 수립)
  → 남용 시 성능 저하
  → 꼭 필요한 경우만 사용 (감사 로그, 독립 커밋)

트랜잭션 격리 수준별 성능:
  READ_UNCOMMITTED: 가장 빠름 (잠금 없음)
  READ_COMMITTED:   권장 (대부분 DB 기본값)
  REPEATABLE_READ:  슬픔 느림 (행 잠금)
  SERIALIZABLE:     가장 느림 (범위 잠금, 교착 가능)
  → 대부분 READ_COMMITTED 유지
```

---

## ⚖️ 트레이드오프

```
@Transactional vs TransactionalOperator:
  @Transactional: 선언적, 간결 → 일반적인 CRUD에 권장
  TransactionalOperator: 유연, 명시적 → 복잡한 조건부 트랜잭션

onErrorResume vs 에러 전파:
  onErrorResume으로 에러 삼키기: 트랜잭션 커밋 위험
  에러 전파 (Mono.error 재방출): 트랜잭션 롤백 보장
  → 트랜잭션 내부 에러 처리는 에러를 살려두거나 변환만

REQUIRES_NEW 트레이드오프:
  장점: 독립 커밋 (외부 롤백에 영향 안 받음)
  단점: 추가 DB 연결 소비, 연결 풀 압박
  → 감사 로그, 이메일 발송 기록 등 최소화해서 사용

테스트에서 @Transactional:
  @DataR2dbcTest + @Transactional → 각 테스트 후 롤백
  → 테스트 간 데이터 오염 없음
  → 실제 커밋은 하지 않으므로 DB 상태 보장
```

---

## 📌 핵심 정리

```
Reactive 트랜잭션 핵심:

왜 Context 기반인가:
  ThreadLocal: 스레드 전환 시 유실
  Reactor Context: 파이프라인 전체 전파, 스레드 독립

@Transactional 작동 조건:
  메서드가 Mono/Flux 반환 필수
  subscribe() 내부 호출 금지 (Context 분리)
  에러를 삼키지 않음 (onError → 롤백)

TransactionalOperator:
  프로그래밍 방식, 조건부 트랜잭션에 유용
  .as(txOp::transactional) 또는 txOp.transactional(mono)

롤백 발생:
  파이프라인에서 onError 신호
  onErrorResume으로 에러 삼키면 롤백 안 됨
  에러 타입 변환은 가능 (Mono.error로 재방출)

전파:
  REQUIRED: Context의 기존 트랜잭션에 참여 (기본)
  REQUIRES_NEW: 독립 트랜잭션 (새 연결, 별도 커밋)
```

---

## 🤔 생각해볼 문제

**Q1.** `@Transactional` 메서드에서 `Mono.just(value)`를 반환하면 트랜잭션이 정상 동작하나요?

<details>
<summary>해설 보기</summary>

`Mono.just(value)`는 이미 값을 가진 Cold Publisher입니다. 트랜잭션은 동작하지만, R2DBC 작업이 없으면 의미가 없습니다.

주의할 점은 `Mono.just()` 안에 블로킹 코드가 있을 때입니다:

```java
@Transactional
public Mono<String> problematic() {
    // Mono.just()의 인자는 즉시 평가됨 (선언 시점에 실행)
    String result = jpaRepo.findAll().toString();  // 블로킹! (트랜잭션 시작 전)
    return Mono.just(result);
    // 트랜잭션은 적용되지만 JPA 작업은 트랜잭션 밖에서 실행됨
}

// 올바른 방식: fromCallable()로 지연 평가
@Transactional
public Mono<String> correct() {
    return Mono.fromCallable(() -> jpaRepo.findAll().toString())
        .subscribeOn(Schedulers.boundedElastic());
    // subscribe() 시점에 트랜잭션 Context가 있음 → 트랜잭션 내 실행
}
```

R2DBC에서는 이 문제가 없습니다. `r2dbcRepo.findAll()`은 Flux를 반환하므로 subscribe 시점, 즉 트랜잭션 Context가 있는 시점에 실행됩니다.

</details>

---

**Q2.** `@Transactional` 메서드에서 `Flux`를 반환할 때 트랜잭션은 언제 커밋되나요?

<details>
<summary>해설 보기</summary>

`Flux`의 `onComplete` 신호가 발생할 때 커밋됩니다. 즉, 모든 항목이 방출된 후입니다.

```java
@Transactional
public Flux<Order> processAllOrders() {
    return orderRepo.findByStatus("PENDING")
        .flatMap(order ->
            processOrder(order)
                .doOnNext(processed ->
                    log.info("처리: {}", processed.id())
                )
        );
    // 모든 주문 처리 완료(onComplete) → 커밋
    // 중간에 에러 → 롤백
}
```

주의: Flux가 무한 스트림이면 커밋이 영원히 발생하지 않습니다. `take(n)`, `timeout()` 등으로 종료 조건을 만들어야 합니다. 또한 Flux가 `100만 건`이면 모두 처리될 때까지 트랜잭션이 열린 상태로 유지됩니다. 대용량 처리에는 배치로 나누어 트랜잭션을 분할하는 것이 좋습니다.

</details>

---

**Q3.** Reactive 트랜잭션에서 외부 API 호출(WebClient)이 실패하면 DB 작업도 롤백되나요?

<details>
<summary>해설 보기</summary>

네, `onError` 신호가 발생하면 트랜잭션이 롤백됩니다. WebClient 실패도 에러 신호를 방출하므로 DB 롤백이 발생합니다.

```java
@Transactional
public Mono<Order> createOrder(OrderRequest req) {
    return orderRepo.save(Order.from(req))  // DB에 저장
        .flatMap(order ->
            paymentClient.charge(order)      // 외부 API 호출
                .map(payment -> order)
        );
    // paymentClient 실패 → onError → 트랜잭션 롤백
    // orderRepo.save()도 롤백됨
}
```

단, 외부 API 호출이 성공했어도 이후 다른 DB 작업이 실패하면 롤백됩니다. 외부 API는 트랜잭션 범위 밖이므로 실제 결제는 이미 처리됐는데 주문 저장이 롤백되는 문제가 생길 수 있습니다.

이런 경우 **Saga 패턴** 또는 **보상 트랜잭션(Compensating Transaction)**을 사용합니다:
```java
return orderRepo.save(order)
    .flatMap(savedOrder ->
        paymentClient.charge(savedOrder)
            .onErrorResume(e ->
                paymentClient.cancel(savedOrder)  // 보상 트랜잭션
                    .then(Mono.error(e))
            )
    );
```

</details>

---

<div align="center">

**[⬅️ 이전: R2DBC 완전 분해](./02-r2dbc-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: R2DBC 실전 패턴 ➡️](./04-r2dbc-practical-patterns.md)**

</div>
