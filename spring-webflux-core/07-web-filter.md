# WebFilter — Reactive 기반 필터 체인

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `WebFilter`가 Servlet `Filter`와 달리 `Mono<Void>`를 반환하는 이유는 무엇인가?
- `WebFilter`에서 요청/응답을 수정하려면 `exchange.mutate()`를 어떻게 사용하는가?
- `WebFilter`의 실행 순서는 어떻게 결정되는가?
- `ExchangeFilterFunction`으로 `WebClient`에 필터를 어떻게 적용하는가?
- 인증, 로깅, CORS를 `WebFilter`로 구현하는 올바른 패턴은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

WebFlux에서 인증, 로깅, 요청 추적, CORS는 모두 WebFilter로 처리합니다. Servlet Filter와 비슷해 보이지만 Reactive 파이프라인 안에서 동작하는 방식이 다릅니다. 잘못 구현하면 비동기 처리 컨텍스트가 누수되거나 필터가 예상과 다른 순서로 실행됩니다. `exchange.mutate()`를 통한 불변 요청 수정 패턴을 이해하면 WebFlux의 Reactive 설계 철학을 몸으로 이해하게 됩니다.

---

## 😱 흔한 실수 (Before — WebFilter를 잘못 사용할 때)

```
실수 1: chain.filter() 호출 없이 응답만 처리

  @Component
  public class AuthFilter implements WebFilter {
      @Override
      public Mono<Void> filter(ServerWebExchange exchange,
                               WebFilterChain chain) {
          String token = exchange.getRequest().getHeaders()
              .getFirst("Authorization");
          if (token == null) {
              exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
              return exchange.getResponse().setComplete();
          }
          // chain.filter() 없음! 다음 필터/핸들러 실행 안 됨
          return Mono.empty();  // 잘못됨
      }
  }
  // 토큰이 있어도 요청이 컨트롤러로 전달 안 됨

실수 2: 응답 헤더를 chain.filter() 이후에 설정 시도

  @Override
  public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
      return chain.filter(exchange)
          .doOnSuccess(v -> {
              // 이미 응답이 전송된 후 → 헤더 설정 무효!
              exchange.getResponse().getHeaders()
                  .add("X-Processed", "true");
          });
  }
  // 응답 헤더는 응답이 커밋되기 전에 설정해야 함

실수 3: 요청 헤더를 직접 수정 시도

  @Override
  public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
      // ServerHttpRequest는 불변!
      exchange.getRequest().getHeaders().add("X-Custom", "value");
      // → UnsupportedOperationException
  }
  // 올바른 방법: exchange.mutate()로 새 exchange 생성
```

---

## ✨ 올바른 접근 (After — WebFilter 올바른 구현)

```
WebFilter 기본 구조:

  @Component
  @Order(1)
  public class LoggingFilter implements WebFilter {

      @Override
      public Mono<Void> filter(ServerWebExchange exchange,
                               WebFilterChain chain) {
          long start = System.currentTimeMillis();
          String path = exchange.getRequest().getURI().getPath();

          return chain.filter(exchange)  // 다음 필터/핸들러 실행
              .doFinally(signal -> {
                  int status = exchange.getResponse()
                      .getStatusCode() != null
                          ? exchange.getResponse().getStatusCode().value()
                          : 0;
                  log.info("{} {} {} {}ms",
                      exchange.getRequest().getMethod(),
                      path, status,
                      System.currentTimeMillis() - start);
              });
      }
  }

핵심 패턴:
  chain.filter() 이전: 요청 처리 (헤더 읽기, 수정)
  chain.filter(modifiedExchange): 수정된 exchange 전달
  chain.filter() 이후 (doOnSuccess/doFinally): 응답 처리
```

---

## 🔬 내부 동작 원리

### 1. WebFilter vs Servlet Filter

