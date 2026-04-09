# 스트리밍 응답 — SSE와 WebSocket

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- SSE(Server-Sent Events)는 일반 HTTP 스트리밍과 어떻게 다른가?
- `Flux<ServerSentEvent>`로 SSE를 구현할 때 재연결과 이벤트 ID는 어떻게 처리하는가?
- WebSocket 양방향 통신은 WebFlux에서 어떻게 구현하는가?
- SSE와 WebSocket을 각각 어떤 상황에서 선택해야 하는가?
- SSE 연결이 끊어졌을 때 서버 측 리소스는 어떻게 정리되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

실시간 알림, 주식 시세, 배송 추적, 채팅 — 이 모든 기능이 서버에서 클라이언트로 데이터를 밀어줘야 합니다. 폴링(Polling)은 단순하지만 불필요한 요청이 많고, Long Polling은 서버 자원을 낭비합니다. SSE와 WebSocket은 하나의 연결을 유지하며 실시간으로 데이터를 전달합니다. 잘못 구현하면 클라이언트가 연결을 끊어도 서버에서 스트림이 계속 실행되어 리소스가 누수됩니다.

---

## 😱 흔한 실수 (Before — SSE/WebSocket을 잘못 구현할 때)

```
실수 1: SSE 연결 해제 시 서버 스트림 미정리

  @GetMapping(value = "/events", produces = TEXT_EVENT_STREAM_VALUE)
  public Flux<StockPrice> streamEvents() {
      return Flux.interval(Duration.ofSeconds(1))
          .map(i -> stockService.getCurrentPrice());
      // 클라이언트가 연결 끊어도 Flux.interval은 계속 실행
      // subscribe가 취소되면 자동 정리되지만
      // Hot Publisher 소스라면 별도 정리 필요
  }

실수 2: SSE에서 Flux를 List로 변환 후 전송

  // 잘못된 방식: 모든 데이터를 먼저 수집 후 전송
  public Mono<List<Event>> streamEvents() {
      return eventFlux.collectList();  // 실시간 스트리밍 아님
  }
  
  // 올바른 방식: Flux 자체를 반환
  public Flux<Event> streamEvents() {
      return eventFlux;
  }

실수 3: WebSocket에서 inbound 처리 없이 outbound만 사용

  @Component
  public class ChatHandler implements WebSocketHandler {
      @Override
      public Mono<Void> handle(WebSocketSession session) {
          return session.send(
              outboundFlux.map(session::textMessage)
          );
          // inbound(클라이언트 메시지)를 구독하지 않음
          // → 클라이언트 메시지 처리 안 됨
          // → 연결 종료 신호도 못 받음
      }
  }
```

---

## ✨ 올바른 접근 (After — SSE/WebSocket 올바른 구현)

```
SSE 기본 패턴:
  @GetMapping(value = "/events", produces = TEXT_EVENT_STREAM_VALUE)
  public Flux<ServerSentEvent<Data>> stream() {
      return dataFlux
          .map(data -> ServerSentEvent.<Data>builder()
              .id(String.valueOf(data.getSequence()))
              .event("data-update")
              .data(data)
              .build()
          )
          .doOnCancel(() -> log.info("SSE 연결 해제"));
      // Flux가 취소(cancel)되면 클라이언트 연결 종료를 의미
      // doOnCancel로 정리 로직 실행

WebSocket 기본 패턴:
  @Override
  public Mono<Void> handle(WebSocketSession session) {
      // inbound와 outbound 모두 처리
      Mono<Void> inbound = session.receive()
          .doOnNext(msg -> processMessage(msg.getPayloadAsText()))
          .then();

      Mono<Void> outbound = session.send(
          outboundFlux.map(session::textMessage)
      );

      return Mono.zip(inbound, outbound).then();
      // inbound 또는 outbound가 완료/에러 시 세션 종료
  }
```

---

## 🔬 내부 동작 원리

### 1. SSE — HTTP 스트리밍의 표준화

```
SSE 프로토콜:
  HTTP/1.1 기반 (일반 HTTP)
  Content-Type: text/event-stream
  단방향: 서버 → 클라이언트

SSE 메시지 형식:
  id: 1\n
  event: price-update\n
  data: {"symbol":"AAPL","price":150.25}\n
  retry: 3000\n
  \n   ← 빈 줄이 메시지 구분자

  id:    클라이언트 재연결 시 Last-Event-ID 헤더로 전송
  event: 이벤트 타입 (JavaScript addEventListener 키)
  data:  실제 데이터 (멀티라인 가능, data: 여러 번)
  retry: 재연결 대기 시간 (ms)
  \n:    메시지 종료

WebFlux SSE 내부 동작:
  Flux<ServerSentEvent<T>> 반환
  → HttpMessageWriter (SseHttpMessageWriter)
  → 각 onNext 신호 → SSE 형식으로 직렬화 → HTTP 청크로 전송
  → 클라이언트가 연결 끊으면 → Netty 연결 close 감지
  → Flux에 cancel 신호 → Flux 파이프라인 정리
```

