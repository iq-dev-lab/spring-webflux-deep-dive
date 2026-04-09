# C10K 문제 — 동시 연결 10,000개의 벽

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- C10K 문제란 무엇이고, 스레드 기반 모델이 왜 10,000 동시 연결에서 실패하는가?
- 스레드 10,000개를 만들었을 때 메모리와 CPU에서 실제로 무슨 일이 일어나는가?
- Apache의 prefork 방식이 Nginx의 이벤트 루프 방식에 밀린 이유는 무엇인가?
- C10K를 돌파한 이벤트 루프 방식의 핵심 통찰은 무엇인가?
- Spring WebFlux + Netty는 C10K 문제를 어떻게 해결하는가?

---

## 🔍 왜 이 개념이 WebFlux에서 중요한가

C10K 문제는 1999년 Dan Kegel이 제기한 문제입니다. "단일 서버에서 동시 연결 10,000개를 처리할 수 있는가?"라는 질문이었고, 당시 대부분의 서버는 수백 개의 동시 연결에서 한계를 보였습니다.

이 문제를 해결하는 과정에서 이벤트 루프와 논블로킹 I/O라는 패러다임이 탄생했고, Nginx, Node.js, Netty, Spring WebFlux로 이어지는 계보가 만들어졌습니다. WebFlux의 이벤트 루프가 "왜" 이런 구조인지 이해하려면, C10K 문제에서 스레드 기반 모델이 어떻게 실패했는지 정확히 알아야 합니다.

---

## 😱 흔한 실수 (Before — C10K를 스케일 아웃으로 해결하려 할 때)

```
상황: 동시 연결 5,000개를 처리해야 하는 실시간 알림 서비스
      WebSocket 연결이 지속적으로 유지되어야 함

잘못된 접근:
  "서버를 더 추가하자" → 10대 → 서버당 500 연결
  "각 서버에 스레드 500개" → 인스턴스당 500MB 스택 메모리
  → 연결 10,000개 → 서버 20대 필요 (비용 폭발)

근본 원인을 모를 때의 또 다른 실수:
  스레드 풀 크기를 무작정 늘림
  server.tomcat.threads.max=5000

결과:
  5,000 스레드 × 1MB = 5GB 스택 메모리
  JVM Heap 2GB + 스택 5GB = 7GB → 일반 인스턴스 한계
  GC 압박: 스레드 로컬 변수, ThreadLocal 객체 증가
  Context Switch: 5,000 스레드 스케줄링 → CPU 과부하
  → 설정은 5,000이지만 실제 안정적 처리는 500개도 어려움

잘못된 문제 정의:
  "서버가 부족하다" → X
  "스레드 기반 모델 자체가 대규모 동시 연결에 적합하지 않다" → O
```

---

## ✨ 올바른 접근 (After — 이벤트 루프 모델로의 전환)

```
문제 재정의:
  10,000 동시 연결 = 10,000 스레드 → 불가능
  10,000 동시 연결 = 소수의 이벤트 루프 스레드로 처리 → 가능

핵심 통찰:
  10,000 WebSocket 연결이 동시에 활성화되어 있어도
  특정 시점에 실제로 데이터를 주고받는 연결은 소수
  → 나머지는 연결만 유지, I/O 없음
  → 이벤트 루프: 실제 이벤트가 있는 연결만 처리

Spring WebFlux + Netty 적용:
  CPU 코어 8개 → EventLoop 스레드 8개
  10,000 WebSocket 연결 → 8개 스레드로 처리
  이벤트 없는 연결: 소켓 파일 디스크립터만 등록, 스레드 점유 없음
  이벤트 발생 → EventLoop가 해당 채널 핸들러 실행

비용 비교:
  Thread-per-Request: 10,000 스레드 × 1MB = 10GB (불가능)
  Event Loop: 8 스레드 × 1MB = 8MB + 채널 객체 × 10,000 (수백 MB, 가능)
```

