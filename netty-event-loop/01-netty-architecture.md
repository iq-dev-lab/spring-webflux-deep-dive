# Netty 아키텍처 완전 분해 — Boss/Worker EventLoopGroup

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Boss EventLoopGroup`과 `Worker EventLoopGroup`은 각각 어떤 역할을 담당하는가?
- 클라이언트 연결 요청이 `accept`되어 데이터 처리까지 이어지는 경로는 어떻게 되는가?
- `Channel`과 `ChannelPipeline`은 어떤 관계인가?
- Spring WebFlux는 Netty를 어떻게 내장 서버로 사용하는가?
- Netty의 스레드 모델이 Tomcat의 스레드 모델과 어떻게 다른가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

WebFlux 애플리케이션의 성능 한계는 대부분 Netty 설정에서 비롯됩니다. EventLoopGroup 크기, 블로킹 코드의 위치, 연결 파이프라인의 구조를 이해하지 못하면 "왜 WebFlux인데 요청이 느리지?"라는 현상의 원인을 찾을 수 없습니다. Netty 내부 구조를 알면 로그의 스레드 이름(`reactor-http-epoll-N`)이 무엇을 의미하는지, 튜닝 포인트는 어디인지 즉시 판단할 수 있습니다.

---

## 😱 흔한 실수 (Before — Netty 구조를 모를 때)

```
실수 1: Boss와 Worker EventLoopGroup을 같은 것으로 혼동

  NioEventLoopGroup group = new NioEventLoopGroup(8);
  ServerBootstrap.group(group, group);  // Boss = Worker = 동일 그룹
  // → accept() 처리와 I/O 처리가 동일 스레드 풀 공유
  // → 대량 연결 시 accept가 지연되어 I/O 처리도 지연

실수 2: Worker 스레드 수를 너무 많이 설정

  // 8코어 서버에서
  WorkerEventLoopGroup = new NioEventLoopGroup(32);  // 과도한 설정
  // → 스레드 수 >> CPU 코어 수 → 컨텍스트 스위치 오버헤드
  // → 권장: CPU 코어 수 × 2 (I/O 대기 고려)

실수 3: Netty가 WebFlux와 무관하다고 생각

  // WebFlux 애플리케이션 로그:
  // reactor-http-epoll-1 → 이 스레드가 Netty Worker EventLoop 스레드
  // 여기서 블로킹 코드 실행 = 모든 연결 지연
```

---

## ✨ 올바른 접근 (After — Netty 구조 이해)

```
Netty 스레드 모델 요약:

Boss Group (1~2개 스레드):
  역할: TCP 연결 수락 (accept 시스템 콜)
  accept() 후 채널을 Worker로 등록하고 끝

Worker Group (CPU 코어 × 2개 스레드, 기본):
  역할: 등록된 채널의 I/O 처리
  각 스레드(EventLoop)가 여러 채널 담당 (비율은 아래 참고)
  WebFlux 핸들러 코드가 실행되는 스레드

Spring WebFlux 연동:
  WebFlux → ReactorHttpHandlerAdapter → Netty ChannelHandler
  Netty가 HTTP 파싱 → WebFlux 라우팅/핸들러 실행
  WebFlux는 Netty 위에서 동작하는 레이어
```

---

## 🔬 내부 동작 원리

### 1. Netty 전체 아키텍처 — 계층 구조

```
Spring WebFlux 요청 처리 계층:

클라이언트
    ↓ TCP 연결 요청
[Boss EventLoopGroup]
    NioEventLoop-0 (Boss)
    → ServerSocketChannel.accept() 시스템 콜
    → 새 SocketChannel 생성
    → Worker EventLoopGroup에 채널 등록
    ↓
[Worker EventLoopGroup]
    NioEventLoop-0: Channel-A, Channel-B, Channel-C, ...
    NioEventLoop-1: Channel-D, Channel-E, Channel-F, ...
    NioEventLoop-2: Channel-G, Channel-H, Channel-I, ...
    ...
    각 EventLoop = 단일 스레드 + 담당 채널들
    ↓
[Channel Pipeline per Channel]
    HttpRequestDecoder
    HttpObjectAggregator
    ReactorBridgeHandler    ← WebFlux 연결 지점
    HttpResponseEncoder
    ↓
[Spring WebFlux]
    DispatcherHandler
    RouterFunction / @Controller
    Handler → Mono<ServerResponse>
    ↓
