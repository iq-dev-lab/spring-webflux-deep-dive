# R2DBC 실전 패턴 — N+1 문제와 배치 처리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Reactive 환경에서 N+1 문제는 왜 더 명시적이고, 어떻게 해결하는가?
- `DatabaseClient`로 복잡한 동적 쿼리를 작성하는 방법은?
- `r2dbc-pool`에서 연결 풀 고갈을 감지하고 모니터링하는 방법은?
- 배치 INSERT를 R2DBC에서 최적화하는 방법은?
- `IN` 조건으로 배치 로딩하여 N+1을 해결하는 패턴은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

R2DBC를 도입했는데 오히려 DB 호출이 N배 늘어나는 경험을 하는 경우가 많습니다. JPA의 지연 로딩이 있을 때는 "필요할 때 자동으로 가져온다"는 편리함이 있었지만, R2DBC에서는 연관 데이터 로딩이 모두 명시적이기 때문에 N+1을 코드에서 직접 봐야 합니다. 이 명시성은 장점이기도 하지만, 올바르게 최적화하지 않으면 심각한 성능 문제로 이어집니다.

---

## 😱 흔한 실수 (Before — N+1 문제 유발)

```
실수 1: flatMap으로 건별 연관 데이터 조회

  orderRepo.findAll()          // 주문 100개
      .flatMap(order ->
          userRepo.findById(order.userId())  // 건별 사용자 조회 100번!
              .map(user -> OrderDto.of(order, user))
      )
  // DB 호출: 1(findAll) + 100(findById×100) = 101번 → N+1 문제

실수 2: 리스트 순회로 개별 저장

  Flux.fromIterable(orderList)
      .flatMap(order -> orderRepo.save(order))  // 1건씩 100번 INSERT
  // → 100개 개별 트랜잭션 또는 100개 라운드트립
  // → 배치 처리보다 10배 이상 느릴 수 있음

실수 3: IN 조건 없이 반복 조회

  Flux<Long> userIds = orderFlux.map(Order::userId).distinct();
  userIds.flatMap(id -> userRepo.findById(id));
  // 여전히 ID 개수만큼 쿼리 발생
  // → WHERE id IN (1,2,3,...) 단 1번으로 해결 가능
```

---

## ✨ 올바른 접근 (After — 배치 로딩과 JOIN)

```
N+1 해결 전략:

1. JOIN 쿼리로 한 번에 조회 (가장 효율적)
   SELECT o.*, u.name FROM orders o JOIN users u ON o.user_id = u.id

2. IN 조건 배치 로딩 (JOIN 불가 시)
   WHERE id IN (:ids)  → 1번 쿼리로 N개 조회

3. Flux.flatMap 동시성 제한 + 캐시 (임시)
   flatMap(fn, 1)로 순차 처리 대신
   IN 조건으로 배치

배치 INSERT:
  Statement.add()로 배치 구성 → 1번 라운드트립
  또는 VALUES (...), (...), (...) 멀티로 INSERT
```

---

## 🔬 내부 동작 원리

### 1. R2DBC의 N+1 — 더 명시적인 이유

```
JPA N+1 (숨겨진 형태):
  List<Order> orders = orderRepo.findAll();  // SELECT * FROM orders
  for (Order o : orders) {
      o.getUser().getName();  // 지연 로딩 → SELECT * FROM users WHERE id=?
  }
  // N+1 문제가 코드에서 보이지 않음

R2DBC N+1 (명시적 형태):
  orderRepo.findAll()
      .flatMap(order ->
          userRepo.findById(order.userId())  // 코드에 명시적으로 보임!
      )
  // 개발자가 직접 호출하므로 N+1이 코드에 드러남
  // 보이는 문제를 해결하기 쉬움

R2DBC에서 N+1 발생 패턴:
  Pattern 1: flatMap + findById
    orders.flatMap(o -> userRepo.findById(o.userId()))
    → 주문 N개 → findById N번

  Pattern 2: flatMap + 개별 집계
    orders.flatMap(o -> reviewRepo.countByOrderId(o.id()))
    → 주문 N개 → count N번

  Pattern 3: 중첩 flatMap
    orders.flatMap(o ->
        items.flatMap(i -> ...)  // 이중 N+1
    )
```

### 2. IN 조건 배치 로딩 패턴

