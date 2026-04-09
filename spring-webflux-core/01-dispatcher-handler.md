# WebFlux 아키텍처 — DispatcherHandler와 Reactive 처리 파이프라인

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `DispatcherHandler`는 `DispatcherServlet`과 무엇이 다른가?
- `HandlerMapping`, `HandlerAdapter`, `HandlerResultHandler`의 Reactive 버전은 어떻게 동작하는가?
- `ServerWebExchange`는 왜 불변(immutable) 설계인가?
- WebFlux 요청이 컨트롤러 메서드까지 도달하는 전체 흐름은 무엇인가?
- `@EnableWebFlux`는 언제 필요하고 언제 불필요한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

WebFlux를 MVC처럼 사용하면 동작은 하지만 내부 원리를 모르면 커스터마이징이 어렵습니다. 글로벌 에러 처리, 커스텀 HandlerMapping, 응답 직렬화 설정을 올바르게 하려면 DispatcherHandler가 어떻게 요청을 위임하는지 알아야 합니다. MVC에서 마이그레이션할 때 "왜 MVC에서 되던 게 WebFlux에서 안 되지?"의 원인도 여기서 찾을 수 있습니다.

---

## 😱 흔한 실수 (Before — DispatcherHandler를 MVC와 동일시할 때)

```
실수 1: WebFlux에서 HttpServletRequest 접근 시도

  @GetMapping("/data")
  public Mono<String> getData(HttpServletRequest request) {
      // HttpServletRequest: Servlet API, WebFlux에서 존재하지 않음
      // → 파라미터 바인딩 실패 → 500 에러
  }
  
  올바른 방식:
  public Mono<String> getData(ServerWebExchange exchange) {
      ServerHttpRequest request = exchange.getRequest();
      ...
  }

실수 2: ModelAndView를 WebFlux 컨트롤러에서 반환

  @GetMapping("/view")
  public Mono<ModelAndView> getView() {
      return Mono.just(new ModelAndView("home"));
      // WebFlux는 Thymeleaf Reactive를 사용하지만
      // 설정 없이는 HandlerResultHandler가 처리 못함
  }

실수 3: @EnableWebFlux와 WebFluxAutoConfiguration 충돌

  @SpringBootApplication
  @EnableWebFlux  // Spring Boot + WebFlux에서는 보통 불필요
  public class MyApp {
      // @EnableWebFlux가 WebFluxAutoConfiguration을 무력화
      // → ObjectMapper 커스터마이징 등 자동 설정이 사라짐
  }
```

---

## ✨ 올바른 접근 (After — WebFlux 아키텍처 이해)

```
WebFlux = Reactive HTTP 처리 파이프라인

  클라이언트 요청
    ↓
  Netty (Chapter 3에서 다룸)
    ↓
  HttpHandler (최상위 Reactive HTTP 추상화)
    ↓
  WebHttpHandlerBuilder (WebFilter 체인 구성)
    ↓
  DispatcherHandler (핸들러 위임)
    ↓
  HandlerMapping → HandlerAdapter → 핸들러 실행
    ↓
  HandlerResultHandler (결과 → HTTP 응답)

MVC vs WebFlux 비교:
  MVC:    DispatcherServlet (Thread-per-Request, 블로킹)
  WebFlux: DispatcherHandler (Reactive, 논블로킹)
  
핵심 차이:
  MVC: 하나의 스레드가 요청 전체를 처리
  WebFlux: Mono<Void>를 반환 → 구독 시 비동기 처리
```

---

## 🔬 내부 동작 원리

### 1. HttpHandler — 최상위 Reactive HTTP 추상화

