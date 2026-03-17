# Gateway Filter Factory — 요청·응답을 변환하는 필터 구현 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `GatewayFilterFactory.apply(Config)`가 `GatewayFilter` 람다를 반환하는 설계 의도는?
- `AddRequestHeaderGatewayFilterFactory`가 요청 헤더를 추가할 때 `exchange.mutate()`를 사용하는 이유는?
- `RewritePathGatewayFilterFactory`가 URI를 재작성하는 정확한 시점과 방법은?
- `RetryGatewayFilterFactory`의 재시도 조건(상태 코드, 메서드, 예외 타입)은 어떻게 설정하는가?
- Route Filter와 Global Filter의 실행 순서가 `@Order`로 결정되는 원리는?
- `GatewayFilter`와 `GlobalFilter`의 인터페이스 차이는 무엇인가?

---

## 🔍 왜 MSA에서 필요한가

### Gateway가 요청·응답을 변환해야 하는 이유

```
클라이언트와 업스트림 서비스 사이에서 Gateway가 해야 할 일:

요청 변환 (Pre-filter):
  클라이언트: GET /api/orders/1
  업스트림:   GET /orders/1      (prefix /api 제거)
  
  클라이언트: 인증 토큰만 전달
  업스트림:   X-User-Id: 123 헤더 추가 (JWT 파싱 후 주입)
  
  클라이언트: 구 API 경로 /v1/users
  업스트림:   /v2/users 로 재작성

응답 변환 (Post-filter):
  업스트림: X-Internal-Server: host123 (내부 정보 노출)
  클라이언트: 해당 헤더 제거
  
  업스트림: 응답 본문 압축되지 않음
  클라이언트: gzip 압축 적용

공통 관심사:
  모든 요청/응답에 X-Request-Id 추가
  모든 응답에 CORS 헤더 추가
  재시도 로직 (5xx → 최대 3회 재시도)

이 모든 변환을 각 서비스에서 구현하면 중복
→ Gateway Filter Factory가 중앙에서 처리
```

---

## 😱 잘못된 구성

### Before: ServerWebExchange 직접 수정 시도

```java
// ❌ ServerWebExchange는 불변 객체 — 직접 수정 불가
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    // ❌ ServerHttpRequest는 불변 → 헤더 직접 추가 불가
    exchange.getRequest().getHeaders().add("X-User-Id", "123");
    // UnsupportedOperationException 발생!

    return chain.filter(exchange);
}
```

```java
// ❌ 응답 헤더 직접 수정 — 이미 응답이 커밋된 경우 효과 없음
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    return chain.filter(exchange)
        .doOnSuccess(v -> {
            // ❌ 응답이 이미 전송 중일 수 있음
            exchange.getResponse().getHeaders().add("X-Response-Time", "100ms");
            // 헤더가 이미 전송됐으면 효과 없음
        });
}
```

---

## ✨ 올바른 패턴

### After: GatewayFilterFactory 구조와 exchange.mutate() 패턴

```java
// GatewayFilterFactory 인터페이스
public interface GatewayFilterFactory<C> extends ShortcutConfigurable, Ordered {

    // Config 클래스 타입 반환 (YAML 설정 바인딩용)
    Class<C> getConfigClass();

    // YAML 설정 → Config 인스턴스 → GatewayFilter 반환
    GatewayFilter apply(C config);

    // 기본 구현: apply() 래핑
    default AsyncGatewayFilterFactory<C> applyAsync(C config) {
        GatewayFilter filter = apply(config);
        return (exchange, chain) -> filter.filter(exchange, chain);
    }
}

// 모든 내장 필터의 공통 패턴:
// apply(Config) → (exchange, chain) -> { 변환 로직 } 람다 반환
```

---

## 🔬 내부 동작 원리

### 1. AddRequestHeaderGatewayFilterFactory — 요청 헤더 추가

