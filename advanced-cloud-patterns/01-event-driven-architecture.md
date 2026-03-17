# Event-Driven Architecture (Spring Cloud Stream) — 함수형 모델로 Kafka·RabbitMQ 연결하기

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@EnableBinding` 기반 레거시 방식과 함수형 프로그래밍 모델의 내부 구현 차이는 무엇인가?
- `Consumer<T>`, `Supplier<T>`, `Function<T,R>` Bean이 Spring Cloud Stream과 자동 연결되는 원리는?
- `Binder` 추상화 계층이 Kafka와 RabbitMQ를 동일한 코드로 연결할 수 있게 하는 이유는?
- `MessageChannel`과 `BindingService`가 채널 이름을 토픽/큐 이름으로 변환하는 과정은?
- 이벤트 직렬화/역직렬화(`ContentTypeConverter`)는 어떻게 동작하는가?
- 컨슈머 그룹(Consumer Group)과 파티셔닝으로 메시지 처리를 확장하는 방법은?

---

## 🔍 왜 MSA에서 필요한가

### 동기 호출의 한계와 이벤트 기반의 장점

```
동기 호출 (RestTemplate, WebClient):
  Order Service → Payment Service (HTTP)
  Order Service → Inventory Service (HTTP)
  Order Service → Notification Service (HTTP)

  문제:
  ① 시간적 결합: Payment가 다운되면 주문 불가
  ② 공간적 결합: Payment URL을 알아야 함
  ③ 연쇄 장애: Payment 느리면 Order도 느려짐
  ④ 확장 어려움: 새 서비스 추가 → Order 코드 수정

이벤트 기반 (Spring Cloud Stream + Kafka):
  Order Service → "OrderCreated 이벤트" → Kafka 토픽
    ← Payment Service: 이벤트 구독, 결제 처리
    ← Inventory Service: 이벤트 구독, 재고 차감
    ← Notification Service: 이벤트 구독, 알림 발송

  장점:
  ① 시간적 독립: Payment 다운돼도 주문 가능 (이벤트 버퍼)
  ② 느슨한 결합: 서비스 간 직접 의존 없음
  ③ 확장 용이: 새 서비스 추가 = 이벤트 구독만 추가

  트레이드오프:
  최종 일관성(Eventual Consistency) — 즉각적 일관성 대신
  복잡성 증가 — 메시지 브로커 운영 필요
  디버깅 어려움 — 비동기 흐름 추적
```

---

## 😱 잘못된 구성

### Before: @EnableBinding 레거시 방식 (deprecated)

```java
// ❌ Spring Cloud Stream 3.x 이전 방식 (2021년 deprecated)
@EnableBinding(Source.class)  // ❌ 레거시
@SpringBootApplication
public class OrderApplication { }

// ❌ MessageChannel 직접 사용
@Service
public class OrderService {

    @Autowired
    private Source source;   // ❌ @EnableBinding에서 자동 생성

    public void createOrder(Order order) {
        orderRepository.save(order);
        // ❌ MessageChannel 직접 주입
        source.output().send(
            MessageBuilder.withPayload(new OrderCreatedEvent(order.getId()))
                .build());
    }
}

// ❌ @StreamListener (deprecated)
@StreamListener(Sink.INPUT)
public void handleOrderCreated(OrderCreatedEvent event) {
    paymentService.processPayment(event.getOrderId());
}
```

---

## ✨ 올바른 패턴

### After: 함수형 프로그래밍 모델 (Spring Cloud Stream 3.x+)

```java
// ✅ 간단하고 선언적인 함수형 방식
// @EnableBinding 없음, @StreamListener 없음

@SpringBootApplication
public class OrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }
}

// ① Supplier<T>: 주기적으로 메시지 발행 (폴링 방식)
// → Kafka/RabbitMQ 토픽으로 자동 전송
@Bean
public Supplier<OrderCreatedEvent> orderSupplier() {
    return () -> {
        // 주기적으로 호출됨 (기본 1초)
        return pendingOrderQueue.poll();
    };
}

