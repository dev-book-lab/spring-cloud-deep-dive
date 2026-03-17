# MDC를 통한 로그 추적 — traceId가 로그에 자동으로 찍히는 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `MDC`에 traceId · spanId가 자동으로 주입되는 정확한 시점과 코드 경로는?
- Logback 패턴에 `%X{traceId}`를 추가하는 방법과 JSON 로그 형식으로 출력하는 방법은?
- `@Async` 메서드에서 MDC 컨텍스트가 손실되는 근본 원인과 해결 방법은?
- Project Reactor(`Mono`/`Flux`) 체인에서 MDC 컨텍스트 전파가 어려운 이유는?
- `MDCAdapter`와 Reactor Context의 통합은 어떻게 구현하는가?
- ELK 스택에서 분산 로그를 traceId로 연결해 조회하는 방법은?

---

## 🔍 왜 MSA에서 필요한가

### Zipkin과 로그의 상호 보완

```
Zipkin(분산 추적)이 주는 것:
  전체 호출 체인 시각화
  각 서비스 소요 시간
  느린 지점, 오류 지점

로그가 주는 것:
  세밀한 애플리케이션 내부 동작
  비즈니스 로직 상태
  Span 단위보다 더 세밀한 디버그 정보

두 가지가 같은 traceId로 연결되면:
  Zipkin: "Payment Service에서 4.8초 소요됨"
  로그(traceId=abc123 검색):
    [INFO] Calling external payment API...
    [WARN] External API attempt 1 failed: timeout
    [WARN] External API attempt 2 failed: timeout
    [ERROR] Payment failed after 3 retries
  
  → 원인 분석: 외부 결제 API 재시도 3번 × 1.6초 = 4.8초
  → Zipkin만으로는 "왜" 4.8초인지 모름
  → 로그만으로는 어떤 요청인지 추적 어려움
  → 두 가지 조합 = 완전한 분석
```

---

## 😱 잘못된 구성

### Before: MDC 없이 분산 로그 분석 불가

```java
// ❌ MDC 없이 로그만 남기면
@Service
public class PaymentService {
    public void processPayment(Long orderId, Amount amount) {
        log.info("Processing payment for order {}", orderId);
        // [INFO] Processing payment for order 12345
        // 문제: 동시에 1000개 요청 처리 중이면
        //       로그가 뒤섞여 "어떤 요청의 로그인가?" 추적 불가
    }
}
```

### Before: @Async에서 MDC 컨텍스트 손실

```java
// ❌ @Async 메서드에서 traceId 손실
@Async
public CompletableFuture<Void> sendNotification(Long orderId) {
    // 새 스레드 → ThreadLocal MDC 없음 → traceId 없음
    log.info("Sending notification for order {}", orderId);
    // [INFO] Sending notification for order 12345
    // traceId 없음! → 어떤 요청의 알림인지 추적 불가
}
```

---

## ✨ 올바른 패턴

### After: MDC 자동 주입 + 로그 패턴 설정

```xml
<!-- logback-spring.xml -->
<configuration>
    <springProperty scope="context" name="serviceName"
                    source="spring.application.name"/>

    <!-- JSON 로그 형식 (Elasticsearch 수집용) -->
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"service":"${serviceName}"}</customFields>
            <!-- MDC 필드 자동 포함 -->
            <!-- traceId, spanId가 자동으로 JSON 필드로 포함됨 -->
        </encoder>
    </appender>

    <!-- 일반 텍스트 로그 (개발용) -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>
                %d{HH:mm:ss.SSS} %5p [${serviceName},%X{traceId},%X{spanId}] %logger{36} - %msg%n
            </pattern>
            <!-- %X{traceId}: MDC에서 traceId 읽기 -->
            <!-- %X{spanId}: MDC에서 spanId 읽기 -->
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

---

## 🔬 내부 동작 원리

### 1. MDC에 traceId가 주입되는 시점

```java
// MDC: Mapped Diagnostic Context
// SLF4J의 LoggingContext 확장
// ThreadLocal 기반 → 현재 스레드에서만 유효

// Micrometer Tracing + Brave 통합에서 자동 주입:
// MDCScopeDecorator.java 또는 BraveMDCScopeDecorator

public class MDCScopeDecorator implements CurrentTraceContext.ScopeDecorator {

