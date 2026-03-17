# Sleuth와 Trace ID / Span ID — 분산 추적의 기본 구조와 컨텍스트 전파 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `TraceContext`의 traceId · spanId · parentSpanId가 각각 의미하는 바는 무엇인가?
- `Propagator`가 HTTP 헤더(`X-B3-TraceId` 등)에 컨텍스트를 주입(inject)·추출(extract)하는 정확한 코드 경로는?
- Spring Cloud Sleuth 3.x → Micrometer Tracing으로 전환된 배경과 핵심 차이는?
- B3 전파 형식과 W3C TraceContext 형식의 차이는 무엇이며 어떻게 선택하는가?
- `ThreadLocal`에 Trace Context가 저장되는 방식과 비동기 처리 시 컨텍스트 손실 문제는?
- Span의 생명주기(시작 → 태그 추가 → 종료 → 전송)는 어떻게 구성되는가?

---

## 🔍 왜 MSA에서 필요한가

### 분산 추적 없이 장애 원인 분석이 어려운 이유

```
MSA 호출 체인:
  Client → Gateway → Order Service → Inventory Service → DB
                                   → Payment Service → External API

장애 시나리오:
  Client: "결제가 안 됩니다" (5초 후 타임아웃)

분산 추적 없을 때:
  Gateway 로그: [ERROR] Upstream timeout after 5s
  Order 로그: [WARN] Payment call took 4.8s
  Payment 로그: [INFO] Calling external API...
  Payment 로그: [ERROR] External API timeout

  문제:
  어떤 요청인지 알 수 없음 (요청 ID 없음)
  어느 서비스에서 얼마나 걸렸는지 시각화 불가
  순서 재구성 불가 (분산 로그)

분산 추적 있을 때:
  모든 로그에 동일한 traceId: "abc123"
  Zipkin UI에서 "abc123" 검색:
  → Gateway: 5000ms (전체)
    → Order: 4900ms
      → Inventory: 100ms ✅
      → Payment: 4800ms ← 여기가 범인!
        → External API: 4800ms ← 실제 원인!

  장애 원인: External API 응답 4.8초 → Payment 타임아웃 → 전체 장애
```

---

## 😱 잘못된 구성

### Before: Spring Cloud Sleuth 2.x 방식 (Boot 3.x 비호환)

```xml
<!-- ❌ Spring Boot 3.x에서 더 이상 지원 안 됨 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>
```

```
Spring Boot 3.x / Spring Cloud 2022.x 변경:
  Sleuth → Micrometer Tracing으로 통합
  기존 Sleuth API (Tracer, Span 등) → Micrometer Tracing API
  이유:
  1. Spring 생태계 표준화 (Micrometer가 메트릭 + 트레이싱 통합)
  2. Sleuth는 Spring Cloud 전용 → Micrometer는 범용
  3. OpenTelemetry 등 다양한 벤더 지원 용이
```

---

## ✨ 올바른 패턴

### After: Spring Boot 3.x Micrometer Tracing 설정

```xml
<!-- pom.xml (Spring Boot 3.x) -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
    <!-- Brave: Zipkin 호환 구현체 -->
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>

<!-- 또는 OpenTelemetry 구현체 선택 -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0   # 100% 샘플링 (개발)
                         # 운영: 0.1 (10%) 또는 0.01 (1%)
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

---

## 🔬 내부 동작 원리

### 1. TraceContext 구조 — 분산 추적의 기본 단위

```java
// TraceContext (Brave 구현체)
// 하나의 논리적 작업 단위(Span)를 식별하는 컨텍스트

public final class TraceContext {

    // traceId: 전체 요청 체인을 식별하는 고유 ID
    // 모든 서비스에서 동일한 값 유지
    // 128비트 (16진수 32자리) 또는 64비트 (16진수 16자리)
    final long traceIdHigh;  // 상위 64비트
    final long traceId;      // 하위 64비트

    // spanId: 현재 서비스에서의 이 작업 단위 ID
    // 각 서비스 진입마다 새로 생성
    final long spanId;

    // parentSpanId: 부모 Span의 ID
    // 루트 Span이면 null (첫 번째 서비스)
    final Long parentId;

    // sampled: 이 Trace를 Zipkin으로 보낼 것인가?
    final Boolean sampled;

    // debug: 무조건 샘플링 (sampler.probability 무시)
    final boolean debug;
}

// 예시: 3개 서비스 호출 체인
//
// Gateway:
//   traceId = "abc123"
//   spanId  = "111"
//   parentId = null (루트)
//
// Order Service:
//   traceId = "abc123"   ← 동일!
//   spanId  = "222"      ← 새로 생성
//   parentId = "111"     ← Gateway의 spanId
//
// Payment Service:
//   traceId = "abc123"   ← 동일!
//   spanId  = "333"      ← 새로 생성
//   parentId = "222"     ← Order의 spanId
```

### 2. Span 생명주기

```java
// Span: 하나의 작업 단위
// 시작 시각, 종료 시각, 태그, 이벤트 포함

