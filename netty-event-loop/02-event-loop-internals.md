# EventLoop 내부 동작 — 하나의 스레드가 여러 Channel을 담당

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `EventLoop`의 내부 루프는 정확히 어떤 순서로 어떤 작업을 처리하는가?
- I/O 이벤트 처리와 태스크 큐 처리를 하나의 스레드에서 어떻게 병행하는가?
- `ioRatio`(I/O 처리 비율)는 무엇이고 어떻게 조정하는가?
- `EventLoop.execute()`와 `EventLoop.schedule()`은 어떻게 사용하는가?
- `Selector`는 어떻게 여러 채널의 I/O 이벤트를 하나의 스레드에서 관리하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"EventLoop 하나가 여러 채널을 처리한다"는 말을 들어도, 실제 루프의 작동 방식을 모르면 성능 이슈의 원인을 찾을 수 없습니다. 태스크 큐가 과부하될 때, I/O 처리와 비I/O 작업 사이 비율이 맞지 않을 때, 그리고 Netty가 CPU 100%를 사용하는 상황에서 어디서 병목이 생기는지 파악하려면 EventLoop 루프의 구조를 알아야 합니다.

---

## 😱 흔한 실수 (Before — EventLoop 동작을 모를 때)

```
실수 1: EventLoop에 무거운 태스크를 직접 제출

  channel.eventLoop().execute(() -> {
      heavyComputation();  // CPU 집약 작업 수백 ms
  });
  // → EventLoop의 태스크 큐에 적재
  // → 이 작업이 실행되는 동안 해당 EventLoop의 모든 I/O 중단

실수 2: schedule()로 너무 많은 타이머 등록

  for (int i = 0; i < 10000; i++) {
      channel.eventLoop().schedule(task, 1, TimeUnit.SECONDS);
  }
  // → 10,000개 scheduled 태스크 → 스케줄 큐 폭발
  // → 정작 I/O 처리 시간 부족

실수 3: EventLoop 스레드에서 EventLoop를 다시 기다림

  channel.eventLoop().execute(() -> {
      Future<?> f = channel.eventLoop().submit(anotherTask);
      f.get();  // 데드락! EventLoop가 자기 자신을 기다림
  });
```

---

## ✨ 올바른 접근 (After — EventLoop 특성 활용)

```
EventLoop 올바른 사용 원칙:

1. EventLoop에 제출할 태스크는 짧고 빠르게 (< 1ms 권장)
2. 무거운 작업은 별도 스레드 풀 (Schedulers.boundedElastic)에서
3. schedule()은 타임아웃, 재시도 등 경량 제어에만
4. EventLoop 스레드 여부 확인 후 분기:
   if (channel.eventLoop().inEventLoop()) {
       // 이미 EventLoop 스레드 — 직접 실행
   } else {
       // 다른 스레드 — submit
       channel.eventLoop().execute(task);
   }
```

---

## 🔬 내부 동작 원리

### 1. EventLoop 루프 전체 구조 (의사 코드)

```
NioEventLoop의 run() 메서드 (단순화):

  while (!isShutdown()) {

    // Phase 1: I/O 이벤트 처리
    int readyKeys = selector.select(selectTimeout);
    // select() = 이벤트가 올 때까지 대기 (timeout 있음)

    // Phase 2: 선택된 키(이벤트) 처리
    if (readyKeys > 0) {
        processSelectedKeys();
        // 읽기 이벤트 → pipeline.fireChannelRead()
        // 쓰기 이벤트 → 쓰기 버퍼 플러시
        // 연결 완료 → pipeline.fireChannelActive()
    }

    // Phase 3: 태스크 큐 처리
    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
    // ioTime: Phase 2에 걸린 시간
    // ioRatio: I/O vs 태스크 처리 시간 비율 (기본 50)
    // 태스크 처리 시간 = ioTime × (100-50)/50 = ioTime
    // → I/O 시간 = 태스크 시간 (1:1 비율)

    // 다시 Phase 1로 (무한 반복)
  }

핵심:
  단일 스레드에서 I/O 처리와 태스크 처리를 번갈아 수행
  select()에서 대기 → 이벤트 도착 → 처리 → 태스크 → 대기...
  스레드가 idle하지 않고 항상 뭔가 처리 중
```

### 2. Selector와 SelectionKey

