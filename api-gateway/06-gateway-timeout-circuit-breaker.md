# Gateway Timeout & Circuit Breaker 통합 — 장애 격리와 Fallback 처리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `connect-timeout`과 `response-timeout`은 각각 어느 레이어에서 어떻게 적용되는가?
- 글로벌 타임아웃과 라우트별 타임아웃은 어떻게 설정하며, 둘이 충돌하면 어느 것이 우선인가?
- `SpringCloudCircuitBreakerFilterFactory`가 Resilience4j `CircuitBreaker`를 호출하는 흐름은?
- Fallback URI `forward:/fallback` 처리 시 요청이 어떻게 Gateway 내부로 다시 포워드되는가?
- Circuit Breaker가 OPEN 상태일 때 Gateway는 업스트림을 호출하는가?
- `CircuitBreaker` 필터와 `Retry` 필터를 함께 사용할 때 실행 순서가 중요한 이유는?

---

## 🔍 왜 MSA에서 필요한가

### Gateway에서의 Timeout과 Circuit Breaker의 역할

```
Gateway가 없는 MSA에서의 장애 전파:

Service A → Service B → Service C (응답 10초)
                           ↓
Service B: 스레드 10초 점유 → 스레드 고갈
Service A: Service B 응답 10초 대기 → 스레드 고갈
Gateway: 모든 요청 타임아웃

Gateway에서 Timeout + Circuit Breaker 적용:

클라이언트 → Gateway → Service A → Service B → Service C
                ↓
           Timeout: 2초 설정
             → Service B가 2초 내 응답 안 하면 504 반환
             → Gateway 스레드 2초만 점유

           Circuit Breaker:
             → 연속 실패 → OPEN 상태
             → OPEN 중: 업스트림 호출 없이 즉시 Fallback 응답
             → Service C 완전 다운이어도 Gateway는 빠르게 응답

결과:
  특정 서비스 장애가 Gateway 전체로 전파되지 않음
  장애 서비스의 복구 시간 동안 Fallback으로 서비스 유지
```

---

## 😱 잘못된 구성

### Before: Timeout 없이 업스트림이 느릴 때

```yaml
# ❌ Timeout 설정 없음
spring:
  cloud:
    gateway:
      routes:
        - id: slow-service-route
          uri: lb://slow-service
          # timeout 설정 없음 → 응답 올 때까지 무한 대기
```

```
결과:
  slow-service가 5분 동안 응답 안 함
  → Gateway의 Netty 연결이 5분간 점유
  → 동시 요청이 쌓이면서 연결 수 고갈
  → 다른 서비스로의 라우팅도 영향받음

  Netty는 Non-Blocking이지만 연결 자체는 유지됨
  → 연결 수 한계(기본 ~5000) 도달 → 새 요청 거절
```

### Before: Retry와 Circuit Breaker 순서 오류

```yaml
# ❌ 잘못된 필터 순서
filters:
  - name: CircuitBreaker     # 먼저 실행 (Order 낮음)
    args:
      name: my-circuit-breaker
      fallbackUri: forward:/fallback
  - name: Retry              # 나중에 실행
    args:
      retries: 3
```

```
문제:
  Circuit Breaker가 먼저 실행되어 Retry보다 바깥 레이어에 위치
  
  실패 시:
  CircuitBreaker → Retry(3회 재시도) → 여전히 실패
  → CircuitBreaker: 이 "실패 1회"로 카운트
  
  올바른 의도:
  Retry(3회) 모두 실패해야 Circuit Breaker 실패로 카운트
  
  올바른 순서:
  Retry가 바깥(먼저 실행), CircuitBreaker가 안쪽(나중 실행)
  → Retry 3회 모두 실패 → 그때 CircuitBreaker가 실패 카운트
```

---

## ✨ 올바른 패턴

### After: Timeout + Circuit Breaker + Retry 완전한 설정