### 2. SSE 구현 — ServerSentEvent 빌더

```
기본 SSE 엔드포인트:

  @GetMapping(value = "/prices/stream",
              produces = MediaType.TEXT_EVENT_STREAM_VALUE)
  public Flux<ServerSentEvent<StockPrice>> streamPrices() {
      return stockPriceService.getPriceFlux()  // Flux<StockPrice>
          .map(price -> ServerSentEvent.<StockPrice>builder()
              .id(String.valueOf(price.getSequence()))     // 재연결용 ID
              .event("price-update")                       // 이벤트 타입
              .data(price)                                 // 실제 데이터
              .comment("price at " + price.getTimestamp()) // 주석 (클라이언트 무시)
              .build()
          )
          .doOnCancel(() ->
              log.info("클라이언트 연결 해제: prices stream")
          );
  }

  // 단순 Flux<T>도 SSE로 전송 가능 (ServerSentEvent 빌더 없이)
  @GetMapping(value = "/events/simple",
              produces = MediaType.TEXT_EVENT_STREAM_VALUE)
  public Flux<StockPrice> simpleStream() {
      return stockPriceService.getPriceFlux();
      // data: {JSON} 형태로 자동 직렬화
      // event 타입, id 없음
  }

재연결 처리:
  클라이언트가 연결 끊기 후 재연결 시:
  GET /prices/stream
  Last-Event-ID: 42  ← 마지막 수신 이벤트 ID

  서버에서 Last-Event-ID 처리:
  @GetMapping(value = "/prices/stream", produces = TEXT_EVENT_STREAM_VALUE)
  public Flux<ServerSentEvent<StockPrice>> streamPrices(
          @RequestHeader(value = "Last-Event-ID", required = false)
          Long lastEventId) {
      Flux<StockPrice> prices = lastEventId != null
          ? stockPriceService.getPriceFluxFrom(lastEventId + 1)  // 이후부터
          : stockPriceService.getPriceFlux();                     // 처음부터
      return prices.map(price -> ServerSentEvent.<StockPrice>builder()
          .id(String.valueOf(price.getSequence()))
          .data(price)
          .build());
  }
```

### 3. WebSocket — 양방향 통신

```
WebSocket 프로토콜:
  HTTP 업그레이드로 시작 → ws:// (또는 wss://) 프로토콜
  양방향: 클라이언트 ↔ 서버
  Full-duplex: 동시 송수신 가능

WebFlux WebSocket 구현:

  @Component
  public class ChatWebSocketHandler implements WebSocketHandler {

      @Override
      public Mono<Void> handle(WebSocketSession session) {
          String sessionId = session.getId();
          log.info("WebSocket 연결: {}", sessionId);

          // Inbound: 클라이언트 → 서버
          Mono<Void> inbound = session.receive()
              .map(WebSocketMessage::getPayloadAsText)
              .doOnNext(msg -> {
                  log.info("수신 [{}]: {}", sessionId, msg);
                  chatService.broadcast(sessionId, msg);  // 다른 클라이언트에 브로드캐스트
              })
              .doOnComplete(() -> {
                  log.info("WebSocket 연결 종료: {}", sessionId);
                  chatService.leave(sessionId);
              })
              .then();

          // Outbound: 서버 → 클라이언트
          Mono<Void> outbound = session.send(
              chatService.getMessagesFor(sessionId)  // Flux<String>
                  .map(session::textMessage)
          );

          // inbound 또는 outbound 중 하나가 완료되면 세션 종료
          return Mono.zip(inbound, outbound).then();
      }
  }

  // 라우팅 설정
  @Configuration
  public class WebSocketConfig {
      @Bean
      public HandlerMapping webSocketMapping(ChatWebSocketHandler handler) {
          Map<String, WebSocketHandler> map = new HashMap<>();
          map.put("/ws/chat", handler);
          SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
          mapping.setUrlMap(map);
          mapping.setOrder(-1);
          return mapping;
      }

      @Bean
      public WebSocketHandlerAdapter handlerAdapter() {
          return new WebSocketHandlerAdapter();
      }
  }
```