---

## 🔬 내부 동작 원리

### 1. C10K 문제 수치 분석

```
스레드 기반 서버에서 10,000 동시 연결 시도:

메모리 계산:
  Java Thread 스택: 기본 512KB ~ 1MB
  10,000 스레드 × 1MB = 10GB (스택만)
  
  + JVM Heap (애플리케이션 객체): 2~4GB
  + OS Thread 구조체: 스레드당 수 KB → 수십 MB
  + ThreadLocal 데이터: 요청 컨텍스트, SecurityContext 등
  
  합계: 최소 12GB ~ 15GB
  → 일반적인 서버(16GB RAM)에서 OS + JVM + 기타 포함 시 불가능

CPU 계산:
  10,000 스레드가 모두 RUNNABLE 상태 진입 순간 (I/O 완료 동시 발생 시):
  CPU 코어 8개 → 코어당 1,250 스레드 스케줄링
  Context Switch 빈도: 수만 회/초
  Context Switch 비용: 수 μs ~ 수십 μs
  → CPU 시간의 상당 부분이 스케줄링에 소비

실험적 한계:
  스레드 기반 서버의 일반적 안정 한계:
    경험치: 스레드 500~2,000개 (서버 사양에 따라)
    2,000개 이상: Context Switch 오버헤드 급증
    5,000개 이상: 메모리 압박 + GC 지옥 진입
    10,000개: 일반 서버에서 사실상 불가능
```

### 2. Apache prefork vs Nginx event 모델 — 역사적 전환점

```
Apache prefork 모델 (C10K 이전 표준):

Master Process
  ├── Worker Process 1 (연결 1개 처리)
  ├── Worker Process 2 (연결 1개 처리)
  ├── Worker Process 3 (연결 1개 처리)
  ...
  └── Worker Process N

특징:
  - 프로세스별 완전히 분리된 메모리 공간
  - 프로세스 하나가 죽어도 다른 프로세스에 영향 없음
  - 안정성 높음 (당시 표준)

문제:
  - 프로세스는 스레드보다 더 무거움 (수 MB ~ 수십 MB)
  - 1,000 동시 연결 → 1,000 프로세스 → 수 GB 메모리
  - 1999년 기준: Apache는 수백 동시 연결에서 한계
  - prefork에서 동시 연결 1,000개 이상 → 서버 불안정

────────────────────────────────────────

Nginx event 모델 (2004년, C10K 해결 선언):

Master Process
  ├── Worker Process 1 (CPU 코어 담당)
  │     └── Event Loop: epoll로 수천 소켓 감시
  │           → 이벤트 있는 연결만 처리
  ├── Worker Process 2 (CPU 코어 담당)
  │     └── Event Loop
  ...
  └── Worker Process N (CPU 코어 수만큼)

특징:
  - Worker 수 = CPU 코어 수 (Context Switch 없음)
  - 각 Worker가 epoll로 수천 개 소켓 비동기 처리
  - 10,000 연결 → Worker 8개로 처리

결과:
  동일 하드웨어에서:
  Apache: ~1,000 동시 연결
  Nginx: ~10,000 ~ 100,000 동시 연결
  → Nginx가 정적 파일 서빙, 리버스 프록시 분야를 장악

Spring WebFlux + Netty:
  Nginx의 이벤트 루프 철학을 Java 애플리케이션 레이어에 적용
  → Java로 C10K를 해결
```

### 3. 파일 디스크립터(File Descriptor) 한계와 C10K

