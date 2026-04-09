# 전통적 스레드 기반 모델의 한계 — Thread-per-Request

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 요청당 스레드(Thread-per-Request) 모델에서 스레드가 I/O를 기다리는 동안 CPU에게 무슨 일이 일어나는가?
- Tomcat의 기본 스레드 수 200개는 왜 한계가 되는가?
- 컨텍스트 스위칭(Context Switching)은 어떤 비용을 만들어 내는가?
- 스레드 풀이 고갈되면 서비스에 정확히 어떤 일이 발생하는가?
- Spring MVC가 Thread-per-Request 모델을 사용한다면, 왜 지금까지 잘 동작했는가?

---

## 🔍 왜 이 개념이 WebFlux에서 중요한가

WebFlux가 왜 필요한지 이해하려면, WebFlux가 무엇을 해결하려 하는지 알아야 합니다. Spring MVC와 Tomcat의 Thread-per-Request 모델은 수십 년간 웹 서버의 표준이었고, 대부분의 서비스에서 충분히 잘 동작했습니다. 그런데 외부 API를 다수 호출하는 서비스, 수천 개의 동시 연결이 필요한 서비스, 실시간 스트리밍 서비스가 늘어나면서 이 모델의 한계가 드러나기 시작했습니다.

WebFlux의 이벤트 루프 모델을 이해하기 전에, 기존 모델이 왜 I/O 대기 상황에서 비효율적인지 정확히 파악해야 합니다. 그래야 "WebClient를 쓰면 됩니다"가 아니라, "왜 이벤트 루프가 I/O 집약적 서비스에서 효율적인가"를 설명할 수 있습니다.

---

## 😱 흔한 실수 (Before — 스레드 모델을 모를 때의 접근)

```
상황: 외부 결제 API와 배송 API를 각각 호출하는 주문 서비스
      각 API 평균 응답 시간 300ms, 동시 요청 500개

원리를 모를 때의 접근:
  "느리다 → Tomcat maxThreads를 500으로 늘리자"
  CONFIG: server.tomcat.threads.max=500

결과:
  500 스레드 × 1MB(스택) = 500MB 추가 메모리 사용
  동시 요청 600개 발생 → 스레드 500개 모두 API 응답 대기 중
  → 601번째 요청: 큐에서 대기 → 응답 지연 → 타임아웃

또 다른 실수:
  "서버를 스케일 아웃하면 된다"
  → 인스턴스를 2배로 늘려도 근본 문제는 동일
  → 각 인스턴스의 스레드가 I/O를 기다리는 동안 CPU는 유휴 상태

실제 상황에서의 진단 오류:
  APM 모니터링 → CPU 사용률 15% → "서버 여유 있음"
  하지만 응답 지연 발생
  → CPU는 놀고 있지만 스레드는 I/O 대기 중
  → 리소스가 남아있어도 처리를 못 하는 상태
  → Thread-per-Request 모델의 구조적 문제
```

---

## ✨ 올바른 접근 (After — 원리를 알고 난 설계)

```
문제 재정의:
  "스레드 수 부족" → X
  "스레드가 I/O를 기다리며 낭비되는 구조" → O

올바른 분석:
  300ms 응답 시간의 외부 API 호출 시 스레드 상태:
    0ms   ~ 1ms:   요청 직렬화, 소켓 write → CPU 사용
    1ms   ~ 299ms: 네트워크 I/O 대기 → CPU 유휴 (99.7% 시간)
    299ms ~ 300ms: 응답 역직렬화 → CPU 사용
  
  → 스레드 생애주기의 99% 이상이 I/O 대기

해결 방향:
  I/O 대기 시간에 스레드를 다른 요청에 재사용
  → 논블로킹 I/O + 이벤트 루프 (Spring WebFlux)

WebFlux 적용 후:
  스레드 8개(CPU 코어 수)로 500개 동시 요청 처리
  각 스레드가 I/O 완료 이벤트가 올 때만 실행
  CPU 유휴 시간 → 다른 요청의 이벤트 처리에 사용
  → 스레드 비용 없이 동시 처리량 증가
```