// ② Consumer<T>: 메시지 구독 및 처리
// → Kafka/RabbitMQ 토픽에서 자동 수신
@Bean
public Consumer<OrderCreatedEvent> paymentProcessor() {
    return event -> {
        log.info("Processing payment for order: {}", event.getOrderId());
        paymentService.processPayment(event.getOrderId());
    };
}

// ③ Function<T,R>: 변환 처리 (입력 → 처리 → 출력)
// → 한 토픽 수신 + 다른 토픽 발행
@Bean
public Function<OrderCreatedEvent, PaymentRequestEvent> orderToPayment() {
    return event -> {
        log.info("Transforming order {} to payment request", event.getOrderId());
        return new PaymentRequestEvent(event.getOrderId(), event.getAmount());
    };
}
```

---

## 🔬 내부 동작 원리

### 1. 함수형 Bean 자동 바인딩 원리

```java
// FunctionConfiguration.java (Spring Cloud Stream 내부)
// @Bean으로 등록된 Function/Consumer/Supplier를 자동으로 스트림에 바인딩

@Configuration
public class FunctionConfiguration {

    @Autowired
    private BindingService bindingService;

    // Spring이 시작 시 자동으로 처리:
    // ① ApplicationContext에서 Function/Consumer/Supplier Bean 탐색
    // ② 각 Bean의 타입에 따라 바인딩 방향 결정
    //    Supplier → output 채널 (메시지 발행)
    //    Consumer → input 채널 (메시지 수신)
    //    Function → input + output 채널 (변환)

    @PostConstruct
    void bindFunctions() {
        // 예: paymentProcessor Bean 발견
        // → Consumer<OrderCreatedEvent> 타입
        // → input 채널: "paymentProcessor-in-0"
        // → 설정: spring.cloud.stream.bindings.paymentProcessor-in-0.destination=order-events
        // → Kafka 토픽 "order-events" 구독 시작
    }
}
```

### 2. Binder 추상화 — Kafka vs RabbitMQ 동일 코드

```java
// Binder 인터페이스: 메시지 브로커 추상화
public interface Binder<T, C extends ConsumerProperties, P extends ProducerProperties> {

    // 입력 채널과 메시지 브로커 연결
    Binding<T> bindConsumer(
        String name,          // 채널 이름
        String group,         // 컨슈머 그룹
        T inboundBindTarget,  // 메시지 수신 대상
        C consumerProperties  // Kafka/RabbitMQ 특화 설정
    );

    // 출력 채널과 메시지 브로커 연결
    Binding<T> bindProducer(
        String name,           // 채널 이름
        T outboundBindTarget,  // 메시지 발신 대상
        P producerProperties   // Kafka/RabbitMQ 특화 설정
    );
}

// KafkaMessageChannelBinder: Kafka 구현체
// RabbitMessageChannelBinder: RabbitMQ 구현체
// → 같은 Binder 인터페이스 구현
// → 애플리케이션 코드 변경 없이 브로커 교체 가능
// → pom.xml에서 binder 의존성만 교체

// Kafka:
// spring-cloud-stream-binder-kafka

// RabbitMQ:
// spring-cloud-stream-binder-rabbit
```

### 3. MessageChannel과 메시지 흐름

```java
// Spring Cloud Stream의 메시지 처리 파이프라인:

// 발행 흐름 (Supplier/Function 출력):
// Bean 실행 → Message 생성
//   → DirectChannel (인메모리 MessageChannel)
//   → BindingService → KafkaBinder
//   → KafkaProducer → Kafka 토픽

// 수신 흐름 (Consumer/Function 입력):
// Kafka 토픽 → KafkaConsumer
//   → KafkaMessageChannelBinder → MessageChannel
//   → DirectChannel → Consumer Bean 호출

// MessageChannel 자동 생성:
// paymentProcessor-in-0 → DirectChannel Bean
// 이름 규칙: {functionName}-{in/out}-{index}
// in: 입력, out: 출력, index: 다중 입출력 시 순서

@Bean
public Consumer<OrderCreatedEvent> paymentProcessor() { ... }
// → 자동으로 "paymentProcessor-in-0" DirectChannel 생성
// → application.yml에서 이 채널을 Kafka 토픽에 매핑
```

### 4. 메시지 직렬화/역직렬화

```java
// ContentTypeConverter: 메시지 페이로드 변환
// Spring Cloud Stream이 자동으로 처리

