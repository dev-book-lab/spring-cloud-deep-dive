# Baggage Propagation — 비즈니스 컨텍스트를 서비스 간에 전파하는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Baggage`와 `Span 태그`의 차이는 무엇인가? 언제 Baggage를 선택하는가?
- `BaggageField`의 remote와 local 구분은 무엇을 의미하는가?
- Baggage가 HTTP 헤더로 전파될 때 사용하는 헤더 이름은 어떻게 결정되는가?
- 사용자 ID, 테넌트 ID를 모든 서비스 로그에 자동으로 포함시키는 완전한 구성 방법은?
- `BaggagePropagation`의 설정과 `baggageField.updateValue()` API 사용법은?
- Baggage 크기 제한과 과도한 Baggage 사용의 문제점은?

---

## 🔍 왜 MSA에서 필요한가

### HTTP 헤더 직접 전달과 Baggage의 차이

```
비즈니스 컨텍스트를 여러 서비스에 전달해야 하는 상황:
  Client → Gateway → Order → Inventory → Payment

전달이 필요한 정보:
  userId: "user-123"      (인증된 사용자)
  tenantId: "tenant-abc"  (멀티테넌트)
  correlationId: "req-xyz" (비즈니스 추적)

방법 1: 모든 메서드 파라미터로 전달
  getOrder(Long id, String userId, String tenantId, ...)
  → 매 메서드마다 파라미터 추가 → 코드 오염
  → 서비스 내부 코드가 트레이싱 컨텍스트를 알아야 함

방법 2: 각 서비스가 헤더를 직접 추출해서 다음 서비스에 전달
  String userId = httpRequest.getHeader("X-User-Id");
  nextServiceCall.header("X-User-Id", userId);
  → 매 서비스마다 헤더 추출/추가 코드 중복
  → 빠뜨리면 전파 끊김

방법 3: Baggage (분산 추적 컨텍스트와 함께 자동 전파)
  BaggageField.create("user-id")   // 필드 정의
  userIdField.updateValue("user-123")  // 값 설정
  → Trace Context와 함께 자동으로 모든 서비스에 전파
  → HTTP 헤더 수동 관리 불필요
  → MDC에도 자동으로 추가됨 (로그에 자동 포함)
```

---

## 😱 잘못된 구성

### Before: 모든 서비스가 수동으로 헤더 전달

```java
// ❌ 각 서비스마다 헤더 추출 + 전달 코드 중복
@RestController
public class OrderController {

    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable Long id,
            HttpServletRequest request) {
        // 헤더 추출
        String userId = request.getHeader("X-User-Id");
        String tenantId = request.getHeader("X-Tenant-Id");

        // 서비스 내부로 전달 (파라미터 오염)
        return orderService.findById(id, userId, tenantId);
    }
}

@Service
public class OrderService {
    public Order findById(Long id, String userId, String tenantId) {
        // ❌ 매 서비스마다 다음 서비스 호출 시 헤더 추가
        inventoryClient.check(productId)  // userId, tenantId 어떻게?
        // → RestTemplate 인터셉터에서 ThreadLocal에서 꺼내거나
        //   파라미터로 계속 전달해야 함
    }
}
```

### Before: Baggage 필드를 너무 많이 정의

```yaml
# ❌ 너무 많은 Baggage 필드
management:
  tracing:
    baggage:
      remote-fields:
        - user-id
        - tenant-id
        - session-id
        - request-id
        - feature-flags    # ❌ 수십 개의 feature flag
        - ab-test-variant
        - locale
        - timezone
        - device-type
        - app-version
        # ...
```

```
문제:
  모든 HTTP 요청마다 이 헤더들이 추가됨
  헤더 수 × 모든 서비스 간 호출 = 네트워크 오버헤드
  
  Baggage는 "꼭 전파해야 할 소수의 컨텍스트"에만 사용
  대용량 데이터 전달 용도로 부적합
```

---

## ✨ 올바른 패턴

### After: 핵심 비즈니스 컨텍스트만 Baggage로 전파