---

## 🔬 내부 동작 원리

### 1. Thread-per-Request 모델 상세 구조

```
Spring MVC + Tomcat 요청 처리 흐름:

클라이언트 요청 도착
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│                   Tomcat Acceptor Thread                │
│  (연결 수락 전담, 1~2개)                                   │
│  새 요청 → Connector → Worker Thread Pool에서 스레드 배정   │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│              Worker Thread Pool (기본 200개)              │
│                                                         │
│  Thread-1 [요청 A 처리 중]                                │
│    └── DispatcherServlet → Controller → Service         │
│         → JdbcTemplate.query()  ← 여기서 블로킹!          │
│           [DB 응답 올 때까지 스레드 점유, CPU 유휴]           │
│                                                         │
│  Thread-2 [요청 B 처리 중]                                │
│    └── RestTemplate.getForObject("https://api...")      │
│         ← 외부 API 응답 올 때까지 블로킹, CPU 유휴             │
│                                                         │
│  Thread-3 [요청 C 처리 중]                                │
│    └── FileInputStream.read() ← 파일 읽기 블로킹            │
│                                                         │
│  Thread-200 [대기 중 or 처리 중]                           │
│                                                         │
│  Thread-201이 필요한 상황 → 큐에서 대기 → 응답 지연 시작         │
└─────────────────────────────────────────────────────────┘

핵심 문제:
  스레드는 I/O가 완료될 때까지 OS에 의해 "WAITING" 상태로 전환
  CPU는 다른 스레드를 실행할 수 있지만, 그 스레드도 I/O 대기 중이라면
  → CPU 코어가 남아있어도 모든 스레드가 대기 중인 상황 발생
```

### 2. 스레드가 I/O를 기다리는 동안 OS 레벨에서 일어나는 일

```
스레드 상태 전환 (Java Thread States):

RUNNABLE ─── CPU 할당 ──→ 실행 중 (코드 실행)
    │
    │ read() / write() / sleep() 호출
    ▼
BLOCKED / WAITING ─── I/O 완료 이벤트 ──→ RUNNABLE
    │
    (이 상태에서 CPU는 이 스레드를 스케줄하지 않음)
    (하지만 스레드 컨텍스트(스택, 레지스터)는 메모리에 유지)

OS 스케줄러 동작:
  I/O 시스템 콜 (read, recv 등) 호출
    ↓
  OS: "이 소켓에 데이터 없음 → 스레드를 대기 큐로 이동"
    ↓
  다른 RUNNABLE 스레드에 CPU 할당 (Context Switch!)
    ↓
  소켓에 데이터 도착 → 인터럽트 → OS가 스레드를 RUNNABLE로 복귀
    ↓
  스케줄러가 다시 이 스레드에 CPU 할당 (또 Context Switch!)

결론:
  I/O 한 번 = 최소 2번의 Context Switch
  Context Switch 1회 비용: 수 μs ~ 수십 μs
  → 스레드가 많을수록 Context Switch 오버헤드 급증
```

### 3. 컨텍스트 스위칭(Context Switch) 비용 상세

```
Context Switch가 비싼 이유:

① CPU 레지스터 저장/복원
   현재 스레드: PC, SP, 범용 레지스터 → 메모리에 저장
   다음 스레드: 메모리에서 레지스터 복원
   비용: 수백 ns ~ 수 μs

② CPU 캐시 오염 (Cache Pollution)
   현재 스레드의 L1/L2 캐시 데이터
   → 다른 스레드로 전환 시 캐시가 무효화
   → 새 스레드: 메모리에서 데이터 재로딩 (100ns → 수 μs)
   이것이 실제로 가장 비싼 비용

③ TLB(Translation Lookaside Buffer) 플러시
   가상 주소 → 물리 주소 변환 캐시
   스레드 전환 시 부분 무효화
   → 페이지 테이블 재조회 오버헤드

스레드 수별 Context Switch 발생 규모:
  스레드 10개:  초당 수백 번의 Context Switch → 무시 가능
  스레드 100개: 초당 수천 번 → 주의 필요
  스레드 1000개: 초당 수만 번 → CPU 시간의 상당 부분이 Context Switch에 소비

측정 방법:
  vmstat 1        # cs 컬럼: 초당 Context Switch 횟수
  pidstat -w 1    # 프로세스별 Context Switch 횟수

실제 사례:
  스레드 200개 서버, 외부 API 다수 호출
  → vmstat로 초당 Context Switch 50,000회 관측
  → CPU 사용률 80%인데 실제 유효 작업은 20%
  → 60%가 Context Switch 오버헤드
```

