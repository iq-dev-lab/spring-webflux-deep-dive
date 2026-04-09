# Context — Reactive에서 ThreadLocal을 쓸 수 없는 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 비동기 파이프라인에서 스레드가 바뀔 때 `ThreadLocal`이 왜 유실되는가?
- Reactor `Context`는 어떻게 파이프라인 전체에 데이터를 전달하는가?
- `contextWrite`와 `deferContextual`은 어떻게 사용하는가?
- Spring Security의 `ReactiveSecurityContextHolder`는 어떤 원리로 동작하는가?
- MDC(Mapped Diagnostic Context) 로깅을 WebFlux에서 어떻게 구현하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

웹 요청마다 고유한 `requestId`를 로그에 남기는 것은 분산 추적의 기본입니다. MVC에서는 `MDC.put("requestId", id)`를 필터에서 한 번 설정하면 요청 처리 스레드 전체에서 사용할 수 있습니다. WebFlux에서 같은 방식을 쓰면 `publishOn`으로 스레드가 바뀌는 순간 MDC 값이 사라집니다.

Spring Security의 `Authentication` 전달, MDC 기반 로깅, 요청 추적 ID 전파 — 이 모든 것이 Reactor Context를 이해해야 올바르게 구현됩니다.

---

## 😱 흔한 실수 (Before — ThreadLocal을 Reactive에서 사용)

```
실수 1: WebFlux 필터에서 MDC 설정 후 손실

  // WebFilter에서 requestId 설정
  MDC.put("requestId", UUID.randomUUID().toString());
  return chain.filter(exchange)
      .doFinally(signal -> MDC.remove("requestId"));

  // 컨트롤러에서
  @GetMapping("/data")
  public Mono<String> getData() {
      return webClient.get().uri("/api").retrieve()
          .bodyToMono(String.class)
          .publishOn(Schedulers.parallel())  // 스레드 전환!
          .map(s -> {
              log.info("requestId: {}", MDC.get("requestId"));
              // → null! 스레드가 바뀌어 MDC 유실
              return s;
          });
  }

실수 2: Reactive Security에서 ThreadLocal SecurityContext 접근

  // 잘못된 방식
  Authentication auth = SecurityContextHolder.getContext().getAuthentication();
  // → WebFlux에서는 null 반환 (ThreadLocal 기반)

  // 올바른 방식
  ReactiveSecurityContextHolder.getContext()
      .map(ctx -> ctx.getAuthentication())
      .flatMap(auth -> process(auth));
```

---

## ✨ 올바른 접근 (After — Reactor Context 사용)

```
Reactor Context 특징:
  - 파이프라인의 "불변 메타데이터 맵"
  - 스레드 전환과 무관하게 파이프라인 전체에서 접근 가능
  - 구독(subscribe)에서 소스 방향으로 전파 (아래→위)
  - 읽기는 어디서든 가능

기본 패턴:
  // Context에 값 추가 (contextWrite)
  mono.contextWrite(Context.of("requestId", requestId))

  // Context에서 값 읽기 (deferContextual)
  Mono.deferContextual(ctx -> {
      String requestId = ctx.get("requestId");
      return process(requestId);
  })

Spring Security와의 통합:
  ReactiveSecurityContextHolder.withAuthentication(auth)
  → Reactor Context에 SecurityContext를 저장
  → 파이프라인 어디서든 ReactiveSecurityContextHolder.getContext()로 접근
```

---

## 🔬 내부 동작 원리

### 1. ThreadLocal 유실 원리

```
MVC에서 ThreadLocal (동작):

  Thread-1: MDC.put("requestId", "abc")
            → Thread-1의 ThreadLocal에 저장
  Thread-1: 컨트롤러 실행
            → MDC.get("requestId") → "abc" (동일 스레드)
  Thread-1: 응답 완료 → MDC.remove("requestId")

  단일 스레드에서 요청 시작~끝 → ThreadLocal 항상 유효

────────────────────────────────────────

WebFlux에서 ThreadLocal (실패):

  EventLoop-1: MDC.put("requestId", "abc")
               → EventLoop-1의 ThreadLocal에 저장
  EventLoop-1: 컨트롤러 실행 → Mono 파이프라인 시작

  publishOn(Schedulers.parallel()) 발생:
    parallel-1 스레드로 전환
    parallel-1: MDC.get("requestId") → null!
                EventLoop-1의 ThreadLocal은 parallel-1에 없음

  또한:
    EventLoop가 다른 요청을 처리하는 동안
    같은 EventLoop가 다른 요청의 MDC도 덮어씀 가능
    → 멀티 요청 환경에서 MDC 값 오염 위험
```