```
Selector = I/O 이벤트 멀티플렉서

  Selector는 등록된 채널들을 감시
  select() = "어떤 채널이 I/O 준비됐나?" 블로킹 확인
           = epoll_wait() 시스템 콜 (Linux)

  SelectionKey = 채널 + 관심 이벤트 조합
    OP_READ:    읽기 가능 (데이터 도착)
    OP_WRITE:   쓰기 가능 (소켓 버퍼 여유)
    OP_ACCEPT:  연결 수락 가능 (Boss)
    OP_CONNECT: 연결 완료 (Client)

채널 등록 흐름:
  Boss: accept() → 새 SocketChannel 생성
     → workerGroup.next().register(channel, OP_READ)
     → Worker EventLoop의 Selector에 OP_READ로 등록

  이후:
    데이터 도착 → Selector가 OP_READ 이벤트 감지
    → processSelectedKey() → pipeline.fireChannelRead()

OP_WRITE 활용:
  SocketChannel 쓰기 버퍼가 가득 찬 경우:
  → write() 실패 (버퍼 꽉 참)
  → OP_WRITE 등록
  → 소켓 버퍼에 여유 생기면 OP_WRITE 이벤트
  → 쓰기 재시도
  → 완료 후 OP_WRITE 해제 (계속 등록하면 CPU 낭비)
```

### 3. 태스크 큐 (Task Queue)

```
EventLoop의 태스크 큐 종류:

1. taskQueue (일반 태스크):
   channel.eventLoop().execute(Runnable)
   → 큐에 적재 → 다음 루프 반복에서 처리

2. scheduledTaskQueue (예약 태스크):
   channel.eventLoop().schedule(task, delay, unit)
   → 우선순위 큐에 저장 → 실행 시점에 taskQueue로 이동

3. tailQueue (루프 마지막에 실행):
   Netty 내부 사용 (플러시 등)

태스크 큐 처리 과정:
  runAllTasks(timeoutNanos):
    while (태스크 있음 && 시간 남음) {
        Runnable task = taskQueue.poll();
        if (task != null) {
            safeExecute(task);
        }
        // 주기적으로 시간 체크
        if (++runCount % 64 == 0) {
            if (System.nanoTime() >= deadlineNanos) break;
        }
    }
  → 시간 초과 시 남은 태스크는 다음 루프에서 처리

EventLoop.inEventLoop() 체크:
  Netty 내부 코드에서 자주 사용하는 패턴:
  if (eventLoop.inEventLoop()) {
      doDirectly();   // 이미 EventLoop 스레드 → 즉시 실행
  } else {
      eventLoop.execute(this::doDirectly);  // 다른 스레드 → 큐에 등록
  }
  → 불필요한 큐 적재 없이 즉시 실행 최적화
```

### 4. ioRatio — I/O vs 태스크 비율 조정

```
ioRatio (기본: 50):
  EventLoop가 I/O 처리에 쓰는 시간 비율
  나머지 (100 - ioRatio)%가 태스크 처리에 할당

  ioRatio = 50:
    I/O 처리 100ms → 태스크 처리도 최대 100ms
    (I/O : 태스크 = 1:1)

  ioRatio = 80:
    I/O 처리 80ms → 태스크 처리 최대 20ms
    I/O 집약 서비스에 적합

  ioRatio = 20:
    I/O 처리 20ms → 태스크 처리 최대 80ms
    태스크가 많은 서비스에 적합 (WebSocket 메시지 처리 등)

커스터마이징:
  ((NioEventLoop) channel.eventLoop()).setIoRatio(80);

주의:
  ioRatio = 100:
    태스크 처리 시간 제한 없음 → runAllTasks()
    태스크가 많으면 I/O 처리 지연 가능

실무 권장:
  대부분의 WebFlux 서비스: 기본 50 유지
  WebSocket 브로드캐스트 (태스크 많음): 30~40으로 조정
```

### 5. Selector Wakeup — select() 깨우기

```
select()는 이벤트가 없으면 대기 (최대 timeout)

태스크 큐에 새 태스크 추가 시:
  taskQueue에 Runnable 추가
  → selector.wakeup() 호출 (select() 즉시 반환)
  → EventLoop가 태스크 처리

wakeup 과정:
  selector.wakeup()
  → Linux: eventfd(8) write로 epoll_wait() 깨움
  → 최소 비용의 시스템 콜

최적화 — 불필요한 wakeup 방지:
  이미 EventLoop 스레드라면 wakeup 불필요
  → inEventLoop() 체크 → true면 wakeup 생략

Netty의 "epoll bug" 해결:
  JDK의 Selector가 특정 조건에서 무한 spin (select가 0 반환)
  → CPU 100% 사용 버그
  → Netty: selectCount가 SELECTOR_AUTO_REBUILD_THRESHOLD(512) 초과 시
     새 Selector 생성 후 채널 재등록 → spin 탈출
```

