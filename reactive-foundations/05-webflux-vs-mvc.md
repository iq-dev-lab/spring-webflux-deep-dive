# WebFlux vs Spring MVC — 언제 무엇을 선택할 것인가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- I/O 집약적 서비스와 CPU 집약적 서비스에서 WebFlux의 이점은 각각 다른가?
- WebFlux에서 JPA를 사용하면 왜 이벤트 루프의 이점이 소멸하는가?
- `Schedulers.boundedElastic()`으로 블로킹 코드를 오프로딩하면 WebFlux의 이점이 유지되는가?
- 팀의 Reactive 학습 비용은 선택 기준에 어떻게 반영해야 하는가?
- 기존 MVC 서비스를 WebFlux로 마이그레이션하는 것이 항상 이득인가?

---

## 🔍 왜 이 개념이 WebFlux에서 중요한가

WebFlux를 배우고 나면 "모든 서비스를 WebFlux로 전환해야겠다"는 충동이 생깁니다. 하지만 WebFlux가 항상 MVC보다 나은 것은 아닙니다. 잘못된 컨텍스트에서 WebFlux를 사용하면 복잡성만 증가하고 성능 이점은 없습니다.

이 문서는 WebFlux를 선택해야 하는 상황과 MVC가 더 적합한 상황을 명확한 기준으로 정리합니다. 잘못된 선택의 비용(JPA + WebFlux 함정, 팀 생산성 저하, 디버깅 어려움)도 함께 다룹니다.

---

## 😱 흔한 실수 (Before — "WebFlux = 무조건 빠름"이라는 오해)

```
오해 1: "새 프로젝트니까 WebFlux로 시작하자"
  상황: 사내 어드민 서비스 (하루 방문자 수백 명, 단순 CRUD)
  결과:
    JPA → JPA는 블로킹 → 모든 Repository 호출마다 Schedulers.boundedElastic() 래핑
    팀원 모두 Reactive 생소 → 코드 리뷰 지연
    기존 Spring Security → Reactive Security로 전환 필요 (추가 복잡도)
    → 개발 속도 50% 감소, 성능 이점 없음 (트래픽이 없음)

오해 2: "WebFlux + JPA 쓰면 더 빠를 것 같은데?"
  실제 코드:
    @GetMapping("/users")
    public Flux<User> getUsers() {
        // JPA는 블로킹 → boundedElastic으로 오프로딩
        return Mono.fromCallable(() -> userRepository.findAll())
                   .subscribeOn(Schedulers.boundedElastic())
                   .flatMapMany(Flux::fromIterable);
    }
  결과:
    EventLoop 스레드: 요청 수신 → boundedElastic으로 던짐
    boundedElastic 스레드: JPA 블로킹 → DB 대기 → 결과 반환
    → 실질적으로 Thread-per-Request와 동일
    → Reactive 복잡도는 추가됐지만 이점 없음

오해 3: "현재 MVC 서비스가 느리다 → WebFlux로 전환하면 해결"
  진짜 원인이 DB 쿼리 최적화 문제인 경우:
    WebFlux + R2DBC로 전환해도 느린 쿼리는 그대로
    → 병목은 쿼리지 스레드 모델이 아님
```

---

## ✨ 올바른 접근 (After — 명확한 선택 기준)

```
WebFlux가 진짜 유리한 상황:
  ✅ 외부 서비스/API를 많이 호출하는 서비스
     → 각 호출이 수백 ms 대기 → EventLoop이 그 시간에 다른 요청 처리
     
  ✅ 동시 연결이 매우 많은 서비스 (C10K 이상)
     → WebSocket, SSE, IoT 연결 관리
     
  ✅ 스트리밍 데이터 처리
     → 대용량 파일 스트리밍, 실시간 데이터 파이프라인
     
  ✅ 논블로킹 드라이버를 사용하는 경우
     → R2DBC (PostgreSQL, MySQL), Reactive Redis (Lettuce), Reactive MongoDB

MVC가 더 적합한 상황:
  ✅ CPU 집약적 서비스 (계산, 이미지 처리, 암호화)
     → EventLoop 이점 없음, MVC + 스레드 풀이 더 단순
     
  ✅ 팀이 Reactive에 익숙하지 않음
     → 학습 비용 > 성능 이점인 경우
     
  ✅ JPA 기반 서비스 (R2DBC 전환 불가 시)
     → 블로킹 JDBC 드라이버 → EventLoop 이점 소멸
     
  ✅ 단순 CRUD (동시 연결 수백 개 이하)
     → MVC 200 스레드로 충분, 복잡도 추가 불필요
```

