# 언제 WebFlux를 쓰지 말아야 하는가 — 실무 판단 기준

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 블로킹 라이브러리 의존성이 많을 때 WebFlux를 도입하면 어떤 문제가 생기는가?
- 팀이 Reactive 패러다임에 익숙하지 않을 때의 실질적 생산성 손실은 얼마나 되는가?
- 단순 CRUD 서비스에서 WebFlux의 학습 비용 대비 이득이 없는 이유는 무엇인가?
- Reactive 스택의 스택 트레이스 파편화가 디버깅에 얼마나 어려움을 주는가?
- WebFlux 도입이 적합한 서비스와 적합하지 않은 서비스를 어떻게 구분하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"WebFlux가 좋다고 하니 무조건 도입하자"는 결정은 팀을 힘들게 합니다. 새 기술 도입의 판단 기준은 "성능이 얼마나 좋은가"가 아니라 "이 서비스에서 얻는 이득이 치르는 비용보다 큰가"입니다. WebFlux가 분명한 이점을 가져다주는 서비스가 있는 반면, 오히려 복잡성과 생산성 손실만 초래하는 서비스도 있습니다. 이 문서는 그 판단 기준을 명확히 제시합니다.

---

## 😱 흔한 실수 (Before — WebFlux 도입을 잘못 결정할 때)

```
실수 1: 성능 수치만 보고 도입 결정

  "WebFlux는 MVC보다 3배 빠르다고 했으니 무조건 도입"
  → 우리 서비스는 동시 접속자 50명 (차이 없음)
  → 기존 JPA 코드 전부 boundedElastic 오프로딩
  → 코드 복잡도 증가, 성능은 오히려 저하

실수 2: 팀 준비 없이 전환

  "WebFlux 도입했는데 왜 자꾸 버그가 나지?"
  → onErrorResume에서 에러를 삼켜 트랜잭션 롤백 안 됨
  → subscribe() 내부에서 또 subscribe() → 컨텍스트 유실
  → 스택 트레이스 파편화로 원인 찾는 데 수일 소요

실수 3: 모든 서비스를 동일한 기술로 통일

  "마이크로서비스 전부 WebFlux로 바꿔야 일관성이 있다"
  → 결제 서비스: 복잡한 도메인 로직, JPA 의존성 → WebFlux 부적합
  → 알림 서비스: I/O 집약, 높은 동시성 → WebFlux 적합
  → 서비스 특성에 맞는 기술 선택이 일관성보다 중요
```

---

## ✨ 올바른 접근 (After — 조건 기반 WebFlux 도입 판단)

```
WebFlux 도입 전 자가 진단 체크리스트:

필수 조건 (하나라도 아니면 MVC 유지 권장):
  □ 동시 접속자가 수천 명 이상이거나 목표로 하는가?
  □ I/O 집약적 서비스인가? (DB 대기, 외부 API 다수)
  □ 팀이 함수형 프로그래밍 경험이 있거나 학습 의지가 강한가?
  □ 기존 블로킹 라이브러리 의존성이 적은가?
  □ 새 서비스이거나 마이그레이션 비용을 감당할 수 있는가?

경고 신호 (하나 이상이면 재검토):
  ☒ JPA + Hibernate에 강하게 의존
  ☒ QueryDSL, 복잡한 JPQL 쿼리 다수
  ☒ 동기 블로킹 외부 라이브러리 사용 (레거시 SOAP 클라이언트 등)
  ☒ 팀원 대부분이 Reactive 경험 없음
  ☒ CRUD 중심 단순 서비스
  ☒ 복잡한 도메인 로직 중심 서비스

결론이 MVC라면:
  부끄러운 일이 아님. MVC + HikariCP 200 스레드로
  대부분의 서비스는 충분히 처리 가능.
```

---

## 🔬 내부 동작 원리

### 1. 블로킹 라이브러리 의존성 — 실질적 문제