---

## 💻 실전 코드

### 실험 1: EventLoop 태스크 제출과 스레드 확인

```java
@GetMapping("/eventloop-demo")
public Mono<String> eventLoopDemo(ServerWebExchange exchange) {
    return Mono.create(sink -> {
        // WebFlux는 이미 EventLoop 스레드에서 실행 중
        String current = Thread.currentThread().getName();
        log.info("핸들러 스레드: {}", current);  // reactor-http-epoll-N

        // EventLoop에 태스크 제출 (이미 EventLoop 스레드라면 즉시 실행)
        exchange.getResponse().bufferFactory();  // EventLoop 내에서 실행

        // 무거운 작업은 반드시 오프로딩
        Schedulers.boundedElastic().schedule(() -> {
            String result = heavyWork();
            sink.success(result);  // EventLoop에 결과 전달
        });
    });
}
```

### 실험 2: EventLoop 큐 과부하 감지

```java
// EventLoop 태스크 큐 크기 모니터링
@Scheduled(fixedDelay = 5000)
public void monitorEventLoops() {
    // NioEventLoopGroup에서 각 EventLoop 확인
    if (eventLoopGroup instanceof NioEventLoopGroup group) {
        for (EventExecutor executor : group) {
            if (executor instanceof SingleThreadEventExecutor loop) {
                int pendingTasks = loop.pendingTasks();
                if (pendingTasks > 100) {
                    log.warn("EventLoop 태스크 큐 과부하: {} tasks", pendingTasks);
                }
            }
        }
    }
}
```

### 실험 3: schedule()로 연결 타임아웃 구현

```java
// Netty에서 연결 타임아웃 구현 (ReadTimeoutHandler 내부 방식과 유사)
public class ConnectionTimeoutHandler extends ChannelInboundHandlerAdapter {
    private ScheduledFuture<?> timeoutTask;

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        // 채널 활성화 시 EventLoop에 타임아웃 예약
        timeoutTask = ctx.executor().schedule(
            () -> {
                log.warn("연결 타임아웃: {}", ctx.channel().remoteAddress());
                ctx.close();
            },
            30, TimeUnit.SECONDS
        );
        super.channelActive(ctx);
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        // 채널 닫힘 시 타임아웃 태스크 취소
        if (timeoutTask != null) {
            timeoutTask.cancel(false);
        }
        super.channelInactive(ctx);
    }
}
```

---

## 📊 성능 비교

```
EventLoop 태스크 처리 성능:

태스크 큐 적재 방식:
  같은 EventLoop 스레드 내 execute():
    wakeup 없음 → 큐에 추가만
    ~수 ns

  다른 스레드에서 execute():
    큐 추가 + selector.wakeup()
    ~수십 ns

  vs 일반 스레드 동기화:
    synchronized 블록 경합 시: ~수백 ns ~ μs
    → Netty 태스크 큐가 훨씬 빠름

select() 타임아웃 설정:
  너무 짧음 (< 1ms): CPU 낭비 (이벤트 없어도 자주 깨어남)
  너무 긺 (> 500ms): 태스크 큐 지연 (태스크가 와도 늦게 처리)
  Netty 기본: 1초 (이벤트 도착 시 즉시 wakeup으로 보완)

ioRatio별 처리 비율:
  ioRatio=50: I/O 50%, 태스크 50%
  ioRatio=80: I/O 80%, 태스크 20%
  → 서비스 특성에 맞게 조정
```

---

## ⚖️ 트레이드오프

```
EventLoop 단일 스레드 모델 트레이드오프:

장점:
  ① lock 없는 채널 처리 → 경합 없음, 빠름
  ② 순서 보장 → 같은 채널의 이벤트는 항상 순서대로
  ③ 컨텍스트 스위치 최소화

단점:
  ① 단일 스레드 장애점 → EventLoop 블로킹 시 전체 영향
  ② 공정성 없음 → 태스크 큐가 많으면 I/O 지연
  ③ 스케일 제한 → 코어 수를 초과하면 이점 없음

태스크 큐 vs 직접 실행:
  태스크 큐 장점: 스레드 안전 (어떤 스레드에서든 제출 가능)
  태스크 큐 단점: 지연 (즉시 실행 아님, 큐 순서 대기)

  → CPU 집약 작업을 태스크 큐에 넣으면 I/O 처리 지연
  → 짧은 콜백, 상태 변경, 타이머 등에만 사용 권장
```