```yaml
spring:
  cloud:
    gateway:
      # 글로벌 타임아웃 (모든 라우트 기본값)
      httpclient:
        connect-timeout: 1000       # 연결 타임아웃: 1초 (ms)
        response-timeout: 5s        # 응답 타임아웃: 5초 (Duration)

      routes:
        - id: order-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            # 올바른 순서: Retry → CircuitBreaker → 실제 호출
            - name: Retry
              args:
                retries: 2
                statuses: BAD_GATEWAY,SERVICE_UNAVAILABLE
                methods: GET
                backoff:
                  firstBackoff: 100ms
                  maxBackoff: 500ms
                  factor: 2

            - name: CircuitBreaker
              args:
                name: order-circuit-breaker
                fallbackUri: forward:/fallback/orders

          # 라우트별 타임아웃 (글로벌 오버라이드)
          metadata:
            response-timeout: 3000    # 이 라우트만 3초
            connect-timeout: 500      # 이 라우트만 0.5초
```

---

## 🔬 내부 동작 원리

### 1. connect-timeout과 response-timeout의 적용 레이어

```java
// HttpClientFactory.java
// connect-timeout: Netty Channel 연결 시 적용

@Bean
public HttpClient gatewayHttpClient(HttpClientProperties properties) {
    HttpClient httpClient = HttpClient.create();

    // ① connect-timeout: TCP 핸드셰이크 완료까지 대기 시간
    if (properties.getConnectTimeout() != null) {
        httpClient = httpClient.option(
            ChannelOption.CONNECT_TIMEOUT_MILLIS,
            properties.getConnectTimeout()
        );
        // Netty Channel 레벨: 소켓 연결 자체의 타임아웃
        // 연결 이후에는 적용 안 됨
    }

    // ② response-timeout: 요청 전송 완료 ~ 응답 첫 바이트 수신까지
    if (properties.getResponseTimeout() != null) {
        httpClient = httpClient.responseTimeout(properties.getResponseTimeout());
        // Netty Channel 레벨에서 ReadTimeoutHandler로 구현
    }

    return httpClient;
}

// 라우트별 타임아웃 오버라이드:
// NettyRoutingFilter가 라우트 metadata에서 값 읽어 적용
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);

    // metadata에서 라우트별 타임아웃 읽기
    Long routeResponseTimeout = getRouteMetadataOrDefault(
        route, RESPONSE_TIMEOUT_ATTR, null);
    Integer routeConnectTimeout = getRouteMetadataOrDefault(
        route, CONNECT_TIMEOUT_ATTR, null);

    // 라우트별 설정이 있으면 글로벌 설정 오버라이드
    HttpClient currentClient = getHttpClient(route, exchange);

    if (routeResponseTimeout != null) {
        currentClient = currentClient.responseTimeout(
            Duration.ofMillis(routeResponseTimeout));
    }
    // ...
}
```

### 2. SpringCloudCircuitBreakerFilterFactory — Resilience4j 통합

```java
// SpringCloudCircuitBreakerFilterFactory.java
public class SpringCloudCircuitBreakerFilterFactory
        extends AbstractGatewayFilterFactory<SpringCloudCircuitBreakerFilterFactory.Config> {

    private final ReactiveCircuitBreakerFactory cbFactory;

    @Override
    public GatewayFilter apply(Config config) {
        // ① Resilience4j CircuitBreaker 인스턴스 생성 (또는 기존 것 재사용)
        ReactiveCircuitBreaker cb = cbFactory.create(config.getName());

        return (exchange, chain) -> {
            // ② chain.filter()를 Circuit Breaker로 감쌈
            return cb.run(
                // 실제 실행: 업스트림으로 요청 전달
                chain.filter(exchange).then(Mono.just(exchange)),

                // Fallback: 실패 시 실행
                throwable -> {
                    if (config.getFallbackUri() != null) {
                        // ③ Fallback URI 처리
                        return handleFallback(exchange, config.getFallbackUri(), throwable);
                    }
                    // Fallback 없으면 예외 전파
                    return Mono.error(throwable);
                }
            ).then();
        };
    }

    private Mono<Void> handleFallback(ServerWebExchange exchange,
            URI fallbackUri, Throwable cause) {

        // fallbackUri = "forward:/fallback/orders"
        // Exchange에 원인 예외와 라우팅 정보 저장
        exchange.getAttributes().put(CIRCUIT_BREAKER_EXECUTION_EXCEPTION_ATTR, cause);

        // forward: 스킴 → Gateway 내부 핸들러로 포워딩
        ServerWebExchangeUtils.setAlreadyRouted(exchange);
        exchange.getAttributes().put(SERVER_WEB_EXCHANGE_ATTR, exchange);

        // ④ Gateway 내부 DispatcherHandler로 재디스패치
        URI fallbackUriToUse = URI.create("forward:" + fallbackUri.getPath());
        return this.dispatcherHandler.handle(
            exchange.mutate()
                .request(exchange.getRequest().mutate()
                    .uri(fallbackUriToUse)
                    .build())
                .build()
        );
    }
}
```