다시 Channel Pipeline (Outbound) → TCP Write → 클라이언트
```

### 2. Boss EventLoopGroup 상세

```
Boss EventLoopGroup:
  스레드 수: 1개 (기본, 필요 시 2개)
  역할:
    1. ServerSocketChannel 감시 (accept 이벤트)
    2. accept() 시스템 콜로 새 SocketChannel 수락
    3. 새 채널을 Worker EventLoopGroup에 등록
    4. 자신은 다시 accept 이벤트 대기

Boss EventLoop 루프 (의사 코드):
  while (running) {
      int readyChannels = selector.select(timeout);
      if (readyChannels > 0) {
          Iterator<SelectionKey> keys = selector.selectedKeys().iterator();
          while (keys.hasNext()) {
              SelectionKey key = keys.next();
              if (key.isAcceptable()) {
                  SocketChannel client = serverChannel.accept();
                  workerGroup.register(client);  // Worker에 등록
              }
              keys.remove();
          }
      }
      runAllTasks();  // 태스크 큐 처리
  }

Boss가 처리하는 시간:
  accept() → Worker 등록: ~수 μs
  → Boss 스레드는 대부분의 시간을 selector.select()에서 대기
  → 1개 스레드로 수만 TPS의 연결 수락 처리 가능
```

### 3. Worker EventLoopGroup 상세

```
Worker EventLoopGroup:
  스레드 수: Runtime.getRuntime().availableProcessors() × 2 (기본)
  8코어 서버 → 16개 Worker 스레드

  각 Worker EventLoop (NioEventLoop):
    단일 스레드
    Selector로 여러 채널 감시
    I/O 이벤트 처리 + 태스크 큐 처리

Worker EventLoop 루프:
  while (running) {
      // I/O 이벤트 처리
      int ready = selector.select(timeout);
      if (ready > 0) {
          for (SelectionKey key : selector.selectedKeys()) {
              if (key.isReadable()) {
                  channel.read(buffer);
                  pipeline.fireChannelRead(buffer);  // Pipeline에 전달
              }
              if (key.isWritable()) {
                  channel.write(outboundBuffer);
              }
          }
      }
      // 태스크 큐 처리 (schedule(), execute() 등)
      runAllTasks(maxTime);
  }

채널-EventLoop 바인딩:
  각 채널은 Worker EventLoopGroup에 등록 시 하나의 EventLoop에 고정
  이후 해당 채널의 모든 처리는 동일 EventLoop가 담당
  → 채널 내 순서 보장, lock 없이 Thread-safe

WebFlux 로그에서 확인:
  [reactor-http-epoll-1] - EventLoop Worker 스레드 1번
  [reactor-http-epoll-2] - EventLoop Worker 스레드 2번
  → 각 요청은 특정 Worker 스레드에 고정
```

### 4. Channel과 ChannelPipeline

```
Channel:
  네트워크 연결 추상화 (TCP SocketChannel)
  읽기/쓰기 버퍼 관리
  EventLoop에 귀속 (1:1 고정)

ChannelPipeline:
  Channel에 연결된 ChannelHandler 체인
  Inbound(읽기): 네트워크 → 애플리케이션 방향
  Outbound(쓰기): 애플리케이션 → 네트워크 방향

WebFlux 기본 Pipeline 구성:

  [네트워크]
      ↓ Inbound
  HttpRequestDecoder          (바이트 → HttpRequest 객체)
  HttpObjectAggregator        (청크 → 완전한 HttpRequest)
  ReactorBridgeHandler        (Netty HttpRequest → Reactor HTTP)
  WebFlux DispatcherHandler   (라우팅 → 핸들러 실행)
      ↓ Outbound (역방향)
  HttpResponseEncoder         (HttpResponse → 바이트)
  ChunkedWriteHandler         (청크 인코딩)
  [네트워크]

ReactorBridgeHandler:
  Netty와 Reactor/WebFlux 사이의 어댑터
  HttpRequest → ServerHttpRequest (WebFlux)
  Mono<Void> → Netty write 완료 신호
```

### 5. Spring WebFlux와 Netty 연동

```
WebFlux 자동 구성 (Spring Boot):

ReactiveWebServerFactory
  → NettyReactiveWebServerFactory (기본)
  → ReactorHttpServer 생성
  → Netty ServerBootstrap 구성

NettyReactiveWebServerFactory 기본 설정:
  Boss EventLoopGroup: 1개 스레드
  Worker EventLoopGroup: availableProcessors() × 2 스레드
  서버 소켓 옵션: SO_REUSEADDR, TCP_NODELAY 등