```
Servlet Filter:
  void doFilter(request, response, chain) {
      // 요청 처리
      chain.doFilter(request, response);  // 동기 블로킹
      // 응답 처리 (chain 완료 후 동기)
  }
  스레드: 동일 스레드에서 요청~응답 처리

WebFilter:
  Mono<Void> filter(exchange, chain) {
      return chain.filter(exchange)  // 비동기 체인
          .doOnSuccess(v -> {/* 응답 처리 */});
      // 반환: Mono<Void> → 실제 실행은 subscribe 시
  }
  스레드: subscribe 후 EventLoop에서 비동기 실행

Mono<Void> 반환 이유:
  필터가 비동기 작업 포함 가능 (예: DB에서 인증 토큰 검증)
  비동기 작업 = 논블로킹 Mono<> 반환
  → 전체 필터 체인이 Reactive 파이프라인으로 구성

WebFilter 체인 구조:
  [Filter-1] → [Filter-2] → [Filter-3] → DispatcherHandler
  각 필터가 chain.filter(exchange)로 다음을 실행
  → Reactive 스트림 체인으로 연결됨
```

### 2. exchange.mutate() — 불변 요청 수정

```
ServerWebExchange는 불변:
  요청 헤더 추가, 수정 → exchange.mutate() 사용
  mutate()는 새 ServerWebExchange 빌더 반환

요청 헤더 추가:
  @Override
  public Mono<Void> filter(ServerWebExchange exchange,
                           WebFilterChain chain) {
      ServerWebExchange mutated = exchange.mutate()
          .request(request -> request
              .header("X-Request-ID", UUID.randomUUID().toString())
              .header("X-Forwarded-Time",
                  String.valueOf(System.currentTimeMillis()))
          )
          .build();
      return chain.filter(mutated);  // 수정된 exchange 전달
  }

응답 수정 (응답 헤더 데코레이터):
  // 응답이 커밋되기 전에 헤더 추가
  ServerHttpResponse decoratedResponse = new ServerHttpResponseDecorator(
      exchange.getResponse()) {
      @Override
      public Mono<Void> writeWith(Publisher<? extends DataBuffer> body) {
          // 응답 쓰기 전 헤더 추가
          getDelegate().getHeaders().add("X-Custom", "value");
          return super.writeWith(body);
      }
  };

  ServerWebExchange mutated = exchange.mutate()
      .response(decoratedResponse)
      .build();
  return chain.filter(mutated);

Reactor Context 추가:
  return chain.filter(exchange)
      .contextWrite(Context.of("requestId", requestId));
  // exchange mutate 없이 Reactor Context만 추가
```

### 3. WebFilter 순서 — @Order와 Ordered

```
실행 순서:
  낮은 Order = 먼저 실행 (높은 우선순위)
  Order 기본값: Integer.MAX_VALUE (가장 나중)

  @Order(-1)    // 가장 먼저 (Security 등 선행 필터)
  @Order(0)     // 다음
  @Order(100)   // 나중
  @Order(Integer.MAX_VALUE)  // 마지막 (기본)

Spring Security WebFlux:
  SecurityWebFiltersOrder:
    FIRST(-100)
    HTTP_HEADERS_WRITER(-200)  // 보안 헤더
    CORS(-300)                  // CORS
    CSRF(-400)                  // CSRF
    AUTHENTICATION(-500)        // 인증
    AUTHORIZATION(-600)         // 인가
  → 모두 음수 (일반 WebFilter보다 먼저 실행)

일반 WebFilter 권장 Order:
  로깅: @Order(1)     (초기, 모든 요청 기록)
  추적 ID: @Order(2)  (로깅 이전에 ID 생성)
  캐싱: @Order(10)    (인증 후 캐시 확인)

Ordered 인터페이스:
  @Component
  public class MyFilter implements WebFilter, Ordered {
      @Override
      public int getOrder() { return 1; }
      @Override
      public Mono<Void> filter(...) { ... }
  }
```

### 4. 인증 WebFilter 구현

```
JWT 인증 WebFilter:

  @Component
  @Order(-100)
  public class JwtAuthenticationFilter implements WebFilter {

      private final JwtService jwtService;

      private static final List<String> EXCLUDED_PATHS = List.of(
          "/api/auth/login", "/api/auth/register", "/actuator/health"
      );

      @Override
      public Mono<Void> filter(ServerWebExchange exchange,
                               WebFilterChain chain) {
          String path = exchange.getRequest().getPath().value();
          
          // 인증 제외 경로
          if (EXCLUDED_PATHS.stream().anyMatch(path::startsWith)) {
              return chain.filter(exchange);
          }

          String authHeader = exchange.getRequest().getHeaders()
              .getFirst(HttpHeaders.AUTHORIZATION);

          if (authHeader == null || !authHeader.startsWith("Bearer ")) {
              exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
              return exchange.getResponse().setComplete();
          }

          String token = authHeader.substring(7);

          return jwtService.validateToken(token)  // Mono<Authentication>
              .flatMap(authentication -> {
                  // SecurityContext에 인증 정보 저장
                  SecurityContext ctx = new SecurityContextImpl(authentication);
                  return chain.filter(exchange)
                      .contextWrite(ReactiveSecurityContextHolder
                          .withSecurityContext(Mono.just(ctx)));
              })
              .onErrorResume(JwtException.class, e -> {
                  exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                  return exchange.getResponse().setComplete();
              });
      }
  }
```

