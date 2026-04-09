# Redis Reactive — ReactiveRedisTemplate와 Pub/Sub

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Lettuce는 왜 논블로킹이고, Jedis는 왜 블로킹인가?
- `ReactiveRedisTemplate`과 `RedisTemplate`은 어떻게 다른가?
- Reactive Pub/Sub에서 메시지가 `Flux<Message>`로 어떻게 스트리밍되는가?
- WebFlux + Redis 조합으로 실시간 알림을 어떻게 구현하는가?
- Redis 연결이 끊어졌을 때 Reactive 스트림은 어떻게 복구되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

WebFlux 애플리케이션에서 캐싱, 세션 관리, 실시간 알림에 Redis를 자주 사용합니다. `RedisTemplate`은 동기 블로킹이므로 WebFlux EventLoop와 함께 쓰면 블로킹 문제가 생깁니다. `ReactiveRedisTemplate` + Lettuce의 조합은 Redis 통신을 완전 논블로킹으로 처리합니다. Pub/Sub을 `Flux`로 스트리밍하면 SSE나 WebSocket과 자연스럽게 연결되어 실시간 기능을 구현할 수 있습니다.

---

## 😱 흔한 실수 (Before — Redis를 WebFlux에서 잘못 사용할 때)

```
실수 1: WebFlux에서 RedisTemplate (블로킹) 사용

  @GetMapping("/user/{id}")
  public Mono<User> getUser(@PathVariable Long id) {
      User cached = redisTemplate.opsForValue()
          .get("user:" + id);  // 블로킹 Redis I/O!
      // → EventLoop 스레드에서 Redis 소켓 대기
      // → 모든 채널 지연
  }

실수 2: Pub/Sub을 subscribe()로 받아 EventLoop 블로킹

  // Jedis Pub/Sub는 블로킹 구독 루프
  jedis.subscribe(new JedisPubSub() {
      void onMessage(String channel, String message) {
          // 블로킹 방식으로 메시지 수신
      }
  }, "channel");  // 이 메서드는 반환되지 않음! (블로킹)

실수 3: ReactiveRedisTemplate의 Mono를 subscribe 없이 방치

  @PostMapping("/notify")
  public Mono<Void> notify(String message) {
      reactiveRedisTemplate.convertAndSend("channel", message);
      // subscribe() 없으면 실제 발행 안 됨!
      return Mono.empty();
  }
```

---

## ✨ 올바른 접근 (After — Reactive Redis 올바른 사용)

```
ReactiveRedisTemplate 기본 패턴:

@Autowired ReactiveRedisTemplate<String, String> redisTemplate;

// 조회 (캐시)
Mono<String> value = redisTemplate.opsForValue().get("key");

// 저장
Mono<Boolean> set = redisTemplate.opsForValue()
    .set("key", "value", Duration.ofMinutes(10));

// Pub/Sub 발행
Mono<Long> published = redisTemplate.convertAndSend("channel", "message");

// Pub/Sub 구독 (Flux 스트리밍)
ReactiveSubscription subscription = redisTemplate.listenTo(
    ChannelTopic.of("channel")
);
Flux<Message<String, String>> messages = subscription.receive();
```

---

## 🔬 내부 동작 원리

### 1. Lettuce vs Jedis — 논블로킹의 차이

```
Jedis (블로킹):
  Redis 명령 실행 시:
    Socket.getOutputStream().write(command);  // 명령 전송
    Socket.getInputStream().read(response);   // 응답 대기 (블로킹!)
    return response;
  → 스레드가 Redis 응답 대기 중 점유 (JDBC와 동일한 문제)

Lettuce (논블로킹):
  Redis 명령 실행 시:
    Netty SocketChannel.write(command);  // 명령 비동기 전송
    CompletableFuture<Response> future; // 즉시 반환
  
  Netty EventLoop:
    Redis 소켓 감시 → 응답 도착 → Future 완료
    → ReactiveRedisTemplate의 Mono/Flux에 onNext 신호
  
  → 스레드는 Redis 대기 없이 즉시 다른 작업 처리 가능
  → WebFlux EventLoop와 완벽히 호환

Lettuce 연결 전략:
  단일 연결 (기본): 하나의 TCP 연결로 모든 명령 멀티플렉싱
    → Redis의 싱글 스레드 모델과 잘 맞음
  연결 풀링: 여러 연결 사용 (트랜잭션, Blocking 명령 필요 시)

Spring Boot 자동 설정:
  spring.redis.host, port 설정 시
  → LettuceConnectionFactory 자동 생성
  → ReactiveRedisTemplate<Object, Object> 빈 등록
  → 별도 설정 없이 논블로킹 Redis 사용 가능
```

