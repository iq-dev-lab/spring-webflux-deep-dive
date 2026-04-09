# 연결 관리 — Keep-Alive, HTTP/2, ConnectionProvider

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Keep-Alive`는 TCP 연결을 어떻게 재사용하고, WebFlux에서 어떻게 설정하는가?
- HTTP/2 멀티플렉싱은 단일 TCP 연결에서 여러 요청을 어떻게 동시에 처리하는가?
- `WebClient`의 `ConnectionProvider`는 연결 풀을 어떻게 관리하는가?
- 연결 풀 고갈(Connection Pool Exhaustion)은 어떻게 발생하고 어떻게 예방하는가?
- `WebClient` 연결 풀을 Micrometer로 어떻게 모니터링하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

MSA 환경에서 서비스 간 HTTP 통신은 `WebClient`를 통해 이루어지며, 이 연결들을 풀로 관리합니다. 연결 풀 설정이 잘못되면 "pending acquire timeout" 에러와 함께 서비스가 멈춥니다. HTTP/2를 도입하면 연결 수는 줄지만 내부 스트림 설정이 중요해집니다. 이 문서를 읽고 나면 WebClient의 연결 풀 튜닝 근거를 가질 수 있습니다.

---

## 😱 흔한 실수 (Before — 연결 관리를 모를 때)

```
실수 1: WebClient를 매 요청마다 새로 생성

  @GetMapping("/data")
  public Mono<String> getData() {
      return WebClient.create("http://external-api")
          .get().uri("/data").retrieve()
          .bodyToMono(String.class);
      // 매 요청마다 새 WebClient = 새 ConnectionProvider = 새 연결 풀
      // 연결 재사용 없음, 매번 TCP 핸드셰이크
      // 기존 연결 정리도 안 됨 → 연결 누수 위험
  }

실수 2: ConnectionProvider 기본 설정 사용 (너무 많은 연결 허용)

  WebClient webClient = WebClient.create("http://external-api");
  // 기본 ConnectionProvider: 최대 500개 연결 (per remote)
  // 외부 API가 100개 연결만 허용하면 → 501번째 연결 거부
  // 또는 외부 API에 과도한 부하

실수 3: Keep-Alive 타임아웃과 서버 설정 불일치

  // 서버: Keep-Alive 30초 설정
  // 클라이언트: 60초 연결 재사용 시도
  // → 서버가 30초에 연결 닫음 → 클라이언트가 죽은 연결로 요청
  // → "Connection reset by peer" 에러
```

---

## ✨ 올바른 접근 (After — 연결 풀 적절히 설정)

```
WebClient 올바른 사용 패턴:

1. WebClient를 Bean으로 등록 (재사용)
   @Bean
   WebClient webClient(WebClient.Builder builder) {
       return builder.baseUrl("http://external-api").build();
   }

2. ConnectionProvider를 서비스 특성에 맞게 설정
   ConnectionProvider provider = ConnectionProvider.builder("api-pool")
       .maxConnections(50)              // 최대 연결 수
       .maxIdleTime(Duration.ofSeconds(20))  // 유휴 연결 유지 시간
       .maxLifeTime(Duration.ofSeconds(60))  // 연결 최대 수명
       .pendingAcquireTimeout(Duration.ofSeconds(60))  // 획득 대기 제한
       .build();

3. HTTP/2 (h2c) 사용으로 연결 효율 향상
   HttpClient httpClient = HttpClient.create(provider)
       .protocol(HttpProtocol.H2C);  // HTTP/2 cleartext
```

---

## 🔬 내부 동작 원리

### 1. HTTP Keep-Alive — TCP 연결 재사용

```
HTTP/1.0 기본: Connection: close (매 요청마다 새 TCP 연결)
  연결 비용:
    TCP 3-way handshake: ~RTT (수십 ms)
    TLS handshake (HTTPS): ~2 × RTT
  → 요청마다 수십~수백 ms 연결 비용

HTTP/1.1 기본: Connection: keep-alive (연결 재사용)
  동작:
    최초 요청: TCP 연결 수립 → 요청 → 응답
    다음 요청: 동일 TCP 연결 재사용 → 요청 → 응답
    ...
    타임아웃 또는 서버 close: 연결 종료