---

## 🔬 내부 동작 원리

### 1. I/O 집약적 vs CPU 집약적 — 성능 차이 이유

```
I/O 집약적 서비스에서 WebFlux 우위:

  요청 처리 시간: 10ms CPU + 290ms I/O 대기 (총 300ms)

  MVC (스레드 200개):
    200 요청까지: 200스레드 × 300ms = 동시 200개 처리
    → TPS = 200 / 0.3 = ~667 TPS
    201번째 요청: 스레드 대기 → 응답 지연 시작

  WebFlux (EventLoop 스레드 8개):
    I/O 대기 290ms 동안 스레드는 다른 요청 처리
    → 스레드 1개가 290ms 안에 여러 요청의 이벤트 처리 가능
    → TPS = CPU 시간(10ms) 기준 스레드당 100 TPS × 8 = ~800 TPS
    → 동시 연결 수천 개도 가능

결론: I/O 시간이 길수록, 동시 요청이 많을수록 WebFlux 우위 커짐

──────────────────────────────────────

CPU 집약적 서비스에서 차이 없음:

  요청 처리 시간: 290ms CPU + 10ms I/O 대기 (총 300ms)

  MVC:
    CPU를 200ms 사용 → 스레드가 항상 RUNNABLE
    → 스레드 수가 CPU 코어 수와 비슷하면 최적
    8 코어 서버 → 스레드 8~16개로 최대 처리량 달성

  WebFlux:
    CPU 작업도 EventLoop 스레드에서 실행
    → 스레드 8개가 290ms CPU 사용 → 다른 요청 못 받음
    → MVC와 동일하거나 오히려 복잡도만 증가

결론: CPU 집약적 서비스는 WebFlux 이점 없음
```

### 2. JPA + WebFlux 함정 — 블로킹 드라이버 문제

```
JPA가 블로킹인 이유:
  JDBC → 소켓을 블로킹 모드로 열고 read() 시스템 콜로 DB 응답 대기
  JPA/Hibernate → JDBC 위에서 동작 → 필연적으로 블로킹

WebFlux에서 JPA 사용 시:
  옵션 A: 그냥 사용
    @GetMapping("/users")
    public Flux<User> getUsers() {
        return Flux.fromIterable(userRepository.findAll());
    }
    → userRepository.findAll()이 EventLoop 스레드에서 블로킹 실행!
    → EventLoop가 DB 응답 올 때까지 점유
    → 해당 EventLoop의 모든 연결 지연
    → 최악의 선택

  옵션 B: boundedElastic 오프로딩
    @GetMapping("/users")
    public Flux<User> getUsers() {
        return Mono.fromCallable(() -> userRepository.findAll())
                   .subscribeOn(Schedulers.boundedElastic())
                   .flatMapMany(Flux::fromIterable);
    }
    → JPA는 boundedElastic 스레드에서 블로킹 실행
    → EventLoop 보호는 됨
    → 하지만 결국 DB 쿼리당 boundedElastic 스레드 1개 점유
    → Thread-per-Request와 본질적으로 동일
    → Reactive 복잡도만 추가됨

  올바른 해결:
    R2DBC로 전환 (완전한 논블로킹 DB 드라이버)
    또는 JPA를 쓰는 서비스는 MVC 유지

JPA → R2DBC 전환 비용:
  - 지연 로딩(Lazy Loading) 없음 → 명시적 쿼리 필요
  - 복잡한 조인 쿼리 → DatabaseClient로 직접 작성
  - 기존 JPA Repository 코드 전면 재작성
  → 레거시 서비스 전환 비용이 큼
```

### 3. 팀 학습 비용 정량화

