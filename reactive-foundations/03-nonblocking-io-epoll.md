# 논블로킹 I/O의 원리 — epoll과 이벤트 루프

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 블로킹 I/O와 논블로킹 I/O는 OS 커널 레벨에서 어떻게 다른가?
- `epoll`은 수천 개의 소켓을 어떻게 단일 시스템 콜로 감시하는가?
- 이벤트 루프는 I/O 완료 이벤트를 받아 어떻게 처리 흐름을 이어가는가?
- Netty의 이벤트 루프가 Spring WebFlux 요청을 처리하는 전체 흐름은 무엇인가?
- `linux-for-backend-deep-dive`의 epoll과 WebFlux의 이벤트 루프는 어떻게 연결되는가?

---

## 🔍 왜 이 개념이 WebFlux에서 중요한가

WebFlux가 "논블로킹"이라는 말의 실체는 결국 OS의 `epoll`과 Netty의 이벤트 루프입니다. `WebClient`가 외부 API를 호출할 때 스레드가 블로킹되지 않는다고 하는데, 그러면 I/O가 완료됐을 때 누가 어떻게 통보를 받는가? `subscribe()` 후에 코드가 어떤 스레드에서 실행되는가? 이 질문에 답하려면 OS 레벨의 논블로킹 I/O 동작을 이해해야 합니다.

이 개념을 모르면 "`subscribeOn`과 `publishOn`을 왜 써야 하는지", "EventLoop 스레드에서 왜 블로킹 코드를 쓰면 안 되는지"를 표면적으로만 외우게 됩니다.

---

## 😱 흔한 실수 (Before — 논블로킹을 단순히 "빠른 것"으로 오해)

```
오해 1: "논블로킹 = 비동기 = 더 빠름"
  실제: I/O 대기 시간이 긴 경우에만 유리
        CPU 집약적 작업에서는 차이 없음

오해 2: "WebClient를 쓰면 자동으로 논블로킹"
  실제: WebClient 자체는 논블로킹이지만
        다음 코드는 블로킹을 만든다:
        
        // WebFlux 컨트롤러 안에서
        String result = webClient.get()
            .uri("/api")
            .retrieve()
            .bodyToMono(String.class)
            .block();  ← 이 순간 EventLoop 스레드 블로킹!

오해 3: "논블로킹이면 여러 스레드가 동시에 실행된다"
  실제: 이벤트 루프는 단일 스레드가 이벤트를 순서대로 처리
        동시성은 "동시에 실행"이 아닌 "I/O 대기 없이 전환"에서 옴

흔한 실수 코드:
  @GetMapping("/data")
  public Mono<String> getData() {
      // subscribeOn 없이 블로킹 DB 코드 실행 시도
      return Mono.fromCallable(() -> {
          return jdbcTemplate.queryForObject(...);  // 블로킹!
      });
      // → EventLoop 스레드에서 JDBC 블로킹 발생
      // → 해당 EventLoop가 담당하는 모든 연결 지연
  }
```

---

## ✨ 올바른 접근 (After — epoll 원리를 알고 난 설계)

```
논블로킹 I/O의 정확한 이해:
  "I/O 작업을 커널에 위임하고, 완료 이벤트가 왔을 때만 처리"
  → 스레드는 I/O를 기다리지 않음
  → 이벤트 루프가 다른 작업 처리 중 I/O 완료 이벤트 수신
  → 해당 채널의 핸들러 실행

올바른 코드 패턴:
  // 논블로킹 HTTP 클라이언트 — EventLoop 스레드 블로킹 없음
  return webClient.get()
      .uri("/api")
      .retrieve()
      .bodyToMono(String.class)  // I/O를 커널에 위임
      .map(result -> process(result));  // 완료 시 이 코드 실행

  // 블로킹 작업이 필요하면 — 별도 스레드 풀 사용
  return Mono.fromCallable(() -> jdbcTemplate.queryForObject(...))
      .subscribeOn(Schedulers.boundedElastic());
      // 블로킹 작업을 boundedElastic 스레드에서 실행
      // EventLoop 스레드는 다른 이벤트 처리 계속
```

---

## 🔬 내부 동작 원리

### 1. 블로킹 vs 논블로킹 I/O — 시스템 콜 레벨 비교

