# ChannelPipeline과 Handler — 요청이 처리되는 경로

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ChannelPipeline`에서 `Inbound`와 `Outbound` 핸들러는 어떤 방향으로 실행되는가?
- HTTP 요청이 바이트에서 `ServerWebExchange`까지 변환되는 경로는 무엇인가?
- `ChannelHandlerContext`를 통해 파이프라인의 다음/이전 핸들러에게 신호를 어떻게 전달하는가?
- WebFlux의 `RouterFunction`과 Netty의 `ChannelHandler`는 어떻게 연결되는가?
- `@ChannelHandler.Sharable`은 무엇이고 왜 주의가 필요한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

WebFlux 요청이 컨트롤러에 도달하기까지 Netty의 여러 핸들러를 거칩니다. 요청 본문이 왜 특정 크기 이상 허용되지 않는지(MaxContentLength 설정), 커스텀 HTTP 핸들러를 어떻게 추가하는지, WebSocket 프로토콜 업그레이드는 어떻게 이루어지는지 — 이 모든 것이 ChannelPipeline 이해에서 출발합니다.

---

## 😱 흔한 실수 (Before — Pipeline 구조를 모를 때)

```
실수 1: Outbound 핸들러에 Inbound 로직 추가

  pipeline.addLast(new ChannelOutboundHandlerAdapter() {
      @Override
      public void read(ChannelHandlerContext ctx, ...) {
          // Outbound 핸들러에서 read 이벤트 처리 시도
          // 실제로는 Inbound 핸들러에서만 read 이벤트 수신
      }
  });

실수 2: @ChannelHandler.Sharable 없이 여러 채널에 동일 인스턴스 공유

  ChannelHandler handler = new StatefulHandler();  // 상태 있는 핸들러
  pipeline1.addLast(handler);
  pipeline2.addLast(handler);  // 예외! 이미 다른 파이프라인에 등록됨

실수 3: ctx.fireChannelRead() 빼먹기

  public void channelRead(ChannelHandlerContext ctx, Object msg) {
      processMessage(msg);
      // ctx.fireChannelRead(msg) 누락 → 다음 핸들러에 전달 안 됨
      // 파이프라인 중단 → 응답 없음
  }
```

---

## ✨ 올바른 접근 (After — Pipeline 동작 이해)

```
ChannelPipeline 핵심 원칙:

Inbound (읽기 방향):
  네트워크 → Head → Handler1 → Handler2 → ... → Tail
  각 핸들러에서 ctx.fireChannelRead(msg)로 다음으로 전달

Outbound (쓰기 방향):
  Tail → ... → Handler2 → Handler1 → Head → 네트워크
  각 핸들러에서 ctx.write(msg, promise)로 다음으로 전달
  (Inbound의 반대 방향)

항상 다음 핸들러에 전달:
  Inbound: ctx.fireChannelRead(msg)
  Outbound: ctx.write(msg, promise)
  예외 전파: ctx.fireExceptionCaught(cause)
  → 누락 시 파이프라인 중단
```

---

## 🔬 내부 동작 원리

### 1. ChannelPipeline 구조

```
Pipeline = 이중 연결 리스트 (HeadContext ↔ ... ↔ TailContext)

초기 상태:
  [HeadContext] ↔ [TailContext]

핸들러 추가 후:
  [HeadContext] ↔ [A] ↔ [B] ↔ [C] ↔ [TailContext]

HeadContext:
  Inbound/Outbound 모두 처리
  네트워크 I/O와 직접 연결 (실제 read/write 수행)
  첫 번째 Inbound 핸들러이자 마지막 Outbound 핸들러

TailContext:
  Inbound만 처리
  마지막 Inbound 핸들러
  처리되지 않은 메시지 자원 해제 (ReferenceCountUtil.release)
  예외 로깅

각 ChannelHandlerContext:
  이전/다음 핸들러 참조 (doubly linked)
  자신의 EventLoop 참조
  핸들러 실행을 적절한 스레드에서 처리
```

### 2. Inbound 이벤트 흐름

```
HTTP 요청 수신 시 Inbound 흐름:

TCP 데이터 도착 → HeadContext.channelRead(ByteBuf)
    ↓ ctx.fireChannelRead()
HttpRequestDecoder.channelRead(ByteBuf → HttpRequest/HttpContent)
    ↓ ctx.fireChannelRead()
HttpObjectAggregator.channelRead(HttpRequest+Content → FullHttpRequest)
    ↓ ctx.fireChannelRead()
ReactorBridgeHandler.channelRead(FullHttpRequest → WebFlux처리)
    ↓ (WebFlux 처리 완료 후 ctx.writeAndFlush()로 Outbound 시작)
