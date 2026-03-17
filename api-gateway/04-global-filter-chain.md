# Global Filter 체인 실행 순서 — FilteringWebHandler의 필터 정렬·병합 메커니즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `FilteringWebHandler`가 Global Filter와 Route Filter를 병합하는 정확한 과정은?
- `@Order` 값에 따른 필터 실행 순서 규칙과 음수 Order의 의미는?
- `NettyRoutingFilter`와 `NettyWriteResponseFilter`의 기본 Order 값과 이 값이 중요한 이유는?
- Pre-filter와 Post-filter의 실행 순서가 반대인 이유는? Spring MVC Interceptor와의 비교는?
- `GatewayFilter`로 변환된 `GlobalFilter`의 Order는 어떻게 결정되는가?
- 내장 Global Filter들의 기본 실행 순서와 각각의 역할은?

---

## 🔍 왜 MSA에서 필요한가

### 필터 실행 순서가 왜 중요한가

```
잘못된 순서의 치명적 예시:

시나리오: JWT 인증 필터 + RewritePath 필터 + 라우팅 필터

순서 A (잘못):
  ① RewritePath: /api/orders → /orders
  ② JWT 인증: Authorization 헤더 파싱
  ③ NettyRoutingFilter: 업스트림으로 전송
  ④ NettyWriteResponseFilter: 응답 클라이언트에 전달

  문제 없어 보임. 하지만:

순서 B (더 잘못):
  ① NettyWriteResponseFilter: 응답 준비 (Post-filter)
  ② JWT 인증: 인증 실패 → 401 반환 시도
     근데 이미 응답 시작됨 → 헤더 추가 불가

  → 인증 로직이 응답 쓰기 이후에 실행되어 완전 무력화

올바른 순서:
  Pre-filter (높은 우선순위 → 낮은 우선순위 순):
    ① JWT 인증 (order: -100) → 실패 시 즉시 401
    ② RewritePath (order: -1)
    ③ NettyRoutingFilter (order: Integer.MIN_VALUE+1 = -2147483647) → 실제 전송

  Post-filter (낮은 우선순위 → 높은 우선순위 역순):
    ④ NettyWriteResponseFilter (order: -1) → 응답 클라이언트 전달
    ⑤ 로깅/메트릭 (order: 0)

  체인 구조 자체가 Pre/Post 실행 순서를 보장
```

---

## 😱 잘못된 구성

### Before: Order를 설정하지 않은 Custom Filter

```java
// ❌ @Order 없는 Custom GlobalFilter
@Component
public class AuthFilter implements GlobalFilter {  // Ordered 미구현!
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // Order 없음 → 기본값: Ordered.LOWEST_PRECEDENCE (Integer.MAX_VALUE)
        // 내장 필터들보다 뒤에서 실행됨
        // NettyRoutingFilter 이후에 실행될 수 있음
        // → 이미 요청이 업스트림으로 전송된 후에 인증 체크
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (token == null) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();  // 이미 늦음
        }
        return chain.filter(exchange);
    }
}
```

### Before: NettyRoutingFilter 이후에 요청을 수정하려는 시도

```java
// ❌ 라우팅 필터 이후에 요청 헤더 수정 시도
@Component
@Order(Integer.MIN_VALUE + 2)  // NettyRoutingFilter보다 나중에 실행
public class LateHeaderFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // NettyRoutingFilter: order = Integer.MIN_VALUE + 1
        // 이 필터: order = Integer.MIN_VALUE + 2 (나중에 실행)
        // → chain.filter()를 호출하면 NettyRoutingFilter가 아직 실행 안 됨
        
        // ❌ 요청 헤더 수정
        ServerHttpRequest req = exchange.getRequest().mutate()
            .header("X-Too-Late", "this-wont-work")
            .build();
        return chain.filter(exchange.mutate().request(req).build());
        // NettyRoutingFilter가 GATEWAY_REQUEST_URL_ATTR을 보고 요청을 이미 보냄 →
        // 실제로는 아직 안 보냈지만, 순서 개념 상 이미 늦은 것
    }
}
```

