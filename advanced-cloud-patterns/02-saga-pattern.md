# Saga Pattern (분산 트랜잭션) — 2PC의 한계를 넘는 장기 실행 트랜잭션

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 2PC(Two-Phase Commit)가 MSA에서 가용성 문제를 일으키는 구체적인 메커니즘은?
- Choreography Saga와 Orchestration Saga의 트레이드오프는 어떻게 비교되는가?
- 보상 트랜잭션(Compensating Transaction)이 롤백과 다른 이유는 무엇인가?
- 멱등성(Idempotency)을 Saga에서 보장하는 구체적인 구현 방법은?
- Orchestration Saga에서 상태 머신(State Machine)을 어떻게 구현하는가?
- Saga 실행 중 실패가 발생했을 때 보상 트랜잭션의 실행 순서와 보장 방법은?

---

## 🔍 왜 MSA에서 필요한가

### 분산 트랜잭션의 근본적 문제

```
단일 DB 트랜잭션:
  BEGIN TRANSACTION
    UPDATE orders SET status='PAID'
    UPDATE inventory SET stock=stock-1
    INSERT INTO payments (order_id, amount) VALUES (...)
  COMMIT  ← 전부 성공 또는 전부 롤백

MSA에서:
  Order Service (DB1): UPDATE orders SET status='PAID' ✅
  Inventory Service (DB2): UPDATE inventory SET stock=stock-1 ✅
  Payment Service (DB3): INSERT INTO payments... ← 실패!

  문제:
  Order는 PAID, Inventory는 차감됐는데 Payment는 실패
  → 시스템 불일치 상태!
  → 하나의 ACID 트랜잭션으로 묶을 수 없음 (다른 DB)

2PC(Two-Phase Commit) 시도:
  Phase 1 (Prepare): 모든 서비스에 "커밋할 준비 됐냐?"
  Phase 2 (Commit/Rollback): 전부 OK면 커밋, 하나라도 NO면 롤백

  문제:
  ① 블로킹: Prepare 후 Commit까지 모든 서비스 잠금 대기
  ② 가용성 희생: 하나 다운 → 전체 블로킹 (CAP의 CP)
  ③ 코디네이터 SPOF: 트랜잭션 코디네이터 다운 → 전체 멈춤

Saga 패턴:
  긴 트랜잭션을 작은 로컬 트랜잭션들의 연쇄로 분해
  실패 시 이미 완료된 트랜잭션을 보상 트랜잭션으로 취소
  → 가용성 유지 + 최종 일관성 달성
```

---

## 😱 잘못된 구성

### Before: 보상 트랜잭션 없는 Saga

```java
// ❌ 실패 시 보상 없는 단순 이벤트 체인
@Bean
public Consumer<OrderCreatedEvent> inventoryProcessor() {
    return event -> {
        // 재고 차감 성공
        inventoryService.decreaseStock(event.getProductId(), event.getQuantity());
        // 이후 Payment 실패해도 재고는 차감된 채로 남음
        // → 고객은 결제 안 됐는데 재고는 줄어있는 상태
    };
}
```

### Before: 멱등성 없는 보상 트랜잭션

```java
// ❌ 중복 보상이 발생하면 재고가 두 번 복구됨
@Bean
public Consumer<PaymentFailedEvent> inventoryCompensation() {
    return event -> {
        // 같은 이벤트가 두 번 오면?
        // → 재고가 두 번 복구됨 → 재고 불일치
        inventoryService.increaseStock(
            event.getProductId(), event.getQuantity());
    };
}
```

---

## ✨ 올바른 패턴

### After: Choreography Saga vs Orchestration Saga 선택