```
WebFlux의 최상위 인터페이스:

  public interface HttpHandler {
      Mono<Void> handle(ServerHttpRequest request,
                        ServerHttpResponse response);
  }

  이 인터페이스 하나가 Netty, Undertow, Jetty 등
  모든 서버를 동일하게 추상화

  Netty 연결:
    ReactorHttpHandlerAdapter (ChannelHandler)
    → Netty 요청/응답을 ServerHttpRequest/Response로 변환
    → HttpHandler.handle() 호출
    → Mono<Void> 구독 → 완료 시 응답 flush

Spring WebFlux가 HttpHandler를 만드는 과정:
  WebHttpHandlerBuilder.applicationContext(context)
    → WebFilter들로 필터 체인 구성
    → ExceptionHandlingWebHandler 래핑
    → DispatcherHandler를 코어 핸들러로 구성
    → 최종 HttpHandler 반환
```

### 2. ServerWebExchange — 불변 요청/응답 컨테이너

```
ServerWebExchange:
  HTTP 요청/응답 쌍을 담는 컨테이너
  WebFilter와 핸들러 간에 공유되는 Context

주요 메서드:
  exchange.getRequest():     ServerHttpRequest (읽기 전용)
  exchange.getResponse():    ServerHttpResponse (쓰기)
  exchange.getSession():     Mono<WebSession>
  exchange.getPrincipal():   Mono<Principal>
  exchange.getAttributes():  Map<String, Object> (Mutable)
  exchange.mutate():         수정된 exchange 빌더 반환

불변 설계 이유:
  Reactive 파이프라인에서 여러 연산자가 exchange를 공유
  mutate() → 새 ServerWebExchange 반환 (기존 유지)
  → 스레드 안전 (공유 상태 변경 없음)
  → WebFilter에서 헤더 추가 시 exchange.mutate() 사용

실무 예시:
  // WebFilter에서 요청 헤더 추가
  ServerWebExchange mutated = exchange.mutate()
      .request(r -> r.header("X-Request-ID", requestId))
      .build();
  return chain.filter(mutated);  // 수정된 exchange 전달
```

### 3. DispatcherHandler — 핵심 위임자

```
DispatcherHandler 처리 흐름:

  public Mono<Void> handle(ServerWebExchange exchange) {
      // 1. 모든 HandlerMapping에 핸들러 조회
      return Flux.fromIterable(this.handlerMappings)
          .concatMap(mapping -> mapping.getHandler(exchange))
          .next()  // 첫 번째 매칭 핸들러
          .switchIfEmpty(createNotFoundError())

          // 2. 핸들러에 맞는 HandlerAdapter로 실행
          .flatMap(handler -> {
              HandlerAdapter adapter = getHandlerAdapter(handler);
              return adapter.handle(exchange, handler);
          })

          // 3. 결과를 적절한 HandlerResultHandler로 응답 변환
          .flatMap(result -> {
              HandlerResultHandler handler = getResultHandler(result);
              return handler.handleResult(exchange, result);
          });
  }

MVC DispatcherServlet과의 차이:
  MVC:    모든 단계가 동기 (블로킹)
  WebFlux: 각 단계가 Mono/Flux 반환 → 논블로킹 체인
```

### 4. HandlerMapping — 핸들러 찾기

```
주요 HandlerMapping 구현체:

RequestMappingHandlerMapping:
  @RequestMapping, @GetMapping 등 어노테이션 파싱
  URI 패턴 + HTTP 메서드 + 조건으로 핸들러 메서드 찾기
  MVC와 유사하지만 Reactive 반환

RouterFunctionMapping:
  RouterFunction으로 등록된 엔드포인트 처리
  함수형 엔드포인트 (Chapter 4-02에서 상세)

SimpleUrlHandlerMapping:
  URL 패턴 → WebSocketHandler 매핑
  WebSocket 엔드포인트 등록에 주로 사용

HandlerMapping 우선순위:
  @Order 또는 Ordered 인터페이스로 결정
  RouterFunctionMapping: Order = -1 (기본)
  RequestMappingHandlerMapping: Order = 0 (기본)
  → RouterFunction이 @Controller보다 먼저 매칭
```

### 5. HandlerAdapter — 핸들러 실행