```
N+1 → 1+1 최적화 (IN 조건):

  // 잘못된 방식 (N+1)
  orderRepo.findAll()
      .flatMap(order -> userRepo.findById(order.userId()))

  // 올바른 방식 (IN 배치)
  orderRepo.findAll()
      .collectList()
      .flatMapMany(orders -> {
          // 1. 필요한 userId 수집
          Set<Long> userIds = orders.stream()
              .map(Order::userId)
              .collect(toSet());

          // 2. IN 조건으로 한 번에 조회
          return userRepo.findAllById(userIds)  // SELECT * FROM users WHERE id IN (...)
              .collectMap(User::id)             // Map<Long, User>로 변환
              .flatMapMany(userMap ->
                  Flux.fromIterable(orders)
                      .map(order -> OrderDto.of(
                          order,
                          userMap.get(order.userId())
                      ))
              );
      });

  DB 호출 비교:
    N+1: 1 + N번 (주문 100개 → 101번)
    배치: 1 + 1번 (주문 100개 → 2번)

  DatabaseClient로 IN 조건:
    List<Long> ids = orders.stream().map(Order::userId).toList();
    databaseClient.sql("SELECT * FROM users WHERE id IN (:ids)")
        .bind("ids", ids)
        .fetch().all()
        .map(User::from)
```

### 3. JOIN 쿼리로 단일 조회

```
가장 효율적인 방법 — SQL JOIN:

  databaseClient.sql("""
      SELECT
          o.id AS order_id,
          o.status,
          o.total,
          u.id AS user_id,
          u.name AS user_name,
          u.email
      FROM orders o
      INNER JOIN users u ON o.user_id = u.id
      WHERE o.status = :status
      ORDER BY o.id DESC
      LIMIT :limit OFFSET :offset
  """)
  .bind("status", status)
  .bind("limit", size)
  .bind("offset", (long) page * size)
  .fetch().all()
  .map(row -> OrderWithUser.builder()
      .orderId((Long) row.get("order_id"))
      .status((String) row.get("status"))
      .total((BigDecimal) row.get("total"))
      .userId((Long) row.get("user_id"))
      .userName((String) row.get("user_name"))
      .email((String) row.get("email"))
      .build()
  );

  DB 호출: 단 1번
  vs N+1의 101번
  → 100배 이상 효율적

One-to-Many JOIN (중복 행 처리):
  주문 1개에 아이템 N개 → JOIN 시 주문 정보 N번 중복
  → groupBy로 합산 필요

  databaseClient.sql("""
      SELECT o.id, o.status, oi.product_id, oi.quantity
      FROM orders o JOIN order_items oi ON o.id = oi.order_id
      WHERE o.user_id = :userId
  """)
  .bind("userId", userId)
  .fetch().all()
  .groupBy(row -> (Long) row.get("id"))  // 주문별 그룹
  .flatMap(group ->
      group.collectList()
          .map(rows -> OrderWithItems.from(group.key(), rows))
  )
```

### 4. 배치 INSERT/UPDATE 최적화

```
방법 1: Statement Batch (가장 효율적)

  databaseClient.inConnectionMany(connection -> {
      Statement stmt = connection.createStatement(
          "INSERT INTO orders (user_id, status, total) VALUES ($1, $2, $3)"
      );
      for (Order order : orders) {
          stmt.bind(0, order.userId())
              .bind(1, order.status())
              .bind(2, order.total())
              .add();  // 배치에 추가
      }
      return Flux.from(stmt.execute())
          .flatMap(result -> result.getRowsUpdated());
  });

방법 2: 멀티 VALUES INSERT (PostgreSQL, MySQL 지원)

  String sql = "INSERT INTO orders (user_id, status, total) VALUES " +
      IntStream.range(0, orders.size())
          .mapToObj(i -> String.format("($%d, $%d, $%d)",
              i*3+1, i*3+2, i*3+3))
          .collect(joining(", "));

  DatabaseClient.GenericExecuteSpec spec = databaseClient.sql(sql);
  for (int i = 0; i < orders.size(); i++) {
      spec = spec
          .bind(i*3,   orders.get(i).userId())
          .bind(i*3+1, orders.get(i).status())
          .bind(i*3+2, orders.get(i).total());
  }
  spec.fetch().rowsUpdated();

배치 크기 선택:
  너무 작음 (< 100): 배치 효과 미미
  너무 큼 (> 10,000): 단일 쿼리 크기 초과, 메모리 압박
  권장: 1,000 ~ 5,000건씩

방법 3: saveAll (간단하지만 비효율)
  orderRepo.saveAll(orders)
  // 개별 INSERT 반복 → 배치 아님
  // 빠른 구현 시 사용, 성능 중요하면 Statement Batch 사용
```

### 5. 연결 풀 모니터링과 튜닝

