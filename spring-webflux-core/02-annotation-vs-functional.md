# 어노테이션 기반 vs 함수형 엔드포인트 — 언제 무엇을 선택하는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@RestController` + `@GetMapping`과 `RouterFunction` + `HandlerFunction`은 내부적으로 어떻게 다르게 처리되는가?
- 어노테이션 방식의 리플렉션 오버헤드는 실제 성능에 얼마나 영향을 미치는가?
- `RouterFunction`에서 요청 조건을 조합하는 방법은 무엇인가?
- 두 방식을 같은 애플리케이션에서 혼용할 때 주의할 점은 무엇인가?
- Spring Native(GraalVM)에서 두 방식의 차이는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"WebFlux는 함수형 방식이 성능이 좋다"는 말을 들어도 실제로 차이가 얼마나 나는지, 어떤 상황에서 전환해야 하는지 판단하기 어렵습니다. 또한 GraalVM Native Image로 컴파일하는 경우 리플렉션 기반 어노테이션 처리가 제약이 됩니다. 두 방식의 차이를 명확히 알면 팀 코드 스타일 선택과 아키텍처 결정을 근거 있게 할 수 있습니다.

---

## 😱 흔한 실수 (Before — 두 방식을 혼동할 때)

```
실수 1: RouterFunction에서 @RequestBody를 기대

  RouterFunction<ServerResponse> route = RouterFunctions.route()
      .GET("/users/{id}", request -> {
          Long id = Long.parseLong(request.pathVariable("id"));
          return userService.findById(id)
              .flatMap(user -> ServerResponse.ok().bodyValue(user));
      })
      .POST("/users", request -> {
          // @RequestBody 사용 불가 — HandlerFunction 파라미터는 ServerRequest뿐
          CreateUserRequest body = request.bodyToMono(CreateUserRequest.class);
          // Mono<CreateUserRequest> → .flatMap으로 처리해야 함
      })
      .build();

실수 2: 함수형과 어노테이션 간 @ExceptionHandler 공유 기대

  // @ControllerAdvice는 어노테이션 방식에만 적용
  // RouterFunction에서 발생한 에러는 @ExceptionHandler로 처리 안 됨
  // → RouterFunction 내부에서 onErrorResume으로 처리 필요

실수 3: RouterFunctionMapping과 RequestMappingHandlerMapping 우선순위 혼동

  // RouterFunction: Order -1 (기본, @Controller보다 우선)
  // 같은 경로를 두 방식 모두 등록 시 RouterFunction이 먼저 매칭
  // → 어노테이션 방식 핸들러가 실행 안 됨
```

---

## ✨ 올바른 접근 (After — 각 방식의 특성 활용)

```
어노테이션 방식 (@RestController):
  - 팀에서 MVC 경험자가 많을 때 (학습 곡선 낮음)
  - CRUD 위주의 REST API
  - Spring Security, Validation, AOP 통합이 중요할 때

함수형 방식 (RouterFunction):
  - 라우팅 로직을 코드로 명확히 표현하고 싶을 때
  - 미들웨어 스타일 처리 (여러 핸들러 조합)
  - GraalVM Native Image 대상 (리플렉션 최소화)
  - 조건부 라우팅이 복잡할 때

혼용:
  - 기존 코드베이스에 점진적 함수형 도입
  - 도메인별로 방식 분리 (인증: 함수형, 비즈니스: 어노테이션)
  - @Order로 RouterFunctionMapping 순서 조정
```

---

## 🔬 내부 동작 원리

### 1. 어노테이션 방식 처리 경로

