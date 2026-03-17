# Zipkin Server 연동 — Span 수집부터 UI 시각화까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ZipkinSpanHandler`(Brave)가 Span 완료 시점에 수집해 Zipkin HTTP API로 전송하는 정확한 코드 경로는?
- 샘플링 전략(`probability`, `AlwaysSampler`, `NeverSampler`)이 Span 생성과 전송에 미치는 영향은?
- Span 전송이 동기인가 비동기인가? 전송 실패 시 어떻게 처리되는가?
- `spring.sleuth.sampler.probability` → `management.tracing.sampling.probability` 설정 변경 이유는?
- Zipkin에서 `parentId` 관계로 Trace를 트리 형태로 시각화하는 원리는?
- 운영 환경에서 Zipkin의 저장소(Elasticsearch)와 인메모리 저장소의 차이는?

---

## 🔍 왜 MSA에서 필요한가

### Span을 수집하지 않으면 추적이 의미 없는 이유

```
각 서비스가 Span을 생성해도:
  → 로컬 로그에만 남음
  → 각 서비스의 로그가 분산됨
  → "traceId abc123으로 무슨 일이 있었는가?" 질문에 답하려면
     모든 서비스 로그를 수동으로 grep해야 함

Zipkin 도입 후:
  각 서비스 → Span 완료 시 → Zipkin HTTP API 전송
  Zipkin: 모든 Span을 traceId 기준으로 조합
  → 하나의 Trace = 전체 호출 체인 시각화

Zipkin UI가 보여주는 것:
  ① 전체 응답 시간 타임라인 (각 서비스별 소요 시간)
  ② 서비스 간 의존성 그래프
  ③ 느린 Span 식별 (병목 지점)
  ④ 오류 발생 Span (붉은색 표시)
  ⑤ 태그 기반 검색 (http.status_code=500 등)
```

---

## 😱 잘못된 구성

### Before: 동기 전송으로 API 응답 지연

```java
// ❌ 동기 Zipkin 전송 (기본 설정으로 이렇게 하면 안 됨)
// 모든 Span 완료마다 HTTP 요청 → API 응답 지연 발생
// Zipkin 서버 다운 시 → 서비스 응답 지연/실패 가능

// Span 전송은 반드시 비동기로!
// Spring Boot 자동 설정은 기본적으로 비동기 전송
// 그러나 커스텀 설정 시 주의 필요
```

### Before: 모든 요청 100% 샘플링으로 운영

```yaml
# ❌ 운영 환경에서 100% 샘플링
management:
  tracing:
    sampling:
      probability: 1.0
```

```
초당 1000 요청인 서비스에서:
  1000 RPS × 5개 Span/요청 = 5000 Span/초 → Zipkin 전송
  Zipkin 서버 과부하 → Span 손실 또는 서비스 성능 영향
  
  권장: 운영 0.01~0.05 (1~5%)
  고트래픽: 0.001 (0.1%)
  장애 분석: 오류 발생 요청 100% 샘플링 (AlwaysSampler 조건부)
```

---

## ✨ 올바른 패턴

### After: 비동기 전송 + 적절한 샘플링

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 0.1    # 운영: 10%
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
      connect-timeout: 1s
      read-timeout: 10s

# 오류 요청 100% 샘플링 (커스텀 Sampler로 구현)
```

```java
// 조건부 샘플링: 오류 발생 시 100%, 정상 시 5%
@Bean
public Sampler errorAwareSampler() {
    return (traceContext) -> {
        // 오류 플래그가 있는 요청은 항상 샘플링
        // (실제로는 Span 완료 시점이 아니라 시작 시점에 결정됨)
        return SamplerFunction.deferDecision();  // 후속 처리에 위임
    };
}
```

---

## 🔬 내부 동작 원리

### 1. ZipkinSpanHandler — Span 수집 및 전송

```java
// ZipkinSpanHandler.java (Brave + Zipkin Reporter 통합)
// SpanHandler: Span 완료 시 콜백되는 훅

public final class ZipkinSpanHandler extends SpanHandler {

