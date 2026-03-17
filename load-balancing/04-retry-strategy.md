# Retry 전략 — 실패한 요청을 다른 인스턴스로 재시도하는 메커니즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RetryLoadBalancerInterceptor`가 일반 `LoadBalancerInterceptor`와 다른 점은 무엇인가?
- "동일 인스턴스 재시도"와 "다른 인스턴스 재시도"를 어떻게 구분해서 설정하는가?
- 재시도가 위험한 멱등성 없는 요청(POST)에서 안전하게 재시도하는 조건은?
- `spring.cloud.loadbalancer.retry` 설정 계층과 각 옵션의 역할은?
- Retry가 Circuit Breaker와 함께 사용될 때 어떤 순서로 동작해야 하는가?
- 지수 백오프(Exponential Backoff)는 LoadBalancer Retry에서 어떻게 구현되는가?

---

## 🔍 왜 MSA에서 필요한가

### 재시도 없이 단순 오류 전파의 문제

```
인스턴스 3개, 그중 하나 다운된 상황:
  [A: UP] [B: DOWN] [C: UP]

Round Robin 순서: A → B → C → A → B ...

Retry 없을 때:
  요청 1 → A (성공)
  요청 2 → B (실패: ConnectException)
  → 클라이언트에게 500 반환
  요청 3 → C (성공)
  → 33% 요청 실패

Retry 있을 때:
  요청 2 → B (실패)
  → 자동 재시도: → C (성공)
  → 클라이언트: 성공 응답 수신
  → 오류율 0% (단, 지연은 증가)

Eureka가 B를 90초 후에 레지스트리에서 제거하기 전까지
Retry가 없으면 지속적인 오류 발생

MSA에서 Retry는 일시적 장애에 대한 기본 방어 메커니즘
Circuit Breaker와 함께 사용해 지속적 장애는 격리
```

---

## 😱 잘못된 구성

### Before: 멱등성 없는 메서드에 무분별한 재시도

```yaml
# ❌ 모든 메서드에 Retry 설정
spring:
  cloud:
    loadbalancer:
      retry:
        enabled: true
        retry-on-all-operations: true  # GET, POST, PUT 모두 재시도
        max-retries-on-same-service-instance: 2
        max-retries-on-next-service-instance: 2
```

```
POST /payments {"amount": 10000} 처리 중:
  → payment-service-1 응답 지연 (처리 중)
  → 타임아웃 발생
  → Retry: payment-service-2 로 동일 요청 전송
  → 두 인스턴스 모두 결제 처리 완료
  → 고객에게 이중 결제 발생!

POST /orders {"items": [...]} 재시도:
  → order-service-1 오류 (DB 저장 전 실패): OK 재시도
  → order-service-1 오류 (DB 저장 후 실패): 재시도 → 중복 주문
```

### Before: Circuit Breaker와 Retry 순서 역전

```yaml
# ❌ Retry가 Circuit Breaker 안쪽에 있으면
# Circuit Breaker가 각 재시도를 별도 실패로 카운트
# → Circuit Breaker가 너무 빨리 OPEN

# 올바른 개념:
# Retry(바깥) → Circuit Breaker(안쪽) → 업스트림 호출
# 재시도 N회 모두 실패 → Circuit Breaker 실패 카운트 1 증가
```

---

## ✨ 올바른 패턴

### After: 올바른 Retry 설정

```yaml
spring:
  cloud:
    loadbalancer:
      retry:
        enabled: true

        # 같은 인스턴스 재시도: 0 권장
        # 같은 인스턴스가 이미 실패했으므로 대부분의 경우 의미 없음
        max-retries-on-same-service-instance: 0

        # 다른 인스턴스 재시도: 인스턴스 수 - 1
        # 최악의 경우 모든 인스턴스 시도 가능
        max-retries-on-next-service-instance: 2

        # 멱등성 있는 메서드만 재시도 (기본값)
        retry-on-all-operations: false
        # → GET, HEAD, OPTIONS만 재시도
        # → POST, PUT, DELETE는 재시도 안 함

        # 재시도할 상태 코드 (5xx 서버 오류)
        retryable-status-codes:
          - 502   # Bad Gateway
          - 503   # Service Unavailable
          - 504   # Gateway Timeout

        # 재시도할 예외 타입
        retryable-exceptions:
          - java.io.IOException
          - java.net.ConnectException
```