### 5. ExchangeFilterFunction — WebClient 필터

```
ExchangeFilterFunction:
  WebClient의 각 요청/응답에 적용되는 필터
  WebFilter의 WebClient 버전

  @Bean
  public WebClient webClient(WebClient.Builder builder) {
      return builder
          .baseUrl("https://api.example.com")
          .filter(loggingFilter())
          .filter(retryFilter())
          .filter(authFilter())
          .build();
  }

  // 요청/응답 로깅
  private ExchangeFilterFunction loggingFilter() {
      return ExchangeFilterFunction.ofRequestProcessor(request -> {
          log.debug("요청: {} {}", request.method(), request.url());
          return Mono.just(request);
      });
      // 응답 처리:
      // ExchangeFilterFunction.ofResponseProcessor(response -> ...)
      // 또는 양쪽 모두:
      // (request, next) -> next.exchange(request).doOnNext(response -> ...)
  }

  // Reactor Context에서 인증 토큰 자동 주입
  private ExchangeFilterFunction authFilter() {
      return (request, next) ->
          Mono.deferContextual(ctx -> {
              String token = ctx.getOrDefault("accessToken", "");
              ClientRequest modified = ClientRequest.from(request)
                  .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
                  .build();
              return next.exchange(modified);
          });
  }

  // 공통 에러 처리 필터
  private ExchangeFilterFunction retryFilter() {
      return (request, next) ->
          next.exchange(request)
              .retryWhen(Retry.backoff(2, Duration.ofMillis(300))
                  .filter(e -> e instanceof ConnectException)
              );
  }
```

---

## 💻 실전 코드

### 실험 1: 요청 추적 ID 필터

```java
@Component
@Order(-50)
public class RequestTraceFilter implements WebFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange,
                             WebFilterChain chain) {
        String requestId = Optional
            .ofNullable(exchange.getRequest().getHeaders()
                .getFirst("X-Request-ID"))
            .orElse(UUID.randomUUID().toString());

        // 요청에 추적 ID 추가 (exchange mutate)
        ServerWebExchange mutated = exchange.mutate()
            .request(r -> r.header("X-Request-ID", requestId))
            .build();

        // 응답에도 추적 ID 추가
        mutated.getResponse().getHeaders().add("X-Request-ID", requestId);

        return chain.filter(mutated)
            .contextWrite(Context.of("requestId", requestId));
    }
}
```

### 실험 2: 응답 캐싱 WebFilter

```java
@Component
@Order(10)
public class ResponseCacheFilter implements WebFilter {

    private final Cache<String, byte[]> cache = Caffeine.newBuilder()
        .expireAfterWrite(Duration.ofMinutes(5))
        .maximumSize(1000)
        .build();

    @Override
    public Mono<Void> filter(ServerWebExchange exchange,
                             WebFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();

        // GET 요청만 캐싱
        if (!HttpMethod.GET.equals(request.getMethod())) {
            return chain.filter(exchange);
        }

        String cacheKey = request.getURI().toString();
        byte[] cached = cache.getIfPresent(cacheKey);

        if (cached != null) {
            exchange.getResponse().setStatusCode(HttpStatus.OK);
            exchange.getResponse().getHeaders()
                .add("X-Cache", "HIT");
            DataBuffer buffer = exchange.getResponse()
                .bufferFactory().wrap(cached);
            return exchange.getResponse().writeWith(Mono.just(buffer));
        }

        exchange.getResponse().getHeaders().add("X-Cache", "MISS");
        return chain.filter(exchange);
    }
}
```