```
블로킹 I/O (기본 소켓):

  // 소켓 생성 (블로킹 모드 기본)
  int fd = socket(AF_INET, SOCK_STREAM, 0);
  connect(fd, ...);
  write(fd, request, len);
  
  ssize_t n = read(fd, buffer, sizeof(buffer));
  // ↑ 이 시스템 콜에서 OS가 현재 스레드를 WAITING 상태로 전환
  //   데이터가 올 때까지 스레드는 여기서 멈춤
  //   OS가 다른 스레드에 CPU 할당 (Context Switch)
  //   데이터 도착 → 인터럽트 → 이 스레드 RUNNABLE 복귀
  //   OS가 다시 이 스레드에 CPU 할당 (또 Context Switch)

논블로킹 I/O (O_NONBLOCK 플래그):

  int fd = socket(AF_INET, SOCK_STREAM, 0);
  // 소켓을 논블로킹 모드로 설정
  fcntl(fd, F_SETFL, O_NONBLOCK);
  
  connect(fd, ...);  // EINPROGRESS 반환 (연결 진행 중)
  write(fd, request, len);
  
  ssize_t n = read(fd, buffer, sizeof(buffer));
  // ↑ 데이터 없으면 즉시 EAGAIN 반환 (블로킹하지 않음)
  //   스레드는 계속 실행
  //   → "데이터 없음"을 알았으니 나중에 다시 확인해야 함
  //   → 이를 효율적으로 관리하는 것이 epoll의 역할

핵심 차이:
  블로킹:    데이터가 올 때까지 스레드 멈춤
  논블로킹: 즉시 반환, 데이터 없으면 EAGAIN
            → "언제 데이터가 오는가"를 epoll로 감시
```

### 2. epoll 동작 원리 상세

```
epoll 3단계 API:

1단계: epoll 인스턴스 생성 (프로세스당 한 번)
  int epfd = epoll_create1(0);
  // 커널 내부에 epoll 이벤트 테이블 생성
  // 내부 구조: red-black tree (등록 소켓) + 이벤트 큐 (준비된 소켓)

2단계: 소켓 등록 (소켓 생성 시마다)
  struct epoll_event ev;
  ev.events = EPOLLIN | EPOLLET;  // 읽기 이벤트, Edge Trigger
  ev.data.fd = client_fd;
  epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);
  // 커널: "이 fd에 데이터 도착하면 이벤트 큐에 추가해"
  // 비용: O(log N) — red-black tree 삽입

3단계: 이벤트 대기 (이벤트 루프 매 반복)
  struct epoll_event events[MAX_EVENTS];
  int nfds = epoll_wait(epfd, events, MAX_EVENTS, timeout);
  // "이벤트 큐에 뭔가 들어올 때까지 대기"
  // 이벤트 발생 시 events 배열에 준비된 fd 목록 반환
  // nfds = 준비된 fd 수
  
  for (int i = 0; i < nfds; i++) {
      handle_event(events[i].data.fd);  // 준비된 것만 처리
  }
  // 비용: O(이벤트 수) — 전체 등록 수와 무관

Level Trigger vs Edge Trigger:
  Level Trigger (기본):
    데이터가 있는 한 계속 이벤트 발생
    처리 못한 데이터가 남아있으면 다음 epoll_wait에서도 반환
  
  Edge Trigger (EPOLLET):
    상태 변화 시에만 이벤트 발생 (새 데이터 도착 순간에만)
    Netty 기본값: ET 사용 → 더 효율적, 구현 복잡도 높음
    → 한 번에 가능한 모든 데이터 읽어야 함 (EAGAIN까지)

epoll vs select 성능 차이 (10,000 소켓 중 10개 이벤트):
  select: 10,000개 fd 비트맵 전체 스캔 → O(10,000)
  epoll:  준비된 10개만 events 배열에서 처리 → O(10)
  → 1,000배 차이
```

### 3. Netty 이벤트 루프와 epoll 연결

