# Reactive Security 아키텍처 — SecurityWebFilterChain

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `SecurityWebFilterChain`은 `FilterChainProxy`를 어떻게 대체하는가?
- `ReactiveAuthenticationManager`는 인증을 왜 비동기로 처리하는가?
- `ServerSecurityContextRepository`는 `SecurityContext`를 어떻게 Reactor Context에 저장하는가?
- WebFlux에서 특정 경로만 보안 적용, 나머지 허용하는 설정은 어떻게 하는가?
- `ServerHttpSecurity`의 DSL로 설정하는 주요 보안 기능은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring Security를 WebFlux에 도입할 때 MVC의 `HttpSecurity` 설정을 그대로 옮기려 하면 동작하지 않습니다. `WebSecurityConfigurerAdapter`는 WebFlux에서 사용할 수 없고, `ServerHttpSecurity`로 대체됩니다. Reactive Security의 전체 동작 원리를 이해하면 인증 흐름을 커스터마이징하고, 복잡한 인가 규칙을 Reactive하게 구현할 수 있습니다.

---

## 😱 흔한 실수 (Before — MVC Security 방식을 WebFlux에 적용)

```
실수 1: WebSecurityConfigurerAdapter를 WebFlux에 사용

  @Configuration
  public class SecurityConfig extends WebSecurityConfigurerAdapter {
      // WebFlux에서 동작하지 않음!
      // NoSuchBeanDefinitionException: WebSecurityConfigurerAdapter
  }

실수 2: SecurityContextHolder (ThreadLocal 기반)를 WebFlux에서 접근

  @GetMapping("/me")
  public Mono<User> getMe() {
      Authentication auth =
          SecurityContextHolder.getContext().getAuthentication();
      // → null! WebFlux에서 ThreadLocal 기반 접근 불가
      return Mono.just(getCurrentUser(auth));
  }

실수 3: 모든 요청에 @PreAuthorize를 달지 않고 SecurityConfig만 의존

  @Bean
  public SecurityWebFilterChain chain(ServerHttpSecurity http) {
      return http.authorizeExchange(auth ->
          auth.anyExchange().authenticated()  // 모든 요청 인증 필요
      ).build();
      // actuator 엔드포인트, static 리소스도 인증 필요
      // → actuator health 체크 실패
  }
```

---

## ✨ 올바른 접근 (After — Reactive Security 설계)

```
올바른 SecurityWebFilterChain 설정:

  @Bean
  public SecurityWebFilterChain securityChain(ServerHttpSecurity http) {
      return http
          .csrf(ServerHttpSecurity.CsrfSpec::disable)
          .authorizeExchange(auth -> auth
              .pathMatchers("/actuator/health").permitAll()
              .pathMatchers("/api/public/**").permitAll()
              .pathMatchers("/api/admin/**").hasRole("ADMIN")
              .anyExchange().authenticated()
          )
          .addFilterAt(jwtAuthFilter(), SecurityWebFiltersOrder.AUTHENTICATION)
          .build();
  }

현재 사용자 접근 (Reactive 방식):
  @GetMapping("/me")
  public Mono<User> getMe() {
      return ReactiveSecurityContextHolder.getContext()
          .map(ctx -> ctx.getAuthentication())
          .flatMap(auth -> userService.findByUsername(auth.getName()));
  }
  // 또는 파라미터 주입
  @GetMapping("/me")
  public Mono<User> getMe(@AuthenticationPrincipal Mono<UserDetails> user) {
      return user.flatMap(u -> userService.findByUsername(u.getUsername()));
  }
```

---

## 🔬 내부 동작 원리

### 1. FilterChainProxy vs SecurityWebFilterChain