### 4. SSE vs WebSocket — 프로토콜 비교

```
특성 비교:

특성              | SSE                    | WebSocket
─────────────────┼───────────────────────┼───────────────────────
방향              | 단방향 (서버→클라이언트) | 양방향
프로토콜          | HTTP/1.1 기반           | ws:// (HTTP 업그레이드)
재연결            | 자동 (브라우저 내장)     | 수동 구현 필요
프록시 호환성     | 좋음 (HTTP 기반)         | 일부 프록시 문제 가능
데이터 형식       | 텍스트 (UTF-8)           | 텍스트/바이너리
HTTP/2 지원      | 네이티브 지원             | HTTP/2 위에서 별도 처리
로드 밸런서 호환  | 좋음                     | 스티키 세션 필요 가능
구현 복잡도      | 단순                     | 복잡

선택 기준:
  SSE 선택:
    서버 → 클라이언트 단방향 (알림, 피드, 대시보드)
    기존 HTTP 인프라 활용 (프록시, LB)
    자동 재연결 필요
    텍스트 데이터 위주

  WebSocket 선택:
    양방향 실시간 통신 (채팅, 게임, 협업 도구)
    낮은 레이턴시 필요 (클라이언트도 자주 메시지 전송)
    바이너리 데이터 전송 필요
```

### 5. SSE 리소스 정리 — 연결 해제 감지

```
연결 해제 감지 방식:

1. Flux cancel 신호:
   클라이언트 연결 종료 → Netty 감지 → Flux에 cancel
   → doOnCancel() 실행

2. Heartbeat로 연결 확인:
   Flux.interval로 주기적 ping 이벤트 추가
   → 클라이언트 응답 없으면 write 실패 → Flux 종료

3. doFinally (모든 종료 케이스):
   doFinally(signalType -> {
       if (signalType == SignalType.CANCEL) {
           // 클라이언트 연결 해제
       } else if (signalType == SignalType.ON_ERROR) {
           // 오류로 스트림 종료
       } else if (signalType == SignalType.ON_COMPLETE) {
           // 정상 완료
       }
   })

실전 패턴:
  @GetMapping(value = "/events", produces = TEXT_EVENT_STREAM_VALUE)
  public Flux<ServerSentEvent<Event>> streamEvents(
          ServerWebExchange exchange) {

      String clientId = exchange.getRequest().getId();

      // Heartbeat 병합 (30초마다 comment 전송으로 연결 유지)
      Flux<ServerSentEvent<Event>> heartbeat = Flux
          .interval(Duration.ofSeconds(30))
          .map(i -> ServerSentEvent.<Event>builder()
              .comment("heartbeat")  // 데이터 없는 주석 (클라이언트 무시)
              .build());

      return Flux.merge(
              eventService.getEventStream(clientId)
                  .map(e -> ServerSentEvent.<Event>builder().data(e).build()),
              heartbeat
          )
          .doOnCancel(() -> {
              log.info("SSE 연결 해제: {}", clientId);
              eventService.removeSubscriber(clientId);
          })
          .doOnError(e -> log.error("SSE 오류: {}", clientId, e));
  }
```

---

## 💻 실전 코드

### 실험 1: 주식 시세 SSE

```java
@RestController
@RequestMapping("/api/stocks")
@RequiredArgsConstructor
public class StockController {

    private final StockPriceService stockPriceService;

    @GetMapping(value = "/{symbol}/stream",
                produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<StockPrice>> streamPrice(
            @PathVariable String symbol,
            @RequestHeader(value = "Last-Event-ID", required = false)
            Long lastEventId) {

        return stockPriceService.getPriceStream(symbol, lastEventId)
            .map(price -> ServerSentEvent.<StockPrice>builder()
                .id(String.valueOf(price.getTimestamp().toEpochMilli()))
                .event("price")
                .data(price)
                .build()
            )
            .mergeWith(heartbeatFlux())  // 연결 유지
            .doOnCancel(() ->
                log.info("{}번 시세 스트림 구독 취소", symbol)
            );
    }

    private Flux<ServerSentEvent<StockPrice>> heartbeatFlux() {
        return Flux.interval(Duration.ofSeconds(15))
            .map(i -> ServerSentEvent.<StockPrice>builder()
                .comment("keep-alive")
                .build());
    }
}
```

### 실험 2: 채팅 WebSocket