```java
// AddRequestHeaderGatewayFilterFactory.java
public class AddRequestHeaderGatewayFilterFactory
        extends AbstractNameValueGatewayFilterFactory {

    @Override
    public GatewayFilter apply(NameValueConfig config) {
        // apply()가 GatewayFilter 람다를 반환
        return new GatewayFilter() {
            @Override
            public Mono<Void> filter(ServerWebExchange exchange,
                    GatewayFilterChain chain) {
                // ① ServerWebExchange.mutate()로 새 요청 생성
                //    (불변 객체이므로 mutate()로 복사본 생성)
                String value = ServerWebExchangeUtils
                    .expand(exchange, config.getValue());
                // SpEL 표현식 지원: {spring.application.name}

                ServerHttpRequest request = exchange.getRequest().mutate()
                    .header(config.getName(), value)
                    .build();  // 기존 헤더 + 새 헤더로 새 Request 생성

                // ② mutate된 exchange로 체인 계속 진행
                return chain.filter(exchange.mutate().request(request).build());
            }

            @Override
            public String toString() {
                return filterToStringCreator(AddRequestHeaderGatewayFilterFactory.this)
                    .append(config.getName(), config.getValue())
                    .toString();
            }
        };
    }
}

// 설정:
// filters:
//   - AddRequestHeader=X-User-Id, {X-JWT-Claim-Sub}
//   - AddRequestHeader=X-Service-Version, v2
```

### 2. RewritePathGatewayFilterFactory — 경로 재작성

```java
// RewritePathGatewayFilterFactory.java
public class RewritePathGatewayFilterFactory
        extends AbstractGatewayFilterFactory<RewritePathGatewayFilterFactory.Config> {

    @Override
    public GatewayFilter apply(Config config) {
        String replacement = config.getReplacement().replace("$\\", "$");

        return new GatewayFilter() {
            @Override
            public Mono<Void> filter(ServerWebExchange exchange,
                    GatewayFilterChain chain) {

                ServerHttpRequest req = exchange.getRequest();
                // 원래 경로를 속성에 저장 (후처리 참조용)
                addOriginalRequestUrl(exchange, req.getURI());

                String path = req.getURI().getRawPath();
                // ① 정규식으로 경로 변환
                // config.regexp = "^/api(?<segment>/?.*)"
                // config.replacement = "/${segment}"
                // /api/orders/1 → /orders/1
                String newPath = path.replaceAll(
                    config.getRegexp(), replacement);

                // ② 새 URI로 요청 재작성
                ServerHttpRequest request = req.mutate()
                    .path(newPath)
                    .build();

                // ③ GATEWAY_REQUEST_URL_ATTR: 실제 라우팅에 사용될 URL 저장
                exchange.getAttributes().put(
                    GATEWAY_REQUEST_URL_ATTR,
                    request.getURI()
                );

                return chain.filter(exchange.mutate().request(request).build());
            }
        };
    }
}

// YAML:
// filters:
//   - RewritePath=/api(?<segment>/?.*), /${segment}
// 효과: /api/orders/1 → /orders/1 (업스트림에 /api prefix 제거)
```

### 3. RetryGatewayFilterFactory — 재시도 로직