```
MVC (Servlet 기반):
  FilterChainProxy (Servlet Filter)
    → 요청마다 SecurityFilterChain 결정
    → 체인 내 SecurityFilter들이 동기 실행
    → ThreadLocal에 SecurityContext 저장

WebFlux (Reactive 기반):
  WebFilterChainProxy (WebFilter)
    → SecurityWebFilterChain들을 순서대로 확인
    → 매칭되는 첫 번째 체인 사용
    → 체인 내 WebFilter들이 Mono<Void> 반환
    → Reactor Context에 SecurityContext 저장

SecurityWebFilterChain 구성:

  @Bean
  public SecurityWebFilterChain chain(ServerHttpSecurity http) {
      // ServerHttpSecurity = WebFlux용 HttpSecurity
      return http
          .securityMatcher(new PathPatternParserServerWebExchangeMatcher("/api/**"))
          // 이 체인이 적용될 경로 패턴
          .authorizeExchange(...)
          .build();
  }

  // 여러 체인 (순서 중요):
  @Bean @Order(1)
  SecurityWebFilterChain apiChain(ServerHttpSecurity http) { ... }

  @Bean @Order(2)
  SecurityWebFilterChain defaultChain(ServerHttpSecurity http) { ... }
```

### 2. Reactive Security Filter 체인

```
SecurityWebFilterChain 내부 필터 순서 (SecurityWebFiltersOrder):

  FIRST(-100)
  HTTP_HEADERS_WRITER(-200)      → 보안 헤더 (X-Frame-Options 등)
  CORS(-300)                     → CORS 처리
  CSRF(-400)                     → CSRF 토큰 검증
  REACTOR_CONTEXT_WEB_FILTER(-500)  → Reactor Context 초기화
  HTTP_BASIC_AUTHENTICATION(-600)   → HTTP Basic 인증
  FORM_LOGIN(-700)               → Form 로그인 처리
  OAUTH2_LOGIN_PAGE_WEB_FILTER(-800) → OAuth2 로그인
  AUTHENTICATION(-900)           → 인증 처리
  OAUTH2_AUTHORIZATION_CODE_WEB_FILTER → OAuth2 코드 교환
  LOGIN_PAGE_GENERATING_WEB_FILTER → 기본 로그인 페이지 생성
  LOGOUT(-1000)                  → 로그아웃 처리
  EXCEPTION_TRANSLATION(-1100)   → 인증/인가 예외 처리
  AUTHORIZATION(-1200)           → 인가 (pathMatchers 체크)
  LAST(Integer.MIN_VALUE)

각 필터가 Mono<Void>를 반환:
  WebFilter.filter(exchange, chain):
    → 처리 완료 시: chain.filter(exchange)
    → 차단 시: exchange.getResponse().setComplete() 등
```

### 3. ReactiveAuthenticationManager — 비동기 인증

```
MVC AuthenticationManager:
  Authentication authenticate(Authentication auth) throws AuthException;
  → 동기 반환, 블로킹 가능

ReactiveAuthenticationManager:
  Mono<Authentication> authenticate(Authentication auth);
  → Mono 반환, 완전 비동기

실제 사용:

  @Bean
  public ReactiveAuthenticationManager authenticationManager(
          ReactiveUserDetailsService userDetailsService,
          PasswordEncoder passwordEncoder) {
      UserDetailsRepositoryReactiveAuthenticationManager manager =
          new UserDetailsRepositoryReactiveAuthenticationManager(userDetailsService);
      manager.setPasswordEncoder(passwordEncoder);
      return manager;
  }

  // ReactiveUserDetailsService
  @Service
  public class UserDetailsServiceImpl
          implements ReactiveUserDetailsService {

      @Override
      public Mono<UserDetails> findByUsername(String username) {
          return userRepository.findByUsername(username)
              .map(user -> User.withUsername(user.getUsername())
                  .password(user.getPasswordHash())
                  .roles(user.getRoles().toArray(new String[0]))
                  .build()
              );
          // 논블로킹 DB 조회 (R2DBC)
      }
  }
```

### 4. ServerSecurityContextRepository — Context 저장소

