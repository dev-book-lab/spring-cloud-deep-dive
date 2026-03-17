# Route Predicate Factory — 요청을 어떤 라우트로 보낼지 결정하는 과정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RoutePredicateHandlerMapping`이 요청을 받아 적합한 Route를 찾는 과정은?
- `RoutePredicateFactory`의 `apply()` 메서드가 `AsyncPredicate<ServerWebExchange>`를 반환하는 이유는?
- Path, Method, Header, Query, Weight Predicate 각각의 매칭 로직 차이는?
- `Weight` Predicate는 트래픽 가중치를 어떻게 분배하는가? 내부 계산 방식은?
- 여러 Predicate를 `AND`/`OR` 조합할 때 `AsyncPredicate` 체인은 어떻게 동작하는가?
- Custom Predicate를 만들어 등록하는 방법은?

---

## 🔍 왜 MSA에서 필요한가

### Gateway가 요청을 올바른 서비스로 보내야 하는 이유

```
단일 Gateway가 수십 개 서비스를 라우팅:

  GET  /api/orders/**      → order-service
  GET  /api/products/**    → product-service
  POST /api/payments/**    → payment-service (POST만)
  GET  /api/v2/users/**    → user-service-v2 (v2 경로만)
  *    /api/**             → Header: X-Canary=true → canary 서비스
  *    /api/**             → 일반 서비스 (90%)

  각 조건을 매칭하는 로직:
    경로(Path) 매칭
    HTTP 메서드(Method) 매칭
    헤더(Header) 존재/값 매칭
    쿼리 파라미터(Query) 매칭
    가중치(Weight) 기반 트래픽 분배

  이 모든 조건을 Reactive 방식으로 비동기 평가해야 함
  → RoutePredicateFactory의 역할
```

---

## 😱 잘못된 구성

### Before: 너무 느슨한 Path Predicate로 라우트 충돌

```yaml
# ❌ 모호한 Path 순서로 인한 라우팅 오류
spring:
  cloud:
    gateway:
      routes:
        - id: general-route
          uri: lb://general-service
          predicates:
            - Path=/api/**          # 모든 /api/ 요청 매칭

        - id: order-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**   # order-route는 영원히 매칭 안 됨!
```

```
문제:
  Gateway는 Route 목록을 순서대로 평가
  general-route의 /api/**가 모든 요청을 먼저 가져감
  order-route는 평가 기회조차 없음

해결:
  구체적인 경로를 먼저, 일반적인 경로를 나중에
  또는 Order를 명시적으로 설정
```

### Before: Weight Predicate 그룹 이름 불일치

```yaml
# ❌ Weight Predicate 그룹 이름 다름
routes:
  - id: stable-v1
    predicates:
      - Weight=service-a, 90    # 그룹: "service-a"
  - id: canary-v2
    predicates:
      - Weight=service_a, 10   # 그룹: "service_a" (다름!)
      # → 두 라우트가 다른 가중치 그룹으로 처리됨
      # → service-a: 100%, service_a: 100% 각자 독립
      # → 모든 요청이 두 라우트 모두에 매칭될 수 있음
```

---

## ✨ 올바른 패턴

### After: Predicate 조합 올바른 설정

```yaml
spring:
  cloud:
    gateway:
      routes:
        # 구체적인 라우트 먼저
        - id: order-route
          uri: lb://order-service
          order: 1                    # 낮을수록 높은 우선순위
          predicates:
            - Path=/api/orders/**
            - Method=GET,POST         # AND 조건

        # 카나리 라우팅 (헤더 기반)
        - id: user-service-canary
          uri: lb://user-service-v2
          order: 2
          predicates:
            - Path=/api/users/**
            - Header=X-Canary, true   # X-Canary: true 헤더 있을 때만

        # 가중치 기반 분배
        - id: product-v1
          uri: lb://product-service
          predicates:
            - Path=/api/products/**
            - Weight=product-group, 90  # 90% 트래픽

        - id: product-v2
          uri: lb://product-service-v2
          predicates:
            - Path=/api/products/**
            - Weight=product-group, 10  # 10% 트래픽

        # 폴백: 나머지 모든 요청
        - id: default-route
          uri: lb://default-service
          order: 9999
          predicates:
            - Path=/api/**
```

---

## 🔬 내부 동작 원리

### 1. RoutePredicateHandlerMapping — Route 탐색

