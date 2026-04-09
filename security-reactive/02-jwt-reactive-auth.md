# JWT 인증 Reactive 구현 — WebFilter에서 토큰 검증

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JWT 검증을 `WebFilter`에서 어떻게 비동기로 처리하는가?
- 검증된 인증 정보를 Reactor Context에 어떻게 주입하는가?
- 토큰 갱신(Refresh Token)을 Reactive 파이프라인에서 어떻게 구현하는가?
- 필터 단락(short-circuit)은 어떻게 구현하고 언제 사용하는가?
- JWT 비밀키 관리와 RS256 공개키 기반 검증은 어떻게 차이나는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

JWT 인증은 MSA에서 가장 흔한 인증 방식입니다. WebFlux에서 JWT를 올바르게 구현하지 않으면 — 예를 들어 토큰 파싱을 동기로 처리하거나 SecurityContext 주입을 잘못하면 — 매 요청마다 인증이 풀리거나 블로킹이 발생합니다. JWT WebFilter의 전체 흐름을 손으로 구현해보면 Spring Security 내부 동작의 많은 부분이 이해됩니다.

---

## 😱 흔한 실수 (Before — JWT WebFilter를 잘못 구현)

```
실수 1: JWT 파싱 블로킹 처리 + subscribeOn 누락

  public Mono<Void> filter(ServerWebExchange exchange,
                           WebFilterChain chain) {
      String token = extractToken(exchange);
      Claims claims = Jwts.parser().parseClaimsJws(token).getBody();
      // JWT 파싱 자체는 CPU 작업 (빠름, 블로킹 아님)
      // 단, DB 검증(토큰 블랙리스트 등)은 블로킹 주의
      ...
  }

실수 2: SecurityContext 주입 없이 다음 필터에 전달

  return chain.filter(exchange);
  // SecurityContext가 없으면 @PreAuthorize, hasRole 등 동작 안 함
  // → 인가 필터에서 "미인증"으로 처리 → 403/401

실수 3: 에러 발생 시 다음 필터로 계속 진행

  return jwtService.validate(token)
      .flatMap(auth -> chain.filter(exchange))
      .onErrorResume(e -> chain.filter(exchange));
      // 토큰 검증 실패해도 다음 필터 실행 → 보안 구멍!
  
  // 올바른 방식: 실패 시 즉시 401 응답
  .onErrorResume(e -> {
      exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
      return exchange.getResponse().setComplete();
  });
```

---

## ✨ 올바른 접근 (After — JWT WebFilter 완전 구현)

```
JWT WebFilter 핵심 흐름:

1. Authorization 헤더에서 Bearer 토큰 추출
2. 토큰 파싱 및 서명 검증 (CPU, 빠름)
3. 토큰 블랙리스트 확인 (선택적, Redis 조회)
4. Authentication 객체 생성
5. SecurityContext에 Authentication 저장
6. Reactor Context에 SecurityContext 주입
7. chain.filter(mutatedExchange or contextWrite)

단락(short-circuit) 조건:
  - Authorization 헤더 없음 → 다음 필터로 (인가 필터가 처리)
  - 토큰 파싱 실패 → 즉시 401
  - 토큰 만료 → 즉시 401
  - 블랙리스트 → 즉시 401
```

---

## 🔬 내부 동작 원리

### 1. JWT WebFilter 전체 흐름

```
요청 처리 흐름:

  HTTP 요청 도착
    ↓
  SecurityWebFilterChain의 WebFilter 체인 진입
    ↓
  JwtAuthenticationFilter (SecurityWebFiltersOrder.AUTHENTICATION)
    ↓
  Authorization 헤더 확인
    ├─ 없음 → chain.filter(exchange) (다음 필터로, 인가가 처리)
    └─ 있음 ("Bearer <token>")
         ↓
       jwtService.validateToken(token) → Mono<Authentication>
         ├─ 성공 → SecurityContext 생성
         │         → contextWrite(SecurityContext 주입)
         │         → chain.filter(exchange)
         └─ 실패 → 즉시 401 응답 (단락)

SecurityContext 주입 방식:
  // contextWrite로 Reactor Context에 SecurityContext 주입
  return chain.filter(exchange)
      .contextWrite(ReactiveSecurityContextHolder
          .withAuthentication(authentication));
  // 이후 파이프라인에서 ReactiveSecurityContextHolder.getContext()로 접근 가능
```