### 2. ReactiveRedisTemplate 주요 API

```
RedisTemplate vs ReactiveRedisTemplate:

  RedisTemplate:
    String value = template.opsForValue().get("key");  // 동기 반환
    
  ReactiveRedisTemplate:
    Mono<String> value = template.opsForValue().get("key");  // Mono 반환

주요 자료구조별 API:

  // String (Key-Value)
  ReactiveValueOperations<K, V> ops = template.opsForValue();
  ops.get("key")                    // Mono<V>
  ops.set("key", value)             // Mono<Boolean>
  ops.set("key", value, ttl)        // Mono<Boolean> (TTL 설정)
  ops.increment("counter")          // Mono<Long>
  ops.getAndSet("key", newValue)    // Mono<V> (이전 값 반환)

  // Hash
  ReactiveHashOperations<K, HK, HV> hash = template.opsForHash();
  hash.put("user:1", "name", "Alice")   // Mono<Boolean>
  hash.get("user:1", "name")            // Mono<HV>
  hash.entries("user:1")                // Flux<Map.Entry<HK, HV>>
  hash.putAll("user:1", map)            // Mono<Boolean>

  // List
  ReactiveListOperations<K, V> list = template.opsForList();
  list.leftPush("queue", item)         // Mono<Long>
  list.rightPop("queue")               // Mono<V>
  list.range("history", 0, 9)          // Flux<V> (최근 10개)

  // Set
  ReactiveSetOperations<K, V> set = template.opsForSet();
  set.add("tags", "java", "reactive")  // Mono<Long>
  set.members("tags")                  // Flux<V>
  set.isMember("tags", "java")         // Mono<Boolean>

  // Sorted Set (ZSet)
  ReactiveZSetOperations<K, V> zset = template.opsForZSet();
  zset.add("ranking", member, score)          // Mono<Boolean>
  zset.reverseRange("ranking", 0, 9)          // Flux<V> (상위 10개)
  zset.reverseRangeWithScores("ranking", 0, 9) // Flux<ZSetOperations.TypedTuple>
```

### 3. Reactive Pub/Sub 구현

```
Pub/Sub 아키텍처:
  Publisher (발행자): Redis 채널에 메시지 전송
  Subscriber (구독자): 채널 구독, 메시지 수신

발행 (Publishing):
  // convertAndSend: 메시지를 직렬화하여 채널에 발행
  Mono<Long> recipients = reactiveRedisTemplate
      .convertAndSend("notification:user:123", "새 주문이 도착했습니다");
  // Long: 수신한 구독자 수

구독 (Subscribing):
  ReactiveRedisMessageListenerContainer container =
      new ReactiveRedisMessageListenerContainer(connectionFactory);

  // 채널 구독 → Flux<Message>
  Flux<Message<String, String>> messages = container.receive(
      ChannelTopic.of("notification:user:123")
  );

  // 패턴 구독 (* 와일드카드)
  Flux<Message<String, String>> patternMessages = container.receive(
      PatternTopic.of("notification:user:*")
  );
  // notification:user:1, notification:user:2 등 모두 수신

Message<K, V> 구조:
  message.getChannel()  // 채널 이름
  message.getBody()     // 메시지 본문

연결 끊김 처리:
  container.receive()가 반환하는 Flux는 연결 끊김 시 onError 발생
  → retryWhen으로 자동 재연결:
  container.receive(topic)
      .retryWhen(Retry.backoff(Long.MAX_VALUE, Duration.ofSeconds(1))
          .maxBackoff(Duration.ofSeconds(30))
      )
```