```
Netty EventLoop 실행 구조:

NioEventLoop (Java NIO Selector 기반):
  while (true) {
      // 1. I/O 이벤트 대기 (epoll_wait 내부적으로 호출)
      int readyKeys = selector.select(timeoutMs);
      
      // 2. 준비된 채널 처리
      if (readyKeys > 0) {
          processSelectedKeys();
          // 각 준비된 채널의 ChannelHandler 체인 실행
      }
      
      // 3. 태스크 큐 처리 (Mono/Flux 연산자 등)
      runAllTasks(ioRatio);
      // ioRatio: I/O 처리 시간 vs 태스크 처리 시간 비율 (기본 50:50)
  }

EpollEventLoop (Linux 전용, Netty 네이티브):
  while (true) {
      // epoll_wait() 직접 호출 (JNI)
      int ready = epollWait();
      
      processReady(ready);
      runAllTasks();
  }
  // Java NIO Selector보다 더 효율적
  // EpollEventLoopGroup 사용 시 활성화

Spring WebFlux에서 활성화:
  // application.yml 또는 NettyReactiveWebServerFactory 커스터마이징으로
  // 기본: NioEventLoopGroup (Java NIO Selector)
  // 최적: EpollEventLoopGroup (Linux epoll 네이티브)

  // Netty 네이티브 라이브러리 사용 설정 (pom.xml)
  // <dependency>
  //   <groupId>io.netty</groupId>
  //   <artifactId>netty-transport-native-epoll</artifactId>
  //   <classifier>linux-x86_64</classifier>
  // </dependency>
```

### 4. WebClient HTTP 요청의 논블로킹 흐름 전체

```
WebClient.get().uri("/api").retrieve().bodyToMono(String.class) 실행 시:

1. subscribe() 호출 → 파이프라인 실행 시작
   (아직 HTTP 요청 안 보냄)

2. Netty Channel 획득 (Connection Pool에서)
   → 논블로킹 소켓 생성 또는 재사용

3. HTTP 요청 직렬화 → Channel에 write
   → Netty: write 버퍼에 저장
   → epoll이 소켓 "쓰기 가능" 이벤트 감지
   → 실제 write() 시스템 콜 실행
   → EventLoop 스레드는 즉시 다음 작업으로

4. EventLoop 계속 다른 이벤트 처리 중...
   (스레드는 응답을 기다리지 않음)

5. 서버 응답 도착
   → 커널: TCP 패킷 수신 → 소켓 수신 버퍼에 저장
   → epoll 이벤트 큐에 해당 fd 추가
   → epoll_wait() 반환

6. EventLoop: 해당 채널 "읽기 가능" 이벤트 감지
   → read() 시스템 콜로 응답 읽기
   → HTTP 응답 파싱

7. Reactor 파이프라인 재개
   → onNext(response) 신호 발생
   → map(), flatMap() 등 후속 연산자 실행
   → 최종 Subscriber(Controller return)에게 전달

핵심:
  4단계와 5단계 사이: 스레드는 다른 요청의 이벤트 처리 중
  → "스레드가 I/O를 기다리지 않는다"의 실체
  → epoll이 I/O 완료를 감지하고 EventLoop에 알림
```

### 5. 이벤트 루프에서 태스크 큐의 역할

```
EventLoop는 I/O 이벤트만 처리하는 것이 아님:

태스크 큐 (Task Queue):
  Mono/Flux 연산자 체인의 실행도 EventLoop의 태스크 큐에 등록됨

예시:
  webClient.get().uri("/api").retrieve()
      .bodyToMono(String.class)
      .map(s -> s.toUpperCase())   // 이 연산자도 태스크로 등록
      .subscribe(result -> ...);

EventLoop 실행 순서:
  루프 1: epoll_wait → API 서버 응답 수신 (I/O 이벤트)
            → HTTP 파싱 → onNext(body) 신호
            → map 연산자를 태스크 큐에 등록
  
  루프 2: 태스크 큐 처리
            → map(s -> s.toUpperCase()) 실행
            → subscribe 콜백 실행
  
  루프 3: epoll_wait → 다음 이벤트 대기...

ioRatio 설정 (기본 50):
  EventLoop가 I/O 처리에 쓰는 시간 : 태스크 처리에 쓰는 시간 = 50 : 50
  → 태스크가 많으면 I/O 처리 지연 가능
  → 튜닝: io.netty.ioRatio 시스템 프로퍼티로 조정
```

---

## 💻 실전 코드

### 실험 1: 논블로킹 WebClient vs 블로킹 RestTemplate 스레드 비교

