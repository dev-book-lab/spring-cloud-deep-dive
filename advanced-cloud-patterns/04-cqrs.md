# CQRS (Command Query Responsibility Segregation) — 쓰기와 읽기를 분리하는 아키텍처

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 단일 모델에서 읽기/쓰기 경합이 발생하는 구체적인 시나리오와 성능 영향은?
- Command 모델과 Query 모델을 분리했을 때 각각 어떤 데이터 저장소가 적합한가?
- Event Sourcing과 CQRS를 조합할 때 Query 모델을 어떻게 구성하는가?
- Spring Data Projection을 활용해 Query 모델을 경량화하는 방법과 인터페이스/클래스 Projection의 차이는?
- Command 모델 변경을 Query 모델에 반영하는 동기화 메커니즘과 Eventually Consistent 전략은?
- CQRS 도입 시 트랜잭션 경계를 어떻게 설정해야 하는가?

---

## 🔍 왜 MSA에서 필요한가

### 단일 모델의 읽기/쓰기 경합 문제

```
주문 서비스의 단일 모델 문제:

쓰기 트래픽:
  - 주문 생성/수정/취소
  - 재고 변경
  - 결제 처리
  → DB 쓰기 위주, 락 경합, 정합성 중요

읽기 트래픽:
  - 주문 목록 조회 (필터, 정렬, 페이징)
  - 주문 통계 (일별 매출, 상품별 주문량)
  - 대시보드 (실시간 현황)
  → 복잡한 조인, 집계 쿼리, 속도 중요, 약간의 지연 허용 가능

단일 모델의 문제:
  ① 스키마 충돌:
    쓰기에 최적화 (정규화) vs 읽기에 최적화 (비정규화)
    → 타협이 필요 → 둘 다 최악
  
  ② 락 경합:
    쓰기 트랜잭션이 읽기 쿼리를 블로킹
    → 읽기 많을수록 쓰기 성능 저하
  
  ③ 스케일링 불균형:
    읽기:쓰기 = 100:1 비율인데
    DB 서버를 균등 확장 → 비효율

CQRS 해결:
  Command Side (쓰기):
    정규화된 DB (MySQL, PostgreSQL)
    ACID 트랜잭션, 락 경합 없음
  
  Query Side (읽기):
    비정규화 읽기 전용 DB (Elasticsearch, Redis, Read Replica)
    빠른 조회, 복잡한 집계, 독립 확장
```

---

## 😱 잘못된 구성

### Before: 복잡한 조회를 Command 모델 DB에서 처리

```java
// ❌ 읽기/쓰기 동일 모델 + 복잡한 조인
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    // ❌ 복잡한 집계 쿼리 — 쓰기 DB에 과도한 부하
    @Query("""
        SELECT new com.example.OrderDashboardDto(
            o.status,
            COUNT(o),
            SUM(o.amount),
            AVG(o.amount)
        )
        FROM Order o
        JOIN o.user u
        JOIN o.items i
        JOIN i.product p
        WHERE o.createdAt BETWEEN :start AND :end
        GROUP BY o.status
        """)
    List<OrderDashboardDto> getOrderDashboard(
        @Param("start") Instant start,
        @Param("end") Instant end);
    // → 대용량 JOIN + 집계 → 쓰기 트랜잭션 블로킹!
}
```

---

## ✨ 올바른 패턴

### After: Command 모델 + Query 모델 분리

```
CQRS 아키텍처:

Command Side (쓰기 전용):
  CreateOrderCommand
    → OrderCommandService
    → OrderRepository (JPA, MySQL)
    → OrderCreatedEvent 발행

  DB 스키마: 정규화 (orders, order_items, users, ...)
  최적화: 쓰기 성능, ACID, 락 최소화

Query Side (읽기 전용):
  GetOrderDetailQuery
    → OrderQueryService
    → OrderReadRepository (Elasticsearch, Redis, or Read Replica)

  DB 스키마: 비정규화 (order_summary에 모든 필드 포함)
  최적화: 읽기 속도, 복잡한 필터/정렬/집계

동기화:
  Command Model → OrderCreatedEvent → Event Handler → Query Model 갱신
  Eventually Consistent (최종 일관성): 수 ms~수 초 지연 허용
```

---

## 🔬 내부 동작 원리

### 1. Command 모델 — 쓰기 전용 설계