```
SecurityContext 저장 전략:

NoOpServerSecurityContextRepository (상태 없음):
  요청마다 새로 인증 처리
  JWT, OAuth2 Bearer Token 방식에 적합
  
  .securityContextRepository(NoOpServerSecurityContextRepository.getInstance())

WebSessionServerSecurityContextRepository (세션 기반):
  SecurityContext를 WebSession에 저장
  Form 로그인 방식에 적합 (기본값)

Reactor Context 저장 과정:

  1. AuthenticationWebFilter가 인증 처리
  2. authentication → SecurityContext 생성
  3. ServerSecurityContextRepository.save(exchange, context) 호출
  4. 실제 저장:
     Reactor Context에 SecurityContext 키로 Mono<SecurityContext> 저장
     → contextWrite(ReactiveSecurityContextHolder.withSecurityContext(...))

  5. 이후 파이프라인에서 접근:
     ReactiveSecurityContextHolder.getContext()
     → Mono.deferContextual(ctx -> ctx.get(SECURITY_CONTEXT_KEY))
     → 스레드 전환 무관하게 Context에서 꺼냄
```

### 5. ServerHttpSecurity DSL — 주요 설정

```
@Bean
public SecurityWebFilterChain securityChain(ServerHttpSecurity http) {
    return http

        // CSRF (API 서버는 보통 비활성화)
        .csrf(ServerHttpSecurity.CsrfSpec::disable)

        // CORS 설정
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))

        // 인가 규칙
        .authorizeExchange(auth -> auth
            .pathMatchers(HttpMethod.GET, "/api/public/**").permitAll()
            .pathMatchers("/api/admin/**").hasAuthority("ROLE_ADMIN")
            .pathMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
            .pathMatchers("/actuator/**").hasIpAddress("127.0.0.1")
            .anyExchange().authenticated()
        )

        // HTTP Basic 인증 비활성화 (JWT 사용 시)
        .httpBasic(ServerHttpSecurity.HttpBasicSpec::disable)
        .formLogin(ServerHttpSecurity.FormLoginSpec::disable)

        // JWT 필터 추가
        .addFilterAt(jwtAuthFilter, SecurityWebFiltersOrder.AUTHENTICATION)

        // 예외 처리
        .exceptionHandling(ex -> ex
            .authenticationEntryPoint((exchange, e) -> {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            })
            .accessDeniedHandler((exchange, e) -> {
                exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
                return exchange.getResponse().setComplete();
            })
        )

        // SecurityContext 저장소 (JWT는 무상태)
        .securityContextRepository(NoOpServerSecurityContextRepository.getInstance())

        .build();
}
```

---

## 💻 실전 코드

### 실험 1: 다중 SecurityWebFilterChain

```java
@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    // API 체인 (JWT 인증)
    @Bean @Order(1)
    public SecurityWebFilterChain apiChain(
            ServerHttpSecurity http,
            JwtAuthenticationFilter jwtFilter) {
        return http
            .securityMatcher(new PathPatternParserServerWebExchangeMatcher("/api/**"))
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .httpBasic(ServerHttpSecurity.HttpBasicSpec::disable)
            .formLogin(ServerHttpSecurity.FormLoginSpec::disable)
            .addFilterAt(jwtFilter, SecurityWebFiltersOrder.AUTHENTICATION)
            .securityContextRepository(NoOpServerSecurityContextRepository.getInstance())
            .authorizeExchange(auth -> auth
                .pathMatchers("/api/auth/**").permitAll()
                .anyExchange().authenticated()
            )
            .build();
    }

    // Actuator 체인 (IP 제한)
    @Bean @Order(2)
    public SecurityWebFilterChain actuatorChain(ServerHttpSecurity http) {
        return http
            .securityMatcher(new PathPatternParserServerWebExchangeMatcher("/actuator/**"))
            .authorizeExchange(auth -> auth
                .pathMatchers("/actuator/health").permitAll()
                .anyExchange().hasIpAddress("127.0.0.1")
            )
            .build();
    }
}
```

### 실험 2: 커스텀 ReactiveAuthorizationManager

```java
// 리소스 소유자만 접근 허용하는 커스텀 인가
@Component
public class ResourceOwnerAuthorizationManager
        implements ReactiveAuthorizationManager<AuthorizationContext> {

    @Override
    public Mono<AuthorizationDecision> check(
            Mono<Authentication> authentication,
            AuthorizationContext context) {

        ServerWebExchange exchange = context.getExchange();
        Long resourceId = Long.parseLong(
            exchange.getRequest().getPath().pathWithinApplication()
                .elements().get(2).value()  // /resources/{id}
        );

        return authentication
            .flatMap(auth -> resourceService.findById(resourceId)
                .map(resource ->
                    new AuthorizationDecision(
                        resource.getOwnerId().equals(
                            Long.parseLong(auth.getName())
                        )
                    )
                )
            );
    }
}

// 사용
.pathMatchers("/resources/{id}").access(resourceOwnerAuthorizationManager)
```