### 2. JWT 파싱 — 동기 vs 비동기

```
JWT 파싱 자체 (서명 검증):
  HMAC-SHA256 (HS256): CPU 연산만, 수 μs
  RSA/EC (RS256/ES256): CPU 연산만, 수십~수백 μs
  → 블로킹 없음, EventLoop에서 안전

  Mono.fromCallable(() -> jwtParser.parseClaimsJws(token))
  // fromCallable 사용 이유: 예외를 onError로 변환하기 위해

비동기가 필요한 경우:
  토큰 블랙리스트 확인 (Redis 조회):
    jwtService.isBlacklisted(jti)  // Mono<Boolean>
  토큰 DB 검증 (불투명 토큰):
    tokenRepository.findByToken(token)  // Mono<TokenEntity>

RS256 공개키 기반 검증:
  // 공개키로 서명 검증 (비밀키 서버 불필요)
  @Bean
  public JwtParser jwtParser() {
      return Jwts.parserBuilder()
          .setSigningKey(rsaPublicKey)  // 공개키만 있으면 됨
          .build();
  }
  // MSA: 인증 서버(비밀키) → 다른 서비스(공개키)
  // 각 서비스가 독립적으로 토큰 검증 가능
```

### 3. 완전한 JwtAuthenticationFilter 구현

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter implements WebFilter {

    private final JwtService jwtService;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange,
                             WebFilterChain chain) {

        return Mono.justOrEmpty(extractToken(exchange))
            .flatMap(token -> jwtService.validateToken(token))  // Mono<Authentication>
            .flatMap(authentication ->
                chain.filter(exchange)
                    .contextWrite(ReactiveSecurityContextHolder
                        .withAuthentication(authentication))
            )
            .switchIfEmpty(chain.filter(exchange))  // 토큰 없음 → 다음 필터
            .onErrorResume(JwtException.class, e -> {
                // 토큰 검증 실패 → 즉시 401
                exchange.getResponse()
                    .setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            });
    }

    private String extractToken(ServerWebExchange exchange) {
        String authHeader = exchange.getRequest().getHeaders()
            .getFirst(HttpHeaders.AUTHORIZATION);
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            return authHeader.substring(7);
        }
        return null;
    }
}

@Service
@RequiredArgsConstructor
public class JwtService {

    private final JwtParser jwtParser;
    private final ReactiveStringRedisTemplate redisTemplate;

    public Mono<Authentication> validateToken(String token) {
        return Mono.fromCallable(() -> {
            // 1. 서명 검증 및 파싱 (CPU, 빠름)
            Claims claims = jwtParser
                .parseClaimsJws(token).getBody();

            String userId = claims.getSubject();
            String jti = claims.getId();  // JWT ID (유니크)
            List<String> roles = claims.get("roles", List.class);

            return new TokenData(userId, jti, roles);
        })
        // 2. 블랙리스트 확인 (Redis, 비동기)
        .flatMap(data ->
            redisTemplate.hasKey("blacklist:" + data.jti())
                .flatMap(blacklisted -> {
                    if (blacklisted) {
                        return Mono.error(
                            new JwtException("토큰 무효화됨"));
                    }
                    // 3. Authentication 객체 생성
                    List<GrantedAuthority> authorities = data.roles()
                        .stream()
                        .map(SimpleGrantedAuthority::new)
                        .toList();
                    return Mono.just(new UsernamePasswordAuthenticationToken(
                        data.userId(), null, authorities
                    ));
                })
        );
    }