```java
// Command: 쓰기 의도를 표현하는 객체
@Value
public class CreateOrderCommand {
    String userId;
    List<OrderItemDto> items;
    String shippingAddress;
}

// CommandHandler: Command를 처리하고 도메인 이벤트 발행
@Service
@Transactional
public class OrderCommandService {

    private final OrderRepository orderRepository;
    private final StreamBridge streamBridge;

    public String handle(CreateOrderCommand command) {
        // ① 비즈니스 로직 (Command Model에서만)
        Order order = Order.create(
            command.getUserId(),
            command.getItems());

        // ② Command Model DB에 저장
        orderRepository.save(order);

        // ③ 도메인 이벤트 발행 → Query Model 동기화 트리거
        streamBridge.send("order-events",
            new OrderCreatedEvent(order.getId(),
                order.getUserId(),
                order.getStatus(),
                order.getTotalAmount(),
                order.getCreatedAt()));

        return order.getId();
    }
}

// Command Model Entity: 쓰기에 최적화 (정규화)
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private String id;

    private String userId;
    private OrderStatus status;
    private BigDecimal totalAmount;

    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<OrderItem> items;  // 별도 테이블 (정규화)

    // 비즈니스 메서드
    public static Order create(String userId, List<OrderItemDto> items) {
        Order order = new Order();
        order.id = UUID.randomUUID().toString();
        order.userId = userId;
        order.status = OrderStatus.PENDING;
        order.items = items.stream().map(OrderItem::from).collect(toList());
        order.totalAmount = items.stream()
            .map(i -> i.getPrice().multiply(new BigDecimal(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        return order;
    }
}
```

### 2. Query 모델 — 읽기 전용 설계

```java
// Query Model: 읽기에 최적화 (비정규화)
// 별도 DB 또는 별도 스키마

// Elasticsearch 기반 Query Model
@Document(indexName = "orders")
public class OrderDocument {
    @Id
    private String id;

    // 비정규화: 모든 필드를 한 곳에
    private String userId;
    private String userName;         // users 테이블에서 역정규화
    private String userEmail;        // 조인 없이 바로 읽기 가능
    private OrderStatus status;
    private BigDecimal totalAmount;
    private List<OrderItemView> items;  // 중첩 객체 (역정규화)
    private Instant createdAt;

    @Data
    public static class OrderItemView {
        private String productId;
        private String productName;  // products에서 역정규화
        private int quantity;
        private BigDecimal price;
    }
}

// Query Service: 읽기 전용 로직
@Service
@Transactional(readOnly = true)
public class OrderQueryService {

    private final ElasticsearchRestTemplate esTemplate;

    // 복잡한 필터/정렬/집계를 Elasticsearch에서 처리
    public Page<OrderDocument> searchOrders(OrderSearchCriteria criteria) {
        Criteria esCriteria = new Criteria();

        if (criteria.getUserId() != null) {
            esCriteria.and("userId").is(criteria.getUserId());
        }
        if (criteria.getStatus() != null) {
            esCriteria.and("status").is(criteria.getStatus());
        }
        if (criteria.getAmountRange() != null) {
            esCriteria.and("totalAmount")
                .between(criteria.getAmountRange().getMin(),
                         criteria.getAmountRange().getMax());
        }

        // 풀텍스트 검색, 집계, 정렬 모두 Elasticsearch에서
        Query query = new CriteriaQuery(esCriteria)
            .addSort(Sort.by("createdAt").descending())
            .setPageable(criteria.getPageable());

        return esTemplate.search(query, OrderDocument.class);
    }

    // 대시보드 집계 (Elasticsearch Aggregation)
    public OrderDashboard getDashboard(Instant from, Instant to) {
        // Elasticsearch Aggregation API로 집계
        // → 쓰기 DB에 부하 없음
    }
}
```

### 3. Spring Data Projection — 경량화 Query 모델