```
어노테이션 방식 내부 처리:

  @GetMapping("/users/{id}")
  public Mono<User> getUser(@PathVariable Long id) { ... }

DispatcherHandler → RequestMappingHandlerMapping:
  1. 애플리케이션 시작 시:
     모든 @RestController 스캔
     @RequestMapping 어노테이션을 파싱하여 RequestMappingInfo 생성
     Method → RequestMappingInfo 맵핑 저장 (Registry)

  2. 요청 시:
     URI, HTTP 메서드, 헤더 조건으로 Registry 조회 (O(1) 해시맵)
     매칭 HandlerMethod 반환

RequestMappingHandlerAdapter:
  1. 파라미터 리졸버로 각 파라미터 바인딩 (리플렉션)
     @PathVariable → URI에서 추출 + 타입 변환
     @RequestBody → HttpMessageReader로 역직렬화
     ServerWebExchange → 직접 주입
  2. 리플렉션으로 메서드 실행 (method.invoke())
  3. 반환값 → HandlerResult로 래핑

오버헤드:
  시작 시 오버헤드: 스캔 + 파싱 (한 번만)
  요청 시 오버헤드: 리플렉션 파라미터 바인딩 (~수μs)
  → 실무에서 무시 가능 (I/O 시간이 수십~수백ms 지배적)
```

### 2. 함수형 방식 처리 경로

```
함수형 방식 내부 처리:

  RouterFunction<ServerResponse> route = route()
      .GET("/users/{id}", handler::getUser)
      .build();

  @Component
  public class UserHandler {
      public Mono<ServerResponse> getUser(ServerRequest request) { ... }
  }

DispatcherHandler → RouterFunctionMapping:
  1. 애플리케이션 시작 시:
     RouterFunction 빈 수집
     함수형 라우팅 규칙 구성 (코드가 곧 라우팅 규칙)

  2. 요청 시:
     RouterFunction.route(request) 호출 → Optional<HandlerFunction>
     리플렉션 없음 → 람다/메서드 참조로 직접 호출

ServerResponseResultHandler:
  ServerResponse (ServerResponse.ok().bodyValue(...))
  → HTTP 응답 변환

차이점:
  어노테이션: 리플렉션으로 파라미터 바인딩
  함수형: ServerRequest에서 직접 꺼냄 (타입 안전, 컴파일 타임)
  
  어노테이션: 프레임워크가 파라미터 처리
  함수형: 개발자가 명시적으로 처리 (더 많은 코드, 더 명확)
```

### 3. RouterFunction DSL — 표현력 있는 라우팅

```
RouterFunctions.route() DSL:

  @Bean
  public RouterFunction<ServerResponse> userRoutes(UserHandler handler) {
      return RouterFunctions.route()
          .path("/api/users", builder -> builder
              .GET("", handler::findAll)              // GET /api/users
              .GET("/{id}", handler::findById)        // GET /api/users/{id}
              .POST("", handler::create)              // POST /api/users
              .PUT("/{id}", handler::update)          // PUT /api/users/{id}
              .DELETE("/{id}", handler::delete)       // DELETE /api/users/{id}
          )
          .build();
  }

요청 조건 조합:
  .GET("/admin", RequestPredicates.headers(h ->
      h.firstHeader("X-Admin-Token") != null
  ), handler::adminAction)
  // 헤더 조건까지 조합 가능

  .GET("/stream",
      RequestPredicates.accept(MediaType.TEXT_EVENT_STREAM),
      handler::streamEvents)
  // Content-Type, Accept 조건 조합

중첩 RouterFunction:
  RouterFunction<ServerResponse> v1 = v1Routes();
  RouterFunction<ServerResponse> v2 = v2Routes();

  @Bean
  RouterFunction<ServerResponse> allRoutes() {
      return RouterFunctions.route()
          .path("/v1", () -> v1)
          .path("/v2", () -> v2)
          .build();
  }
```

### 4. HandlerFunction 구현 패턴