TailContext (도달 시 미처리 메시지 릴리스)

Inbound 이벤트 종류:
  channelRegistered:  채널이 EventLoop에 등록됨
  channelUnregistered: 채널이 EventLoop에서 해제됨
  channelActive:      TCP 연결 확립
  channelInactive:    TCP 연결 종료
  channelRead:        데이터 수신
  channelReadComplete: 읽기 완료 (현재 읽기 사이클)
  exceptionCaught:    예외 발생
  userEventTriggered: 커스텀 이벤트 (예: WebSocket 핸드셰이크)
```

### 3. Outbound 이벤트 흐름

```
HTTP 응답 전송 시 Outbound 흐름:
(Inbound와 반대 방향)

ctx.writeAndFlush(response) 호출
    ↓ 반대 방향
ReactorBridgeHandler (WebFlux 응답 → Netty HttpResponse 변환)
    ↓ ctx.write()
HttpResponseEncoder.write(HttpResponse → ByteBuf)
    ↓ ctx.write()
HeadContext.write(ByteBuf → 실제 소켓 write)

Outbound 이벤트 종류:
  bind:         서버 소켓 바인딩
  connect:      연결 시작 (클라이언트 측)
  disconnect:   연결 끊기
  close:        채널 닫기
  read:         읽기 요청 (Selector OP_READ 등록)
  write:        데이터 쓰기 (버퍼에 추가)
  flush:        버퍼를 소켓에 실제 전송

write vs writeAndFlush:
  write(msg):         내부 버퍼에만 추가 (아직 전송 안 함)
  flush():            버퍼의 모든 데이터 전송
  writeAndFlush(msg): write + flush 한 번에
  → 성능: write 여러 번 + flush 한 번이 writeAndFlush 반복보다 효율적
```

### 4. WebFlux의 Netty 연결 지점

```
ReactorHttpHandlerAdapter (핵심 브릿지):

Netty ChannelHandler 구현:

  void channelRead(ChannelHandlerContext ctx, Object msg) {
      if (msg instanceof HttpRequest request) {
          // Netty HttpRequest → Reactor HTTP 래핑
          ReactorServerHttpRequest serverRequest =
              new ReactorServerHttpRequest(request, ctx);
          ReactorServerHttpResponse serverResponse =
              new ReactorServerHttpResponse(ctx);

          // WebFlux 핸들러 실행 (Mono<Void> 반환)
          Mono<Void> completion = httpHandler
              .handle(serverRequest, serverResponse);

          // 완료 시 채널에 신호
          completion.subscribe(
              null,
              err -> ctx.fireExceptionCaught(err),
              () -> ctx.writeAndFlush(Unpooled.EMPTY_BUFFER)
                       .addListener(ChannelFutureListener.CLOSE)
          );
      }
  }

WebFlux HttpHandler → DispatcherHandler:
  DispatcherHandler.handle(exchange)
  → HandlerMapping으로 핸들러 찾기 (RouterFunction / @RequestMapping)
  → HandlerAdapter로 핸들러 실행
  → Mono<HandlerResult> → HttpMessageWriter로 응답 직렬화
  → ServerHttpResponse.writeWith(Flux<DataBuffer>)
  → Netty ChannelHandlerContext.write(ByteBuf)
```

### 5. 커스텀 ChannelHandler 추가

```
Netty 파이프라인에 커스텀 핸들러 추가:

@Configuration
public class NettyPipelineConfig {

    @Bean
    public NettyServerCustomizer customPipeline() {
        return server -> server.doOnChannelInit(
            (observer, channel, remoteAddress) -> {
                ChannelPipeline pipeline = channel.pipeline();

                // 기존 핸들러 이름 확인 후 삽입 위치 결정
                // 기본 파이프라인:
                //   reactor.left.httpCodec
                //   reactor.left.httpTrafficHandler
                //   reactor.right.reactiveBridge

                // 요청 로깅 핸들러를 HTTP 핸들러 앞에 추가
                pipeline.addBefore(
                    "reactor.left.httpTrafficHandler",
                    "requestLogger",
                    new RequestLoggingHandler()
                );
            }
        );
    }
}