```
Choreography Saga:
  서비스들이 이벤트를 보고 스스로 판단해 다음 단계 실행
  중앙 조율자 없음

  주문 생성:
  Order → OrderCreated 이벤트
    → Inventory: 재고 차감 → InventoryReserved 이벤트
      → Payment: 결제 → PaymentCompleted 이벤트
        → Order: 상태 완료 업데이트

  실패 시:
  Payment → PaymentFailed 이벤트
    → Inventory: 재고 복구 (보상)
      → Order: 상태 취소 업데이트 (보상)

  장점: 중앙 조율자 없음, 서비스 독립적
  단점: 흐름 파악 어려움, 사이클 위험

Orchestration Saga:
  중앙 오케스트레이터(Saga)가 각 서비스에 명령 전달
  전체 흐름을 한 곳에서 관리

  OrderSaga:
  step1: InventoryService.reserveStock() → 성공
  step2: PaymentService.processPayment() → 실패!
  → compensate step1: InventoryService.releaseStock()
  → Saga 종료 (실패)

  장점: 흐름 명확, 디버깅 쉬움, 사이클 없음
  단점: 중앙 오케스트레이터 필요, 단일 장애점 가능성
```

---

## 🔬 내부 동작 원리

### 1. Choreography Saga — 이벤트 기반 자율 조율

```java
// Choreography Saga 구현 예시

// Order Service: Saga 시작
@Service
public class OrderService {

    @Autowired
    private StreamBridge streamBridge;

    @Transactional
    public Order createOrder(OrderRequest request) {
        // ① 주문 생성 (PENDING 상태)
        Order order = Order.builder()
            .status(OrderStatus.PENDING)
            .build();
        orderRepository.save(order);

        // ② OrderCreated 이벤트 발행 (Outbox 패턴 권장)
        streamBridge.send("order-events",
            new OrderCreatedEvent(order.getId(), request.getProductId(),
                request.getQuantity(), request.getAmount()));

        return order;
    }

    // Saga 완료 이벤트 처리 (PaymentCompleted 수신 시)
    @Bean
    public Consumer<PaymentCompletedEvent> orderCompleter() {
        return event -> {
            Order order = orderRepository.findById(event.getOrderId())
                .orElseThrow();
            order.setStatus(OrderStatus.CONFIRMED);
            orderRepository.save(order);
        };
    }

    // Saga 보상 이벤트 처리 (PaymentFailed 수신 시)
    @Bean
    public Consumer<PaymentFailedEvent> orderCanceller() {
        return event -> {
            Order order = orderRepository.findById(event.getOrderId())
                .orElseThrow();
            order.setStatus(OrderStatus.CANCELLED);
            orderRepository.save(order);
        };
    }
}

// Inventory Service: 재고 예약 + 보상
@Configuration
public class InventoryEventHandlers {

    @Autowired
    private StreamBridge streamBridge;

    // 재고 예약 (정상 흐름)
    @Bean
    public Consumer<OrderCreatedEvent> inventoryReserver() {
        return event -> {
            String sagaId = event.getOrderId();

            // 멱등성: 이미 처리된 sagaId면 스킵
            if (sagaRepo.isProcessed(sagaId, "INVENTORY_RESERVE")) {
                return;
            }

            try {
                inventoryService.reserveStock(
                    event.getProductId(), event.getQuantity());

                // 처리 완료 기록
                sagaRepo.markProcessed(sagaId, "INVENTORY_RESERVE");

                // InventoryReserved 이벤트 발행
                streamBridge.send("payment-events",
                    new InventoryReservedEvent(event.getOrderId(),
                        event.getAmount()));

            } catch (InsufficientStockException e) {
                // 재고 부족 → Saga 실패 이벤트
                streamBridge.send("order-events",
                    new InventoryReservationFailedEvent(
                        event.getOrderId(), "INSUFFICIENT_STOCK"));
            }
        };
    }

    // 재고 보상 (PaymentFailed 수신 시 재고 복구)
    @Bean
    public Consumer<PaymentFailedEvent> inventoryReleaser() {
        return event -> {
            String sagaId = event.getOrderId();

            // 멱등성: 이미 보상했으면 스킵
            if (sagaRepo.isCompensated(sagaId, "INVENTORY_RELEASE")) {
                return;
            }

            inventoryService.releaseStock(event.getProductId(),
                event.getQuantity());
            sagaRepo.markCompensated(sagaId, "INVENTORY_RELEASE");
        };
    }
}
```