Netty 서버 Keep-Alive 기본 설정:
  HTTP/1.1: Keep-Alive 헤더 없이 기본 활성화
  비활성화:
    httpServer.option(ChannelOption.SO_KEEPALIVE, false)

Keep-Alive 타임아웃:
  서버 측: 클라이언트가 idle 상태로 N초 지나면 연결 종료
  WebFlux 설정:
    server.netty.idle-timeout=30s (application.yml)
    또는
    httpServer.idleTimeout(Duration.ofSeconds(30))

클라이언트 WebClient Keep-Alive:
  ConnectionProvider.builder("pool")
      .maxIdleTime(Duration.ofSeconds(20))  // 유휴 연결 20초 유지
      // 서버의 30초 타임아웃보다 짧게 설정 (서버가 먼저 닫기 전에 클라이언트가 닫음)
      .build()
```

### 2. HTTP/2 멀티플렉싱 — 단일 연결로 다수 요청

```
HTTP/1.1의 한계:
  연결당 한 번에 1개 요청 처리 (순서 보장)
  병렬 요청: 연결 여러 개 필요 (브라우저 기본 6~8개)
  Head-of-line blocking: 앞 요청이 느리면 뒤도 대기

HTTP/2 멀티플렉싱:
  단일 TCP 연결에서 여러 스트림 동시 처리
  각 요청/응답 = 독립적 스트림 (스트림 ID로 구분)
  순서 무관 처리 (Head-of-line blocking 없음)

HTTP/2 스트림 구조:
  TCP 연결
    ├─ 스트림 1: GET /api/users  → 응답 중...
    ├─ 스트림 3: GET /api/orders → 응답 중...
    ├─ 스트림 5: POST /api/items → 응답 중...
    └─ 스트림 7: GET /api/config → 응답 완료 (먼저 완료됨)

  → 하나의 TCP 연결에서 4개 동시 요청/응답

HTTP/2 설정:
  서버 측:
    httpServer.protocol(HttpProtocol.H2)    // HTTPS + HTTP/2
    httpServer.protocol(HttpProtocol.H2C)   // HTTP + HTTP/2 (개발용)

  클라이언트 (WebClient):
    HttpClient.create().protocol(HttpProtocol.H2)

  SETTINGS_MAX_CONCURRENT_STREAMS:
    서버가 허용하는 동시 스트림 수 제한
    기본: Integer.MAX_VALUE (Netty)
    실무: 100~1000 사이로 설정 권장
```

### 3. ConnectionProvider — WebClient 연결 풀

```
ConnectionProvider 내부 구조:

  ConnectionProvider
    └── PooledConnectionProvider (기본)
          ├── remote1 → FixedChannelPool (연결 풀)
          ├── remote2 → FixedChannelPool
          └── remote3 → FixedChannelPool
    각 remote(호스트:포트)마다 독립 연결 풀

ConnectionProvider 주요 설정:

  ConnectionProvider.builder("my-provider")
    .maxConnections(50)
    // 각 remote에 대한 최대 연결 수
    // 기본: 500 (너무 많음, 외부 API 제한 고려)

    .maxIdleTime(Duration.ofSeconds(20))
    // 유휴 연결 유지 시간 (서버 Keep-Alive보다 짧게)
    // 초과 시 연결 닫고 제거

    .maxLifeTime(Duration.ofSeconds(60))
    // 연결 최대 수명 (장기간 사용 후 갱신)
    // TLS 인증서 갱신, 서버 재시작 등 대응

    .pendingAcquireTimeout(Duration.ofSeconds(45))
    // 연결 획득 대기 최대 시간
    // 모든 연결이 사용 중일 때 큐에서 대기하는 시간
    // 초과 시 PoolAcquireTimeoutException

    .pendingAcquireMaxCount(100)
    // 연결 획득 대기 큐의 최대 크기
    // 초과 시 즉시 PoolAcquireTimeoutException

    .evictInBackground(Duration.ofSeconds(120))
    // 백그라운드에서 만료된 연결 주기적 정리
    .build()
```

### 4. 연결 풀 고갈 — 원인과 해결

```
연결 풀 고갈 시나리오:
  maxConnections = 10, pendingAcquireTimeout = 45s

  t=0:  10개 연결 모두 사용 중 (외부 API 응답 느림)
  t=5:  새 요청 → 연결 없음 → 대기 큐에 적재
  t=10: 더 많은 요청 대기 (누적)
  t=45: 대기 큐 타임아웃 → PoolAcquireTimeoutException

  에러 로그:
  reactor.netty.internal.shaded.reactor.pool.PoolAcquireTimeoutException:
  Pool#acquire(Duration) has been pending for more than the configured timeout