```
ServerRequest → ServerResponse:

  @Component
  public class UserHandler {

      // GET /users/{id}
      public Mono<ServerResponse> findById(ServerRequest request) {
          Long id = Long.parseLong(request.pathVariable("id"));

          return userService.findById(id)
              .flatMap(user ->
                  ServerResponse.ok()
                      .contentType(MediaType.APPLICATION_JSON)
                      .bodyValue(user)
              )
              .switchIfEmpty(
                  ServerResponse.notFound().build()
              );
      }

      // POST /users (요청 본문 역직렬화)
      public Mono<ServerResponse> create(ServerRequest request) {
          return request.bodyToMono(CreateUserRequest.class)
              .flatMap(userService::create)
              .flatMap(user ->
                  ServerResponse.created(
                      URI.create("/users/" + user.getId())
                  ).bodyValue(user)
              )
              .onErrorResume(ValidationException.class, e ->
                  ServerResponse.badRequest().bodyValue(e.getMessage())
              );
      }

      // GET /users (스트리밍)
      public Mono<ServerResponse> findAll(ServerRequest request) {
          return ServerResponse.ok()
              .contentType(MediaType.APPLICATION_JSON)
              .body(userService.findAll(), User.class);
      }
  }
```

### 5. 두 방식 혼용 시 주의점

```
혼용 시나리오:
  기존 어노테이션 방식 + 신규 함수형 방식 추가

라우팅 우선순위:
  RouterFunctionMapping: Order = -1 (기본)
  RequestMappingHandlerMapping: Order = 0 (기본)
  → 동일 경로라면 RouterFunction이 먼저 매칭

  동일 경로 중복 등록 시:
    RouterFunction의 route()가 먼저 처리
    → @Controller 핸들러는 실행 안 됨
  
  해결: RouterFunction에 더 구체적인 조건 적용 or 경로 분리

@ExceptionHandler 범위:
  어노테이션 방식: @ControllerAdvice의 @ExceptionHandler 적용
  함수형 방식: 적용 안 됨 → HandlerFunction 내 onErrorResume 필요

Bean Validation:
  어노테이션 방식: @Valid, @Validated 자동 처리
  함수형 방식: 직접 Validator 주입 후 호출 필요
    Mono<CreateUserRequest> validated = request
        .bodyToMono(CreateUserRequest.class)
        .doOnNext(validator::validate);
```

---

## 💻 실전 코드

### 실험 1: 어노테이션 방식

```java
@RestController
@RequestMapping("/api/orders")
@Validated
public class OrderController {

    @GetMapping("/{id}")
    public Mono<ResponseEntity<OrderResponse>> findById(@PathVariable Long id) {
        return orderService.findById(id)
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<OrderResponse> create(@Valid @RequestBody CreateOrderRequest req) {
        return orderService.create(req);
    }

    @GetMapping(produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<OrderEvent> streamEvents() {
        return orderService.eventStream();
    }
}
```

### 실험 2: 함수형 방식

```java
@Configuration
public class OrderRouterConfig {

    @Bean
    public RouterFunction<ServerResponse> orderRoutes(OrderHandler handler) {
        return RouterFunctions.route()
            .path("/api/orders", builder -> builder
                .GET("/{id}", handler::findById)
                .POST("", handler::create)
                .GET("",
                    RequestPredicates.accept(TEXT_EVENT_STREAM),
                    handler::streamEvents)
            )
            .filter((request, next) -> {
                // 함수형 미들웨어 (WebFilter 대신 라우트 수준 필터)
                log.info("라우트 필터: {}", request.uri());
                return next.handle(request);
            })
            .build();
    }
}

@Component
public class OrderHandler {

    public Mono<ServerResponse> findById(ServerRequest request) {
        Long id = Long.parseLong(request.pathVariable("id"));
        return orderService.findById(id)
            .flatMap(order -> ServerResponse.ok().bodyValue(order))
            .switchIfEmpty(ServerResponse.notFound().build());
    }

    public Mono<ServerResponse> create(ServerRequest request) {
        return request.bodyToMono(CreateOrderRequest.class)
            .flatMap(req -> {
                // 직접 검증
                Set<ConstraintViolation<CreateOrderRequest>> violations =
                    validator.validate(req);
                if (!violations.isEmpty()) {
                    return ServerResponse.badRequest()
                        .bodyValue(violations.stream()
                            .map(ConstraintViolation::getMessage)
                            .toList());
                }
                return orderService.create(req)
                    .flatMap(order -> ServerResponse
                        .created(URI.create("/api/orders/" + order.getId()))
                        .bodyValue(order));
            });
    }
}
```

