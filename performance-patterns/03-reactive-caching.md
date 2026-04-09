# Reactive Caching — Mono.cache()와 중복 구독 방지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Mono.cache()`는 내부적으로 어떻게 결과를 캐싱하는가?
- 중복 구독 문제란 무엇이고 실제 서비스에서 어떤 피해를 주는가?
- Caffeine 캐시와 Reactor를 어떻게 통합하는가?
- `Mono.cache()`의 TTL 설정과 갱신 전략은 어떻게 구현하는가?
- Redis와 Reactor를 결합한 분산 캐시는 어떻게 구현하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"Mono는 subscribe할 때마다 실행된다"는 Cold Publisher의 특성이 캐싱과 결합하면 중복 실행 문제를 만듭니다. 동일한 `Mono`를 여러 곳에서 구독하면 DB나 외부 API가 불필요하게 여러 번 호출됩니다. `Mono.cache()`와 Caffeine, Redis를 Reactive 파이프라인과 올바르게 통합하면 캐시 미스 시에만 실제 작업을 수행하고 이후 구독은 캐시된 결과를 즉시 반환할 수 있습니다.

---

## 😱 흔한 실수 (Before — 중복 구독으로 인한 중복 실행)

```
실수 1: 동일 Mono를 여러 곳에서 구독

  Mono<Config> config = configService.loadConfig();

  config.subscribe(c -> processA(c));  // DB 쿼리 1번
  config.subscribe(c -> processB(c));  // DB 쿼리 또 1번!
  config.subscribe(c -> processC(c));  // DB 쿼리 또 1번!

  // Cold Publisher: subscribe마다 새 실행 → 3번 DB 쿼리
  // 원하는 것: DB 1번 조회 후 3곳에서 공유

실수 2: cache()를 잘못된 위치에 사용

  public Mono<Config> getConfig() {
      return configRepository.findGlobal()
          .cache();  // 호출마다 새 Mono 생성 후 cache → 공유 없음
  }

  // 올바른 방식: 하나의 Mono 인스턴스를 필드로 유지
  private final Mono<Config> cachedConfig = configRepository.findGlobal()
      .cache();  // 이 Mono를 재사용

실수 3: TTL 없는 cache() 사용으로 오래된 데이터 반환

  private final Mono<Config> config = configRepository.findGlobal()
      .cache();  // 영구 캐싱
  // → 설정 변경 시 앱 재시작 없이는 반영 안 됨
```

---

## ✨ 올바른 접근 (After — 올바른 Reactive 캐싱)

```
Mono.cache() 올바른 사용:

  // 필드 레벨에서 단일 인스턴스
  private final Mono<Config> cachedConfig;

  public ConfigService(ConfigRepository repo) {
      this.cachedConfig = repo.findGlobal()
          .cache(Duration.ofMinutes(10));  // 10분 TTL
      // 이 Mono가 처음 subscribe 시 DB 조회
      // 이후 subscribe는 캐시에서 즉시 반환
  }

  public Mono<Config> getConfig() {
      return cachedConfig;  // 동일 인스턴스 반환
  }

Caffeine + Reactor 통합:
  Cache<String, Mono<User>> userCache = Caffeine.newBuilder()
      .expireAfterWrite(Duration.ofMinutes(5))
      .maximumSize(10_000)
      .build();

  public Mono<User> findUser(Long id) {
      return userCache.get(id.toString(), key ->
          userRepository.findById(id).cache()
      );
  }
```

---

## 🔬 내부 동작 원리

### 1. Mono.cache() 내부 동작

```
Mono.cache()가 하는 일:

  초기 상태: CacheMono 생성, 결과 없음

  첫 번째 subscribe():
    → 내부적으로 MonoProcessor (Sinks.One) 생성
    → 원본 Mono 구독 시작
    → 결과(onNext) 도착 → MonoProcessor에 저장
    → 현재 subscriber에게 전달

  두 번째 subscribe() (결과 이미 있음):
    → MonoProcessor에서 즉시 캐시된 값 반환
    → 원본 Mono 재실행 없음

  에러 발생 시:
    → 에러도 캐싱 (!)
    → 이후 구독도 동일 에러 반환
    → 원본 재실행 없음

  TTL 만료 시 (Duration 지정한 경우):
    → 캐시 무효화 → 다음 subscribe 시 원본 재실행

  다중 동시 구독 (캐시 없는 상태):
    → 첫 번째 subscribe: 원본 실행 시작
    → 두 번째 subscribe: 첫 번째 결과 대기 (원본 중복 실행 안 함)
    → 원본 완료 → 두 subscribe 모두 동일 결과 수신
    → "Let One In" 패턴으로 중복 실행 방지

내부 구현:
  MonoCache<T> extends Mono<T>:
    volatile Object value;  // 캐시된 값 (null = 미실행)
    List<MonoCacheSubscriber> subscribers; // 대기 중인 구독자들
    
    subscribe() 시:
      if (value != null) → 즉시 캐시 반환
      else → subscribers에 추가, 아직 없으면 원본 구독 시작
```