```java
// Spring Data Projection: 엔티티의 일부 필드만 조회
// 별도 Query Model DB 없이도 경량화 가능 (부분 CQRS)

// ① 인터페이스 Projection: 프록시 기반, 필요한 필드만
public interface OrderSummaryProjection {
    String getId();
    OrderStatus getStatus();
    BigDecimal getTotalAmount();
    Instant getCreatedAt();
    // userId, items 등 불필요한 필드 제외

    // 중첩 Projection (연관 엔티티)
    UserSummary getUser();

    interface UserSummary {
        String getName();
    }
}

// Repository에서 Projection 사용
public interface OrderRepository extends JpaRepository<Order, Long> {

    // 일부 필드만 조회 (SELECT id, status, total_amount, created_at FROM orders)
    List<OrderSummaryProjection> findByUserId(String userId);

    // @Query와 함께 사용
    @Query("SELECT o.id as id, o.status as status, o.totalAmount as totalAmount" +
           " FROM Order o WHERE o.userId = :userId")
    List<OrderSummaryProjection> findSummaryByUserId(@Param("userId") String userId);
}

// ② 클래스 기반 Projection (DTO Projection): 더 명확한 타입
@Value
public class OrderSummaryDto {
    String id;
    OrderStatus status;
    BigDecimal totalAmount;
    Instant createdAt;

    // JPA가 이 생성자를 사용해 직접 매핑
    public OrderSummaryDto(String id, OrderStatus status,
            BigDecimal totalAmount, Instant createdAt) {
        this.id = id;
        this.status = status;
        this.totalAmount = totalAmount;
        this.createdAt = createdAt;
    }
}

// @Query에서 new 키워드로 DTO 직접 생성
@Query("""
    SELECT new com.example.OrderSummaryDto(
        o.id, o.status, o.totalAmount, o.createdAt
    )
    FROM Order o
    WHERE o.userId = :userId
    ORDER BY o.createdAt DESC
    """)
List<OrderSummaryDto> findSummariesByUserId(@Param("userId") String userId);
```

### 4. Event Handler — Query 모델 동기화

```java
// Command Model 변경 → 이벤트 → Query Model 업데이트

@Service
public class OrderQueryModelUpdater {

    private final ElasticsearchRestTemplate esTemplate;
    private final UserQueryClient userClient;  // 비정규화 데이터 가져오기

    // OrderCreated 이벤트 수신 → Elasticsearch 인덱스 업데이트
    @Bean
    public Consumer<OrderCreatedEvent> orderIndexer() {
        return event -> {
            // 비정규화: 사용자 정보 가져와서 함께 저장
            UserDto user = userClient.getUser(event.getUserId());

            OrderDocument doc = OrderDocument.builder()
                .id(event.getOrderId())
                .userId(event.getUserId())
                .userName(user.getName())       // 역정규화
                .userEmail(user.getEmail())     // 역정규화
                .status(event.getStatus())
                .totalAmount(event.getAmount())
                .createdAt(event.getOccurredAt())
                .build();

            esTemplate.save(doc);
            log.info("Query model updated for order: {}", event.getOrderId());
        };
    }

    // OrderStatusChanged → Elasticsearch 부분 업데이트
    @Bean
    public Consumer<OrderStatusChangedEvent> orderStatusIndexer() {
        return event -> {
            // 전체 문서 교체 대신 상태만 업데이트
            UpdateQuery updateQuery = UpdateQuery.builder(event.getOrderId())
                .withDocument(Document.create()
                    .append("status", event.getNewStatus().name()))
                .build();

            esTemplate.update(updateQuery, IndexCoordinates.of("orders"));
        };
    }
}
```

### 5. Event Sourcing + CQRS 조합

```java
// Event Sourcing: 상태 변경 대신 이벤트를 저장
// 현재 상태 = 모든 이벤트를 재실행한 결과

// Event Store (이벤트 저장소)
@Entity
@Table(name = "order_events")
public class OrderEvent {
    @Id
    private String eventId;
    private String orderId;         // Aggregate ID
    private int sequenceNumber;     // 이벤트 순서
    private String eventType;       // "OrderCreated", "OrderPaid", ...
    private String eventData;       // JSON
    private Instant occurredAt;
}

// Command Handler: 상태 저장 대신 이벤트 저장
@Transactional
public void handle(CreateOrderCommand command) {
    // ① 이벤트 생성
    OrderCreatedEvent event = OrderCreatedEvent.from(command);

    // ② Event Store에 저장 (상태 아님!)
    orderEventStore.save(new OrderEvent(event));

    // ③ Query Model 업데이트 트리거 (같은 이벤트로)
    streamBridge.send("order-events", event);
}

// 현재 상태 재구성 (Command 처리 시 필요)
public Order reconstitute(String orderId) {
    List<OrderEvent> events = orderEventStore.findByOrderId(orderId);

    Order order = new Order();
    for (OrderEvent event : events) {
        // 이벤트를 순서대로 재실행 → 현재 상태 도출
        order.apply(deserialize(event));
    }
    return order;
}

// CQRS + Event Sourcing 조합의 Query Model:
// Event Store → (Stream) → Query Model Projector → Query DB
// → Event Sourcing의 이벤트가 CQRS의 Query Model 업데이트에 활용
```