### 2. Reactor Context 구조

```
Reactor Context = 불변 key-value 맵

  Context.empty():         빈 Context
  Context.of("k", "v"):   하나의 키-값
  ctx.put("k", "v"):      새 Context 반환 (불변)
  ctx.get("k"):           값 조회
  ctx.getOrDefault("k", default): 기본값으로 조회

전파 방향:
  contextWrite()는 subscribe() 방향으로 전파 (아래→위)
  → 파이프라인을 구성할 때 상위 연산자는 Context를 볼 수 없음
  → contextWrite는 파이프라인 맨 아래에 위치해야 함

예시:
  Mono.fromCallable(() -> {
      // 여기서 Context 읽기 → contextWrite 이후이므로 OK
      return "data";
  })
  .transformDeferredContextual((mono, ctx) ->
      mono.map(s -> s + "-" + ctx.getOrDefault("reqId", "none"))
  )
  .contextWrite(Context.of("reqId", "abc123"))  // 맨 아래에 위치
  .subscribe();
```

### 3. contextWrite와 deferContextual 사용법

```
contextWrite(Function<Context, Context>):
  파이프라인에 Context를 추가/수정
  subscribe() 시 Context가 상위로 전파

  mono.contextWrite(ctx -> ctx.put("key", "value"))
  또는
  mono.contextWrite(Context.of("key", "value"))

deferContextual(Function<ContextView, Mono<T>>):
  Context를 읽어 새로운 Mono를 반환
  구독 시점에 Context 읽기

  Mono<String> result = Mono.deferContextual(ctx -> {
      String reqId = ctx.getOrDefault("requestId", "none");
      return Mono.just("result-" + reqId);
  });

transformDeferredContextual:
  기존 Mono/Flux에 Context를 결합
  중간 연산자로 사용

  flux.transformDeferredContextual((f, ctx) ->
      f.map(item -> enrich(item, ctx.get("userId")))
  )

주의: Context 읽기 시점
  contextWrite 이후에만 해당 키에 접근 가능
  파이프라인 위(upstream)에서는 contextWrite 이전이므로 키 없음

올바른 위치:
  Mono.fromCallable(...)         // 소스
      .flatMap(...)               // 처리
      .contextWrite(ctx ->        // ← 맨 아래
          ctx.put("requestId", requestId)
      )
  
잘못된 위치:
  Mono.contextWrite(...)         // 여기 설정
      .fromCallable(...)          // X — fromCallable은 contextWrite 위
```

### 4. MDC 로깅 — Reactor Context와 연동

```
WebFlux에서 MDC 전파 패턴:

// WebFilter에서 requestId를 Context에 설정
@Component
public class RequestIdFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String requestId = UUID.randomUUID().toString();

        return chain.filter(exchange)
            .contextWrite(Context.of("requestId", requestId));
        // Context에 requestId 저장 → 파이프라인 전체 전파
    }
}

// 로깅 시 Context에서 MDC 복원
// Reactor Hook을 사용하여 자동화:
Hooks.onEachOperator("mdc-propagator",
    Operators.lift((scannable, subscriber) ->
        new MdcContextSubscriber<>(subscriber)
    )
);

// MdcContextSubscriber: onNext 수신 시 Context에서 MDC 값 복원 후 실행
// → 모든 연산자에서 MDC 자동 복원
// 또는 reactor-logback-appender 같은 라이브러리 사용
```

### 5. Spring Security와 Reactor Context

```
ReactiveSecurityContextHolder 내부:

// Security Context를 Reactor Context에 저장하는 필터
public class ReactiveSecurityContextHolder {
    static final String SECURITY_CONTEXT_KEY = "...SecurityContext";

    public static Mono<SecurityContext> getContext() {
        return Mono.deferContextual(ctx ->
            ctx.hasKey(SECURITY_CONTEXT_KEY)
                ? ctx.get(SECURITY_CONTEXT_KEY)
                : Mono.empty()
        );
    }

    public static Function<Context, Context> withSecurityContext(
            Mono<SecurityContext> securityContext) {
        return ctx -> ctx.put(SECURITY_CONTEXT_KEY, securityContext);
    }
}

// 인증 필터에서 Context에 설정
return chain.filter(exchange)
    .contextWrite(ReactiveSecurityContextHolder
        .withSecurityContext(Mono.just(securityContext)));

// 서비스에서 사용
public Mono<Void> checkPermission() {
    return ReactiveSecurityContextHolder.getContext()
        .map(ctx -> ctx.getAuthentication())
        .flatMap(auth -> {
            if (!auth.getAuthorities().contains("ADMIN"))
                return Mono.error(new AccessDeniedException("권한 없음"));
            return Mono.empty();
        });
}
```