// Brave의 Span 인터페이스
public interface Span {
    // Span 시작 (현재 시각 기록)
    Span start();
    Span start(long timestamp);  // 수동 시작 시각

    // 태그 추가 (key-value)
    Span tag(String key, String value);

    // 이벤트 기록 (시각이 있는 로그 엔트리)
    Span annotate(String value);
    Span annotate(long timestamp, String value);

    // 에러 기록
    Span error(Throwable throwable);

    // Span 종료 (Zipkin으로 전송 트리거)
    void finish();
    void finish(long timestamp);
    void abandon();  // 취소 (Zipkin으로 전송 안 함)

    // 현재 Span이 속한 TraceContext
    TraceContext context();

    // 이 Span의 컨텍스트를 현재 스코프로 활성화
    Tracer.SpanInScope makeCurrent();
}

// 실제 Span 사용 패턴
Tracer tracer = ...;
Span span = tracer.nextSpan()
    .name("inventory-check")
    .start();

try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
    // 이 블록 내의 모든 코드: span이 현재 Span
    Inventory inv = inventoryService.check(productId);
    span.tag("product.id", String.valueOf(productId));
    span.tag("inventory.count", String.valueOf(inv.getCount()));
    return inv;
} catch (Exception e) {
    span.error(e);
    throw e;
} finally {
    span.finish();  // Zipkin으로 전송
}
```

### 3. Propagator — HTTP 헤더로 컨텍스트 전파

```java
// Propagator: 서비스 간 TraceContext 전파 담당
// 두 가지 방향:
//   inject: 발신 요청 헤더에 컨텍스트 삽입
//   extract: 수신 요청 헤더에서 컨텍스트 추출

// B3 형식 (Zipkin 표준):
// X-B3-TraceId: abc123...       (traceId)
// X-B3-SpanId: def456...        (spanId)
// X-B3-ParentSpanId: ghi789...  (parentSpanId)
// X-B3-Sampled: 1               (sampled)

// inject: 발신 HTTP 요청에 헤더 추가
// (RestTemplate, WebClient 인터셉터에서 자동 호출)
B3Propagation propagation = B3Propagation.newFactoryBuilder()
    .injectFormat(B3Propagation.Format.MULTI)  // 여러 헤더 (B3)
    .build()
    .create(...)

propagation.injector((carrier, key, value) ->
    carrier.header(key, value)  // HTTP 헤더에 추가
).inject(traceContext, httpRequest);

// extract: 수신 HTTP 요청에서 헤더 추출
TraceContextOrSamplingFlags result =
    propagation.extractor((carrier, key) ->
        carrier.header(key)  // HTTP 헤더에서 읽기
    ).extract(httpRequest);

// extract 결과로 서버 Span 생성
Span serverSpan = tracer.nextSpan(result)
    .kind(Span.Kind.SERVER)
    .name(httpRequest.getMethod() + " " + httpRequest.getPath())
    .start();
```

### 4. B3 vs W3C TraceContext 전파 형식

```
B3 형식 (Brave/Zipkin 기본):
  X-B3-TraceId: 463ac35c9f6413ad48485a3953bb6124
  X-B3-SpanId: a2fb4a1d1a96d312
  X-B3-ParentSpanId: 0020000000000001
  X-B3-Sampled: 1

B3 Single Header:
  b3: 463ac35c9f6413ad48485a3953bb6124-a2fb4a1d1a96d312-1-0020000000000001

W3C TraceContext (IETF 표준, OpenTelemetry 기본):
  traceparent: 00-463ac35c9f6413ad48485a3953bb6124-a2fb4a1d1a96d312-01
  tracestate: rojo=00f067aa0ba902b7  (벤더별 추가 데이터)

선택 기준:
  Zipkin/Sleuth 기존 환경: B3
  OpenTelemetry/Jaeger 신규 환경: W3C
  혼재 환경: 두 형식 모두 지원 설정 가능

Spring Boot 설정:
  management.tracing.propagation.type: B3  (기본)
  또는
  management.tracing.propagation.type: W3C
```

### 5. ThreadLocal 컨텍스트 저장과 전파

```java
// CurrentTraceContext: 현재 스레드의 Span 관리
// ThreadLocal 기반 → 같은 스레드 내에서 자동 접근 가능