```
주요 HandlerAdapter 구현체:

RequestMappingHandlerAdapter:
  @RequestMapping 메서드 실행
  파라미터 바인딩:
    @PathVariable → URI 변수 추출
    @RequestBody → HttpMessageReader로 본문 역직렬화
    ServerWebExchange → 직접 주입
  반환값 처리:
    Mono<T> → 그대로 반환
    T (비 Reactive) → Mono.just(T)로 래핑

WebFluxResponseBodyResultHandler:
  Reactive 컨트롤러 반환값을 HTTP 응답으로 직렬화
  HttpMessageWriter로 JSON 등 직렬화

주요 HandlerResultHandler:
  ResponseEntityResultHandler:   ResponseEntity<T> 처리
  ResponseBodyResultHandler:     @ResponseBody 결과 처리
  ViewResolutionResultHandler:   View 이름 → Reactive View 렌더링
  ServerResponseResultHandler:   RouterFunction의 ServerResponse 처리
```

### 6. @EnableWebFlux와 Spring Boot 자동 설정

```
@EnableWebFlux:
  WebFlux 설정을 수동으로 활성화
  WebFluxConfigurationSupport를 임포트
  직접 WebFluxConfigurer를 구현할 때 사용

Spring Boot + spring-boot-starter-webflux:
  WebFluxAutoConfiguration이 자동으로 WebFlux 설정
  → @EnableWebFlux 불필요 (겹치면 자동 설정 무력화)

@EnableWebFlux가 필요한 경우:
  Spring Boot 없이 순수 Spring WebFlux 사용 시
  WebFluxAutoConfiguration을 완전히 대체하는 커스텀 설정 시

커스터마이징 올바른 방법 (Spring Boot):
  @Configuration
  public class WebFluxConfig implements WebFluxConfigurer {
      @Override
      public void configureHttpMessageCodecs(
              ServerCodecConfigurer configurer) {
          configurer.defaultCodecs()
              .maxInMemorySize(2 * 1024 * 1024);  // 2MB
      }
  }
  // @EnableWebFlux 없이 WebFluxConfigurer 구현
  // WebFluxAutoConfiguration에 훅으로 동작
```

---

## 💻 실전 코드

### 실험 1: ServerWebExchange로 요청 정보 접근

```java
@RestController
public class ExchangeController {

    @GetMapping("/exchange-info")
    public Mono<Map<String, Object>> exchangeInfo(ServerWebExchange exchange) {
        ServerHttpRequest request = exchange.getRequest();
        return exchange.getSession()
            .map(session -> Map.of(
                "method",    request.getMethod().name(),
                "uri",       request.getURI().toString(),
                "headers",   request.getHeaders().toSingleValueMap(),
                "sessionId", session.getId(),
                "attributes", exchange.getAttributes()
            ));
    }

    // exchange.mutate()로 요청 수정 후 전달 (필터 패턴)
    @GetMapping("/mutate-demo")
    public Mono<String> mutateDemo(ServerWebExchange exchange) {
        // 속성 추가 (mutate 불필요, getAttributes()는 Mutable)
        exchange.getAttributes().put("processedAt", System.currentTimeMillis());
        return Mono.just("OK");
    }
}
```

### 실험 2: 글로벌 에러 처리

```java
@Configuration
public class GlobalErrorConfig {

    // DispatcherHandler 이전 레벨의 에러 처리
    @Bean
    @Order(-1)  // WebExceptionHandler 중 최우선
    public WebExceptionHandler globalErrorHandler(
            ObjectProvider<ErrorAttributes> errorAttributes,
            ApplicationContext context) {
        return (exchange, ex) -> {
            log.error("처리되지 않은 에러: {}", ex.getMessage(), ex);

            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            exchange.getResponse().getHeaders()
                .setContentType(MediaType.APPLICATION_JSON);

            byte[] bytes = """
                {"error": "Internal Server Error", "message": "%s"}
                """.formatted(ex.getMessage()).getBytes();

            DataBuffer buffer = exchange.getResponse()
                .bufferFactory().wrap(bytes);
            return exchange.getResponse().writeWith(Mono.just(buffer));
        };
    }
}

// @ControllerAdvice도 동작 (DispatcherHandler 레벨)
@ControllerAdvice
public class ReactiveExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public Mono<ResponseEntity<ErrorResponse>> handleUserNotFound(
            UserNotFoundException ex) {
        return Mono.just(ResponseEntity.status(404)
            .body(new ErrorResponse(ex.getMessage())));
    }
}
```