```java
// RoutePredicateHandlerMapping.java
// WebFlux의 HandlerMapping 구현 → 요청에 맞는 Route 찾기

public class RoutePredicateHandlerMapping extends AbstractHandlerMapping {

    private final FilteringWebHandler webHandler;
    private final RouteLocator routeLocator;

    @Override
    protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
        // Exchange에 Gateway Handler 속성이 이미 있으면 스킵 (재진입 방지)
        if (this.managementPortType == DIFFERENT
                && this.managementPort != null
                && exchange.getRequest().getURI().getPort() == this.managementPort) {
            return Mono.empty();
        }

        exchange.getAttributes().put(GATEWAY_HANDLER_MAPPER_ATTR, getSimpleName());

        return lookupRoute(exchange)
            .flatMap(r -> {
                exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
                exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);
                // 매칭된 Route를 Exchange 속성에 저장 → FilteringWebHandler에서 사용
                return Mono.just(webHandler);
            })
            .switchIfEmpty(Mono.empty().then(Mono.fromRunnable(() -> {
                // 매칭되는 Route 없음 → 404
                exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
            })));
    }

    protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
        return this.routeLocator
            .getRoutes()  // Flux<Route>: 모든 Route 스트림
            .concatMap(route -> Mono.just(route)
                .filterWhen(r -> {
                    exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, r.getId());
                    // ① 각 Route의 Predicate 평가
                    return r.getPredicate().apply(exchange);
                })
            )
            .next()  // 첫 번째 매칭 Route 선택 (순서 중요!)
            .map(route -> {
                validateCookieNameKey(route);
                return route;
            });
    }
}
```

### 2. AsyncPredicate — Reactive Predicate 래퍼

```java
// AsyncPredicate.java
// 표준 Predicate<T>의 Reactive 버전
// Predicate<T>.test() → boolean (블로킹)
// AsyncPredicate<T>.apply() → Publisher<Boolean> (Non-Blocking)

public interface AsyncPredicate<T> {
    Publisher<Boolean> apply(T t);

    // AND 조합
    default AsyncPredicate<T> and(AsyncPredicate<? super T> other) {
        return t -> Flux.zip(
            Flux.from(apply(t)),
            Flux.from(other.apply(t))
        ).map(tuple -> tuple.getT1() && tuple.getT2());
        // 두 Predicate를 병렬 평가 후 AND
    }

    // OR 조합
    default AsyncPredicate<T> or(AsyncPredicate<? super T> other) {
        return t -> Flux.zip(
            Flux.from(apply(t)),
            Flux.from(other.apply(t))
        ).map(tuple -> tuple.getT1() || tuple.getT2());
    }

    // 동기 Predicate → AsyncPredicate 변환 (Static 팩토리)
    static <T> AsyncPredicate<T> from(Predicate<T> predicate) {
        return t -> Mono.just(predicate.test(t));
    }
}
```

### 3. PathRoutePredicateFactory — 경로 매칭

```java
// PathRoutePredicateFactory.java
public class PathRoutePredicateFactory
        extends AbstractRoutePredicateFactory<PathRoutePredicateFactory.Config> {

    private PathPatternParser pathPatternParser = new PathPatternParser();

    @Override
    public AsyncPredicate<ServerWebExchange> applyAsync(Config config) {
        // YAML의 patterns를 PathPattern으로 컴파일
        List<PathPattern> pathPatterns = config.getPatterns().stream()
            .map(pattern -> {
                pathPatternParser.setMatchOptionalTrailingSeparator(
                    config.isMatchTrailingSlash());
                return pathPatternParser.parse(pattern);
            })
            .collect(Collectors.toList());

        return new AsyncPredicate<ServerWebExchange>() {
            @Override
            public Publisher<Boolean> apply(ServerWebExchange exchange) {
                // 요청 경로 추출
                PathContainer path = parsePath(
                    exchange.getRequest().getURI().getRawPath());

                // ① 모든 PathPattern 중 하나라도 매칭되면 true
                Optional<PathPattern> optionalPathPattern = pathPatterns.stream()
                    .filter(pattern -> pattern.matches(path))
                    .findFirst();

                if (optionalPathPattern.isPresent()) {
                    PathPattern pathPattern = optionalPathPattern.get();
                    // ② 매칭 결과(URI 변수 등)를 Exchange 속성에 저장
                    //    → PathVariable 추출, 경로 재작성에 활용
                    traceMatch("Pattern", pathPattern.getPatternString(), path, true);
                    PathMatchInfo pathMatchInfo = pathPattern.matchAndExtract(path);
                    putUriTemplateVariables(exchange, pathMatchInfo.getUriVariables());
                    return Mono.just(Boolean.TRUE);
                }
                return Mono.just(Boolean.FALSE);
            }
        };
    }
}
```

