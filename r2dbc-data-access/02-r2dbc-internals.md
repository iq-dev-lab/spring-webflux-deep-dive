# R2DBC 완전 분해 — 논블로킹 DB 드라이버의 구조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- R2DBC는 JDBC와 달리 어떻게 논블로킹 DB 통신을 구현하는가?
- `DatabaseClient`와 `R2dbcRepository`는 언제 각각 사용하는가?
- JPA에서 R2DBC로 전환할 때 포기해야 하는 기능은 무엇인가?
- R2DBC에서 N+1 문제는 왜 더 명시적으로 드러나는가?
- 연결 풀(`r2dbc-pool`)은 어떻게 설정하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

R2DBC를 도입하면 JPA의 편리함을 많이 포기해야 합니다. 지연 로딩이 없고, 복잡한 연관관계 자동 처리가 없으며, JPQL/QueryDSL 대신 SQL을 직접 써야 합니다. 이 트레이드오프를 정확히 알아야 "어느 서비스를 R2DBC로 전환할 것인가"를 결정할 수 있습니다. R2DBC의 내부 구조를 이해하면 왜 이러한 제약이 있는지도 납득할 수 있습니다.

---

## 😱 흔한 실수 (Before — R2DBC를 JPA처럼 사용하려 할 때)

```
실수 1: @ManyToOne 연관관계로 자동 조인 기대

  @Table("orders")
  public class Order {
      @Id Long id;
      String name;
      // @ManyToOne 불가 → R2DBC는 연관관계 자동 로딩 없음
      User user;  // R2DBC가 자동으로 User를 조회하지 않음
  }
  // → user 필드는 null (JPA처럼 자동 조회 안 됨)

실수 2: 지연 로딩 기대

  // JPA에서는:
  Order order = orderRepository.findById(id);
  order.getUser().getName();  // 지연 로딩 → 이 시점에 SELECT user

  // R2DBC에서는:
  // 지연 로딩 개념 자체가 없음
  // 사용자가 필요 시 명시적으로 별도 조회해야 함

실수 3: JPQL 스타일 쿼리 작성

  // R2DBC 불가
  @Query("SELECT o FROM Order o WHERE o.user.name = :name")
  Flux<Order> findByUserName(String name);
  // → R2DBC는 JPQL 불가, SQL만 가능
  // @Query("SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE name = :name)")
```

---

## ✨ 올바른 접근 (After — R2DBC 특성에 맞는 설계)

```
R2DBC 설계 원칙:

1. 엔티티는 단순하게 (연관관계 없이)
   @Table("orders")
   public record Order(@Id Long id, Long userId, String status) {}
   // userId만 저장, User 객체 참조 없음

2. 연관 데이터는 명시적으로 조회
   orderRepository.findById(id)
       .flatMap(order ->
           userRepository.findById(order.userId())
               .map(user -> OrderDetail.of(order, user))
       )

3. 복잡한 쿼리는 DatabaseClient로 SQL 직접 작성
   databaseClient.sql("""
       SELECT o.*, u.name as user_name
       FROM orders o JOIN users u ON o.user_id = u.id
       WHERE o.status = :status
   """).bind("status", "PENDING")
     .fetch().all()
     .map(row -> OrderWithUser.of(row))
```

---

## 🔬 내부 동작 원리

### 1. R2DBC 논블로킹 구조

```
JDBC 동작 (블로킹):
  Thread: executeQuery() → 소켓 write → 소켓 read(블로킹) → ResultSet
  → DB 응답 대기 중 스레드 점유

R2DBC 동작 (논블로킹):
  EventLoop: 쿼리 전송 → 즉시 반환 (Mono/Flux 파이프라인 등록)
  Netty:     DB 소켓 감시 (select/epoll)
  EventLoop: DB 응답 도착 → onNext 신호 → Subscriber 실행

R2DBC 소켓 통신 방식:
  Java NIO SocketChannel 사용 (논블로킹 소켓)
  Netty 기반 드라이버: r2dbc-postgresql, r2dbc-mysql 등
  → DB와의 TCP 통신이 논블로킹
  → DB 응답 대기 중 EventLoop는 다른 요청 처리 가능

R2DBC vs JDBC 소켓 비교:
  JDBC: java.net.Socket (블로킹) + InputStream.read()
  R2DBC: java.nio.SocketChannel (논블로킹) + Netty epoll
  → 동일한 쿼리라도 소켓 I/O 모델이 근본적으로 다름
```

### 2. DatabaseClient — 저수준 SQL API