```yaml
# application.yml
management:
  tracing:
    baggage:
      # 원격 전파 (HTTP 헤더로 서비스 간 전파)
      remote-fields:
        - user-id         # 인증된 사용자 ID
        - tenant-id       # 멀티테넌트 구분

      # 로컬에서 MDC로도 접근 가능하게 (로그에 자동 포함)
      correlation-fields:
        - user-id
        - tenant-id
```

```java
// Gateway에서 Baggage 설정 (최초 진입 지점)
@Component
@Order(-100)
public class BaggageInitFilter implements GlobalFilter {

    private final BaggageField USER_ID = BaggageField.create("user-id");
    private final BaggageField TENANT_ID = BaggageField.create("tenant-id");

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // JWT에서 userId, tenantId 추출 후 Baggage에 설정
        String userId = extractUserId(exchange);
        String tenantId = extractTenantId(exchange);

        USER_ID.updateValue(userId);
        TENANT_ID.updateValue(tenantId);

        return chain.filter(exchange);
        // 이후 모든 서비스 호출 시 자동으로 Baggage 전파
    }
}

// 하위 서비스에서 Baggage 읽기 (헤더 추출 불필요)
@Service
public class InventoryService {

    private final BaggageField USER_ID = BaggageField.create("user-id");

    public Inventory checkInventory(Long productId) {
        // Baggage에서 userId 자동으로 읽기
        String userId = USER_ID.getValue();

        log.info("Checking inventory for product {} (user: {})",
            productId, userId);
        // MDC에도 자동 추가 → 로그에 userId 포함됨

        return inventoryRepository.findByProductId(productId);
    }
}
```

---

## 🔬 내부 동작 원리

### 1. BaggageField — Baggage 필드 정의와 접근

```java
// BaggageField.java (Brave)
public final class BaggageField {

    private final String name;  // HTTP 헤더 이름 (예: "user-id" → X-B3-user-id)

    // 정적 팩토리: 필드 이름으로 생성
    public static BaggageField create(String name) {
        return new BaggageField(name);
    }

    // 현재 TraceContext에서 Baggage 값 읽기
    public String getValue() {
        return getValue(currentTraceContext().get());
    }

    // 특정 TraceContext에서 Baggage 값 읽기
    public String getValue(TraceContext context) {
        if (context == null) return null;
        // TraceContext에 연결된 BaggageValues에서 조회
        ExtraBaggageFields extra = ExtraBaggageFields.findExtra(
            ExtraBaggageFields.class, context.extra());
        return extra != null ? extra.getValue(this) : null;
    }

    // 현재 TraceContext에 Baggage 값 설정
    public boolean updateValue(String value) {
        TraceContext context = currentTraceContext().get();
        if (context == null) return false;

        ExtraBaggageFields extra = ExtraBaggageFields.findExtra(
            ExtraBaggageFields.class, context.extra());
        if (extra == null) return false;

        extra.updateValue(this, value);
        return true;
    }
}
```

### 2. Baggage 전파 — HTTP 헤더로 주입/추출

```java
// BaggagePropagation.java
// Baggage 전용 Propagator
// Trace Context와 함께 HTTP 헤더에 Baggage 추가

// inject: 발신 요청에 Baggage 헤더 추가
// 예: user-id=user-123 → X-B3-Bag-user-id: user-123
// (B3 전파 방식: X-B3-Bag-{field-name})
// W3C baggage 헤더: baggage: user-id=user-123,tenant-id=abc

// extract: 수신 요청에서 Baggage 헤더 추출
// "X-B3-Bag-user-id: user-123" → BaggageField("user-id").value = "user-123"

// 설정:
BaggagePropagation.newFactoryBuilder(B3Propagation.FACTORY)
    .add(BaggagePropagationConfig.SingleBaggageField.remote(
        BaggageField.create("user-id")))        // remote: 서비스 간 전파
    .add(BaggagePropagationConfig.SingleBaggageField.local(
        BaggageField.create("debug-flag")))     // local: 같은 프로세스 내만
    .build();
```

### 3. remote vs local BaggageField