```
Unix 시스템에서 소켓 = 파일 디스크립터(fd):
  각 TCP 연결 → 소켓 파일 디스크립터 1개 소비

기본 fd 한계:
  프로세스당 최대 fd 수: 기본 1,024 (ulimit -n)
  10,000 연결 → 10,000 fd 필요
  → 기본 설정에서는 1,024 연결이 최대!

C10K 해결을 위한 OS 설정:
  # 프로세스당 최대 파일 디스크립터 수 증가
  ulimit -n 65536
  
  # 시스템 전체 최대 fd 수
  sysctl -w fs.file-max=1000000
  
  # TCP 소켓 관련 설정
  sysctl -w net.core.somaxconn=65535      # accept 큐 크기
  sysctl -w net.ipv4.tcp_max_syn_backlog=65535

select vs epoll — fd 수 한계:
  select(): fd_set의 비트맵 → 기본 FD_SETSIZE=1024
    → 1,024개 초과 연결 감시 불가 (구조적 한계)
  epoll: fd 수 제한 없음 (ulimit -n까지)
    → C10K 이상도 가능

Netty의 fd 설정:
  // EventLoopGroup 생성 시 자동으로 epoll 사용 (Linux)
  EventLoopGroup bossGroup = new EpollEventLoopGroup(1);
  EventLoopGroup workerGroup = new EpollEventLoopGroup();
  // ulimit -n 설정에 따라 수만 개 연결 가능
```

### 4. C10K 이후 — C100K, C1M 시대

```
C10K 해결 이후 목표가 높아진 배경:

C10K (10,000 동시 연결):
  2004년 Nginx로 해결
  핵심: epoll + 이벤트 루프

C100K (100,000 동시 연결):
  2013년경 실용적 달성
  핵심: 커널 파라미터 튜닝 + 이벤트 루프 최적화

C1M (1,000,000 동시 연결):
  2013년 "The Secret to 1 Million Connections" (Erlich Bachman style)
  핵심: 소켓당 메모리 최소화 + NUMA 인식 + 커널 바이패스(DPDK)

현재 실용적 관심사:
  WebSocket 실시간 서버: C100K 수준
  IoT 디바이스 연결: C1M 목표
  Spring WebFlux + Netty: C10K ~ C100K 범위에서 실용적

WebFlux의 현실적 한계:
  JVM 기반 → 커널 바이패스 불가 → C1M은 어려움
  GC 정지 → 대규모 연결 수에서 순간적 지연 가능
  실용 범위: 동시 연결 수만 개 수준
```

### 5. Spring WebFlux가 C10K를 해결하는 구조

```
Spring WebFlux + Netty 동시 연결 처리 구조:

10,000 WebSocket 클라이언트 연결 유지 상황:

┌───────────────────────────────────────────────────────────┐
│                      Netty Server                         │
│                                                           │
│  Boss EventLoopGroup (스레드 1~2개)                        │
│    └── accept() → 새 연결 수락 → Channel 생성              │
│         → Worker EventLoopGroup에 Channel 등록             │
│                                                           │
│  Worker EventLoopGroup (스레드 8개, CPU 코어 수)            │
│                                                           │
│  Worker Thread 1:                                         │
│    epoll 감시 중인 소켓: 1,250개 (10,000 / 8)              │
│    현재 이벤트 있는 소켓: 3개                                │
│    → 3개만 처리, 1,247개는 대기 (스레드 점유 없음)            │
│                                                           │
│  Worker Thread 2: 1,250개 소켓 감시, 이벤트 5개 처리         │
│  ...                                                      │
│  Worker Thread 8: 1,250개 소켓 감시, 이벤트 2개 처리         │
│                                                           │
│  이벤트 없는 연결 9,990개:                                   │
│    파일 디스크립터만 epoll에 등록                             │
│    스레드: 0개 점유                                          │
│    메모리: Channel 객체 (수 KB/연결) × 9,990 = 수십 MB       │
└───────────────────────────────────────────────────────────┘

Thread-per-Request로 동일 상황:
  10,000 스레드 × 1MB = 10GB → 불가능

WebFlux:
  8 스레드 × 1MB = 8MB + Channel 객체 수십 MB = 수백 MB → 가능
```

---

## 💻 실전 코드

### 실험 1: WebFlux 서버에서 다수 동시 연결 처리