// 일반 동기 코드: 자동 전파
@RestController
public class OrderController {
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable Long id) {
        // Spring MVC가 자동으로 TraceContext를 ThreadLocal에 설정
        // 이 메서드 내 모든 코드가 같은 Span 내에서 실행
        return orderService.findById(id);  // 자동 추적
    }
}

// 비동기 코드: 컨텍스트 명시적 전파 필요
@Service
public class AsyncService {

    // ❌ ThreadLocal 컨텍스트 손실
    public CompletableFuture<Void> processAsync(Long orderId) {
        return CompletableFuture.runAsync(() -> {
            // 새 스레드 → ThreadLocal 없음 → Span 없음
            // 이 코드의 로그: traceId 없음!
            process(orderId);
        });
    }

    // ✅ 컨텍스트 명시적 전달
    @Autowired
    private CurrentTraceContext currentTraceContext;

    public CompletableFuture<Void> processAsyncCorrect(Long orderId) {
        // 현재 스레드의 컨텍스트 캡처
        TraceContext capturedContext = currentTraceContext.get();

        return CompletableFuture.runAsync(
            // Executor에 컨텍스트 래핑
            currentTraceContext.wrap(() -> process(orderId))
            // 또는: Executors에 전파용 래퍼 사용
        );
    }
}
```

### 6. Micrometer Tracing 자동 계측 — 어디서 Span이 생성되는가

```java
// Spring Boot가 자동으로 Span을 생성하는 지점들:

// ① HTTP 요청 수신 (ServerHttpObservationFilter)
// → SERVER Span: "http.server.requests"
// → 태그: http.method, http.url, http.status_code

// ② HTTP 요청 발신 (ClientHttpObservationFilter)
// → CLIENT Span: "http.client.requests"
// → RestTemplate, WebClient에 자동 적용

// ③ @Scheduled 메서드 (ScheduledTaskObservation)
// → 새로운 Trace 시작

// ④ @Async 메서드 (ThreadPoolTaskExecutor)
// → 부모 Span 전파 (자동 설정 시)

// Auto-configuration:
// ObservationAutoConfiguration → Observation.createNotStarted("http.server.requests")
// ObservationRegistry → Span 생성 결정
```

---

## 💻 실전 구성

### 의존성 및 설정 (Spring Boot 3.x)

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0    # 개발: 100%, 운영: 0.05~0.1
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
      connect-timeout: 1s
      read-timeout: 10s

logging:
  pattern:
    # traceId, spanId 로그에 포함
    level: "%5p [${spring.application.name},%X{traceId},%X{spanId}]"
```

```yaml
# docker-compose.yml
services:
  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"
    environment:
      - STORAGE_TYPE=mem    # 개발용 인메모리 저장
      # 운영: STORAGE_TYPE=elasticsearch
```

### Trace 흐름 직접 확인

```bash
# 1. API 호출
curl -v http://localhost:8080/api/orders/1

# 2. 응답 헤더에서 traceId 확인
# < HTTP/1.1 200 OK
# < X-B3-TraceId: 463ac35c9f6413ad48485a3953bb6124

# 3. Zipkin UI에서 traceId 검색
# http://localhost:9411
# → Find Traces → Search by traceId: 463ac35c9f6413ad48485a3953bb6124
# → 전체 호출 체인 타임라인 시각화
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 서비스 간 헤더 전파 검증

```bash
# 서비스 A → 서비스 B 호출 시 헤더 전파 확인

# 서비스 B에 요청 로깅 필터 추가
@Component
public class TraceHeaderLogFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ...) {
        HttpServletRequest httpReq = (HttpServletRequest) req;
        log.info("Received trace headers: TraceId={}, SpanId={}, ParentSpanId={}",
            httpReq.getHeader("X-B3-TraceId"),
            httpReq.getHeader("X-B3-SpanId"),
            httpReq.getHeader("X-B3-ParentSpanId"));
        chain.doFilter(req, res);
    }
}

# 로그 확인:
# [service-a] Sending request to service-b
#   traceId=abc123, spanId=111
#
# [service-b] Received trace headers:
#   TraceId=abc123,   ← service-a와 동일!
#   SpanId=222,       ← 새로 생성
#   ParentSpanId=111  ← service-a의 spanId
```

---

## ⚖️ 트레이드오프

| 방식 | 자동화 | 오버헤드 | 세밀도 |
|------|--------|---------|--------|
| **자동 계측만** | 높음 | 낮음 | HTTP 레벨만 |
| **자동 + @NewSpan** | 중간 | 중간 | 메서드 레벨 |
| **완전 수동** | 낮음 | 높음 | 세밀한 제어 |

```
샘플링 확률 선택:
  개발: 1.0 (100%) — 모든 요청 추적
  스테이징: 0.1 (10%)
  프로덕션: 0.01~0.05 (1~5%)

  이유: 고트래픽에서 100% 샘플링 = Zipkin 서버 부하 폭증
        Span 생성/전송 오버헤드 ∝ 트래픽 × 샘플링 확률

  알람 기반 샘플링:
  오류 발생 요청은 100% 샘플링
  정상 요청은 1% 샘플링
  → 장애 분석 데이터 보장 + 일반 오버헤드 최소
