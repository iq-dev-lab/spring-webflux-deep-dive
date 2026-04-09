# OAuth2 Reactive — ReactiveOAuth2AuthorizedClientManager

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Reactive 기반 OAuth2 Login 흐름은 MVC와 어떻게 다른가?
- `ReactiveOAuth2AuthorizedClientManager`는 토큰을 어떻게 관리하고 갱신하는가?
- `ServerOAuth2AuthorizedClientExchangeFilterFunction`은 어떻게 WebClient에 OAuth2 토큰을 자동 첨부하는가?
- OAuth2 Client Credentials 흐름은 서비스 간 통신에서 어떻게 사용하는가?
- `ReactiveOAuth2AuthorizedClientRepository`는 토큰을 어디에 저장하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

MSA에서 서비스 간 인증(Machine-to-Machine)은 OAuth2 Client Credentials 흐름으로 처리하고, 사용자 인증은 OAuth2 Authorization Code 흐름을 사용합니다. WebFlux에서 이를 구현할 때 `OAuth2AuthorizedClientManager`(MVC 버전)를 그대로 사용하면 블로킹이 발생합니다. `ReactiveOAuth2AuthorizedClientManager`와 WebClient의 `ExchangeFilterFunction` 조합으로 토큰 관리와 자동 갱신을 완전 논블로킹으로 구현할 수 있습니다.

---

## 😱 흔한 실수 (Before — OAuth2를 WebFlux에서 잘못 사용)

```
실수 1: OAuth2AuthorizedClientManager (MVC 버전)를 WebFlux에 사용

  @Bean
  public OAuth2AuthorizedClientManager authorizedClientManager(...) {
      // MVC 버전 → WebFlux에서 블로킹
      return new DefaultOAuth2AuthorizedClientManager(...);
  }
  
  // 올바른 방식:
  @Bean
  public ReactiveOAuth2AuthorizedClientManager authorizedClientManager(...) {
      return new DefaultReactiveOAuth2AuthorizedClientManager(...);
  }

실수 2: 토큰을 직접 관리 (갱신 로직 누락)

  // 직접 토큰 조회 (만료 체크 없음)
  Mono<OAuth2AuthorizedClient> client =
      authorizedClientRepository.loadAuthorizedClient(
          "my-provider", auth, exchange
      );
  // 토큰 만료 시 자동 갱신 없음 → 401 에러 발생
  
  // 올바른 방식: ReactiveOAuth2AuthorizedClientManager 사용
  // → 만료 감지 + 자동 갱신 처리

실수 3: WebClient에 토큰을 매 요청마다 수동으로 첨부

  return authorizedClientManager.authorize(request)
      .flatMap(client -> {
          String token = client.getAccessToken().getTokenValue();
          return webClient.get().uri("/api")
              .header("Authorization", "Bearer " + token)
              .retrieve()...
      });
  // → 토큰 만료 시 401, 재시도 로직 없음
  // 올바른 방식: ExchangeFilterFunction 사용
```

---

## ✨ 올바른 접근 (After — Reactive OAuth2 자동 토큰 관리)

```
WebClient + OAuth2 자동 토큰 첨부:

  @Bean
  public WebClient externalServiceClient(
          ReactiveOAuth2AuthorizedClientManager authorizedClientManager) {

      ServerOAuth2AuthorizedClientExchangeFilterFunction oauth2 =
          new ServerOAuth2AuthorizedClientExchangeFilterFunction(
              authorizedClientManager);
      oauth2.setDefaultClientRegistrationId("my-service");

      return WebClient.builder()
          .baseUrl("https://external-service.com")
          .filter(oauth2)  // 자동 토큰 첨부 + 만료 시 갱신
          .build();
  }

  // 사용
  externalServiceClient.get()
      .uri("/resource")
      .retrieve()
      .bodyToMono(Resource.class);
  // → 자동으로 Bearer 토큰 첨부
  // → 토큰 만료 시 자동 갱신 후 재시도
```

---

## 🔬 내부 동작 원리

### 1. Reactive OAuth2 Login 흐름