@ChannelHandler.Sharable  // 여러 채널에 동일 인스턴스 공유 시 필수
public class RequestLoggingHandler
        extends ChannelInboundHandlerAdapter {

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        log.info("새 연결: {}", ctx.channel().remoteAddress());
        ctx.fireChannelActive();  // 반드시 전달!
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        if (msg instanceof HttpRequest request) {
            log.info("요청: {} {}", request.method(), request.uri());
        }
        ctx.fireChannelRead(msg);  // 반드시 전달!
    }
}
```

---

## 💻 실전 코드

### 실험 1: 파이프라인 핸들러 목록 확인

```java
@GetMapping("/pipeline-info")
public Mono<List<String>> pipelineInfo(ServerWebExchange exchange) {
    return Mono.fromCallable(() -> {
        // 실제 프로덕션에서는 Netty 내부 채널 접근이 어려움
        // 아래는 개념 설명용 코드
        return List.of(
            "reactor.left.httpCodec",           // HTTP 코덱
            "reactor.left.httpTrafficHandler",  // HTTP 트래픽 핸들러
            "reactor.right.reactiveBridge"      // WebFlux 브릿지
        );
    });
}
```

### 실험 2: 최대 요청 크기 설정

```java
@Configuration
public class NettyMaxContentConfig {

    @Bean
    public NettyServerCustomizer maxContentLength() {
        return server -> server.httpRequestDecoder(
            spec -> spec
                .maxInitialLineLength(8192)  // 첫 줄 최대 길이 (URL 포함)
                .maxHeaderSize(16384)         // 헤더 최대 크기
                .maxChunkSize(8192)           // 청크 최대 크기
        );
        // HttpObjectAggregator의 maxContentLength도 별도 설정:
        // .maxContentLength(10 * 1024 * 1024)  // 10MB
    }
}
```

### 실험 3: WebSocket 핸드셰이크 파이프라인

```java
// WebSocket 업그레이드 시 파이프라인 변화
// 초기: HttpRequestDecoder → HttpObjectAggregator → WebFluxHandler
// 업그레이드 후:
//   HttpRequestDecoder → WebSocketFrameDecoder
//                      → WebSocketFrameEncoder
//                      → WebSocketHandshakeHandler
//                      → WebFlux WebSocket 핸들러

@Configuration
public class WebSocketConfig {

    @Bean
    public HandlerMapping webSocketHandlerMapping() {
        Map<String, WebSocketHandler> map = new HashMap<>();
        map.put("/ws/chat", new ChatWebSocketHandler());
        SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        mapping.setUrlMap(map);
        mapping.setOrder(-1);
        return mapping;
    }
}
```

---

## 📊 성능 비교

```
파이프라인 핸들러 오버헤드:

각 ctx.fireChannelRead() 호출 비용:
  핸들러 간 이벤트 전파: ~수십 ns
  EventLoop 스레드 체크: ~수 ns
  → 핸들러 10개라도 총 수백 ns — 무시 가능

핸들러 실행 스레드:
  모든 Inbound/Outbound 핸들러는 채널의 EventLoop 스레드에서 실행
  (별도 지정 없으면)
  → 스레드 전환 없음 → 빠름

@ChannelHandler.Sharable vs 인스턴스별:
  Sharable: 모든 채널이 동일 인스턴스 → 메모리 절약
            단, 내부 상태 없어야 함 (Thread-safe 필요)
  인스턴스별: 채널마다 새 인스턴스 → 상태 격리
              채널이 많으면 메모리 사용 증가

WebFlux 기본 파이프라인 핸들러:
  reactor.left.httpCodec:          HTTP 인코딩/디코딩
  reactor.left.httpTrafficHandler: 트래픽 관리, Keep-Alive, HTTP/2
  reactor.right.reactiveBridge:    WebFlux 연결
  → 3개 핸들러 × 수십 ns = 수백 ns 오버헤드 (무시 가능)
```

---

## ⚖️ 트레이드오프

```
커스텀 파이프라인 핸들러 추가 트레이드오프:

장점:
  HTTP 레벨에서의 커스터마이징 (인증, 압축, 로깅)
  WebFilter보다 낮은 레벨에서 동작 (헤더 직접 조작 등)
  프로토콜 수준 처리 (WebSocket, HTTP/2 등)

단점:
  Netty 내부 API 의존 → 버전 변경 시 호환성 위험
  WebFlux 추상화 우회 → 테스트 복잡
  @Sharable 핸들러: 상태 관리 주의 필요

권장:
  대부분의 경우: WebFilter로 처리 (더 안전하고 WebFlux와 통합)
  반드시 필요한 경우만: Netty ChannelHandler 직접 사용
  (예: 프로토콜 수준 처리, 극한 성능 최적화)
```

---

## 📌 핵심 정리

```
ChannelPipeline과 Handler 핵심:

Pipeline 구조:
  HeadContext ↔ [핸들러들] ↔ TailContext
  Inbound: Head → Tail (네트워크 → 앱)
  Outbound: Tail → Head (앱 → 네트워크)