```
문제 시나리오:

  프로젝트 의존성:
    spring-boot-starter-webflux  ← 비동기
    mybatis-spring                ← 블로킹 JDBC
    spring-data-jpa               ← 블로킹 JDBC
    legacy-payment-sdk            ← 블로킹 HTTP
    report-generator              ← 블로킹 파일 I/O

  WebFlux 도입 후 해야 할 일:
    모든 JPA 호출 → Mono.fromCallable() + subscribeOn(boundedElastic)
    모든 MyBatis 호출 → 동일
    레거시 SDK → 동일
    파일 생성 → 동일

  결과:
    ① 코드 전체에 subscribeOn 도배
    ② 트랜잭션 관리 복잡 (ThreadLocal → Reactor Context 불일치)
    ③ boundedElastic 스레드 수가 병목 → MVC 대비 성능 저하
    ④ 개발 생산성 급감

  실질적 비교:
    MVC:
      userRepository.findById(id);  // 1줄

    WebFlux + JPA:
      Mono.fromCallable(() -> userRepository.findById(id).orElseThrow())
          .subscribeOn(Schedulers.boundedElastic());  // 3줄
      // + 트랜잭션 처리 복잡성
      // + BlockHound 경고 관리
```

### 2. 팀 생산성 손실 — 학습 곡선의 현실

```
Reactive 학습 단계별 생산성:

  0단계 (MVC 기준): 100% 생산성

  1단계 - Reactive 입문 (~1개월):
    생산성: 30~50%
    증상:
      Mono/Flux 반환 타입 혼란
      subscribe() 직접 호출 남발
      에러 처리 패턴 미숙 (try-catch 습관)
    실수:
      flatMap에서 블로킹 메서드 호출
      Mono.empty()와 Mono.just(null) 혼동
      Cold/Hot Publisher 개념 미이해

  2단계 - 기본 패턴 습득 (~3개월):
    생산성: 60~80%
    남은 어려움:
      복잡한 에러 전파 (onError vs onErrorResume)
      트랜잭션과 Reactive 결합
      테스트 (StepVerifier, WebTestClient)

  3단계 - 실무 응용 (~6~12개월):
    생산성: 90~100%
    여전히 어려운 점:
      메모리 누수 진단
      스택 트레이스 해석
      성능 튜닝 (스케줄러, 연결 풀)

  비용 계산 (팀원 5명):
    1인 3개월 생산성 손실 = 50% × 3개월 = 1.5개월
    5인: 7.5개월 분량의 개발 일정 지연
    → ROI: 이 비용을 상쇄할 성능/확장성 이득이 있는가?
```

### 3. 단순 CRUD 서비스 — 이득 없는 복잡성

```
단순 CRUD 서비스 특성:
  동시 접속자: 수십~수백명
  트랜잭션: 단순 (저장/조회/수정/삭제)
  외부 I/O: 최소 (DB 쿼리 1~2개)
  비즈니스 로직: 단순

  MVC + JPA:
    @Transactional
    public User createUser(CreateUserRequest req) {
        validateEmail(req.getEmail());
        User user = User.from(req);
        return userRepository.save(user);
    }
    // 직관적, 간결, 팀 누구나 이해

  WebFlux + R2DBC:
    @Transactional
    public Mono<User> createUser(CreateUserRequest req) {
        return Mono.fromCallable(() -> validateEmail(req.getEmail()))
            .flatMap(valid -> userRepository.save(User.from(req)));
        // 더 복잡, 동일한 기능
    }

  성능 비교 (동시 100명):
    MVC:    p95 = 50ms, 처리량 = 2,000 req/s
    WebFlux: p95 = 48ms, 처리량 = 2,100 req/s
    → 차이: 거의 없음

  결론: CRUD 서비스에서 WebFlux의 이득 = 거의 없음
        비용 = 학습 곡선 + 코드 복잡성 + JPA 마이그레이션
```

### 4. 스택 트레이스 파편화 — 디버깅 난이도