```
OAuth2 Authorization Code 흐름 (WebFlux):

  사용자 → GET /oauth2/authorization/{registrationId}
            ↓
  OAuth2AuthorizationRequestRedirectWebFilter
    → Authorization Request 생성
    → State 파라미터 (CSRF 방지)
    → ServerWebSession에 저장
    → 302 → Authorization Server

  사용자 → Authorization Server 로그인/동의
            ↓
  Authorization Server → GET /login/oauth2/code/{registrationId}?code=xxx&state=xxx
            ↓
  OAuth2LoginAuthenticationWebFilter
    → State 검증 (세션에서)
    → code → Authorization Server로 토큰 교환 요청
    → Access Token + Refresh Token 수신
    → ReactiveOAuth2AuthorizedClientRepository에 저장
    → SecurityContext에 OAuth2AuthenticationToken 저장
    → 원래 URL로 리다이렉트

MVC와의 차이:
  MVC: Servlet Filter + HttpSession + 블로킹 HTTP 호출
  WebFlux: WebFilter + WebSession + 논블로킹 WebClient 토큰 교환
```

### 2. ReactiveOAuth2AuthorizedClientManager

```
역할: 토큰 획득, 갱신, 저장 관리

  DefaultReactiveOAuth2AuthorizedClientManager:
    주입: ReactiveClientRegistrationRepository (클라이언트 설정)
          ServerOAuth2AuthorizedClientRepository (토큰 저장소)

  authorize(OAuth2AuthorizeRequest) 처리:
    1. ServerOAuth2AuthorizedClientRepository에서 기존 토큰 로드
    2. 토큰 없음 → 새 토큰 획득 (흐름에 따라 다름)
    3. 토큰 만료 → 갱신 시도 (Refresh Token 사용)
    4. 토큰 저장 후 반환

  @Bean
  public ReactiveOAuth2AuthorizedClientManager authorizedClientManager(
          ReactiveClientRegistrationRepository clientRegistrationRepository,
          ServerOAuth2AuthorizedClientRepository authorizedClientRepository) {

      ReactiveOAuth2AuthorizedClientProvider provider =
          ReactiveOAuth2AuthorizedClientProviderBuilder.builder()
              .authorizationCode()     // 사용자 로그인
              .refreshToken()          // 토큰 갱신
              .clientCredentials()     // M2M 통신
              .build();

      DefaultReactiveOAuth2AuthorizedClientManager manager =
          new DefaultReactiveOAuth2AuthorizedClientManager(
              clientRegistrationRepository, authorizedClientRepository);
      manager.setAuthorizedClientProvider(provider);
      return manager;
  }
```

### 3. Client Credentials 흐름 (서비스 간 통신)

```
application.yml 설정:
  spring:
    security:
      oauth2:
        client:
          registration:
            payment-service:
              provider: auth-server
              client-id: api-gateway
              client-secret: ${OAUTH2_SECRET}
              authorization-grant-type: client_credentials
              scope: payment:read,payment:write
          provider:
            auth-server:
              token-uri: https://auth.example.com/oauth2/token

  Client Credentials 흐름:
    1. WebClient가 외부 서비스 호출
    2. ExchangeFilterFunction이 토큰 필요 감지
    3. ReactiveOAuth2AuthorizedClientManager.authorize() 호출
    4. ClientCredentialsReactiveOAuth2AuthorizedClientProvider:
       a. 기존 토큰 확인 (인메모리 캐시)
       b. 없거나 만료 → token-uri에 POST (client_credentials)
       c. 새 Access Token 수신
    5. Authorization: Bearer <token> 헤더 추가
    6. 원래 요청 전송

  토큰 갱신 자동화:
    만료 5초 전부터 갱신 시도 (clockSkew 설정)
    갱신 실패 시 → 새 토큰 획득
    → 개발자가 토큰 만료 처리 코드 작성 불필요
```

### 4. ServerOAuth2AuthorizedClientExchangeFilterFunction

```java
// WebClient에 OAuth2 토큰 자동 첨부하는 필터

@Bean
public WebClient paymentWebClient(
        ReactiveOAuth2AuthorizedClientManager authorizedClientManager) {

    ServerOAuth2AuthorizedClientExchangeFilterFunction oauth2Filter =
        new ServerOAuth2AuthorizedClientExchangeFilterFunction(
            authorizedClientManager);

    // Client Credentials (M2M): 기본 클라이언트 지정
    oauth2Filter.setDefaultClientRegistrationId("payment-service");

    // Authorization Code (사용자 로그인): 현재 사용자 토큰 사용
    // oauth2Filter.setDefaultOAuth2AuthorizedClient(true);

    return WebClient.builder()
        .baseUrl("https://payment-service.example.com")
        .filter(oauth2Filter)
        .build();
}

// 내부 동작:
// 1. 요청 전: authorizedClientManager.authorize() 호출
// 2. Access Token 획득 (캐시 또는 신규 발급)
// 3. Authorization: Bearer <token> 헤더 추가
// 4. 요청 전송
// 5. 401 응답 시: 토큰 갱신 후 재시도

// 요청별 다른 클라이언트 사용:
webClient.get()
    .uri("/resource")
    .attributes(clientRegistrationId("other-service"))  // 요청별 지정
    .retrieve()
    .bodyToMono(Resource.class);
```