### 2. 중복 구독 문제 시나리오

```
실제 문제 시나리오:

  @GetMapping("/dashboard")
  public Mono<Dashboard> getDashboard() {
      Mono<UserProfile> profile = userService.getCurrentUser();
      Mono<List<Order>> orders = orderService.getRecentOrders();
      Mono<List<Notification>> notifications = notifService.getUnread();

      return Mono.zip(
          profile,
          orders,
          notifications
      ).map(tuple -> Dashboard.of(...));
  }

  // 문제: 위 코드는 문제없음 (각 Mono 한 번만 구독)

  // 문제 발생 시나리오:
  Mono<UserProfile> profile = userService.getCurrentUser();

  return profile.flatMap(user ->
      orderService.getOrders(user.getId())
          .doOnNext(orders ->
              // doOnNext 내에서 profile을 또 구독!
              profile.subscribe(p -> audit(p, orders))  // 두 번째 구독!
          )
          .then()
  );
  // → userService.getCurrentUser() 두 번 호출 → DB 쿼리 2번
```

### 3. Caffeine + Reactor 완전 통합

```java
@Service
@RequiredArgsConstructor
public class CachingUserService {

    private final UserRepository userRepository;
    private final Cache<Long, Mono<User>> userCache;

    @Bean
    public Cache<Long, Mono<User>> userCache() {
        return Caffeine.newBuilder()
            .expireAfterWrite(Duration.ofMinutes(5))
            .maximumSize(10_000)
            .removalListener((key, value, cause) ->
                log.debug("캐시 제거: userId={}, reason={}", key, cause)
            )
            .build();
    }

    public Mono<User> findById(Long userId) {
        // Caffeine 캐시 + Mono.cache() 조합
        return userCache.get(userId,
            id -> userRepository.findById(id)
                .switchIfEmpty(Mono.error(new UserNotFoundException(id)))
                .cache()  // Mono 자체를 캐싱 → 동시 요청 중복 실행 방지
        );
    }

    // 캐시 무효화
    public Mono<Void> invalidateUser(Long userId) {
        userCache.invalidate(userId);
        return Mono.empty();
    }

    // 업데이트 후 캐시 갱신
    public Mono<User> updateUser(Long userId, UpdateUserRequest req) {
        return userRepository.update(userId, req)
            .doOnSuccess(updatedUser -> {
                // 업데이트된 값으로 캐시 갱신
                userCache.put(userId, Mono.just(updatedUser).cache());
            });
    }
}
```

### 4. TTL 기반 자동 갱신 전략

```java
// 전략 1: Mono.cache(Duration) — 단순 TTL
private final Mono<Config> config = configRepository.findGlobal()
    .cache(Duration.ofMinutes(10));
// 10분마다 캐시 무효화 → 다음 구독 시 DB 재조회

// 전략 2: 갱신 중 이전 캐시 서빙 (Stale-While-Revalidate)
@Service
public class RefreshableConfig {

    private final AtomicReference<Mono<Config>> cachedConfig =
        new AtomicReference<>();
    private volatile Instant lastRefresh = Instant.EPOCH;

    public Mono<Config> getConfig() {
        Instant now = Instant.now();
        Mono<Config> cached = cachedConfig.get();

        if (cached == null) {
            // 최초 로드
            Mono<Config> newMono = configRepository.findGlobal().cache();
            cachedConfig.set(newMono);
            lastRefresh = now;
            return newMono;
        }

        if (Duration.between(lastRefresh, now).toMinutes() >= 10) {
            // 갱신 시작 (이전 캐시 유지)
            lastRefresh = now;
            configRepository.findGlobal()
                .doOnSuccess(newConfig -> {
                    cachedConfig.set(Mono.just(newConfig).cache());
                    log.info("Config 갱신 완료");
                })
                .subscribe();  // fire-and-forget
        }

        return cached;  // 갱신 중에도 이전 캐시 반환
    }
}

// 전략 3: Schedulers.single()로 주기적 갱신
@Component
public class ScheduledCacheRefresher {

    @Scheduled(fixedDelay = 600_000)  // 10분마다
    public void refreshConfig() {
        configRepository.findGlobal()
            .subscribe(
                config -> {
                    cachedConfig.set(Mono.just(config).cache());
                    log.info("Config 주기적 갱신 완료");
                },
                err -> log.error("Config 갱신 실패", err)
            );
    }
}
```