    // Refresh Token 처리
    public Mono<TokenPair> refresh(String refreshToken) {
        return Mono.fromCallable(() ->
            jwtParser.parseClaimsJws(refreshToken).getBody()
        )
        .flatMap(claims ->
            redisTemplate.opsForValue()
                .get("refresh:" + claims.getSubject())
                .filter(stored -> stored.equals(refreshToken))
                .switchIfEmpty(Mono.error(
                    new JwtException("유효하지 않은 Refresh Token")))
                .flatMap(valid -> generateTokenPair(claims.getSubject()))
        );
    }
}
```

### 4. 토큰 발급 — 로그인 엔드포인트

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final ReactiveAuthenticationManager authenticationManager;
    private final JwtTokenProvider tokenProvider;

    @PostMapping("/login")
    public Mono<TokenPair> login(@RequestBody LoginRequest req) {
        Authentication authRequest =
            new UsernamePasswordAuthenticationToken(
                req.username(), req.password()
            );

        return authenticationManager.authenticate(authRequest)
            .flatMap(auth -> tokenProvider.generateTokenPair(auth))
            .onErrorMap(BadCredentialsException.class,
                e -> new ResponseStatusException(
                    HttpStatus.UNAUTHORIZED, "인증 실패"));
    }

    @PostMapping("/refresh")
    public Mono<TokenPair> refresh(@RequestBody RefreshRequest req) {
        return jwtService.refresh(req.refreshToken());
    }

    @PostMapping("/logout")
    public Mono<Void> logout(
            @AuthenticationPrincipal Mono<UserDetails> user,
            @RequestHeader("Authorization") String authHeader) {
        String token = authHeader.substring(7);
        return jwtService.blacklistToken(token)
            .then();
    }
}
```

### 5. 필터 순서와 단락 처리

```
필터 체인에서 JWT 필터 위치:

addFilterAt(jwtFilter, SecurityWebFiltersOrder.AUTHENTICATION)
  → AUTHENTICATION Order(-900)에 배치
  → AUTHORIZATION(-1200) 이전에 실행
  → 인가 전에 인증 완료

단락(short-circuit) 패턴:
  정상 경우:
    JWT 없음 → chain.filter() (인가 필터가 처리)
    JWT 유효 → context 주입 후 chain.filter()
  단락 경우:
    JWT 파싱 실패 → 401 즉시 반환 (chain.filter() 호출 안 함)
    JWT 만료 → 401 즉시 반환
    JWT 블랙리스트 → 401 즉시 반환

SecurityWebFiltersOrder.AUTHENTICATION vs addFilterBefore:
  addFilterAt(filter, ORDER): 해당 Order에 배치 (같은 위치 교체)
  addFilterBefore(filter, clazz): 특정 필터 클래스 이전에 배치
  addFilterAfter(filter, clazz): 특정 필터 클래스 이후에 배치
```

---

## 💻 실전 코드

### 실험 1: 토큰 생성 서비스

```java
@Service
@RequiredArgsConstructor
public class JwtTokenProvider {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.access-token-expiry:3600}")
    private long accessTokenExpiry;  // 초

    @Value("${jwt.refresh-token-expiry:2592000}")
    private long refreshTokenExpiry; // 초 (30일)

    private final ReactiveStringRedisTemplate redisTemplate;

    public Mono<TokenPair> generateTokenPair(Authentication auth) {
        String userId = auth.getName();
        List<String> roles = auth.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .toList();

        String jti = UUID.randomUUID().toString();
        Instant now = Instant.now();

        String accessToken = Jwts.builder()
            .setId(jti)
            .setSubject(userId)
            .claim("roles", roles)
            .setIssuedAt(Date.from(now))
            .setExpiration(Date.from(now.plusSeconds(accessTokenExpiry)))
            .signWith(Keys.hmacShaKeyFor(secret.getBytes()))
            .compact();

        String refreshToken = Jwts.builder()
            .setSubject(userId)
            .setIssuedAt(Date.from(now))
            .setExpiration(Date.from(now.plusSeconds(refreshTokenExpiry)))
            .signWith(Keys.hmacShaKeyFor(secret.getBytes()))
            .compact();

        // Refresh Token을 Redis에 저장
        return redisTemplate.opsForValue()
            .set("refresh:" + userId, refreshToken,
                Duration.ofSeconds(refreshTokenExpiry))
            .thenReturn(new TokenPair(accessToken, refreshToken));
    }

    public Mono<Void> blacklistToken(String token) {
        try {
            Claims claims = Jwts.parserBuilder()
                .setSigningKey(Keys.hmacShaKeyFor(secret.getBytes()))
                .build()
                .parseClaimsJws(token).getBody();
            
            Duration remaining = Duration.between(
                Instant.now(), claims.getExpiration().toInstant());
            
            return redisTemplate.opsForValue()
                .set("blacklist:" + claims.getId(), "1", remaining)
                .then();
        } catch (Exception e) {
            return Mono.empty();
        }
    }
}
```