### 실험 3: 커스텀 HandlerMethodArgumentResolver

```java
// 커스텀 파라미터 바인딩
@Configuration
public class WebFluxConfig implements WebFluxConfigurer {

    @Override
    public void configureArgumentResolvers(
            ArgumentResolverConfigurer configurer) {
        configurer.addCustomResolver(new CurrentUserArgumentResolver());
    }
}

// @CurrentUser → Reactor Context에서 사용자 추출
public class CurrentUserArgumentResolver
        implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(CurrentUser.class);
    }

    @Override
    public Mono<Object> resolveArgument(MethodParameter parameter,
            BindingContext bindingContext, ServerWebExchange exchange) {
        return ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication().getPrincipal())
            .cast(Object.class);
    }
}
```

---

## 📊 성능 비교

```
DispatcherServlet vs DispatcherHandler 처리 흐름 비교:

DispatcherServlet (MVC):
  Thread-1: HTTP 요청 수신
  Thread-1: HandlerMapping 조회 (동기)
  Thread-1: HandlerAdapter 실행 (동기, 블로킹 I/O 포함)
  Thread-1: ViewResolver 처리 (동기)
  Thread-1: 응답 완료
  → 1 요청 = 1 스레드 전체 점유

DispatcherHandler (WebFlux):
  EventLoop: HTTP 요청 수신 → Mono<Void> 파이프라인 구성
  EventLoop: 구독 시작 (subscribe)
  (I/O 대기 중 EventLoop는 다른 요청 처리)
  EventLoop: 핸들러 결과 수신 → 응답 직렬화
  → 1 EventLoop가 여러 요청의 파이프라인 관리

처리량 비교 (I/O 집약, 100ms 외부 API 호출):
  MVC (200 스레드): ~2,000 req/s
  WebFlux (16 EventLoop + 논블로킹): ~10,000+ req/s
  → 논블로킹 I/O가 실제 병목인 경우의 이론적 차이
```

---

## ⚖️ 트레이드오프

```
WebFlux 아키텍처 트레이드오프:

장점:
  ① 동일 하드웨어로 높은 동시 처리 (논블로킹 I/O 시)
  ② 메모리 효율 (스레드 수 감소)
  ③ 서버 추상화 (Netty, Undertow 등 교체 가능)

단점:
  ① 블로킹 코드 혼용 시 오히려 성능 저하
  ② 디버깅 복잡 (스택 트레이스가 비동기로 분산)
  ③ MVC 일부 기능 미지원 (일부 Filter, Interceptor)

ServerWebExchange 불변 설계:
  장점: Thread-safe, 부수 효과 없는 파이프라인
  단점: 요청 수정 시 mutate() 필요 (코드 장황)

@EnableWebFlux 주의:
  Spring Boot에서 사용 시 자동 설정 무력화
  → ObjectMapper 커스터마이징, Actuator 통합 등 사라짐
  → WebFluxConfigurer 구현으로 대체
```

---

## 📌 핵심 정리

```
DispatcherHandler 핵심:

MVC vs WebFlux:
  DispatcherServlet: 동기, Thread-per-Request
  DispatcherHandler: Reactive, Mono<Void> 기반

처리 파이프라인:
  HandlerMapping → 핸들러 찾기
  HandlerAdapter → 핸들러 실행 (Mono<HandlerResult>)
  HandlerResultHandler → 응답 변환

ServerWebExchange:
  요청/응답 컨테이너 (불변)
  mutate() → 새 exchange 생성 (헤더 추가 등)
  getAttributes() → Mutable (요청 범위 데이터 저장)

설정:
  Spring Boot: WebFluxConfigurer 구현 (자동 설정 유지)
  비 Spring Boot: @EnableWebFlux 필요
```