    @Override
    public CurrentTraceContext.Scope decorateScope(TraceContext context,
            CurrentTraceContext.Scope scope) {

        // ① Span 시작 시: MDC에 traceId, spanId 주입
        String previousTraceId = MDC.get("traceId");
        String previousSpanId = MDC.get("spanId");

        if (context != null) {
            MDC.put("traceId", context.traceIdString());   // "abc123..."
            MDC.put("spanId", context.spanIdString());      // "def456..."
            if (context.parentId() != null) {
                MDC.put("parentId", context.parentIdString());
            }
        }

        // ② 범위(Scope) 반환: Span 종료 시 MDC 복원
        return () -> {
            // ③ Span 종료 시: MDC 이전 값으로 복원 (또는 제거)
            scope.close();
            MDC.put("traceId", previousTraceId != null ? previousTraceId : "");
            MDC.put("spanId", previousSpanId != null ? previousSpanId : "");
        };
    }
}

// 로그 출력 시:
// Logback의 %X{traceId} → MDC.get("traceId") 호출
// → ThreadLocal에서 현재 스레드의 traceId 반환
```

### 2. Spring MVC에서 자동 주입 흐름

```
HTTP 요청 수신
  ↓
ServerHttpObservationFilter (또는 TracingFilter)
  ↓
Propagator.extract() → TraceContext 생성 (또는 복원)
  ↓
tracer.withSpan(serverSpan).makeCurrent()
  ↓
CurrentTraceContext.newScope(traceContext)
  ↓
MDCScopeDecorator.decorateScope() 호출
  → MDC.put("traceId", traceId)
  → MDC.put("spanId", spanId)
  ↓
Controller 메서드 실행 (이 시점에 MDC 설정됨)
  ↓
모든 log.info() 호출 → MDC에서 traceId/spanId 읽음
  ↓
HTTP 응답 반환
  ↓
Scope.close() → MDC 이전 값으로 복원
```

### 3. @Async 메서드 — MDC 컨텍스트 전파 해결

```java
// 문제: @Async는 새 스레드 풀 스레드 사용 → ThreadLocal MDC 없음

// 해결 1: MDC 전파 TaskDecorator 커스터마이징
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);

        // ① TaskDecorator: 실행 전후 MDC 복사
        executor.setTaskDecorator(runnable -> {
            // 현재 스레드(호출 스레드)의 MDC 캡처
            Map<String, String> contextMap = MDC.getCopyOfContextMap();

            return () -> {
                try {
                    // 새 스레드에 MDC 복원
                    if (contextMap != null) {
                        MDC.setContextMap(contextMap);
                    }
                    runnable.run();
                } finally {
                    // 실행 완료 후 MDC 정리
                    MDC.clear();
                }
            };
        });

        executor.initialize();
        return executor;
    }
}

// 또는 Micrometer Tracing의 자동 전파 설정
// (Spring Boot 3.x에서 자동 설정 가능)
@Bean
public ObservationRegistry observationRegistry() {
    ObservationRegistry registry = ObservationRegistry.create();
    // AsyncTaskExecutor에 Observation 전파 설정
    return registry;
}
```

```java
// 해결 2: Micrometer Tracing Context Propagation 활용
// (Spring Boot 3.x 자동 설정)

// application.yml:
# spring.task.execution.thread-name-prefix: async-
# micrometer context propagation은 자동으로 @Async 지원

// 수동 컨텍스트 전파:
@Service
public class AsyncNotificationService {

    @Autowired
    private Tracer tracer;

    @Async
    public void sendNotificationAsync(Long orderId) {
        // 이미 Micrometer Tracing 자동 설정으로 컨텍스트 전파됨
        // Spring Boot 3.x: @Async + tracing 자동 통합
        log.info("Sending async notification for order {}", orderId);
        // → traceId, spanId 자동 포함
    }
}
```

### 4. Project Reactor — MDC 컨텍스트 전파

```java
// Reactor는 실행 스레드가 런타임에 변경될 수 있음
// ThreadLocal MDC가 자동으로 전파되지 않음

// 해결 1: micrometer-context-propagation 라이브러리 활용
// (Spring Boot 3.x + WebFlux 자동 설정)

// pom.xml
// <dependency>
//     <groupId>io.micrometer</groupId>
//     <artifactId>context-propagation</artifactId>
// </dependency>

// 자동 설정 시 Reactor Context → MDC 자동 전파
// 별도 설정 없이 WebFlux에서 traceId 자동 포함

// 수동 구현 (이해를 위한 예시):
public Mono<Order> getOrderReactive(Long orderId) {
    // ① 현재 TraceContext를 Reactor Context에 저장
    return Mono.deferContextual(ctx -> {
        // Reactor Context에서 traceId 읽기
        String traceId = ctx.getOrDefault("traceId", "");
        MDC.put("traceId", traceId);

        return Mono.fromSupplier(() -> orderRepository.findById(orderId))
            .doOnNext(order -> log.info("Found order: {}", order.getId()))
            // 비동기 전환 시점마다 Context 복원
            .contextWrite(Context.of("traceId", traceId));
    });
}