---

## 🔬 내부 동작 원리

### 1. RetryLoadBalancerInterceptor — Retry 가능한 인터셉터

```java
// RetryLoadBalancerInterceptor.java
// spring-retry 의존성 필요
// LoadBalancerInterceptor를 확장해 재시도 로직 추가

public class RetryLoadBalancerInterceptor implements ClientHttpRequestInterceptor {

    private final LoadBalancerClient loadBalancer;
    private final LoadBalancerRetryProperties lbProperties;
    private final LoadBalancerRequestFactory requestFactory;
    private final LoadBalancedRetryFactory lbRetryFactory;

    @Override
    public ClientHttpResponse intercept(final HttpRequest request,
            final byte[] body,
            final ClientHttpRequestExecution execution) throws IOException {

        final URI originalUri = request.getURI();
        final String serviceName = originalUri.getHost();

        // ① HTTP 메서드 확인: 재시도 허용 여부
        Assert.state(lbProperties.isRetryOnAllOperations()
            || isRequestMethodRetryable(request.getMethod()),
            "Retry is only supported for GET requests");

        // ② 서비스별 RetryPolicy 생성
        final LoadBalancedRetryPolicy retryPolicy =
            lbRetryFactory.createRetryPolicy(serviceName, loadBalancer);

        // ③ RetryTemplate: 재시도 실행 엔진
        RetryTemplate retryTemplate = createRetryTemplate(serviceName,
            request, retryPolicy);

        return retryTemplate.execute(context -> {
            // 매 시도마다 실행되는 콜백

            // ④ 컨텍스트에서 이전 선택 인스턴스 가져오기
            ServiceInstance serviceInstance = null;
            if (context instanceof LoadBalancedRetryContext lbContext) {
                serviceInstance = lbContext.getServiceInstance();
            }

            // ⑤ 인스턴스 선택
            //    - 첫 번째 시도: 새 인스턴스 선택
            //    - 재시도: 정책에 따라 동일 또는 다른 인스턴스
            if (serviceInstance == null) {
                serviceInstance = loadBalancer.choose(serviceName);
            }

            // ⑥ 실제 HTTP 요청
            ClientHttpResponse response = loadBalancer.execute(serviceName,
                serviceInstance,
                requestFactory.createRequest(request, body, execution));

            // ⑦ 응답 상태 코드가 재시도 대상이면 예외 발생
            if (retryPolicy.retryableStatusCode(
                    response.getStatusCode().value())) {
                throw new ClientHttpResponseStatusCodeException(
                    serviceName, response, HttpClientErrorException.class);
            }

            return response;
        }, throwable -> {
            // 재시도 소진 시 Recover 콜백 (없으면 예외 전파)
            throw new RuntimeException(throwable);
        });
    }

    private RetryTemplate createRetryTemplate(String serviceName,
            HttpRequest request, LoadBalancedRetryPolicy retryPolicy) {

        RetryTemplate template = new RetryTemplate();

        // BackOffPolicy: 재시도 간 대기 시간
        BackOffPolicy backOffPolicy = lbRetryFactory
            .createBackOffPolicy(serviceName);
        template.setBackOffPolicy(backOffPolicy != null
            ? backOffPolicy
            : new NoBackOffPolicy());  // 기본: 즉시 재시도

        // RetryPolicy: 재시도 조건 판단
        template.setRetryPolicy(new InterceptorRetryPolicy(
            request, retryPolicy, loadBalancer, serviceName));

        return template;
    }
}
```

### 2. LoadBalancedRetryPolicy — 재시도 조건 판단