WebFlux 핸들러 연결:
  ReactorHttpHandlerAdapter:
    Netty ChannelHandler 구현
    HttpRequest → WebFlux ServerWebExchange 변환
    핸들러 실행 → Mono<Void> 결과 → Netty response write

커스터마이징:
  @Bean
  public NettyServerCustomizer nettyCustomizer() {
      return httpServer -> httpServer
          .option(ChannelOption.SO_BACKLOG, 1024)
          .childOption(ChannelOption.TCP_NODELAY, true)
          .wiretap(true);  // 요청/응답 로깅
  }
```

---

## 💻 실전 코드

### 실험 1: Netty 스레드 이름 확인

```java
@GetMapping("/thread-info")
public Mono<String> threadInfo() {
    return Mono.fromCallable(() -> {
        String threadName = Thread.currentThread().getName();
        // reactor-http-epoll-1 (Linux epoll)
        // reactor-http-nio-1   (Windows/macOS NIO)
        return "현재 스레드: " + threadName;
    });
}

// 출력 예:
// GET /thread-info → 현재 스레드: reactor-http-epoll-2
// 이 스레드가 Worker EventLoop 스레드
// 여기서 블로킹 코드 실행 시 epoll-2가 담당한 모든 채널 지연
```

### 실험 2: Worker EventLoopGroup 커스터마이징

```java
@Configuration
public class NettyConfig {

    @Bean
    public NettyServerCustomizer workerGroupCustomizer() {
        // I/O 집약적 서비스: 코어 수 × 2 (기본)
        // CPU 집약적 서비스: 코어 수와 동일하게
        int workerCount = Runtime.getRuntime().availableProcessors();

        return httpServer -> httpServer
            .runOn(LoopResources.create("my-http", 1, workerCount, true));
        // 파라미터: prefix, boss 수, worker 수, daemon 여부
    }
}
```

### 실험 3: 채널 파이프라인 커스터마이징

```java
@Configuration
public class NettyPipelineConfig {

    @Bean
    public NettyServerCustomizer pipelineCustomizer() {
        return httpServer -> httpServer
            .doOnChannelInit((observer, channel, remoteAddress) -> {
                // Channel Pipeline에 커스텀 핸들러 추가
                channel.pipeline().addFirst("readTimeout",
                    new ReadTimeoutHandler(30, TimeUnit.SECONDS));
            })
            .childOption(ChannelOption.SO_KEEPALIVE, true)
            .option(ChannelOption.SO_BACKLOG, 512);
    }
}
```

---

## 📊 성능 비교

```
Tomcat vs Netty 스레드 모델:

Tomcat (Thread-per-Request):
  기본 스레드 수: 200
  각 스레드가 1개 요청 전담
  I/O 대기 중 스레드 블로킹
  1000 동시 연결 → 1000 스레드 필요

Netty (Event-Driven):
  Worker 스레드: 8코어 → 16개 (기본)
  각 스레드가 수천 채널 담당
  I/O 대기 중 다른 채널 처리
  1000 동시 연결 → 16개 스레드로 처리 가능

메모리 비교 (1만 동시 연결 기준):
  Tomcat: 1만 스레드 × 512KB = 5GB 스택 메모리
  Netty:  16 스레드 × 512KB = 8MB 스택 메모리
  → Netty가 약 600배 적은 스레드 스택 사용

처리량 비교 (논블로킹 작업):
  Tomcat: 200 동시 처리 (스레드 풀 한계)
  Netty:  CPU 코어 수가 한계 (스레드 전환 최소)
  → 논블로킹 I/O 집약 시나리오에서 Netty가 우위
```

---

## ⚖️ 트레이드오프

```
Netty 아키텍처 트레이드오프:

장점:
  ① 적은 스레드로 많은 연결 처리
  ② 컨텍스트 스위치 최소화
  ③ 메모리 효율 (스레드 수)

단점:
  ① EventLoop 블로킹 시 전체 채널 영향 (설계 제약)
  ② 블로킹 코드와 혼용 어려움
  ③ 디버깅 복잡 (동일 스레드에서 여러 채널 처리)

Worker 스레드 수 설정 트레이드오프:
  너무 적음: CPU 활용률 저하, 처리량 감소
  너무 많음: 컨텍스트 스위치 오버헤드 증가
  권장: CPU 코어 수 × 2 (I/O 대기 고려)
  I/O가 거의 없는 경우: CPU 코어 수와 동일하게