### 5. Redis 분산 캐시 + Reactor

```java
@Service
@RequiredArgsConstructor
public class RedisBackedUserService {

    private final UserRepository userRepository;
    private final ReactiveRedisTemplate<String, User> redisTemplate;

    // Cache-Aside 패턴 (Reactive)
    public Mono<User> findById(Long userId) {
        String cacheKey = "user:" + userId;

        return redisTemplate.opsForValue()
            .get(cacheKey)                          // 1. Redis 캐시 확인
            .switchIfEmpty(
                userRepository.findById(userId)      // 2. 캐시 미스 → DB 조회
                    .flatMap(user ->
                        redisTemplate.opsForValue()
                            .set(cacheKey, user,    // 3. Redis에 저장 (5분)
                                Duration.ofMinutes(5))
                            .thenReturn(user)
                    )
            );
    }

    // 동시 요청 중복 실행 방지 (Redis SETNX 기반 뮤텍스)
    public Mono<User> findByIdWithMutex(Long userId) {
        String cacheKey = "user:" + userId;
        String lockKey = "lock:user:" + userId;

        return redisTemplate.opsForValue().get(cacheKey)
            .switchIfEmpty(
                redisTemplate.opsForValue()
                    .setIfAbsent(lockKey, "1", Duration.ofSeconds(5))
                    .flatMap(locked -> {
                        if (!locked) {
                            // 다른 요청이 이미 조회 중 → 재시도 (캐시 대기)
                            return Mono.delay(Duration.ofMillis(100))
                                .flatMap(d -> findByIdWithMutex(userId));
                        }
                        // 락 획득 → DB 조회 → 캐시 저장
                        return userRepository.findById(userId)
                            .flatMap(user ->
                                redisTemplate.opsForValue()
                                    .set(cacheKey, user, Duration.ofMinutes(5))
                                    .then(redisTemplate.delete(lockKey))
                                    .thenReturn(user)
                            );
                    })
            );
    }
}
```

---

## 💻 실전 코드

### 실험 1: cache()로 초기화 설정 캐싱

```java
@Service
public class AppConfigService {

    // 앱 시작 시 한 번만 DB 조회, 이후 캐시 반환
    private final Mono<AppConfig> config;

    public AppConfigService(AppConfigRepository repo) {
        this.config = repo.findAll()
            .collectMap(Config::getKey, Config::getValue)
            .map(AppConfig::fromMap)
            .cache();  // 영구 캐싱 (앱 재시작 전까지)
    }

    public Mono<AppConfig> getConfig() {
        return config;
    }

    public Mono<String> getConfigValue(String key) {
        return config.map(c -> c.get(key));
    }
}

// 사용 - 100번 호출해도 DB는 1번만 조회
@Test
void configCachingTest() {
    AtomicInteger dbCallCount = new AtomicInteger(0);

    AppConfigService svc = new AppConfigService(new TestRepo(dbCallCount));

    // 100번 구독
    Flux.range(1, 100)
        .flatMap(i -> svc.getConfig())
        .blockLast();

    assertThat(dbCallCount.get()).isEqualTo(1);  // DB는 딱 1번만
}
```

### 실험 2: StepVerifier로 cache() 동작 검증

```java
@Test
void cacheShouldPreventDuplicateExecution() {
    AtomicInteger execCount = new AtomicInteger(0);

    Mono<String> original = Mono.fromCallable(() -> {
        execCount.incrementAndGet();
        return "result";
    });

    Mono<String> cached = original.cache();

    // 3번 구독
    StepVerifier.create(cached).expectNext("result").verifyComplete();
    StepVerifier.create(cached).expectNext("result").verifyComplete();
    StepVerifier.create(cached).expectNext("result").verifyComplete();

    assertThat(execCount.get()).isEqualTo(1);  // 1번만 실행됨
}

@Test
void cacheShouldExpireAfterTTL() {
    AtomicInteger execCount = new AtomicInteger(0);

    Mono<String> cached = Mono.fromCallable(() -> {
        execCount.incrementAndGet();
        return "result";
    }).cache(Duration.ofMillis(100));

    cached.block();
    cached.block();  // 캐시 히트 (총 1번)

    Thread.sleep(150);  // TTL 만료

    cached.block();  // 캐시 미스 → 재실행 (총 2번)
    cached.block();  // 캐시 히트 (총 2번)

    assertThat(execCount.get()).isEqualTo(2);
}
```