### 2. Orchestration Saga — 상태 머신 기반 조율

```java
// OrderSaga: 중앙 오케스트레이터
// Spring StateMachine 또는 직접 구현

@Component
public class OrderSaga {

    // Saga 상태 정의
    public enum SagaState {
        STARTED,
        INVENTORY_RESERVING,
        INVENTORY_RESERVED,
        PAYMENT_PROCESSING,
        PAYMENT_COMPLETED,
        // 보상 상태
        INVENTORY_RELEASING,
        CANCELLED
    }

    @Autowired
    private SagaRepository sagaRepository;
    @Autowired
    private InventoryServiceClient inventoryClient;
    @Autowired
    private PaymentServiceClient paymentClient;

    // Saga 시작
    @Transactional
    public void startOrderSaga(String orderId, OrderRequest request) {
        // ① Saga 상태 저장 (영속화)
        SagaInstance saga = SagaInstance.builder()
            .id(orderId)
            .state(SagaState.STARTED)
            .payload(request)
            .build();
        sagaRepository.save(saga);

        // ② 첫 번째 단계 실행 (비동기)
        executeStep1(saga);
    }

    private void executeStep1(SagaInstance saga) {
        saga.setState(SagaState.INVENTORY_RESERVING);
        sagaRepository.save(saga);

        // InventoryService에 재고 예약 명령 (이벤트 또는 직접 호출)
        inventoryClient.reserveStock(
            saga.getPayload().getProductId(),
            saga.getPayload().getQuantity(),
            saga.getId()  // 응답 시 sagaId 포함
        );
    }

    // InventoryReserved 이벤트 수신 → 다음 단계
    @Bean
    public Consumer<InventoryReservedEvent> onInventoryReserved() {
        return event -> {
            SagaInstance saga = sagaRepository.findById(event.getSagaId());
            saga.setState(SagaState.INVENTORY_RESERVED);
            sagaRepository.save(saga);

            // 다음 단계: 결제 처리
            executeStep2(saga);
        };
    }

    private void executeStep2(SagaInstance saga) {
        saga.setState(SagaState.PAYMENT_PROCESSING);
        sagaRepository.save(saga);

        paymentClient.processPayment(
            saga.getPayload().getAmount(),
            saga.getId());
    }

    // PaymentCompleted → Saga 완료
    @Bean
    public Consumer<PaymentCompletedEvent> onPaymentCompleted() {
        return event -> {
            SagaInstance saga = sagaRepository.findById(event.getSagaId());
            saga.setState(SagaState.PAYMENT_COMPLETED);
            sagaRepository.save(saga);

            // Order 상태 업데이트
            orderService.confirmOrder(event.getSagaId());
        };
    }

    // PaymentFailed → 보상 트랜잭션 시작
    @Bean
    public Consumer<PaymentFailedEvent> onPaymentFailed() {
        return event -> {
            SagaInstance saga = sagaRepository.findById(event.getSagaId());

            // 보상: 재고 해제
            startCompensation(saga);
        };
    }

    private void startCompensation(SagaInstance saga) {
        saga.setState(SagaState.INVENTORY_RELEASING);
        sagaRepository.save(saga);

        // 역순으로 보상
        inventoryClient.releaseStock(
            saga.getPayload().getProductId(),
            saga.getPayload().getQuantity(),
            saga.getId());
    }
}
```

### 3. 보상 트랜잭션 설계 원칙