### 4. 스레드 메모리 비용

```
Java Thread 메모리 구성:

각 스레드별 고정 비용:
  JVM Thread Stack:  기본 512KB ~ 1MB (-Xss 설정)
  OS Thread 구조체:  수 KB
  Thread Local Storage: 애플리케이션 데이터

스레드 수별 메모리 사용량:
  스레드 50개:   50MB  (스택만)
  스레드 200개:  200MB (Tomcat 기본값)
  스레드 1000개: 1GB   (스케일 아웃 전 한계)
  스레드 10000개: 10GB  → 일반 서버에서 불가능

실제 Tomcat 설정과 한계:
  server.tomcat.threads.max=200  (기본값)
  200개 스레드가 모두 외부 API 응답 대기 중
  → 201번째 요청: acceptCount 큐에서 대기
  → acceptCount 초과: Connection Refused

문제의 핵심:
  "동시에 처리할 수 있는 요청 수 = 스레드 수"
  → 동시 처리 수를 늘리려면 스레드를 늘려야 함
  → 스레드를 늘리면 메모리와 Context Switch 비용 증가
  → 선형이 아닌 제곱 수준으로 비용 증가
  → 한계에 도달 (C10K 문제, 다음 문서 참고)
```

### 5. 블로킹 I/O 시스템 콜 레벨 분석

```
Java RestTemplate.getForObject() 내부:

애플리케이션 코드:
  String result = restTemplate.getForObject(url, String.class);
  // ← 이 한 줄에서 스레드 수백 ms 블로킹

JVM → OS 시스템 콜 흐름:
  ① socket() → 소켓 생성
  ② connect() → TCP 3-way handshake (블로킹 or 논블로킹)
  ③ write() → HTTP 요청 전송
  ④ read()  ← 여기서 블로킹!
     "서버가 응답할 때까지 이 스레드를 WAITING 상태로"
     OS가 스레드를 대기 큐로 이동
     CPU는 다른 스레드로 Context Switch
     응답 도착 → 인터럽트 → 스레드 RUNNABLE 복귀
  ⑤ close() → 연결 종료

strace로 확인:
  strace -f -e trace=network java App 2>&1 | grep "read\|write\|connect"
  ...
  [pid 1234] connect(5, ...) = 0           ← TCP 연결
  [pid 1234] write(5, "GET /api/...", 64)  ← 요청 전송
  [pid 1234] read(5,               ← 여기서 블로킹 시작
  (300ms 대기)
  [pid 1234] read(5, "HTTP/1.1 200 OK...", 4096) = 892  ← 응답 도착
```

---

## 💻 실전 코드

### 실험 1: 블로킹 모델의 스레드 점유 시각화

```java
// Spring MVC 컨트롤러 — Thread-per-Request 모델
@RestController
@RequestMapping("/blocking")
public class BlockingController {

    private final RestTemplate restTemplate = new RestTemplate();

    @GetMapping("/order")
    public OrderResult processOrder() {
        // 요청 시작 시 스레드 확인
        String threadName = Thread.currentThread().getName();
        log.info("[{}] 요청 시작", threadName);

        // 결제 API 호출 — 300ms 블로킹
        // 이 300ms 동안 threadName 스레드는 완전히 점유됨
        PaymentResult payment = restTemplate.getForObject(
            "http://slow-api.example.com/payment",
            PaymentResult.class
        );

        // 배송 API 호출 — 추가 200ms 블로킹
        ShippingResult shipping = restTemplate.getForObject(
            "http://slow-api.example.com/shipping",
            ShippingResult.class
        );

        // 총 500ms 동안 스레드 1개가 점유됨
        // 이 시간 동안 동일 스레드로 다른 요청 처리 불가
        log.info("[{}] 요청 완료", threadName);
        return new OrderResult(payment, shipping);
    }
}
```