```java
@Component
@RequiredArgsConstructor
public class ChatWebSocketHandler implements WebSocketHandler {

    private final ChatService chatService;
    private final ObjectMapper objectMapper;

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        String userId = getUserId(session);

        // 이 사용자를 채팅방에 등록
        chatService.join(userId, session.getId());

        Mono<Void> inbound = session.receive()
            .map(msg -> parseMessage(msg.getPayloadAsText()))
            .flatMap(msg -> chatService.send(userId, msg))
            .onErrorContinue((e, msg) ->
                log.error("메시지 처리 실패: {}", msg, e))
            .then();

        Flux<WebSocketMessage> outboundMessages = chatService
            .getIncomingMessages(userId)
            .map(msg -> {
                try {
                    return session.textMessage(
                        objectMapper.writeValueAsString(msg));
                } catch (JsonProcessingException e) {
                    throw new RuntimeException(e);
                }
            });

        Mono<Void> outbound = session.send(outboundMessages);

        return Mono.zip(inbound, outbound)
            .doFinally(signal -> {
                chatService.leave(userId);
                log.info("채팅 세션 종료: userId={}, signal={}", userId, signal);
            })
            .then();
    }

    private String getUserId(WebSocketSession session) {
        return session.getHandshakeInfo().getHeaders()
            .getFirst("X-User-ID");
    }
}
```

### 실험 3: 배치 진행 상황 SSE

```java
@PostMapping(value = "/batch/start",
             produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<BatchProgress>> startBatch(
        @RequestBody BatchRequest request) {

    String batchId = UUID.randomUUID().toString();

    return batchService.execute(batchId, request)
        .map(progress -> ServerSentEvent.<BatchProgress>builder()
            .id(String.valueOf(progress.getStep()))
            .event(progress.isComplete() ? "complete" : "progress")
            .data(progress)
            .build()
        )
        .doOnComplete(() ->
            log.info("배치 완료: {}", batchId)
        );
}
```

---

## 📊 성능 비교

```
SSE vs WebSocket vs Long Polling 비교:

방식           | 연결 수        | 서버 부하     | 레이턴시  | 복잡도
──────────────┼──────────────┼────────────┼─────────┼────────
Polling(1초)  | N/초          | 높음         | ~500ms  | 낮음
Long Polling  | N (대기 중)    | 중간          | ~즉시   | 중간
SSE           | N (유지)       | 낮음          | ~즉시   | 낮음
WebSocket     | N (유지)       | 낮음          | ~즉시   | 높음

10,000 동시 사용자 기준:
  Polling(1초): 10,000 req/s 추가 발생 → 서버 부하 높음
  SSE/WebSocket: 10,000 열린 연결 → 서버 부하 낮음 (I/O 대기 위주)

SSE 연결당 서버 메모리:
  Netty 채널: ~수 KB
  Flux 구독 상태: ~수 백 B
  총 ~수 KB/연결 → 10,000 연결: 수십 MB

WebSocket 연결당:
  SSE와 유사 (양방향이라 약간 더 많음)
  배지 교환 등 추가 처리 비용 있음
```

---

## ⚖️ 트레이드오프

```
SSE 트레이드오프:
  장점: 간단한 구현, HTTP 기반(프록시 친화적), 자동 재연결
  단점: 단방향만 가능, 최대 동시 연결 수 (HTTP/1.1: 브라우저당 6개)
  HTTP/2: SSE 연결 수 제한 없음 (멀티플렉싱)

WebSocket 트레이드오프:
  장점: 양방향, 낮은 오버헤드(헤더 없음), 바이너리 지원
  단점: 프록시/LB 설정 복잡, 재연결 수동, 구현 복잡

Hot Publisher 공유 SSE:
  장점: 외부 소스 단 1개 연결로 모든 SSE 클라이언트 서빙
  단점: 구독자 없으면 소스 중단 (share() 특성)

연결 수명:
  SSE: HTTP Keep-Alive로 유지, 프록시 타임아웃 주의
  WebSocket: 명시적 ping/pong으로 유지 (60초 기본)
```

---

## 📌 핵심 정리

```
SSE와 WebSocket 핵심:

SSE:
  HTTP 기반, 단방향 (서버→클라이언트)
  Flux<ServerSentEvent<T>> 또는 Flux<T> 반환
  클라이언트 연결 해제 → Flux cancel → doOnCancel 정리
  재연결 시 Last-Event-ID 헤더로 이어서 수신

WebSocket:
  ws:// 프로토콜, 양방향
  WebSocketHandler.handle(session) → Mono<Void>
  session.receive(): 클라이언트 → 서버
  session.send(): 서버 → 클라이언트
  Mono.zip(inbound, outbound).then()으로 세션 관리

선택 기준:
  알림/피드/대시보드 → SSE (간단, HTTP 친화)
  채팅/게임/협업 → WebSocket (양방향)

리소스 정리:
  doOnCancel / doFinally 필수
  Heartbeat로 연결 상태 확인
```