```
MVC 에러 스택 트레이스:
  java.lang.NullPointerException
    at com.example.OrderService.createOrder(OrderService.java:42)
    at com.example.OrderController.create(OrderController.java:28)
    at sun.reflect.NativeMethodAccessorImpl.invoke(...)
    ...
  → 파일명 + 줄 번호로 즉시 원인 파악

WebFlux 에러 스택 트레이스 (디버그 없을 때):
  reactor.core.Exceptions$ErrorCallbackNotImplemented:
    java.lang.NullPointerException
  Caused by: java.lang.NullPointerException
    at com.example.OrderService.lambda$createOrder$2(OrderService.java:42)
    at reactor.core.publisher.MonoFlatMap$FlatMapMain.onNext(...)
    at reactor.core.publisher.FluxMap$MapSubscriber.onNext(...)
    at reactor.core.publisher.FluxOnAssembly$OnAssemblySubscriber.onNext(...)
    at io.reactor.netty.channel.FluxReceive.drainReceiver(...)
    ... (내부 Reactor 프레임 20~30줄)
  → Reactor 내부 클래스로 가득 → 원인 불명

해결 방법 (복잡성 증가):
  ① Hooks.onOperatorDebug() → 성능 오버헤드 20~40%
  ② ReactorDebugAgent (-javaagent) → 오버헤드 3~8%
  ③ checkpoint() 수동 추가 → 코드 분산

  개발팀의 실질적 영향:
    MVC: 에러 발생 → 줄 번호 확인 → 5분 내 해결
    WebFlux: 에러 발생 → 스택 트레이스 해석 → 로그 추가 → 재배포 → 1~2시간
    → 개발 사이클 당 디버깅 시간 급증
```

### 5. WebFlux가 적합한 서비스 vs 부적합한 서비스

```
적합한 서비스 유형:

  ① API 게이트웨이 / BFF (Backend for Frontend)
    - 여러 하위 서비스 병렬 호출 집계
    - 높은 동시 연결 처리
    - I/O 집약, 비즈니스 로직 최소
    → WebFlux 최적

  ② 실시간 스트리밍 서비스
    - 알림 서버, 시세 정보, 배송 추적
    - SSE, WebSocket 기반
    - 수만 개 장기 연결 유지 필요
    → WebFlux 최적

  ③ 고동시성 조회 API
    - 읽기 전용, 동시 요청 많음
    - 캐싱과 결합 (Redis Reactive)
    - R2DBC 사용 가능
    → WebFlux 적합

부적합한 서비스 유형:

  ① 복잡한 도메인 서비스
    - 주문, 결제, 재고 등 복잡한 비즈니스 규칙
    - JPA Dirty Checking, 복잡한 연관관계 필요
    - 도메인 전문가와 협업 중심 (코드 가독성 중요)
    → MVC + JPA 유지 권장

  ② 레거시 통합 서비스
    - 블로킹 레거시 SDK 다수
    - SOAP, 레거시 DB 드라이버
    → MVC 유지, 또는 boundedElastic 오프로딩 (임시)

  ③ 내부 어드민/배치 서비스
    - 낮은 동시성 (관리자만 사용)
    - 복잡한 리포트, 데이터 집계
    - 개발 속도와 유지보수성 중요
    → MVC 유지

  ④ 스타트업 MVP
    - 빠른 개발이 최우선
    - 팀 규모 작음 (2~3명)
    - 기술 부채 최소화
    → MVC로 시작, 성장 후 필요 시 전환
```

---

## 💻 실전 코드

### 실험 1: WebFlux 도입 적합성 자가 진단 도구