```java
// LoadBalancedRetryPolicy.java
public interface LoadBalancedRetryPolicy {

    // ① 이 예외로 재시도할 것인가?
    boolean canRetryNextServer(LoadBalancedRetryContext context);

    // ② 같은 인스턴스에서 재시도할 것인가?
    boolean canRetrySameServer(LoadBalancedRetryContext context);

    // ③ 이 상태 코드로 재시도할 것인가?
    boolean retryableStatusCode(int statusCode);

    // ④ 다음 인스턴스로 전환 시 호출 (로드밸런서에 알림)
    void close(LoadBalancedRetryContext context);

    // ⑤ 재시도 인스턴스 등록
    void registerThrowable(LoadBalancedRetryContext context,
        Throwable throwable);
}

// BlockingLoadBalancedRetryPolicy (기본 구현체):
public class BlockingLoadBalancedRetryPolicy implements LoadBalancedRetryPolicy {

    private final String serviceId;
    private final LoadBalancedRetryProperties properties;

    @Override
    public boolean canRetryNextServer(LoadBalancedRetryContext context) {
        // 다음 인스턴스로 재시도 가능한지 확인
        int maxRetries = properties.getMaxRetriesOnNextServiceInstance();
        // context.getRetryCount(): 현재까지 "다른 인스턴스" 재시도 횟수
        return context.getRetryCount() < maxRetries
            && isRetryable(context.getLastThrowable());
    }

    @Override
    public boolean canRetrySameServer(LoadBalancedRetryContext context) {
        // 같은 인스턴스 재시도 가능한지
        int maxRetries = properties.getMaxRetriesOnSameServiceInstance();
        return context.getSameServerCount() < maxRetries
            && isRetryable(context.getLastThrowable());
    }

    private boolean isRetryable(Throwable throwable) {
        if (throwable == null) return false;
        // 설정된 예외 타입 중 하나이면 재시도
        return properties.getRetryableExceptions().stream()
            .anyMatch(ex -> ex.isInstance(throwable)
                || ex.isInstance(throwable.getCause()));
    }
}
```

### 3. 지수 백오프 설정

```java
// BackOffPolicy: 재시도 간 대기 시간 설정
// spring-cloud-starter-loadbalancer에서 직접 지원하지 않음
// spring-retry의 BackOffPolicy를 LoadBalancedRetryFactory로 커스터마이징

@Configuration
public class RetryConfig {

    @Bean
    public LoadBalancedRetryFactory retryFactory() {
        return new LoadBalancedRetryFactory() {
            @Override
            public BackOffPolicy createBackOffPolicy(String service) {
                ExponentialRandomBackOffPolicy backOffPolicy =
                    new ExponentialRandomBackOffPolicy();
                backOffPolicy.setInitialInterval(100);  // 첫 재시도 대기 100ms
                backOffPolicy.setMaxInterval(1000);     // 최대 1000ms
                backOffPolicy.setMultiplier(2.0);       // 2배씩 증가
                return backOffPolicy;
                // 재시도 간격: 100ms → 200ms → 400ms → ...최대 1000ms
            }
        };
    }
}
```

### 4. Retry + Circuit Breaker 올바른 조합

```
실행 순서 (올바른 설계):

요청 → Retry 레이어 (바깥)
         → Circuit Breaker 레이어 (안쪽)
              → LoadBalancer (인스턴스 선택)
                   → HTTP 요청 (업스트림)

1차 시도: 인스턴스 A → 실패 (ConnectException)
          Circuit Breaker: 실패 카운트 +1
          Retry: 재시도 조건 충족 → 2차 시도

2차 시도: 인스턴스 B → 실패 (503)
          Circuit Breaker: 실패 카운트 +2
          Retry: 재시도 소진 (max=2)

클라이언트: 오류 응답

Circuit Breaker 실패 카운트:
  2회 실패 (1차 + 2차)
  slidingWindowSize=10, failureRateThreshold=50
  → 아직 OPEN 조건 미달

여러 요청이 이 패턴으로 실패하면 결국 Circuit Breaker OPEN
→ 이후 요청은 업스트림 호출 없이 즉시 Fallback

잘못된 순서 (Circuit Breaker 바깥, Retry 안쪽):
  Circuit Breaker → Retry 3회 시도 → 모두 실패
  Circuit Breaker는 이 전체를 "1번의 실패"로 카운트
  → Circuit Breaker가 너무 느리게 OPEN
  → 과도한 재시도로 업스트림 부하 증가
```

---

## 💻 실전 구성

### 의존성 추가

```xml
<!-- pom.xml: Retry 기능 활성화 -->
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
    <!-- spring-retry의 @Retryable 사용 시 AOP 필요 -->
</dependency>
```