### 실험 2: WebTestClient로 JWT 인증 테스트

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@AutoConfigureWebTestClient
class JwtAuthTest {

    @Autowired WebTestClient webTestClient;
    @Autowired JwtTokenProvider tokenProvider;

    @Test
    void authenticatedRequest_withValidJwt_shouldSucceed() {
        Authentication auth = new UsernamePasswordAuthenticationToken(
            "user1", null, List.of(new SimpleGrantedAuthority("ROLE_USER"))
        );
        String token = tokenProvider.generateTokenPair(auth).block()
            .accessToken();

        webTestClient.get().uri("/api/users/me")
            .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
            .exchange()
            .expectStatus().isOk();
    }

    @Test
    void unauthenticatedRequest_shouldReturn401() {
        webTestClient.get().uri("/api/users/me")
            .exchange()
            .expectStatus().isUnauthorized();
    }

    @Test
    void expiredToken_shouldReturn401() {
        String expiredToken = Jwts.builder()
            .setSubject("user1")
            .setExpiration(Date.from(Instant.now().minusSeconds(1)))
            .signWith(Keys.hmacShaKeyFor("secret".getBytes()))
            .compact();

        webTestClient.get().uri("/api/users/me")
            .header(HttpHeaders.AUTHORIZATION, "Bearer " + expiredToken)
            .exchange()
            .expectStatus().isUnauthorized();
    }
}
```

---

## 📊 성능 비교

```
JWT 방식별 성능:

HS256 (HMAC-SHA256):
  토큰 생성: ~수 μs
  토큰 검증: ~수 μs
  키: 단일 비밀키 (발급/검증 동일 서버)
  MSA: 모든 서비스가 비밀키 공유 필요 → 키 노출 위험

RS256 (RSA-SHA256):
  토큰 생성: ~수백 μs (RSA 서명)
  토큰 검증: ~수십 μs (공개키 검증)
  키: 비밀키(인증 서버) + 공개키(각 서비스)
  MSA: 각 서비스는 공개키만 보유 → 안전

블랙리스트 Redis 확인:
  Redis 조회: ~1ms
  요청마다 Redis 호출 → 전체 레이턴시 ~1ms 추가
  → 성능 중요 시: 짧은 만료시간 + 블랙리스트 없음 (로그아웃 즉각 무효화 포기)

토큰 캐싱 (파싱 결과):
  동일 토큰 반복 파싱 시: Caffeine 캐시로 파싱 결과 캐시
  TTL = 토큰 만료시간까지
  → Redis 확인도 캐시 가능 (짧은 TTL로)
```

---

## ⚖️ 트레이드오프

```
JWT 무상태 vs 세션 상태:
  JWT:
    장점: 수평 확장 쉬움, 서버 저장소 불필요
    단점: 토큰 즉각 무효화 어려움 (블랙리스트 필요)

  세션:
    장점: 즉각 로그아웃 가능, 서버에서 완전 제어
    단점: 세션 저장소 필요 (Redis 등), 수평 확장 복잡

블랙리스트 전략:
  블랙리스트 없음: 로그아웃 후 토큰 만료까지 유효
  블랙리스트 있음: 즉각 무효화 가능, Redis 추가 호출 발생
  짧은 만료시간: 15분 Access + 30일 Refresh (절충)

RS256 vs HS256:
  HS256: 빠름, 비밀키 공유 필요
  RS256: 느리지만 MSA에서 안전한 검증
  → MSA: RS256, 단일 서버: HS256
```

---

## 📌 핵심 정리

```
JWT Reactive 인증 핵심:

WebFilter 흐름:
  토큰 추출 → 검증 → Authentication 생성
  → contextWrite(withAuthentication(auth))
  → chain.filter(exchange) (다음 필터로)

단락 처리:
  토큰 없음 → switchIfEmpty → chain.filter() (인가에 위임)
  토큰 무효 → onErrorResume → 401 즉시 반환

SecurityContext 주입:
  chain.filter(exchange)
      .contextWrite(ReactiveSecurityContextHolder
          .withAuthentication(authentication))
  → 파이프라인 전체에서 getContext() 접근 가능