### 4. WebFlux + Redis Pub/Sub — 실시간 알림

```
SSE + Redis Pub/Sub 조합:
  1. 사용자 A가 SSE 연결 (/notifications/stream)
  2. 서버: Redis 채널 "notification:userA" 구독 시작
  3. 다른 서비스에서 Redis에 메시지 발행
  4. 서버: Redis 메시지 수신 → SSE로 사용자 A에게 전달

  @GetMapping(value = "/notifications/stream",
              produces = TEXT_EVENT_STREAM_VALUE)
  public Flux<ServerSentEvent<Notification>> streamNotifications(
          @AuthenticationPrincipal Mono<UserDetails> user) {
      return user.flatMapMany(u ->
          notificationService.subscribeToNotifications(u.getUsername())
              .map(notification -> ServerSentEvent.<Notification>builder()
                  .id(String.valueOf(notification.getId()))
                  .event("notification")
                  .data(notification)
                  .build()
              )
      );
  }

NotificationService:
  public Flux<Notification> subscribeToNotifications(String userId) {
      String channel = "notification:" + userId;
      return messageContainer.receive(ChannelTopic.of(channel))
          .map(msg -> objectMapper.readValue(msg.getBody(), Notification.class))
          .doOnCancel(() ->
              log.info("알림 구독 취소: {}", userId)
          );
  }

  public Mono<Void> sendNotification(String userId, Notification notification) {
      String channel = "notification:" + userId;
      return reactiveRedisTemplate
          .convertAndSend(channel, objectMapper.writeValueAsString(notification))
          .then();
  }
```

### 5. Redis 캐시 — Reactive 패턴

```
Reactive 캐시 패턴 (Cache-Aside):

  public Mono<User> findUser(Long userId) {
      String cacheKey = "user:" + userId;

      return reactiveRedisTemplate.opsForValue()
          .get(cacheKey)                          // 1. 캐시 조회
          .switchIfEmpty(
              userRepository.findById(userId)      // 2. DB 조회 (캐시 미스)
                  .flatMap(user ->
                      reactiveRedisTemplate.opsForValue()
                          .set(cacheKey, user,     // 3. 캐시 저장 (10분 TTL)
                              Duration.ofMinutes(10))
                          .thenReturn(user)
                  )
          );
  }

  // 캐시 무효화
  public Mono<Void> evictUser(Long userId) {
      return reactiveRedisTemplate.delete("user:" + userId).then();
  }

Hash로 부분 필드 캐싱:
  // 사용자 객체의 일부 필드만 캐시 (자주 변경되지 않는 필드)
  public Mono<UserProfile> getUserProfile(Long userId) {
      String hashKey = "user:profile:" + userId;
      return reactiveRedisTemplate.opsForHash()
          .entries(hashKey)
          .collectMap(Map.Entry::getKey, Map.Entry::getValue)
          .filter(map -> !map.isEmpty())
          .switchIfEmpty(
              userRepository.findById(userId)
                  .flatMap(user -> {
                      Map<String, String> profile = Map.of(
                          "name", user.getName(),
                          "email", user.getEmail()
                      );
                      return reactiveRedisTemplate.opsForHash()
                          .putAll(hashKey, profile)
                          .then(reactiveRedisTemplate.expire(hashKey,
                              Duration.ofHours(1)))
                          .thenReturn(profile);
                  })
          )
          .map(UserProfile::from);
  }
```

---

## 💻 실전 코드

### 실험 1: 완전한 알림 서비스