---

## 📊 성능 비교

```
캐싱 전략별 성능:

캐싱 없음:
  요청마다 DB 조회: ~10ms
  100 req/s → 100 DB 쿼리/s

Mono.cache() (메모리):
  최초: ~10ms (DB 조회)
  이후: ~수ns (메모리 접근)
  100 req/s → 1 DB 쿼리 (최초만)

Caffeine (로컬 메모리):
  히트: ~수μs
  미스: ~10ms (DB) + Caffeine 오버헤드 (무시 가능)
  TTL 만료 시: 재조회

Redis (분산 캐시):
  히트: ~1ms
  미스: ~10ms (DB) + 1ms (Redis 쓰기)
  수평 확장 가능

선택 기준:
  단일 서버, 앱 생명주기 캐시: Mono.cache()
  단일 서버, TTL 기반: Caffeine
  다수 서버, 캐시 공유: Redis

캐시 히트율 중요성:
  히트율 90%, 쿼리 10ms:
    평균 레이턴시 = 90% × 0.001ms + 10% × 10ms = ~1ms (10배 개선)
  히트율 50%:
    평균 레이턴시 = 50% × 0.001ms + 50% × 10ms = ~5ms (2배 개선)
```

---

## ⚖️ 트레이드오프

```
Mono.cache() 트레이드오프:
  장점: 단순, 추가 인프라 없음, 동시 요청 중복 실행 방지
  단점: JVM 메모리 사용, 단일 서버, 갱신 어려움

Caffeine 트레이드오프:
  장점: TTL, 크기 제한, eviction 정책, 통계
  단점: 로컬 캐시 (서버 간 불일치 가능)

Redis 트레이드오프:
  장점: 분산 캐시, 서버 재시작 후 유지, 다수 서버 공유
  단점: 네트워크 레이턴시, 추가 인프라, 직렬화 필요

캐시 무효화:
  TTL 기반: 단순, 오래된 데이터 가능성
  이벤트 기반: 즉각 무효화, 복잡성 증가
  → 대부분의 경우 TTL + 짧은 만료시간(1~5분)이 균형점

에러 캐싱 주의:
  Mono.cache()는 에러도 캐싱
  DB 일시 장애 시 에러가 캐싱되면 복구 후에도 에러 반환
  → 에러 발생 시 캐시 미스로 처리:
    .onErrorResume(e -> Mono.empty())로 에러를 빈 값으로 처리
    또는 .cache(Duration.ofSeconds(0))으로 에러는 캐시 안 함
```

---

## 📌 핵심 정리

```
Reactive Caching 핵심:

중복 구독 문제:
  Cold Publisher: subscribe마다 새 실행
  동일 Mono를 여러 곳에서 구독 → 중복 실행
  해결: Mono.cache()로 결과 캐싱

Mono.cache():
  첫 구독: 원본 실행 + 결과 저장
  이후 구독: 캐시에서 즉시 반환
  TTL 지정 가능: .cache(Duration.ofMinutes(10))
  에러도 캐싱됨 (주의)

올바른 사용 위치:
  필드로 단일 Mono 인스턴스 유지
  메서드마다 new Mono().cache() 하면 공유 안 됨

계층별 캐싱:
  앱 생명주기: Mono.cache() (메모리, 빠름)
  TTL 로컬: Caffeine (eviction, 크기 제한)
  분산: Redis (서버 간 공유)

갱신 전략:
  TTL 만료: 단순하지만 오래된 데이터 가능
  이벤트 기반: 즉각적이지만 복잡
  Stale-While-Revalidate: 갱신 중에도 이전 캐시 서빙
```

---

## 🤔 생각해볼 문제

**Q1.** `Mono.cache()`와 `Mono.share()`의 차이는 무엇인가요?

<details>
<summary>해설 보기</summary>

**`Mono.cache()`**: 결과를 영구적으로(또는 TTL까지) 저장. 이후 모든 구독에서 캐시된 값을 반환. 원본 구독 완료 후에 구독해도 캐시된 값을 받음.