항상 지켜야 할 규칙:
  Inbound: ctx.fireChannelRead(msg) 필수
  Outbound: ctx.write(msg, promise) 필수
  누락 시 파이프라인 중단

WebFlux 연결 지점:
  reactor.right.reactiveBridge → ReactorHttpHandlerAdapter
  Netty HttpRequest → ServerWebExchange 변환
  Mono<Void> 완료 → Netty 응답 flush

핸들러 공유:
  @ChannelHandler.Sharable 선언 시 여러 채널 공유 가능
  내부 상태 없어야 함 (stateless 설계)
```

---

## 🤔 생각해볼 문제

**Q1.** `ChannelPipeline.addLast()`와 `addBefore()`로 핸들러를 추가할 때 순서가 왜 중요한가요?

<details>
<summary>해설 보기</summary>

Inbound 이벤트는 파이프라인의 **추가된 순서대로** 처리되고, Outbound 이벤트는 **역순으로** 처리됩니다.

예를 들어 인증 핸들러를 HTTP 디코딩 핸들러 앞에 추가하면, 아직 HTTP 요청 객체가 만들어지기 전에 원시 바이트 수준에서 인증을 시도합니다. 반드시 `HttpRequestDecoder` 뒤에 인증 핸들러를 추가해야 합니다.

```
잘못된 순서:
  AuthHandler → HttpRequestDecoder → ...
  AuthHandler가 수신하는 것은 ByteBuf (HTTP 파싱 안 됨)

올바른 순서:
  HttpRequestDecoder → AuthHandler → ...
  AuthHandler가 수신하는 것은 HttpRequest 객체
```

`addBefore("reactor.left.httpTrafficHandler", "myHandler", handler)`처럼 명확한 기준점을 지정하는 것이 안전합니다.

</details>

---

**Q2.** `ChannelDuplexHandler`는 무엇이고 언제 사용하나요?

<details>
<summary>해설 보기</summary>

`ChannelDuplexHandler`는 `ChannelInboundHandlerAdapter`와 `ChannelOutboundHandlerAdapter`를 모두 상속한 클래스입니다. 하나의 핸들러에서 Inbound와 Outbound 이벤트를 모두 처리할 때 사용합니다.

```java
@ChannelHandler.Sharable
public class MetricsHandler extends ChannelDuplexHandler {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        requestCounter.increment();  // Inbound: 요청 수 증가
        ctx.fireChannelRead(msg);
    }

    @Override
    public void write(ChannelHandlerContext ctx, Object msg,
                      ChannelPromise promise) {
        responseCounter.increment();  // Outbound: 응답 수 증가
        ctx.write(msg, promise);
    }
}
```

요청/응답 쌍을 추적해야 하는 경우(레이턴시 측정, 트레이싱)에 적합합니다. 단, 읽기(Inbound)와 쓰기(Outbound) 방향이 다르므로 각 메서드에서 올바른 `fire*()` 또는 `ctx.write()` 전달을 빼먹지 않아야 합니다.

</details>

---

**Q3.** `ctx.channel().writeAndFlush()`와 `ctx.writeAndFlush()`의 차이는 무엇인가요?

<details>
<summary>해설 보기</summary>

**`ctx.writeAndFlush(msg)`:**
- 현재 `ChannelHandlerContext`에서 시작
- 파이프라인에서 현재 핸들러의 **이전(Outbound 방향)** 핸들러부터 실행
- 즉, 현재 핸들러를 포함한 나머지 Outbound 핸들러를 거침

**`ctx.channel().writeAndFlush(msg)`:**
- `Channel` 자체에서 시작 = `TailContext`에서 시작
- 파이프라인의 **모든** Outbound 핸들러를 순서대로 거침
- 파이프라인 전체를 통과 보장

```java
// 현재 핸들러 이후의 Outbound만 실행 (앞 Outbound 핸들러 건너뜀)
ctx.writeAndFlush(response);

// 파이프라인 처음(Tail)부터 모든 Outbound 핸들러 실행
ctx.channel().writeAndFlush(response);
```

일반적으로 핸들러 내에서는 `ctx.writeAndFlush()`를 사용합니다. `ctx.channel().writeAndFlush()`는 파이프라인의 모든 Outbound 핸들러를 거쳐야 할 때, 또는 핸들러 외부에서 채널에 직접 쓸 때 사용합니다.

</details>

---

<div align="center">

**[⬅️ 이전: EventLoop 내부 동작](./02-event-loop-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: EventLoop 블로킹 위험 ➡️](./04-blocking-in-eventloop.md)**

</div>