```
r2dbc-pool 메트릭 활성화:

  @Bean
  public ConnectionPool connectionPool(ConnectionFactory factory) {
      return new ConnectionPool(
          ConnectionPoolConfiguration.builder()
              .connectionFactory(factory)
              .maxSize(20)
              .metricsRecorder(new MicrometerPoolMetricsRecorder(meterRegistry))
              .build()
      );
  }

주요 메트릭:
  r2dbc.pool.acquired.connections    # 현재 사용 중인 연결 수
  r2dbc.pool.idle.connections        # 유휴 연결 수
  r2dbc.pool.pending.connections     # 연결 획득 대기 중
  r2dbc.pool.max.allocated.connections  # 총 할당 연결 수

연결 풀 고갈 징후:
  pending.connections > 0 지속 → max-size 증가 고려
  acquired = max-size 유지 → 쿼리 지연 확인

튜닝 포인트:
  max-size: DB 코어 수 × 2~4 (8코어 DB → 16~32)
  max-acquire-time: SLA의 절반 (응답 SLA 10초 → 5초)
  max-idle-time: DB의 idle timeout보다 짧게

느린 쿼리 감지:
  databaseClient 실행 시간 측정:
  Mono<Long> start = Mono.fromCallable(System::nanoTime);
  start.flatMap(t ->
      databaseClient.sql(sql).fetch().all()
          .collectList()
          .doOnSuccess(rows ->
              log.info("쿼리 시간: {}ms", (System.nanoTime() - t) / 1_000_000)
          )
  );
```

---

## 💻 실전 코드

### 실험 1: N+1 해결 — 배치 로딩 전체 패턴

```java
@Service
@RequiredArgsConstructor
public class OrderQueryService {

    private final OrderRepository orderRepo;
    private final UserRepository userRepo;
    private final DatabaseClient databaseClient;

    // N+1 발생 버전 (잘못된 방식)
    public Flux<OrderDto> findAllWithUserBad() {
        return orderRepo.findAll()
            .flatMap(order ->
                userRepo.findById(order.userId())  // N번 호출!
                    .map(user -> OrderDto.of(order, user))
            );
    }

    // 배치 로딩 버전 (올바른 방식)
    public Flux<OrderDto> findAllWithUserGood() {
        return orderRepo.findAll()
            .collectList()
            .flatMapMany(orders -> {
                if (orders.isEmpty()) return Flux.empty();

                Set<Long> userIds = orders.stream()
                    .map(Order::userId).collect(toSet());

                return userRepo.findAllById(userIds)
                    .collectMap(User::id)
                    .flatMapMany(userMap ->
                        Flux.fromIterable(orders)
                            .map(o -> OrderDto.of(o,
                                userMap.getOrDefault(o.userId(), User.empty())))
                    );
            });
        // DB 호출: 2번 (findAll + findAllById IN)
    }

    // JOIN 버전 (가장 효율적)
    public Flux<OrderDto> findAllWithUserJoin() {
        return databaseClient.sql("""
                SELECT o.id, o.status, o.total, u.name, u.email
                FROM orders o JOIN users u ON o.user_id = u.id
                ORDER BY o.id
            """)
            .fetch().all()
            .map(OrderDto::fromRow);
        // DB 호출: 1번
    }
}
```

### 실험 2: 대용량 배치 처리 — 페이지 단위

```java
@Service
public class OrderBatchProcessor {

    @Transactional
    public Mono<ProcessSummary> processAllPendingOrders() {
        AtomicLong processed = new AtomicLong(0);
        AtomicLong failed = new AtomicLong(0);

        return orderRepo.findByStatus("PENDING")
            .buffer(500)  // 500개씩 배치
            .flatMap(batch ->
                processBatch(batch)
                    .doOnNext(result -> {
                        processed.addAndGet(result.success());
                        failed.addAndGet(result.failure());
                    }),
                3  // 최대 3개 배치 동시 처리
            )
            .then(Mono.fromCallable(() ->
                new ProcessSummary(processed.get(), failed.get())
            ));
    }

    private Mono<BatchResult> processBatch(List<Order> batch) {
        return Flux.fromIterable(batch)
            .flatMap(order ->
                processOrder(order)
                    .map(result -> 1)
                    .onErrorResume(e -> {
                        log.error("주문 처리 실패: {}", order.id(), e);
                        return Mono.just(0);  // 개별 실패는 계속 진행
                    }),
                10  // 배치 내 10개 동시 처리
            )
            .reduce(new int[]{0, 0}, (acc, r) -> {
                acc[r == 1 ? 0 : 1]++;
                return acc;
            })
            .map(acc -> new BatchResult(acc[0], acc[1]));
    }
}
```

