# Custom Span Tags — 비즈니스 데이터를 Span에 추가해 Zipkin 검색 강화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Tracer.currentSpan().tag(key, value)` API가 Span에 태그를 추가하는 내부 동작은?
- `@NewSpan`과 `@ContinueSpan`은 각각 어떤 상황에 사용하며 AOP 동작 원리는?
- `@SpanTag`로 메서드 파라미터를 자동으로 Span 태그에 추가하는 방법은?
- Span 태그와 Baggage의 근본적인 차이는 무엇인가? (전파 여부)
- Zipkin UI에서 태그 값으로 검색하는 방법과 태그 활용 전략은?
- 태그 값이 너무 클 때 발생하는 문제와 최대 크기 제한은?

---

## 🔍 왜 MSA에서 필요한가

### 기본 Span으로는 부족한 정보

```
자동 생성 Span이 주는 정보:
  GET /orders/{id}:
    http.method: GET
    http.url: /orders/123
    http.status_code: 200
    duration: 850ms

이것만으로는 모르는 것:
  - 주문 ID는 무엇인가?
  - 사용자는 누구인가?
  - DB 쿼리를 몇 번 했는가?
  - 결과가 캐시 히트였는가?
  - 비즈니스 실패 이유는?

Custom Tag 추가 후:
  GET /orders/{id}:
    http.method: GET
    http.url: /orders/123
    http.status_code: 200
    order.id: "123"           ← 추가
    order.status: "PAID"      ← 추가
    order.user-id: "user-789" ← 추가
    db.queries: "3"           ← 추가
    cache.hit: "false"        ← 추가
    duration: 850ms

Zipkin에서:
  - "order.status=PENDING인 요청만 느린가?" 검색 가능
  - "db.queries > 5인 요청 찾기" (성능 문제 추적)
  - "cache.hit=false인 요청 비율" (캐시 효율 분석)
```

---

## 😱 잘못된 구성

### Before: 태그에 민감 정보나 대용량 데이터 포함

```java
// ❌ 태그에 포함하면 안 되는 것들
span.tag("user.password", user.getPassword());    // 민감 정보
span.tag("request.body", requestBody);             // 수KB~수MB → Zipkin 과부하
span.tag("error.stack-trace", e.getStackTrace().toString()); // 수십KB

// 태그는 Zipkin 서버에 저장됨 → 크기 제한 필요
// Brave 기본 태그 값 최대 크기: 128자 (설정 변경 가능하지만 권장 안 함)
```

### Before: 현재 Span이 없을 때 태그 추가 시도

```java
// ❌ Span이 없는 컨텍스트에서 태그 추가
@Scheduled(fixedDelay = 5000)
public void scheduledTask() {
    Span currentSpan = tracer.currentSpan();
    // @Scheduled는 새 Trace를 자동으로 시작하지 않을 수 있음
    // currentSpan이 null이면 NullPointerException!
    currentSpan.tag("task.name", "scheduled");  // ❌ NPE

    // ✅ null 체크 필요
    if (currentSpan != null) {
        currentSpan.tag("task.name", "scheduled");
    }
}
```

---

## ✨ 올바른 패턴

### After: 세 가지 방법으로 태그 추가

```java
// 방법 1: 프로그래매틱 태그 추가
@Service
public class OrderService {

    @Autowired
    private Tracer tracer;

    public Order processOrder(Long orderId, String userId) {
        // 현재 Span에 태그 추가 (자동 생성된 http.server Span)
        Span currentSpan = tracer.currentSpan();
        if (currentSpan != null) {
            currentSpan.tag("order.id", String.valueOf(orderId));
            currentSpan.tag("order.user", userId);
        }

        Order order = orderRepository.findById(orderId);
        currentSpan.tag("order.status", order.getStatus().name());

        return order;
    }
}

// 방법 2: @NewSpan으로 새 Span 생성
// 방법 3: @ContinueSpan + @SpanTag로 선언적 방식
```