```java
// 보상 트랜잭션은 롤백이 아님
// → 새로운 로컬 트랜잭션으로 이전 효과를 취소하는 것

// ❌ 보상이 불가능한 작업:
// - 이메일 발송 → 이미 받은 이메일을 "취소"할 수 없음
// → 해결: "주문 취소 안내" 이메일을 새로 발송 (다른 이메일로 보상)

// ✅ 보상 가능한 작업:
// - 재고 차감 → 재고 복구
// - 포인트 사용 → 포인트 환불
// - DB 상태 변경 → 반대 상태로 변경

// 보상 트랜잭션 멱등성 보장:
@Service
public class CompensatingInventoryService {

    @Transactional
    public void releaseStock(String productId, int quantity, String sagaId) {
        // ① 이미 보상했는지 확인 (멱등성 키)
        Optional<CompensationRecord> existing =
            compensationRepo.findBySagaIdAndType(sagaId, "RELEASE_STOCK");

        if (existing.isPresent()) {
            log.info("Stock release already compensated for saga: {}", sagaId);
            return;  // 중복 보상 방지
        }

        // ② 실제 보상
        inventoryRepository.increaseStock(productId, quantity);

        // ③ 보상 완료 기록 (같은 트랜잭션)
        compensationRepo.save(new CompensationRecord(
            sagaId, "RELEASE_STOCK", Instant.now()));

        log.info("Stock released for product {} (sagaId: {})",
            productId, sagaId);
    }
}
```

### 4. Outbox 패턴 — 트랜잭션과 이벤트 발행의 원자성 보장

```java
// 문제: DB 저장 성공 + 이벤트 발행이 원자적이지 않음
// DB 저장 후, 이벤트 발행 전 서버 크래시 → 이벤트 유실

// Outbox 패턴 해결책:
// 이벤트를 DB에 같은 트랜잭션으로 저장 → 별도 프로세스가 발행

@Entity
@Table(name = "outbox")
public class OutboxEvent {
    @Id
    private String id;
    private String aggregateId;     // orderId
    private String eventType;       // "OrderCreated"
    private String payload;         // JSON
    private Instant createdAt;
    private boolean published;
}

@Transactional
public Order createOrder(OrderRequest request) {
    // ① 주문 저장
    Order order = orderRepository.save(request.toOrder());

    // ② 같은 트랜잭션에 이벤트도 Outbox에 저장
    outboxRepository.save(new OutboxEvent(
        UUID.randomUUID().toString(),
        order.getId(),
        "OrderCreated",
        objectMapper.writeValueAsString(new OrderCreatedEvent(order))));

    return order;
    // ③ 커밋: 주문 + OutboxEvent 동시에 저장
}

// Outbox Publisher (별도 스케줄러)
@Scheduled(fixedDelay = 1000)
public void publishOutboxEvents() {
    List<OutboxEvent> pending = outboxRepository.findByPublished(false);
    for (OutboxEvent event : pending) {
        streamBridge.send("order-events",
            objectMapper.readValue(event.getPayload(), OrderCreatedEvent.class));
        event.setPublished(true);
        outboxRepository.save(event);
    }
}

// 또는 Debezium (CDC) 사용:
// DB outbox 테이블 변경 → Kafka Connect → Kafka 토픽 자동 발행
// → 별도 Publisher 스케줄러 불필요
```

---

## 💻 실전 구성

### Docker Compose: Saga 환경

```yaml
services:
  kafka:
    image: confluentinc/cp-kafka:7.5.0

  order-service:
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://order-db:5432/orders

  inventory-service:
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://inventory-db:5432/inventory

  payment-service:
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://payment-db:5432/payments
```

### Saga 상태 모니터링

```bash
# Saga 인스턴스 상태 조회
curl http://localhost:8080/admin/sagas/{sagaId}
# {
#   "sagaId": "order-123",
#   "state": "PAYMENT_PROCESSING",
#   "createdAt": "2024-04-14T10:00:00Z",
#   "steps": [
#     {"step": "INVENTORY_RESERVE", "status": "COMPLETED", "at": "10:00:01"},
#     {"step": "PAYMENT_PROCESS", "status": "IN_PROGRESS", "at": "10:00:02"}
#   ]
# }
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: Payment 실패 시 보상 체인 실행

```
t=0:  OrderSaga.start(orderId=123)
      → Saga 상태: STARTED → DB 저장

