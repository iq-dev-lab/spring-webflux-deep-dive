# Method Security — Reactive 메서드에서 @PreAuthorize

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@PreAuthorize`가 `Mono`/`Flux` 반환 메서드에서 어떻게 동작하는가?
- `ReactiveMethodSecurityConfiguration`은 AOP를 Reactive 파이프라인에 어떻게 끼워 넣는가?
- `@PreAuthorize`에서 `#id == authentication.name`처럼 SpEL로 파라미터를 검사하는 방법은?
- 메서드 보안과 URL 기반 보안을 어떻게 조합하는가?
- 커스텀 SpEL 보안 표현식을 만드는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

URL 기반 보안(`pathMatchers`)만으로는 세밀한 인가를 구현하기 어렵습니다. 예를 들어 "자신의 주문만 조회 가능" 같은 규칙은 URL로 표현할 수 없습니다. `@PreAuthorize`를 Reactive 메서드에 적용하면 비즈니스 로직과 인가 로직을 함께 선언할 수 있습니다. WebFlux에서 Method Security의 동작 원리를 알면 AOP 프록시가 Reactive 파이프라인을 어떻게 감싸는지 이해할 수 있습니다.

---

## 😱 흔한 실수 (Before — Method Security를 잘못 사용)

```
실수 1: @EnableReactiveMethodSecurity 없이 @PreAuthorize 사용

  @Service
  public class OrderService {
      @PreAuthorize("hasRole('ADMIN')")  // 설정 없이 사용
      public Mono<Order> getAdminOrder(Long id) { ... }
  }
  // → @PreAuthorize 무시됨! 아무나 접근 가능
  // → @EnableReactiveMethodSecurity 필수

실수 2: SpEL에서 파라미터 이름 접근 실패

  @PreAuthorize("#orderId == authentication.name")  // 잘못된 SpEL
  public Mono<Order> getOrder(Long orderId) { ... }
  // → UnsupportedOperationException 또는 항상 false
  // 이유: 컴파일 시 파라미터 이름이 제거될 수 있음
  // 해결: @Param 어노테이션 또는 -parameters 컴파일 옵션

실수 3: @PostAuthorize에서 Reactive 반환값 접근

  @PostAuthorize("returnObject.block().ownerId == authentication.name")
  public Mono<Order> getOrder(Long id) { ... }
  // block() → EventLoop 블로킹! 절대 금지
  // @PostAuthorize는 Mono/Flux에서 안전하게 동작하지 않음
```

---

## ✨ 올바른 접근 (After — Reactive Method Security 올바른 설정)

```
설정 활성화:

  @Configuration
  @EnableReactiveMethodSecurity  // 필수!
  public class MethodSecurityConfig {
      // 추가 설정 없으면 이것만으로 충분
  }

올바른 @PreAuthorize 사용:
  // 역할 기반
  @PreAuthorize("hasRole('ADMIN')")
  Mono<List<User>> getAllUsers();

  // 파라미터 기반 (소유자 확인)
  @PreAuthorize("#id.toString() == authentication.name")
  Mono<Order> getMyOrder(Long id);

  // 복잡한 표현식
  @PreAuthorize("hasRole('ADMIN') or #userId.toString() == authentication.name")
  Mono<Order> getUserOrder(Long userId, Long orderId);

  // 커스텀 SpEL 빈
  @PreAuthorize("@orderSecurity.isOwner(#orderId, authentication)")
  Mono<Order> getOrder(Long orderId);
```

---

## 🔬 내부 동작 원리

### 1. ReactiveMethodSecurityConfiguration