---

## 📌 핵심 정리

```
EventLoop 내부 동작 핵심:

루프 3단계:
  1. selector.select() — I/O 이벤트 대기
  2. processSelectedKeys() — I/O 처리 (read/write)
  3. runAllTasks() — 태스크 큐 처리

ioRatio:
  I/O 시간 : 태스크 시간 비율 (기본 50:50)
  I/O 집약 서비스는 ioRatio 높이기

Selector:
  epoll (Linux), NIO Selector (기타)
  여러 채널을 단일 스레드로 감시
  select()에서 대기, wakeup으로 즉시 깨우기

태스크 큐 사용 원칙:
  짧고 빠른 태스크만 제출 (< 1ms)
  무거운 작업은 boundedElastic 오프로딩
  inEventLoop() 확인 후 분기
```

---

## 🤔 생각해볼 문제

**Q1.** `EventLoop`가 `select()`에서 대기 중인데 새로운 태스크가 큐에 추가되면 즉시 처리되나요?

<details>
<summary>해설 보기</summary>

즉시 처리됩니다. `execute(Runnable)`이 호출되면 내부적으로 `selector.wakeup()`이 호출됩니다.

```
execute(task) 내부:
  1. taskQueue.offer(task)  — 큐에 태스크 추가
  2. if (!inEventLoop() && wakenUp.compareAndSet(false, true))
       selector.wakeup()    — select()를 즉시 깨움
  3. EventLoop가 select()에서 반환 → runAllTasks() 실행
```

`inEventLoop()`가 true면 이미 EventLoop 스레드에서 실행 중이므로 select()가 대기 중일 수 없습니다. 따라서 `wakeup()`이 불필요하고 생략됩니다.

</details>

---

**Q2.** Netty의 "epoll 버그"란 무엇이고 어떻게 해결했나요?

<details>
<summary>해설 보기</summary>

JDK의 `Selector`에는 특정 조건에서 `select()`가 이벤트 없이 0을 반환하는 버그가 있습니다. 이 경우 EventLoop가 `select(0) → 처리할 것 없음 → select(0) → ...`를 무한 반복하며 CPU를 100% 소모합니다.

Netty의 해결책:
```
select() 연속 호출 횟수 카운트
→ SELECTOR_AUTO_REBUILD_THRESHOLD (기본: 512) 초과 시
→ 새로운 Selector 생성
→ 기존 채널을 새 Selector에 재등록
→ 기존 Selector 닫기
→ spin 탈출
```

이 자가 복구 메커니즘 덕분에 epoll 버그가 있는 JDK 버전에서도 Netty가 안정적으로 동작합니다. `netty-transport-native-epoll`을 사용하면 JDK Selector 대신 JNI 기반 epoll을 직접 사용하므로 이 버그를 근본적으로 회피합니다.

</details>

---

**Q3.** `EventLoop.schedule()`과 `Schedulers.boundedElastic()`의 지연 실행의 차이는 무엇인가요?

<details>
<summary>해설 보기</summary>

**`EventLoop.schedule(task, 1, TimeUnit.SECONDS)`:**
- EventLoop의 `scheduledTaskQueue`에 등록
- 1초 후 해당 EventLoop 스레드에서 실행
- 경량 태스크에 적합 (채널 타임아웃, 재시도 등)
- 실행 중 블로킹 시 EventLoop 전체 영향

**`Schedulers.boundedElastic().schedule(task, 1, TimeUnit.SECONDS)`:**
- Reactor의 boundedElastic 스레드 풀에서 실행
- 블로킹 코드 허용 (전용 스레드에서 실행)
- EventLoop와 독립적으로 동작
- 오버헤드: 스레드 풀 스케줄링 비용 추가

```java
// 경량 타임아웃 → EventLoop.schedule
ctx.executor().schedule(
    () -> ctx.close(),  // 채널 닫기: 가볍고 빠름
    30, TimeUnit.SECONDS
);

// 블로킹 작업 → boundedElastic
Schedulers.boundedElastic().schedule(
    () -> jdbcTemplate.query(...),  // 블로킹 DB 쿼리
    0, TimeUnit.SECONDS
);
```

</details>

---

<div align="center">

**[⬅️ 이전: Netty 아키텍처](./01-netty-architecture.md)** | **[홈으로 🏠](../README.md)** | **[다음: ChannelPipeline ➡️](./03-channel-pipeline-handler.md)**

</div>