```
WebFlux 도입 시 팀 학습 단계:

1단계 (1~2주): 기본 개념
  Mono, Flux, subscribe(), Cold Publisher 이해
  map vs flatMap 차이
  → 이 단계에서 subscribe() 빠뜨리는 버그, block() 오남용 발생

2단계 (2~4주): 에러 처리, 스케줄러
  onErrorReturn, onErrorResume 패턴
  subscribeOn vs publishOn 혼동
  → 스케줄러 설정 실수로 블로킹 코드가 EventLoop에서 실행

3단계 (1~2개월): 디버깅 능력
  Reactor 스택 트레이스 해석
  Hooks.onOperatorDebug() 활성화
  StepVerifier로 테스트 작성
  → 이 단계 전까지 디버깅에 일반 Java보다 2~3배 시간 소요

4단계 (3~6개월): 숙달
  Context, Backpressure 전략, 복잡한 파이프라인 설계
  WebFlux + R2DBC + Security 통합
  → 팀 전체가 이 단계에 도달하려면 상당한 시간 필요

비용 대비 이점 계산:
  단순 CRUD, 낮은 트래픽: 학습 비용 >> 성능 이점 → MVC 선택
  높은 동시성, I/O 집약: 학습 비용 < 장기 이점 → WebFlux 고려
```

### 4. Spring MVC Async와 WebFlux의 차이

```
Spring MVC + DeferredResult / WebAsyncTask:
  Tomcat 스레드를 즉시 해제하고 비동기로 처리
  
  @GetMapping("/async")
  public DeferredResult<String> async() {
      DeferredResult<String> result = new DeferredResult<>();
      CompletableFuture.supplyAsync(() -> fetchData())
          .thenAccept(data -> result.setResult(data));
      return result;
  }
  → Tomcat 스레드: 요청 수신 후 즉시 반환 (풀에 반납)
  → 비동기 완료 시 DeferredResult를 통해 응답

MVC Async의 한계:
  CompletableFuture 내부의 fetchData()가 블로킹이면 결국 다른 스레드 점유
  스트리밍 처리 불가 (단일 결과)
  에코시스템 통합 부족 (Security, Tracing 등 컨텍스트 전달 어려움)

WebFlux의 차이:
  Netty 이벤트 루프 + Reactor 파이프라인
  → 에코시스템 전체가 Reactive (Security, Data, Tracing)
  → 스트리밍(Flux) 네이티브 지원
  → 파이프라인 전체가 논블로킹 (올바르게 사용 시)

실용적 선택:
  MVC에서 부분적 비동기가 필요 → DeferredResult/WebAsyncTask
  완전한 논블로킹 + 스트리밍 + 높은 동시성 → WebFlux
```

### 5. 선택 결정 플로우차트

```
WebFlux vs MVC 선택 결정 트리:

동시 연결이 수천 개 이상 필요한가?
  └─ YES → WebFlux 후보
  └─ NO  → 다음 질문으로

외부 API/DB I/O 대기 시간이 응답 시간의 50% 이상인가?
  └─ YES → WebFlux 후보
  └─ NO  → CPU 집약적 → MVC 고려

논블로킹 드라이버를 사용할 수 있는가?
  (R2DBC, Reactive Redis, Reactive MongoDB)
  └─ YES → WebFlux 적합
  └─ NO  → JPA 의존 → MVC 고려 또는 R2DBC 전환 비용 계산

팀이 Reactive에 익숙한가?
  └─ YES → WebFlux
  └─ NO  → 학습 비용 감수 여부 판단
            현재 성능 문제 있음? → 학습 투자 가치 있음
            현재 성능 충분? → MVC 유지

실시간 스트리밍, SSE, WebSocket이 필요한가?
  └─ YES → WebFlux (Flux 네이티브 지원)
  └─ NO  → MVC로도 충분 가능
```

---

## 💻 실전 코드

### 비교 실험: 동일 서비스 MVC vs WebFlux 처리량