### 3. ReactiveCircuitBreakerFactory — Resilience4j 설정

```java
// Resilience4jReactiveCircuitBreakerFactory.java

// application.yml에서 Circuit Breaker 동작 설정:
// resilience4j:
//   circuitbreaker:
//     instances:
//       order-circuit-breaker:
//         slidingWindowSize: 10
//         failureRateThreshold: 50
//         waitDurationInOpenState: 5s
//         permittedNumberOfCallsInHalfOpenState: 3

// CircuitBreaker 상태 전이 (Ch5에서 상세):
// CLOSED → 실패율 50% 이상 → OPEN
// OPEN → 5초 후 → HALF_OPEN (3회 허용 호출)
// HALF_OPEN → 성공 → CLOSED
//           → 실패 → OPEN

// ReactiveCircuitBreaker.run() 동작:
@Override
public <T> Mono<T> run(Mono<T> toRun, Function<Throwable, Mono<T>> fallback) {
    // ① Reactive 체인을 Circuit Breaker로 변환
    Mono<T> decorated = circuitBreakerOperator.apply(toRun);
    // CircuitBreakerOperator: 호출 성공/실패를 Circuit Breaker에 통보

    if (fallback != null) {
        // ② 예외 시 Fallback 실행
        return decorated
            .onErrorResume(CallNotPermittedException.class, fallback)  // OPEN 상태
            .onErrorResume(fallback);                                   // 일반 오류
    }
    return decorated;
}
```

### 4. Fallback Controller 구현

```java
// Gateway 내부에 Fallback 핸들러 구현
// forward:/fallback/orders → 이 컨트롤러로 포워딩

@RestController
@RequestMapping("/fallback")
public class FallbackController {

    // Circuit Breaker가 발동된 원인 예외 접근 가능
    @GetMapping("/orders")
    public Mono<ResponseEntity<Map<String, Object>>> ordersFallback(
            ServerWebExchange exchange) {

        // 원인 예외 추출
        Throwable cause = exchange.getAttribute(
            CIRCUIT_BREAKER_EXECUTION_EXCEPTION_ATTR);

        String reason = cause instanceof CallNotPermittedException
            ? "Circuit Breaker OPEN - service unavailable"
            : "Service temporarily unavailable";

        Map<String, Object> body = Map.of(
            "status", 503,
            "message", reason,
            "timestamp", Instant.now(),
            "fallback", true
        );

        return Mono.just(ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(body));
    }

    // 범용 Fallback (모든 서비스 공통)
    @RequestMapping("/default")
    public Mono<ResponseEntity<Map<String, Object>>> defaultFallback() {
        return Mono.just(ResponseEntity
            .status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of("message", "Service temporarily unavailable")));
    }
}
```

---

## 💻 실전 구성

### 완전한 장애 격리 구성

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        connect-timeout: 1000
        response-timeout: 5s
        pool:
          max-connections: 500        # 최대 커넥션 풀
          acquire-timeout: 3000       # 풀에서 커넥션 획득 타임아웃

      routes:
        - id: payment-route
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
          filters:
            - name: RequestRateLimiter   # 요청 속도 제한
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200

            - name: Retry
              args:
                retries: 2
                statuses: SERVICE_UNAVAILABLE
                methods: GET,POST
                backoff:
                  firstBackoff: 200ms
                  maxBackoff: 2000ms
                  factor: 2

            - name: CircuitBreaker
              args:
                name: payment-cb
                fallbackUri: forward:/fallback/payments
                statusCodes:            # 이 상태 코드를 실패로 간주
                  - BAD_GATEWAY
                  - SERVICE_UNAVAILABLE
                  - GATEWAY_TIMEOUT

          metadata:
            response-timeout: 10000   # 결제는 10초 허용
            connect-timeout: 1000