    private final Reporter<zipkin2.Span> spanReporter;

    @Override
    public boolean end(TraceContext context, MutableSpan span, Cause cause) {
        if (!context.sampled()) {
            // sampled=false → Zipkin으로 전송 안 함
            return true;
        }

        // ① Brave MutableSpan → Zipkin Span 변환
        zipkin2.Span zipkinSpan = convert(span, context);

        // ② Reporter를 통해 비동기 전송
        // AsyncReporter: 로컬 버퍼에 쌓고 주기적으로 배치 전송
        spanReporter.report(zipkinSpan);

        return true;
    }

    private zipkin2.Span convert(MutableSpan span, TraceContext context) {
        // Brave Span → Zipkin JSON API 형식으로 변환
        zipkin2.Span.Builder builder = zipkin2.Span.newBuilder()
            .traceId(context.traceIdString())   // "abc123..."
            .id(context.spanIdString())          // "def456..."
            .name(span.name())                   // "GET /orders"
            .timestamp(span.startTimestamp())    // 마이크로초 단위
            .duration(span.finishTimestamp() - span.startTimestamp())
            .kind(zipkin2.Span.Kind.valueOf(span.kind().name()))
            .localEndpoint(Endpoint.newBuilder()
                .serviceName(span.localServiceName())
                .ip(span.localIp())
                .port(span.localPort())
                .build());

        // 태그 변환
        span.forEachTag((key, value) -> builder.putTag(key, value));

        // parentId 설정 (없으면 루트 Span)
        if (context.parentId() != null) {
            builder.parentId(context.parentIdString());
        }

        return builder.build();
    }
}
```

### 2. AsyncReporter — 비동기 배치 전송

```java
// AsyncReporter.java (Zipkin Reporter 라이브러리)
// Span을 로컬 버퍼에 쌓고 주기적으로 Zipkin으로 배치 전송
// → 개별 Span마다 HTTP 요청 방지 (성능 최적화)

public class AsyncReporter<S> extends Component implements Reporter<S> {

    private final BoundedAsyncReporter<S> delegate;

    @Override
    public void report(S span) {
        // ① 로컬 큐(ByteBoundedQueue)에 추가
        // 큐가 꽉 차면 Span 드롭 (운영 주의!)
        if (!delegate.pending.offer(span)) {
            delegate.metrics.incrementSpansDropped(1);
        }
    }

    // 백그라운드 스레드가 주기적으로 배치 전송
    // messageMaxBytes: 한 번에 보낼 최대 바이트 (기본 512KB)
    // closeTimeout: flush 최대 대기 시간
    void flush() {
        // ② 큐에서 Span 꺼내기 (최대 messageMaxBytes까지)
        List<S> nextBatch = new ArrayList<>();
        long sizeInBytes = 0;

        while (!pending.isEmpty()
                && sizeInBytes < messageMaxBytes) {
            S next = pending.poll();
            nextBatch.add(next);
            sizeInBytes += encoder.sizeInBytes(next);
        }

        if (!nextBatch.isEmpty()) {
            // ③ HTTP POST /api/v2/spans 전송
            sender.sendSpans(encoder.encodeList(nextBatch));
        }
    }
}
```

### 3. Zipkin HTTP API — Span 데이터 구조

```json
// POST http://zipkin:9411/api/v2/spans
// Content-Type: application/json
// Body: (배열로 여러 Span 한 번에 전송)