```
DatabaseClient:
  SQL을 직접 작성하는 저수준 R2DBC API
  가장 유연하고 강력, 모든 SQL 표현 가능

  @Autowired DatabaseClient databaseClient;

  // SELECT
  Flux<Order> orders = databaseClient
      .sql("SELECT id, user_id, status, total FROM orders WHERE status = :status")
      .bind("status", "PENDING")
      .fetch()
      .all()
      .map(row -> new Order(
          (Long) row.get("id"),
          (Long) row.get("user_id"),
          (String) row.get("status"),
          (BigDecimal) row.get("total")
      ));

  // INSERT
  Mono<Long> insertedId = databaseClient
      .sql("INSERT INTO orders (user_id, status, total) VALUES (:userId, :status, :total)")
      .bind("userId", order.getUserId())
      .bind("status", order.getStatus())
      .bind("total", order.getTotal())
      .filter(s -> s.returnGeneratedValues("id"))
      .fetch()
      .first()
      .map(row -> (Long) row.get("id"));

  // UPDATE
  Mono<Long> updated = databaseClient
      .sql("UPDATE orders SET status = :status WHERE id = :id")
      .bind("status", newStatus)
      .bind("id", orderId)
      .fetch()
      .rowsUpdated();

fetch() 메서드:
  .fetch().one():   Mono<Map<String, Object>> (최대 1행)
  .fetch().first(): Mono<Map<String, Object>> (첫 번째 행)
  .fetch().all():   Flux<Map<String, Object>> (모든 행)
  .fetch().rowsUpdated(): Mono<Long> (변경된 행 수)
```

### 3. Spring Data R2DBC Repository

```
R2dbcRepository 기본 사용:

  // 엔티티 정의
  @Table("users")
  public record User(
      @Id Long id,
      String name,
      String email,
      @Column("created_at") LocalDateTime createdAt
  ) {}

  // Repository 인터페이스
  public interface UserRepository extends R2dbcRepository<User, Long> {
      // 메서드 이름으로 쿼리 생성
      Mono<User> findByEmail(String email);
      Flux<User> findByNameStartingWith(String prefix);
      Flux<User> findByCreatedAtAfter(LocalDateTime since);
      Mono<Long> countByName(String name);

      // SQL 직접 작성
      @Query("SELECT * FROM users WHERE email LIKE :pattern ORDER BY name")
      Flux<User> findByEmailPattern(String pattern);

      // 페이징
      Flux<User> findAll(Sort sort);
      // R2dbcRepository는 Page 미지원 → Flux + count 별도 조합
  }

페이징 (Pageable 제한):
  R2dbcRepository는 Pageable 지원하지만 Page 객체 반환 불가
  → Flux + count 별도 쿼리 조합 필요

  public Mono<PaginatedResult<User>> findPage(int page, int size) {
      Flux<User> users = userRepository.findAll(
          PageRequest.of(page, size, Sort.by("name"))
      );
      Mono<Long> total = userRepository.count();
      return Mono.zip(users.collectList(), total)
          .map(t -> new PaginatedResult<>(t.getT1(), t.getT2(), page, size));
  }
```

### 4. JPA와 R2DBC 기능 비교

```
기능 비교표:

기능                    | JPA (Hibernate)    | Spring Data R2DBC
───────────────────────┼──────────────────┼─────────────────────
블로킹 여부             | 블로킹 (JDBC)      | 논블로킹 (NIO)
지연 로딩               | ✅ 자동            | ❌ 없음
캐스케이드              | ✅ 자동            | ❌ 수동 처리
복잡한 연관관계         | ✅ @ManyToMany 등  | ❌ 직접 구현
Dirty Checking         | ✅ 자동 감지/저장   | ❌ 명시적 save()
2차 캐시               | ✅ EhCache 등      | ❌ 없음
JPQL/HQL               | ✅                 | ❌ (SQL만)
QueryDSL               | ✅                 | ❌ (jOOQ 일부 가능)
자동 스키마 생성        | ✅ (DDL auto)      | ⚠️ (Flyway/Liquibase)
낙관적 잠금             | ✅ @Version        | ✅ @Version
복잡한 조인             | ✅ JPQL로 표현     | ⚠️ SQL 직접 작성
배치 처리               | ✅ (JPA batch)     | ✅ (Statement batch)

지연 로딩이 없는 이유:
  JPA 지연 로딩: 프록시 객체 반환 → 접근 시 동기 DB 쿼리
  Reactive: 접근 시점에 비동기 쿼리 발생 → Mono 반환 필요
  → 객체 필드가 Mono<User>가 되어야 함 → 엔티티 설계 불가능
  → 지연 로딩 대신: flatMap으로 명시적 연관 데이터 로딩
```