### 실험 3: WebClient ExchangeFilterFunction 조합

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient(WebClient.Builder builder) {
        return builder
            .baseUrl("https://api.example.com")
            .filter(timingFilter())
            .filter(contextualAuthFilter())
            .filter(errorMappingFilter())
            .build();
    }

    // 요청 처리 시간 측정
    private ExchangeFilterFunction timingFilter() {
        return (request, next) -> {
            long start = System.nanoTime();
            return next.exchange(request)
                .doOnSuccess(response ->
                    log.debug("요청 완료: {} {}ms",
                        request.url(),
                        (System.nanoTime() - start) / 1_000_000));
        };
    }

    // Reactor Context에서 인증 토큰 주입
    private ExchangeFilterFunction contextualAuthFilter() {
        return (request, next) ->
            Mono.deferContextual(ctx -> {
                if (ctx.hasKey("accessToken")) {
                    ClientRequest modified = ClientRequest.from(request)
                        .header(HttpHeaders.AUTHORIZATION,
                            "Bearer " + ctx.get("accessToken"))
                        .build();
                    return next.exchange(modified);
                }
                return next.exchange(request);
            });
    }

    // 에러 응답 → 커스텀 예외 변환
    private ExchangeFilterFunction errorMappingFilter() {
        return ExchangeFilterFunction.ofResponseProcessor(response -> {
            if (response.statusCode().equals(HttpStatus.UNAUTHORIZED)) {
                return response.createException()
                    .flatMap(e -> Mono.error(new UnauthorizedException()));
            }
            return Mono.just(response);
        });
    }
}
```

---

## 📊 성능 비교

```
WebFilter 체인 오버헤드:

필터 1개당 오버헤드:
  Mono<Void> 래핑 + chain.filter() 호출: ~수μs
  실제 I/O 없는 순수 필터: < 1ms
  → 필터 10개라도 < 10ms (I/O 지배적 요청에서 무시 가능)

contextWrite 오버헤드:
  불변 Context 생성: ~수십 ns
  → 무시 가능

exchange.mutate() 오버헤드:
  ServerWebExchange 새 인스턴스 생성: ~수백 ns
  → 무시 가능

총 WebFilter 체인 오버헤드:
  5개 필터, 각 1ms: 5ms
  vs 전체 요청 처리 (DB 100ms): 5% 오버헤드
  → 실무에서 의미 없는 수준

Spring Security WebFilter 오버헤드:
  JWT 파싱 + 검증: ~1~5ms (CPU 집약)
  DB 기반 인증: ~10~50ms (I/O)
  → 보안 필터가 실질적 오버헤드
```

---

## ⚖️ 트레이드오프

```
WebFilter vs @ControllerAdvice:
  WebFilter: 모든 요청 (정적 리소스 포함), 낮은 레벨
  @ControllerAdvice: 컨트롤러 예외만, 높은 레벨
  → 전역 에러: WebExceptionHandler (낮은 레벨)
  → 비즈니스 에러: @ControllerAdvice (컨트롤러 수준)

exchange.mutate() vs 직접 수정:
  mutate(): 불변성 유지, Thread-safe, WebFlux 권장 방식
  직접 수정: 불가 (UnsupportedOperationException) 또는 위험
  → 항상 mutate() 사용

WebFilter 순서 관리:
  @Order 숫자가 많아지면 관리 어려움
  → Enum으로 Order 상수 관리 (Spring Security FilterOrder 방식)
  → 팀 내 Filter Order 문서화 필수

ExchangeFilterFunction 재사용:
  WebClient마다 동일 필터 중복 → @Bean 분리 후 공유
  필터 조합은 WebClient.Builder.filter()로 순서 있게 추가
```

---

## 📌 핵심 정리

```
WebFilter 핵심:

Servlet Filter vs WebFilter:
  Servlet: 동기, doFilter()
  WebFlux: Reactive, Mono<Void> filter(), chain.filter()로 다음 실행

불변 요청 수정:
  exchange.mutate().request(r -> r.header(...)).build()
  → 새 ServerWebExchange 반환 (원본 불변)
  → chain.filter(mutated)으로 수정된 exchange 전달

실행 순서:
  @Order 낮을수록 먼저 실행
  Spring Security: 음수 Order (일반 Filter보다 먼저)

인증 패턴:
  체인 전: 토큰 검증 → 실패 시 즉시 응답
  체인 후: 불필요 (인증은 사전 처리)
  contextWrite: Reactor Context에 인증 정보 전달