---

## 💻 실전 코드

### 실험 1: Context 기본 사용

```java
@Test
void contextBasicUsage() {
    Mono<String> result = Mono.deferContextual(ctx ->
        Mono.just("Hello, " + ctx.getOrDefault("name", "World"))
    )
    .contextWrite(Context.of("name", "Reactor"));

    StepVerifier.create(result)
        .expectNext("Hello, Reactor")
        .expectComplete()
        .verify();
}
```

### 실험 2: WebFilter에서 requestId 전파

```java
@Component
@Order(-1)
public class RequestTraceFilter implements WebFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String requestId = exchange.getRequest().getHeaders()
            .getFirst("X-Request-ID");
        if (requestId == null) {
            requestId = UUID.randomUUID().toString();
        }
        final String id = requestId;

        return chain.filter(exchange)
            .contextWrite(Context.of("requestId", id));
    }
}

// 서비스에서 requestId 읽기
@Service
public class AuditService {
    public Mono<Void> auditAction(String action) {
        return Mono.deferContextual(ctx -> {
            String requestId = ctx.getOrDefault("requestId", "unknown");
            log.info("[{}] 감사 로그: {}", requestId, action);
            return auditRepository.save(new AuditLog(requestId, action));
        });
    }
}
```

### 실험 3: Micrometer Tracing과 Context 통합

```java
// Micrometer Tracing + WebFlux: Context에 Span 전파
@Configuration
public class TracingConfig {

    @Bean
    public WebFilter tracingFilter(Tracer tracer) {
        return (exchange, chain) -> {
            Span span = tracer.nextSpan()
                .name("http-" + exchange.getRequest().getPath())
                .start();

            return chain.filter(exchange)
                .contextWrite(ctx -> ctx.put("currentSpan", span))
                .doFinally(signal -> span.end());
        };
    }
}

// 서비스에서 Span을 이용한 트레이싱
public Mono<User> findUser(Long id) {
    return Mono.deferContextual(ctx -> {
        Span parentSpan = ctx.getOrDefault("currentSpan", null);
        return userRepository.findById(id);
    });
}
```

---

## 📊 성능 비교

```
ThreadLocal vs Reactor Context 비교:

특성              | ThreadLocal          | Reactor Context
─────────────────┼─────────────────────┼─────────────────────
스레드 안전성      | 스레드 로컬 (격리)    | 불변 (공유 안전)
스레드 전환        | 유실                 | 자동 전파
메모리            | 스레드당 저장         | 구독당 저장
접근 방식         | 전역 (MDC.get)       | 파이프라인 내 (deferContextual)
오버헤드          | 없음 (직접 접근)      | 약간 (ContextView 조회)

Reactor Context 오버헤드:
  Context 읽기: Map 조회 → ~수십 ns
  contextWrite: 불변 Context 생성 → ~수십 ns
  → 일반적인 I/O 지배적 파이프라인에서 무시 가능

Context 크기:
  큰 객체를 Context에 저장하면 모든 연산자가 참조
  → 대용량 객체 저장 지양 (ID, 메타데이터 수준 권장)
```

---

## ⚖️ 트레이드오프

```
Reactor Context 트레이드오프:

장점:
  ① 스레드 전환 무관 → publishOn 해도 Context 유지
  ② 불변 → Thread-safe, 동시성 문제 없음
  ③ Spring Security, Micrometer 등 에코시스템 통합

단점:
  ① 명시적 API → ThreadLocal보다 코드 장황
  ② 전파 방향 주의 → contextWrite는 파이프라인 아래에 위치
  ③ 외부 라이브러리와의 연동 복잡
     MDC는 ThreadLocal 기반 → Hook으로 자동 복원 필요

MVC에서 WebFlux로 마이그레이션 시:
  ThreadLocal 기반 코드 전면 점검 필요
  - MDC 로깅: Hooks + MdcContextSubscriber 도입
  - SecurityContextHolder: ReactiveSecurityContextHolder 교체
  - RequestAttributes: ServerWebExchange + Context 활용
```

---

## 📌 핵심 정리