### 5. 토큰 저장소 — ReactiveOAuth2AuthorizedClientRepository

```
저장소 종류:

1. WebSessionServerOAuth2AuthorizedClientRepository (기본):
   → WebSession에 저장 (브라우저 기반 앱)
   → 사용자 로그아웃 시 자동 정리
   → 단점: 세션 저장소 필요 (클러스터 환경)

2. InMemoryReactiveOAuth2AuthorizedClientService:
   → JVM 메모리 내 저장
   → 서버 재시작 시 모든 토큰 소멸
   → 개발/테스트 환경에 적합

3. 커스텀 저장소 (Redis):
   @Bean
   public ServerOAuth2AuthorizedClientRepository authorizedClientRepository(
           ReactiveOAuth2AuthorizedClientService authorizedClientService) {
       return new AuthenticatedPrincipalServerOAuth2AuthorizedClientRepository(
           authorizedClientService);
   }

   @Bean
   public ReactiveOAuth2AuthorizedClientService authorizedClientService(
           ReactiveClientRegistrationRepository clientRegistrationRepository) {
       return new RedisOAuth2AuthorizedClientService(
           redisTemplate, clientRegistrationRepository);
   }
   // 커스텀 Redis 저장소로 수평 확장 가능
```

---

## 💻 실전 코드

### 실험 1: 완전한 OAuth2 Client Credentials 설정

```java
@Configuration
@EnableWebFluxSecurity
public class OAuth2Config {

    @Bean
    public ReactiveOAuth2AuthorizedClientManager authorizedClientManager(
            ReactiveClientRegistrationRepository repo,
            ServerOAuth2AuthorizedClientRepository store) {

        ReactiveOAuth2AuthorizedClientProvider provider =
            ReactiveOAuth2AuthorizedClientProviderBuilder.builder()
                .clientCredentials(c ->
                    c.clockSkew(Duration.ofSeconds(30))  // 만료 30초 전 갱신
                )
                .refreshToken()
                .build();

        DefaultReactiveOAuth2AuthorizedClientManager manager =
            new DefaultReactiveOAuth2AuthorizedClientManager(repo, store);
        manager.setAuthorizedClientProvider(provider);
        return manager;
    }

    @Bean
    public WebClient inventoryWebClient(
            ReactiveOAuth2AuthorizedClientManager manager) {

        var oauth2 = new ServerOAuth2AuthorizedClientExchangeFilterFunction(manager);
        oauth2.setDefaultClientRegistrationId("inventory-service");

        return WebClient.builder()
            .baseUrl("https://inventory.example.com")
            .filter(oauth2)
            .filter(loggingFilter())
            .build();
    }
}
```

### 실험 2: 사용자 OAuth2 토큰으로 외부 API 호출

```java
@Service
@RequiredArgsConstructor
public class UserGitHubService {

    private final WebClient webClient;

    // 현재 로그인한 사용자의 GitHub 토큰으로 API 호출
    public Flux<Repository> getUserRepositories(
            @RegisteredOAuth2AuthorizedClient("github")
            OAuth2AuthorizedClient authorizedClient) {

        String token = authorizedClient.getAccessToken().getTokenValue();

        return webClient.get()
            .uri("https://api.github.com/user/repos")
            .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
            .retrieve()
            .bodyToFlux(Repository.class);
    }

    // WebClient에 사용자 토큰 자동 첨부 (ExchangeFilterFunction)
    public Mono<GitHubUser> getGitHubUser(Authentication auth) {
        return ReactiveSecurityContextHolder.getContext()
            .map(SecurityContext::getAuthentication)
            .cast(OAuth2AuthenticationToken.class)
            .flatMap(oauthAuth ->
                webClient.get()
                    .uri("https://api.github.com/user")
                    .attributes(oauth2AuthorizedClient(
                        oauthAuth.getAuthorizedClientRegistrationId()
                    ))
                    .retrieve()
                    .bodyToMono(GitHubUser.class)
            );
    }
}
```