resilience4j:
  circuitbreaker:
    instances:
      payment-cb:
        registerHealthIndicator: true
        slidingWindowSize: 20
        minimumNumberOfCalls: 5
        failureRateThreshold: 60
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 5
```

```bash
# Circuit Breaker 상태 모니터링
curl http://api-gateway:8080/actuator/circuitbreakers
# 응답:
# {
#   "circuitBreakers": {
#     "payment-cb": {
#       "failureRate": "35.0%",
#       "state": "CLOSED",
#       "numberOfBufferedCalls": 20,
#       "numberOfSuccessfulCalls": 13,
#       "numberOfFailedCalls": 7
#     }
#   }
# }
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: payment-service 장애 → Circuit Breaker 동작

```
t=0:   payment-service 인스턴스 모두 다운

t=0~5s: 요청들이 payment-service로 전달됨
         → ConnectException (연결 실패)
         → Retry 2회 → 모두 실패
         → CircuitBreaker: 실패 카운트 증가

t=5s:  failureRateThreshold(60%) 초과
         → CircuitBreaker: CLOSED → OPEN 전환
         → 이후 요청: 업스트림 호출 없이 즉시 Fallback 응답
         → 응답 시간: ~1ms (Fallback 즉시 반환)

t=5~15s: Circuit Breaker OPEN 상태
          모든 payment 요청 → Fallback: {"status": 503, "fallback": true}
          payment-service 복구 시간 확보

t=15s: waitDurationInOpenState(10s) 경과
        → OPEN → HALF_OPEN
        → permittedNumberOfCallsInHalfOpenState(5): 5개 요청 허용

t=15~16s: 5개 테스트 요청 → payment-service 응답 정상
           → HALF_OPEN → CLOSED
           → 정상 라우팅 재개
```

---

## ⚖️ 트레이드오프

| 설정 | 빠른 장애 감지 | 안정성 | 주의사항 |
|------|--------------|--------|---------|
| `connect-timeout: 100ms` | 매우 빠름 | 낮음 (정상 네트워크 지연에 오탐) | 네트워크 RTT 기준 설정 |
| `response-timeout: 1s` | 빠름 | 낮음 (느린 정상 처리 오탐) | 업스트림 SLA 기준 설정 |
| `failureRateThreshold: 30%` | 빠름 | 낮음 (오탐 가능) | 트래픽 많을 때만 유효 |
| `waitDurationInOpenState: 30s` | - | 높음 (장기 복구 시간) | 너무 길면 서비스 중단 연장 |

```
Retry + Circuit Breaker 조합 시 Retry 재시도 횟수 계산:

Circuit Breaker slidingWindowSize: 10
Retry retries: 3

요청 1: 실패 → Retry 3회 → Circuit Breaker 실패 카운트 +1 (재시도는 같은 요청)
요청 2: 실패 → Retry 3회 → Circuit Breaker 실패 카운트 +1

총 10번의 "논리적 요청"이 실패해야 OPEN

주의:
  Retry가 바깥이면 "논리적 실패 단위"로 카운트
  CircuitBreaker가 바깥이면 "물리적 요청 실패"로 카운트
  → Retry를 안쪽에 두면 Circuit Breaker가 너무 빨리 열릴 수 있음
```

---

## 📌 핵심 정리