t=1:  Step1: InventoryService.reserveStock()
      → 성공 → Saga 상태: INVENTORY_RESERVED

t=2:  Step2: PaymentService.processPayment()
      → 실패 (카드 한도 초과)
      → PaymentFailedEvent 발행

t=3:  Saga 보상 시작: Saga 상태: COMPENSATING
      → 보상 Step1: InventoryService.releaseStock()
      → 성공 → 재고 복구

t=4:  Saga 보상 완료: Saga 상태: CANCELLED
      → OrderService: 주문 CANCELLED 처리
      → 사용자에게 "재고는 있지만 결제 실패" 알림

최종 상태:
  Order: CANCELLED ✅
  Inventory: stock +1 (복구) ✅
  Payment: 거래 없음 ✅
  → 일관성 달성 (최종 일관성)
```

---

## ⚖️ 트레이드오프

| 방식 | 일관성 | 가용성 | 복잡도 | 디버깅 |
|------|--------|--------|--------|--------|
| **2PC** | 강한 일관성 | 낮음 (블로킹) | 중간 | 쉬움 |
| **Choreography Saga** | 최종 일관성 | 높음 | 높음 | 어려움 |
| **Orchestration Saga** | 최종 일관성 | 높음 | 매우 높음 | 중간 |

```
Choreography vs Orchestration 선택:
  Choreography (이벤트 기반 자율):
    ✅ 중앙 조율자 없음 → 서비스 독립성 높음
    ✅ 서비스 추가 용이 (이벤트 구독만 추가)
    ❌ 전체 흐름 파악 어려움 (이벤트 분산)
    ❌ 사이클 이벤트 위험 (A→B→A 무한 루프)
    적합: 단순한 선형 흐름, 3단계 이하

  Orchestration (중앙 조율):
    ✅ 전체 흐름이 한 곳에 (Saga 클래스)
    ✅ 디버깅 용이 (상태 추적 가능)
    ✅ 복잡한 흐름 (분기, 조건) 구현 용이
    ❌ Saga 오케스트레이터 자체가 복잡
    적합: 복잡한 비즈니스 흐름, 5단계 이상
```

---

## 📌 핵심 정리

```
Saga Pattern 핵심:

2PC의 문제:
  Phase1 → Phase2 사이 블로킹 → 가용성 희생
  코디네이터 SPOF → 전체 멈춤

Saga 원칙:
  긴 트랜잭션 = 로컬 트랜잭션들의 연쇄
  실패 시 = 보상 트랜잭션으로 역순 취소

보상 트랜잭션 ≠ 롤백:
  새로운 로컬 트랜잭션 (이전 효과를 새로운 방법으로 취소)
  멱등성 필수 (중복 보상 방지)

Choreography:
  이벤트로 서비스 간 자율 조율
  흐름 분산 → 이해 어려움

Orchestration:
  중앙 Saga 클래스가 명시적 조율
  상태 머신 + Saga 인스턴스 DB 저장

Outbox 패턴:
  DB 저장 + 이벤트 발행의 원자성 보장
  DB Outbox 테이블 → 별도 Publisher (또는 CDC)

멱등성 보장:
  sagaId + stepType을 복합 키로 중복 처리 방지
  보상 처리에도 동일하게 적용
