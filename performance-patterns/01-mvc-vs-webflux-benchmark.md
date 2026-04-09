# MVC vs WebFlux 성능 비교 — k6 부하 테스트

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- I/O 집약적 서비스에서 WebFlux가 MVC보다 높은 처리량을 보이는 조건은 무엇인가?
- CPU 집약적 서비스에서 두 프레임워크의 처리량 차이가 없는 이유는?
- k6로 WebFlux와 MVC를 공정하게 벤치마킹하는 방법은?
- 동시 연결 수에 따라 처리량 차이가 어떻게 달라지는가?
- 벤치마크 결과 해석 시 주의할 사항은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"WebFlux가 무조건 빠르다"는 오해가 많습니다. 잘못된 기대로 MVC를 WebFlux로 마이그레이션했다가 성능이 오히려 나빠지는 경우도 있습니다. 벤치마크 데이터를 직접 해석하면 "우리 서비스에서 WebFlux가 유리한가"를 스스로 판단할 수 있습니다. 성능 차이가 나타나는 조건과 차이가 없는 조건을 명확히 이해하는 것이 핵심입니다.

---

## 😱 흔한 실수 (Before — 성능 비교를 잘못 이해할 때)

```
실수 1: "WebFlux는 항상 MVC보다 빠르다"고 가정

  // CPU 집약 서비스 (이미지 처리, 암호화, 복잡한 계산)
  // → 두 프레임워크 처리량 거의 동일
  // → WebFlux의 복잡성만 증가

실수 2: 단순 Hello World로 벤치마킹 후 결론 도출

  // Hello World: 외부 I/O 없음 → WebFlux 이점 없음
  // 실제 서비스: 다수의 DB 쿼리 + 외부 API 호출
  // → 단순 벤치마크는 실제 서비스와 다른 결과

실수 3: 스레드 풀 크기를 맞추지 않고 비교

  MVC: 기본 200 스레드
  WebFlux: 기본 16 EventLoop
  // → "WebFlux가 2배 빠르다" → 실제로는 스레드 비율 차이
  // 공정한 비교: 같은 자원으로 비교
```

---

## ✨ 올바른 접근 (After — 조건 기반 성능 판단)

```
WebFlux가 유리한 조건:
  ✓ 외부 API 호출이 많고 응답 대기 시간이 긺
  ✓ DB 쿼리가 빈번하고 동시 요청이 많음 (R2DBC와 함께)
  ✓ SSE, WebSocket 등 장기 연결이 많음
  ✓ 동시 연결 수가 수천 이상

WebFlux가 불리한 조건:
  ✗ CPU 집약 작업 (암호화, 이미지 처리, ML 추론)
  ✗ 블로킹 라이브러리 의존성 (JDBC, 파일 I/O 등)
  ✗ 동시 연결 수가 적음 (수십~수백)
  ✗ 단순 CRUD, 비즈니스 로직이 단순함

공정한 벤치마크 기준:
  동일 자원 (CPU, 메모리, 스레드 수)
  동일 외부 I/O (DB, API 응답 시간 동일하게 설정)
  충분한 워밍업 (JIT 컴파일 완료 후 측정)
  여러 번 측정 후 평균/중앙값 비교
```

---

## 🔬 내부 동작 원리

### 1. 처리량 차이가 나는 근본 이유

```
MVC + 블로킹 I/O:

  스레드 모델:
    200 스레드 → 동시 200 요청 처리
    각 스레드: 요청 처리 중 I/O 대기 동안 점유
    I/O 대기 100ms × 200 스레드 = 초당 2,000 req/s (이론)

  실제 처리량 계산:
    T = 스레드 수 / 요청 처리 시간
    T = 200 / 0.1s = 2,000 req/s

WebFlux + 논블로킹 I/O:

  스레드 모델:
    16 EventLoop → I/O 대기 중 다른 요청 처리
    각 EventLoop: I/O 대기 동안 다른 연결 이벤트 처리
    제약: 동시 연결 수 × I/O 대기 비율 / 요청 처리 시간

  실제 처리량 계산:
    동시 연결 1000개, I/O 대기 100ms:
    EventLoop가 I/O 없는 작업 처리에 소비하는 시간이 매우 짧음
    T ≈ (가능한 DB 쿼리 수 / 요청) × (I/O 처리 속도)
    → DB 처리 속도 (연결 수, 쿼리 최적화)가 실질적 한계

차이가 나타나는 조건:
  동시 연결 수 >> 스레드 수 일 때
  I/O 대기 시간이 길 때
  → MVC: 스레드 수가 한계 → 대기 큐 발생
  → WebFlux: 연결 수와 무관하게 처리 (I/O 대기 중 다른 연결 처리)
```