```java
// Spring MVC 버전
@RestController
public class MvcOrderController {
    private final RestTemplate restTemplate = new RestTemplate();

    @GetMapping("/mvc/order/{id}")
    public OrderResponse getOrder(@PathVariable Long id) {
        // 각 호출 300ms 대기, 스레드 블로킹
        PaymentInfo payment = restTemplate.getForObject(
            "http://payment-service/payment/" + id, PaymentInfo.class);
        ShippingInfo shipping = restTemplate.getForObject(
            "http://shipping-service/shipping/" + id, ShippingInfo.class);
        return new OrderResponse(payment, shipping);
        // 총 600ms 스레드 점유 (순차 호출)
    }
}

// Spring WebFlux 버전
@RestController
public class WebFluxOrderController {
    private final WebClient webClient = WebClient.create();

    @GetMapping("/webflux/order/{id}")
    public Mono<OrderResponse> getOrder(@PathVariable Long id) {
        // 병렬 호출 — 두 요청을 동시에 (300ms만 대기)
        Mono<PaymentInfo> payment = webClient.get()
            .uri("http://payment-service/payment/" + id)
            .retrieve()
            .bodyToMono(PaymentInfo.class);

        Mono<ShippingInfo> shipping = webClient.get()
            .uri("http://shipping-service/shipping/" + id)
            .retrieve()
            .bodyToMono(ShippingInfo.class);

        // Mono.zip: 두 Mono가 모두 완료될 때 합산
        return Mono.zip(payment, shipping)
            .map(tuple -> new OrderResponse(tuple.getT1(), tuple.getT2()));
        // 총 300ms (병렬 처리) + EventLoop 스레드 블로킹 없음
    }
}
```

```bash
# k6 부하 테스트 — 동시 500 요청
cat > compare.js << 'EOF'
import http from 'k6/http';

export let options = { vus: 500, duration: '30s' };

export default function () {
    // 각 엔드포인트 순서대로 테스트
    let res = http.get('http://localhost:8080/mvc/order/1');
    // let res = http.get('http://localhost:8080/webflux/order/1');
    check(res, { '200 OK': r => r.status === 200 });
}
EOF

k6 run compare.js

# 예상 결과 (각 외부 API 300ms 응답 시):
#                MVC (200 스레드)    WebFlux (16 스레드)
# p95 응답시간:  ~1800ms (대기 포함)  ~320ms (병렬 처리)
# TPS:          ~110                 ~1500
# 에러율:       ~30% (스레드 고갈)    ~0%
```

### JPA + WebFlux 안티패턴 vs R2DBC 비교

```java
// 안티패턴: WebFlux + JPA (블로킹)
@GetMapping("/bad/users")
public Flux<User> badGetUsers() {
    // 이 코드는 EventLoop 스레드를 블로킹할 가능성!
    return Flux.fromIterable(userJpaRepository.findAll());
    // BlockHound 활성화 시 에러 발생
}

// 차선책: WebFlux + JPA + boundedElastic (복잡하고 이점 제한)
@GetMapping("/okay/users")
public Flux<User> okayGetUsers() {
    return Mono.fromCallable(() -> userJpaRepository.findAll())
               .subscribeOn(Schedulers.boundedElastic())
               .flatMapMany(Flux::fromIterable);
    // 작동하지만 Thread-per-Request와 본질적으로 동일
}

// 최선: WebFlux + R2DBC (완전한 논블로킹)
@GetMapping("/best/users")
public Flux<User> bestGetUsers() {
    return userR2dbcRepository.findAll();
    // 완전한 논블로킹 — EventLoop 스레드 블로킹 없음
}
```

---

## 📊 성능 비교