원인 분석:
  1. maxConnections가 너무 적음 (트래픽 대비)
  2. 연결을 오래 점유 (외부 API 응답 느림, 타임아웃 설정 없음)
  3. 연결 반환 안 됨 (구독 취소 없이 방치)

해결책:
  1. maxConnections 조정 (외부 API 허용 범위 내에서)
  2. 요청 타임아웃 추가:
     webClient.get()
         .uri("/api")
         .retrieve()
         .bodyToMono(String.class)
         .timeout(Duration.ofSeconds(5));  // 5초 초과 시 반환

  3. 연결 풀 크기 모니터링 후 조정
  4. Circuit Breaker 적용 (외부 API 장애 시 연결 점유 차단)
```

### 5. WebClient 연결 풀 모니터링

```
Micrometer 메트릭 활성화:

ConnectionProvider provider = ConnectionProvider.builder("api-pool")
    .maxConnections(50)
    .metrics(true)  // Micrometer 메트릭 활성화
    .build();

주요 메트릭:
  reactor.netty.connection.provider.total.connections
    → 현재 총 연결 수

  reactor.netty.connection.provider.active.connections
    → 현재 활성(사용 중) 연결 수

  reactor.netty.connection.provider.idle.connections
    → 현재 유휴 연결 수

  reactor.netty.connection.provider.pending.connections
    → 현결 획득 대기 중인 요청 수

경보 설정 (Prometheus + Alertmanager):
  # 대기 연결이 10개 이상이면 경보
  - alert: WebClientPoolExhausted
    expr: reactor_netty_connection_provider_pending_connections > 10
    for: 1m
    annotations:
      summary: "WebClient 연결 풀 고갈 위험"

  # 활성 연결이 max의 80% 초과 시 경보
  - alert: WebClientPoolHighUtilization
    expr: reactor_netty_connection_provider_active_connections /
          reactor_netty_connection_provider_total_connections > 0.8
```

---

## 💻 실전 코드

### 실험 1: 연결 풀 설정 완전 예시

```java
@Configuration
public class WebClientConfig {

    @Bean
    public ConnectionProvider paymentApiConnectionProvider() {
        return ConnectionProvider.builder("payment-api-pool")
            .maxConnections(20)                             // 결제 API 최대 연결
            .maxIdleTime(Duration.ofSeconds(15))            // 서버 30s보다 짧게
            .maxLifeTime(Duration.ofMinutes(2))             // 연결 수명
            .pendingAcquireTimeout(Duration.ofSeconds(30))  // 대기 타임아웃
            .pendingAcquireMaxCount(50)                     // 대기 큐 크기
            .evictInBackground(Duration.ofMinutes(1))       // 백그라운드 정리
            .metrics(true)                                  // 모니터링
            .build();
    }

    @Bean("paymentWebClient")
    public WebClient paymentWebClient(
            ConnectionProvider paymentApiConnectionProvider) {
        HttpClient httpClient = HttpClient.create(paymentApiConnectionProvider)
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
            .responseTimeout(Duration.ofSeconds(5))
            .doOnConnected(conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(5))
                .addHandlerLast(new WriteTimeoutHandler(5))
            );

        return WebClient.builder()
            .baseUrl("https://payment-api.example.com")
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .defaultHeader(HttpHeaders.CONTENT_TYPE,
                MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
}
```

### 실험 2: HTTP/2 클라이언트 설정

```java
@Bean("http2WebClient")
public WebClient http2WebClient() {
    HttpClient httpClient = HttpClient.create()
        .protocol(HttpProtocol.H2)  // HTTP/2 (TLS 필요)
        // 또는 .protocol(HttpProtocol.H2C)  // HTTP/2 without TLS (개발용)
        .secure(sslSpec -> sslSpec
            .sslContext(SslContextBuilder.forClient()
                .applicationProtocolConfig(new ApplicationProtocolConfig(
                    Protocol.ALPN,
                    SelectorFailureBehavior.NO_ADVERTISE,
                    SelectedListenerFailureBehavior.ACCEPT,
                    ApplicationProtocolNames.HTTP_2
                ))
                .build()
            )
        );

    return WebClient.builder()
        .baseUrl("https://h2-api.example.com")
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();
}
```

### 실험 3: 연결 풀 상태 확인 엔드포인트

```java
@RestController
@RequestMapping("/actuator/webClient")
public class WebClientPoolStatusController {
    private final ConnectionProvider connectionProvider;
    private final MeterRegistry meterRegistry;