```java
// WebFlux WebSocket 서버 — 10,000 연결 유지
@Configuration
public class WebSocketConfig {

    @Bean
    public HandlerMapping webSocketMapping() {
        Map<String, WebSocketHandler> map = new HashMap<>();
        map.put("/ws/realtime", new RealtimeWebSocketHandler());

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

@Component
public class RealtimeWebSocketHandler implements WebSocketHandler {

    private final Set<WebSocketSession> sessions = ConcurrentHashMap.newKeySet();

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        sessions.add(session);
        log.info("연결 수: {}, 스레드: {}", sessions.size(),
                 Thread.currentThread().getName());

        // 클라이언트가 보내는 메시지 수신 (이벤트 기반)
        return session.receive()
            .doOnNext(msg -> log.info("수신: {}", msg.getPayloadAsText()))
            .doFinally(signal -> {
                sessions.remove(session);
                log.info("연결 종료. 남은 연결: {}", sessions.size());
            })
            .then();
    }

    // 연결된 모든 클라이언트에게 브로드캐스트
    public Mono<Void> broadcast(String message) {
        return Flux.fromIterable(sessions)
            .flatMap(session -> session.send(
                Mono.just(session.textMessage(message))
            ))
            .then();
    }
}
```

```bash
# application.yml — Netty EventLoop 스레드 수 설정
spring:
  webflux:
    # 기본값: CPU 코어 수 × 2
    # Netty 이벤트 루프 스레드 수 조정 (필요 시)
    # server.netty.connection-timeout=10s
```

### 실험 2: 동시 연결 수 vs 스레드 수 비교

```bash
# WebFlux 서버 기동 후 WebSocket 클라이언트 1,000개 동시 연결
# k6 WebSocket 부하 테스트

cat > ws-load.js << 'EOF'
import ws from 'k6/ws';
import { sleep } from 'k6';

export let options = {
    vus: 1000,      // 동시 WebSocket 연결 1,000개
    duration: '60s',
};

export default function () {
    ws.connect('ws://localhost:8080/ws/realtime', {}, function(socket) {
        socket.on('open', () => {
            socket.send('Hello');
        });
        socket.on('message', (data) => {
            // 메시지 수신
        });
        sleep(50);  // 50초 연결 유지
        socket.close();
    });
}
EOF

k6 run ws-load.js

# 동시에 서버 스레드 수 모니터링:
watch -n 1 'jstack $(pgrep java) | grep "nioEventLoopGroup\|reactor-http" | wc -l'
# 결과: 연결 1,000개지만 스레드는 16개 (CPU 코어 8 × 2) 수준 유지

# Thread-per-Request(MVC) 동일 테스트:
watch -n 1 'jstack $(pgrep java) | grep "http-nio" | wc -l'
# 결과: 연결 1,000개 → 스레드 최소 200개 (maxThreads) 모두 점유됨
```

### 실험 3: 서버 메모리 사용량 비교

```bash
# WebFlux: 1,000 WebSocket 연결 시 메모리
jcmd $(pgrep java) VM.native_memory summary | grep -E "Thread|Total"
# Thread (Committed): ~20MB (연결 1,000개, 스레드 16개)

# 이론적 Thread-per-Request:
# 1,000 스레드 × 1MB = 1,000MB (Committed)
# → WebFlux 대비 50배 메모리 차이
```

---

## 📊 성능 비교