// 발행 시:
// OrderCreatedEvent (Java 객체)
// → Jackson ObjectMapper → JSON 문자열
// → MessageBuilder.withPayload(json)
//     .setHeader(MessageHeaders.CONTENT_TYPE, "application/json")
//     .build()
// → Kafka 토픽 전송

// 수신 시:
// Kafka 토픽 → JSON 문자열
// → MessageConverter 탐색 (application/json)
// → MappingJackson2MessageConverter
// → OrderCreatedEvent (Java 객체)
// → Consumer<OrderCreatedEvent>.accept(event) 호출

// 커스텀 직렬화 (Avro, Protobuf 등):
@Bean
public MessageConverter customConverter() {
    return new AbstractMessageConverter(new MimeType("application", "avro")) {
        @Override
        protected Object convertFromInternal(Message<?> message,
                Class<?> targetClass, Object conversionHint) {
            // Avro 역직렬화
            return avroDeserializer.deserialize(
                (byte[]) message.getPayload(), targetClass);
        }
    };
}
```

### 5. 컨슈머 그룹과 파티셔닝

```yaml
# application.yml
spring:
  cloud:
    stream:
      bindings:
        paymentProcessor-in-0:
          destination: order-events        # Kafka 토픽 이름
          group: payment-service           # 컨슈머 그룹
          # 같은 group의 인스턴스들이 토픽 파티션을 나눠서 처리
          # payment-service 인스턴스 3개 → 각각 파티션의 1/3씩 처리
          # 동일 메시지를 여러 서비스가 받으려면: 다른 group 이름 사용

        inventoryProcessor-in-0:
          destination: order-events        # 같은 Kafka 토픽
          group: inventory-service         # 다른 그룹 → 모든 메시지 독립적으로 수신

      # 파티셔닝: 특정 키로 같은 파티션에 라우팅
      kafka:
        bindings:
          orderSupplier-out-0:
            producer:
              partition-key-expression: payload.userId  # userId 기반 파티셔닝
              partition-count: 3                         # 파티션 수
```

```java
// 파티셔닝 효과:
// userId=user-1의 모든 이벤트 → 파티션 0 → 인스턴스 A (순서 보장)
// userId=user-2의 모든 이벤트 → 파티션 1 → 인스턴스 B (순서 보장)
// userId=user-3의 모든 이벤트 → 파티션 2 → 인스턴스 C (순서 보장)
// → 같은 사용자의 이벤트는 항상 순서 처리됨
```

---

## 💻 실전 구성

### 완전한 주문 이벤트 처리 예시

```java
// 이벤트 모델
@Data
public class OrderCreatedEvent {
    private String orderId;
    private String userId;
    private BigDecimal amount;
    private List<OrderItem> items;
    private Instant occurredAt;
}

// Order Service (발행자)
@Service
public class OrderService {

    // StreamBridge: 프로그래매틱 메시지 발행 (동적 채널)
    @Autowired
    private StreamBridge streamBridge;

    @Transactional
    public Order createOrder(OrderRequest request) {
        Order order = orderRepository.save(request.toOrder());

        // ① DB 저장 성공 후 이벤트 발행
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(), order.getUserId(), order.getAmount());

        // "order-events" 채널로 발행 (설정에서 Kafka 토픽과 매핑)
        streamBridge.send("order-events", event);

        return order;
    }
}

// Payment Service (구독자)
@Configuration
public class PaymentEventConfig {

    @Bean
    public Consumer<OrderCreatedEvent> paymentProcessor(
            PaymentService paymentService) {
        return event -> {
            log.info("[payment] Processing order: {}", event.getOrderId());
            paymentService.processPayment(
                event.getOrderId(), event.getAmount());
        };
    }
}

// Notification Service (구독자)
@Configuration
public class NotificationEventConfig {

    @Bean
    public Consumer<OrderCreatedEvent> notificationSender(
            NotificationService notificationService) {
        return event -> {
            log.info("[notification] Sending confirmation for: {}",
                event.getOrderId());
            notificationService.sendOrderConfirmation(
                event.getUserId(), event.getOrderId());
        };
    }
}
```

```yaml
# Order Service application.yml
spring:
  cloud:
    stream:
      default-binder: kafka
      bindings:
        order-events:
          destination: order-events-topic
  kafka:
    bootstrap-servers: kafka:9092