```

---

## 📌 핵심 정리

```
Trace / Span 구조 핵심:

TraceContext:
  traceId: 전체 요청 체인 식별 (모든 서비스 동일)
  spanId: 이 서비스에서의 작업 단위 (각 서비스마다 새로 생성)
  parentSpanId: 부모 서비스의 spanId (계층 관계 표현)
  sampled: Zipkin 전송 여부

Propagator (B3 형식):
  inject: 발신 요청 → X-B3-TraceId, X-B3-SpanId 헤더 추가
  extract: 수신 요청 → 헤더에서 컨텍스트 복원
  → 서비스 간 동일 traceId 유지

Spring Boot 3.x:
  Spring Cloud Sleuth → Micrometer Tracing
  micrometer-tracing-bridge-brave (Zipkin) 또는
  micrometer-tracing-bridge-otel (OpenTelemetry)
  자동 계측: HTTP 요청/응답, RestTemplate, WebClient

ThreadLocal 주의:
  동기 코드: 자동 전파
  비동기(@Async, CompletableFuture): 명시적 전파 필요
  Reactive (Reactor): Reactor Context + 통합 라이브러리 필요
```

---

## 🤔 생각해볼 문제

**Q1.** traceId는 128비트와 64비트 두 가지 옵션이 있다. 언제 128비트를 사용해야 하는가?

<details>
<summary>해설 보기</summary>

128비트(16진수 32자리)가 권장됩니다. UUID 기반 시스템과 OpenTelemetry, W3C TraceContext 모두 128비트를 사용합니다. 64비트는 충돌 가능성이 이론상 더 높으며(실용적으로는 거의 없음), 다른 시스템과의 통합 시 불일치가 발생할 수 있습니다.

Brave(Spring Boot 3.x 기본)에서 128비트 설정:
```java
@Bean
Tracing tracing(ZipkinSpanHandler zipkinSpanHandler) {
    return Tracing.newBuilder()
        .traceId128Bit(true)  // 128비트 traceId 활성화
        .addSpanHandler(zipkinSpanHandler)
        .build();
}
```

128비트가 필수인 경우: OpenTelemetry와 함께 사용, W3C TraceContext 헤더 사용, 여러 분산 추적 시스템 혼용.

</details>

---

**Q2.** `sampled: false`인 요청에서 로그에는 traceId가 찍히는가?

<details>
<summary>해설 보기</summary>

찍힙니다. `sampled: false`는 Zipkin(외부 추적 서버)으로 데이터를 전송하지 않는다는 의미이지, Span 자체를 생성하지 않는다는 것이 아닙니다. `TraceContext`는 여전히 생성되고 ThreadLocal에 저장되며, MDC에도 `traceId`와 `spanId`가 주입됩니다. 따라서 로그에는 traceId가 정상적으로 나타납니다.

샘플링은 Zipkin 전송 여부만 결정합니다. 로그 기반 추적은 샘플링 여부와 무관하게 항상 동작합니다. 이를 통해 운영에서 낮은 샘플링 확률(1%)로도 로그 추적은 100% 가능하고, Zipkin 부하는 최소화할 수 있습니다.

</details>

---

**Q3.** 외부 클라이언트(앱, 브라우저)가 `X-B3-TraceId` 헤더 없이 요청하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Propagator의 `extract()`가 헤더에서 컨텍스트를 찾지 못하면 새 TraceContext가 생성됩니다. Gateway 또는 첫 번째 서비스가 새 `traceId`와 `spanId`를 생성하고 이것이 루트 Span이 됩니다. 이후 내부 서비스 호출 시 이 traceId가 전파됩니다.

반대로 외부 클라이언트가 `X-B3-TraceId`를 직접 전달하면(특정 테스트 도구나 디버그 목적) Gateway는 이 값을 그대로 사용합니다. 이는 보안 관점에서 주의가 필요합니다. 악의적 클라이언트가 의도적으로 traceId를 조작해 다른 요청의 Trace에 자신의 Span을 끼워 넣을 수 있습니다. 프로덕션에서는 외부 헤더를 신뢰하지 않고 항상 새 traceId를 생성하거나, 신뢰 가능한 내부 헤더만 전파하도록 설정해야 합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Zipkin Server 연동 ➡️](./02-zipkin-integration.md)**

</div>