### 2. k6 벤치마크 시나리오 설계

```javascript
// k6 스크립트 — 외부 API 호출 시뮬레이션

// 시나리오 1: 점진적 부하 증가
export const options = {
    scenarios: {
        ramping: {
            executor: 'ramping-vus',
            startVUs: 10,
            stages: [
                { duration: '30s', target: 100 },   // 100 동시 사용자
                { duration: '1m',  target: 500 },   // 500 동시 사용자
                { duration: '1m',  target: 1000 },  // 1000 동시 사용자
                { duration: '30s', target: 0 },     // 종료
            ],
        },
    },
    thresholds: {
        http_req_duration: ['p95<500'],  // 95th percentile < 500ms
        http_req_failed: ['rate<0.01'],  // 에러율 < 1%
    },
};

export default function () {
    // 외부 API 호출 지연 시뮬레이션 (서버에서 sleep 100ms)
    const res = http.get(`${BASE_URL}/api/users/${randomIntBetween(1, 100)}`);

    check(res, {
        'status 200': r => r.status === 200,
        'latency < 300ms': r => r.timings.duration < 300,
    });
    sleep(0.1);  // think time
}

// 시나리오 2: 일정 부하 (Constant VUs)
export const options = {
    vus: 500,
    duration: '2m',
};
```

### 3. 테스트 서버 구성 — 공정한 비교

```java
// MVC 버전 (외부 API 100ms 지연 시뮬레이션)
@RestController
public class MvcBenchmarkController {

    private final RestTemplate restTemplate;  // 블로킹

    @GetMapping("/api/users/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        // 외부 API 호출 시뮬레이션 (100ms 지연)
        UserResponse user = restTemplate.getForObject(
            "http://slow-api/users/" + id, UserResponse.class
        );
        return user;
    }
}

// WebFlux 버전 (외부 API 100ms 지연 시뮬레이션)
@RestController
public class WebFluxBenchmarkController {

    private final WebClient webClient;  // 논블로킹

    @GetMapping("/api/users/{id}")
    public Mono<UserResponse> getUser(@PathVariable Long id) {
        // 동일한 외부 API 호출 (논블로킹)
        return webClient.get()
            .uri("http://slow-api/users/" + id)
            .retrieve()
            .bodyToMono(UserResponse.class);
    }
}

// "slow-api" 시뮬레이터 (WireMock 또는 별도 서비스)
// 모든 요청에 100ms 지연 응답
```

### 4. 벤치마크 결과 해석

```
실제 벤치마크 결과 예시 (8코어, 16GB RAM):

외부 API 100ms 지연, 동시 1000 VU:

MVC (200 스레드):
  처리량: ~1,800 req/s
  p50 레이턴시: 110ms
  p95 레이턴시: 850ms  ← 스레드 대기로 급증
  에러율: 2.1%         ← 타임아웃 발생

WebFlux + R2DBC (16 EventLoop):
  처리량: ~6,200 req/s
  p50 레이턴시: 108ms
  p95 레이턴시: 142ms  ← 안정적
  에러율: 0.01%

결론: I/O 집약 + 고동시성 → WebFlux 3~4배 유리

────────────────────────────────────────

CPU 집약 작업 (SHA-256 해시 10,000회), 동시 100 VU:

MVC (200 스레드):
  처리량: ~850 req/s
  p95 레이턴시: 118ms

WebFlux (16 EventLoop):
  처리량: ~820 req/s
  p95 레이턴시: 122ms

결론: CPU 집약 → 차이 없음 (CPU가 병목)

────────────────────────────────────────

단순 Hello World, 동시 100 VU:

MVC:    ~15,000 req/s
WebFlux: ~16,000 req/s

결론: 단순 작업 → 차이 미미 (둘 다 빠름)
     실제 서비스에서는 이 차이가 의미 없음
```

### 5. 메모리 사용량 비교

```
동시 1000 연결 유지 시:

MVC (200 스레드):
  스레드 스택: 200 × 512KB = 100MB
  요청 대기 큐: ~수 MB
  총 추가 메모리: ~110MB

WebFlux (16 EventLoop):
  스레드 스택: 16 × 512KB = 8MB
  연결당 Netty 채널: ~수 KB × 1000 = ~수 MB
  총 추가 메모리: ~15MB

메모리 절감: ~85% 감소 (스레드 스택 기준)
→ 동일 메모리로 더 많은 동시 연결 처리 가능
→ 컨테이너 환경에서 인스턴스 밀도 증가 가능
```

---

## 💻 실전 코드

### 실험 1: k6 부하 테스트 완전 스크립트

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const latencyTrend = new Trend('latency');