```java
@Service
@RequiredArgsConstructor
public class NotificationService {

    private final ReactiveRedisTemplate<String, String> redisTemplate;
    private final ReactiveRedisMessageListenerContainer messageContainer;
    private final ObjectMapper objectMapper;

    // 알림 발송
    public Mono<Void> send(String userId, Notification notification) {
        try {
            String payload = objectMapper.writeValueAsString(notification);
            return redisTemplate
                .convertAndSend("notification:" + userId, payload)
                .doOnNext(count ->
                    log.debug("알림 발송: userId={}, receivers={}", userId, count)
                )
                .then();
        } catch (JsonProcessingException e) {
            return Mono.error(e);
        }
    }

    // 알림 구독 (SSE와 연결)
    public Flux<Notification> subscribe(String userId) {
        return messageContainer
            .receive(ChannelTopic.of("notification:" + userId))
            .map(msg -> {
                try {
                    return objectMapper.readValue(
                        msg.getBody(), Notification.class);
                } catch (JsonProcessingException e) {
                    throw new RuntimeException(e);
                }
            })
            .retryWhen(Retry.backoff(Long.MAX_VALUE, Duration.ofSeconds(1))
                .maxBackoff(Duration.ofSeconds(30))
                .doBeforeRetry(s ->
                    log.warn("Redis 구독 재연결 시도: {}", userId))
            )
            .doOnCancel(() ->
                log.info("알림 구독 해제: {}", userId)
            );
    }

    // 브로드캐스트 (모든 사용자에게)
    public Mono<Long> broadcast(Notification notification) {
        try {
            String payload = objectMapper.writeValueAsString(notification);
            return redisTemplate.convertAndSend("notification:broadcast", payload);
        } catch (JsonProcessingException e) {
            return Mono.error(e);
        }
    }
}
```

### 실험 2: Reactive 분산 락

```java
@Component
@RequiredArgsConstructor
public class ReactiveDistributedLock {

    private final ReactiveStringRedisTemplate redisTemplate;

    // 분산 락 획득 (SET NX EX)
    public Mono<Boolean> acquire(String lockKey, String lockValue, Duration ttl) {
        return redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, ttl);
        // SET lockKey lockValue NX EX {ttl.seconds}
        // NX: 없을 때만 설정
        // → true: 락 획득, false: 이미 락 있음
    }

    // 락 해제 (Lua 스크립트로 원자적 처리)
    public Mono<Boolean> release(String lockKey, String lockValue) {
        String script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
        return redisTemplate.execute(
            RedisScript.of(script, Long.class),
            List.of(lockKey),
            lockValue
        ).map(result -> result != null && result == 1L).next();
    }

    // 락 사용 패턴
    public <T> Mono<T> withLock(String resource, Duration ttl,
                                 Mono<T> operation) {
        String lockKey = "lock:" + resource;
        String lockValue = UUID.randomUUID().toString();

        return acquire(lockKey, lockValue, ttl)
            .flatMap(acquired -> {
                if (!acquired) {
                    return Mono.error(
                        new LockNotAcquiredException(resource));
                }
                return operation
                    .doFinally(signal ->
                        release(lockKey, lockValue).subscribe()
                    );
            });
    }
}
```

### 실험 3: 실시간 채팅방 (Redis Pub/Sub + WebSocket)

```java
@Component
@RequiredArgsConstructor
public class ChatHandler implements WebSocketHandler {

    private final ReactiveRedisTemplate<String, String> redisTemplate;
    private final ReactiveRedisMessageListenerContainer container;

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        String roomId = extractRoomId(session);
        String userId = extractUserId(session);
        String channel = "chat:" + roomId;

        // Inbound: 클라이언트 메시지 → Redis 발행
        Mono<Void> inbound = session.receive()
            .map(WebSocketMessage::getPayloadAsText)
            .flatMap(text -> {
                String message = formatMessage(userId, text);
                return redisTemplate.convertAndSend(channel, message)
                    .then();
            })
            .then();

        // Outbound: Redis 구독 → 클라이언트 전송
        Flux<WebSocketMessage> outbound = container
            .receive(ChannelTopic.of(channel))
            .map(msg -> session.textMessage(msg.getBody()));

        return Mono.zip(
            inbound,
            session.send(outbound)
        )
        .doFinally(signal ->
            log.info("채팅 세션 종료: userId={}, room={}", userId, roomId)
        )
        .then();
    }
}
```