```
동시 연결 수에 따른 서버 모델별 특성:

동시 연결  | Thread-per-Req (메모리) | WebFlux (메모리) | 비율
──────────┼───────────────────────┼────────────────┼──────
100       | 100MB                 | ~8MB + α       | 12x
1,000     | 1GB                   | ~8MB + α       | 120x
10,000    | 10GB (불가)            | ~수백 MB        | -
100,000   | 불가                   | 가능 (튜닝 필요) | -

처리량 비교 (외부 API 200ms 응답, CPU 8코어):

동시 요청  | Thread 200개 서버(TPS) | WebFlux 서버(TPS)
──────────┼─────────────────────┼──────────────────
50        | ~250                | ~250   (차이 없음)
200       | ~1,000              | ~1,000 (차이 없음)
500       | ~600 (스레드 고갈)   | ~2,500 (이벤트 루프)
1,000     | ~200 (심각한 지연)   | ~5,000
5,000     | 불안정               | 수천 TPS 유지

결론:
  동시 연결이 적고 I/O 시간이 짧으면: 차이 없음
  동시 연결이 많고 I/O 집약적이면: WebFlux 압도적 우위
  CPU 집약적 작업: 두 모델 동일 (CPU 코어 수가 병목)
```

---

## ⚖️ 트레이드오프

```
C10K 해결을 위한 이벤트 루프 방식의 트레이드오프:

장점:
  ① 동시 연결 수 확장
     - 스레드 수가 아닌 이벤트 루프 + 파일 디스크립터 한계까지
  ② 메모리 효율
     - 연결당 스레드 없음 → 연결당 수 KB의 Channel 객체만
  ③ Context Switch 최소화
     - 소수의 EventLoop 스레드 → OS 스케줄러 부담 최소

단점:
  ① 블로킹 코드 단 하나가 전체 이벤트 루프 차단
     - EventLoop 스레드에서 Thread.sleep() → 해당 루프의 모든 연결 지연
     - 반드시 논블로킹 코드 또는 별도 스레드 오프로딩 필요
  ② 프로그래밍 복잡도 증가
     - 콜백/Reactive 파이프라인으로 코드 작성 필요
     - 스택 트레이스 파편화 → 디버깅 어려움
  ③ 블로킹 라이브러리 사용 불가
     - JDBC, Hibernate 등 블로킹 드라이버 직접 사용 시 장점 소멸
     - R2DBC로 전환 또는 boundedElastic 스레드 풀로 오프로딩 필요

실용적 판단:
  C10K 이상의 동시 연결이 필요한가? → WebFlux
  대부분의 요청이 빠르고 동시 연결이 수백 개 이하? → MVC로 충분
```

---

## 📌 핵심 정리

```
C10K 문제 핵심:

문제:
  스레드 기반 서버: 동시 연결 10,000개 = 스레드 10,000개 = 10GB 메모리
  → 일반 서버에서 불가능

핵심 통찰:
  10,000 연결이 있어도 특정 시점에 데이터를 주고받는 연결은 소수
  → "연결 유지"와 "스레드 점유"를 분리하면 됨
  → epoll로 이벤트 있는 연결만 처리 (이벤트 루프)

해결:
  Nginx(2004): 이벤트 루프 기반, C10K 해결
  Netty:       Java에서 동일 원리 구현
  Spring WebFlux: Netty 위에서 Reactive 프레임워크 제공

Spring WebFlux의 구조:
  스레드 8개(CPU 코어) × 연결 10,000개
  → 연결당 스레드 없음, Channel 객체만 (수 KB/연결)
  → epoll이 이벤트 발생 시 해당 채널 처리
  → C10K 이상 달성 가능
```

---

## 🤔 생각해볼 문제

**Q1.** Nginx가 Apache를 이긴 결정적 이유가 "이벤트 루프"라면, Apache도 이벤트 루프 방식(Event MPM)을 추가하지 않았나요? 그렇다면 지금도 Nginx가 유리한가요?

<details>
<summary>해설 보기</summary>

Apache는 이후 `mpm_event` 모듈을 추가하여 이벤트 방식을 지원했습니다. 그러나 Apache는 여전히 PHP 같은 블로킹 모듈(`mod_php`)과의 통합을 유지해야 했고, 아키텍처의 근본적 변경이 어려웠습니다.