---

## 🔬 내부 동작 원리

### 1. Span.tag() — 태그 저장 메커니즘

```java
// MutableSpan.java (Brave)
// Span 완료 전까지 태그를 가변 저장소에 보관

public final class MutableSpan {

    // 태그 저장 (ArrayList로 순서 유지)
    // 짝수 인덱스: 키, 홀수 인덱스: 값
    private ArrayList<String> tags;

    public void tag(String key, String value) {
        // null 체크
        if (key == null) throw new NullPointerException("key == null");
        if (value == null) throw new NullPointerException("value == null");

        // ① 같은 키가 이미 있으면 값 업데이트
        if (tags != null) {
            for (int i = 0, length = tags.size(); i < length; i += 2) {
                if (key.equals(tags.get(i))) {
                    tags.set(i + 1, value);
                    return;
                }
            }
        } else {
            tags = new ArrayList<>();
        }

        // ② 새 키-값 쌍 추가
        tags.add(key);
        tags.add(value);
    }

    // 태그 순회 (ZipkinSpanHandler에서 JSON 변환 시 사용)
    public void forEachTag(TagConsumer consumer) {
        if (tags == null) return;
        for (int i = 0, length = tags.size(); i < length; i += 2) {
            consumer.accept(tags.get(i), tags.get(i + 1));
        }
    }
}
```

### 2. @NewSpan — 새 Span 생성 AOP

```java
// @NewSpan: 메서드 호출 시 새 자식 Span 생성
// 부모 Span(현재 HTTP Span)의 자식으로 생성됨

@Service
public class PaymentService {

    // ① @NewSpan: 이 메서드 실행이 별도 Span으로 추적됨
    @NewSpan("payment-processing")
    public PaymentResult processPayment(PaymentRequest request) {
        // 이 메서드 내의 모든 log, 내부 호출이 "payment-processing" Span 내에서 실행
        log.info("Processing payment: {}", request.getOrderId());
        return externalPaymentApi.process(request);
    }
}

// AOP 구현 (SpanAspect.java):
@Around("@annotation(newSpan)")
public Object newSpanAspect(ProceedingJoinPoint pjp, NewSpan newSpan) throws Throwable {
    // ② 새 Span 시작 (현재 Span의 자식)
    String spanName = newSpan.name().isEmpty()
        ? pjp.getSignature().getName()          // 메서드 이름 사용
        : newSpan.name();                        // 어노테이션 이름 사용

    Span span = tracer.nextSpan()
        .name(spanName)
        .start();

    // ③ 새 Span을 현재 스코프로 설정
    try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
        // ④ 실제 메서드 실행
        Object result = pjp.proceed();
        return result;
    } catch (Throwable throwable) {
        // ⑤ 예외 기록
        span.error(throwable);
        throw throwable;
    } finally {
        // ⑥ Span 종료 (Zipkin 전송)
        span.finish();
    }
}

// 결과: Zipkin에서
// HTTP GET /api/payment (부모)
//   └── payment-processing (자식) ← @NewSpan으로 생성된 것
//         └── HTTP POST external-api (손자)
```

### 3. @ContinueSpan + @SpanTag — 선언적 태그 추가