토큰 관리:
  블랙리스트: Redis "blacklist:{jti}" 키
  Refresh Token: Redis "refresh:{userId}" 키
  만료 시간 = Redis TTL로 자동 정리
```

---

## 🤔 생각해볼 문제

**Q1.** JWT의 `jti`(JWT ID)를 블랙리스트에 사용하지 않고 `sub`(사용자 ID)를 사용하면 어떤 문제가 생기나요?

<details>
<summary>해설 보기</summary>

`sub`(사용자 ID)를 블랙리스트로 사용하면 해당 사용자의 **모든 토큰**이 무효화됩니다. 이는 원하지 않는 결과를 낳습니다.

예를 들어 사용자가 여러 디바이스에서 로그인했을 때, 하나의 디바이스에서 로그아웃하면 다른 모든 디바이스도 로그아웃됩니다.

`jti`(JWT ID)는 각 토큰마다 유니크한 값입니다. 특정 토큰만 정확히 무효화할 수 있습니다.

```java
// 올바른 방식: jti 블랙리스트
redisTemplate.set("blacklist:" + claims.getId(), "1", remaining);

// 잘못된 방식: sub 블랙리스트
redisTemplate.set("blacklist:" + claims.getSubject(), "1", remaining);
// → 해당 사용자의 모든 토큰 무효화
```

특정 디바이스 로그아웃에는 `jti`, "모든 디바이스 로그아웃"에는 버전 카운터 방식을 사용합니다: 토큰에 `version` 클레임을 포함하고, 로그아웃 시 서버의 버전을 올려 이전 버전 토큰을 모두 무효화합니다.

</details>

---

**Q2.** `contextWrite`를 `chain.filter(exchange)` 앞에 써야 하나요, 뒤에 써야 하나요?

<details>
<summary>해설 보기</summary>

`contextWrite`는 파이프라인의 **뒤**에 위치해야 합니다. Reactor Context는 아래에서 위 방향(subscribe 방향)으로 전파되기 때문입니다.

```java
// 올바른 위치 — contextWrite가 chain.filter() 뒤에
return chain.filter(exchange)
    .contextWrite(ReactiveSecurityContextHolder
        .withAuthentication(authentication));
// chain.filter(exchange)가 subscribe될 때 contextWrite가 Context를 주입

// 잘못된 위치 — contextWrite가 chain.filter() 앞에
return Mono.just(authentication)
    .contextWrite(ReactiveSecurityContextHolder.withAuthentication(auth))
    .flatMap(a -> chain.filter(exchange));
// chain.filter(exchange)의 파이프라인에는 Context가 전파되지 않음
```

`chain.filter(exchange)`가 실행되는 파이프라인에 Context가 전파되려면, `contextWrite`가 해당 `Mono/Flux`의 **하위(downstream)**에 위치해야 합니다. subscribe 방향이 아래에서 위이므로, 코드상으로는 `chain.filter()` **뒤**에 `.contextWrite()`를 붙입니다.

</details>

---

**Q3.** 같은 요청에서 여러 `WebFilter`가 각각 `contextWrite`를 하면 Context가 덮어씌워지나요?

<details>
<summary>해설 보기</summary>

덮어씌워지지 않습니다. Reactor Context는 불변이며, 여러 `contextWrite`가 체인되면 **병합**됩니다.

```java
// Filter1: RequestID Context 추가
chain.filter(exchange).contextWrite(Context.of("requestId", uuid));

// Filter2: SecurityContext 추가
chain.filter(exchange).contextWrite(
    ReactiveSecurityContextHolder.withAuthentication(auth));

// 결과 Context: {"requestId": uuid, SECURITY_CONTEXT_KEY: securityContext}
// 두 키가 모두 존재
```

단, 동일한 키를 두 번 추가하면 나중에 실행되는 `contextWrite`(subscribe 방향으로 더 아래에 위치한 것)가 우선합니다. 실제로는 JWT 필터의 SecurityContext와 RequestId 필터의 requestId가 독립적인 키를 사용하므로 충돌 없이 병합됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: Reactive Security 아키텍처](./01-security-web-filter-chain.md)** | **[홈으로 🏠](../README.md)** | **[다음: Method Security ➡️](./03-method-security.md)**

</div>