---

## 💻 실전 구성

### 단계적 CQRS 도입

```
단계 1 (기본): Spring Data Projection + Read Replica
  단일 DB 유지 + Projection으로 쿼리 경량화
  복잡한 조회는 Read Replica로 분리
  도입 비용: 낮음, 효과: 중간

단계 2 (중간): 별도 Query Model DB 도입
  Command DB: MySQL (트랜잭션)
  Query DB: Elasticsearch (검색/집계) 또는 Redis (캐시)
  이벤트로 동기화
  도입 비용: 중간, 효과: 높음

단계 3 (고급): Event Sourcing 도입
  Command Store: 이벤트 스토어
  Query Model: 다양한 뷰 (DB, 캐시, 검색)
  도입 비용: 매우 높음, 효과: 최고
  
  실무 권장: 단계 1 또는 2부터 시작
             Event Sourcing은 복잡성이 높아 신중히 선택
```

```yaml
# application.yml - 다중 DataSource 설정 (Command/Query DB 분리)
spring:
  datasource:
    command:
      url: jdbc:mysql://command-db:3306/orders
      username: order_writer
    query:
      url: jdbc:mysql://query-db:3306/orders_read  # Read Replica
      username: order_reader
      hikari:
        read-only: true   # 읽기 전용 커넥션

# Elasticsearch (Query Model)
  elasticsearch:
    uris: http://elasticsearch:9200
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 주문 생성 → Query Model 반영까지 지연

```
t=0s:   POST /orders → CreateOrderCommand
         → MySQL에 주문 저장 (ACID)
         → OrderCreated 이벤트 발행 → Kafka

t=0.1s: 클라이언트: GET /orders/123 (방금 만든 주문 조회)
         Query Service → Elasticsearch 조회
         → 아직 인덱싱 안 됨 → 404 Not Found?

t=0.5s: Kafka Consumer → Elasticsearch 인덱싱 완료

t=0.5s+: GET /orders/123 → 200 OK (이제 조회됨)

"최종 일관성" 지연 (0~500ms):
  클라이언트 처리:
  ① 쓰기 직후 즉각 읽기가 필요하면: Command DB에서 직접 조회 (예외)
  ② 대부분의 조회: Query Model 사용 (약간의 지연 허용)
  ③ UI: "주문이 접수됐습니다. 잠시 후 목록에 표시됩니다."

  설계 원칙:
  "Write-to-Read 일관성이 필요한 경우"를 명확히 파악
  → 해당 케이스만 Command DB 직접 조회
  → 나머지는 Query Model 사용
```

---

## ⚖️ 트레이드오프

| 방식 | 복잡도 | 일관성 | 확장성 | 적합한 케이스 |
|------|--------|--------|--------|--------------|
| **단일 모델** | 낮음 | 강한 일관성 | 낮음 | 단순 CRUD |
| **Projection만** | 낮음 | 강한 일관성 | 중간 | 쿼리 최적화 필요 |
| **CQRS (별도 DB)** | 높음 | 최종 일관성 | 높음 | 읽기 집중 서비스 |
| **CQRS + Event Sourcing** | 매우 높음 | 최종 일관성 | 최고 | 복잡 도메인 |

```
CQRS 도입 판단 기준:
  도입 권장:
  ✅ 읽기/쓰기 비율 불균형 심함 (읽기 10배 이상)
  ✅ 복잡한 집계 쿼리가 쓰기 성능 저하
  ✅ 다양한 읽기 뷰 필요 (UI별, 고객별 다른 형태)
  ✅ 쓰기와 읽기의 스케일링 요구사항이 다름

  도입 보류:
  ❌ 단순 CRUD 애플리케이션
  ❌ 강한 일관성이 반드시 필요
  ❌ 팀이 Eventually Consistent 설계에 익숙하지 않음
  ❌ 빠른 MVP 개발이 우선
```

---

## 📌 핵심 정리

```
CQRS 핵심:

핵심 아이디어:
  쓰기(Command) 모델과 읽기(Query) 모델을 분리
  각각 다른 DB, 스키마, 최적화 전략 적용

Command Side:
  정규화 DB (ACID, 트랜잭션)
  비즈니스 로직 집중
  도메인 이벤트 발행

Query Side:
  비정규화 DB (Elasticsearch, Redis)
  복잡한 조회/집계 최적화
  이벤트로 동기화 (Eventually Consistent)