```java
// @ContinueSpan: 새 Span 생성하지 않고 현재 Span에 태그만 추가

@Service
public class InventoryService {

    // @ContinueSpan: 현재 활성 Span을 계속 사용
    // @SpanTag: 파라미터 값을 자동으로 태그로 추가
    @ContinueSpan(log = "inventory-check")
    public Inventory checkInventory(
            @SpanTag("product.id") Long productId,
            @SpanTag("warehouse") String warehouse) {

        // 현재 HTTP Span에 자동으로 태그 추가:
        // product.id = "123"
        // warehouse = "seoul-dc-1"

        return inventoryRepository.find(productId, warehouse);
    }
}

// @SpanTag 내부 동작 (SpanAspect):
@Around("@annotation(continueSpan)")
public Object continueSpanAspect(ProceedingJoinPoint pjp,
        ContinueSpan continueSpan) throws Throwable {

    // 현재 Span 가져오기 (새로 생성 안 함)
    Span span = tracer.currentSpan();

    // 파라미터의 @SpanTag 어노테이션 처리
    MethodSignature signature = (MethodSignature) pjp.getSignature();
    Method method = signature.getMethod();
    Object[] args = pjp.getArgs();

    Annotation[][] paramAnnotations = method.getParameterAnnotations();
    for (int i = 0; i < paramAnnotations.length; i++) {
        for (Annotation annotation : paramAnnotations[i]) {
            if (annotation instanceof SpanTag spanTag) {
                String key = spanTag.value().isEmpty()
                    ? method.getParameters()[i].getName()  // 파라미터 이름
                    : spanTag.value();                     // 어노테이션 값

                // 태그 추가 (값을 String으로 변환)
                if (args[i] != null && span != null) {
                    span.tag(key, String.valueOf(args[i]));
                }
            }
        }
    }

    return pjp.proceed();
}
```

### 4. 커스텀 오류 태그 추가

```java
// 비즈니스 오류를 Zipkin에서 검색 가능하게 태그로 기록

@Service
public class OrderService {

    @NewSpan("order-creation")
    public Order createOrder(OrderRequest request) {
        Span span = tracer.currentSpan();

        try {
            // 재고 확인
            if (!inventoryService.hasStock(request.getProductId())) {
                // ① 비즈니스 오류 태그
                span.tag("error.type", "out-of-stock");
                span.tag("error.product-id", String.valueOf(request.getProductId()));
                throw new OutOfStockException(request.getProductId());
            }

            Order order = orderRepository.save(request.toOrder());
            // ② 성공 태그
            span.tag("order.id", String.valueOf(order.getId()));
            span.tag("order.amount", String.valueOf(order.getAmount()));

            return order;

        } catch (PaymentException e) {
            // ③ 시스템 오류: span.error() 사용
            span.error(e);  // Zipkin에 "error" 태그로 기록
            span.tag("error.type", "payment-failed");
            span.tag("error.code", e.getErrorCode());
            throw e;
        }
    }
}
```

### 5. Zipkin 태그 검색 활용

```
Zipkin UI에서 태그 기반 고급 검색:

Find Traces:
  - serviceName: payment-service
  - annotationQuery: order.status=FAILED      ← 실패한 주문만
  - minDuration: 1000ms                       ← 1초 이상 느린 것
  - limit: 10

태그 검색 쿼리:
  order.status=FAILED          ← 정확히 일치
  http.status_code=500         ← HTTP 오류 필터
  db.queries="5"               ← DB 쿼리 5번인 것
  cache.hit=false              ← 캐시 미스

Zipkin API (프로그래매틱 검색):
  GET /api/v2/traces?
    serviceName=order-service&
    annotationQuery=order.status=FAILED&
    endTs=1714000000000&
    lookback=3600000&   ← 1시간
    limit=10
```

---

## 💻 실전 구성

### 비즈니스 메트릭 Span 태그 전략

```java
// 모든 핵심 비즈니스 이벤트에 태그 추가하는 AOP
@Aspect
@Component
public class BusinessSpanTagAspect {

    @Autowired
    private Tracer tracer;

    // ① 주문 생성 태그
    @AfterReturning(pointcut = "execution(* *.OrderService.createOrder(..))",
                    returning = "order")
    public void tagOrderCreation(Order order) {
        Span span = tracer.currentSpan();
        if (span != null && order != null) {
            span.tag("order.id", String.valueOf(order.getId()));
            span.tag("order.total", String.valueOf(order.getTotal()));
            span.tag("order.item-count", String.valueOf(order.getItems().size()));
        }
    }

    // ② DB 쿼리 횟수 태그 (Hibernate Statistics 연동)
    @Around("execution(* *.Repository.*(..))")
    public Object tagDbMetrics(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        long elapsed = System.currentTimeMillis() - start;

        Span span = tracer.currentSpan();
        if (span != null) {
            span.tag("db.repository", pjp.getSignature().getDeclaringTypeName());
            span.tag("db.method", pjp.getSignature().getName());
            span.tag("db.elapsed-ms", String.valueOf(elapsed));
        }
        return result;
    }
}
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 느린 요청 원인 분석

```bash
# 문제: 일부 주문 조회가 3초 이상 걸림