---

## ✨ 올바른 패턴

### After: 내장 Global Filter의 실행 순서 전체 맵

```
Order    Filter                              역할
─────────────────────────────────────────────────────────────────
-2147483647  NettyWriteResponseFilter       업스트림 응답 → 클라이언트 전달 (Post 전용)
-2147483646  WebsocketRoutingFilter         WebSocket 프록시 라우팅
-2147483645  NettyRoutingFilter             HTTP 업스트림 요청 전송
-2147483644  ForwardRoutingFilter           로컬 포워드 (forward:// URI)

...중간: 커스텀 Global Filter, Route Filter 자리...

-1           GatewayMetricsFilter           Actuator 메트릭 수집
0            AdaptCachedBodyGlobalFilter    요청 본문 캐시 (Retry용)
0            ForwardPathFilter              forward:// 경로 처리
1            RouteToRequestUrlFilter        lb:// → 실제 URL 변환
             LoadBalancerClientFilter       Eureka 인스턴스 선택 (LoadBalancer용)

Pre-filter 실행: 낮은 Order → 높은 Order
Post-filter 실행: 높은 Order → 낮은 Order (체인 역방향)
```

---

## 🔬 내부 동작 원리

### 1. FilteringWebHandler — Global + Route Filter 병합

```java
// FilteringWebHandler.java
public class FilteringWebHandler implements WebHandler {

    private final List<GatewayFilter> globalFilters;  // Global Filter 목록 (미리 변환됨)

    public FilteringWebHandler(List<GlobalFilter> globalFilters) {
        // ① GlobalFilter → GatewayFilter 변환 (Order 유지)
        this.globalFilters = loadFilters(globalFilters);
    }

    private static List<GatewayFilter> loadFilters(List<GlobalFilter> filters) {
        return filters.stream()
            .map(filter -> {
                // GlobalFilter를 GatewayFilter로 어댑터 변환
                GatewayFilterAdapter adapted = new GatewayFilterAdapter(filter);
                if (filter instanceof Ordered) {
                    int order = ((Ordered) filter).getOrder();
                    return new OrderedGatewayFilter(adapted, order);
                    // Order 유지
                }
                return adapted;
            })
            .collect(Collectors.toList());
    }

    @Override
    public Mono<Void> handle(ServerWebExchange exchange) {
        // ② 매칭된 Route 가져오기
        Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
        List<GatewayFilter> gatewayFilters = route.getFilters();

        // ③ Global Filter + Route Filter 병합
        List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
        combined.addAll(gatewayFilters);

        // ④ Order 기준으로 정렬
        AnnotationAwareOrderComparator.sort(combined);
        // → Order 값 오름차순: 작을수록 먼저 실행

        if (logger.isDebugEnabled()) {
            logger.debug("Sorted filters: " + combined);
        }

        // ⑤ 정렬된 필터 체인 실행
        return new DefaultGatewayFilterChain(combined).filter(exchange);
    }
}
```

### 2. Pre-filter vs Post-filter 실행 역전 원리

```java
// 필터 체인이 어떻게 Pre / Post를 구분하는가

// 필터 A (Order: 1), 필터 B (Order: 2), 필터 C (Order: 3)
// 정렬 후: A → B → C

// DefaultGatewayFilterChain 실행:
// A.filter(exchange, chainBC) 호출
//   chain.filter(exchange) 이전: A의 Pre 로직 실행
//   chain.filter(exchange) 호출 → B.filter(exchange, chainC) 호출
//     chain.filter(exchange) 이전: B의 Pre 로직
//     chain.filter(exchange) 호출 → C.filter(exchange, emptyChain) 호출
//       chain.filter(exchange) 이전: C의 Pre 로직 (NettyRoutingFilter)
//       → 업스트림에 요청 전송 (비동기)
//       응답 도착 후 Mono 완료
//     C의 Post 로직 (없음)
//   B의 Post 로직 (chain.filter().then() 이후 코드)
// A의 Post 로직

// 실행 순서:
// Pre:  A(pre) → B(pre) → C(pre) → [업스트림 호출]
// Post: C(post) → B(post) → A(post)
// Post는 Pre의 역순!

// 코드로 표현:
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    // === PRE 로직 (chain.filter 이전) ===
    log.info("Pre: Order=1");

    return chain.filter(exchange)  // 다음 필터 호출
        .then(Mono.fromRunnable(() -> {
            // === POST 로직 (chain.filter 완료 후) ===
            log.info("Post: Order=1");
            // 이 시점에 업스트림 응답이 exchange에 저장됨
        }));
}
```