[
  {
    "traceId": "463ac35c9f6413ad48485a3953bb6124",
    "id": "a2fb4a1d1a96d312",
    "parentId": "0020000000000001",   // 부모 spanId (루트면 없음)
    "name": "get /orders/{id}",       // Span 이름 (소문자)
    "kind": "SERVER",                 // SERVER, CLIENT, PRODUCER, CONSUMER
    "timestamp": 1714000000000000,    // 마이크로초 단위 시작 시각
    "duration": 250000,               // 마이크로초 단위 소요 시간 (250ms)
    "localEndpoint": {
      "serviceName": "order-service",
      "ipv4": "10.0.1.5",
      "port": 8080
    },
    "remoteEndpoint": {               // 원격 엔드포인트 (CLIENT Span)
      "serviceName": "payment-service",
      "ipv4": "10.0.1.6",
      "port": 8081
    },
    "tags": {
      "http.method": "GET",
      "http.url": "/orders/1",
      "http.status_code": "200",
      "order.id": "1",
      "db.statement": "SELECT * FROM orders WHERE id=?"
    },
    "annotations": [                  // 시각이 있는 이벤트
      {
        "timestamp": 1714000000100000,
        "value": "payment-service-called"
      }
    ],
    "shared": false                   // true: 서버와 클라이언트가 같은 spanId 공유 (HTTP 서버)
  }
]
```

### 4. 샘플링 전략 내부 구현

```java
// Sampler: 각 요청을 샘플링할지 결정
// 트레이스 시작 시점(첫 번째 서비스)에 한 번만 결정
// 이후 서비스들은 전파된 sampled 플래그를 따름

// ① 확률 기반 샘플러 (기본)
public final class RateLimitingSampler extends Sampler {
    private final AtomicLong count = new AtomicLong();
    private final int decisionsPerSecond;

    @Override
    public boolean isSampled(long traceId) {
        // traceId 하위 비트로 결정 (결정론적 = 같은 traceId는 항상 같은 결정)
        // decisionsPerSecond: 초당 샘플링할 최대 Trace 수
        long current = count.get();
        return current < decisionsPerSecond;
    }
}

// ② 확률 기반 (Probability)
// management.tracing.sampling.probability: 0.1
// → 각 요청을 10% 확률로 샘플링
// traceId를 해시해서 확률 결정 (결정론적)

// ③ 항상 샘플링 / 절대 안 함
Sampler always = Sampler.ALWAYS_SAMPLE;  // 100%
Sampler never = Sampler.NEVER_SAMPLE;    // 0%

// ④ 커스텀: 오류 요청 항상 샘플링
@Bean
public Sampler customSampler() {
    return (samplerFunction) -> {
        // HttpServerRequest에서 조건 확인 (인터셉터에서)
        return Sampler.ALWAYS_SAMPLE;
    };
}
```

### 5. Zipkin UI 시각화 원리

```
Zipkin이 Trace를 렌더링하는 과정:

① 같은 traceId의 모든 Span 조회
   GET /api/v2/trace/{traceId}

② parentId로 트리 구조 재구성
   Span A (parentId=null) = 루트
     ├── Span B (parentId=A.spanId)
     │     └── Span D (parentId=B.spanId)
     └── Span C (parentId=A.spanId)

③ 타임라인 레이아웃 계산
   전체 타임라인 = 루트 Span의 duration
   각 자식 Span: timestamp와 duration으로 가로 막대 위치/크기 결정

④ 색상 코딩
   오류(error=true 태그): 빨간색
   느린 Span (>p99): 노란색
   정상: 파란색

⑤ SERVER/CLIENT Span 매칭
   ORDER-SERVICE의 CLIENT Span과
   PAYMENT-SERVICE의 SERVER Span이 같은 spanId를 공유하면
   → 하나의 Span으로 합쳐서 표시 (전송 시간 포함)
```

---

## 💻 실전 구성

### Docker Compose: 완전한 Zipkin 환경

```yaml
# docker-compose.yml
services:
  zipkin:
    image: openzipkin/zipkin:latest
    container_name: zipkin
    ports:
      - "9411:9411"
    environment:
      - STORAGE_TYPE=mem        # 개발용 인메모리
      # 운영용 Elasticsearch:
      # STORAGE_TYPE=elasticsearch
      # ES_HOSTS=http://elasticsearch:9200
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:9411/health"]
      interval: 10s
      timeout: 5s
      retries: 10

  # 운영용 Elasticsearch 저장소
  elasticsearch:
    image: elasticsearch:8.12.0
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    volumes:
      - zipkin-es-data:/usr/share/elasticsearch/data

volumes:
  zipkin-es-data:
```

### Span 전송 배치 설정 튜닝

```yaml
# application.yml (Brave AsyncReporter 설정)
spring:
  zipkin:
    base-url: http://zipkin:9411
    sender:
      type: web   # HTTP 전송 (기본값)

# Micrometer Tracing 상세 설정
management:
  tracing:
    brave:
      # 배치 전송 버퍼 크기 (기본값 사용)
      # 커스텀 필요 시 AsyncReporter Bean 직접 등록

# 전송 실패 시:
# Span 드롭 (손실) - 서비스 정상 운영에는 영향 없음
# 로그: [WARN] Dropped x spans due to send error
# → Zipkin 장애가 서비스 장애로 전파되지 않음 (설계 의도)
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: Zipkin 서버 다운 시 서비스 영향

```
Zipkin 서버 다운:
  AsyncReporter: Zipkin 전송 실패
  → Span이 로컬 큐(ByteBoundedQueue)에 쌓임
  → 큐 꽉 차면 새 Span 드롭 (로그: "Dropped x spans")
  
  서비스 자체: 정상 운영 계속
  (Span 전송은 비동기 → 서비스 응답에 영향 없음)

Zipkin 복구 후:
  큐에 남은 Span 전송 재시도
  → 일부 Span 복구 가능 (큐 용량 내)
  → 다운 기간 Span은 손실

교훈:
  분산 추적은 "관측용" 인프라
  → 없어도 서비스 정상 운영
  → 있으면 장애 분석 용이
  
  설계 원칙: "Fail Open"
  → Zipkin 장애 → Span 드롭 → 서비스 무영향
  (vs "Fail Closed" → Zipkin 장애 → 서비스 장애 → 최악)
```

### 시나리오: 느린 쿼리 병목 지점 찾기

```bash
# 1. 느린 API 응답 발생 (5초)
curl http://localhost:8080/api/orders?userId=123

# 2. 응답 헤더에서 traceId 추출
# X-B3-TraceId: abc123def456

# 3. Zipkin UI에서 검색
# http://localhost:9411 → Find Traces
# serviceName: gateway
# Sort: Longest First
# → abc123def456 traceId의 Trace 클릭

# 4. 타임라인 분석
# gateway:       5000ms (전체)
#   order-svc:   4900ms ←
#     inventory: 50ms  ✅
#     db-query:  4800ms ← 이게 범인!
#       [tag] db.statement: SELECT * FROM orders WHERE userId=?
#       [tag] db.rows_affected: 50000  ← 풀스캔!

# 5. 인덱스 추가 → 재테스트 → 50ms로 개선
```

---

## ⚖️ 트레이드오프

| 저장소 | 장점 | 단점 | 적합한 환경 |
|--------|------|------|------------|
| **인메모리 (기본)** | 설정 최소, 빠름 | 재시작 시 데이터 손실 | 개발 |
| **MySQL** | 설정 간단 | 대용량 처리 어려움 | 소규모 운영 |
| **Elasticsearch** | 고성능, 검색 강력 | 설정 복잡, 리소스 많이 사용 | 대규모 운영 |
| **Cassandra** | 고가용성, 대용량 | 운영 복잡 | 초대규모 |

```
샘플링 전략 선택 가이드:

초당 요청 수별 권장 확률:
  < 100 RPS: 100% (1.0) — 모든 요청 추적
  100~1000 RPS: 10% (0.1)
  1000~10000 RPS: 1% (0.01)
  > 10000 RPS: 0.1% (0.001)

오류 기반 샘플링:
  정상: 1%
  HTTP 5xx: 100%
  HTTP 4xx: 10% (클라이언트 오류도 일부 추적)

Rate Limiting 샘플러 (Brave):
  초당 최대 N개만 Zipkin 전송
  트래픽 급등해도 Zipkin 부하 일정하게 유지
```

---

## 📌 핵심 정리