```bash
# 스레드 점유 확인 — jstack으로 스레드 덤프
curl -s http://localhost:8080/blocking/order &  # 백그라운드에서 요청
sleep 0.1
jstack $(pgrep java) | grep -A 5 "http-nio"

# 출력 예시 (요청 처리 중):
# "http-nio-8080-exec-1" #23 daemon prio=5 os_prio=0
#    java.lang.Thread.State: WAITING (parking)
#       at sun.misc.Unsafe.park(Native Method)
#       - waiting on <0x000000076b872a60> (Socket read)
#       at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
# → http-nio-exec-1 스레드가 소켓 읽기 대기 중 (WAITING)
```

### 실험 2: 동시 요청 증가 시 스레드 고갈 재현

```java
// application.properties
// server.tomcat.threads.max=10  ← 의도적으로 낮게 설정
```

```bash
# k6로 동시 20개 요청 발사 (스레드 10개 서버에)
cat > load-test.js << 'EOF'
import http from 'k6/http';
import { sleep } from 'k6';

export let options = {
    vus: 20,        // 가상 사용자 20명 (동시 요청 20개)
    duration: '30s',
};

export default function () {
    let start = Date.now();
    let res = http.get('http://localhost:8080/blocking/order');
    let duration = Date.now() - start;
    console.log(`응답 시간: ${duration}ms, 상태: ${res.status}`);
}
EOF

k6 run load-test.js

# 결과 예시:
# VU 1~10:  응답 시간 ~500ms  (정상, 스레드 배정됨)
# VU 11~20: 응답 시간 ~1500ms  (스레드 대기 500ms + 처리 500ms + α)
# → 스레드 10개가 I/O 대기 중 → 나머지 10개 요청이 큐에서 대기

# 동시에 JMX로 스레드 수 모니터링:
# Tomcat → Catalina → ThreadPool → currentThreadsBusy
# → 10개 스레드 모두 WAITING 상태 확인 가능
```

### 실험 3: CPU 유휴 vs 스레드 점유 확인

```bash
# top으로 CPU 사용률 확인
top -p $(pgrep java)

# 결과 예시:
#   CPU: 8.5%  ← CPU는 여유 있음
#   메모리: 512MB
#   스레드: 210개

# 하지만 응답 지연 발생 중
# → CPU는 놀고 있는데 스레드는 I/O 대기 중
# → Thread-per-Request 모델의 핵심 비효율

# Java 스레드 상태별 분류
jstack $(pgrep java) | grep "Thread.State" | sort | uniq -c | sort -rn
# 출력:
#  189 Thread.State: WAITING (on object monitor)   ← I/O 대기
#   12 Thread.State: RUNNABLE                       ← 실제 실행 중
#    8 Thread.State: TIMED_WAITING                  ← 타임아웃 대기
# → 189개 스레드가 대기 중, 12개만 실제 작업
```

---

## 📊 성능 비교