### 3. NettyRoutingFilter와 NettyWriteResponseFilter의 Order

```java
// NettyRoutingFilter.java
// 업스트림으로 실제 HTTP 요청 전송
@Override
public int getOrder() {
    return props.getOrder();  // 기본값: Integer.MIN_VALUE + 1
}
// = -2147483647
// 거의 모든 커스텀 필터보다 나중에 (높은 Order) 실행
// Pre-filter 중 가장 마지막에 실행 → 그 전에 모든 요청 변환 완료

// NettyWriteResponseFilter.java
// 업스트림 응답을 클라이언트에 전달
@Override
public int getOrder() {
    return NettyWriteResponseFilter.WRITE_RESPONSE_FILTER_ORDER;
    // = Integer.MIN_VALUE (= -2147483648)
}
// 모든 필터 중 가장 낮은 Order
// Post-filter 중 가장 마지막에 실행 (역순이므로)
// → 모든 Post 로직이 완료된 후 응답 전송

// 실행 순서 시각화:
// Pre:  커스텀(-100) → GatewayMetrics(-1) → NettyRoutingFilter(MIN+1)
//       [업스트림 통신]
// Post: NettyWriteResponseFilter(MIN) ← 역순 → 커스텀(-100)의 Post
```

### 4. Route Filter의 Order 처리

```java
// Route Filter(GatewayFilterFactory로 생성된 필터)는
// 기본적으로 Ordered를 구현하지 않음
// → AnnotationAwareOrderComparator: Order 없는 것은 Ordered.LOWEST_PRECEDENCE로 처리
// → Route Filter는 Global Filter보다 나중에 실행될 수 있음

// OrderedGatewayFilter로 명시적 Order 지정
public class OrderedGatewayFilter implements GatewayFilter, Ordered {
    private final GatewayFilter delegate;
    private final int order;

    @Override
    public int getOrder() {
        return this.order;
    }
}

// YAML에서 Route Filter Order 지정:
// filters:
//   - name: AddRequestHeader
//     args:
//       name: X-User-Id
//       value: "123"
//     order: -50   # ← Route Filter에 Order 지정

// 또는 GatewayFilterFactory 구현 시 Ordered 인터페이스 구현
```

### 5. 내장 Global Filter 실행 순서 전체 흐름

```java
// ① RouteToRequestUrlFilter (Order: 10000)
// lb://order-service/orders/1 → http://10.0.1.5:8080/orders/1 (Eureka 조회)
// GATEWAY_REQUEST_URL_ATTR 갱신

// ② LoadBalancerClientFilter (Order: 10150)
// Eureka에서 인스턴스 선택, URI 확정

// ③ 커스텀 Global Filter (Order: 개발자 지정)
// JWT 인증, 로깅, 메트릭 등

// ④ Route Filter (Order: Ordered.LOWEST_PRECEDENCE)
// RewritePath, AddRequestHeader 등
// (명시적 Order 없으면 Global Filter보다 나중)

// ⑤ NettyRoutingFilter (Order: Integer.MIN_VALUE + 1)
// 업스트림으로 실제 HTTP 요청 전송
// 이 시점부터 요청 변환 불가

// --- 업스트림 응답 수신 ---

// ⑥ NettyWriteResponseFilter (Order: Integer.MIN_VALUE)
// 응답 바이트를 클라이언트 소켓으로 전달 시작
// 이 시점부터 응답 헤더 변경 불가
```