```
@EnableReactiveMethodSecurity 활성화 효과:

1. ReactiveMethodSecurityConfiguration 임포트
2. ReactiveMethodSecurityInterceptor 빈 등록
   → AOP Advice로 @PreAuthorize, @PostAuthorize, @PreFilter, @PostFilter 처리

3. 메서드 호출 가로채기 (AOP 프록시):
   @PreAuthorize 메서드 호출 시:
     → AOP 프록시 진입
     → SpEL 표현식 평가 (SecurityContext 접근 포함)
     → 평가 결과가 Mono<Boolean>
     → true: 원본 메서드 실행
     → false: Mono.error(AccessDeniedException) 방출

4. Reactive 파이프라인 통합:
   기존 Sync AOP: 메서드 실행 전에 동기 체크
   Reactive AOP: Mono/Flux 파이프라인으로 감쌈
   
   // 내부적으로 이렇게 동작 (단순화):
   return authentication.flatMap(auth -> {
       boolean allowed = evaluateSpEL(expression, auth, args);
       if (!allowed) return Mono.error(new AccessDeniedException(...));
       return originalMethod.invoke(args);  // Mono<Order>
   });
```

### 2. @PreAuthorize SpEL 표현식

```
SpEL에서 사용 가능한 객체:

authentication:      현재 Authentication 객체
  authentication.name         # principal의 이름
  authentication.principal    # UserDetails 객체
  authentication.authorities  # 권한 목록

hasRole('ADMIN'):   ROLE_ADMIN 권한 확인 (ROLE_ 접두사 자동 추가)
hasAuthority('x'):  정확한 권한명으로 확인

파라미터 접근:
  #paramName: 메서드 파라미터 (컴파일 옵션 또는 @Param 필요)

  // build.gradle에 컴파일 옵션 추가
  tasks.withType(JavaCompile).configureEach {
      options.compilerArgs << '-parameters'
  }

  @PreAuthorize("#userId.toString() == authentication.name")
  Mono<User> getUser(Long userId);
  // authentication.name = "42" (Long을 String으로 비교)

자주 사용하는 표현식:
  hasRole('ADMIN')
  hasAnyRole('USER', 'ADMIN')
  hasAuthority('READ_ORDERS')
  isAuthenticated()
  isAnonymous()
  #id.toString() == authentication.name
  hasRole('ADMIN') or #id.toString() == authentication.name
```

### 3. 커스텀 SpEL 빈 보안 표현식

```java
// 커스텀 보안 빈 (복잡한 인가 로직)
@Component("orderSecurity")
@RequiredArgsConstructor
public class OrderSecurityBean {

    private final OrderRepository orderRepository;

    // 주문 소유자 확인
    public Mono<Boolean> isOwner(Long orderId, Authentication auth) {
        return orderRepository.findById(orderId)
            .map(order -> order.getUserId().toString().equals(auth.getName()))
            .defaultIfEmpty(false);
    }

    // 관리자이거나 소유자인지 확인
    public Mono<Boolean> canAccess(Long orderId, Authentication auth) {
        boolean isAdmin = auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
        if (isAdmin) return Mono.just(true);
        return isOwner(orderId, auth);
    }
}

// 사용
@Service
public class OrderService {

    @PreAuthorize("@orderSecurity.isOwner(#orderId, authentication)")
    public Mono<Order> getOrder(Long orderId) {
        return orderRepository.findById(orderId);
    }

    @PreAuthorize("@orderSecurity.canAccess(#orderId, authentication)")
    public Mono<Void> deleteOrder(Long orderId) {
        return orderRepository.deleteById(orderId);
    }
}
```

### 4. @PreFilter와 @PostFilter (Flux)