---

## 📊 성능 비교

```
Jedis(블로킹) vs Lettuce(논블로킹) 비교:

시나리오: Redis 조회 1ms, 동시 1000 요청

Jedis + WebFlux (잘못된 조합):
  EventLoop에서 Redis 블로킹 → 모든 채널 지연
  효과적 처리량: EventLoop 수 × (1000ms / 1ms) = 16,000 req/s
  실제: EventLoop 블로킹으로 훨씬 낮음

Jedis + WebFlux + boundedElastic (임시 해결):
  boundedElastic 스레드 80개 → 80,000 req/s
  단, 스레드 수 한계

Lettuce + ReactiveRedisTemplate:
  EventLoop 논블로킹 → Redis 1ms 대기 중 다른 요청 처리
  처리량: 이론적으로 EventLoop 수 × (1000ms / 1ms) × 멀티플렉싱
  실제: ~수십만 req/s (Redis 단순 조회 기준)

단일 연결 멀티플렉싱:
  Lettuce 기본: 1 TCP 연결로 다수 명령 파이프라이닝
  Redis 응답 시간 1ms, 1000 동시 요청:
  → 파이프라인으로 한 번에 1000개 전송, 응답 순서대로 처리
  → 연결 오버헤드 최소화

Pub/Sub 처리량:
  Redis: 초당 수백만 메시지 처리 가능
  Reactive 구독자 1만 명: SSE × 1만 = Redis 메시지 1 → SSE 1만 건 팬아웃
  → Redis가 아닌 애플리케이션의 SSE 처리가 병목
```

---

## ⚖️ 트레이드오프

```
Redis Pub/Sub vs Redis Streams:
  Pub/Sub:
    장점: 간단, 실시간 전달
    단점: 오프라인 구독자 메시지 유실, 재전송 없음
    적합: 실시간 알림 (유실 허용)

  Redis Streams (XADD/XREAD):
    장점: 메시지 영속화, 소비자 그룹, 재처리 가능
    단점: 복잡한 API
    적합: 이벤트 소싱, 메시지 유실 불가 시나리오
    Reactive: ReactiveRedisTemplate.opsForStream()

단일 연결 vs 연결 풀:
  단일 연결 (Lettuce 기본):
    장점: 연결 오버헤드 없음, Redis 싱글스레드와 최적 조합
    단점: BLPOP 등 블로킹 명령 사용 불가
  연결 풀:
    장점: MULTI/EXEC 트랜잭션, 블로킹 명령 지원
    단점: 연결 수 증가

Cache-Aside vs Write-Through:
  Cache-Aside: 읽기 시 캐시 미스 → DB 조회 → 캐시 저장
    → 구현 간단, 캐시 무효화 시 DB 부하 스파이크 가능
  Write-Through: 쓰기 시 DB + 캐시 동시 업데이트
    → 캐시 항상 최신, 쓰기 지연 증가
```

---

## 📌 핵심 정리

```
Redis Reactive 핵심:

Lettuce vs Jedis:
  Jedis: 블로킹 소켓 → WebFlux와 충돌
  Lettuce: Netty 기반 논블로킹 → WebFlux와 완벽 호환

ReactiveRedisTemplate:
  RedisTemplate의 Reactive 버전
  모든 메서드가 Mono<T> 또는 Flux<T> 반환
  Spring Boot 자동 설정: LettuceConnectionFactory 기반

Pub/Sub:
  convertAndSend(): 발행 → Mono<Long> (수신자 수)
  listenTo(): 구독 → Flux<Message>
  retryWhen으로 자동 재연결 구현 필수

실시간 패턴:
  Redis Pub/Sub → Flux<Message> → SSE 또는 WebSocket
  알림: 채널당 1명
  브로드캐스트: 패턴 구독 or 공유 채널

캐시 패턴:
  get() → switchIfEmpty(DB 조회 → set()) → Cache-Aside
  TTL 설정 필수 (만료 없으면 메모리 누수)
```