```yaml
# application.yml: 완전한 Retry 설정
spring:
  cloud:
    loadbalancer:
      retry:
        enabled: true
        retry-on-all-operations: false   # GET만 재시도
        max-retries-on-same-service-instance: 0
        max-retries-on-next-service-instance: 2
        retryable-status-codes:
          - 502
          - 503
          - 504

# 서비스별 재시도 설정 오버라이드
      clients:
        order-service:
          retry:
            max-retries-on-next-service-instance: 1   # order-service는 1회만
        payment-service:
          retry:
            enabled: false   # payment-service는 재시도 없음 (멱등성 없음)
```

### Retry 동작 확인

```bash
# 인스턴스 1개를 임시 다운 후 테스트
docker stop service-order-2

# 요청 전송 (Retry 로그 확인)
curl -v http://localhost:8080/api/orders/1

# 로그:
# [DEBUG] Trying service-order at 10.0.1.6:8080 (attempt 1)
# [WARN] Failed to connect to 10.0.1.6:8080 - Connection refused
# [DEBUG] Retrying with new instance for service-order
# [DEBUG] Trying service-order at 10.0.1.5:8080 (attempt 2)
# [DEBUG] Success at 10.0.1.5:8080
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: Rolling Update 중 502 발생 시 Retry

```
K8s Rolling Update 진행 중:
  service-order-v1 (Pod 2개 → 1개로 줄이는 중)
  service-order-v2 (Pod 0개 → 1개로 늘리는 중)

t=0s: v1-pod-1 종료 시작
       Graceful Shutdown: 기존 요청 완료 대기
       Eureka: 아직 v1-pod-1 등록 상태 (DELETE 전송 전)

t=1s: v1-pod-1이 일부 요청에 502 반환 (종료 중)
       LoadBalancer: v1-pod-1 선택 → 502 수신
       RetryPolicy.retryableStatusCode(502) = true
       → 재시도: v1-pod-2 선택 → 성공!

t=5s: v1-pod-1 완전 종료 (DELETE → Eureka에서 제거)
      v2-pod-1 시작 완료 → Eureka 등록 → UP

결과:
  Rolling Update 중에도 Retry 덕분에 클라이언트 오류 없음
  요청 지연은 재시도 시간만큼 증가 (수십~수백ms)
```

---

## ⚖️ 트레이드오프

| 설정 | 장점 | 단점 | 적합한 케이스 |
|------|------|------|--------------|
| **동일 인스턴스 재시도 (max>0)** | 일시적 오류 극복 | 이미 죽은 인스턴스에 부하 | 일시적 오류가 많은 환경 |
| **다른 인스턴스 재시도** | 죽은 인스턴스 우회 | 요청 지연 증가 | 권장 기본 전략 |
| **retry-on-all-operations=true** | 높은 가용성 | 멱등성 없는 요청 중복 위험 | 업스트림 멱등성 보장 시만 |
| **지수 백오프** | 폭발적 재시도 방지 | 응답 지연 증가 | 부하 높은 환경 |

```
Retry와 Timeout의 상호작용:
  connect-timeout: 1s
  max-retries-on-next-service-instance: 3
  지수 백오프: 100ms, 200ms, 400ms

  최악의 경우 총 시간:
  (1s + 100ms) + (1s + 200ms) + (1s + 400ms) + 1s
  = 1.1 + 1.2 + 1.4 + 1 = 4.7초

  클라이언트가 이 시간을 기다릴 수 있는지 고려해야 함
  클라이언트 timeout < 서버 총 retry 시간 → 클라이언트가 먼저 포기

  실무: 클라이언트 timeout을 충분히 넉넉하게 설정하거나
        서버 retry 횟수를 줄여 총 시간 제한
```

---

## 📌 핵심 정리

```
Retry 전략 핵심:

RetryLoadBalancerInterceptor:
  LoadBalancerInterceptor + spring-retry RetryTemplate
  RetryPolicy: 재시도 조건 (예외 타입, 상태 코드, 횟수)
  BackOffPolicy: 재시도 간 대기 시간 (즉시, 고정, 지수)

핵심 설정:
  same-service-instance: 0 (같은 인스턴스 재시도 안 함)
  next-service-instance: 1~2 (다른 인스턴스로 재시도)
  retry-on-all-operations: false (GET만, POST 제외)

멱등성 원칙:
  재시도 = 같은 요청을 여러 번 보내는 것
  GET, HEAD → 멱등 → 재시도 안전
  POST, PUT → 멱등성 보장 안 됨 → 재시도 위험
  Idempotency Key로 업스트림에서 멱등성 구현 시 POST도 가능