```
Timeout & Circuit Breaker 핵심:

connect-timeout:
  Netty ChannelOption.CONNECT_TIMEOUT_MILLIS
  TCP 연결 성립까지 대기 시간
  → 라우트별: metadata.connect-timeout

response-timeout:
  응답 첫 바이트 수신까지 대기 시간
  → 라우트별: metadata.response-timeout (우선)
  → 글로벌: httpclient.response-timeout (기본값)

CircuitBreaker 필터:
  SpringCloudCircuitBreakerFilterFactory
  → Resilience4j ReactiveCircuitBreaker.run() 호출
  → 실패/OPEN 시: fallbackUri로 포워딩
  → forward:/fallback → Gateway 내부 컨트롤러

Retry + CircuitBreaker 올바른 순서:
  Retry 필터 (바깥) → CircuitBreaker 필터 (안쪽) → 업스트림
  Retry가 먼저 실행되어 재시도를 전부 처리한 후
  그 결과를 CircuitBreaker가 실패/성공으로 카운트

Fallback 처리:
  exchange.getAttribute(CIRCUIT_BREAKER_EXECUTION_EXCEPTION_ATTR)
  → 원인 예외 분석 (CallNotPermittedException vs 실제 오류)
  → 적절한 Fallback 응답 구성
```

---

## 🤔 생각해볼 문제

**Q1.** `response-timeout`이 3초로 설정된 상태에서 업스트림이 2.9초 만에 응답한다. 그런데 Retry 횟수가 3회면 총 응답 시간은 얼마까지 가능한가?

<details>
<summary>해설 보기</summary>

각 재시도마다 `response-timeout`이 독립적으로 적용됩니다. 따라서:
- 1차 시도: 2.9초 → 실패(타임아웃)
- 재시도 간 backoff: firstBackoff 100ms → 200ms → 400ms
- 2차 시도: 2.9초 → 실패
- 3차 시도: 2.9초 → 실패
- 총 시간: 2.9 + 0.1 + 2.9 + 0.2 + 2.9 = 약 9초

클라이언트는 9초 동안 대기 후 최종 실패 응답을 받습니다. Gateway의 `response-timeout`을 전체 Retry 포함 총 시간의 상한으로 설정하려면 Reactor의 `timeout()` 연산자를 Custom Filter에서 추가하거나, 클라이언트 측 타임아웃을 더 짧게 설정해야 합니다.

</details>

---

**Q2.** Circuit Breaker가 OPEN 상태일 때 Gateway가 업스트림을 호출하지 않는다고 했다. 이때 `NettyRoutingFilter`는 실행되는가?

<details>
<summary>해설 보기</summary>

실행되지 않습니다. `CircuitBreakerFilter`가 `chain.filter(exchange)`를 호출하지 않고 즉시 Fallback Mono를 반환하기 때문입니다. 이 필터가 체인의 중간에 위치하므로, `chain.filter(exchange)` 호출이 없으면 체인의 나머지 필터들(나중 Order를 가진 것들, 포함해서 `NettyRoutingFilter`)이 실행되지 않습니다. `CallNotPermittedException`이 발생하면 즉시 Fallback으로 분기됩니다. 이것이 Circuit Breaker의 핵심 이점입니다 — 업스트림 호출 자체를 완전히 차단하여 장애 서비스에 불필요한 부하를 가하지 않습니다.

</details>

---

**Q3.** `fallbackUri: forward:/fallback/orders`가 설정됐을 때 이 Fallback 컨트롤러가 없으면 어떻게 되는가? 또한 Fallback 컨트롤러 자체에서 예외가 발생하면 어떻게 처리되는가?

<details>
<summary>해설 보기</summary>

**Fallback 컨트롤러가 없을 때**: Gateway의 `DispatcherHandler`가 `/fallback/orders` 경로를 처리할 핸들러를 찾지 못해 404 Not Found가 반환됩니다. 클라이언트는 원래 기대했던 503 대신 404를 받게 되어 혼란스러울 수 있습니다. 반드시 Fallback URI에 대응하는 컨트롤러를 구현해야 합니다.

**Fallback 컨트롤러에서 예외 발생 시**: 예외가 일반 WebFlux 예외 처리로 넘어가 Gateway의 `DefaultErrorWebExceptionHandler`가 처리합니다. 기본적으로 500 Internal Server Error와 에러 응답 바디가 반환됩니다. Fallback 자체가 실패하는 상황은 최악의 경우이므로, Fallback 컨트롤러는 외부 의존성 없이 단순히 정적 응답을 반환하도록 구현해야 합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Route Locator 동적 라우팅](./05-route-locator-dynamic-routing.md)** | **[홈으로 🏠](../README.md)** | **[다음: Custom Filter 작성 ➡️](./07-custom-filter.md)**

</div>