```java
// @PreFilter: 입력 컬렉션 필터링
// @PostFilter: 출력 컬렉션 필터링
// 주의: Flux에서 완전히 지원되지 않을 수 있음 → 명시적 filter()로 대체 권장

// @PostFilter 대신 명시적 filter
@Service
public class DocumentService {

    // 내 문서만 반환 (@PostFilter 대신 명시적)
    public Flux<Document> getMyDocuments() {
        return ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication().getName())
            .flatMapMany(userId ->
                documentRepository.findAll()
                    .filter(doc -> doc.getOwnerId().toString().equals(userId))
            );
    }

    // 또는 DB 쿼리 수준에서 필터링 (권장)
    public Flux<Document> getMyDocumentsOptimized() {
        return ReactiveSecurityContextHolder.getContext()
            .map(ctx -> Long.parseLong(ctx.getAuthentication().getName()))
            .flatMapMany(userId ->
                documentRepository.findByOwnerId(userId)  // DB 수준 필터
            );
    }
}
```

### 5. URL 기반 보안과 Method Security 조합

```
계층적 보안 적용:

Layer 1: URL 기반 (SecurityWebFilterChain)
  - 조잡한 인가 (로그인 여부, 역할)
  - 빠름, 비즈니스 로직 없음
  - 예: /api/admin/** → ADMIN만

Layer 2: Method 기반 (@PreAuthorize)
  - 세밀한 인가 (소유자 확인 등)
  - 비즈니스 로직 포함 가능
  - 예: 자신의 주문만 조회

Layer 3: 도메인 기반 (코드 내 검증)
  - 비즈니스 규칙에 따른 검증
  - 예: 취소 가능한 상태인지 확인

권장 패턴:
  // URL: 인증 여부 + 조잡한 역할
  .authorizeExchange(auth -> auth
      .pathMatchers("/api/**").authenticated()
      .pathMatchers("/api/admin/**").hasRole("ADMIN")
  )

  // Method: 세밀한 소유자/권한 확인
  @PreAuthorize("@orderSecurity.canAccess(#orderId, authentication)")
  Mono<Order> getOrder(Long orderId);
  
  // 코드 내: 비즈니스 규칙
  return order.isCancellable()
      ? orderRepository.cancel(order)
      : Mono.error(new BusinessException("취소 불가 상태"));
```

---

## 💻 실전 코드

### 실험 1: Method Security 통합 테스트

```java
@SpringBootTest
@Import(TestSecurityConfig.class)
class OrderServiceSecurityTest {

    @Autowired OrderService orderService;

    @Test
    @WithMockUser(username = "1", roles = "USER")  // userId=1인 사용자
    void getOrder_ownOrder_shouldSucceed() {
        StepVerifier.create(orderService.getOrder(1L))  // userId=1의 주문
            .expectNextCount(1)
            .verifyComplete();
    }

    @Test
    @WithMockUser(username = "2", roles = "USER")  // userId=2인 사용자
    void getOrder_otherOrder_shouldBeDenied() {
        StepVerifier.create(orderService.getOrder(1L))  // userId=1의 주문
            .expectError(AccessDeniedException.class)
            .verify();
    }

    @Test
    @WithMockUser(username = "admin", roles = "ADMIN")
    void getOrder_admin_shouldSucceed() {
        StepVerifier.create(orderService.getOrder(1L))  // 누구의 주문이든
            .expectNextCount(1)
            .verifyComplete();
    }
}
```

### 실험 2: Reactive 환경에서 @WithMockUser

```java
// WebTestClient와 @WithMockUser 통합
@SpringBootTest(webEnvironment = RANDOM_PORT)
@AutoConfigureWebTestClient
class OrderControllerSecurityTest {

    @Autowired WebTestClient webTestClient;

    // SecurityMockServerConfigurers를 사용
    @Test
    void asAdmin_canAccessAdminEndpoint() {
        webTestClient
            .mutateWith(mockUser("admin").roles("ADMIN"))
            .get().uri("/api/admin/orders")
            .exchange()
            .expectStatus().isOk();
    }

    @Test
    void asUser_cannotAccessAdminEndpoint() {
        webTestClient
            .mutateWith(mockUser("user1").roles("USER"))
            .get().uri("/api/admin/orders")
            .exchange()
            .expectStatus().isForbidden();
    }

    @Test
    void unauthenticated_cannotAccessProtected() {
        webTestClient.get().uri("/api/orders")
            .exchange()
            .expectStatus().isUnauthorized();
    }
}
```