---

## 🤔 생각해볼 문제

**Q1.** Redis Pub/Sub에서 서버가 재시작되면 구독 중이던 SSE 클라이언트는 어떻게 되나요?

<details>
<summary>해설 보기</summary>

서버 재시작 시:
1. 모든 Redis 구독이 종료됩니다 (Lettuce 연결 끊김).
2. SSE 연결 유지 중인 클라이언트는 `onError` 또는 연결 종료를 받습니다.
3. 브라우저의 `EventSource`는 자동으로 재연결을 시도합니다 (기본 3초 후).
4. 재연결 성공 시 새 SSE 스트림이 시작됩니다.

서버 측 복구를 위한 패턴:
```java
// Redis 연결 복구 후 구독 재시작
container.receive(ChannelTopic.of(channel))
    .retryWhen(Retry.backoff(Long.MAX_VALUE, Duration.ofSeconds(1))
        .maxBackoff(Duration.ofSeconds(30))
    )
```

재연결 시 누락된 메시지를 복구하려면 Redis Streams(`XREADGROUP`)를 사용하거나, 마지막 수신 메시지 ID를 클라이언트가 `Last-Event-ID`로 전달하는 SSE 표준을 활용합니다.

</details>

---

**Q2.** `ReactiveRedisTemplate`에서 직렬화/역직렬화 설정은 어떻게 하나요?

<details>
<summary>해설 보기</summary>

기본 `ReactiveRedisTemplate<Object, Object>`는 JDK 직렬화를 사용합니다. JSON 직렬화를 위해 Jackson 설정이 필요합니다:

```java
@Bean
public ReactiveRedisTemplate<String, Object> reactiveRedisTemplate(
        ReactiveRedisConnectionFactory factory) {
    ObjectMapper mapper = new ObjectMapper()
        .registerModule(new JavaTimeModule());

    Jackson2JsonRedisSerializer<Object> serializer =
        new Jackson2JsonRedisSerializer<>(mapper, Object.class);

    RedisSerializationContext<String, Object> context =
        RedisSerializationContext.<String, Object>newSerializationContext()
            .key(RedisSerializer.string())    // 키: 문자열
            .value(serializer)                 // 값: JSON
            .hashKey(RedisSerializer.string()) // Hash 키: 문자열
            .hashValue(serializer)             // Hash 값: JSON
            .build();

    return new ReactiveRedisTemplate<>(factory, context);
}
```

타입별 템플릿 분리 권장:
```java
// 문자열 전용
ReactiveStringRedisTemplate stringTemplate; // 내장 자동 설정

// 도메인 객체 전용
ReactiveRedisTemplate<String, User> userTemplate;
```

</details>

---

**Q3.** Redis 클러스터 환경에서 Pub/Sub은 어떻게 동작하나요?

<details>
<summary>해설 보기</summary>

Redis 클러스터에서 Pub/Sub은 **클러스터 전체로 전파**됩니다. 메시지를 어느 노드에 발행해도 모든 구독자가 수신합니다. 단, 각 노드에 직접 연결된 구독자에게만 전달되므로 클러스터 내 모든 노드에 구독이 설정되어 있어야 합니다.

Lettuce는 클러스터 Pub/Sub을 자동으로 처리합니다:
```yaml
spring:
  redis:
    cluster:
      nodes:
        - redis1:6379
        - redis2:6379
        - redis3:6379
```

클러스터 설정 시 `ReactiveRedisMessageListenerContainer`는 모든 클러스터 노드에 구독을 등록합니다.

**주의**: Redis 클러스터 Pub/Sub은 `MOVED` 오류가 발생하지 않으므로 일반 키-값 명령과 다르게 동작합니다. 구독 채널은 슬롯에 기반하지 않고 브로드캐스트됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: R2DBC 실전 패턴](./04-r2dbc-practical-patterns.md)** | **[홈으로 🏠](../README.md)**

</div>