```java
// remote 필드: HTTP 헤더로 서비스 간 전파됨
BaggagePropagationConfig.SingleBaggageField.remote(
    BaggageField.create("user-id"))
// → 발신 요청 헤더에 "user-id" 값 포함
// → 수신 서비스도 이 값 읽을 수 있음

// local 필드: 같은 프로세스 내에서만 유효
BaggagePropagationConfig.SingleBaggageField.local(
    BaggageField.create("debug-flag"))
// → HTTP 헤더로 전달 안 됨
// → 같은 서비스의 메서드 간 전달만 (ThreadLocal과 유사)
// 사용 케이스: 중간 계산 결과, 임시 플래그 등

// 커스텀 헤더 이름 지정:
BaggagePropagationConfig.SingleBaggageField
    .newBuilder(BaggageField.create("user-id"))
    .addKeyName("x-user-id")    // HTTP 헤더 이름 명시
    .addKeyName("X-User-Id")    // 여러 이름 지원 (읽기 시)
    .build();
```

### 4. MDC 자동 연동 — correlation-fields

```java
// correlation-fields 설정 시 Baggage → MDC 자동 연동
// management.tracing.baggage.correlation-fields: [user-id, tenant-id]

// 내부 동작 (CorrelationScopeDecorator):
public class CorrelationScopeDecorator implements CurrentTraceContext.ScopeDecorator {

    private final List<BaggageField> correlationFields;

    @Override
    public CurrentTraceContext.Scope decorateScope(TraceContext context, ...) {
        // Scope 진입 시: Baggage → MDC 동기화
        Map<String, String> previousValues = new LinkedHashMap<>();

        for (BaggageField field : correlationFields) {
            String value = field.getValue(context);
            String mdcKey = field.name();   // Baggage 이름 = MDC 키

            previousValues.put(mdcKey, MDC.get(mdcKey));

            if (value != null) {
                MDC.put(mdcKey, value);     // MDC에 Baggage 값 복사
            } else {
                MDC.remove(mdcKey);
            }
        }

        return () -> {
            // Scope 종료 시: MDC 이전 값으로 복원
            for (Map.Entry<String, String> entry : previousValues.entrySet()) {
                if (entry.getValue() != null) {
                    MDC.put(entry.getKey(), entry.getValue());
                } else {
                    MDC.remove(entry.getKey());
                }
            }
        };
    }
}

// 결과:
// Baggage의 user-id → MDC의 user-id
// 로그 패턴: %X{user-id} → "user-123" 자동 출력
// → 로그에 사용자 ID 자동 포함, 추가 코드 없음
```

### 5. 멀티테넌트 전파 실전 패턴

```java
// 테넌트별 격리 패턴
@Configuration
public class BaggageConfig {

    // Spring Boot 3.x 설정 방식
    @Bean
    public BaggagePropagation.FactoryBuilder baggagePropagationFactoryBuilder() {
        return BaggagePropagation.newFactoryBuilder(B3Propagation.FACTORY)
            // 사용자 ID: 서비스 간 전파 + MDC 연동
            .add(BaggagePropagationConfig.SingleBaggageField
                .newBuilder(BaggageField.create("user-id"))
                .addKeyName("x-user-id")        // 커스텀 헤더 이름
                .build())

            // 테넌트 ID: 서비스 간 전파
            .add(BaggagePropagationConfig.SingleBaggageField.remote(
                BaggageField.create("tenant-id")));
    }
}

// 테넌트별 DB 라우팅 예시
@Component
public class TenantDataSourceRouter extends AbstractRoutingDataSource {

    private final BaggageField TENANT_ID = BaggageField.create("tenant-id");

    @Override
    protected Object determineCurrentLookupKey() {
        // Baggage에서 테넌트 ID 읽어 DB 라우팅 결정
        String tenantId = TENANT_ID.getValue();
        return tenantId != null ? tenantId : "default";
        // → 테넌트 ID별 다른 DB 커넥션 풀 사용
    }
}
```

---

## 💻 실전 구성

### 전체 Baggage + MDC + 로그 연동