Retry + Circuit Breaker 순서:
  Retry(바깥) → Circuit Breaker(안쪽) → 업스트림
  재시도 N회 모두 실패 = CB 실패 카운트 1 증가
  (반대 순서면 CB가 너무 느리게 OPEN)
```

---

## 🤔 생각해볼 문제

**Q1.** `max-retries-on-same-service-instance: 1`로 설정하면 어떤 케이스에서 유용한가? 대부분의 경우 0을 권장하는 이유는?

<details>
<summary>해설 보기</summary>

같은 인스턴스 재시도가 유용한 케이스: 연결은 됐지만 단순 타임아웃이 발생한 경우(서버가 살아있지만 잠깐 부하가 높았음)입니다. 같은 인스턴스에 한 번 더 시도하면 성공할 수 있습니다.

대부분 0을 권장하는 이유:
1. `ConnectException`(연결 자체 실패)은 같은 인스턴스에 재시도해도 동일하게 실패합니다.
2. 인스턴스가 과부하 상태이면 같은 인스턴스 재시도가 상황을 악화시킵니다.
3. 다른 인스턴스로 전환하는 것이 더 효과적입니다(분산 처리).
4. 같은 인스턴스 재시도는 "인스턴스 전환" 재시도 횟수를 소모하지 않아 총 재시도 횟수 계산이 복잡해집니다.

예외적으로 1을 설정하는 경우: 단일 인스턴스 환경(개발), 다른 인스턴스가 없을 때의 폴백입니다.

</details>

---

**Q2.** `retryable-status-codes`에 `500 Internal Server Error`를 추가하면 어떤 위험이 있는가?

<details>
<summary>해설 보기</summary>

500은 서버 내부 오류로, 비즈니스 로직 오류, DB 제약 조건 위반, 입력 검증 실패 등 다양한 이유로 발생합니다. 이런 경우 재시도해도 같은 500이 반환됩니다.

위험:
1. **불필요한 재시도**: 입력 데이터 문제라면 재시도해도 항상 500 → retry 횟수 낭비.
2. **부하 증가**: 모든 500에 N번 재시도 → 업스트림 서버에 N배 부하.
3. **중복 처리 위험**: 500이 DB 저장 후 발생했다면 재시도 시 중복 처리.
4. **Circuit Breaker 오작동**: 정상적인 비즈니스 오류(400 계열이어야 하는데 잘못 500으로 반환)에도 Circuit Breaker가 열릴 수 있음.

권장: 502(Bad Gateway), 503(Service Unavailable), 504(Gateway Timeout)만 재시도 대상으로 설정합니다. 이들은 인프라 수준 문제이므로 다른 인스턴스에서 성공할 가능성이 높습니다.

</details>

---

**Q3.** RestTemplate에 `@LoadBalanced`와 `RetryLoadBalancerInterceptor`가 모두 설정됐을 때, 요청 인터셉터 실행 순서는 어떻게 결정되는가?

<details>
<summary>해설 보기</summary>

`ClientHttpRequestInterceptor`는 `RestTemplate.getInterceptors()` 리스트 순서대로 실행됩니다. Spring Cloud LoadBalancer Auto-configuration은 `RetryLoadBalancerInterceptor`를 `LoadBalancerInterceptor` 대신 등록합니다(둘 다 동시에 등록되지 않음). `spring-retry`가 클래스패스에 있으면 `RetryLoadBalancerInterceptor`가, 없으면 `LoadBalancerInterceptor`가 등록됩니다.

`RestTemplateCustomizer`를 통해 인터셉터가 추가되는 순서는 다음과 같습니다:
1. `@LoadBalanced` RestTemplate에 `LoadBalancerAutoConfiguration`이 `LoadBalancerInterceptor`(또는 `RetryLoadBalancerInterceptor`) 추가
2. 다른 `RestTemplateCustomizer` Bean들이 추가한 인터셉터들

순서 변경이 필요하면 직접 `restTemplate.setInterceptors(list)`로 인터셉터 목록을 재구성합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Load Balancing 알고리즘](./03-load-balancing-algorithms.md)** | **[홈으로 🏠](../README.md)** | **[다음: Custom LoadBalancer 구현 ➡️](./05-custom-load-balancer.md)**

</div>