---

## 🤔 생각해볼 문제

**Q1.** HTTP/2를 사용하는 서버에서 SSE를 사용하면 어떤 이점이 있나요?

<details>
<summary>해설 보기</summary>

HTTP/1.1에서 브라우저는 같은 도메인에 최대 6개의 동시 연결을 허용합니다. SSE 연결이 6개 중 하나를 차지하면 나머지 HTTP 요청을 위한 연결이 줄어듭니다.

**HTTP/2의 SSE 이점:**
- 멀티플렉싱: SSE 스트림이 HTTP/2 스트림 하나로 처리됨 → 연결 수 제한 없음
- 동시에 여러 SSE 스트림 구독 가능 (하나의 TCP 연결에서)
- 일반 HTTP 요청과 SSE가 동일 연결 공유

```javascript
// HTTP/1.1: 브라우저 연결 6개 중 1개 차지
const sse1 = new EventSource('/prices/stream');
const sse2 = new EventSource('/alerts/stream');
// 이미 2개 사용 → 남은 4개로 일반 요청

// HTTP/2: 하나의 TCP 연결에서 모든 스트림 처리
// 연결 제한 없음
```

서버에서 HTTP/2 활성화:
```yaml
server:
  http2:
    enabled: true
  ssl:
    enabled: true  # HTTP/2는 대부분 TLS 필요
```

</details>

---

**Q2.** WebSocket 연결에서 서버가 클라이언트에게 연결 종료를 알리려면 어떻게 하나요?

<details>
<summary>해설 보기</summary>

WebSocket 종료는 Close 프레임(opcode 8)으로 이루어집니다. WebFlux에서는 `WebSocketSession.close()`를 호출합니다.

```java
// 서버에서 클라이언트에게 연결 종료 요청
session.close(CloseStatus.NORMAL)
    .subscribe();

// CloseStatus 코드 종류:
// NORMAL (1000): 정상 종료
// GOING_AWAY (1001): 서버 종료
// POLICY_VIOLATION (1008): 정책 위반
// MESSAGE_TOO_BIG (1009): 메시지 크기 초과

// 서버에서 일정 시간 후 자동 종료
session.receive()
    .timeout(Duration.ofMinutes(30))  // 30분 동안 메시지 없으면
    .onErrorResume(TimeoutException.class, e ->
        session.close(CloseStatus.GOING_AWAY).then(Flux.empty())
    )
    .then();
```

클라이언트가 보낸 Close 프레임은 `session.receive()`가 `onComplete`로 처리합니다. 이후 `session.send()`도 자동으로 완료됩니다.

</details>

---

**Q3.** SSE를 Nginx 뒤에서 운영할 때 주의할 설정은 무엇인가요?

<details>
<summary>해설 보기</summary>

Nginx는 기본적으로 응답을 버퍼링합니다. SSE는 실시간 스트리밍이므로 버퍼링을 비활성화해야 합니다.

```nginx
location /api/events {
    proxy_pass http://backend;
    proxy_http_version 1.1;  # HTTP/1.1 필수 (chunked transfer)

    # 버퍼링 비활성화 (SSE 핵심 설정)
    proxy_buffering off;
    proxy_cache off;

    # Keep-Alive 유지
    proxy_set_header Connection '';

    # SSE 연결 타임아웃 (기본 60초 → 연장)
    proxy_read_timeout 3600s;  # 1시간
    proxy_send_timeout 3600s;

    # SSE 헤더 전달
    proxy_set_header Cache-Control 'no-cache';
    add_header X-Accel-Buffering no;  # Nginx X-Accel-Buffering 비활성화
}
```

`proxy_buffering off`가 없으면 Nginx가 SSE 이벤트를 버퍼에 쌓다가 한꺼번에 클라이언트에 전송합니다. 실시간성이 깨지고 클라이언트는 이벤트를 묶음으로 받게 됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: WebClient 고급 패턴](./04-webclient-advanced-patterns.md)** | **[홈으로 🏠](../README.md)** | **[다음: 요청/응답 처리 ➡️](./06-request-response-handling.md)**

</div>