### 실험 3: OAuth2 토큰 검사 및 갱신

```java
@Service
@RequiredArgsConstructor
public class TokenManagementService {

    private final ReactiveOAuth2AuthorizedClientManager authorizedClientManager;

    // 토큰 상태 확인 및 필요 시 갱신
    public Mono<String> getValidToken(String clientId, Authentication auth,
                                      ServerWebExchange exchange) {
        OAuth2AuthorizeRequest request = OAuth2AuthorizeRequest
            .withClientRegistrationId(clientId)
            .principal(auth)
            .attribute(ServerWebExchange.class.getName(), exchange)
            .build();

        return authorizedClientManager.authorize(request)
            .map(client -> client.getAccessToken().getTokenValue())
            .doOnNext(token ->
                log.debug("토큰 획득: clientId={}", clientId)
            )
            .onErrorMap(OAuth2AuthorizationException.class, e ->
                new ServiceException("OAuth2 인증 실패: " + e.getError().getDescription())
            );
    }
}
```

---

## 📊 성능 비교

```
토큰 획득 전략별 성능:

인메모리 캐시 (토큰 유효):
  tokenValue 즉시 반환: ~수 ns
  오버헤드 없음 → 권장 (만료 전까지)

토큰 만료 → 갱신 (Refresh Token):
  Token 갱신 HTTP 요청: ~50~200ms
  요청당 1회 (갱신 후 캐시)

토큰 만료 → 신규 발급 (Client Credentials):
  Authorization Server HTTP 요청: ~50~200ms
  clockSkew 30초: 만료 30초 전에 미리 갱신
  → 실제 API 요청 시 지연 없음

첫 요청 시나리오 (토큰 없음):
  Cold Start: 첫 요청 → 토큰 발급 → 실제 요청
  → 첫 요청만 지연 (100~300ms)
  → 이후 캐시 활용 → 즉시

병렬 요청 토큰 갱신 (Race Condition):
  여러 요청이 동시에 토큰 갱신 시도 → 중복 발급 위험
  Reactor의 Mono.cache() 또는 Sinks 활용으로 단일 갱신 보장
```

---

## ⚖️ 트레이드오프

```
Authorization Code vs Client Credentials:
  Authorization Code (사용자 대리):
    사용자 동의 필요, 브라우저 리다이렉트
    사용자별 토큰 (각 사용자의 권한 범위)
    적합: 사용자 데이터 접근 (GitHub, Google API)

  Client Credentials (서비스 자신):
    서버 간 통신, 브라우저 불필요
    서비스 단위 토큰 (전체 접근 또는 스코프 기반)
    적합: MSA 내 서비스 간 인증

토큰 저장소:
  WebSession: 사용자별 토큰, 세션 만료 시 정리
  InMemory: 빠름, 서버 재시작 시 소멸, 수평 확장 어려움
  Redis: 영속성, 공유 가능, 추가 인프라 필요

clockSkew (만료 전 갱신 시점):
  너무 이름 (e.g., 5분): 토큰 자주 갱신 → Authorization Server 부하
  너무 늦음 (e.g., 0초): 경쟁 조건에서 만료 토큰 사용 가능
  권장: 30~60초
```

---

## 📌 핵심 정리

```
OAuth2 Reactive 핵심:

MVC vs WebFlux:
  OAuth2AuthorizedClientManager → ReactiveOAuth2AuthorizedClientManager
  OAuth2AuthorizedClientRepository → ReactiveOAuth2AuthorizedClientRepository
  모든 반환값이 Mono<T>

자동 토큰 관리:
  ServerOAuth2AuthorizedClientExchangeFilterFunction
  → WebClient에 filter로 등록
  → 자동 토큰 첨부 + 만료 시 자동 갱신

Client Credentials (M2M):
  setDefaultClientRegistrationId("service-name")
  → 모든 요청에 서비스 토큰 자동 첨부

저장소:
  WebSession: 브라우저 앱 기본
  InMemory: 개발/테스트
  커스텀(Redis): 프로덕션 수평 확장

clockSkew:
  토큰 만료 전 30~60초에 미리 갱신
  실제 요청 시 지연 없도록
```

---

## 🤔 생각해볼 문제

**Q1.** OAuth2 Login 후 원래 요청한 URL로 리다이렉트하는 과정에서 WebFlux는 어떻게 원래 URL을 기억하나요?