Nginx는 처음부터 이벤트 루프로 설계되어 정적 파일 서빙, 리버스 프록시, 로드 밸런서 역할에 최적화되어 있습니다. Apache의 `mpm_event`는 Keep-Alive 연결을 이벤트 방식으로 처리하지만, 동적 컨텐츠 처리(PHP 등)에서는 여전히 스레드나 프로세스에 의존합니다.

현재 실무에서는 Apache와 Nginx 모두 각자의 강점이 있어 공존합니다. Spring WebFlux 관점에서는 둘 다 리버스 프록시로 사용 가능하지만, WebFlux 자체가 내장 Netty로 동작하므로 리버스 프록시 없이도 C10K 수준의 연결을 처리할 수 있습니다.

</details>

---

**Q2.** 이벤트 루프 기반 서버에서 CPU 집약적 연산(암호화, 이미지 처리 등)을 수행하면 어떤 문제가 발생하나요?

<details>
<summary>해설 보기</summary>

이벤트 루프의 단점이 정확히 이 지점입니다. CPU 집약적 연산이 EventLoop 스레드에서 실행되면, 그 스레드가 담당하는 **모든 연결의 처리가 연산이 끝날 때까지 차단**됩니다.

```java
// 잘못된 예시 — EventLoop 스레드에서 CPU 집약적 작업
@GetMapping("/encrypt")
public Mono<String> encrypt(@RequestBody String data) {
    // 이 코드가 EventLoop 스레드에서 실행됨
    // 암호화에 100ms 소요 → 이 EventLoop의 다른 연결 100ms 지연
    String encrypted = heavyEncryption(data);  // 위험!
    return Mono.just(encrypted);
}

// 올바른 예시 — boundedElastic 스레드 풀로 오프로딩
@GetMapping("/encrypt")
public Mono<String> encrypt(@RequestBody String data) {
    return Mono.fromCallable(() -> heavyEncryption(data))
               .subscribeOn(Schedulers.boundedElastic());
    // CPU 작업을 별도 스레드 풀에서 실행
    // EventLoop 스레드는 I/O 이벤트 처리 계속
}
```

이것이 WebFlux에서 **CPU 집약적 서비스와 I/O 집약적 서비스를 구분**해야 하는 이유입니다. CPU 집약적 서비스는 WebFlux보다 MVC에서 더 단순하게 처리할 수 있습니다(스케줄러 오프로딩 없이도 멀티코어 활용).

</details>

---

**Q3.** C10K 문제를 해결한 이벤트 루프 방식에서 `accept()` 시스템 콜은 어디서 담당하나요? Boss EventLoopGroup과 Worker EventLoopGroup의 역할 분리 이유는 무엇인가요?

<details>
<summary>해설 보기</summary>

Netty는 역할을 분리합니다.

**Boss EventLoopGroup** (보통 1~2개 스레드):
- `accept()` 시스템 콜로 새 연결 수락
- 새 연결마다 Channel 생성
- 생성된 Channel을 Worker EventLoopGroup에 등록

**Worker EventLoopGroup** (CPU 코어 수만큼):
- 등록된 Channel들의 epoll 이벤트 처리
- 실제 데이터 읽기/쓰기, 핸들러 실행

이렇게 분리하는 이유는 `accept()`와 I/O 처리를 동일 스레드에서 하면, 대량의 I/O 처리로 인해 새 연결 수락이 지연될 수 있기 때문입니다. Boss가 연결 수락에만 집중하므로, 기존 10,000 연결의 I/O가 바빠도 새 연결 수락은 영향받지 않습니다.

Spring WebFlux는 이 구조를 자동으로 설정합니다. `ReactiveWebServerFactoryAutoConfiguration`이 Netty `HttpServer`를 생성하고, Boss/Worker EventLoopGroup을 자동으로 구성합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Thread-per-Request 모델의 한계](./01-thread-per-request-model.md)** | **[홈으로 🏠](../README.md)** | **[다음: 논블로킹 I/O의 원리 — epoll과 이벤트 루프 ➡️](./03-nonblocking-io-epoll.md)**

</div>