### 실험 3: GraalVM 힌트 차이

```java
// 어노테이션 방식 — 리플렉션 힌트 필요
// native-image.properties 또는 @RegisterReflectionForBinding
@RegisterReflectionForBinding(CreateOrderRequest.class)
@RestController
public class OrderController { ... }

// 함수형 방식 — 리플렉션 힌트 최소화
// RouterFunction + HandlerFunction은 컴파일 타임 타입 참조
// GraalVM이 더 쉽게 정적 분석 가능
@Bean
public RouterFunction<ServerResponse> routes(OrderHandler h) {
    return route().POST("/orders", h::create).build();
}
```

---

## 📊 성능 비교

```
어노테이션 vs 함수형 성능 비교:

요청 처리 오버헤드 (I/O 없는 순수 라우팅 성능):
  어노테이션: 리플렉션 파라미터 바인딩 ~2~5μs
  함수형:     직접 람다 호출 ~0.5~1μs
  차이: ~2~5μs 차이 (요청 처리 100ms 대비 0.002~0.005%)

실무 결론:
  → I/O 집약 서비스에서 이 차이는 측정 오차 수준
  → 성능 이유로 함수형으로 전환 불필요
  → GraalVM Native 또는 코드 스타일 이유가 더 실질적

처리량 비교 (JMH 벤치마크 기준):
  Hello World 수준의 순수 라우팅:
    어노테이션: ~150,000 req/s
    함수형:     ~165,000 req/s
  → 약 10% 차이 (실제 서비스 DB/I/O 있으면 무의미)

GraalVM Native 빌드 시간 및 크기:
  어노테이션: 더 많은 리플렉션 힌트 필요 → 빌드 복잡
  함수형: 힌트 최소화 → 빌드 단순, 실행 파일 크기 소폭 감소
```

---

## ⚖️ 트레이드오프

```
어노테이션 방식:
  장점: 친숙함 (MVC 경험 활용), 보일러플레이트 최소
  단점: 리플렉션 의존, GraalVM 설정 복잡, 라우팅이 분산

함수형 방식:
  장점: 명시적 라우팅 (코드가 문서), GraalVM 친화적, 조합 용이
  단점: 더 많은 코드, Bean Validation 수동, @ExceptionHandler 미적용

혼용 전략:
  소규모 팀/신규 서비스: 하나만 선택 (일관성)
  레거시 마이그레이션: 기존 어노테이션 유지 + 신규는 함수형
  GraalVM 목표: 핵심 경로는 함수형으로 전환

실무 선택 기준:
  Spring MVC 팀 → 어노테이션 방식 (WebFlux로 전환 시)
  새 서비스, GraalVM 고려 → 함수형 방식
  복잡한 라우팅 조건 → 함수형 방식 (RequestPredicates 조합)
```

---

## 📌 핵심 정리

```
어노테이션 vs 함수형 핵심:

어노테이션 방식:
  @RestController + @GetMapping
  RequestMappingHandlerMapping → 리플렉션 바인딩
  @ExceptionHandler, @Valid 자동 지원

함수형 방식:
  RouterFunction + HandlerFunction
  RouterFunctionMapping → 람다 직접 호출 (리플렉션 없음)
  에러/검증 직접 처리 필요

라우팅 우선순위:
  RouterFunctionMapping (Order -1) > RequestMappingHandlerMapping (Order 0)

성능 차이:
  I/O 있는 실제 서비스에서 무시 가능
  GraalVM Native에서 함수형이 유리

선택 기준:
  팀 친숙도, GraalVM 필요 여부, 라우팅 복잡도
```

---

## 🤔 생각해볼 문제