# Payment Service application.yml
spring:
  cloud:
    function:
      definition: paymentProcessor   # 활성화할 Bean 이름 명시
    stream:
      bindings:
        paymentProcessor-in-0:
          destination: order-events-topic
          group: payment-service-group

# Notification Service application.yml
spring:
  cloud:
    function:
      definition: notificationSender
    stream:
      bindings:
        notificationSender-in-0:
          destination: order-events-topic
          group: notification-service-group   # 다른 그룹 → 독립 수신
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: Payment Service 다운 중 주문 발생

```
t=0s:   Payment Service 다운 (배포 중)

t=10s:  사용자가 주문 생성
         Order Service → OrderCreated 이벤트 → Kafka 토픽
         Kafka: 이벤트 안전하게 보관 (durable)

t=10s~60s: Payment Service 다운 상태
            Order Service: 정상 동작 (주문 계속 받음)
            Kafka: 이벤트들 쌓임 (메시지 보존 기간 내)

t=60s:  Payment Service 복구
         Kafka에서 미처리 이벤트 일괄 수신
         → 복구 중 쌓인 모든 주문에 대해 결제 처리
         → 처리 순서: Kafka 파티션 오프셋 순서대로

결과:
  Payment 다운 중에도 주문 가능
  복구 후 자동으로 모든 이벤트 처리
  데이터 손실 없음 (Kafka 내구성)

동기 방식이었다면:
  Payment 다운 → 주문 불가 → 고객 서비스 중단
```

### 시나리오: 멱등성 처리 (중복 메시지 방어)

```java
// Kafka 컨슈머 재처리(Rebalance, 재시작) 시 중복 메시지 수신 가능
// → 멱등성 처리 필수

@Bean
public Consumer<OrderCreatedEvent> paymentProcessor() {
    return event -> {
        String orderId = event.getOrderId();

        // ① 이미 처리된 이벤트인지 확인 (Redis 또는 DB)
        if (processedEventCache.contains(orderId)) {
            log.info("Skipping duplicate event for order: {}", orderId);
            return;  // 중복 → 스킵
        }

        try {
            // ② 결제 처리
            paymentService.processPayment(orderId, event.getAmount());
            // ③ 처리 완료 마킹
            processedEventCache.add(orderId, Duration.ofDays(7));
        } catch (Exception e) {
            log.error("Payment failed for order: {}", orderId, e);
            throw e;  // 예외 → Kafka가 재시도
        }
    };
}
```

---

## ⚖️ 트레이드오프

| 방식 | 결합도 | 응답성 | 복잡도 | 일관성 |
|------|--------|--------|--------|--------|
| **동기 HTTP 호출** | 높음 | 즉각 | 낮음 | 강한 일관성 |
| **EDA (이벤트 기반)** | 낮음 | 지연 | 높음 | 최종 일관성 |
| **혼합 (중요 동기, 부가 비동기)** | 중간 | 중간 | 중간 | 핵심은 강함 |

```
함수형 모델 vs @EnableBinding:
  @EnableBinding (deprecated):
    MessageChannel을 직접 주입해 사용
    @StreamListener로 수신 처리
    채널 설정이 코드에 의존
  
  함수형 모델 (권장):
    표준 Java 함수형 인터페이스 (Consumer/Supplier/Function)
    Spring Cloud Stream이 채널을 자동 생성/연결
    스트리밍 없이도 테스트 가능 (순수 함수)
    브로커 교체: pom.xml + yml만 변경

Supplier vs StreamBridge:
  Supplier: 폴링 방식 (정기적 실행)
             배치 처리, 스케줄 발행에 적합
  StreamBridge: 명시적 발행 (트랜잭션 내 사용 가능)
                요청-응답 흐름에서 이벤트 발행에 적합
```

---

## 📌 핵심 정리