### 4. WeightRoutePredicateFactory — 가중치 기반 트래픽 분배

```java
// WeightRoutePredicateFactory.java
// Weight=group, weight → 같은 group 내에서 weight 비율로 트래픽 분배

public class WeightRoutePredicateFactory
        extends AbstractRoutePredicateFactory<WeightRoutePredicateFactory.Config> {

    private final WeightCalculatorWebFilter calculator;

    @Override
    public AsyncPredicate<ServerWebExchange> applyAsync(Config config) {
        return exchange -> {
            // WeightCalculatorWebFilter가 미리 계산해둔 그룹별 선택 Route를 읽음
            Map<String, String> weights = exchange.getAttribute(WEIGHT_ATTR);
            if (weights != null) {
                // 현재 요청의 그룹에서 이 Route가 선택됐는지 확인
                String selectedRouteId = weights.get(config.getGroup());
                boolean matches = exchange.getAttribute(GATEWAY_ROUTE_ATTR) != null
                    && exchange.getAttribute(GATEWAY_ROUTE_ATTR).equals(config.getGroup());
                // 미리 선택된 Route ID와 일치하면 true
                return Mono.just(config.getGroup().equals(selectedRouteId));
            }
            return Mono.just(false);
        };
    }
}

// WeightCalculatorWebFilter: 모든 요청 전에 실행
// 그룹별 가중치 합산 → 랜덤값으로 어느 Route로 갈지 미리 결정
public class WeightCalculatorWebFilter implements WebFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        Map<String, List<WeightConfig>> groupWeights = ...; // 그룹별 가중치 목록

        // 각 그룹에서 어느 Route로 보낼지 결정
        Map<String, String> weights = new HashMap<>();
        for (Map.Entry<String, List<WeightConfig>> entry : groupWeights.entrySet()) {
            String group = entry.getKey();
            List<WeightConfig> configs = entry.getValue();

            // 누적 가중치: [0, w1), [w1, w1+w2), ...
            double random = this.random.nextDouble(); // 0.0 ~ 1.0
            double total = configs.stream().mapToInt(WeightConfig::getWeight).sum();
            double cumulative = 0;
            for (WeightConfig config : configs) {
                cumulative += (double) config.getWeight() / total;
                if (random < cumulative) {
                    weights.put(group, config.getRouteId());
                    break;
                }
            }
        }

        // Exchange 속성에 저장 → Predicate 평가 시 참조
        exchange.getAttributes().put(WEIGHT_ATTR, weights);
        return chain.filter(exchange);
    }
}
```

### 5. 주요 Predicate 비교

```java
// ① HeaderRoutePredicateFactory
// - Header=X-Request-Id, \d+  → X-Request-Id 헤더가 숫자이면 매칭
predicates:
  - Header=X-Request-Id, \d+   # 정규식 지원

// ② QueryRoutePredicateFactory
// - Query=color, green  → ?color=green 쿼리 파라미터
predicates:
  - Query=color, gre.n   # 정규식: green, grebn 등 매칭

// ③ MethodRoutePredicateFactory
predicates:
  - Method=GET,POST   # GET 또는 POST만

// ④ AfterRoutePredicateFactory
// - After=2024-01-01T00:00:00+09:00[Asia/Seoul]
// → 특정 시각 이후에만 라우팅 (배포 예약, 이벤트 오픈)
predicates:
  - After=2024-12-25T09:00:00+09:00[Asia/Seoul]

// ⑤ RemoteAddrRoutePredicateFactory
// - RemoteAddr=192.168.1.0/24  → 특정 IP 대역만
predicates:
  - RemoteAddr=10.0.0.0/8   # 내부망에서만 접근 가능
```

### 6. Custom Predicate Factory 구현