```java
@RestController
@RequestMapping("/compare")
public class IoCompareController {

    private final WebClient webClient = WebClient.create("http://httpbin.org");
    private final RestTemplate restTemplate = new RestTemplate();

    // 논블로킹 — EventLoop 스레드에서 실행, I/O 대기 없음
    @GetMapping("/nonblocking")
    public Mono<String> nonBlocking() {
        log.info("시작 스레드: {}", Thread.currentThread().getName());
        // http-nio-... or reactor-http-... 스레드 (EventLoop)

        return webClient.get()
            .uri("/delay/1")  // 1초 지연 응답 (httpbin.org)
            .retrieve()
            .bodyToMono(String.class)
            .doOnNext(r -> log.info(
                "응답 스레드: {}", Thread.currentThread().getName()
            ));
        // → 동일 EventLoop 스레드 (reactor-http-epoll-n)
        // → 1초 동안 이 스레드는 다른 요청 처리 가능
    }

    // 블로킹 — Tomcat 스레드에서 실행, 1초 블로킹
    @GetMapping("/blocking")
    public String blocking() {
        log.info("시작 스레드: {}", Thread.currentThread().getName());
        // http-nio-... (Tomcat worker 스레드)
        
        String result = restTemplate.getForObject(
            "http://httpbin.org/delay/1", String.class
        );
        // → 동일 스레드 1초 블로킹
        log.info("응답 스레드: {}", Thread.currentThread().getName());
        return result;
    }
}
```

```bash
# 동시 100개 요청 발사 → 스레드 수 비교
curl -s "http://localhost:8080/compare/nonblocking" &
# 100개 발사 후:
jstack $(pgrep java) | grep -E "reactor-http|nioEventLoop" | wc -l
# 결과: ~16개 (EventLoop 스레드 수, CPU 코어 × 2)

curl -s "http://localhost:8080/compare/blocking" &
jstack $(pgrep java) | grep "http-nio" | wc -l
# 결과: ~100개 (요청당 1개 스레드)
```

### 실험 2: epoll 이벤트를 strace로 관찰

```bash
# WebFlux 애플리케이션 실행 후 Netty의 epoll 시스템 콜 추적
strace -f -e trace=epoll_create1,epoll_ctl,epoll_wait \
    -p $(pgrep java) 2>&1 | head -50

# 출력 예시:
# [pid 1234] epoll_create1(EPOLL_CLOEXEC) = 7         ← epoll 인스턴스 생성
# [pid 1234] epoll_ctl(7, EPOLL_CTL_ADD, 10, ...) = 0 ← 소켓 10 등록
# [pid 1234] epoll_wait(7, [{EPOLLIN, {u32=10, ...}}], 8192, -1) = 1
#                                                       ← 소켓 10에 이벤트 발생
# [pid 1234] epoll_ctl(7, EPOLL_CTL_ADD, 11, ...) = 0 ← 새 연결 소켓 11 등록
# [pid 1234] epoll_wait(7, [], 8192, 100) = 0          ← 타임아웃(이벤트 없음)
# [pid 1234] epoll_wait(7, [{EPOLLIN, ...}], 8192, -1) = 3 ← 3개 이벤트
```

### 실험 3: EventLoop 스레드에서 블로킹 코드 영향 측정

```java
// BlockHound로 EventLoop 블로킹 감지
@SpringBootApplication
public class WebFluxApp {
    public static void main(String[] args) {
        // 개발/테스트 환경에서 활성화
        BlockHound.install(builder ->
            builder.blockingMethodCallback(it ->
                log.error("블로킹 호출 감지: {}.{}",
                    it.getClassName(), it.getName())
            )
        );
        SpringApplication.run(WebFluxApp.class, args);
    }
}

// 블로킹 코드 테스트
@GetMapping("/bad")
public Mono<String> bad() {
    return Mono.fromCallable(() -> {
        Thread.sleep(100);  // → BlockHound가 즉시 예외 발생!
        return "done";
    });
    // BlockingOperationError: Blocking call! java.lang.Thread.sleep
}

// 올바른 코드
@GetMapping("/good")
public Mono<String> good() {
    return Mono.fromCallable(() -> {
        Thread.sleep(100);  // 같은 코드지만
        return "done";
    }).subscribeOn(Schedulers.boundedElastic());  // ← 별도 스레드에서 실행
    // → BlockHound 예외 없음 (boundedElastic은 블로킹 허용)
}
```

---

## 📊 성능 비교