### 5. r2dbc-pool 연결 풀 설정

```
R2DBC 연결 풀:
  r2dbc-pool: R2DBC용 논블로킹 연결 풀 (기본 포함)
  ConnectionPool: 사용 가능한 연결을 관리하는 풀

application.yml 설정:
  spring:
    r2dbc:
      url: r2dbc:postgresql://localhost:5432/mydb
      username: user
      password: pass
      pool:
        initial-size: 5      # 초기 연결 수
        max-size: 20          # 최대 연결 수 (JDBC의 1/10 수준으로 충분)
        max-idle-time: 30m   # 유휴 연결 유지 시간
        max-acquire-time: 10s # 연결 획득 대기 최대 시간
        validation-query: SELECT 1  # 연결 유효성 검사 쿼리

프로그래밍 방식 설정:
  @Bean
  public ConnectionPool connectionPool(ConnectionFactory factory) {
      ConnectionPoolConfiguration config = ConnectionPoolConfiguration.builder()
          .connectionFactory(factory)
          .initialSize(5)
          .maxSize(20)
          .maxIdleTime(Duration.ofMinutes(30))
          .maxAcquireTime(Duration.ofSeconds(10))
          .validationQuery("SELECT 1")
          .build();
      return new ConnectionPool(config);
  }

최적 크기 계산:
  논블로킹: 연결당 여러 요청 처리 가능
  DB 서버 CPU 코어 수가 실질적 한계
  권장: DB 서버 코어 수 × 2 ~ 4 (예: 8코어 DB → 16~32 연결)
  JDBC 대비 1/5~1/10 연결로 동일 처리량 달성 가능
```

---

## 💻 실전 코드

### 실험 1: R2DBC 엔티티와 Repository 완전 설정

```java
// 엔티티 (연관관계 없음)
@Table("products")
public record Product(
    @Id Long id,
    String name,
    BigDecimal price,
    Integer stock,
    Long categoryId,  // 외래키 ID만 저장 (Category 객체 참조 없음)
    @Column("created_at") LocalDateTime createdAt
) {
    public static Product of(String name, BigDecimal price, Long categoryId) {
        return new Product(null, name, price, 0, categoryId, LocalDateTime.now());
    }
}

// Repository
public interface ProductRepository extends R2dbcRepository<Product, Long> {
    Flux<Product> findByCategoryId(Long categoryId);
    Flux<Product> findByPriceBetween(BigDecimal min, BigDecimal max);
    Mono<Long> countByCategoryId(Long categoryId);

    @Query("UPDATE products SET stock = stock - :quantity WHERE id = :id AND stock >= :quantity")
    Mono<Long> decreaseStock(Long id, int quantity);
}

// 서비스 (연관관계 명시적 처리)
@Service
@RequiredArgsConstructor
public class ProductService {
    private final ProductRepository productRepo;
    private final CategoryRepository categoryRepo;
    private final DatabaseClient databaseClient;

    public Mono<ProductDetail> findWithCategory(Long productId) {
        return productRepo.findById(productId)
            .switchIfEmpty(Mono.error(new ProductNotFoundException(productId)))
            .flatMap(product ->
                categoryRepo.findById(product.categoryId())
                    .map(category -> ProductDetail.of(product, category))
            );
    }

    // 복잡한 조회는 DatabaseClient로
    public Flux<ProductSummary> findByCategory(Long categoryId, int page, int size) {
        return databaseClient.sql("""
                SELECT p.id, p.name, p.price, p.stock, c.name as category_name
                FROM products p
                JOIN categories c ON p.category_id = c.id
                WHERE p.category_id = :categoryId
                ORDER BY p.name
                LIMIT :size OFFSET :offset
                """)
            .bind("categoryId", categoryId)
            .bind("size", size)
            .bind("offset", (long) page * size)
            .fetch().all()
            .map(row -> ProductSummary.from(row));
    }
}
```

### 실험 2: 배치 INSERT