### 실험 3: 동적 정렬 + 페이징

```java
public Flux<Order> findOrders(OrderSearchCondition cond) {
    // 정렬 컬럼 화이트리스트 (SQL 인젝션 방지)
    Set<String> allowedSortColumns = Set.of("id", "status", "total", "created_at");
    String sortCol = allowedSortColumns.contains(cond.getSortBy())
        ? cond.getSortBy() : "id";
    String sortDir = "DESC".equalsIgnoreCase(cond.getSortDir()) ? "DESC" : "ASC";

    StringBuilder sql = new StringBuilder(
        "SELECT * FROM orders WHERE 1=1"
    );
    Map<String, Object> params = new LinkedHashMap<>();

    if (cond.getUserId() != null) {
        sql.append(" AND user_id = :userId");
        params.put("userId", cond.getUserId());
    }
    if (cond.getStatus() != null) {
        sql.append(" AND status = :status");
        params.put("status", cond.getStatus());
    }
    if (cond.getFromDate() != null) {
        sql.append(" AND created_at >= :fromDate");
        params.put("fromDate", cond.getFromDate());
    }

    sql.append(" ORDER BY ").append(sortCol).append(" ").append(sortDir);
    sql.append(" LIMIT :size OFFSET :offset");
    params.put("size", cond.getSize());
    params.put("offset", (long) cond.getPage() * cond.getSize());

    DatabaseClient.GenericExecuteSpec spec = databaseClient.sql(sql.toString());
    for (var entry : params.entrySet()) {
        spec = spec.bind(entry.getKey(), entry.getValue());
    }
    return spec.fetch().all().map(Order::from);
}
```

---

## 📊 성능 비교

```
N+1 vs 배치 로딩 vs JOIN 비교 (주문 100개, 사용자 조회):

N+1 (flatMap + findById):
  DB 쿼리: 1 + 100 = 101번
  네트워크 왕복: 101번
  총 시간: 1ms(findAll) + 100 × 2ms(findById) = 201ms

배치 로딩 (IN 조건):
  DB 쿼리: 1 + 1 = 2번
  네트워크 왕복: 2번
  총 시간: 1ms(findAll) + 5ms(findAllById IN) = 6ms
  개선율: 201ms → 6ms (33배 빠름)

JOIN 쿼리:
  DB 쿼리: 1번
  네트워크 왕복: 1번
  총 시간: 7ms (JOIN 쿼리)
  개선율: 201ms → 7ms (28배 빠름, 배치 로딩과 비슷)

배치 INSERT 비교 (1,000건):
  개별 INSERT × 1,000: ~500ms
  Statement Batch:       ~50ms (10배 빠름)
  멀티 VALUES INSERT:    ~30ms (17배 빠름)
  COPY FROM (PostgreSQL): ~10ms (50배 빠름)
```

---

## ⚖️ 트레이드오프

```
JOIN vs 배치 로딩:
  JOIN:
    장점: 단 1번 쿼리, DB 최적화 가능
    단점: SQL 복잡도 증가, 서비스 경계 강결합
          마이크로서비스에서 다른 서비스 DB를 JOIN 불가
  배치 로딩 (IN):
    장점: 서비스 독립성 유지, MSA 친화적
    단점: 2번 쿼리, IN 절 크기 제한 (Oracle 1000개)

배치 크기 선택:
  너무 작음: 배치 효과 미미, 쿼리 많이 발생
  너무 큼: 메모리 압박, DB 쿼리 크기 초과
  권장: 100~1,000건 (서비스 특성에 따라 조정)

연결 풀 크기:
  max-size 너무 작음: 대기 발생 → 레이턴시 증가
  max-size 너무 큼: DB 연결 과부하, 메모리 낭비
  권장: DB 서버 코어 × 2~4, 실측 기반 조정

collectList() 주의:
  전체 결과를 메모리에 올림 → 대용량 시 OOM
  대용량: buffer(N)으로 청크 처리
  소용량(< 10,000건): collectList() 사용 가능
```

---

## 📌 핵심 정리

```
R2DBC 실전 패턴 핵심:

N+1 문제:
  발생: flatMap 내 findById 반복
  해결 1: JOIN 쿼리로 한 번에
  해결 2: IN 조건 배치 로딩
  collectList() → userIds 추출 → findAllById(userIds)

배치 INSERT:
  saveAll(): 개별 INSERT (편함, 성능 낮음)
  Statement.add(): 진정한 배치 (성능 높음)
  권장: 대용량 INSERT는 Statement Batch 사용

연결 풀 튜닝:
  max-size: DB 코어 × 2~4
  pending > 0 지속 시 max-size 증가 검토
  max-acquire-time: SLA 절반

배치 처리:
  buffer(N).flatMap(batch -> processBatch, 동시수)
  배치 내 개별 실패는 onErrorResume으로 격리
  배치 크기와 동시 처리 수 균형
```