<details>
<summary>해설 보기</summary>

`OAuth2AuthorizationRequestRedirectWebFilter`가 인증이 필요한 요청을 가로채면, 원래 요청 URL을 `ServerWebSession`에 `RedirectServerAuthenticationSuccessHandler.SAVED_REQUEST_SESSION_KEY`로 저장합니다.

Authorization Server에서 인증 후 콜백이 오면 `RedirectServerAuthenticationSuccessHandler`가 세션에서 원래 URL을 꺼내 리다이렉트합니다.

```java
// 기본 동작 커스터마이징
@Bean
public ServerAuthenticationSuccessHandler authSuccessHandler() {
    var handler = new RedirectServerAuthenticationSuccessHandler();
    handler.setRequestCache(new WebSessionServerRequestCache());
    return handler;
}
```

세션리스(JWT) 환경에서는 세션에 저장하는 것이 불가능합니다. 이 경우 `state` 파라미터에 원래 URL을 인코딩하거나 쿠키에 저장하는 커스텀 구현이 필요합니다.

</details>

---

**Q2.** 여러 OAuth2 제공자(Google, GitHub)를 동시에 지원할 때 `WebClient`에서 올바른 토큰을 사용하려면 어떻게 설정하나요?

<details>
<summary>해설 보기</summary>

제공자별로 다른 `WebClient` 빈을 등록하거나, 요청마다 `clientRegistrationId`를 지정합니다.

```java
// 방법 1: 제공자별 WebClient 빈
@Bean("googleWebClient")
public WebClient googleWebClient(ReactiveOAuth2AuthorizedClientManager manager) {
    var oauth2 = new ServerOAuth2AuthorizedClientExchangeFilterFunction(manager);
    oauth2.setDefaultClientRegistrationId("google");
    return WebClient.builder()
        .baseUrl("https://www.googleapis.com")
        .filter(oauth2).build();
}

@Bean("githubWebClient")
public WebClient githubWebClient(ReactiveOAuth2AuthorizedClientManager manager) {
    var oauth2 = new ServerOAuth2AuthorizedClientExchangeFilterFunction(manager);
    oauth2.setDefaultClientRegistrationId("github");
    return WebClient.builder()
        .baseUrl("https://api.github.com")
        .filter(oauth2).build();
}

// 방법 2: 요청별 지정
sharedWebClient.get().uri("/api")
    .attributes(clientRegistrationId("github"))  // 이 요청만 GitHub 토큰
    .retrieve()...
```

사용자 로그인 OAuth2 토큰을 서비스에서 사용할 때는 `@RegisteredOAuth2AuthorizedClient("provider")` 파라미터로 현재 사용자의 해당 제공자 토큰을 주입받을 수 있습니다.

</details>

---

**Q3.** `ReactiveOAuth2AuthorizedClientManager`에서 토큰 갱신이 실패하면 어떻게 처리되나요?

<details>
<summary>해설 보기</summary>

Refresh Token이 만료되거나 취소된 경우 `OAuth2AuthorizationException`이 발생합니다. `ExchangeFilterFunction`의 `onErrorResume`으로 처리할 수 있습니다.

```java
ServerOAuth2AuthorizedClientExchangeFilterFunction oauth2Filter = ...;

WebClient webClient = WebClient.builder()
    .filter(oauth2Filter)
    .filter(ExchangeFilterFunction.ofResponseProcessor(response -> {
        if (response.statusCode() == HttpStatus.UNAUTHORIZED) {
            // 토큰 갱신 실패로 인한 401 처리
            return Mono.error(new TokenRefreshException("재로그인 필요"));
        }
        return Mono.just(response);
    }))
    .build();

// 서비스 레벨에서 처리
public Mono<Resource> getResource() {
    return webClient.get().uri("/resource").retrieve()
        .bodyToMono(Resource.class)
        .onErrorResume(TokenRefreshException.class, e ->
            // 사용자에게 재로그인 요청 또는 에러 응답
            Mono.error(new ResponseStatusException(
                HttpStatus.UNAUTHORIZED, "세션이 만료되었습니다"))
        );
}
```

`ServerOAuth2AuthorizedClientExchangeFilterFunction`에는 자체적으로 `setAuthorizationFailureHandler()`를 통해 인가 실패 시 처리를 커스터마이징할 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Method Security](./03-method-security.md)** | **[홈으로 🏠](../README.md)**

</div>