### 실험 3: 동적 권한 확인

```java
@Service
@RequiredArgsConstructor
public class SecureOrderService {

    private final OrderRepository orderRepository;

    // 동적 권한 확인 (SpEL 빈 없이 코드로)
    public Mono<Order> getOrderSecure(Long orderId) {
        return ReactiveSecurityContextHolder.getContext()
            .map(SecurityContext::getAuthentication)
            .flatMap(auth -> {
                boolean isAdmin = auth.getAuthorities().stream()
                    .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));

                if (isAdmin) {
                    return orderRepository.findById(orderId);
                }

                return orderRepository.findById(orderId)
                    .filter(order ->
                        order.getUserId().toString().equals(auth.getName()))
                    .switchIfEmpty(Mono.error(
                        new AccessDeniedException("접근 권한 없음")));
            });
    }
}
```

---

## 📊 성능 비교

```
URL 보안 vs Method 보안 오버헤드:

URL 기반 (pathMatchers):
  SecurityWebFilterChain의 AUTHORIZATION 필터
  Map 조회로 패턴 매칭: ~수μs
  AOP 없음 → 경량

Method 기반 (@PreAuthorize):
  AOP 프록시 호출: ~수십μs
  SpEL 평가: ~수백μs
  커스텀 빈 (@orderSecurity.isOwner): DB 쿼리 포함 시 ~수ms

@PreAuthorize + DB 조회:
  요청마다 DB 접근 → 레이턴시 증가
  해결: 권한 정보를 JWT에 포함하거나 캐싱

권장 패턴:
  간단한 역할 확인: URL 기반 (빠름)
  세밀한 소유자 확인: Method 기반 (정확함)
  소유자 정보를 JWT에 포함: DB 조회 없이 Method 보안 가능
    → authentication.name == #userId.toString()
```

---

## ⚖️ 트레이드오프

```
@PreAuthorize vs 코드 내 명시적 검증:
  @PreAuthorize:
    장점: 선언적, 인가 로직 분리, 테스트 용이(@WithMockUser)
    단점: SpEL 복잡도, 반응형 커스텀 빈 필요
  
  코드 내 검증:
    장점: 명시적, 디버깅 쉬움
    단점: 비즈니스 로직에 인가 로직 혼재

@PostFilter vs 명시적 filter():
  @PostFilter: 선언적, 하지만 Flux와 완전 호환 안 됨
  명시적 .filter(): 안전, 예측 가능
  → Flux에서는 명시적 filter() 권장

커스텀 SpEL 빈의 Mono<Boolean> 반환:
  장점: 비동기 DB 조회 가능
  단점: SpEL 평가가 비동기 → 파이프라인 복잡성
  → 단순 소유자 확인 선호, 복잡하면 코드 내 검증
```

---

## 📌 핵심 정리

```
Reactive Method Security 핵심:

활성화 필수:
  @EnableReactiveMethodSecurity
  (없으면 @PreAuthorize 무시됨)

SpEL 변수:
  authentication.name: 사용자 ID
  authentication.authorities: 권한 목록
  hasRole('ROLE'): 역할 확인
  #paramName: 메서드 파라미터 (-parameters 컴파일 옵션 필요)

커스텀 SpEL 빈:
  @Component("beanName")
  → @PreAuthorize("@beanName.method(#param, authentication)")
  → Mono<Boolean> 반환 가능 (비동기 검증)

계층 보안:
  URL 기반: 조잡한 인가 (빠름)
  Method 기반: 세밀한 인가 (유연)
  두 가지 조합이 권장 패턴

테스트:
  @WithMockUser(username="1", roles="USER")
  webTestClient.mutateWith(mockUser("admin").roles("ADMIN"))
```

---

## 🤔 생각해볼 문제