```
서비스 유형별 MVC vs WebFlux 처리량 비교 (8코어 서버, 스레드 200개):

I/O 집약적 서비스 (외부 API 각 200ms, 3개 순차 호출 = 600ms):

동시 요청  | MVC TPS     | WebFlux TPS | 비고
──────────┼────────────┼────────────┼──────────────
100       | ~165        | ~165        | 스레드 여유 있음, 차이 없음
200       | ~330        | ~330        | 스레드 한계 도달
500       | ~280 (지연)  | ~800        | 스레드 고갈 시작
1,000     | ~180 (심각) | ~1,600      | MVC 심각한 지연
5,000     | 불안정       | ~3,000+     | MVC 사실상 불가

CPU 집약적 서비스 (CPU 작업 500ms, I/O 없음):

동시 요청  | MVC TPS     | WebFlux TPS | 비고
──────────┼────────────┼────────────┼──────────────
8         | ~16         | ~16         | CPU 코어 한계, 동일
100       | ~16         | ~16         | CPU 병목, 동일
500       | ~16         | ~16         | CPU 병목, 동일

결론: CPU 집약적에서 WebFlux 추가 이득 없음

단순 CRUD 서비스 (DB 쿼리 10ms, R2DBC vs JDBC):

동시 요청  | MVC+JDBC    | WebFlux+R2DBC | 비고
──────────┼────────────┼──────────────┼──────────────
100       | ~10,000     | ~10,000       | 차이 없음
1,000     | ~8,000      | ~10,000       | 약간 차이
5,000     | ~3,000      | ~9,500        | 의미있는 차이
```

---

## ⚖️ 트레이드오프

```
WebFlux 도입 트레이드오프 요약:

도입 비용:
  팀 학습 시간: 최소 1~3개월 (숙달까지 6개월)
  코드베이스 전환: JPA → R2DBC (대규모 리팩토링)
  디버깅 방식 변경: 스택 트레이스 해석 방식 변화
  테스트 방식 변경: StepVerifier, WebTestClient 학습

기대 이익:
  I/O 집약적 서비스: 동시 처리 수 대폭 증가 (수십 배)
  메모리 효율: 스레드 수 감소 → 힙 메모리 가용 증가
  스트리밍: SSE, WebSocket 간결한 구현

손익 분기점:
  트래픽이 MVC 200 스레드 한계를 초과하기 시작하면 → WebFlux 전환 가치 있음
  현재 트래픽이 낮고 I/O 집약 아니면 → MVC 유지가 합리적

실용적 권장:
  신규 서비스: 트래픽 예측이 높고 I/O 집약적이면 WebFlux 시작
  기존 MVC 서비스: 성능 문제 없으면 전환 불필요
  JPA 의존 서비스: R2DBC 전환 비용 계산 후 결정
```

---

## 📌 핵심 정리

```
WebFlux vs MVC 선택 기준:

WebFlux가 유리한 조건:
  ✅ 외부 API/DB I/O가 응답 시간의 주요 부분 (I/O 집약적)
  ✅ 동시 연결 수천 개 이상 (C10K 이상)
  ✅ SSE, WebSocket, 스트리밍 데이터
  ✅ 논블로킹 드라이버 사용 가능 (R2DBC, Reactive Redis)
  ✅ 팀이 Reactive에 익숙 또는 학습 투자 의지 있음

MVC가 적합한 조건:
  ✅ CPU 집약적 서비스
  ✅ JPA 의존 + R2DBC 전환 불가
  ✅ 단순 CRUD, 낮은 트래픽
  ✅ 팀의 Reactive 익숙도 낮음
  ✅ 빠른 개발이 최우선

WebFlux + JPA 함정:
  JPA(블로킹 JDBC) + WebFlux = 복잡도만 증가, 이점 없음
  해결: R2DBC 전환 또는 MVC 유지

핵심 원칙:
  "WebFlux = 무조건 빠름" ← 틀린 명제
  "I/O 집약적 + 높은 동시성에서 WebFlux가 유리" ← 올바른 이해
```

---

## 🤔 생각해볼 문제

**Q1.** 현재 서비스가 MVC + JPA이고, 응답 시간의 70%가 외부 API 호출입니다. WebFlux로 전환하는 것이 항상 정답인가요?

<details>
<summary>해설 보기</summary>

외부 API 호출 최적화는 WebFlux 전환 없이도 가능합니다. 먼저 다음을 시도해볼 수 있습니다.

**MVC에서 병렬 API 호출 최적화:**
```java
// MVC에서 CompletableFuture + 병렬 호출
@GetMapping("/order/{id}")
public OrderResponse getOrder(@PathVariable Long id) {
    CompletableFuture<PaymentInfo> payment = CompletableFuture
        .supplyAsync(() -> restTemplate.getForObject(...));
    CompletableFuture<ShippingInfo> shipping = CompletableFuture
        .supplyAsync(() -> restTemplate.getForObject(...));
    
    return new OrderResponse(
        payment.join(),  // 두 요청 병렬 처리 후 합산
        shipping.join()
    );
}
```