// Spring Boot 3.x + Micrometer context-propagation:
// Hooks.enableAutomaticContextPropagation() 활성화 시
// Reactor 체인에서 ThreadLocal(MDC) 자동 전파
```

### 5. Logstash JSON 로그 출력

```xml
<!-- logback-spring.xml (운영 환경 JSON 형식) -->
<configuration>
    <appender name="JSON_STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <!-- 서비스 이름 자동 포함 -->
            <customFields>
                {"service":"${SPRING_APPLICATION_NAME:-unknown}"}
            </customFields>
            <!-- MDC 필드 (traceId, spanId)는 자동으로 JSON 필드로 포함 -->
            <!-- 출력 예시:
            {
              "timestamp": "2024-04-14T10:00:00.000Z",
              "level": "INFO",
              "logger": "com.example.OrderService",
              "message": "Order 12345 processed",
              "service": "order-service",
              "traceId": "abc123def456...",
              "spanId": "789abc...",
              "thread": "http-nio-8080-exec-1"
            }
            -->
        </encoder>
    </appender>

    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="JSON_STDOUT"/>
        </root>
    </springProfile>

    <springProfile name="!prod">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>  <!-- 텍스트 형식 -->
        </root>
    </springProfile>
</configuration>
```

---

## 💻 실전 구성

### ELK 스택 연동

```yaml
# docker-compose.yml (ELK + Zipkin)
services:
  elasticsearch:
    image: elasticsearch:8.12.0
    environment:
      - discovery.type=single-node

  logstash:
    image: logstash:8.12.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    # logstash.conf: 서비스 로그 → Elasticsearch 인덱싱

  kibana:
    image: kibana:8.12.0
    ports:
      - "5601:5601"

  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"
    environment:
      - STORAGE_TYPE=elasticsearch
      - ES_HOSTS=http://elasticsearch:9200
```

### Kibana에서 traceId로 로그 연결 검색

```
Kibana Discover에서 트레이스 연결 분석:

1. Zipkin에서 느린 요청 traceId 발견: abc123def456

2. Kibana Discover 검색:
   traceId: abc123def456
   → 해당 요청의 모든 서비스 로그 조회

3. 타임라인 순으로 정렬:
   10:00:01.000 [gateway]       Request received
   10:00:01.005 [order-service] Processing order 12345
   10:00:01.010 [order-service] Calling inventory service
   10:00:01.015 [inventory]     Checking stock for product 789
   10:00:01.800 [inventory]     DB query took 785ms (warn!)
   ...

4. Zipkin과 교차 분석:
   Zipkin: inventory 800ms
   Kibana: "DB query took 785ms" → 인덱스 없는 쿼리

5. 해결: 인덱스 추가 → 30ms로 감소
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: @Async MDC 전파 문제 디버깅

```java
// 문제 재현:
@Service
public class NotificationService {

    @Async
    public void sendEmail(String userId) {
        log.info("Sending email to user: {}", userId);
        // 로그: [INFO] Sending email to user: user123
        // traceId 없음 → Kibana에서 이 로그 찾기 어려움
    }
}

// TaskDecorator 적용 후:
// 로그: [INFO] [order-service,abc123,def456] Sending email to user: user123
// traceId abc123으로 Kibana에서 검색 → 발견!

// 검증 방법:
@Test
void asyncMdcPropagationTest() {
    MDC.put("traceId", "test-trace-123");

    CompletableFuture<String> future = CompletableFuture.runAsync(
        () -> {
            String traceId = MDC.get("traceId");
            assertThat(traceId).isEqualTo("test-trace-123");
        },
        asyncExecutor  // TaskDecorator가 적용된 Executor
    );

    future.get();
    MDC.remove("traceId");
}
```

---

## ⚖️ 트레이드오프

| 방식 | 자동화 | 성능 | 적합한 환경 |
|------|--------|------|------------|
| **Spring MVC 자동** | 완전 자동 | 없음 | 동기 서블릿 환경 |
| **@Async + TaskDecorator** | 설정 필요 | 미미 | @Async 사용 환경 |
| **Reactor + context-propagation** | 라이브러리 필요 | 미미 | WebFlux 환경 |
| **수동 MDC 관리** | 없음 | 없음 | 특수 케이스 |

```
주의사항:
  MDC는 ThreadLocal 기반 → 스레드 전환 시 자동 전파 안 됨
  반드시 처리해야 하는 케이스:
  ① @Async → TaskDecorator
  ② CompletableFuture.runAsync() → 명시적 전파
  ③ Reactor(Mono/Flux) → context-propagation 라이브러리
  ④ 직접 Thread 생성 → 명시적 MDC 복사

  자동으로 처리되는 케이스:
  ① Spring MVC 요청 스레드 내
  ② 같은 스레드의 메서드 호출 체인
```