```yaml
# application.yml (Spring Boot 3.x)
management:
  tracing:
    sampling:
      probability: 1.0
    baggage:
      remote-fields:
        - user-id
        - tenant-id
      correlation-fields:    # MDC 자동 연동
        - user-id
        - tenant-id

logging:
  pattern:
    level: >
      %5p [${spring.application.name},
          %X{traceId},%X{spanId},
          user=%X{user-id},
          tenant=%X{tenant-id}]
```

```bash
# 로그 출력 예시:
#  INFO [order-service,abc123,def456,user=user-789,tenant=corp-a] OrderService - Order created: 12345
#  INFO [inventory-service,abc123,789ghi,user=user-789,tenant=corp-a] InventoryService - Stock checked: 10

# user-id와 tenant-id가 모든 서비스 로그에 자동으로 포함됨!
# → ELK에서 user-id 검색 → 특정 사용자의 모든 요청 추적 가능
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 특정 사용자 요청 추적

```
문제: user-789가 "결제가 안 된다"고 신고

Baggage 없는 경우:
  Zipkin: traceId가 뭔지 모름
  로그: 수천 개 로그 중 user-789 검색 → 어렵거나 불가

Baggage 있는 경우:
  1. ELK에서 검색: user-id: user-789
  2. 해당 사용자의 모든 로그 조회
  3. 오류 로그에서 traceId 확인: traceId=abc123
  4. Zipkin에서 abc123 검색 → 전체 호출 체인 시각화

  결과:
  ELK 로그: [Payment Service] PaymentException: Card declined
  Zipkin: Payment Service 1200ms → External API 1150ms

  원인: 카드사 API가 1.1초 걸려서 타임아웃 → 결제 실패
  해결: 카드사 API 타임아웃 설정 조정
```

---

## ⚖️ 트레이드오프

| 방식 | 자동화 | 크기 제한 | 보안 |
|------|--------|---------|------|
| **Baggage (remote)** | 완전 자동 전파 | 소량만 | 헤더 노출 주의 |
| **Baggage (local)** | 프로세스 내 자동 | 메모리 제한 | 외부 노출 없음 |
| **HTTP 헤더 수동** | 서비스마다 구현 | 제한 없음 | 제어 가능 |
| **파라미터 전달** | 없음 | 제한 없음 | 가장 안전 |

```
Baggage 사용 시 주의사항:

보안:
  remote 필드는 HTTP 헤더로 전파 → 외부에서 볼 수 있음
  민감 정보(비밀번호, 개인정보) → Baggage에 넣지 않기
  userId는 괜찮지만 사용자 이름, 이메일은 주의

크기:
  헤더 크기 제한 (Nginx 기본 8KB)
  Baggage 값이 크면 헤더 제한 초과 → 요청 거부
  값은 짧은 ID (UUID, 숫자) 정도로 제한

필드 수:
  remote 필드 수 = 모든 서비스 간 HTTP 요청에 추가되는 헤더 수
  2~5개가 적절, 10개 이상은 과도
```

---

## 📌 핵심 정리

```
Baggage Propagation 핵심:

목적:
  비즈니스 컨텍스트(userId, tenantId)를
  Trace Context와 함께 모든 서비스에 자동 전파
  코드 변경 없이 로그에 자동 포함

remote vs local:
  remote: HTTP 헤더로 서비스 간 전파 (X-B3-Bag-{name})
  local: 같은 프로세스 내에서만 유효

MDC 연동 (correlation-fields):
  Baggage 값 → MDC 자동 복사
  로그 패턴에 %X{user-id} → 자동 출력

BaggageField API:
  create("name"): 필드 정의
  getValue(): 현재 컨텍스트에서 읽기
  updateValue("value"): 현재 컨텍스트에 쓰기

활용 패턴:
  Gateway: updateValue(userId) 최초 설정
  하위 서비스: getValue() 자동 읽기
  로그: MDC 통해 자동 포함
  DB 라우팅: 테넌트 ID 기반 라우팅

주의:
  민감 정보 제외
  필드 수 최소화 (2~5개)
  값 크기 소량 유지