**Q1.** `RouterFunction`에서 `filter()`와 `WebFilter`의 차이는 무엇인가요?

<details>
<summary>해설 보기</summary>

**WebFilter**: 모든 요청에 적용, DispatcherHandler 이전에 실행, 필터 체인 전체.

**RouterFunction의 filter()**: 해당 RouterFunction 내의 라우트에만 적용, HandlerFunction 실행 바로 전/후.

```java
// WebFilter: 모든 요청 (인증, 로깅 전역)
@Component
public class AuthFilter implements WebFilter { ... }

// RouterFunction filter: 특정 라우트 그룹만 (라우트 수준 미들웨어)
RouterFunctions.route()
    .path("/api/admin", builder -> builder
        .GET("", adminHandler::list)
        .POST("", adminHandler::create)
    )
    .filter((request, next) -> {
        // 이 경로 내 요청에만 적용
        if (!isAdmin(request)) {
            return ServerResponse.status(403).build();
        }
        return next.handle(request);
    })
    .build();
```

라우트별로 다른 필터를 적용해야 할 때 `RouterFunction.filter()`가 유용합니다.

</details>

---

**Q2.** 함수형 방식에서 `@Validated`처럼 자동 Bean Validation을 적용하는 방법이 있나요?

<details>
<summary>해설 보기</summary>

함수형 방식에서는 직접 `Validator`를 주입하여 사용합니다. 공통 유틸리티로 추출하면 반복을 줄일 수 있습니다.

```java
@Component
public class RequestValidator {
    private final Validator validator;

    public <T> Mono<T> validate(T body) {
        Set<ConstraintViolation<T>> violations = validator.validate(body);
        if (violations.isEmpty()) {
            return Mono.just(body);
        }
        String message = violations.stream()
            .map(v -> v.getPropertyPath() + ": " + v.getMessage())
            .collect(joining(", "));
        return Mono.error(new ValidationException(message));
    }
}

// HandlerFunction에서 사용
public Mono<ServerResponse> create(ServerRequest request) {
    return request.bodyToMono(CreateUserRequest.class)
        .flatMap(requestValidator::validate)
        .flatMap(userService::create)
        .flatMap(user -> ServerResponse.created(...).bodyValue(user))
        .onErrorResume(ValidationException.class, e ->
            ServerResponse.badRequest().bodyValue(e.getMessage()));
}
```

</details>

---

**Q3.** 함수형 방식의 `RouterFunction`을 여러 파일로 분리하고 조합하는 방법은?

<details>
<summary>해설 보기</summary>

`RouterFunction`은 `andOther()`나 `RouterFunctions.route()`로 조합할 수 있습니다.

```java
// UserRouterConfig.java
@Configuration
public class UserRouterConfig {
    @Bean
    public RouterFunction<ServerResponse> userRoutes(UserHandler h) {
        return RouterFunctions.route()
            .path("/api/users", b -> b
                .GET("/{id}", h::findById)
                .POST("", h::create))
            .build();
    }
}

// OrderRouterConfig.java
@Configuration
public class OrderRouterConfig {
    @Bean
    public RouterFunction<ServerResponse> orderRoutes(OrderHandler h) {
        return RouterFunctions.route()
            .path("/api/orders", b -> b
                .GET("/{id}", h::findById)
                .POST("", h::create))
            .build();
    }
}
```

Spring이 `RouterFunction` 빈을 모두 수집하여 자동으로 조합합니다. `RouterFunctionMapping`이 `ApplicationContext`에서 모든 `RouterFunction` 빈을 가져와 하나의 체인으로 구성합니다. 별도의 조합 코드 없이 분리된 파일이 자동으로 합쳐집니다.

</details>

---

<div align="center">

**[⬅️ 이전: DispatcherHandler](./01-dispatcher-handler.md)** | **[홈으로 🏠](../README.md)** | **[다음: WebClient 완전 분해 ➡️](./03-webclient-internals.md)**

</div>