```
Zipkin 연동 핵심:

Span 수집 흐름:
  서비스 내 Span 완료
  → ZipkinSpanHandler.end() 호출
  → sampled 확인 → true면 변환
  → AsyncReporter 큐에 추가 (비동기)
  → 백그라운드 스레드: 배치 전송
  → POST /api/v2/spans to Zipkin

AsyncReporter 특성:
  비동기 전송 → 서비스 성능에 영향 없음
  로컬 큐 → Zipkin 다운 시 일시적 버퍼
  큐 초과 → Span 드롭 (서비스 무영향)
  "Fail Open" 설계

샘플링:
  probability: 0.01~0.1 (운영 권장)
  traceId 해시 기반 → 결정론적 (같은 요청은 항상 같은 결정)
  sampled=false → Zipkin 전송 안 함, 로그 traceId는 유지

Zipkin UI:
  traceId로 전체 체인 조회
  parentId → 트리 구조 재구성
  timestamp/duration → 타임라인 렌더링
  태그 기반 검색
```

---

## 🤔 생각해볼 문제

**Q1.** 서비스 A → 서비스 B를 호출할 때 A에서 CLIENT Span이 생성되고, B에서 SERVER Span이 생성된다. 두 Span의 spanId 관계는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

B3 전파 방식에 따라 다릅니다. 기본적으로 A의 CLIENT Span이 B에게 자신의 spanId를 `X-B3-SpanId`로 전달합니다. B는 이를 자신의 SERVER Span의 spanId로 사용합니다. 즉, A의 CLIENT Span과 B의 SERVER Span은 **동일한 spanId**를 공유합니다. Zipkin에서 `shared=true`로 표시되며, 두 Span을 하나의 RPC 호출로 묶어 표시합니다. 이를 통해 네트워크 전송 시간(A의 CLIENT 종료 시각 - B의 SERVER 시작 시각)도 시각화됩니다.

</details>

---

**Q2.** AsyncReporter의 로컬 큐(ByteBoundedQueue)가 꽉 찼을 때 Span이 드롭되면 traceId 추적에 어떤 구멍이 생기는가?

<details>
<summary>해설 보기</summary>

특정 Span들이 Zipkin에 전달되지 않습니다. Zipkin UI에서 해당 traceId를 조회하면 일부 Span이 빠진 불완전한 Trace가 보입니다. 예를 들어 Order Service의 Span은 있지만 Payment Service의 Span이 없다면 "Payment Service를 호출했는가?"를 추적할 수 없습니다. 그러나 서비스 자체는 정상 동작하며 로그에는 traceId/spanId가 남아 있습니다. 로그를 활용한 수동 추적이 대안입니다.

드롭을 줄이는 방법: 큐 크기 증가(`maxBytesInFlight`), 배치 전송 주기 단축, Zipkin 서버 용량 증가, 샘플링 확률 감소.

</details>

---

**Q3.** `management.tracing.sampling.probability: 0.1`이면 10%만 Zipkin으로 전송되는데, 나머지 90%의 요청에서 오류가 발생하면 어떻게 추적하는가?

<details>
<summary>해설 보기</summary>

세 가지 방법이 있습니다.

1. **로그 기반 추적**: 90% 요청에도 MDC에 traceId/spanId가 있으므로, 로그 집계 시스템(ELK)에서 traceId로 검색해 오류를 추적합니다.

2. **커스텀 Sampler로 오류 100% 샘플링**:
```java
@Bean
public Sampler errorAwareSampler() {
    return (samplingFlags) -> {
        // HTTP 응답 상태 코드 확인 (Span 완료 시점 후처리)
        // 오류면 Span을 Zipkin으로 강제 전송
    };
}
```
실제로는 Span 완료 후 `SpanHandler`에서 오류 여부를 확인해 강제 전송하는 것이 더 실용적입니다.

3. **알람 연동**: 특정 서비스에서 오류 발생 시 알람 → 운영자가 해당 시간대 `probability: 1.0`으로 임시 변경 (Config Server + @RefreshScope) → 문제 재현 및 추적.

</details>

---

<div align="center">

**[⬅️ 이전: Sleuth와 Trace ID / Span ID](./01-sleuth-trace-span.md)** | **[홈으로 🏠](../README.md)** | **[다음: MDC를 통한 로그 추적 ➡️](./03-mdc-log-tracing.md)**

</div>