**`Mono.share()`** (`= publish().refCount(1)`): Hot Publisher 변환. 진행 중인 구독에 참여. 원본이 완료된 후 구독하면 새로운 실행 시작(캐시 없음).

```java
Mono<String> cached = longRunningMono.cache();
Mono<String> shared = longRunningMono.share();

// cache(): 원본 완료 후 구독해도 캐시된 값 받음
cached.subscribe(v -> log.info("A: {}", v));  // 원본 실행
Thread.sleep(2000);
cached.subscribe(v -> log.info("B: {}", v));  // 캐시에서 즉시 반환

// share(): 원본 완료 후 구독하면 새로 실행
shared.subscribe(v -> log.info("A: {}", v));  // 원본 실행
Thread.sleep(2000);
shared.subscribe(v -> log.info("B: {}", v));  // 원본 재실행!
```

캐싱 목적: `Mono.cache()`. 실시간 공유 목적: `Mono.share()`.

</details>

---

**Q2.** `Caffeine.build()` 캐시에 `Mono<User>`를 값으로 저장할 때 메모리 관리는 어떻게 되나요?

<details>
<summary>해설 보기</summary>

Caffeine 캐시에 `Mono<User>`를 저장하면, Caffeine의 `expireAfterWrite`나 `maximumSize` eviction이 `Mono` 객체에 적용됩니다. `Mono`가 evict되면 GC 대상이 됩니다.

단, `Mono<User>` 내부에 캐시된 `User` 객체가 있다면, `Mono`가 캐시에서 제거될 때까지 `User` 객체도 메모리에 유지됩니다.

```java
// Mono 자체가 Caffeine에 의해 크기/TTL로 관리됨
Cache<Long, Mono<User>> cache = Caffeine.newBuilder()
    .expireAfterWrite(Duration.ofMinutes(5))   // Mono 객체 5분 후 제거
    .maximumSize(10_000)                        // 최대 10,000개 Mono
    .build();

// User 객체 크기 추정: 각 Mono<User>가 ~수백 바이트 User 캐싱
// 10,000개 × 500 바이트 = ~5MB (작음)
```

`Mono.cache()` 내부의 결과값도 `Mono` 객체가 참조하는 동안 GC되지 않습니다. Caffeine의 `softValues()`를 사용하면 메모리 압박 시 자동으로 제거할 수 있습니다.

</details>

---

**Q3.** 분산 환경에서 여러 서버가 Redis 캐시를 공유할 때, 한 서버에서 데이터를 업데이트하면 다른 서버의 로컬 Caffeine 캐시는 어떻게 무효화하나요?

<details>
<summary>해설 보기</summary>

Redis Pub/Sub으로 캐시 무효화 이벤트를 브로드캐스트합니다:

```java
// 업데이트 서버: Redis에 무효화 이벤트 발행
@Service
public class UserService {
    public Mono<User> updateUser(Long userId, UpdateRequest req) {
        return userRepository.update(userId, req)
            .flatMap(user ->
                // Redis 캐시 삭제
                redisTemplate.delete("user:" + userId)
                // + 모든 서버에 로컬 캐시 무효화 이벤트 발행
                .then(redisTemplate.convertAndSend(
                    "cache:invalidate:user", userId.toString()
                ))
                .thenReturn(user)
            );
    }
}

// 모든 서버: 무효화 이벤트 수신 → Caffeine 캐시 제거
@Component
public class CacheInvalidationListener {
    @PostConstruct
    public void listen() {
        messageContainer.receive(ChannelTopic.of("cache:invalidate:user"))
            .subscribe(msg -> {
                Long userId = Long.parseLong(msg.getBody());
                localUserCache.invalidate(userId);
                log.debug("로컬 캐시 무효화: userId={}", userId);
            });
    }
}
```

이 패턴이 바로 "Local Cache + Redis Pub/Sub"으로 구현하는 캐시 동기화입니다. 업데이트 이후 짧은 시간 동안 다른 서버에서 오래된 값을 반환할 수 있습니다(Eventually Consistent).

</details>

---

<div align="center">

**[⬅️ 이전: 운영 중 발생하는 문제 패턴](./02-production-problem-patterns.md)** | **[홈으로 🏠](../README.md)** | **[다음: 마이크로서비스에서 WebFlux ➡️](./04-microservice-webflux.md)**

</div>