```java
public Flux<Product> batchInsert(List<Product> products) {
    // R2DBC saveAll: 개별 INSERT 반복 (배치 아님)
    // 성능 개선: DatabaseClient로 배치 처리
    return databaseClient.inConnectionMany(connection -> {
        Statement stmt = connection.createStatement(
            "INSERT INTO products (name, price, stock, category_id) " +
            "VALUES ($1, $2, $3, $4) RETURNING id"
        );

        for (Product p : products) {
            stmt.bind(0, p.name())
                .bind(1, p.price())
                .bind(2, p.stock())
                .bind(3, p.categoryId())
                .add();  // 배치에 추가
        }

        return Flux.from(stmt.execute())
            .flatMap(result -> result.map((row, meta) ->
                (Long) row.get("id")
            ))
            .zipWith(Flux.fromIterable(products),
                (id, product) -> new Product(id, product.name(),
                    product.price(), product.stock(),
                    product.categoryId(), LocalDateTime.now())
            );
    });
}
```

### 실험 3: 동적 쿼리 (DatabaseClient 조건 조합)

```java
public Flux<Product> search(ProductSearchCondition condition) {
    StringBuilder sql = new StringBuilder(
        "SELECT * FROM products WHERE 1=1"
    );
    Map<String, Object> params = new LinkedHashMap<>();

    if (condition.getName() != null) {
        sql.append(" AND name ILIKE :name");
        params.put("name", "%" + condition.getName() + "%");
    }
    if (condition.getMinPrice() != null) {
        sql.append(" AND price >= :minPrice");
        params.put("minPrice", condition.getMinPrice());
    }
    if (condition.getCategoryId() != null) {
        sql.append(" AND category_id = :categoryId");
        params.put("categoryId", condition.getCategoryId());
    }
    sql.append(" ORDER BY ").append(condition.getSortBy())
       .append(" ").append(condition.getSortDir())
       .append(" LIMIT :size OFFSET :offset");
    params.put("size", condition.getSize());
    params.put("offset", (long) condition.getPage() * condition.getSize());

    DatabaseClient.GenericExecuteSpec spec =
        databaseClient.sql(sql.toString());
    for (var entry : params.entrySet()) {
        spec = spec.bind(entry.getKey(), entry.getValue());
    }
    return spec.fetch().all().map(Product::from);
}
```

---

## 📊 성능 비교

```
R2DBC vs JDBC 처리량 비교 (8코어, 동시 1000 요청, 쿼리 50ms):

JDBC (HikariCP 200 연결):
  최대 동시 DB 쿼리: 200개 (연결 수)
  초당 처리: 200 / 0.05s = 4,000 req/s
  메모리: 200 스레드 × 512KB = 100MB 스택

R2DBC (r2dbc-pool 20 연결):
  연결 20개로 논블로킹 처리
  쿼리 완료 즉시 연결 반환 → 높은 연결 재사용률
  초당 처리: 20 / 0.05s = 400 req/s (연결 재사용률 고려 시 더 높음)
  실제 처리: 연결당 여러 쿼리 처리 가능 → ~2,000~5,000 req/s
  메모리: 16 EventLoop × 512KB = 8MB 스택

DatabaseClient 쿼리 매핑 오버헤드:
  Map<String, Object> → 객체 변환: ~수십 μs
  vs JPA ResultSet → 엔티티: ~수십 μs
  → 실질적 차이 없음

r2dbc-pool 연결 획득 시간:
  풀에 여유 연결 있음: ~수 μs (즉시)
  모든 연결 사용 중: 쿼리 완료 대기 (논블로킹, EventLoop 차단 없음)
```

---

## ⚖️ 트레이드오프

```
R2DBC 도입 트레이드오프:

얻는 것:
  완전 논블로킹 DB I/O → 고처리량
  스레드 수 감소 → 메모리 효율
  WebFlux 이점 극대화

포기하는 것:
  지연 로딩 → 명시적 연관 데이터 로딩
  Dirty Checking → 명시적 save() 호출
  JPQL/QueryDSL → SQL 직접 작성
  복잡한 조인 → DatabaseClient로 수동 처리
  JPA 캐싱 → Redis 등 외부 캐시로 대체
  스키마 자동 생성 → Flyway/Liquibase 필수

적합한 서비스:
  CRUD 위주, 복잡한 도메인 로직 없음
  단순한 엔티티 (연관관계 적음)
  높은 동시 처리 필요
  I/O 집약, DB 응답 대기가 길고 요청이 많음

부적합한 서비스:
  복잡한 도메인 모델 (많은 연관관계)
  JPA Dirty Checking에 크게 의존
  JPQL 복잡한 쿼리가 많음
  → 이 경우 MVC + JPA 유지 권장
```

---

## 📌 핵심 정리