---

## 💻 실전 구성

### Custom Global Filter Order 설정 패턴

```java
// 인증 필터: 모든 필터 중 가장 먼저
@Component
@Order(-100)  // 또는 implements Ordered { return -100; }
public class JwtAuthGlobalFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (token == null || !isValid(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        // 인증 성공 → 다음 필터로
        return chain.filter(exchange);
    }
}

// 요청/응답 로깅 필터: 모든 처리 전후를 감쌀 것
@Component
@Order(-50)
public class LoggingGlobalFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime = System.currentTimeMillis();
        String path = exchange.getRequest().getPath().toString();
        log.info("→ {} {}", exchange.getRequest().getMethod(), path);

        return chain.filter(exchange)
            .doFinally(signalType -> {
                // Post: 응답 완료 후 (성공/오류 모두)
                long elapsed = System.currentTimeMillis() - startTime;
                HttpStatus status = exchange.getResponse().getStatusCode();
                log.info("← {} {} {}ms", status, path, elapsed);
            });
    }
}

// 메트릭 필터: 로깅보다 나중 (감싸는 구조로 더 바깥에서 시간 측정)
@Component
@Order(-30)
public class MetricsGlobalFilter implements GlobalFilter { ... }
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 잘못된 Order로 인한 인증 우회 버그

```
설정:
  JwtAuthFilter: Order = 10 (기본값 → Order 미지정)
  NettyRoutingFilter: Order = Integer.MIN_VALUE + 1

실행 순서:
  Pre:
    NettyRoutingFilter(MIN+1 = -2147483647)   ← 먼저 실행
    [업스트림으로 요청 전송]
    JwtAuthFilter(LOWEST = Integer.MAX_VALUE)  ← 나중에 실행
    → "401 반환" 시도 → 이미 요청이 업스트림으로 갔음

  결과:
    인증 없이 모든 요청이 업스트림에 도달
    클라이언트는 401을 받지만 업스트림에서 이미 처리됨

수정:
  JwtAuthFilter에 @Order(-100) 추가
  → JwtAuthFilter(-100) → NettyRoutingFilter(MIN+1)
  → 인증 실패 시 chain.filter() 호출 안 하므로 업스트림 미도달
```

---

## ⚖️ 트레이드오프

| Order 범위 | 용도 | 주의사항 |
|-----------|------|---------|
| `< -100` | 보안 인증, 핵심 Pre-filter | 매우 이른 실행, 다른 필터 정보 없음 |
| `-100 ~ -1` | 로깅, 메트릭, 헤더 조작 | 인증 이후, 라우팅 이전 |
| `0 ~ 100` | 일반 비즈니스 로직 | Route Filter와 혼용 |
| `Integer.MIN_VALUE+1` | NettyRoutingFilter (변경 금지) | 이후 요청 변환 불가 |
| `Integer.MIN_VALUE` | NettyWriteResponseFilter (변경 금지) | 이후 응답 변환 불가 |

```
Global Filter vs Route Filter 선택:
  Global Filter:
    모든 Route에 적용
    인증, 로깅, 트레이싱, CORS 등
    GlobalFilter 인터페이스 구현

  Route Filter (GatewayFilterFactory):
    특정 Route에만 적용
    경로 재작성, 헤더 변환, 재시도 등
    GatewayFilterFactory 구현 또는 YAML에서 기본 제공 필터 사용
```

---

## 📌 핵심 정리

```
Global Filter 체인 실행 순서 핵심:

병합:
  FilteringWebHandler:
    globalFilters(Global) + route.getFilters(Route) → 합치기
    AnnotationAwareOrderComparator.sort() → Order 오름차순 정렬

Pre-filter: Order 낮은 것부터
Post-filter: Pre와 반대 순서 (체인 역방향)