```

---

## 🤔 생각해볼 문제

**Q1.** Choreography Saga에서 서비스 A가 이벤트를 발행하고 서비스 B가 그에 반응해 이벤트를 발행하고, 서비스 A가 다시 그 이벤트에 반응하면 무한 루프가 발생할 수 있다. 어떻게 방지하는가?

<details>
<summary>해설 보기</summary>

몇 가지 방어 전략이 있습니다.

1. **이벤트 타입 명확히 구분**: `OrderCreated`와 `OrderUpdated`를 별도 타입으로 분리해 A가 B의 이벤트를 구독하지 않도록 설계합니다.

2. **이벤트에 출처 포함**: `sourceService` 필드를 추가해 자신이 발행한 이벤트는 무시합니다.
```java
@Bean
public Consumer<OrderEvent> orderEventProcessor() {
    return event -> {
        if ("order-service".equals(event.getSourceService())) {
            return;  // 자신이 발행한 이벤트 무시
        }
        // 처리...
    };
}
```

3. **상태 기반 처리**: 이미 특정 상태에 있는 Saga는 특정 이벤트를 무시합니다.

4. **이벤트 버전 또는 시퀀스**: 이벤트에 버전/시퀀스를 추가해 이미 처리한 버전보다 오래된 이벤트는 무시합니다.

5. **Orchestration으로 전환**: 루프 위험이 높은 복잡한 흐름에서는 Orchestration Saga 패턴을 선택합니다.

</details>

---

**Q2.** Outbox 패턴에서 DB 저장은 성공했지만 Outbox Publisher가 Kafka 전송에 실패하면 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

Outbox 패턴의 핵심 장점이 여기서 드러납니다. Outbox 테이블에 `published=false`인 행이 남아있으므로 Publisher가 재시작하거나 다음 폴링 주기에 다시 전송을 시도합니다. Kafka 전송 성공 후 `published=true`로 업데이트합니다.

구현 고려사항:
1. **재시도 간격**: 지수 백오프로 Kafka 부하 방지
2. **최대 재시도**: 재시도 횟수 초과 시 Alert + Dead Letter Outbox
3. **락(Lock)**: 다수의 Publisher 인스턴스가 동시에 같은 이벤트를 처리하지 않도록 SELECT FOR UPDATE 또는 낙관적 락
4. **CDC 활용**: Debezium 같은 CDC 도구를 사용하면 DB binlog에서 변경 사항을 Kafka로 자동 전달 → 별도 Publisher 불필요, At-least-once 보장

</details>

---

**Q3.** Orchestration Saga에서 오케스트레이터 서비스(Order Service) 자체가 다운됐을 때, 진행 중인 Saga는 어떻게 복구되는가?

<details>
<summary>해설 보기</summary>

Saga 상태를 DB에 영속화하는 것이 핵심입니다. Order Service 재시작 시:

1. **미완료 Saga 조회**: `state NOT IN (PAYMENT_COMPLETED, CANCELLED)`인 Saga 인스턴스 조회
2. **상태별 재개**: 각 상태에서 멱등하게 다음 단계를 다시 실행
```java
@PostConstruct
public void recoverPendingSagas() {
    List<SagaInstance> pending = sagaRepository.findByStateNotIn(
        List.of(SagaState.PAYMENT_COMPLETED, SagaState.CANCELLED));

    for (SagaInstance saga : pending) {
        switch (saga.getState()) {
            case INVENTORY_RESERVING -> executeStep1(saga);  // 재실행 (멱등)
            case PAYMENT_PROCESSING -> executeStep2(saga);   // 재실행 (멱등)
            case INVENTORY_RELEASING -> startCompensation(saga);  // 보상 재실행
        }
    }
}
```

3. **멱등성 보장**: 각 단계가 이미 완료됐다면 중복 실행을 방어합니다 (sagaId + step type으로 체크).

이것이 Saga 상태를 DB에 저장하는 이유입니다 — 서비스 재시작 후 복구 가능성을 보장하기 위해서입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Event-Driven Architecture](./01-event-driven-architecture.md)** | **[홈으로 🏠](../README.md)** | **[다음: API Composition ➡️](./03-api-composition.md)**

</div>