```

---

## 🤔 생각해볼 문제

**Q1.** 외부 클라이언트가 `X-B3-Bag-user-id: hacked-user` 헤더를 임의로 설정해서 보낸다면 어떤 보안 위협이 있는가?

<details>
<summary>해설 보기</summary>

외부 클라이언트가 Baggage 헤더를 위조하면 시스템 내부에서 잘못된 userId로 동작할 수 있습니다. 예를 들어 TenantDataSourceRouter가 위조된 tenantId를 읽어 다른 테넌트의 DB에 접근할 수 있습니다. 또한 로그에 위조된 userId가 기록되어 감사 로그가 오염됩니다.

방어 방법:
1. **Gateway에서 Baggage 헤더 제거 후 재설정**: 들어오는 모든 외부 요청에서 Baggage 헤더를 제거하고 JWT 검증 후 신뢰할 수 있는 값으로 재설정합니다.
2. **서비스 메시(Service Mesh)**: Istio 등에서 외부 → 내부 트래픽 경계에서 Baggage 헤더 필터링.
3. **HMAC 서명**: Baggage 값에 서버 비밀키로 서명, 수신 시 검증.

일반적으로 Baggage는 신뢰할 수 있는 내부 서비스 간 통신에만 사용하고, 외부 입력값은 반드시 검증 후 재설정해야 합니다.

</details>

---

**Q2.** `BaggageField.create("user-id")`를 여러 곳에서 각각 호출해 다른 BaggageField 인스턴스를 만들어도 같은 Baggage 값에 접근하는가?

<details>
<summary>해설 보기</summary>

같은 이름(`"user-id"`)을 가진 BaggageField 인스턴스는 실제로 동일한 Baggage 데이터에 접근합니다. Brave의 `ExtraBaggageFields`는 BaggageField 인스턴스 참조가 아닌 **이름(String)** 을 기준으로 데이터를 저장하기 때문입니다. 따라서:

```java
// 서로 다른 클래스에서
BaggageField field1 = BaggageField.create("user-id");
BaggageField field2 = BaggageField.create("user-id");

field1.updateValue("user-123");
field2.getValue();  // → "user-123" 동일한 값 반환
```

그러나 실용적으로는 `static final` 상수로 정의해 재사용하는 것이 성능상 좋습니다. 매번 새 인스턴스를 생성해도 동작하지만 불필요한 객체 생성입니다.

</details>

---

**Q3.** `correlation-fields`에 등록된 Baggage 필드가 MDC에 자동으로 들어가는데, `updateValue()`를 호출한 시점과 MDC가 갱신되는 시점 사이에 지연이 있는가?

<details>
<summary>해설 보기</summary>

`updateValue()`를 호출하면 TraceContext의 `ExtraBaggageFields` 데이터가 즉시 업데이트됩니다. 그러나 MDC는 `CurrentTraceContext.Scope` 기준으로 관리됩니다. `CorrelationScopeDecorator.decorateScope()`는 Scope 진입 시점에 Baggage 값을 MDC에 복사합니다.

따라서:
- **Scope 내에서 updateValue() 호출**: MDC는 즉시 갱신되지 않습니다. 현재 Scope에서는 이전 MDC 값을 사용합니다.
- **새 Span(새 Scope) 시작 후**: decorateScope()가 다시 호출되어 최신 Baggage 값이 MDC에 반영됩니다.

실용적 의미: `updateValue()`를 호출한 직후의 같은 Span 내 로그에서는 MDC가 구 값을 가질 수 있습니다. 로그에 즉시 반영이 필요하면 `MDC.put("user-id", value)`를 직접 호출해야 합니다. 그러나 대부분의 경우 Scope 시작 시점(요청 수신, 새 Span 시작)에 값을 설정하므로 이 지연은 실용적으로 문제가 되지 않습니다.

</details>

---

<div align="center">

**[⬅️ 이전: MDC를 통한 로그 추적](./03-mdc-log-tracing.md)** | **[홈으로 🏠](../README.md)** | **[다음: Custom Span Tags ➡️](./05-custom-span-tags.md)**

</div>