Boss 스레드 수:
  일반적으로 1개 (수만 TPS도 1개로 충분)
  멀티 포트 서버나 극한 연결 수락 필요 시 2개
```

---

## 📌 핵심 정리

```
Netty 아키텍처 핵심:

Boss EventLoopGroup:
  TCP 연결 수락 전담 (accept 시스템 콜)
  수락 후 채널을 Worker에 등록하고 끝
  스레드 1~2개로 충분

Worker EventLoopGroup:
  채널의 실제 I/O 처리 (읽기/쓰기)
  각 EventLoop = 단일 스레드 + 여러 채널 담당
  스레드 수: CPU 코어 × 2 (기본)

Channel + Pipeline:
  각 채널은 하나의 EventLoop에 고정
  Pipeline = 핸들러 체인 (Inbound/Outbound)
  WebFlux = Pipeline 끝에 연결된 Handler

로그에서 확인:
  reactor-http-epoll-N = Worker EventLoop N번 스레드
  여기서 블로킹 코드 = 모든 채널 지연
```

---

## 🤔 생각해볼 문제

**Q1.** Boss와 Worker를 동일한 `EventLoopGroup`으로 설정하면 어떤 문제가 발생하나요?

<details>
<summary>해설 보기</summary>

`ServerBootstrap.group(group, group)`처럼 Boss와 Worker에 동일한 그룹을 사용하면, accept 처리와 I/O 처리가 같은 스레드 풀을 공유합니다.

대량의 연결이 들어오는 순간, Worker 스레드들이 accept 처리에도 관여하게 되어 I/O 처리 스레드가 부족해질 수 있습니다. 반대로 I/O 처리가 무거울 때 새로운 연결 수락이 지연됩니다. 두 가지 서로 다른 성격의 작업(accept vs I/O)이 하나의 풀을 경쟁적으로 사용하므로 성능 예측이 어려워집니다.

WebFlux 기본 설정은 Boss와 Worker를 분리합니다. 직접 Netty를 사용할 때도 `group(bossGroup, workerGroup)`으로 명시적으로 분리하는 것이 권장됩니다.

</details>

---

**Q2.** WebFlux 로그에서 `reactor-http-epoll-1`과 `reactor-http-nio-1`의 차이는 무엇인가요?

<details>
<summary>해설 보기</summary>

OS별로 사용하는 I/O 멀티플렉싱 시스템 콜이 다릅니다.

- **Linux**: `epoll` 사용 → `reactor-http-epoll-N`
- **Windows / macOS**: `NIO Selector` 사용 → `reactor-http-nio-N`
- **macOS (네이티브)**: `kqueue` 사용 가능 (`netty-transport-native-kqueue` 의존성 추가 시)

`epoll`은 `NIO Selector`보다 대규모 연결에서 성능이 우수합니다. `NIO Selector`는 `O(N)` (등록된 채널 수), `epoll`은 `O(ready)` (준비된 이벤트 수)의 복잡도를 가집니다.

프로덕션 Linux 서버에서 `reactor-http-nio-N`이 보인다면 `netty-transport-native-epoll` 의존성이 누락된 것이므로 확인이 필요합니다.

</details>

---

**Q3.** `Channel`이 특정 `EventLoop`에 고정되어 있다는 것이 스레드 안전성에 어떤 이점을 주나요?

<details>
<summary>해설 보기</summary>

하나의 채널에 대한 모든 I/O 이벤트(읽기, 쓰기, 연결, 해제)가 항상 동일한 EventLoop 스레드에서 처리되므로, **채널 상태에 대한 동기화가 불필요합니다.**

예를 들어 채널의 수신 버퍼, 흐름 제어 상태, Pipeline의 핸들러 상태를 변경할 때 `synchronized` 없이도 Thread-safe합니다. 단일 스레드가 독점적으로 접근하기 때문입니다.

이것이 Netty가 높은 처리량에서도 lock 경합이 없는 핵심 이유입니다. 동일 EventLoop 내에서 여러 채널이 처리되지만, 채널 하나에 대한 처리는 항상 순차적으로 이루어집니다.

반면 `ChannelHandler`가 여러 채널에서 공유되는 경우(`@ChannelHandler.Sharable`)에는 핸들러 내부 상태에 대한 동기화가 별도로 필요합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: EventLoop 내부 동작 ➡️](./02-event-loop-internals.md)**

</div>