```java
/**
 * WebFlux 도입 적합성 자가 진단 체크리스트
 * 각 항목을 팀에서 평가하여 도입 여부 판단
 */
public class WebFluxAdoptionChecklist {

    /*
     * 이득 요인 (높을수록 WebFlux 적합)
     *
     * [동시성]
     *   현재 최대 동시 접속자: ___명
     *   목표 동시 접속자:     ___명
     *   → 500명 이상이면 WebFlux 이득 시작
     *
     * [I/O 프로파일]
     *   요청당 외부 API 호출: ___회
     *   요청당 DB 쿼리 수: ___회
     *   평균 I/O 대기 시간: ___ms
     *   → I/O 비율 높을수록 WebFlux 이득
     *
     * [스트리밍]
     *   SSE 또는 WebSocket 필요: YES / NO
     *   장기 연결 수: ___개
     *   → YES이면 WebFlux 강력 권장
     *
     * 비용 요인 (높을수록 MVC 유지 권장)
     *
     * [팀 역량]
     *   Reactive 경험자: ___명 / 전체 ___명
     *   함수형 프로그래밍 경험: HIGH / MID / LOW
     *   → 경험자 50% 미만이면 학습 비용 크음
     *
     * [기술 부채]
     *   JPA/Hibernate 코드 비율: ___%
     *   블로킹 외부 라이브러리 수: ___개
     *   → JPA 코드 50% 이상이면 마이그레이션 비용 큼
     *
     * [서비스 복잡도]
     *   도메인 모델 복잡도: HIGH / MID / LOW
     *   비즈니스 규칙 복잡도: HIGH / MID / LOW
     *   → 복잡도 높을수록 WebFlux 가독성 저하
     */
}
```

### 실험 2: MVC 유지 + 부분 WebFlux 전략

```java
// 전략: 동일 프로젝트에서 MVC 유지하되
// 일부 고성능 필요 엔드포인트만 WebFlux 스타일로 작성

// ❌ 불가능: 동일 JVM에서 MVC + WebFlux 혼용
// → 스프링은 하나의 컨텍스트에서 하나의 웹 스택만 지원

// ✅ 가능: 별도 서비스 분리
// legacy-service:8080 (Spring MVC + JPA)
//   → 기존 CRUD API 전담

// streaming-service:8081 (Spring WebFlux)
//   → 실시간 알림, SSE 전담
//   → Redis Pub/Sub 구독 → SSE 스트리밍

// ✅ 가능: MVC에서 CompletableFuture + 비동기
@RestController
public class MvcParallelController {

    @GetMapping("/dashboard")
    public CompletableFuture<Dashboard> getDashboard() {
        // MVC에서도 병렬 호출 가능 (비동기 컨트롤러)
        CompletableFuture<User> user =
            CompletableFuture.supplyAsync(() -> userService.getCurrent());
        CompletableFuture<List<Order>> orders =
            CompletableFuture.supplyAsync(() -> orderService.getRecent());
        return CompletableFuture.allOf(user, orders)
            .thenApply(v -> Dashboard.of(user.join(), orders.join()));
    }
}
// WebFlux만이 비동기 처리를 할 수 있는 게 아님
// MVC도 비동기 컨트롤러 지원 (단, EventLoop 방식보다 스레드 소비 많음)
```

### 실험 3: "지금 당장 WebFlux가 필요한가" 판단 플로우

```
WebFlux 도입 결정 플로우차트:

START
  │
  ▼
현재 MVC 서버가 성능 문제를 겪고 있는가?
  ├─ NO → MVC 유지 (문제 없으면 바꾸지 말 것)
  └─ YES
       │
       ▼
     성능 문제의 원인은 무엇인가?
       ├─ DB 쿼리 최적화 미흡 → DB 인덱스/쿼리 최적화 먼저
       ├─ 단일 서버 한계 → 수평 확장 고려 (비용 vs WebFlux 전환)
       ├─ I/O 집약 + 높은 동시성 → WebFlux 검토 진행
       └─ CPU 병목 → WebFlux 도움 안 됨, 캐싱/알고리즘 최적화

WebFlux 검토:
  블로킹 의존성 조사
    ├─ JPA/JDBC 비율 > 50% → 마이그레이션 비용 높음 → 재검토
    └─ 낮음 → 계속

  팀 역량 평가
    ├─ Reactive 경험자 < 30% → 교육 계획 먼저
    └─ 충분 → 계속

  서비스 분리 가능한가?
    ├─ 성능 필요 부분만 새 서비스로 분리 가능 → 분리 후 WebFlux 적용
    └─ 전체 마이그레이션 필요 → 비용/일정 계획 후 결정

END
```

---

## 📊 성능 비교