# 1. Zipkin에서 느린 Trace 검색
GET /api/v2/traces?serviceName=order-service&minDuration=3000000&limit=5

# 2. 느린 Trace 분석
# traceId: abc123
# Span 목록:
#   GET /orders/123: 3200ms
#     ├── DB findById: 3100ms  ← 이게 느린 것
#     │   [tag] db.statement: "SELECT * FROM orders WHERE..."
#     │   [tag] db.rows: "15000"  ← 풀스캔!

# 3. 또 다른 패턴 발견
# 특정 상품 ID에서만 느림
GET /api/v2/traces?annotationQuery=product.id=789&minDuration=1000000

# 4. product.id=789인 주문만 3초 → 해당 상품 관련 쿼리 분석
# → product_id에 인덱스 없음 → 추가 후 해결
```

---

## ⚖️ 트레이드오프

| 방식 | 자동화 | 가독성 | 적합한 케이스 |
|------|--------|--------|--------------|
| **수동 tag()** | 없음 | 명시적 | 특수 조건, 동적 값 |
| **@NewSpan** | AOP 자동 | 선언적 | 메서드 단위 Span |
| **@ContinueSpan + @SpanTag** | AOP 자동 | 선언적 | 파라미터 태그 |
| **AOP Aspect** | 자동 | 집중화 | 공통 패턴 |

```
태그 vs Baggage:
  태그:
    현재 Span에만 저장
    Zipkin에 전송됨 (검색 가능)
    다른 서비스로 전파 안 됨
  
  Baggage:
    Trace Context와 함께 전파됨
    Zipkin에 저장 안 됨 (검색 불가)
    모든 서비스에서 읽기 가능

  userId를 Zipkin에서 검색하려면: 태그
  userId를 다른 서비스에 전달하려면: Baggage
  둘 다 필요하면: 태그 + Baggage 동시 사용

태그 값 크기 제한:
  Brave 기본: 128자 (설정 변경 가능)
  큰 값은 잘라서 저장하거나 해시로 대체
  JSON이나 XML을 태그로 넣는 것은 피할 것
```

---

## 📌 핵심 정리

```
Custom Span Tags 핵심:

태그 추가 방법:
  1. span.tag(key, value): 직접 추가 (null 체크 필요)
  2. @NewSpan("name"): 새 자식 Span 생성 + AOP
  3. @ContinueSpan: 현재 Span 계속 + 태그 추가
  4. @SpanTag: 파라미터 자동 태그화

@NewSpan vs @ContinueSpan:
  @NewSpan: 새 Span 생성 → Zipkin에 별도 타임라인
  @ContinueSpan: 현재 Span 계속 → 추가 Span 없음, 태그만 추가

Span 태그의 특성:
  현재 Span에만 저장 (다른 서비스 전파 없음)
  Span 완료 시 Zipkin으로 전송
  Zipkin UI에서 태그 값으로 검색 가능

vs Baggage:
  태그: Zipkin 검색 가능, 다른 서비스 전파 불가
  Baggage: 다른 서비스 전파, Zipkin 검색 불가

실무 태그 전략:
  비즈니스 ID: order.id, user.id, product.id
  상태: order.status, payment.result
  성능: db.queries, cache.hit, external-api.retry-count
  오류: error.type, error.code, error.message