```
Reactor Context 핵심:

ThreadLocal 유실 이유:
  스레드가 바뀌면 (publishOn, subscribeOn 등)
  이전 스레드의 ThreadLocal 값은 새 스레드에 없음

Reactor Context 특징:
  파이프라인의 불변 메타데이터 맵
  subscribe() 방향 (아래→위)으로 전파
  스레드 전환과 무관하게 파이프라인 전체 접근 가능

사용법:
  contextWrite(Context.of("key", value)):  Context 추가
  Mono.deferContextual(ctx -> ...):        Context 읽기 후 Mono 생성
  transformDeferredContextual(...):        중간 연산자에서 Context 결합

Spring Security:
  ReactiveSecurityContextHolder → Reactor Context 기반
  SecurityContext가 파이프라인 전체에 전파됨

MDC 로깅:
  contextWrite로 requestId 전파
  Hooks.onEachOperator로 MDC 자동 복원 또는
  Logback + reactor-logback-appender 사용
```

---

## 🤔 생각해볼 문제

**Q1.** `contextWrite`를 파이프라인 중간에 넣으면 어떻게 되나요?

<details>
<summary>해설 보기</summary>

Context는 **아래에서 위 방향(subscribe 방향)**으로 전파됩니다. 따라서 `contextWrite`는 그 위치보다 **하위(downstream)**의 코드에만 영향을 줍니다.

```java
Mono.fromCallable(() -> {
    // Context 없음 — contextWrite 이전(upstream)
    return "data";
})
.map(s -> s.toUpperCase())  // Context 없음
.contextWrite(Context.of("key", "value"))  // 여기 설정
.map(s -> {
    // Context 없음 — contextWrite 이후의 downstream에서 읽어야 함
    return s;
})
.subscribe();
```

올바른 패턴은 `contextWrite`를 파이프라인의 **가장 마지막**에 배치하고, Context 읽기는 `deferContextual`이나 `transformDeferredContextual`로 수행하는 것입니다. Spring WebFlux 필터에서 `chain.filter(exchange).contextWrite(...)` 패턴이 올바른 이유이기도 합니다.

</details>

---

**Q2.** 여러 `contextWrite`를 체인하면 나중 것이 덮어쓰나요, 아니면 합쳐지나요?

<details>
<summary>해설 보기</summary>

Context는 불변이며, `contextWrite`를 여러 번 사용하면 **기존 Context에 새 키-값이 추가된 새 Context**가 생성됩니다. 덮어쓰지 않고 합쳐집니다.

```java
mono.contextWrite(Context.of("a", "1"))
    .contextWrite(Context.of("b", "2"))
    // 실제 Context: {"a": "1", "b": "2"}

// 단, 같은 키를 두 번 넣으면 아래쪽(마지막 contextWrite)이 유효
mono.contextWrite(Context.of("key", "new"))  // subscribe 방향으로 먼저 적용
    .contextWrite(Context.of("key", "old"))  // 이게 덮어씌워짐

// subscribe 방향 = 아래 → 위
// "new"를 설정한 contextWrite가 subscribe 시 먼저 만남
// "old"는 이미 "new"가 있으므로 기존 값 유지 (put은 이미 있으면 기존 유지)
// → 실제로는 가장 마지막 contextWrite("new")가 우선
```

혼란을 피하려면 하나의 `contextWrite`에서 모든 키-값을 설정하는 것이 좋습니다.

</details>

---

**Q3.** MDC를 WebFlux에서 완벽하게 지원하기 위해 `Hooks.onEachOperator`를 사용하는 방법의 단점은 무엇인가요?

<details>
<summary>해설 보기</summary>

`Hooks.onEachOperator`는 **모든** Reactor 연산자에 Subscriber 래핑을 추가합니다. 이는 두 가지 단점이 있습니다.

**성능 오버헤드:**
모든 `onNext` 신호마다 Context를 읽고 MDC를 설정/해제합니다. 초당 수만 개의 신호가 있는 고처리량 파이프라인에서는 눈에 띄는 오버헤드가 될 수 있습니다.

**전역 설정:**
애플리케이션 전체 Reactor 파이프라인에 영향을 미칩니다. 테스트 환경에서 의도치 않게 동작에 영향을 줄 수 있습니다.

대안:
- **Logback + MDC propagating Appender**: Reactor 스케줄러와 통합된 Appender 사용
- **Micrometer Tracing (Observation API)**: Spring Boot 3+의 자동 Tracing 통합, Context 자동 전파
- **개별 연산자에서 명시적 MDC 설정**: Hook 없이 필요한 지점에서만 `deferContextual`로 처리

실무에서는 Spring Boot 3 + Micrometer Tracing 조합이 WebFlux MDC 전파의 권장 방식입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Backpressure 전략](./06-backpressure-strategies.md)** | **[홈으로 🏠](../README.md)** | **[다음: 테스트 ➡️](./08-testing-stepverifier.md)**

</div>