```
WebFlux 도입 비용 vs 이득 분석:

서비스 유형별 ROI:

단순 CRUD API (동시 200명):
  이득: p95 레이턴시 ~5% 개선
  비용: 팀 학습 3~6개월, 코드 복잡도 증가
  ROI: 음수 (비용 > 이득) → MVC 유지

중간 규모 API (동시 1,000명, I/O 집약):
  이득: 처리량 2~3배, 레이턴시 안정화
  비용: R2DBC 전환, 팀 학습 6개월
  ROI: 트래픽 성장 전망에 따라 다름

고동시성 스트리밍 (SSE, 동시 10,000 연결):
  이득: MVC로는 불가능, WebFlux만 가능
  비용: 팀 학습 + R2DBC 전환
  ROI: 양수 (WebFlux 없이는 요구사항 충족 불가)

API Gateway (다수 하위 서비스 집계):
  이득: 병렬 호출로 레이턴시 50~70% 감소
  비용: 학습 + 전환 (비즈니스 로직 적어 전환 용이)
  ROI: 양수

스타트업 MVP:
  이득: 처리량 향상 (미래)
  비용: 학습 시간 → 제품 출시 지연
  ROI: 음수 (지금은 빠른 출시가 더 중요)
```

---

## ⚖️ 트레이드오프

```
WebFlux 사용 vs 미사용 종합 트레이드오프:

WebFlux 사용:
  장점:
    높은 동시성 처리 (수천~수만 연결)
    메모리 효율 (스레드 스택 절감)
    SSE/WebSocket 자연스러운 지원
    서비스 간 병렬 호출 표현 용이
  단점:
    학습 곡선 (Reactive 패러다임)
    디버깅 복잡 (스택 트레이스 파편화)
    JPA 사용 불가 (R2DBC 전환 필요)
    블로킹 라이브러리와 충돌

MVC 사용:
  장점:
    팀 친숙도 높음 (생산성 유지)
    JPA, 풍부한 에코시스템 그대로 사용
    직관적 디버깅 (동기 스택 트레이스)
    낮은 기술 부채
  단점:
    동시 연결 한계 (스레드 수 = 동시 처리 수)
    SSE/WebSocket 구현 불편
    장기 연결 메모리 비효율

결론:
  서비스 특성과 팀 역량에 맞는 선택이 최선
  "최신 기술 = 최선"이 아님
  MVC로 충분한 서비스에 WebFlux를 강제하면 기술 부채 증가
```

---

## 📌 핵심 정리

```
WebFlux를 쓰지 말아야 할 때:

① 블로킹 의존성이 많을 때
  JPA, 레거시 SDK → boundedElastic 오프로딩
  → 복잡성만 증가, MVC 대비 성능 이득 없음

② 팀이 준비되지 않았을 때
  Reactive 경험자 30% 미만
  → 학습 비용 = 수개월 생산성 손실
  → 버그 증가, 유지보수 어려움

③ 단순 CRUD 서비스
  동시 접속 수백명 이하
  → WebFlux 이득 거의 없음
  → 코드 복잡도만 증가

④ 복잡한 도메인 서비스
  JPA Dirty Checking, 복잡한 연관관계 필수
  → R2DBC로 대체 불가
  → MVC + JPA가 더 적합

WebFlux를 써야 할 때:
  ✓ 수천 이상 동시 연결 + I/O 집약
  ✓ SSE, WebSocket 기반 실시간 서비스
  ✓ API Gateway (병렬 서비스 집계)
  ✓ 신규 서비스 + 팀 학습 의지 충분

판단 기준:
  "이 서비스에서 WebFlux의 이득 > 도입 비용인가?"
  → YES이면 도입, NO이면 MVC 유지
```

---

## 🤔 생각해볼 문제

**Q1.** Spring MVC로 개발 중인 서비스에서 WebFlux의 이점이 필요한 특정 기능(예: SSE 알림)만 추가하려면 어떻게 해야 하나요?

<details>
<summary>해설 보기</summary>

동일한 JVM에서 MVC와 WebFlux를 혼용할 수 없습니다. 가장 현실적인 방법은 **별도 서비스로 분리**하는 것입니다.