### 실험 3: 현재 사용자 접근 방법

```java
@RestController
@RequestMapping("/api")
public class MeController {

    // 방법 1: ReactiveSecurityContextHolder (명시적)
    @GetMapping("/me/explicit")
    public Mono<UserResponse> getMeExplicit() {
        return ReactiveSecurityContextHolder.getContext()
            .map(SecurityContext::getAuthentication)
            .flatMap(auth -> userService.findById(
                Long.parseLong(auth.getName())
            ))
            .map(UserResponse::from);
    }

    // 방법 2: @AuthenticationPrincipal (자동 주입)
    @GetMapping("/me/injected")
    public Mono<UserResponse> getMeInjected(
            @AuthenticationPrincipal Mono<UserPrincipal> principal) {
        return principal
            .flatMap(p -> userService.findById(p.getId()))
            .map(UserResponse::from);
    }

    // 방법 3: ServerWebExchange (하위 수준)
    @GetMapping("/me/exchange")
    public Mono<UserResponse> getMeExchange(ServerWebExchange exchange) {
        return exchange.getPrincipal()
            .cast(Authentication.class)
            .flatMap(auth -> userService.findById(
                Long.parseLong(auth.getName())
            ))
            .map(UserResponse::from);
    }
}
```

---

## 📊 성능 비교

```
MVC Security vs Reactive Security 필터 오버헤드:

MVC (Servlet Filter):
  각 필터: 동기 메서드 호출
  ThreadLocal에서 SecurityContext 읽기/쓰기: ~수십 ns
  필터 체인 10개: ~수백 ns 오버헤드

Reactive Security (WebFilter):
  각 필터: Mono<Void> 래핑
  Reactor Context에서 SecurityContext 접근: ~수십 ns
  필터 체인 10개: ~수백 ns 오버헤드

실질적 차이: 거의 없음
  → 보안 필터 오버헤드보다 인증 처리(DB 조회 등)가 훨씬 큼

NoOpServerSecurityContextRepository vs WebSession:
  NoOp (JWT): 요청마다 토큰 파싱/검증 ~1~5ms
  WebSession: 세션 조회 (Redis 등) ~1~10ms
  → JWT가 세션보다 빠른 경우가 많음 (외부 저장소 불필요)
```

---

## ⚖️ 트레이드오프

```
SecurityWebFilterChain 수:
  하나로 통합: 설정 단순
  여러 체인 분리: 경로별 다른 보안 정책 (API vs Admin vs Actuator)
  → 복잡한 서비스는 체인 분리 권장

NoOp vs WebSession SecurityContextRepository:
  NoOp (JWT): 무상태, 수평 확장 용이, 토큰 검증 비용
  WebSession: 상태 있음, 세션 저장소 필요 (Redis 등), 토큰 파싱 없음

인가 방식:
  pathMatchers: URL 기반, 빠름, 세분화 한계
  @PreAuthorize: 메서드 기반, 유연, AOP 오버헤드 약간
  ReactiveAuthorizationManager: 완전 커스텀, 복잡한 로직 가능
```

---

## 📌 핵심 정리

```
Reactive Security 핵심:

FilterChainProxy → SecurityWebFilterChain:
  Servlet Filter → WebFilter (Mono<Void> 반환)
  ThreadLocal → Reactor Context (SecurityContext 저장)

주요 컴포넌트:
  ServerHttpSecurity: WebFlux용 보안 설정 DSL
  ReactiveAuthenticationManager: Mono<Authentication> 반환
  ReactiveUserDetailsService: Mono<UserDetails> 반환
  ServerSecurityContextRepository: Context 저장 전략

현재 사용자 접근:
  ReactiveSecurityContextHolder.getContext()
  @AuthenticationPrincipal Mono<UserDetails>
  exchange.getPrincipal()

무상태 JWT:
  NoOpServerSecurityContextRepository
  .securityContextRepository(NoOp.getInstance())
  요청마다 JWT 파싱/검증 → Context에 주입
```