ExchangeFilterFunction:
  WebClient에 적용하는 필터
  ofRequestProcessor / ofResponseProcessor / (req, next) -> ...
```

---

## 🤔 생각해볼 문제

**Q1.** `WebFilter`에서 요청 본문을 읽으면 어떤 문제가 생기나요?

<details>
<summary>해설 보기</summary>

`ServerHttpRequest.getBody()`로 요청 본문을 읽으면 **본문 스트림이 소비**됩니다. 이후 컨트롤러에서 `@RequestBody`로 읽으려 하면 이미 소비된 스트림이라 빈 결과가 됩니다.

해결책은 본문을 캐싱하는 데코레이터를 사용합니다:

```java
@Override
public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
    return DataBufferUtils.join(exchange.getRequest().getBody())
        .flatMap(buffer -> {
            byte[] bytes = new byte[buffer.readableByteCount()];
            buffer.read(bytes);
            DataBufferUtils.release(buffer);

            // 캐싱된 본문으로 새 요청 생성
            ServerHttpRequest mutated = new ServerHttpRequestDecorator(
                exchange.getRequest()) {
                @Override
                public Flux<DataBuffer> getBody() {
                    return Flux.just(
                        exchange.getResponse().bufferFactory().wrap(bytes)
                    );
                }
            };

            log.info("요청 본문: {}", new String(bytes));  // 로깅 등 처리

            return chain.filter(exchange.mutate()
                .request(mutated).build());
        });
}
```

단, 이 방식은 전체 본문을 메모리에 올리므로 대용량 요청에는 부적합합니다.

</details>

---

**Q2.** `WebFilter`와 Spring Security의 `SecurityWebFilterChain`은 어떻게 공존하나요?

<details>
<summary>해설 보기</summary>

Spring Security WebFlux는 `SecurityWebFilterChain`을 통해 보안 필터를 등록합니다. 이는 `WebFilter` 체인의 일부로 동작합니다.

`SecurityWebFilterChain`은 내부적으로 여러 보안 `WebFilter`를 특정 URL 패턴에 조건부로 적용합니다:

```java
@Bean
public SecurityWebFilterChain securityChain(ServerHttpSecurity http) {
    return http
        .csrf(ServerHttpSecurity.CsrfSpec::disable)
        .authorizeExchange(auth -> auth
            .pathMatchers("/api/public/**").permitAll()
            .anyExchange().authenticated()
        )
        .addFilterBefore(myFilter, SecurityWebFiltersOrder.AUTHENTICATION)
        .build();
}
```

일반 `@Component WebFilter`와 `SecurityWebFilterChain` 내 필터는 모두 같은 WebFilter 체인에 있지만, `@Order`로 실행 순서가 결정됩니다. Spring Security의 기본 Order는 `-100`으로, 대부분의 일반 WebFilter보다 먼저 실행됩니다.

</details>

---

**Q3.** `chain.filter(exchange)` 이후에 응답 상태 코드를 변경할 수 있나요?

<details>
<summary>해설 보기</summary>

응답 상태 코드는 **응답이 커밋되기 전**에만 변경 가능합니다. `chain.filter(exchange)` 이후 `doOnSuccess()`에서는 일반적으로 이미 응답이 커밋된 후라 변경이 무효입니다.

응답을 수정하려면 `ServerHttpResponseDecorator`를 사용합니다:

```java
@Override
public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
    ServerHttpResponse originalResponse = exchange.getResponse();

    ServerHttpResponseDecorator decoratedResponse =
        new ServerHttpResponseDecorator(originalResponse) {
            @Override
            public Mono<Void> writeWith(Publisher<? extends DataBuffer> body) {
                // 응답 쓰기 직전에 상태 코드/헤더 수정 가능
                if (getStatusCode() == HttpStatus.OK) {
                    // 특정 조건에서 상태 코드 변경
                }
                return super.writeWith(body);
            }
        };

    return chain.filter(exchange.mutate()
        .response(decoratedResponse).build());
}
```

이 방식은 `writeWith()`가 실제 응답을 쓰기 직전에 호출되므로, 응답 커밋 전 마지막 기회에 수정할 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: 요청/응답 처리](./06-request-response-handling.md)** | **[홈으로 🏠](../README.md)**

</div>