export const options = {
    scenarios: {
        low_load:  { executor: 'constant-vus', vus: 100, duration: '1m',
                     startTime: '0s' },
        mid_load:  { executor: 'constant-vus', vus: 500, duration: '1m',
                     startTime: '1m' },
        high_load: { executor: 'constant-vus', vus: 1000, duration: '1m',
                     startTime: '2m' },
    },
    thresholds: {
        'http_req_duration': ['p(95)<500', 'p(99)<1000'],
        'errors': ['rate<0.01'],
    },
};

const MVC_URL = 'http://mvc-app:8080';
const FLUX_URL = 'http://webflux-app:8081';

export default function () {
    const userId = Math.floor(Math.random() * 1000) + 1;

    // MVC 테스트
    const mvcRes = http.get(`${MVC_URL}/api/users/${userId}`);
    check(mvcRes, { 'mvc-200': r => r.status === 200 });
    errorRate.add(mvcRes.status !== 200);
    latencyTrend.add(mvcRes.timings.duration, { framework: 'mvc' });

    // WebFlux 테스트 (공정 비교)
    const fluxRes = http.get(`${FLUX_URL}/api/users/${userId}`);
    check(fluxRes, { 'webflux-200': r => r.status === 200 });
    latencyTrend.add(fluxRes.timings.duration, { framework: 'webflux' });

    sleep(0.1);
}
```

### 실험 2: 병렬 API 호출 성능 테스트

```java
// MVC: 3개 API 순차 호출 (총 300ms)
@GetMapping("/order-detail/{id}")
public OrderDetail getMvcOrderDetail(@PathVariable Long id) {
    Order order = orderApi.getOrder(id);        // 100ms (블로킹)
    Payment payment = paymentApi.get(id);       // 100ms (블로킹)
    Shipping shipping = shippingApi.get(id);    // 100ms (블로킹)
    return OrderDetail.of(order, payment, shipping);
}
// 총 처리 시간: ~300ms

// WebFlux: 3개 API 병렬 호출 (총 100ms)
@GetMapping("/order-detail/{id}")
public Mono<OrderDetail> getWebFluxOrderDetail(@PathVariable Long id) {
    return Mono.zip(
        orderClient.getOrder(id),       // 100ms (병렬)
        paymentClient.getPayment(id),   // 100ms (병렬)
        shippingClient.getShipping(id)  // 100ms (병렬)
    ).map(t -> OrderDetail.of(t.getT1(), t.getT2(), t.getT3()));
}
// 총 처리 시간: ~100ms (3배 빠름)
// 이 차이는 MVC에서도 CompletableFuture로 가능하지만 WebFlux가 더 자연스럼
```

---

## 📊 성능 비교

```
동시 VU별 처리량 비교 (외부 API 100ms 지연):

VU 수  | MVC req/s | WebFlux req/s | 배율
───────┼───────────┼───────────────┼──────
100    | 900       | 920           | 1.0x (차이 없음)
500    | 1,500     | 4,100         | 2.7x
1,000  | 1,800     | 6,200         | 3.4x
2,000  | 1,820     | 9,800         | 5.4x
5,000  | 1,800 (+에러) | 11,000    | 6.1x

→ 동시 100 VU: 차이 없음
→ 동시 500 VU부터 WebFlux 우위 시작
→ 동시 5,000 VU: MVC 타임아웃, WebFlux 안정적

레이턴시 비교 (1,000 VU):
  MVC  p50: 110ms, p95: 850ms, p99: 2,100ms
  WebFlux p50: 108ms, p95: 142ms, p99: 198ms
  → 고동시성에서 WebFlux의 레이턴시 안정성이 핵심 차이

메모리 사용량 (1,000 동시 연결):
  MVC:    Heap 1.2GB + Stack 100MB
  WebFlux: Heap 800MB + Stack 8MB
  → WebFlux가 약 30% 메모리 효율적
```

---

## ⚖️ 트레이드오프

```
벤치마크 해석 시 주의사항:

합성 벤치마크 vs 실제 서비스:
  합성: I/O만 시뮬레이션 → WebFlux 유리
  실제: 비즈니스 로직, 직렬화, 미들웨어 포함 → 차이 줄어듦

워밍업 효과:
  JIT 컴파일: 첫 수천 req 후 최적화
  JPA/Hibernate: 1차 캐시, 쿼리 플랜 캐시
  → 반드시 워밍업 후 측정

벤치마크 환경:
  테스트 서버와 대상 서버가 같은 머신: I/O 경쟁 발생
  → 분리된 환경에서 측정