---

## 🤔 생각해볼 문제

**Q1.** `@EnableWebFluxSecurity`와 `@EnableWebSecurity`는 같은 환경에서 함께 사용할 수 있나요?

<details>
<summary>해설 보기</summary>

함께 사용할 수 없습니다. Spring Boot 애플리케이션에서 WebFlux를 사용하면 `@EnableWebFluxSecurity`가 자동으로 활성화됩니다. `@EnableWebSecurity`는 Servlet 기반 MVC 전용입니다.

`spring-boot-starter-security` + `spring-boot-starter-webflux` 조합:
- `@EnableWebFluxSecurity`는 `@SpringBootApplication`이 있으면 자동 설정됨
- `@Configuration` 클래스에 `@EnableWebFluxSecurity`를 추가해도 동일 효과 (중복이지만 무해)
- `@EnableWebSecurity`는 Servlet Context가 없으면 빈 등록 실패

단, 동일 JVM에서 WebFlux와 MVC를 혼용하는 것은 불가합니다. 둘 중 하나만 선택해야 하므로 보안 설정도 자연히 하나만 사용합니다.

</details>

---

**Q2.** `SecurityWebFilterChain`에서 특정 요청만 보안 적용에서 제외하는 올바른 방법은 무엇인가요?

<details>
<summary>해설 보기</summary>

두 가지 방법이 있으며 의미가 다릅니다.

**방법 1: `permitAll()`** - 보안 체인 내에서 인증 없이 허용
```java
.authorizeExchange(auth -> auth
    .pathMatchers("/api/public/**").permitAll()
    .anyExchange().authenticated()
)
```
→ 보안 필터 체인은 모두 거침 (CSRF, 헤더 등 적용)
→ 단지 인가에서 허용

**방법 2: `WebSecurityCustomizer`** - 보안 체인 자체를 건너뜀 (WebFlux 미지원)
MVC에서는 `WebSecurityCustomizer.configure(WebSecurity)`로 특정 경로를 보안 체인에서 완전히 제외할 수 있습니다.

WebFlux에서는 이와 동등한 기능이 없습니다. 대신 별도 `SecurityWebFilterChain`을 높은 Order로 등록하여 특정 경로는 `permitAll()`로 처리합니다.

```java
@Bean @Order(Ordered.HIGHEST_PRECEDENCE)
public SecurityWebFilterChain staticResourceChain(ServerHttpSecurity http) {
    return http
        .securityMatcher(new PathPatternParserServerWebExchangeMatcher("/static/**"))
        .authorizeExchange(auth -> auth.anyExchange().permitAll())
        .build();
}
```

</details>

---

**Q3.** `ReactiveAuthorizationManager`에서 DB를 조회하는 인가 로직이 성능에 미치는 영향은 어떻게 최소화하나요?

<details>
<summary>해설 보기</summary>

인가 로직에서 DB 조회가 매 요청마다 일어나면 레이턴시가 증가합니다. 다음 방법으로 최소화합니다.

**1. JWT 페이로드에 권한 포함** (가장 효율적):
```java
// JWT 생성 시 roles 클레임 포함
Jwts.builder()
    .claim("roles", user.getRoles())  // ["ROLE_ADMIN", "ROLE_USER"]
    .build();

// 검증 시 DB 조회 없이 토큰에서 권한 추출
```

**2. 캐싱**:
```java
return authentication.flatMap(auth ->
    permissionCache.get(auth.getName() + ":" + resourceId,
        () -> resourceService.findById(resourceId)
            .map(r -> new AuthorizationDecision(r.isOwner(auth)))
    )
);
```

**3. 인가 정보를 SecurityContext에 미리 로드**:
JWT 파싱 시 권한 정보를 `GrantedAuthority`로 로드하면, `hasRole()`/`hasAuthority()` 기반 인가는 추가 DB 조회 없이 동작합니다. 리소스 소유자 확인처럼 동적 인가만 DB를 조회합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: JWT 인증 Reactive 구현 ➡️](./02-jwt-reactive-auth.md)**

</div>
