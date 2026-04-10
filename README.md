<div align="center">

# ⚡ Spring WebFlux Deep Dive

**"스레드를 늘리는 것과, 스레드가 I/O를 기다리는 시간을 없애는 것은 다르다"**

<br/>

> *"`WebClient` 쓰면 되겠지 — 와 — 왜 스레드 8개로 10,000 동시 연결을 처리할 수 있는지, `subscribe()` 전까지 파이프라인이 왜 실행되지 않는지, `block()`이 이벤트 루프를 어떻게 죽이는지 아는 것의 차이를 만드는 레포"*

C10K 문제가 왜 스레드 기반 모델의 한계인지, Netty의 `EventLoop`가 `epoll`로 수천 개의 소켓을 소수의 스레드로 처리하는 원리, `Mono`/`Flux`가 지연 평가(Lazy Evaluation)인 이유, Backpressure가 생산자와 소비자를 어떻게 조율하는지까지  
**왜 이렇게 설계됐는가** 라는 질문으로 Spring WebFlux 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Spring WebFlux](https://img.shields.io/badge/Spring_WebFlux-6.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html)
[![Project Reactor](https://img.shields.io/badge/Project_Reactor-3.x-FF6B6B?style=flat-square&logo=reactivex&logoColor=white)](https://projectreactor.io/docs/core/release/reference/)
[![Docs](https://img.shields.io/badge/Docs-40개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

Spring WebFlux에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "`WebClient`를 쓰면 됩니다" | `RestTemplate`이 왜 스레드를 블로킹하는지, `WebClient`가 `Netty`의 이벤트 루프와 어떻게 연결되는지, `ConnectionProvider`가 연결을 재사용하는 원리 |
| "`Mono`/`Flux`를 반환하세요" | `subscribe()` 호출 전까지 파이프라인이 왜 실행되지 않는지(Cold Publisher), 연산자 체인이 어떻게 `Publisher` → `Subscriber` 신호 그래프를 만드는지 |
| "WebFlux는 빠릅니다" | 스레드 기반 Tomcat 모델이 I/O 대기 중 CPU를 낭비하는 이유, `EventLoop`가 `epoll`로 수천 개의 소켓을 하나의 스레드에서 처리하는 원리 |
| "`block()`을 쓰지 마세요" | `EventLoop` 스레드에서 `block()`이 왜 데드락을 유발하는지, `BlockHound`로 블로킹 코드를 감지하는 방법, `Schedulers.boundedElastic()`으로 오프로딩해야 하는 이유 |
| "`flatMap`을 쓰세요" | `flatMap`(순서 무보장, 최대 동시성) vs `concatMap`(순서 보장, 순차 처리) vs `switchMap`(최신 것만 유지)의 내부 동작 차이 |
| "R2DBC를 쓰면 됩니다" | JPA가 내부적으로 JDBC 블로킹 스레드를 사용하는 이유, WebFlux에서 JPA가 `EventLoop`를 블로킹하는 문제, R2DBC가 없는 기능(지연 로딩, 복잡한 조인) 트레이드오프 |
| 이론 나열 | 실행 가능한 Spring WebFlux + R2DBC + `BlockHound` 실험 + Docker Compose 환경(PostgreSQL, Redis, WireMock, k6) |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Reactive](https://img.shields.io/badge/🔹_Chapter1-블로킹_모델의_한계-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./reactive-foundations/01-thread-per-request-model.md)
[![Reactor](https://img.shields.io/badge/🔹_Chapter2-Mono와_Flux_지연_평가-FF6B6B?style=for-the-badge&logo=reactivex&logoColor=white)](./project-reactor/01-mono-flux-lazy-evaluation.md)
[![Netty](https://img.shields.io/badge/🔹_Chapter3-Netty_아키텍처-4A90D9?style=for-the-badge&logo=apache&logoColor=white)](./netty-event-loop/01-netty-architecture.md)
[![WebFlux](https://img.shields.io/badge/🔹_Chapter4-DispatcherHandler-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./spring-webflux-core/01-dispatcher-handler.md)
[![R2DBC](https://img.shields.io/badge/🔹_Chapter5-JPA와_WebFlux의_충돌-DC382D?style=for-the-badge&logo=postgresql&logoColor=white)](./r2dbc-data-access/01-jpa-webflux-conflict.md)
[![Security](https://img.shields.io/badge/🔹_Chapter6-Reactive_Security-6DB33F?style=for-the-badge&logo=springsecurity&logoColor=white)](./security-reactive/01-security-web-filter-chain.md)
[![Performance](https://img.shields.io/badge/🔹_Chapter7-MVC_vs_WebFlux_비교-FF6B6B?style=for-the-badge&logo=grafana&logoColor=white)](./performance-patterns/01-mvc-vs-webflux-benchmark.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: 왜 Reactive인가 — 블로킹 모델의 한계

> **핵심 질문:** 스레드 기반 모델은 왜 10,000 동시 연결에서 실패하는가? 논블로킹 I/O는 어떻게 스레드 없이 I/O 완료를 통보받는가? Reactive Streams 스펙은 무엇을 표준화했는가?

<details>
<summary><b>Thread-per-Request 한계부터 Reactive Streams 스펙 완전 분해까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 전통적 스레드 기반 모델의 한계 — Thread-per-Request](./reactive-foundations/01-thread-per-request-model.md) | 요청당 스레드 모델에서 스레드가 I/O 대기 중 CPU를 낭비하는 원리, 컨텍스트 스위칭 비용, 스레드 풀 고갈 시 요청 큐 대기와 응답 지연, Tomcat 기본 스레드 200개의 한계 |
| [02. C10K 문제 — 동시 연결 10,000개의 벽](./reactive-foundations/02-c10k-problem.md) | 스레드 하나 ~1MB 메모리 × 10,000 = 10GB 계산, Apache가 Nginx에 밀린 이유(prefork vs event 모델), C10K를 돌파한 이벤트 루프 방식의 핵심 차이 |
| [03. 논블로킹 I/O의 원리 — epoll과 이벤트 루프](./reactive-foundations/03-nonblocking-io-epoll.md) | I/O 작업을 커널에 위임하고 완료 이벤트를 통보받는 방식, `epoll_wait`으로 수천 개의 소켓을 단일 시스템 콜로 감시하는 원리, 이벤트 루프가 I/O 완료 이벤트를 처리하는 흐름 |
| [04. Reactive Programming의 탄생 — 콜백에서 스트림으로](./reactive-foundations/04-reactive-programming-history.md) | 비동기 콜백의 Callback Hell 문제, `Promise`/`Future`가 해결한 것과 못 한 것, Reactive Streams 스펙이 표준화한 4개 인터페이스(`Publisher`/`Subscriber`/`Subscription`/`Processor`) |
| [05. WebFlux vs Spring MVC — 언제 무엇을 선택할 것인가](./reactive-foundations/05-webflux-vs-mvc.md) | I/O 집약적 서비스(외부 API 다수 호출, 스트리밍)에서 WebFlux의 우위, CPU 집약적 서비스에서 MVC와 차이 없는 이유, 블로킹 라이브러리(JPA) 사용 시 WebFlux의 함정, 팀 학습 비용 |
| [06. Reactive Streams 스펙 완전 분해 — push vs pull](./reactive-foundations/06-reactive-streams-spec.md) | `Publisher`가 `Subscriber`에게 데이터를 push하는 방식, `Subscription.request(n)`으로 소비자가 생산자를 조절하는 Backpressure 원리, `onNext`/`onError`/`onComplete` 신호 흐름 |

</details>

<br/>

### 🔹 Chapter 2: Project Reactor 핵심

> **핵심 질문:** `Mono`와 `Flux`는 왜 지연 평가인가? `flatMap`과 `concatMap`은 내부에서 어떻게 다르게 동작하는가? Backpressure 전략 4가지는 어떤 상황에서 선택하는가?

<details>
<summary><b>Lazy Evaluation 원리부터 StepVerifier 테스트까지 (8개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Mono와 Flux — 왜 subscribe() 전에 실행되지 않는가](./project-reactor/01-mono-flux-lazy-evaluation.md) | `Mono`(0~1개 값)와 `Flux`(0개 이상 값)의 차이, Cold Publisher의 지연 평가 원리, `subscribe()` 호출 시 `Subscription` 생성 후 `request(n)` 신호로 데이터 흐름이 시작되는 과정 |
| [02. Reactor 연산자 완전 분해 — map vs flatMap vs concatMap](./project-reactor/02-reactor-operators.md) | `map`(동기 변환) vs `flatMap`(비동기 변환, 순서 무보장) vs `concatMap`(순서 보장, 순차 처리) vs `switchMap`(최신 것만 유지) 내부 동작, 마블 다이어그램(텍스트 버전), 실수하기 쉬운 패턴 |
| [03. 에러 처리 — Reactive에서 try-catch가 없는 이유](./project-reactor/03-error-handling.md) | `onErrorReturn`(기본값 반환) / `onErrorResume`(대체 Publisher) / `onErrorMap`(에러 변환) / `doOnError`(로깅) 차이, `retry`/`retryWhen`(지수 백오프), 에러 신호가 파이프라인을 타고 전파되는 원리 |
| [04. 스케줄러(Scheduler) — 스레드 전환이 일어나는 시점](./project-reactor/04-scheduler-thread-switching.md) | `subscribeOn`(구독 시 실행 스레드 결정) vs `publishOn`(이후 연산자 실행 스레드 전환), `Schedulers.boundedElastic()`(블로킹 작업 오프로딩) vs `Schedulers.parallel()`(CPU 집약적), 스레드 전환 시점 |
| [05. Cold vs Hot Publisher — 구독 시점의 차이](./project-reactor/05-cold-vs-hot-publisher.md) | Cold: 구독할 때마다 새로운 데이터 스트림 생성, Hot: 구독 시점과 관계없이 진행 중인 스트림, `publish().refCount()`로 Cold를 Hot으로 변환, 실시간 이벤트 스트림에서 Hot Publisher 필요성 |
| [06. Backpressure 전략 — BUFFER / DROP / LATEST / ERROR](./project-reactor/06-backpressure-strategies.md) | 생산자가 소비자보다 빠를 때 `OutOfMemoryError`가 발생하는 원리, `BUFFER`(소비 못한 데이터 버퍼) / `DROP`(초과 데이터 버림) / `LATEST`(최신 것만 유지) / `ERROR`(소비자 지연 시 에러) 각 전략의 메모리·정확성 트레이드오프 |
| [07. Context — Reactive에서 ThreadLocal을 쓸 수 없는 이유](./project-reactor/07-reactor-context.md) | 비동기 파이프라인에서 스레드가 바뀔 때 `ThreadLocal`이 유실되는 이유, Reactor `Context`로 파이프라인 전체에 데이터 전달하는 방식, Spring Security의 `ReactiveSecurityContextHolder` 동작 원리 |
| [08. 테스트 — StepVerifier와 WebTestClient](./project-reactor/08-testing-stepverifier.md) | `StepVerifier`로 `Mono`/`Flux`의 신호를 단계별로 검증하는 방법, `VirtualTimeScheduler`로 `delayElements` 시간 기반 연산자 테스트, `TestPublisher`로 커스텀 시나리오, `WebTestClient`로 통합 테스트 |

</details>

<br/>

### 🔹 Chapter 3: Netty와 이벤트 루프

> **핵심 질문:** Netty의 `Boss`/`Worker` `EventLoopGroup`은 어떻게 분리되는가? `EventLoop` 스레드에서 블로킹 코드가 왜 전체 채널을 멈추는가? HTTP/2 멀티플렉싱은 어떻게 단일 연결로 여러 요청을 처리하는가?

<details>
<summary><b>Netty 아키텍처부터 연결 풀 관리까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Netty 아키텍처 완전 분해 — Boss/Worker EventLoopGroup](./netty-event-loop/01-netty-architecture.md) | `Boss EventLoopGroup`(연결 수락, `accept` 시스템 콜), `Worker EventLoopGroup`(I/O 처리, 데이터 읽기/쓰기), `Channel Pipeline`, `ChannelHandler` 체인, Spring WebFlux가 Netty를 서버로 사용하는 방식 |
| [02. EventLoop 내부 동작 — 하나의 스레드가 여러 Channel을 담당](./netty-event-loop/02-event-loop-internals.md) | 하나의 `EventLoop`가 여러 `Channel`을 담당하는 방식, I/O 이벤트와 태스크 큐를 단일 스레드에서 처리하는 루프 구조, CPU 코어 수 × 2를 `EventLoopGroup` 크기로 설정하는 이유 |
| [03. ChannelPipeline과 Handler — 요청이 처리되는 경로](./netty-event-loop/03-channel-pipeline-handler.md) | `Inbound`(읽기 방향) vs `Outbound`(쓰기 방향) 핸들러, HTTP 디코딩/인코딩 핸들러 체인, WebFlux가 Netty `ChannelHandler`를 Spring `RouterFunction`에 연결하는 방법 |
| [04. EventLoop에서 블로킹 코드의 위험 — 전체 채널이 멈추는 이유](./netty-event-loop/04-blocking-in-eventloop.md) | `EventLoop` 스레드에서 `Thread.sleep()` / JDBC 호출 시 해당 `EventLoop`가 담당하는 모든 채널이 멈추는 원리, `BlockHound`로 블로킹 호출을 감지하는 방법, `Schedulers.boundedElastic()`로 오프로딩하는 패턴 |
| [05. 연결 관리 — Keep-Alive, HTTP/2, ConnectionProvider](./netty-event-loop/05-connection-management.md) | `Keep-Alive`로 TCP 연결을 재사용하는 방식, HTTP/2 멀티플렉싱으로 단일 연결에서 여러 요청을 동시에 처리하는 원리, `ConnectionProvider`로 `WebClient`의 연결 풀을 설정·모니터링하는 방법 |

</details>

<br/>

### 🔹 Chapter 4: Spring WebFlux 핵심

> **핵심 질문:** `DispatcherHandler`는 `DispatcherServlet`과 어떻게 다른가? `WebClient`의 `retrieve()`와 `exchangeToMono()`는 에러 처리에서 왜 다르게 동작하는가? SSE와 WebSocket은 언제 선택하는가?

<details>
<summary><b>DispatcherHandler부터 WebFilter까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. WebFlux 아키텍처 — DispatcherHandler와 Reactive 처리 파이프라인](./spring-webflux-core/01-dispatcher-handler.md) | `DispatcherHandler`가 `DispatcherServlet`을 대체하는 방식, `HandlerMapping`/`HandlerAdapter`/`HandlerResultHandler`의 Reactive 버전, `ServerWebExchange`로 요청/응답을 불변 객체로 다루는 이유 |
| [02. 어노테이션 기반 vs 함수형 엔드포인트 — 언제 무엇을 선택하는가](./spring-webflux-core/02-annotation-vs-functional.md) | `@RestController` + `@GetMapping`(MVC 스타일, 어노테이션 처리 오버헤드), `RouterFunction` + `HandlerFunction`(함수형 스타일, 명시적 라우팅), 각각의 장단점과 선택 기준, 두 방식 혼용 시 주의점 |
| [03. WebClient 완전 분해 — RestTemplate과 근본적으로 다른 이유](./spring-webflux-core/03-webclient-internals.md) | `RestTemplate`의 블로킹 방식과 `WebClient`의 논블로킹 방식 비교, `retrieve()`(4xx/5xx 자동 에러) vs `exchangeToMono()`(직접 응답 제어), 재시도 전략(`retryWhen`), 타임아웃(`responseTimeout`), `ConnectionProvider` 설정 |
| [04. WebClient 고급 패턴 — 병렬 호출과 서킷 브레이커](./spring-webflux-core/04-webclient-advanced-patterns.md) | 여러 외부 API 병렬 호출(`Mono.zip`, `Flux.merge`)의 성능 이점, 순차 호출 vs 병렬 호출 응답 시간 비교, `Resilience4j Reactive` 서킷 브레이커 통합, `WebClient`를 Bean으로 관리하는 패턴 |
| [05. 스트리밍 응답 — SSE와 WebSocket](./spring-webflux-core/05-streaming-sse-websocket.md) | `Server-Sent Events`(SSE)로 서버에서 클라이언트로 실시간 데이터 push하는 방식, `Flux<ServerSentEvent>`로 SSE 구현, WebSocket 양방향 통신 구현, SSE vs WebSocket 선택 기준 |
| [06. 요청/응답 처리 — ServerRequest와 Reactive 기반 본문 읽기](./spring-webflux-core/06-request-response-handling.md) | `ServerRequest`/`ServerResponse`로 HTTP 요청을 Reactive 방식으로 처리, `multipart/form-data` 스트리밍 처리, 파일 업로드/다운로드를 `Flux<DataBuffer>`로 스트리밍하는 방법, 메모리 누수 방지 (`DataBufferUtils.release`) |
| [07. WebFilter — Reactive 기반 필터 체인](./spring-webflux-core/07-web-filter.md) | `WebFilter`가 Servlet `Filter`와 다르게 `Mono<Void>`를 반환하는 이유, 인증/로깅/CORS를 `WebFilter`로 처리하는 방법, `WebFilter` 순서 지정(`@Order`/`Ordered`), `ExchangeFilterFunction`으로 `WebClient`에 필터 적용 |

</details>

<br/>

### 🔹 Chapter 5: R2DBC와 Reactive 데이터 액세스

> **핵심 질문:** JPA가 왜 WebFlux의 EventLoop를 블로킹하는가? R2DBC는 JPA의 어떤 기능을 포기했고, 그 트레이드오프는 무엇인가? Reactive 컨텍스트에서 트랜잭션은 어떻게 전파되는가?

<details>
<summary><b>JPA-WebFlux 충돌부터 Redis Reactive까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. JPA와 WebFlux의 충돌 — 블로킹 JDBC 문제](./r2dbc-data-access/01-jpa-webflux-conflict.md) | JPA가 내부적으로 블로킹 JDBC를 사용하기 때문에 `EventLoop` 스레드를 점유하는 문제, `Schedulers.boundedElastic()`로 임시 해결하는 방법과 그 한계(스레드 수 증가, C10K 문제 부활), R2DBC로의 전환 판단 기준 |
| [02. R2DBC 완전 분해 — 논블로킹 DB 드라이버의 구조](./r2dbc-data-access/02-r2dbc-internals.md) | JDBC를 완전히 대체하는 논블로킹 DB 드라이버 구조, `DatabaseClient` 저수준 API, Spring Data R2DBC Repository 사용법, JPA와 비교한 기능 제약(지연 로딩 없음, 복잡한 조인 제한) |
| [03. Reactive Transaction — @Transactional이 Reactive 컨텍스트에서 동작하는 방식](./r2dbc-data-access/03-reactive-transaction.md) | Reactor `Context`를 통해 트랜잭션 바인딩 정보를 전달하는 방식, `@Transactional`이 `TransactionalOperator`로 동작하는 내부 원리, 프로그래밍 방식 트랜잭션 관리, 트랜잭션 롤백 시 Reactive 파이프라인에서의 에러 전파 |
| [04. R2DBC 실전 패턴 — N+1 문제와 배치 처리](./r2dbc-data-access/04-r2dbc-practical-patterns.md) | Reactive에서 N+1 문제를 `Flux.flatMap` 배치 처리로 해결하는 방법, `DatabaseClient`로 복잡한 쿼리 작성, `R2DBC Connection Pool`(r2dbc-pool) 설정, 배치 INSERT/UPDATE 최적화 |
| [05. Redis Reactive — ReactiveRedisTemplate와 Pub/Sub](./r2dbc-data-access/05-redis-reactive.md) | `Lettuce` 기반 Reactive Redis 클라이언트 구조, `ReactiveRedisTemplate` 사용법과 `RedisTemplate` 차이, Reactive Pub/Sub 구현, WebFlux + Redis 조합으로 실시간 기능(알림, 채팅) 구현 |

</details>

<br/>

### 🔹 Chapter 6: Spring Security Reactive

> **핵심 질문:** `SecurityWebFilterChain`은 어떻게 `FilterChainProxy`를 대체하는가? Reactive 파이프라인에서 현재 사용자 정보를 어떻게 전달하는가?

<details>
<summary><b>Reactive Security 아키텍처부터 OAuth2 Reactive까지 (4개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Reactive Security 아키텍처 — SecurityWebFilterChain](./security-reactive/01-security-web-filter-chain.md) | `SecurityWebFilterChain`이 `FilterChainProxy`를 대체하는 방식, `ReactiveAuthenticationManager`로 인증을 비동기 처리하는 원리, `ServerSecurityContextRepository`가 Reactor `Context`에 `SecurityContext`를 저장하는 방법 |
| [02. JWT 인증 Reactive 구현 — WebFilter에서 토큰 검증](./security-reactive/02-jwt-reactive-auth.md) | `WebFilter`에서 JWT 추출·검증 후 `ReactiveSecurityContextHolder`에 인증 정보를 저장하는 전 과정, 각 요청마다 `SecurityContext`를 Reactor `Context`에 전달하는 방법, 필터 체인 순서와 단락(short-circuit) 처리 |
| [03. Method Security — Reactive 메서드에서 @PreAuthorize](./security-reactive/03-method-security.md) | `@PreAuthorize`가 `Mono`/`Flux`를 반환하는 메서드에서 동작하는 방식, `ReactiveMethodSecurityConfiguration`이 AOP를 Reactive 파이프라인에 끼워 넣는 원리, `Mono<Authentication>`으로 현재 사용자 정보 접근 |
| [04. OAuth2 Reactive — ReactiveOAuth2AuthorizedClientManager](./security-reactive/04-oauth2-reactive.md) | Reactive 기반 OAuth2 Login 흐름, `ReactiveOAuth2AuthorizedClientManager`로 토큰을 관리하는 방식, `ServerOAuth2AuthorizedClientExchangeFilterFunction`으로 `WebClient`에 OAuth2 토큰을 자동 첨부하는 방법 |

</details>

<br/>

### 🔹 Chapter 7: 성능 튜닝과 실전 패턴

> **핵심 질문:** MVC와 WebFlux의 처리량 차이는 어떤 조건에서 나타나는가? BlockHound로 블로킹 코드를 어떻게 감지하는가? 언제 WebFlux를 쓰지 말아야 하는가?

<details>
<summary><b>벤치마크부터 WebFlux를 쓰지 말아야 할 때까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. MVC vs WebFlux 성능 비교 — k6 부하 테스트](./performance-patterns/01-mvc-vs-webflux-benchmark.md) | 외부 API 호출 수에 따른 처리량 차이(동시 연결 100 / 500 / 1000), I/O 집약적 서비스에서 WebFlux의 우위가 나타나는 조건, CPU 집약적 서비스에서 차이가 없는 이유, k6 부하 테스트 스크립트 |
| [02. 운영 중 발생하는 문제 패턴 — BlockHound와 메모리 누수](./performance-patterns/02-production-problem-patterns.md) | `BlockHound`로 `EventLoop` 스레드에서 블로킹 호출을 감지하는 방법, 구독 해제가 안 된 `Flux`로 인한 메모리 누수 진단, 스케줄러 선택 실수(`parallel`에서 블로킹 작업 실행)로 인한 성능 저하 |
| [03. Reactive Caching — Mono.cache()와 중복 구독 방지](./performance-patterns/03-reactive-caching.md) | `Mono.cache()`로 구독할 때마다 외부 API를 호출하는 중복 구독 문제를 방지하는 방법, Caffeine/Redis와 Reactor 통합 패턴, 캐시된 `Publisher`의 갱신 전략, TTL 기반 자동 갱신 구현 |
| [04. 마이크로서비스에서 WebFlux — 서비스 간 호출 체인](./performance-patterns/04-microservice-webflux.md) | 서비스 간 `WebClient` 호출 체인에서 에러 전파와 폴백 처리, Reactive 기반 Circuit Breaker(`Resilience4j`) 통합, `Micrometer` + Zipkin으로 분산 추적 시 `Reactive Context`를 통한 TraceId 전파 |
| [05. 언제 WebFlux를 쓰지 말아야 하는가 — 실무 판단 기준](./performance-patterns/05-when-not-to-use-webflux.md) | 블로킹 라이브러리 의존성이 많은 경우, 팀이 Reactive 패러다임에 익숙하지 않은 경우의 생산성 손실, 단순 CRUD 서비스에서 학습 비용 대비 이득이 없는 이유, 스택 트레이스 파편화와 디버깅 난이도 |

</details>

---

## 🔬 실험 환경

```yaml
# docker-compose.yml
services:
  # WebFlux 앱 (R2DBC + 논블로킹)
  webflux-app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_R2DBC_URL: r2dbc:postgresql://postgres:5432/webflux_db
      SPRING_REDIS_HOST: redis

  # MVC 앱 (JDBC + 블로킹) — Ch7-01 MVC vs WebFlux 처리량 비교 실험에 필요
  mvc-app:
    build:
      context: .
      dockerfile: Dockerfile.mvc
    ports:
      - "8081:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/webflux_db
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: postgres
    depends_on:
      - postgres

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: webflux_db
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"

  redis:
    image: redis:7.0
    ports:
      - "6379:6379"

  # 외부 API 목킹 — 느린 응답(200ms~2s) 시뮬레이션
  wiremock:
    image: wiremock/wiremock:latest
    ports:
      - "8090:8080"
    volumes:
      - ./wiremock:/home/wiremock

  # 부하 테스트 — MVC vs WebFlux 처리량 비교
  k6:
    image: grafana/k6:latest
    volumes:
      - ./k6:/scripts
```

```java
// BlockHound — 개발/테스트 환경에서만 블로킹 코드 감지
// ⚠️ 운영 환경에서는 사용 금지 (성능 오버헤드 3~8%, EventLoop 예외로 앱 중단 가능)
@SpringBootApplication
public class WebFluxApplication {
    public static void main(String[] args) {
        // -DblockHound VM 옵션이 있을 때만 설치 (개발/CI 환경 전용)
        // 예: java -DblockHound -jar app.jar
        if (System.getProperty("blockHound") != null) {
            BlockHound.install();
        }
        SpringApplication.run(WebFluxApplication.class, args);
    }
}
```

```java
// CI 파이프라인에서는 테스트 코드로 감지 (운영 코드 수정 불필요)
@BeforeAll
static void installBlockHound() {
    BlockHound.install();  // 테스트 JVM에서만 실행됨
}
```

```java
// Reactive 파이프라인 실행 원리 핵심 패턴
Mono<String> result = webClient.get()
    .uri("/users/1")
    .retrieve()
    .bodyToMono(User.class)         // 아직 실행 안 됨 — 파이프라인 설계도만 생성
    .map(user -> user.getName())    // 아직 실행 안 됨
    .flatMap(name ->                // 아직 실행 안 됨
        emailService.findByName(name))
    .onErrorReturn("unknown");      // 아직 실행 안 됨

// subscribe() 호출 시점에 Subscription 생성 → request(n) 신호 → 데이터 흐름 시작
result.subscribe(
    value -> log.info("Result: {}", value),
    error -> log.error("Error", error)
);
```

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 개념이 WebFlux에서 중요한가** | 실무에서 마주치는 문제 상황과 이 개념의 연결 |
| 😱 **흔한 실수** | Before — 블로킹 방식 또는 잘못된 Reactive 코드와 그 결과 |
| ✨ **올바른 접근** | After — 올바른 Reactive 코드 |
| 🔬 **내부 동작 원리** | Reactor 연산자 내부 / Netty 이벤트 루프 / 신호 전파 + 마블 다이어그램(텍스트 버전) |
| 💻 **실전 코드** | Spring WebFlux + WebClient + R2DBC 기반 실험 코드 |
| 📊 **성능 비교** | MVC vs WebFlux 처리량 / 블로킹 vs 논블로킹 응답 시간 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "subscribe()를 빠뜨려서 요청이 실행 안 됐다" — Reactive 긴급 투입 (3일)</b></summary>

<br/>

```
Day 1  Ch1-01  Thread-per-Request 한계 → 왜 Reactive가 필요한지
       Ch1-06  Reactive Streams 스펙 → Publisher/Subscriber 신호 원리
Day 2  Ch2-01  Mono/Flux 지연 평가 → subscribe() 전까지 실행 안 되는 이유
       Ch2-02  map vs flatMap vs concatMap → 연산자 선택 기준
Day 3  Ch4-03  WebClient 완전 분해 → RestTemplate과의 차이
       Ch3-04  EventLoop 블로킹 위험 → block() 금지 이유
```

</details>

<details>
<summary><b>🟡 "WebFlux 내부를 제대로 이해하고 싶다" — 핵심 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch1-03  논블로킹 I/O + epoll 원리
       Ch1-02  C10K 문제 → 이벤트 루프의 탄생 배경
Day 2  Ch2-01  Mono/Flux 지연 평가
       Ch2-02  Reactor 연산자 완전 분해
Day 3  Ch2-06  Backpressure 전략 4가지
       Ch2-04  스케줄러와 스레드 전환 시점
Day 4  Ch3-01  Netty Boss/Worker EventLoopGroup
       Ch3-02  EventLoop 내부 동작 원리
Day 5  Ch4-03  WebClient 완전 분해
       Ch4-04  WebClient 병렬 호출 고급 패턴
Day 6  Ch5-01  JPA와 WebFlux 충돌
       Ch5-02  R2DBC 완전 분해
Day 7  Ch7-01  MVC vs WebFlux 성능 비교
       Ch7-05  언제 WebFlux를 쓰지 말아야 하는가
```

</details>

<details>
<summary><b>🔴 "Reactor 소스코드까지 파고들어 내부를 완전히 정복한다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — 블로킹 모델의 한계와 Reactive 탄생 배경
        → epoll 시스템 콜을 strace로 관찰, Thread-per-Request vs EventLoop 처리량 측정

2주차  Chapter 2 전체 — Project Reactor 핵심
        → StepVerifier로 연산자 신호 흐름 단계별 검증, Backpressure 전략별 메모리 사용량 비교

3주차  Chapter 3 전체 — Netty와 이벤트 루프
        → BlockHound로 EventLoop 블로킹 코드 감지, 연결 수에 따른 EventLoopGroup 크기 실험

4주차  Chapter 4 전체 — Spring WebFlux 핵심
        → WebClient 병렬 호출 vs 순차 호출 응답 시간 비교, SSE 실시간 스트리밍 구현

5주차  Chapter 5 전체 — R2DBC와 Reactive 데이터 액세스
        → JPA vs R2DBC 처리량 비교, Reactive Transaction 롤백 시나리오 실험

6주차  Chapter 6 전체 — Spring Security Reactive
        → JWT 인증 WebFilter 구현, ReactiveSecurityContextHolder Context 전달 확인

7주차  Chapter 7 전체 — 성능 튜닝과 실전 패턴
        → k6로 MVC vs WebFlux 부하 테스트, BlockHound 블로킹 코드 감지 실험
```

</details>

---

## 🔗 연관 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [spring-mvc-deep-dive](https://github.com/dev-book-lab/spring-mvc-deep-dive) | `DispatcherServlet`, 스레드 기반 모델 | Ch1-01(Thread-per-Request 한계), Ch4-01(DispatcherHandler vs DispatcherServlet 대비) |
| [linux-for-backend-deep-dive](https://github.com/dev-book-lab/linux-for-backend-deep-dive) | `epoll`, 논블로킹 I/O, 시스템 콜 | Ch1-03(논블로킹 I/O 원리), Ch3-01(Netty가 epoll을 사용하는 방식) |
| [network-deep-dive](https://github.com/dev-book-lab/network-deep-dive) | TCP 연결, HTTP/1.1 vs HTTP/2 | Ch3-05(HTTP/2 멀티플렉싱과 WebClient 연결 재사용) |
| [spring-boot-internals](https://github.com/dev-book-lab/spring-boot-internals) | Auto-configuration | Ch4-01(WebFlux 자동 설정 — `ReactiveWebServerFactoryAutoConfiguration`) |
| [redis-deep-dive](https://github.com/dev-book-lab/redis-deep-dive) | Redis 단일 스레드 이벤트 루프, Pub/Sub | Ch5-05(Reactive Redis Pub/Sub — 두 이벤트 루프 모델 비교) |

> 💡 이 레포는 **Spring WebFlux 내부 동작**에 집중합니다. `linux-for-backend-deep-dive`의 epoll 챕터와 `spring-mvc-deep-dive`의 스레드 모델 챕터를 선행하면 Chapter 1~3을 훨씬 깊이 이해할 수 있습니다. R2DBC 없이 시작하더라도 Chapter 1~4는 독립적으로 학습 가능합니다.

---

## 🙏 Reference

- [Project Reactor 공식 문서](https://projectreactor.io/docs/core/release/reference/)
- [Spring WebFlux 공식 문서](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html)
- [Reactive Programming with RxJava — Tomasz Nurkiewicz, Ben Christensen](https://www.oreilly.com/library/view/reactive-programming-with/9781491931646/)
- [Hands-On Reactive Programming in Spring 5 — Oleh Dokuka, Igor Lozynskyi](https://www.packtpub.com/product/hands-on-reactive-programming-in-spring-5/9781787284951)
- [Netty in Action — Norman Maurer, Marvin Wolfthal](https://www.manning.com/books/netty-in-action)
- [BlockHound — 블로킹 코드 감지 라이브러리](https://github.com/reactor/BlockHound)
- [Reactive Streams 스펙](https://www.reactive-streams.org/)
- [The C10K Problem — Dan Kegel](http://www.kegel.com/c10k.html)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

**"스레드를 늘리는 것과, 스레드가 I/O를 기다리는 시간을 없애는 것은 다르다"**

</div>