이것만으로도 순차 호출 대비 응답 시간 절반 가능.

**WebFlux 전환 결정 기준:**
- 현재 MVC 스레드 풀이 고갈되어 대기가 발생하는가? → YES면 전환 가치
- 동시 연결이 수천 개 수준인가? → YES면 전환 고려
- R2DBC로 전환 가능한가? → NO면 전환 이점 제한적

JPA를 계속 써야 한다면 MVC에서 `AsyncRestTemplate`(deprecated) 또는 `WebClient`만 도입하는 혼용 방식도 가능합니다. Spring MVC에서도 `WebClient`를 사용할 수 있습니다(단, `.block()`으로 동기 전환 필요 또는 `DeferredResult` 활용).

</details>

---

**Q2.** Spring WebFlux와 Spring MVC를 동일 애플리케이션에서 혼용할 수 있나요?

<details>
<summary>해설 보기</summary>

**불가능합니다.** Spring WebFlux와 Spring MVC는 서로 다른 서블릿 스택을 기반으로 하기 때문에 동일 애플리케이션에서 혼용할 수 없습니다.

- **Spring MVC**: Servlet API(`javax.servlet.Servlet`) 기반, Tomcat/Jetty에서 동작
- **Spring WebFlux**: Reactive Streams 기반, Netty 또는 Undertow에서 동작

classpath에 `spring-webmvc`와 `spring-webflux`가 모두 있으면, Spring Boot Auto-configuration은 `spring-webmvc`를 우선합니다.

다만 **WebClient는 MVC 환경에서도 사용 가능**합니다. MVC 컨트롤러에서 WebClient를 사용하면 논블로킹 HTTP 클라이언트의 이점을 부분적으로 활용할 수 있습니다 (단, MVC의 Tomcat 스레드 모델은 그대로).

**마이크로서비스 아키텍처라면 혼용 가능:**
- 서비스 A: Spring MVC (단순 CRUD)
- 서비스 B: Spring WebFlux (높은 동시성, 스트리밍)
- → 각 서비스가 독립 배포 → 서로 다른 스택 사용 가능

</details>

---

**Q3.** WebFlux를 도입했지만 블로킹 라이브러리를 모두 `Schedulers.boundedElastic()`으로 래핑했습니다. 이것으로 WebFlux의 이점을 충분히 얻고 있다고 볼 수 있나요?

<details>
<summary>해설 보기</summary>

부분적으로만 이점을 얻고 있습니다.

**얻는 것:**
- EventLoop 스레드 보호 (블로킹 코드가 EventLoop를 차단하지 않음)
- 병렬 I/O 호출 (`Mono.zip`, `Flux.merge`)의 이점
- WebClient를 통한 논블로킹 HTTP 클라이언트

**잃는 것:**
- DB I/O의 논블로킹 이점: `boundedElastic` 스레드가 블로킹 DB 대기
  - `boundedElastic`은 기본 최대 10 × CPU 코어 수 스레드 생성
  - 결국 이 스레드들이 Thread-per-Request처럼 DB 대기

**실질적 평가:**
```
MVC 200스레드 서버 vs WebFlux + boundedElastic
  외부 API(WebClient): WebFlux 명확한 우위
  DB 쿼리(JPA + boundedElastic): 사실상 동일
  복잡도: WebFlux가 훨씬 높음
```

결론: DB가 주요 I/O이고 JPA를 써야 한다면, WebFlux + boundedElastic보다 MVC가 더 단순합니다. 진짜 이점을 얻으려면 DB 레이어도 R2DBC로 완전히 논블로킹으로 전환해야 합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Reactive Programming의 탄생](./04-reactive-programming-history.md)** | **[홈으로 🏠](../README.md)** | **[다음: Reactive Streams 스펙 완전 분해 ➡️](./06-reactive-streams-spec.md)**

</div>