```
Thread-per-Request 모델 동시 요청 처리 한계 (외부 API 300ms 응답 가정):

스레드 수 | 동시 처리 가능 요청 | 메모리(스택) | CPU(Context Switch)
─────────┼──────────────────┼────────────┼───────────────────
10       | 10개             | 10MB       | 무시 가능
50       | 50개             | 50MB       | 미미
200      | 200개            | 200MB      | 약간
500      | 500개            | 500MB      | 눈에 띔
1000     | 1000개           | 1GB        | 유의미한 오버헤드
5000     | 5000개           | 5GB        | 심각 (GC 압박 포함)

WebFlux(이벤트 루프, CPU 코어 8개) 동일 조건:

스레드 수 | 동시 처리 가능 요청 | 메모리     | CPU
─────────┼──────────────────┼──────────┼──────────
8        | 수천 개(*)        | 최소      | 이벤트 처리에만 집중

(*) I/O 대기 중 스레드가 다른 요청 처리 → 스레드와 동시 처리 수 분리

핵심 차이:
  Thread-per-Request: 동시 처리 수 = 스레드 수 (선형 관계)
  Event Loop:         동시 처리 수 >> 스레드 수  (I/O 완료 이벤트 기반)

외부 API 응답 시간이 길수록 Event Loop 우위 증가:
  API 응답 10ms:  MVC 200스레드 → 초당 20,000 TPS 이론치
  API 응답 1000ms: MVC 200스레드 → 초당 200 TPS 이론치
                    WebFlux 8스레드 → 수천 TPS 가능 (I/O 대기 없음)
```

---

## ⚖️ 트레이드오프

```
Thread-per-Request 모델:

장점:
  ① 단순한 프로그래밍 모델
     - 동기 코드, 위에서 아래로 읽힘
     - try-catch로 에러 처리
     - ThreadLocal로 요청 컨텍스트 전달 (SecurityContext 등)
  ② 디버깅 용이
     - 스택 트레이스가 연속적 → 어디서 막혔는지 명확
  ③ 블로킹 라이브러리 호환
     - JDBC, 대부분의 Java 라이브러리가 블로킹 모델

단점:
  ① I/O 대기 시간에 스레드 낭비
     - 외부 API 다수 호출 시 스레드 대부분이 대기 상태
  ② 메모리 비용 선형 증가
     - 동시 처리 수 = 스레드 수 → 스레드 = 메모리
  ③ Context Switch 오버헤드
     - 스레드가 많아질수록 스케줄링 비용 증가

WebFlux Event Loop 모델:
  장점:
  ① 적은 스레드로 많은 동시 연결 처리
  ② I/O 대기 시간을 다른 요청 처리에 활용
  단점:
  ① 학습 곡선 (Reactive 패러다임)
  ② 블로킹 코드 혼입 시 전체 처리 지연
  ③ 스택 트레이스 파편화 → 디버깅 어려움

선택 기준:
  외부 I/O 많음 + 높은 동시성 필요 → WebFlux
  CPU 집약적 + 팀 익숙도 낮음 + 블로킹 라이브러리 의존 → MVC
```

---

## 📌 핵심 정리

```
Thread-per-Request 모델 핵심:

구조:
  요청 하나 = 스레드 하나 (Tomcat Worker Thread Pool)
  스레드가 I/O를 기다리는 동안 = WAITING 상태 (CPU 유휴)
  동시 처리 수 = 스레드 수 (선형 한계)

비용 구조:
  스레드 = 메모리(스택 ~1MB) + Context Switch 비용
  스레드 200개: 200MB + CS 오버헤드
  스레드 1000개: 1GB + 심각한 CS 오버헤드

I/O 대기 시간의 문제:
  외부 API 300ms 응답: 스레드 생애의 99% 이상이 WAITING
  CPU는 여유 있지만 스레드 고갈 → 응답 지연

WebFlux가 해결하는 것:
  I/O 대기 시간에 스레드를 다른 요청에 재사용
  스레드 수(8개)와 동시 처리 수(수천 개)를 분리
  → I/O 집약적 서비스에서 극적인 효율 개선
```

---

## 🤔 생각해볼 문제

**Q1.** CPU 사용률이 10%인 서버에서 응답 지연이 발생하고 있습니다. Thread-per-Request 모델에서 이것이 가능한 이유는 무엇이고, 어떻게 확인하겠습니까?

<details>
<summary>해설 보기</summary>

CPU 사용률 10%는 CPU가 실제로 할 일이 없다는 의미가 아닙니다. Thread-per-Request 모델에서 이 상황은 **모든 스레드가 I/O 대기(WAITING) 상태**일 때 발생합니다.