Spring Data Projection (경량 CQRS):
  인터페이스 Projection: 프록시 기반, 필드 선택적 조회
  DTO Projection: 생성자 매핑, 명시적 타입
  별도 Query DB 없이도 쿼리 최적화 가능

Event Sourcing 조합:
  이벤트 저장 → 이벤트 재실행 = 현재 상태
  같은 이벤트로 다양한 Query Model 구성 가능

일관성 전략:
  최종 일관성 허용: Query Model (ms~초 지연)
  강한 일관성 필요: Command DB 직접 조회
```

---

## 🤔 생각해볼 문제

**Q1.** CQRS를 적용했는데 Query Model(Elasticsearch)이 다운됐을 때 어떻게 처리해야 하는가?

<details>
<summary>해설 보기</summary>

여러 전략이 있습니다.

1. **Circuit Breaker 적용**: Elasticsearch 호출에 Circuit Breaker를 적용해 다운 시 Fallback으로 Command DB에서 직접 조회합니다. 성능은 저하되지만 서비스 가용성은 유지합니다.

2. **읽기 전용 캐시**: Redis에 최근 조회 결과를 캐싱하고, Elasticsearch 다운 시 캐시에서 제공합니다.

3. **Graceful Degradation**: 전체 기능 대신 핵심 기능만 제공합니다. 복잡한 검색은 불가하지만 주문 ID로 직접 조회는 Command DB에서 제공합니다.

4. **Command DB Read Replica**: Command DB에 Read Replica를 두고 Query Model로 전환합니다. 성능 최적화는 덜하지만 기본 조회는 가능합니다.

설계 원칙: "Query Model은 부가적인 것" — 다운돼도 쓰기는 계속 가능해야 합니다. 복구 후 이벤트 리플레이로 Query Model을 재구성합니다.

</details>

---

**Q2.** Query Model을 재구성(Rebuild)해야 할 때(예: 스키마 변경) 어떤 전략을 사용하는가?

<details>
<summary>해설 보기</summary>

Event Sourcing과 함께 사용하면 이벤트를 처음부터 재실행(Event Replay)해 새 스키마로 Query Model을 구성합니다. Event Sourcing 없다면:

1. **블루-그린 인덱스 전환 (Elasticsearch)**:
   - 새 인덱스(v2)를 생성
   - Command DB에서 모든 데이터를 새 인덱스로 백필
   - 완료 후 별칭(alias)을 v1 → v2로 전환
   - 다운타임 없이 전환 가능

2. **배치 재구성**:
   - 스케줄러로 Command DB → Query DB 전체 동기화
   - 야간에 실행 (트래픽 낮을 때)

3. **이벤트 큐 재처리**:
   - Kafka의 메시지 보존 기간 내 이벤트를 처음부터 재처리
   - 컨슈머 그룹 오프셋을 0으로 리셋

설계 원칙: "Query Model은 언제든지 버리고 재구성 가능해야 한다" — Command DB(또는 Event Store)가 Source of Truth입니다.

</details>

---

**Q3.** 인터페이스 기반 Projection과 DTO 기반 Projection의 성능 차이는 무엇인가? 어떤 것을 선택해야 하는가?

<details>
<summary>해설 보기</summary>

**인터페이스 Projection**: Spring Data가 JDK Dynamic Proxy를 생성해 각 `get*()` 호출 시 SpEL 표현식으로 값을 반환합니다. 약간의 프록시 오버헤드가 있습니다. 중첩 Projection(연관 엔티티)도 지원합니다.

**DTO Projection**: 생성자를 통해 직접 객체를 생성합니다. 프록시 없음 → 더 빠릅니다. 단, `@Query`에 `new` 키워드를 사용해야 합니다.

성능: DTO > 인터페이스 (프록시 오버헤드 없음)

선택 기준:
- **단순 읽기, 연관 없음**: DTO Projection 권장 (더 빠름, 타입 안전)
- **중첩 구조 (연관 엔티티 포함)**: 인터페이스 Projection (더 편함)
- **동적 SQL**: 인터페이스 Projection (SpEL로 조건부 필드 가능)
- **IDE 지원**: DTO가 더 좋음 (인터페이스는 리팩토링 시 경고 없음)

실무: 단순 목록 조회 → DTO Projection, 복잡한 중첩 응답 → 인터페이스 Projection 또는 `@EntityGraph`와 조합.

</details>

---

<div align="center">

**[⬅️ 이전: API Composition](./03-api-composition.md)** | **[홈으로 🏠](../README.md)**

</div>