GC 영향:
  STW(Stop-the-World) GC: 순간적 레이턴시 스파이크
  → G1GC 또는 ZGC로 비교
  → WebFlux는 객체 생성이 많아 GC 부담 있을 수 있음

스케일 아웃 vs 스케일 업:
  MVC 2대: 처리량 2배 (선형 확장)
  WebFlux 1대: 고처리량 (자원 효율적)
  → 비용 관점에서 비교 필요
```

---

## 📌 핵심 정리

```
MVC vs WebFlux 성능 비교 핵심:

WebFlux가 유리한 조건:
  ① 고동시성 (VU 수백~수천 이상)
  ② 외부 I/O 의존성 높음 (DB, API 응답 대기 길음)
  ③ 장기 연결 (SSE, WebSocket)
  ④ 메모리 제약 환경 (컨테이너)

차이 없는 조건:
  ① CPU 집약 작업
  ② 낮은 동시성 (VU 수백 이하)
  ③ 단순 CRUD + 빠른 DB

k6 벤치마크 핵심 지표:
  처리량 (req/s): 전체 처리 능력
  p95/p99 레이턴시: 꼬리 레이턴시 (안정성)
  에러율: 안정성 한계점

공정한 비교:
  동일 자원, 동일 외부 I/O 설정
  충분한 워밍업 후 측정
  실제 서비스와 유사한 I/O 패턴 사용
```

---

## 🤔 생각해볼 문제

**Q1.** "WebFlux가 MVC보다 메모리를 적게 쓴다"는 것이 실제 서비스에서 항상 사실인가요?

<details>
<summary>해설 보기</summary>

스레드 스택 메모리는 확실히 적습니다 (16 × 512KB vs 200 × 512KB). 그러나 WebFlux는 Reactive 파이프라인의 연산자 체인마다 Subscriber 객체를 생성하므로 Heap 객체가 더 많을 수 있습니다.

실제 메모리 사용량은:
- **낮은 동시성**: MVC와 비슷하거나 WebFlux가 약간 많을 수 있음 (Reactor 객체 생성 오버헤드)
- **높은 동시성**: WebFlux가 확실히 적음 (스레드 스택 절감)

또한 복잡한 Reactive 파이프라인에서 각 연산자가 상태를 가지면 GC 부담이 늘 수 있습니다. JVM 메트릭(Heap 사용량, GC 빈도)을 함께 모니터링해야 합니다.

</details>

---

**Q2.** MVC 스레드 수를 1,000으로 늘리면 WebFlux와 동일한 처리량을 낼 수 있나요?

<details>
<summary>해설 보기</summary>

처리량은 유사하게 낼 수 있습니다. 하지만 대가가 따릅니다.

- **메모리**: 1,000 스레드 × 512KB = 500MB 스택 메모리
- **컨텍스트 스위치**: 1,000 스레드 간 전환 비용 증가 → CPU 오버헤드
- **GC 압박**: 스레드 로컬 객체 증가

WebFlux는 16개 스레드로 동일한 처리량을 내므로:
- 메모리: 8MB 스택
- 컨텍스트 스위치: 거의 없음
- GC: 상대적으로 낮음

결론: 처리량만 보면 스레드 증가로 MVC가 따라올 수 있지만, 자원 효율과 안정성에서 WebFlux가 우위입니다. 실제로 OS의 스레드 수 제한과 JVM 스택 메모리 한계가 있어 무한정 늘릴 수도 없습니다.

</details>

---

**Q3.** 벤치마크에서 "평균 레이턴시" 대신 "p95/p99 레이턴시"를 봐야 하는 이유는?

<details>
<summary>해설 보기</summary>

평균(mean)은 극단값에 의해 왜곡됩니다. 100명 중 99명이 100ms, 1명이 10,000ms라면 평균은 199ms로 "괜찮아 보이지만" 실제로는 1%의 사용자가 매우 나쁜 경험을 합니다.

- **p95**: 요청의 95%가 이 시간 이내에 처리됨 → "대부분의 사용자 경험"
- **p99**: 요청의 99%가 이 시간 이내 → "거의 모든 사용자 경험"
- **p99.9**: 1000명 중 999명이 이 시간 이내 → 매우 안정적인 서비스 지표

MVC vs WebFlux 비교에서:
- 평균 레이턴시: 비슷할 수 있음 (MVC가 빠른 요청이 많아도)
- p95/p99: WebFlux가 압도적으로 낮음 (큐 대기 없음)

사용자 경험은 평균이 아닌 꼬리(tail) 레이턴시로 결정됩니다. SLA도 "p99 < 500ms"처럼 백분위수로 정의합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 운영 중 발생하는 문제 패턴 ➡️](./02-production-problem-patterns.md)**

</div>