```java
// RetryGatewayFilterFactory.java
public class RetryGatewayFilterFactory
        extends AbstractGatewayFilterFactory<RetryGatewayFilterFactory.RetryConfig> {

    @Override
    public GatewayFilter apply(RetryConfig config) {
        config.validate();

        return new GatewayFilter() {
            @Override
            public Mono<Void> filter(ServerWebExchange exchange,
                    GatewayFilterChain chain) {

                // Retry 횟수 추적을 위한 원자적 카운터
                AtomicInteger iteration = new AtomicInteger();

                return chain.filter(exchange)
                    // ① 업스트림 호출 결과 확인
                    .then(Mono.defer(() -> {
                        HttpStatus statusCode = exchange.getResponse().getStatusCode();

                        // ② 재시도 조건 확인: HTTP 상태 코드 기반
                        boolean shouldRetry = config.getStatuses()
                            .contains(statusCode);
                        // 또는 상태 코드 시리즈: 5xx 전체
                        if (!shouldRetry) {
                            shouldRetry = config.getSeries().stream()
                                .anyMatch(series ->
                                    series.equals(HttpStatus.Series.resolve(
                                        statusCode.value())));
                        }

                        if (shouldRetry
                                && iteration.incrementAndGet() <= config.getRetries()) {
                            // ③ 재시도: exchange의 Response를 초기화하고 재실행
                            exchange.getResponse().setStatusCode(null);
                            return filter(exchange, chain);  // 재귀
                        }
                        return Mono.empty();  // 재시도 없음
                    }))
                    // ④ 예외 기반 재시도 (연결 실패 등)
                    .retryWhen(Retry.max(config.getRetries())
                        .filter(throwable -> {
                            // 재시도할 예외 타입 확인
                            return config.getExceptions().stream()
                                .anyMatch(c -> c.isInstance(throwable));
                        }));
            }
        };
    }
}

// YAML:
// filters:
//   - name: Retry
//     args:
//       retries: 3
//       statuses: BAD_GATEWAY, SERVICE_UNAVAILABLE
//       methods: GET      # 멱등성: GET만 재시도 (POST 제외)
//       backoff:
//         firstBackoff: 100ms
//         maxBackoff: 500ms
//         factor: 2       # 지수 백오프
//         basedOnPreviousValue: false
```

### 4. RemoveRequestHeaderGatewayFilterFactory + AddResponseHeaderGatewayFilterFactory

```java
// ① 요청 헤더 제거 (내부 정보 업스트림에 전달 방지)
// filters:
//   - RemoveRequestHeader=X-Internal-Token

// ② 응답 헤더 추가 (CORS, 보안 헤더 등)
// filters:
//   - AddResponseHeader=X-Content-Type-Options, nosniff
//   - AddResponseHeader=X-Frame-Options, DENY

// 응답 헤더 수정은 Post-filter에서 실행:
@Override
public GatewayFilter apply(NameValueConfig config) {
    return (exchange, chain) -> chain.filter(exchange)
        .then(Mono.fromRunnable(() -> {
            // chain.filter() 완료 후 실행
            // 응답 헤더는 커밋 전에 추가해야 함
            exchange.getResponse().getHeaders()
                .add(config.getName(), config.getValue());
        }));
}
```

### 5. StripPrefixGatewayFilterFactory

```java
// StripPrefixGatewayFilterFactory
// /api/v1/orders → /orders (2개 prefix 제거)

@Override
public GatewayFilter apply(Config config) {
    return (exchange, chain) -> {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getURI().getRawPath();
        String[] parts = path.split("/");

        // config.getParts() = 제거할 prefix 세그먼트 수
        StringBuilder newPath = new StringBuilder("/");
        for (int i = 0; i < parts.length; i++) {
            if (i == 0 || (i <= config.getParts() && parts[i].isEmpty())) {
                continue;
            }
            if (i <= config.getParts()) {
                continue;  // 제거할 세그먼트 건너뜀
            }
            if (newPath.length() > 1) {
                newPath.append("/");
            }
            newPath.append(parts[i]);
        }

        ServerHttpRequest newRequest = request.mutate().path(newPath.toString()).build();
        exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, newRequest.getURI());
        return chain.filter(exchange.mutate().request(newRequest).build());
    };
}

// YAML:
// filters:
//   - StripPrefix=2
// /api/v1/orders → /orders
```

---

## 💻 실전 구성