**Q1.** `@PreAuthorize`가 `Mono.empty()`를 반환하는 메서드에 적용되면 어떻게 동작하나요?

<details>
<summary>해설 보기</summary>

`@PreAuthorize` 검증은 메서드 실행 전에 이루어집니다. 권한이 없으면 `AccessDeniedException`이 `onError`로 방출됩니다. 권한이 있으면 메서드가 실행되고 `Mono.empty()`가 반환됩니다.

```java
@PreAuthorize("hasRole('ADMIN')")
public Mono<Void> deleteAll() {
    return orderRepository.deleteAll();
    // deleteAll()이 Mono<Void> = Mono.empty()를 반환
}

// ADMIN: @PreAuthorize 통과 → deleteAll() 실행 → onComplete
// USER:  @PreAuthorize 실패 → AccessDeniedException (onError)
```

`@PostAuthorize`는 `Mono.empty()`에서 `returnObject`가 null이므로 주의가 필요합니다. `@PostAuthorize`는 Flux에서는 일반적으로 권장되지 않습니다.

</details>

---

**Q2.** `@PreAuthorize`와 `@Transactional`을 같은 메서드에 함께 사용하면 어떤 순서로 AOP가 적용되나요?

<details>
<summary>해설 보기</summary>

AOP의 `@Order`로 적용 순서가 결정됩니다.

기본 순서:
- `ReactiveMethodSecurityInterceptor`의 Order: 낮음 (나중 적용 = 가장 바깥쪽 프록시)
- `@Transactional`의 Order: 높음 (먼저 적용 = 안쪽 프록시)

실제 실행 순서:
```
클라이언트 호출
  → Security 프록시(@PreAuthorize 체크)
    → Transaction 프록시(트랜잭션 시작)
      → 실제 메서드 실행
    ← Transaction 프록시(커밋/롤백)
  ← Security 프록시(결과 반환)
```

권한 없으면 트랜잭션이 시작되지 않습니다(Security가 바깥쪽). 권한 있고 비즈니스 로직 실패 시 트랜잭션이 롤백됩니다.

`@Order`로 순서를 변경할 수 있지만, 일반적으로 기본 순서가 적합합니다.

</details>

---

**Q3.** 서비스 계층이 아닌 Repository 계층에 `@PreAuthorize`를 적용하는 것이 적절한가요?

<details>
<summary>해설 보기</summary>

기술적으로 가능하지만 권장하지 않습니다.

**문제점:**
- Repository는 데이터 액세스 레이어로, 인가 로직은 비즈니스 레이어(Service)의 책임입니다.
- Repository에 `@PreAuthorize`를 붙이면 테스트 시 항상 보안 컨텍스트가 필요합니다.
- 관심사 분리가 깨집니다.

```java
// 나쁜 예: Repository에 @PreAuthorize
@Repository
public interface OrderRepository extends R2dbcRepository<Order, Long> {
    @PreAuthorize("hasRole('ADMIN')")
    Flux<Order> findAll();  // 모든 조회에 ADMIN 필요 → 서비스에서 쓸 수 없음
}

// 좋은 예: Service에 @PreAuthorize
@Service
public class OrderService {
    @PreAuthorize("hasRole('ADMIN')")
    public Flux<Order> getAllOrders() {
        return orderRepository.findAll();
    }

    // 일반 서비스 메서드는 보안 없이 Repository 직접 호출
    public Mono<Order> findById(Long id) {
        return orderRepository.findById(id);
    }
}
```

인가 로직은 Service 계층에 두어 Repository의 재사용성과 테스트 용이성을 유지합니다.

</details>

---

<div align="center">

**[⬅️ 이전: JWT 인증 Reactive 구현](./02-jwt-reactive-auth.md)** | **[홈으로 🏠](../README.md)** | **[다음: OAuth2 Reactive ➡️](./04-oauth2-reactive.md)**

</div>