```java
// Custom Predicate: 특정 쿠키 값으로 라우팅
@Component
public class HasRoleRoutePredicateFactory
        extends AbstractRoutePredicateFactory<HasRoleRoutePredicateFactory.Config> {

    public HasRoleRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return List.of("role");  // YAML shortcut: HasRole=ADMIN
    }

    @Override
    public AsyncPredicate<ServerWebExchange> applyAsync(Config config) {
        return AsyncPredicate.from(exchange -> {
            // JWT 클레임에서 role 확인 (Spring Security Principal)
            return exchange.getPrincipal()
                .cast(JwtAuthenticationToken.class)
                .map(token -> token.getAuthorities().stream()
                    .anyMatch(a -> a.getAuthority().equals("ROLE_" + config.getRole())))
                .defaultIfEmpty(false)
                .block();  // ⚠️ 실제로는 Reactive하게 처리해야 함
        });
    }

    @Data
    public static class Config {
        private String role;
    }
}

// YAML에서 사용:
// predicates:
//   - HasRole=ADMIN
```

---

## 💻 실전 구성

### 전체 라우팅 전략 예시

```yaml
spring:
  cloud:
    gateway:
      routes:
        # 1. 관리자 API: 내부 IP + ADMIN 역할
        - id: admin-route
          uri: lb://admin-service
          order: 1
          predicates:
            - Path=/admin/**
            - RemoteAddr=10.0.0.0/8    # 내부망만

        # 2. 카나리 배포: 헤더 기반
        - id: order-canary
          uri: lb://order-service-v2
          order: 2
          predicates:
            - Path=/api/orders/**
            - Header=X-Canary, true

        # 3. 가중치 기반 A/B 테스트
        - id: payment-stable
          uri: lb://payment-service
          order: 3
          predicates:
            - Path=/api/payments/**
            - Weight=payment, 95

        - id: payment-new
          uri: lb://payment-service-v2
          order: 3
          predicates:
            - Path=/api/payments/**
            - Weight=payment, 5

        # 4. 이벤트 기간 한정 라우팅
        - id: event-route
          uri: lb://event-service
          order: 2
          predicates:
            - Path=/api/event/**
            - Between=2024-12-24T00:00:00+09:00[Asia/Seoul],2024-12-26T00:00:00+09:00[Asia/Seoul]
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: Weight Predicate로 카나리 배포 트래픽 점진적 증가

```bash
# 초기: 카나리 5%
# predicates: Weight=service-group, 95  (stable)
# predicates: Weight=service-group, 5   (canary)

# 검증 완료 후 Config Server에서 설정 변경 → /actuator/busrefresh
# → RefreshRoutesEvent 발행 → 라우트 갱신

# 단계별 트래픽 증가:
# 5% → 10% → 25% → 50% → 100% (stable 완전 교체)

# 각 단계에서 모니터링:
# - 에러율 증가 없음 확인
# - p99 레이턴시 이상 없음 확인
# - 비즈니스 메트릭 이상 없음 확인
```

### 시나리오: 복합 Predicate AND 조건 디버깅

```bash
# 요청이 route에 매칭이 안 될 때 디버그
logging:
  level:
    org.springframework.cloud.gateway: TRACE

# 로그 예시:
# [TRACE] Route [order-route] not matched: Path /api/products/ does not match /api/orders/**
# [TRACE] Route [product-route] matched: Path /api/products/ matches /api/products/**
# [TRACE] Route [product-route]: Method GET in [GET, POST]
# [TRACE] Route [product-route] matched all predicates, selected.
```

---

## ⚖️ 트레이드오프

| Predicate 방식 | 유연성 | 성능 | 적합한 케이스 |
|---------------|--------|------|--------------|
| **Path** | 낮음 | 최고 | 기본 서비스 라우팅 |
| **Header** | 높음 | 높음 | 카나리, 버전 라우팅 |
| **Weight** | 중간 | 높음 | A/B 테스트, 점진적 배포 |
| **RemoteAddr** | 낮음 | 높음 | 보안 제어 (내부망만) |
| **Custom** | 최고 | 의존 | 도메인 특화 로직 |

```
Predicate 개수와 성능:
  모든 Predicate는 매 요청마다 평가됨
  Predicate 수가 많을수록 CPU 사용량 증가
  단, 대부분 메모리 내 연산이므로 실제 영향 미미

  AsyncPredicate AND 조합:
  Flux.zip()으로 병렬 평가 → 순차 평가보다 빠름
  
  가장 비용 높은 Predicate:
  DB 조회나 외부 서비스 호출 포함 → 비동기로 처리해야 함
  Custom Predicate에서 블로킹 금지
```

---

## 📌 핵심 정리

```
Route Predicate Factory 핵심:

Route 매칭 과정:
  RoutePredicateHandlerMapping.lookupRoute()
  → Flux<Route>.concatMap(route → predicate.apply(exchange))
  → 첫 번째 true인 Route 선택 (순서 중요!)

AsyncPredicate:
  표준 Predicate의 Reactive 버전
  apply() → Publisher<Boolean>
  AND/OR 조합 가능 (Flux.zip으로 병렬 평가)

주요 Predicate:
  Path: PathPatternParser로 URI 패턴 매칭
  Method: HTTP 메서드 목록 비교
  Header: 헤더 키/정규식 값 매칭
  Weight: WeightCalculatorWebFilter가 미리 선택한 Route와 비교
  After/Before/Between: 시각 기반 라우팅

우선순위:
  Route.order 값 낮을수록 먼저 평가
  YAML 순서대로 동일 order는 정렬
  구체적인 Path → 일반적인 Path 순으로 작성 권장
```

---

## 🤔 생각해볼 문제

**Q1.** 동일한 Path 패턴(`/api/**`)을 가진 두 Route가 있고, 하나는 `Header=X-Version, v2`, 다른 하나는 헤더 조건 없이 `order: 10`이다. `X-Version: v2` 헤더가 없는 요청이 오면 어느 Route로 라우팅되는가?

<details>
<summary>해설 보기</summary>

`order` 값이 낮은 Route부터 Predicate를 평가합니다. 만약 `Header=X-Version, v2` Route의 order가 더 낮으면 먼저 평가되지만 헤더가 없어 false가 되고, 다음으로 헤더 조건 없는 Route(order: 10)가 평가되어 Path만 매칭되면 이 Route로 라우팅됩니다. `order` 값이 같은 경우 YAML에서의 순서가 적용됩니다. 이처럼 "fallback" 라우트는 항상 높은 order 값으로 설정하고, 특수 조건 라우트는 낮은 order 값으로 설정하는 것이 올바른 패턴입니다.

</details>

---

**Q2.** `Weight=group-a, 90`과 `Weight=group-a, 10`으로 설정한 두 Route가 있다. 실제 트래픽이 정확히 90:10으로 분배되는가? 어떤 경우에 편차가 발생하는가?

<details>
<summary>해설 보기</summary>

`Math.random()`(0.0~1.0 균등 분포)을 사용하므로 통계적으로 90:10에 수렴하지만 단기적으로는 편차가 있습니다. 요청 수가 적을수록 편차가 큽니다(10개 요청에서 7:3이 나올 수도 있음). 수천 개 이상 요청이 쌓이면 90:10에 근접합니다. 또한 `WeightCalculatorWebFilter`는 요청마다 독립적으로 랜덤 값을 계산하므로 세션이나 사용자 기반 고정 분배가 아닙니다. 같은 사용자가 연속으로 요청해도 매번 다른 Route로 라우팅될 수 있습니다. 사용자 기반 고정 라우팅이 필요하다면 Cookie나 Header 기반 커스텀 Predicate를 구현해야 합니다.

</details>

---

**Q3.** Custom Predicate Factory에서 Reactive하게 DB를 조회해 권한을 확인해야 한다. 어떻게 구현해야 하는가?

<details>
<summary>해설 보기</summary>

`AsyncPredicate`의 `apply()` 메서드에서 Reactive 타입을 반환하면 됩니다.

```java
@Override
public AsyncPredicate<ServerWebExchange> applyAsync(Config config) {
    return exchange -> {
        String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
        if (userId == null) return Mono.just(false);

        // R2DBC(Reactive DB) 또는 WebClient로 비동기 권한 조회
        return permissionRepository.hasPermission(userId, config.getResource())
            .defaultIfEmpty(false);
        // Mono<Boolean> 반환 → Publisher<Boolean> 호환
    };
}
```

블로킹 DB 클라이언트(JDBC)를 사용해야 한다면 반드시 `subscribeOn(Schedulers.boundedElastic())`으로 이벤트 루프 스레드를 보호해야 합니다. 하지만 Gateway 내에서 매 요청마다 DB를 조회하면 성능 병목이 되므로, Caffeine 같은 로컬 캐시를 두고 TTL 기반으로 캐싱하는 것이 실무적입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Gateway vs Zuul](./01-gateway-vs-zuul.md)** | **[홈으로 🏠](../README.md)** | **[다음: Gateway Filter Factory ➡️](./03-gateway-filter-factory.md)**

</div>