### 실전 Route Filter 조합 예시

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            # 1. 경로 재작성: /api/orders → /orders
            - RewritePath=/api(?<segment>/?.*), /${segment}

            # 2. 인증 정보 헤더로 전달 (JWT 파싱 결과)
            - AddRequestHeader=X-User-Id, ${X-JWT-Sub}

            # 3. 내부 헤더 제거
            - RemoveRequestHeader=Authorization

            # 4. 보안 응답 헤더 추가
            - AddResponseHeader=X-Content-Type-Options, nosniff

            # 5. 재시도 설정
            - name: Retry
              args:
                retries: 3
                statuses: BAD_GATEWAY
                methods: GET

            # 6. 요청 크기 제한
            - name: RequestSize
              args:
                maxSize: 5MB

        - id: legacy-route
          uri: lb://legacy-service
          predicates:
            - Path=/old/api/**
          filters:
            # 구 경로 → 새 경로로 리다이렉트
            - RedirectTo=301, /api/
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: RewritePath + AddRequestHeader 조합으로 API 버전 마이그레이션

```
상황: 클라이언트는 /api/v1/orders 계속 사용
     내부적으로 order-service는 /orders로 경로 변경

Gateway 설정:
  - RewritePath=/api/v1(?<segment>/?.*), /${segment}
    /api/v1/orders/1 → /orders/1
    
  - AddRequestHeader=X-API-Version, v1
    업스트림에서 "이 요청이 v1 클라이언트에서 왔구나" 인식 가능

마이그레이션 완료 후:
  클라이언트 코드 변경 없이
  Gateway 필터만 교체하면 됨 (중앙 관리 효과)
```

### 시나리오: Retry + Circuit Breaker 조합

```yaml
filters:
  - name: Retry
    args:
      retries: 2
      statuses: BAD_GATEWAY, SERVICE_UNAVAILABLE
      methods: GET
      backoff:
        firstBackoff: 100ms
        maxBackoff: 1000ms
        factor: 2

  - name: CircuitBreaker
    args:
      name: order-circuit-breaker
      fallbackUri: forward:/fallback/orders

# Retry 먼저 → 재시도 모두 실패 → Circuit Breaker OPEN → Fallback
# 순서: Retry Filter(@Order: N) → CircuitBreaker Filter(@Order: N+1)
```

---

## ⚖️ 트레이드오프

| Filter 종류 | 실행 위치 | 성능 영향 | 주의사항 |
|------------|----------|----------|---------|
| **RewritePath** | Pre | 없음 | 정규식 복잡도 주의 |
| **AddRequestHeader** | Pre | 없음 | 민감 정보 로그 노출 주의 |
| **Retry** | Pre+Post | 있음 (재시도 시간) | 멱등성 있는 메서드만 |
| **RequestSize** | Pre | 없음 | 최대 크기 명확히 설정 |
| **AddResponseHeader** | Post | 없음 | 커밋 전에 추가해야 함 |

```
exchange.mutate() 비용:
  ServerWebExchange.mutate()는 얕은 복사(shallow copy)
  큰 비용 없음 — 참조만 복사
  단, 잦은 mutate() 연쇄 호출은 피하는 것이 코드 가독성에 좋음

Retry와 비멱등성:
  POST /payments → 재시도 시 결제 중복 발생 가능
  → Retry에서 POST 제외 (methods: GET,HEAD 등 멱등 메서드만)
  → 또는 업스트림이 멱등성 보장 시 포함 가능 (idempotency key 활용)
```

---

## 📌 핵심 정리

```
GatewayFilterFactory 핵심:

구조:
  GatewayFilterFactory<Config>.apply(Config)
  → GatewayFilter 람다 반환 (실제 필터 로직)
  → (exchange, chain) → Mono<Void>

exchange.mutate() 패턴 (불변 객체 수정):
  exchange.getRequest().mutate()
    .header(name, value)
    .path(newPath)
    .build()
  → exchange.mutate().request(newRequest).build()
  → chain.filter(newExchange)

Pre-filter vs Post-filter:
  Pre: chain.filter(exchange) 전 실행
       요청 변환, 헤더 추가/제거, 경로 재작성
  Post: chain.filter(exchange).then(...) 후 실행
        응답 헤더 추가, 로깅, 메트릭 수집

주요 내장 필터:
  RewritePath: 정규식으로 경로 재작성
  AddRequestHeader/AddResponseHeader: 헤더 추가
  StripPrefix: 경로 prefix 제거
  Retry: 실패 시 재시도 (멱등 메서드만)
  RequestSize: 요청 크기 제한
  CircuitBreaker: 장애 격리 (Ch3-06)
```

---

## 🤔 생각해볼 문제

**Q1.** `AddRequestHeader` 필터와 `AddResponseHeader` 필터의 `exchange.mutate()` 사용 방식이 다르다. 응답 헤더는 왜 `exchange.mutate()`를 쓰지 않고 직접 `exchange.getResponse().getHeaders().add()`를 쓰는가?

<details>
<summary>해설 보기</summary>

`ServerHttpResponse`는 `ServerHttpRequest`와 달리 응답이 전송(commit)되기 전까지 헤더를 직접 추가할 수 있습니다. 응답 헤더는 응답 본문이 쓰이기 시작할 때 자동으로 flush됩니다. `exchange.mutate()`로 새 `ServerHttpResponse`를 만드는 것도 가능하지만(`exchange.mutate().response(newResponse).build()`), 응답 헤더 단순 추가는 직접 접근이 더 간단합니다. 반면 `ServerHttpRequest`는 완전히 불변이어서 `mutate()`를 통한 복사본 생성이 유일한 방법입니다. 응답 헤더를 추가할 때는 반드시 `chain.filter(exchange).then(...)` 이후, 즉 Post-filter에서 실행해야 합니다. `then()` 이전에 추가하면 응답이 아직 없으므로 효과가 없을 수 있습니다.

</details>

---

**Q2.** `RewritePathGatewayFilterFactory`가 경로를 재작성한 후 `GATEWAY_REQUEST_URL_ATTR`을 설정하는 이유는 무엇인가? 이를 설정하지 않으면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`GATEWAY_REQUEST_URL_ATTR`은 `NettyRoutingFilter`가 실제로 업스트림에 요청을 보낼 때 사용할 URI를 담는 속성입니다. `NettyRoutingFilter`는 `exchange.getRequest().getURI()`가 아니라 `exchange.getAttribute(GATEWAY_REQUEST_URL_ATTR)`을 참조하여 요청 URI를 결정합니다. 만약 이 속성을 설정하지 않으면 `NettyRoutingFilter`가 원래 요청 URI를 그대로 사용하여 경로 재작성이 무시됩니다. `LoadBalancerClientFilter`도 이 속성을 업데이트하여 `lb://service-name`을 실제 IP:Port로 치환한 결과를 저장합니다. 필터 체인에서 이 속성이 올바르게 관리되어야 최종적으로 올바른 업스트림 URL로 요청이 전달됩니다.

</details>

---

**Q3.** `Retry` 필터를 POST 요청에 적용하면 안 된다고 했는데, Gateway 레벨에서 POST 재시도를 안전하게 구현하려면 어떤 조건이 충족되어야 하는가?

<details>
<summary>해설 보기</summary>

POST 재시도를 안전하게 하려면 업스트림 서비스가 **멱등성(Idempotency)**을 보장해야 합니다. 구현 방법:

1. **Idempotency Key 사용**: 클라이언트가 요청에 `Idempotency-Key: uuid` 헤더를 포함하면, 업스트림 서비스가 같은 키로 들어오는 중복 요청을 감지해 첫 번째 처리 결과만 반환합니다.

2. **Gateway에서 키 주입**: Gateway 필터에서 `Idempotency-Key`가 없는 경우 UUID를 생성해 헤더로 추가합니다. 재시도 시 같은 키가 유지되어 업스트림에서 중복 처리를 막을 수 있습니다.

3. **업스트림 서비스 구현**: 처리된 Idempotency-Key를 Redis나 DB에 저장하고, 같은 키가 오면 저장된 응답을 반환합니다.

이 조건이 충족되지 않는다면 POST 재시도는 결제 중복, 주문 중복 등 심각한 비즈니스 오류를 야기할 수 있으므로 반드시 멱등성이 보장된 후에만 활성화해야 합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Route Predicate Factory](./02-route-predicate-factory.md)** | **[홈으로 🏠](../README.md)** | **[다음: Global Filter 체인 실행 순서 ➡️](./04-global-filter-chain.md)**

</div>