---

## 🤔 생각해볼 문제

**Q1.** `findAllById(Iterable<ID>)`는 내부적으로 어떻게 `IN` 쿼리를 생성하나요? ID가 1000개 이상이면 어떻게 되나요?

<details>
<summary>해설 보기</summary>

Spring Data R2DBC의 `findAllById()`는 내부적으로 `WHERE id IN (:ids)` 쿼리를 생성합니다. 파라미터 바인딩 방식은 드라이버에 따라 다릅니다.

PostgreSQL R2DBC: `WHERE id = ANY($1)` (배열 파라미터)
MySQL R2DBC: `WHERE id IN (?, ?, ?, ...)`

**ID가 1000개 이상일 때 문제:**
- Oracle: IN 절 최대 1000개 제한 → 초과 시 오류
- MySQL/PostgreSQL: 제한 없지만 쿼리 크기 증가 → 성능 저하

해결책:
```java
// IN 절 크기 제한 시 청크 분할
Flux<User> findByIdsInChunks(Set<Long> ids, int chunkSize) {
    return Flux.fromIterable(ids)
        .buffer(chunkSize)  // chunkSize씩 분할
        .flatMap(chunk ->
            userRepo.findAllById(chunk)  // 청크별 IN 쿼리
        );
}
// 1000개를 500씩 2번 조회 → Oracle IN 제한 회피
```

</details>

---

**Q2.** `DatabaseClient`에서 SQL 인젝션을 방어하는 방법은 무엇인가요?

<details>
<summary>해설 보기</summary>

항상 **파라미터 바인딩**을 사용합니다. SQL 문자열에 값을 직접 문자열 조합하는 것은 절대 금지입니다.

```java
// 취약한 코드 (SQL 인젝션 위험!)
String sql = "SELECT * FROM users WHERE name = '" + name + "'";
databaseClient.sql(sql).fetch().all();

// 안전한 코드 (파라미터 바인딩)
databaseClient.sql("SELECT * FROM users WHERE name = :name")
    .bind("name", name)  // 드라이버가 이스케이프 처리
    .fetch().all();
```

동적 정렬 컬럼은 파라미터 바인딩이 안 되므로 **화이트리스트 검증**을 사용합니다:

```java
Set<String> allowed = Set.of("name", "created_at", "email");
String sortCol = allowed.contains(userInput) ? userInput : "id";
String sql = "SELECT * FROM users ORDER BY " + sortCol;  // 화이트리스트 통과 후 안전
```

</details>

---

**Q3.** Reactive 환경에서 페이지네이션 쿼리(`SELECT` + `COUNT`)를 어떻게 효율적으로 처리하나요?

<details>
<summary>해설 보기</summary>

총 건수 쿼리(`COUNT`)와 데이터 쿼리를 병렬로 실행합니다:

```java
public Mono<PageResult<Order>> findPage(SearchCondition cond) {
    String baseWhere = "WHERE status = :status";

    Flux<Order> dataFlux = databaseClient.sql(
        "SELECT * FROM orders " + baseWhere +
        " ORDER BY id DESC LIMIT :size OFFSET :offset"
    )
    .bind("status", cond.getStatus())
    .bind("size", cond.getSize())
    .bind("offset", (long) cond.getPage() * cond.getSize())
    .fetch().all()
    .map(Order::from);

    Mono<Long> countMono = databaseClient.sql(
        "SELECT COUNT(*) as cnt FROM orders " + baseWhere
    )
    .bind("status", cond.getStatus())
    .fetch().one()
    .map(row -> (Long) row.get("cnt"));

    // 두 쿼리 병렬 실행
    return Mono.zip(dataFlux.collectList(), countMono)
        .map(t -> new PageResult<>(t.getT1(), t.getT2(),
            cond.getPage(), cond.getSize()));
}
// 총 2번 쿼리 (병렬) → 더 느린 쪽 시간만큼 소요
```

</details>

---

<div align="center">

**[⬅️ 이전: Reactive Transaction](./03-reactive-transaction.md)** | **[홈으로 🏠](../README.md)** | **[다음: Redis Reactive ➡️](./05-redis-reactive.md)**

</div>