```
Spring Cloud Stream 함수형 모델 핵심:

함수형 Bean 역할:
  Supplier<T>: 메시지 발행 (폴링)
  Consumer<T>: 메시지 수신·처리
  Function<T,R>: 수신 + 변환 + 발행

자동 바인딩 채널 이름:
  {functionName}-in-{index}   (Consumer/Function 입력)
  {functionName}-out-{index}  (Supplier/Function 출력)

Binder 추상화:
  KafkaMessageChannelBinder / RabbitMessageChannelBinder
  → 동일 코드로 Kafka·RabbitMQ 연결
  → pom.xml에서 binder 의존성 교체로 브로커 변경

컨슈머 그룹:
  같은 그룹: 파티션 분산 처리 (수평 확장)
  다른 그룹: 동일 이벤트 독립 수신 (여러 서비스 구독)

StreamBridge:
  @Transactional 내에서 이벤트 발행 시 사용
  streamBridge.send("채널명", payload)
```

---

## 🤔 생각해볼 문제

**Q1.** `Consumer<OrderCreatedEvent>` Bean이 처리 중 예외를 던지면 Kafka에서 어떻게 처리되는가? 재시도는 자동으로 되는가?

<details>
<summary>해설 보기</summary>

Consumer Bean에서 예외가 발생하면 Spring Cloud Stream의 에러 처리 체인이 동작합니다.

1. **재시도**: `spring.cloud.stream.bindings.paymentProcessor-in-0.consumer.max-attempts=3`으로 재시도 횟수 설정. 기본 3회 재시도 후 실패.

2. **DLQ (Dead Letter Queue)**: 모든 재시도 실패 시 Dead Letter 토픽으로 이동.
```yaml
spring:
  cloud:
    stream:
      kafka:
        bindings:
          paymentProcessor-in-0:
            consumer:
              enableDlq: true
              dlqName: order-events-dlq  # 실패 메시지 별도 토픽
```

3. **Kafka 오프셋**: 예외 발생 시 해당 오프셋을 커밋하지 않음 → Kafka가 같은 메시지를 다시 전달.

4. **멱등성 필수**: 재시도 시 같은 메시지를 여러 번 처리하므로 멱등성 구현이 필수입니다.

</details>

---

**Q2.** `Supplier<T>` Bean의 폴링 주기를 변경하는 방법과 `Supplier`가 `null`을 반환하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

폴링 주기 변경:
```yaml
spring:
  cloud:
    stream:
      poller:
        fixed-delay: 5000      # 5초마다 폴링 (기본: 1000ms)
        initial-delay: 0
        cron: "0 * * * * *"   # 또는 cron 표현식
```

`Supplier`가 `null`을 반환하면: 해당 주기에 메시지를 발행하지 않고 다음 폴링까지 기다립니다. 예외가 발생하지 않으며 조용히 스킵됩니다. 이를 이용해 "보낼 데이터 없음"을 표현할 수 있습니다:
```java
@Bean
public Supplier<OrderCreatedEvent> orderSupplier() {
    return () -> {
        OrderCreatedEvent pending = queue.poll();
        return pending;  // null이면 이번 폴링은 발행 없음
    };
}
```

실시간 이벤트 발행이 필요하면 `Supplier` 대신 `StreamBridge`를 사용하는 것이 권장됩니다.

</details>

---

**Q3.** `Function<T,R>`을 사용해 이벤트 변환 처리 시, 처리 중 예외가 발생했을 때 입력 메시지와 출력 메시지의 상태는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`Function<T,R>`에서 예외 발생 시:
- **입력 메시지**: 오프셋이 커밋되지 않아 Kafka가 같은 메시지를 다시 전달합니다 (재시도 설정에 따라).
- **출력 메시지**: 예외 발생 전까지 생성된 출력은 발행되지 않습니다. `R`이 반환되기 전에 예외가 던져지면 출력 채널로 아무것도 전달되지 않습니다.

주의할 점: 정상 처리와 예외 사이에 출력이 부분적으로 발행된 경우(Reactive `Flux<R>` 반환 시) 더 복잡한 상황이 됩니다. 일반적으로 `Function<T,R>`은 원자적으로 동작하도록 설계해야 합니다. 부분 실패가 우려되면 명시적 트랜잭션이나 아웃박스(Outbox) 패턴을 고려해야 합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Saga Pattern (분산 트랜잭션) ➡️](./02-saga-pattern.md)**

</div>