```
기존 MVC 서비스 (8080):
  /api/** → 모든 CRUD API

신규 WebFlux 서비스 (8081):
  /notifications/stream → SSE 전용
  Redis Pub/Sub → SSE 클라이언트에게 브로드캐스트
```

또는 **Spring WebMVC.fn**의 비동기 응답을 사용하는 방법도 있습니다:

```java
// MVC에서 SSE (제한적 지원)
@GetMapping(value = "/events", produces = "text/event-stream")
public ResponseBodyEmitter streamEvents() {
    SseEmitter emitter = new SseEmitter(Long.MAX_VALUE);
    scheduledExecutor.scheduleAtFixedRate(() -> {
        emitter.send(SseEmitter.event().data("ping"));
    }, 0, 1, TimeUnit.SECONDS);
    return emitter;
}
```

MVC의 `SseEmitter`도 SSE를 지원하지만, 연결마다 스레드를 점유하므로 수천 개 동시 연결에는 적합하지 않습니다. 대규모 SSE는 WebFlux로 분리하는 것이 맞습니다.

</details>

---

**Q2.** WebFlux 서비스에서 레거시 블로킹 라이브러리를 사용해야 할 때 `boundedElastic`외에 다른 대안이 있나요?

<details>
<summary>해설 보기</summary>

세 가지 대안이 있습니다.

**1. `boundedElastic` (기본 대안)**: 가장 흔한 방법이지만 스레드 수 한계가 있습니다.

**2. 커스텀 스레드 풀 + `Schedulers.fromExecutor()`**: 블로킹 작업 전용 스레드 풀을 명시적으로 관리합니다.
```java
ExecutorService blockingPool = Executors.newFixedThreadPool(
    50, new NamedThreadFactory("blocking-worker")
);
Scheduler blockingScheduler = Schedulers.fromExecutor(blockingPool);

Mono.fromCallable(() -> legacyClient.call())
    .subscribeOn(blockingScheduler);
```

**3. 비동기 래퍼 서비스**: 레거시 라이브러리를 별도 경량 서비스로 분리하고 WebClient로 호출합니다.
```
WebFlux 서비스 → WebClient → legacy-adapter (Spring MVC + 레거시 SDK)
```
네트워크 오버헤드가 생기지만, WebFlux 서비스의 논블로킹 특성을 완전히 유지합니다.

**4. 근본 해결**: 레거시 라이브러리를 논블로킹 대체제로 교체합니다. 이것이 최선이지만 비용이 큽니다.

</details>

---

**Q3.** "WebFlux를 도입했다가 MVC로 다시 되돌린" 실제 사례는 어떤 경우인가요?

<details>
<summary>해설 보기</summary>

실무에서 WebFlux를 포기하고 MVC로 돌아가는 주요 원인은 다음과 같습니다.

**사례 1 — 팀 역량 불일치**:
중견 기업 내부 어드민 시스템을 WebFlux로 구축. 팀원 대부분이 MVC 경험자. 6개월 후 버그 빈발, 디버깅에 과도한 시간 소모. 결국 MVC로 재작성.

**사례 2 — 블로킹 라이브러리 과다**:
레거시 ERP와 연동하는 서비스에 WebFlux 도입. 레거시 SOAP 클라이언트, 레거시 DB 드라이버가 모두 블로킹. 모든 호출에 `boundedElastic` 오프로딩 → 결국 MVC보다 느림.

**사례 3 — 과도한 복잡성**:
단순 주문 관리 시스템에 WebFlux 도입. 복잡한 비즈니스 로직을 Reactive 파이프라인으로 표현하다 보니 코드 가독성 급락. 신규 팀원 온보딩 시간 3배 증가.

**공통 교훈**:
WebFlux는 "더 나은 MVC"가 아니라 "다른 도구"입니다. 망치로 나사를 박을 수 없듯, 도구는 용도에 맞게 선택해야 합니다. 성능이 문제가 아닌 곳에서 성능 도구를 쓰는 것은 낭비입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 마이크로서비스에서 WebFlux](./04-microservice-webflux.md)** | **[홈으로 🏠](../README.md)**

</div>