```
I/O 모델별 시스템 콜 비용 비교 (10,000 소켓, 10개 이벤트):

모델           | 시스템 콜         | 비용             | 준비된 fd 탐색
──────────────┼──────────────────┼────────────────┼──────────────
select()      | 매번 전체 복사     | O(N) 복사+스캔   | 전체 fd 스캔
poll()        | 매번 전체 전달     | O(N) 전달+스캔   | 전체 fd 스캔
epoll_wait()  | 등록은 한 번      | O(이벤트 수)     | 준비된 것만

epoll 세부 비용:
  epoll_ctl(ADD): O(log N) — red-black tree 삽입
  epoll_wait():   O(1) 대기 + O(이벤트 수) 반환

WebClient vs RestTemplate 동시 100요청, 각 1초 응답:

지표                    | RestTemplate(MVC) | WebClient(WebFlux)
──────────────────────┼──────────────────┼──────────────────
처리에 사용된 스레드 수   | ~100개            | ~16개 (코어 × 2)
스레드 메모리(스택)      | ~100MB           | ~16MB
Context Switch/초       | 수천 회           | 수십 회
100번째 요청 응답시간    | ~1초 (스레드 있을 때) | ~1초 (동일)
500번째 요청 응답시간    | ~2.5초 (스레드 대기) | ~1초 (이벤트 루프)
```

---

## ⚖️ 트레이드오프

```
논블로킹 I/O + epoll 이벤트 루프:

장점:
  ① 스레드와 동시 I/O 처리 수를 분리
     - 스레드 16개로 수천 개 동시 I/O 처리 가능
  ② Context Switch 최소화
     - epoll_wait 하나가 수천 소켓 감시 → 스케줄러 개입 최소
  ③ 메모리 효율
     - 연결당 소켓 fd + Channel 객체만 필요 (스레드 스택 불필요)

단점:
  ① 콜백/Reactive 프로그래밍 모델 강제
     - 동기 코드처럼 작성 불가
     - 학습 곡선 높음
  ② EventLoop 스레드 보호 필수
     - 블로킹 코드 혼입 시 전체 영향
     - BlockHound로 감지하더라도 개발자 주의 필요
  ③ 디버깅 어려움
     - I/O 완료 시 어떤 스레드가 실행될지 예측 어려움
     - 스택 트레이스가 여러 스레드에 분산

Linux 환경 최적:
  epoll은 Linux 전용 (macOS: kqueue, Windows: IOCP)
  프로덕션 Linux에서 Netty EpollEventLoopGroup 사용 권장
  → Java NIO Selector 대비 성능 향상
```

---

## 📌 핵심 정리

```
논블로킹 I/O와 epoll 이벤트 루프 핵심:

블로킹 vs 논블로킹:
  블로킹:    read() 시스템 콜 → 데이터 없으면 스레드 WAITING
  논블로킹: read() 시스템 콜 → 데이터 없으면 EAGAIN 즉시 반환
             → "언제 데이터 오는지"를 epoll이 감시

epoll 3단계:
  epoll_create1() → 인스턴스 생성 (한 번)
  epoll_ctl(ADD)  → 소켓 등록 (소켓 생성 시마다)
  epoll_wait()    → 이벤트 대기 (이벤트 루프 매 반복)
                    준비된 fd만 반환 → O(이벤트 수)

Netty EventLoop + WebFlux:
  epoll_wait로 이벤트 대기
  → 소켓 읽기 가능 → HTTP 파싱 → onNext() 신호
  → Reactor 파이프라인 재개
  → 스레드 8~16개로 수천 동시 I/O 처리

주의:
  EventLoop 스레드에서 블로킹 코드 절대 금지
  블로킹 필요 시: Schedulers.boundedElastic()로 오프로딩
  BlockHound로 개발 중 감지
```

---

## 🤔 생각해볼 문제

**Q1.** Netty의 `NioEventLoop`(Java NIO Selector)와 `EpollEventLoop`(Linux 네이티브 epoll)의 내부 차이는 무엇이고, 실무에서 어떤 것을 선택해야 하나요?

<details>
<summary>해설 보기</summary>

**NioEventLoop (기본):**
- Java NIO `Selector`를 사용, 내부적으로 OS의 epoll/kqueue/select를 JVM이 추상화
- 플랫폼 독립적 (Linux, macOS, Windows 모두 동작)
- JVM 레이어를 거치므로 약간의 오버헤드 존재

**EpollEventLoop (Linux 네이티브):**
- JNI를 통해 `epoll_create1`, `epoll_ctl`, `epoll_wait` 직접 호출
- JVM 추상화 레이어 제거 → 더 낮은 지연, 더 높은 처리량
- `EPOLLET`(Edge Trigger) 기본 사용 → 더 효율적인 이벤트 처리
- `epoll_wait`에서 반환 시 추가적인 `wakeup` 메커니즘 최적화

선택 기준:
- **개발/테스트 환경**: `NioEventLoopGroup` (플랫폼 독립)
- **프로덕션 Linux**: `EpollEventLoopGroup` 권장