핵심 내장 필터 Order:
  NettyWriteResponseFilter: Integer.MIN_VALUE (-2147483648)
  NettyRoutingFilter: Integer.MIN_VALUE + 1 (-2147483647)
  (이 둘 사이에 끼어드는 커스텀 필터는 의미 없음)

커스텀 필터 Order 권장:
  인증/보안: -200 ~ -100 (가장 먼저)
  로깅: -100 ~ -50
  일반 변환: -50 ~ 0

Route Filter Order:
  Ordered 미구현 시: Integer.MAX_VALUE → 거의 모든 Global Filter보다 나중
  중요 Route Filter는 명시적 Order 지정 권장
```

---

## 🤔 생각해볼 문제

**Q1.** `chain.filter(exchange)` 이후에 `exchange.getResponse().getStatusCode()`를 읽을 수 있는가? 언제부터 읽을 수 있는가?

<details>
<summary>해설 보기</summary>

`chain.filter(exchange)` 이후 `.then()`이나 `.doFinally()` 등의 Post 콜백에서 읽을 수 있습니다. `NettyRoutingFilter`가 업스트림 응답을 받아 `exchange.getResponse()`에 상태 코드를 설정한 후 Mono가 완료됩니다. 그 이후의 Post 콜백에서 상태 코드를 읽을 수 있습니다. 단, `NettyWriteResponseFilter`가 실행되면서 응답이 클라이언트에게 전송되기 시작하면 그 이후에는 상태 코드를 변경할 수 없습니다(이미 커밋됨). 따라서 응답 상태 코드를 변경해야 하는 Post-filter는 반드시 `NettyWriteResponseFilter`(Order: MIN)보다 높은 Order(나중에 실행 → Pre 기준 낮은 Order)를 가져야 합니다.

</details>

---

**Q2.** Global Filter가 100개 Route 모두에 적용되는데, 특정 Route에서만 Global Filter를 제외하고 싶다. 어떻게 하는가?

<details>
<summary>해설 보기</summary>

Spring Cloud Gateway는 Global Filter를 특정 Route에서 제외하는 내장 메커니즘이 없습니다. 대안:

1. **Global Filter 내부에서 직접 처리**: Route ID나 경로를 확인하여 특정 Route는 로직을 건너뜁니다.
```java
String routeId = exchange.getAttribute(GATEWAY_ROUTE_ATTR).getId();
if ("public-route".equals(routeId)) {
    return chain.filter(exchange);  // 이 Route는 인증 건너뜀
}
```

2. **Exchange 속성 활용**: Route Filter에서 `exchange.getAttributes().put("skip-auth", true)`를 설정하고, Global Filter에서 이 속성을 확인합니다.

3. **Global Filter를 Route Filter로 변환**: 필요한 Route에만 명시적으로 필터 추가합니다(가장 명시적이고 권장되는 방법).

4. **Path 기반 예외 처리**: 특정 경로 패턴(`/public/**`)이면 인증 건너뜀.

</details>

---

**Q3.** `AnnotationAwareOrderComparator.sort()`로 정렬할 때, 두 필터가 동일한 Order 값을 가지면 어떤 순서로 실행되는가?

<details>
<summary>해설 보기</summary>

동일한 Order 값을 가진 경우 실행 순서는 비결정적입니다. `AnnotationAwareOrderComparator`는 동일 Order 시 추가 정렬 기준이 없으므로 리스트에서의 위치(삽입 순서)에 의존합니다. 하지만 이 순서는 Spring Bean 등록 순서에 따라 달라질 수 있어 재현하기 어렵습니다. 실무에서는 서로 의존 관계가 있는 필터끼리는 반드시 다른 Order 값을 사용해야 합니다. 독립적인 필터(예: 로깅과 메트릭)는 동일 Order를 가져도 실용적으로 문제없지만, 예측 가능성을 위해 명시적으로 구분하는 것을 권장합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Gateway Filter Factory](./03-gateway-filter-factory.md)** | **[홈으로 🏠](../README.md)** | **[다음: Route Locator 동적 라우팅 ➡️](./05-route-locator-dynamic-routing.md)**

</div>