---

## 🤔 생각해볼 문제

**Q1.** `DispatcherHandler`에서 어떤 `HandlerMapping`도 핸들러를 찾지 못하면 어떻게 되나요?

<details>
<summary>해설 보기</summary>

`DispatcherHandler.handle()`에서 `.next()`가 `empty`를 반환하면 `.switchIfEmpty(createNotFoundError())`가 실행됩니다.

기본 동작은 `ResponseStatusException(HttpStatus.NOT_FOUND)`를 에러 신호로 방출합니다. 이 에러는 `WebExceptionHandler` 체인으로 전파되어 최종적으로 HTTP 404 응답이 됩니다.

```java
// 커스텀 404 처리
@Configuration
public class NotFoundConfig {
    @Bean
    @Order(-2)  // 기본 핸들러보다 우선
    public WebExceptionHandler notFoundHandler() {
        return (exchange, ex) -> {
            if (ex instanceof ResponseStatusException rse
                    && rse.getStatusCode() == HttpStatus.NOT_FOUND) {
                exchange.getResponse().setStatusCode(HttpStatus.NOT_FOUND);
                // 커스텀 404 응답
                return exchange.getResponse().setComplete();
            }
            return Mono.error(ex);  // 다른 에러는 전파
        };
    }
}
```

</details>

---

**Q2.** WebFlux에서 `HandlerInterceptor` 대신 무엇을 사용하나요?

<details>
<summary>해설 보기</summary>

WebFlux는 Servlet `HandlerInterceptor`를 지원하지 않습니다. 대신 두 가지 방법을 사용합니다.

**WebFilter** (모든 요청에 적용):
```java
@Component
public class LoggingFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        long start = System.currentTimeMillis();
        return chain.filter(exchange)
            .doFinally(signal ->
                log.info("{} {} {}ms",
                    exchange.getRequest().getMethod(),
                    exchange.getRequest().getURI(),
                    System.currentTimeMillis() - start)
            );
    }
}
```

**WebFluxConfigurer.addArgumentResolvers()**: 파라미터 바인딩 커스터마이징.

`HandlerInterceptor`의 `preHandle`/`postHandle`/`afterCompletion`을 Reactive로 대체하려면 `WebFilter`의 `chain.filter(exchange)` 앞/뒤에 로직을 배치합니다. `doOnSuccess()`, `doFinally()`가 `postHandle`, `afterCompletion`에 해당합니다.

</details>

---

**Q3.** `WebClient`로 WebFlux 앱 내부의 다른 엔드포인트를 호출할 때 `DispatcherHandler`를 거치지 않고 직접 호출하는 방법이 있나요?

<details>
<summary>해설 보기</summary>

테스트 목적이라면 `WebTestClient.bindToApplicationContext(context)`를 사용합니다. 실제 HTTP 없이 `DispatcherHandler`를 직접 호출합니다.

```java
// 테스트에서 HTTP 없이 DispatcherHandler 직접 호출
WebTestClient client = WebTestClient
    .bindToApplicationContext(applicationContext)
    .build();

client.get().uri("/users/1")
    .exchange()
    .expectStatus().isOk();
```

프로덕션에서 같은 JVM 내 서비스 간 호출은 WebClient 대신 **직접 서비스 빈을 주입**하는 것이 좋습니다. 네트워크 오버헤드와 직렬화/역직렬화 비용을 없앨 수 있습니다.

```java
// 나쁜 예: 같은 JVM 내에서 WebClient로 자기 자신 호출
webClient.get().uri("http://localhost:8080/users/1")...

// 좋은 예: 직접 서비스 주입
userService.findById(1L)...
```

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 어노테이션 vs 함수형 엔드포인트 ➡️](./02-annotation-vs-functional.md)**

</div>