```

---

## 🤔 생각해볼 문제

**Q1.** `@NewSpan`과 `@Transactional`이 같이 붙은 메서드에서 트랜잭션과 Span의 시작/종료 순서는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Spring AOP Advice의 Order에 따라 결정됩니다. `@Transactional`의 기본 Order는 `Integer.MAX_VALUE - 1`이고, `@NewSpan`의 SpanAspect Order는 보통 더 낮은 값(예: `Integer.MAX_VALUE - 3`)입니다. 낮은 Order가 바깥(먼저 시작, 나중에 종료)입니다.

일반적으로:
1. SpanAspect: 새 Span 시작 (외부 레이어)
2. TransactionInterceptor: 트랜잭션 시작 (내부 레이어)
3. 메서드 실행
4. TransactionInterceptor: 트랜잭션 커밋/롤백
5. SpanAspect: Span 종료 (Zipkin 전송)

결과: Span duration에 트랜잭션 시간이 포함됩니다. 트랜잭션 커밋/롤백 시간도 Span에 포함됩니다. Span 태그에 트랜잭션 상태를 `span.tag("db.transaction", "committed")`로 추가하면 유용합니다.

</details>

---

**Q2.** Span이 완료(finish())된 후에도 `tag()`를 호출할 수 있는가? 어떤 일이 발생하는가?

<details>
<summary>해설 보기</summary>

기술적으로 호출 가능하지만 효과가 없거나 무시됩니다. Brave의 `RealSpan.tag()`는 내부적으로 `MutableSpan.tag()`를 호출하는데, Span이 이미 완료됐다면 `MutableSpan`이 이미 `ZipkinSpanHandler`에 의해 처리됐습니다. finish() 이후에 추가된 태그는 Zipkin으로 이미 전송된 데이터에 포함되지 않습니다.

실제로 Brave는 `finish()` 이후 태그 추가를 조용히 무시합니다(예외 없음). 이를 방지하려면:
1. Span 생명주기 내에서만 `tag()`를 호출
2. `try-with-resources`(SpanInScope) 패턴 사용
3. `@NewSpan` / `@ContinueSpan` AOP 사용 (자동으로 생명주기 관리)

</details>

---

**Q3.** `tracer.currentSpan()`이 null을 반환하는 상황은 어떤 케이스인가? 어떻게 안전하게 처리하는가?

<details>
<summary>해설 보기</summary>

`currentSpan()`이 null을 반환하는 케이스:

1. **@Scheduled 메서드**: Spring은 스케줄 작업 실행 시 자동으로 Trace Context를 시작하지 않습니다.
2. **직접 생성한 스레드**: `new Thread(() -> ...)` 내부에서 TraceContext가 전파되지 않을 때.
3. **비동기 코드**: MDC/Trace 전파 없이 실행되는 @Async 메서드.
4. **테스트 환경**: Tracing 설정 없이 단위 테스트 실행 시.
5. **`NEVER_SAMPLE`**: 샘플링되지 않는 요청의 `NoopSpan`이 반환될 수도 있지만 null은 아님.

안전한 처리 방법:
```java
// 방법 1: null 체크
Span span = tracer.currentSpan();
if (span != null) {
    span.tag("key", "value");
}

// 방법 2: NoopSpan 활용 (null safe)
// Brave: currentSpan()이 null이면 Tracer.nextSpan()으로 noop span 생성
Span span = tracer.currentSpan();
if (span == null) {
    span = tracer.nextSpan().start();
    // 이 경우 새 루트 Span이 생성됨 (의도에 맞게 사용)
}

// 방법 3: Optional 패턴
Optional.ofNullable(tracer.currentSpan())
    .ifPresent(s -> s.tag("key", "value"));
```

</details>

---

<div align="center">

**[⬅️ 이전: Baggage Propagation](./04-baggage-propagation.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 7 — Advanced Cloud Patterns ➡️](../advanced-cloud-patterns/01-event-driven-architecture.md)**

</div>