```
R2DBC 핵심:

논블로킹 구조:
  Java NIO SocketChannel + Netty epoll
  JDBC의 블로킹 소켓 대신 논블로킹 소켓
  DB 응답 대기 중 EventLoop는 다른 요청 처리

주요 API:
  DatabaseClient: SQL 직접 작성, 완전한 제어
  R2dbcRepository: 메서드 이름 기반 쿼리, 간단한 CRUD

JPA vs R2DBC:
  지연 로딩: ❌ → flatMap으로 명시적 로딩
  연관관계: ❌ → ID만 저장, 별도 조회
  JPQL: ❌ → SQL 직접 작성

r2dbc-pool:
  JDBC 연결의 1/5~1/10으로 동일 처리량
  논블로킹 연결 획득 (EventLoop 차단 없음)
  최대 크기: DB 코어 수 × 2~4 권장
```

---

## 🤔 생각해볼 문제

**Q1.** R2DBC에서 `@Version`을 이용한 낙관적 잠금은 어떻게 동작하나요?

<details>
<summary>해설 보기</summary>

R2DBC도 `@Version`을 지원합니다. JPA와 동일한 방식으로 버전 필드를 사용하여 낙관적 잠금을 구현합니다.

```java
@Table("products")
public record Product(
    @Id Long id,
    String name,
    BigDecimal price,
    @Version Long version  // 낙관적 잠금
) {}
```

`save(product)` 시 내부적으로:
```sql
UPDATE products SET name = ?, price = ?, version = version + 1
WHERE id = ? AND version = ?  -- 버전 체크
```

버전이 다르면(다른 요청이 먼저 수정) `OptimisticLockingFailureException`이 발생합니다. Reactive에서는 `onErrorResume(OptimisticLockingFailureException.class, e -> retry)`로 처리할 수 있습니다.

</details>

---

**Q2.** R2DBC에서 `Flux<Product> saveAll(Iterable<Product>)`는 내부적으로 배치 INSERT를 하나요?

<details>
<summary>해설 보기</summary>

`saveAll()`은 **개별 INSERT를 반복**합니다. 진정한 배치 처리가 아닙니다.

```
saveAll([p1, p2, p3]):
  INSERT INTO products (...) VALUES (p1...) -- 개별 실행
  INSERT INTO products (...) VALUES (p2...) -- 개별 실행
  INSERT INTO products (...) VALUES (p3...) -- 개별 실행
```

진정한 배치 INSERT를 원한다면 `DatabaseClient`의 `Statement.add()`로 배치를 구성해야 합니다. R2DBC 스펙은 `Batch` 인터페이스를 제공하지만 Spring Data R2DBC의 `saveAll()`은 이를 활용하지 않습니다.

대용량 INSERT 성능이 중요하다면 `DatabaseClient.inConnectionMany()`로 직접 배치를 구현하거나, 임시 테이블 방식을 사용하는 것이 좋습니다.

</details>

---

**Q3.** R2DBC에서 JPA의 `@EntityGraph`처럼 연관 데이터를 효율적으로 로딩하는 방법이 있나요?

<details>
<summary>해설 보기</summary>

R2DBC에는 `@EntityGraph`가 없습니다. 대신 두 가지 방법을 사용합니다.

**방법 1: JOIN SQL로 한 번에 조회** (가장 효율적)
```java
databaseClient.sql("""
    SELECT p.*, c.name as category_name
    FROM products p LEFT JOIN categories c ON p.category_id = c.id
    WHERE p.id = :id
    """)
    .bind("id", productId)
    .fetch().one()
    .map(row -> ProductWithCategory.from(row));
```

**방법 2: 병렬 조회** (독립적 데이터일 때)
```java
Mono.zip(
    productRepo.findById(productId),
    reviewRepo.findByProductId(productId).collectList()
)
.map(t -> ProductDetail.of(t.getT1(), t.getT2()));
```

**방법 3: 배치 로딩으로 N+1 방지**
```java
Flux<Order> orders = orderRepo.findAll();
orders.collectList()
    .flatMapMany(orderList -> {
        Set<Long> userIds = orderList.stream()
            .map(Order::userId).collect(toSet());
        return userRepo.findAllById(userIds)
            .collectMap(User::id)
            .flatMapMany(userMap ->
                Flux.fromIterable(orderList)
                    .map(o -> OrderDetail.of(o, userMap.get(o.userId())))
            );
    });
```

JOIN SQL이 가장 효율적이고, 독립 데이터는 `Mono.zip`으로 병렬 조회합니다.

</details>

---

<div align="center">

**[⬅️ 이전: JPA와 WebFlux의 충돌](./01-jpa-webflux-conflict.md)** | **[홈으로 🏠](../README.md)** | **[다음: Reactive Transaction ➡️](./03-reactive-transaction.md)**

</div>