---

## 📌 핵심 정리

```
MDC 로그 추적 핵심:

자동 주입:
  HTTP 요청 수신 → ServerHttpObservationFilter/TracingFilter
  → CurrentTraceContext.newScope(traceContext)
  → MDCScopeDecorator.decorateScope()
  → MDC.put("traceId", ...) MDC.put("spanId", ...)
  → Span 종료 시 MDC 복원

Logback 설정:
  %X{traceId} → MDC.get("traceId") 읽기
  LogstashEncoder → JSON 자동 포함

비동기 전파 문제:
  @Async: TaskDecorator로 MDC 복사
  Reactor: context-propagation 라이브러리 (Boot 3.x 자동)
  CompletableFuture: 수동 MDC 캡처/복원

ELK 연동:
  JSON 로그 → Logstash → Elasticsearch
  Kibana에서 traceId로 검색
  Zipkin + ELK: 분산 추적 + 로그 교차 분석
```

---

## 🤔 생각해볼 문제

**Q1.** MDC가 ThreadLocal 기반인데, 서블릿 환경에서 Connection-per-Thread와 ThreadPool Reuse 시 MDC가 오염되는 케이스가 있는가?

<details>
<summary>해설 보기</summary>

있습니다. 스레드 풀에서 스레드를 재사용할 때 이전 요청의 MDC가 남아있을 수 있습니다. 예를 들어 요청 처리 중 예외가 발생해 Scope.close()가 호출되지 않으면, 해당 스레드의 MDC에 이전 traceId가 남습니다. 다음 요청이 같은 스레드를 사용하면 잘못된 traceId로 로그가 남습니다.

Micrometer Tracing의 `MDCScopeDecorator`는 try-finally 패턴으로 항상 cleanup을 보장합니다. Spring MVC의 `DispatcherServlet`도 요청 처리 전후로 정리합니다. 그러나 커스텀 코드에서 `MDC.put()`을 직접 호출하고 cleanup을 빠뜨리면 이 문제가 발생합니다. 항상 `try-finally { MDC.remove(key); }` 또는 `MDC.clear()` 패턴을 사용해야 합니다.

</details>

---

**Q2.** `%X{traceId}`가 로그에 빈 문자열로 나온다. 어디서 설정이 잘못된 것인가?

<details>
<summary>해설 보기</summary>

가능한 원인:

1. **micrometer-tracing 의존성 누락**: `micrometer-tracing-bridge-brave` 또는 `-otel`이 클래스패스에 없으면 MDC 자동 주입이 안 됩니다.
2. **MDC Key 이름 불일치**: Brave는 `traceId`, `spanId` 키를 사용하지만, Sleuth 2.x는 `X-B3-TraceId` 형식을 사용했습니다. Brave의 MDC Key를 확인:
```java
// Brave 기본 MDC Keys:
// "traceId", "spanId", "parentId"
// (구버전 Sleuth: "X-B3-TraceId")
```
3. **필터 순서 문제**: TracingFilter가 LoggingFilter보다 뒤에서 실행되면 로그 출력 시 MDC가 없음.
4. **비동기 코드**: @Async, CompletableFuture 등에서 MDC 전파 누락.
5. **자동 설정 비활성화**: `management.tracing.enabled=false` 설정.

</details>

---

**Q3.** Reactor의 `Mono.fromCallable()`과 `Mono.defer()`에서 MDC 컨텍스트 전파 방식이 다른가?

<details>
<summary>해설 보기</summary>

실질적으로 같습니다. 두 방법 모두 구독(subscribe) 시점에 코드가 실행되므로 컨텍스트 전파 문제는 동일합니다. 핵심은 `Hooks.enableAutomaticContextPropagation()`이 활성화됐는지 여부입니다.

`context-propagation` 라이브러리가 있고 자동 설정이 활성화됐다면, Reactor 연산자 체인에서 스레드가 바뀔 때마다 ThreadLocal(MDC 포함)이 자동으로 복원됩니다. `publishOn(Schedulers.boundedElastic())`으로 스레드가 바뀌어도 MDC가 전파됩니다.

활성화 방법:
```java
// Spring Boot 3.x 자동 설정 또는 수동:
Hooks.enableAutomaticContextPropagation();
```

이것이 없으면 `Mono.fromCallable()`이든 `Mono.defer()`이든 스레드가 바뀌는 순간 MDC 컨텍스트를 잃습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Zipkin Server 연동](./02-zipkin-integration.md)** | **[홈으로 🏠](../README.md)** | **[다음: Baggage Propagation ➡️](./04-baggage-propagation.md)**

</div>