스레드 200개 중 190개가 외부 API 응답을 기다리고 있다면, OS는 이 스레드들을 CPU에 스케줄하지 않습니다. 결과적으로 CPU는 할 일이 없어 보이지만, 새 요청이 들어와도 처리할 스레드가 없습니다.

확인 방법:
```bash
# 1. 스레드 상태 확인
jstack $(pgrep java) | grep "Thread.State" | sort | uniq -c
# WAITING 비율이 높으면 I/O 대기 문제

# 2. CPU vs 스레드 사용률 비교
top -H -p $(pgrep java)  # 스레드별 CPU 사용률

# 3. Tomcat 스레드 풀 상태
curl http://localhost:8080/actuator/metrics/tomcat.threads.busy
# 최대값(max)에 가까우면 스레드 고갈 직전
```

이것이 Thread-per-Request 모델의 핵심 문제이며, WebFlux의 이벤트 루프 모델이 해결하려는 것입니다.

</details>

---

**Q2.** Tomcat의 `maxThreads`를 200에서 2000으로 늘리면 외부 API 집약적 서비스의 처리량이 선형으로 10배 증가할까요?

<details>
<summary>해설 보기</summary>

이론적으로는 그럴 수 있지만, 실제로는 다음 한계에 부딪힙니다.

**메모리 한계:** 2000 스레드 × 1MB = 2GB 추가 메모리. JVM Heap을 제외하고도 스택만으로 2GB를 요구합니다.

**Context Switch 한계:** 2000개 스레드가 모두 RUNNABLE 상태가 되는 순간(I/O 완료 이벤트 동시 도착 등), OS 스케줄러는 초당 수만 번의 Context Switch를 수행합니다. CPU 시간의 상당 부분이 실제 작업 대신 스케줄링에 소비됩니다.

**GC 압박:** 스레드 수 증가 → ThreadLocal 등 스레드 로컬 객체 증가 → GC 빈도와 시간 증가.

실험적으로는 일반적으로 스레드 수가 CPU 코어의 수십 배를 넘어가면 처리량 증가가 멈추고 오히려 감소하는 경향이 있습니다. I/O 집약적 서비스의 근본 해결책은 스레드를 늘리는 것이 아니라, I/O 대기 중에 스레드를 재사용하는 논블로킹 모델로 전환하는 것입니다.

</details>

---

**Q3.** Spring MVC에서 `@Async`와 `CompletableFuture`를 사용하면 Thread-per-Request 문제를 해결할 수 있지 않을까요?

<details>
<summary>해설 보기</summary>

`@Async`와 `CompletableFuture`는 **스레드를 다른 스레드 풀로 옮길 뿐**, 블로킹 자체를 제거하지 않습니다.

```java
// @Async 사용 예시
@Async
public CompletableFuture<String> callApi() {
    // Async 스레드 풀의 스레드 하나를 점유
    // RestTemplate 호출 → 이 스레드가 블로킹됨
    return CompletableFuture.completedFuture(
        restTemplate.getForObject(url, String.class)
    );
}
```

이 코드는 Tomcat 스레드를 해제하지만, `@Async` 스레드 풀의 스레드를 블로킹합니다. 결국 스레드가 다를 뿐, I/O 대기 문제는 동일합니다.

진짜 해결책은 **논블로킹 I/O 클라이언트(WebClient)**와 **이벤트 루프 기반 런타임(Netty)**을 조합하는 것입니다. I/O 작업 자체를 커널에 위임하고 완료 시 콜백만 받는 방식으로, 어떤 스레드도 I/O를 기다리며 블로킹되지 않습니다.

`@Async` + `CompletableFuture`는 MVC에서 비동기성을 흉내 낼 수 있지만, 진정한 논블로킹은 WebFlux + WebClient의 조합에서만 달성됩니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: C10K 문제 — 동시 연결 10,000개의 벽 ➡️](./02-c10k-problem.md)**

</div>