설정 방법:
```java
@Bean
public NettyReactiveWebServerFactory nettyFactory() {
    return new NettyReactiveWebServerFactory() {
        @Override
        protected HttpServer createHttpServer() {
            return super.createHttpServer()
                .runOn(new EpollEventLoopGroup());  // Linux 네이티브
        }
    };
}
```

실제 성능 차이는 워크로드에 따라 5~15% 수준으로 알려져 있으며, 초고부하 서비스에서 유의미한 차이가 납니다.

</details>

---

**Q2.** `publishOn(Schedulers.parallel())`과 `subscribeOn(Schedulers.boundedElastic())`은 언제 각각 사용해야 하나요? epoll 이벤트 루프와의 관계를 설명해주세요.

<details>
<summary>해설 보기</summary>

**`subscribeOn(Schedulers.boundedElastic())`:**
- 파이프라인이 **시작되는 스레드**를 변경
- 블로킹 코드를 EventLoop 스레드에서 분리할 때 사용
- 예: `Mono.fromCallable(() -> jdbcTemplate.query(...)). subscribeOn(Schedulers.boundedElastic())`
- subscribe() 호출 시 boundedElastic 스레드에서 블로킹 작업 실행

**`publishOn(Schedulers.parallel())`:**
- 해당 연산자 **이후** 코드가 실행되는 스레드를 변경
- CPU 집약적 작업을 parallel 스레드에서 실행할 때 사용

epoll 이벤트 루프 관점:
```
WebClient 요청
  → EventLoop 스레드에서 HTTP 전송
  → I/O 완료 이벤트 → 동일 EventLoop 스레드에서 .map() 실행

.subscribeOn(Schedulers.boundedElastic()) 추가 시:
  → 구독이 boundedElastic 스레드에서 시작
  → 블로킹 코드 실행 가능
  → 결과를 EventLoop에 전달

.publishOn(Schedulers.parallel()) 추가 시:
  → 이전: EventLoop 스레드
  → publishOn 이후: parallel 스레드 (ForkJoinPool 기반)
  → CPU 집약적 연산에 적합
```

실무 규칙: EventLoop 스레드에서 블로킹을 피하는 것이 최우선. `subscribeOn(Schedulers.boundedElastic())`이 블로킹 오프로딩의 표준 패턴입니다.

</details>

---

**Q3.** WebFlux에서 `Mono.fromFuture()`와 `Mono.fromCallable().subscribeOn(boundedElastic())`의 차이는 무엇인가요?

<details>
<summary>해설 보기</summary>

**`Mono.fromFuture(CompletableFuture)`:**
- `CompletableFuture`가 완료되면 Mono로 변환
- Future 자체의 실행 스레드는 Mono가 제어하지 않음
- Future가 `ForkJoinPool.commonPool()`에서 실행된다면 해당 풀의 스레드 사용
- Future에 블로킹 코드가 있어도 Mono 파이프라인의 EventLoop와 무관

**`Mono.fromCallable().subscribeOn(Schedulers.boundedElastic())`:**
- Callable의 실행 스레드를 `boundedElastic`으로 명시적 지정
- 블로킹 코드를 안전하게 실행하는 표준 패턴
- 실행 스레드 예측 가능

선택 기준:
- 이미 `CompletableFuture`를 반환하는 라이브러리 → `fromFuture()`
- 블로킹 코드를 직접 래핑 → `fromCallable().subscribeOn(boundedElastic())`
- 단, `fromFuture()`를 사용하더라도 Future 내부의 블로킹 코드는 적절한 스레드 풀에서 실행해야 합니다:

```java
// 위험: commonPool에서 블로킹 실행 가능성
Mono.fromFuture(CompletableFuture.supplyAsync(() -> jdbcTemplate.query(...)));

// 안전: boundedElastic에서 블로킹 실행
Mono.fromFuture(CompletableFuture.supplyAsync(
    () -> jdbcTemplate.query(...),
    Executors.newCachedThreadPool()  // 또는 boundedElastic executor
));

// 가장 명시적:
Mono.fromCallable(() -> jdbcTemplate.query(...))
    .subscribeOn(Schedulers.boundedElastic());
```

</details>

---

<div align="center">

**[⬅️ 이전: C10K 문제](./02-c10k-problem.md)** | **[홈으로 🏠](../README.md)** | **[다음: Reactive Programming의 탄생 ➡️](./04-reactive-programming-history.md)**

</div>