    @GetMapping("/pool-status")
    public Mono<Map<String, Object>> poolStatus() {
        return Mono.fromCallable(() -> {
            Map<String, Object> status = new HashMap<>();

            // Micrometer에서 연결 풀 메트릭 조회
            Gauge active = meterRegistry.find(
                "reactor.netty.connection.provider.active.connections"
            ).gauge();
            Gauge pending = meterRegistry.find(
                "reactor.netty.connection.provider.pending.connections"
            ).gauge();

            status.put("activeConnections",
                active != null ? active.value() : -1);
            status.put("pendingAcquireCount",
                pending != null ? pending.value() : -1);
            return status;
        }).subscribeOn(Schedulers.boundedElastic());
    }
}
```

---

## 📊 성능 비교

```
HTTP/1.1 vs HTTP/2 비교 (외부 API 10개 동시 요청):

HTTP/1.1 Keep-Alive:
  연결 10개 필요 (동시 요청 = 동시 연결)
  각 연결: TCP 핸드셰이크 생략 (Keep-Alive 덕분)
  처리 시간: max(10개 요청 시간)

HTTP/2 멀티플렉싱:
  연결 1개 (하나의 TCP에서 10개 스트림)
  연결 오버헤드 90% 감소
  처리 시간: max(10개 요청 시간) — 동일
  서버 연결 수: 1/10로 감소

연결 풀 maxConnections 조정 효과:

  maxConnections=10, 동시 100 요청:
    90개 대기 → pendingAcquireTimeout 위험
  maxConnections=50, 동시 100 요청:
    50개 즉시 처리, 50개 대기 → 안정적
  maxConnections=100, 동시 100 요청:
    모두 즉시 처리 → 외부 API에 100개 연결 부하

Keep-Alive vs 매번 새 연결:
  TLS 포함 연결 수립: ~100ms
  Keep-Alive 재사용: ~1ms
  → 고빈도 API 호출 시 Keep-Alive로 수십 배 성능 향상
```

---

## ⚖️ 트레이드오프

```
연결 풀 설정 트레이드오프:

maxConnections 높음:
  장점: 대기 없이 즉시 연결 확보
  단점: 외부 API 부하, 서버 자원 사용 증가

maxConnections 낮음:
  장점: 외부 API 보호, 자원 절약
  단점: 대기 발생 → 레이턴시 증가 위험

maxIdleTime 설정:
  서버 Keep-Alive보다 짧게: 안전 (서버가 먼저 닫기 전에 클라이언트가 닫음)
  서버 Keep-Alive보다 길게: "Connection reset by peer" 위험

HTTP/2 사용 트레이드오프:
  장점: 연결 수 감소, Head-of-line blocking 없음
  단점: TLS 필요 (h2c는 비추), 설정 복잡, 디버깅 어려움
        서버도 HTTP/2 지원 필요

연결 풀 모니터링:
  필수 지표: active, pending, idle 연결 수
  경보: pending > maxConnections × 20% → 풀 확장 검토
```

---

## 📌 핵심 정리

```
연결 관리 핵심:

Keep-Alive:
  TCP 연결 재사용 → 핸드셰이크 비용 제거
  maxIdleTime을 서버 Keep-Alive보다 짧게 설정

HTTP/2 멀티플렉싱:
  단일 TCP 연결에서 여러 스트림 동시 처리
  연결 수 감소, Head-of-line blocking 없음
  TLS 필수 (h2)

ConnectionProvider:
  WebClient 연결 풀 관리
  maxConnections: 외부 API 허용 범위 내 설정
  pendingAcquireTimeout: 반드시 설정 (기본 45초)
  모니터링: metrics(true) + Micrometer

연결 풀 고갈 방지:
  요청 timeout 설정 (responseTimeout)
  Circuit Breaker 적용
  pending 메트릭 경보 설정
```

---

## 🤔 생각해볼 문제

**Q1.** HTTP/2를 사용할 때 `WebClient` 연결 풀의 `maxConnections`를 HTTP/1.1보다 훨씬 낮게 설정해도 되는 이유는 무엇인가요?

<details>
<summary>해설 보기</summary>

HTTP/1.1에서는 **연결당 1개의 요청**만 처리하므로, 동시 요청 수만큼 연결이 필요합니다. 50개 동시 요청 → 50개 연결.

HTTP/2에서는 **하나의 연결에서 여러 스트림(요청)을 동시에** 처리합니다. `SETTINGS_MAX_CONCURRENT_STREAMS`가 100이라면, 연결 1개로 100개 요청을 동시에 처리 가능합니다.

```java
// HTTP/1.1: 50개 동시 요청 처리
ConnectionProvider.builder("h1-pool").maxConnections(50).build();

// HTTP/2: 동일한 50개 동시 요청을 연결 5개로 처리 가능
// (연결당 10개 스트림 기준)
ConnectionProvider.builder("h2-pool").maxConnections(5).build();
```

단, 실제 `SETTINGS_MAX_CONCURRENT_STREAMS` 값에 따라 최적 `maxConnections`가 달라집니다. 서버의 `max-concurrent-streams` 설정을 확인하고 `maxConnections = ceil(동시요청 / max-concurrent-streams)`로 계산합니다.

</details>

---

**Q2.** `WebClient`를 `@Bean`으로 등록하지 않고 매 요청마다 `WebClient.create()`를 호출하면 어떤 문제가 생기나요?

<details>
<summary>해설 보기</summary>

`WebClient.create()`는 내부적으로 새로운 `ConnectionProvider`와 새로운 `EventLoopGroup`을 생성합니다.

**연결 재사용 불가**: 매번 새 연결 풀이므로 이전 연결을 재사용할 수 없습니다. Keep-Alive의 이점이 사라집니다.

**자원 누수**: 생성된 `ConnectionProvider`와 `EventLoopGroup`이 명시적으로 닫히지 않으면 스레드와 소켓 자원이 누수됩니다.

**과도한 연결**: 초당 100 요청이 있다면 매번 새 TCP 연결 → DB나 외부 API에 초당 100개 연결 시도.

올바른 방법:
```java
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder.baseUrl("http://api.example.com").build();
}
// 애플리케이션 전체에서 하나의 WebClient 인스턴스 재사용
// 내부 ConnectionProvider도 공유 → 연결 풀 재사용
```

`WebClient.Builder`는 Spring Boot가 자동으로 Bean으로 등록하며, 적절한 기본 설정(Actuator 통합 등)도 포함합니다.

</details>

---

**Q3.** `maxIdleTime`이 서버의 Keep-Alive 타임아웃보다 길 때 발생하는 "Connection reset by peer" 에러를 어떻게 방지하나요?

<details>
<summary>해설 보기</summary>

서버가 먼저 연결을 닫은 후 클라이언트가 해당 연결로 요청을 보내면 RST(연결 초기화) 패킷을 받아 "Connection reset by peer"가 발생합니다.

**주요 해결책:**

1. **`maxIdleTime`을 서버 타임아웃보다 짧게 설정** (핵심):
```java
// 서버 Keep-Alive: 30초
// 클라이언트 maxIdleTime: 20초 (안전 마진 확보)
ConnectionProvider.builder("pool")
    .maxIdleTime(Duration.ofSeconds(20))
    .build()
```

2. **`evictInBackground`로 주기적 정리**:
```java
.evictInBackground(Duration.ofSeconds(30))
// 30초마다 만료된 연결 제거
```

3. **자동 재연결 허용** (Reactor Netty 기본):
Reactor Netty는 연결 오류 시 자동으로 새 연결을 시도합니다. 단, 이미 보낸 요청의 재시도는 별도 설정 필요 (`retryWhen`).

안전 마진:
- `서버 Keep-Alive × 0.7` 을 `maxIdleTime`으로 설정 (약 30% 여유)
- 서버 설정을 확인할 수 없을 때: 보수적으로 10~15초

</details>

---

<div align="center">

**[⬅️ 이전: EventLoop 블로킹 위험](./04-blocking-in-eventloop.md)** | **[홈으로 🏠](../README.md)**

</div>
