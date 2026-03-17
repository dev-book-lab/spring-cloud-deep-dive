# Route Locator 동적 라우팅 — 런타임에 라우트를 추가·변경·삭제하는 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RouteLocator` 인터페이스의 `getRoutes()` 반환 타입이 `Flux<Route>`인 이유는?
- Java DSL(`RouteLocatorBuilder`)과 YAML 방식의 내부 처리 흐름 차이는?
- `RefreshRoutesEvent`가 발행될 때 Gateway가 라우트를 어떻게 재로드하는가?
- `CachingRouteLocator`가 라우트를 캐시하는 이유와 캐시 무효화 시점은?
- Config Server 연동으로 라우트를 동적으로 변경하는 완전한 구성 방법은?
- `CompositeRouteLocator`가 여러 `RouteLocator`를 어떻게 통합하는가?

---

## 🔍 왜 MSA에서 필요한가

### 재배포 없이 라우트를 변경해야 하는 상황

```
정적 라우트(YAML 고정)의 한계:

케이스 1: 긴급 트래픽 전환
  "service-b에 장애 발생 → 즉시 service-b-fallback으로 라우팅"
  → YAML 수정 → 배포 → 수 분 소요
  → 이 시간 동안 장애 지속

케이스 2: 실시간 A/B 테스트 비율 조정
  Weight=group, 95 → Weight=group, 50 으로 변경
  → 재배포 없이 Config Server에서 설정만 변경

케이스 3: 새 마이크로서비스 온보딩
  새로운 payment-v3 서비스 추가
  → Gateway 재시작 없이 새 라우트 추가

케이스 4: 시간 기반 라우팅
  트래픽 적은 새벽에만 배치 서비스 라우트 활성화
  → 스케줄러가 런타임에 라우트 추가/제거

RouteLocator + RefreshRoutesEvent:
  이 모든 케이스를 재배포 없이 처리
```

---

## 😱 잘못된 구성

### Before: YAML만으로 모든 라우트 하드코딩

```yaml
# ❌ 변경 불가능한 정적 라우트
spring:
  cloud:
    gateway:
      routes:
        - id: payment-route
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
          # Config Server를 써도 이 변경이 Gateway에 즉시 반영되지 않음
          # → /actuator/refresh로 @RefreshScope Bean 갱신해도
          #    YAML 기반 라우트는 RouteDefinitionLocator가 별도 관리
          #    → 재시작 없이 변경 불가능
```

### Before: RefreshRoutesEvent 발행 없이 RouteLocator Bean만 변경

```java
// ❌ RouteLocator Bean을 @RefreshScope로만 갱신 시도
@RefreshScope
@Bean
public RouteLocator dynamicRoutes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("payment", r -> r
            .path("/api/payments/**")
            .uri(paymentServiceUri))  // @Value로 주입된 값
        .build();
}
// @RefreshScope로 Bean이 재생성되어도
// Gateway의 CachingRouteLocator 캐시는 그대로
// → RefreshRoutesEvent 발행 없이는 새 라우트 적용 안 됨
```

---

## ✨ 올바른 패턴

### After: RouteLocator 계층 구조 전체

```
RouteLocator 계층 구조:

  CachingRouteLocator                     ← 최상위, 실제 요청 처리에 사용
    (getRoutes() 결과를 캐시)
    ↓ 위임
  CompositeRouteLocator                   ← 여러 RouteLocator 통합
    ↓ 위임
    ├── RouteDefinitionRouteLocator       ← YAML/Properties 기반
    │     (RouteDefinition → Route 변환)
    └── Custom RouteLocator Bean          ← Java DSL 또는 DB 기반
          (RouteLocatorBuilder로 생성)

RefreshRoutesEvent 발행:
  CachingRouteLocator가 이벤트 수신
  → 캐시 무효화
  → 다음 getRoutes() 호출 시 하위 RouteLocator들 재조회
  → 새 라우트 적용
```

---

## 🔬 내부 동작 원리

### 1. RouteLocator 인터페이스

```java
// RouteLocator.java
public interface RouteLocator {
    // Flux<Route>: 여러 Route를 Reactive 스트림으로 반환
    // 요청이 올 때마다 호출될 수 있음 → 매번 최신 상태 반환 가능
    // CachingRouteLocator가 이를 캐시
    Flux<Route> getRoutes();
}

// Route 객체: 불변 (빌더 패턴)
public class Route implements Ordered {
    private final String id;
    private final URI uri;
    private final int order;
    private final AsyncPredicate<ServerWebExchange> predicate;
    private final List<GatewayFilter> filters;
    private final Map<String, Object> metadata;
}
```

### 2. RouteLocatorBuilder — Java DSL

```java
// RouteLocatorBuilder.java
// YAML보다 타입 안전하고 조건부 라우트 구성 가능

@Configuration
public class GatewayRoutesConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder,
            @Value("${gateway.payment.uri}") String paymentUri) {

        return builder.routes()
            // ① 기본 라우팅
            .route("order-route", r -> r
                .path("/api/orders/**")
                .filters(f -> f
                    .rewritePath("/api(?<seg>/?.*)", "/${seg}")
                    .addRequestHeader("X-Service", "gateway")
                    .retry(config -> config
                        .setRetries(3)
                        .setStatuses(HttpStatus.BAD_GATEWAY)))
                .uri("lb://order-service"))

            // ② 조건부 라우팅: 헤더 + 경로 AND
            .route("canary-route", r -> r
                .path("/api/orders/**")
                .and()
                .header("X-Canary", "true")
                .filters(f -> f.rewritePath("/api(?<seg>/?.*)", "/${seg}"))
                .uri("lb://order-service-v2"))

            // ③ Lambda로 복잡한 조건 처리
            .route("dynamic-route", r -> r
                .asyncPredicate(exchange -> {
                    // 커스텀 조건 (Reactive)
                    return isBusinessHour()
                        ? Mono.just(true)
                        : Mono.just(false);
                })
                .uri(paymentUri))

            .build();
    }
}
```

### 3. CachingRouteLocator — 캐시와 갱신

```java
// CachingRouteLocator.java
// 라우트 조회 결과를 캐시 → 매 요청마다 RouteLocator 조회 방지

public class CachingRouteLocator implements Ordered, RouteLocator,
        ApplicationListener<RefreshRoutesEvent>, ApplicationEventPublisherAware {

    private final RouteLocator delegate;
    // 캐시: Flux<Route>를 구독한 결과를 보관
    private volatile List<Route> cachedRoutes;

    @Override
    public Flux<Route> getRoutes() {
        if (this.cachedRoutes == null) {
            // 최초 또는 캐시 무효화 후: delegate에서 로드
            this.cachedRoutes = fetch();
        }
        return Flux.fromIterable(this.cachedRoutes);
    }

    private List<Route> fetch() {
        // CompositeRouteLocator의 getRoutes() 호출
        // → RouteDefinitionRouteLocator + Custom RouteLocator 모두 조회
        return this.delegate.getRoutes()
            .sort(AnnotationAwareOrderComparator.INSTANCE)  // Order 정렬
            .collectList()
            .block();  // 초기화 시 한 번만 블로킹 허용
    }

    @Override
    public void onApplicationEvent(RefreshRoutesEvent event) {
        // ① RefreshRoutesEvent 수신
        // ② 캐시 무효화
        this.cachedRoutes = fetch();  // 재로드
        // ③ RefreshRoutesResultEvent 발행 (갱신 완료 알림)
        this.publisher.publishEvent(new RefreshRoutesResultEvent(this));
    }
}
```

### 4. Config Server 연동 동적 라우팅 전체 흐름

```java
// ① Config Server에서 라우트 설정 관리
// config-repo/api-gateway.yml:
// gateway.routes:
//   - id: payment-route
//     uri: lb://payment-service
//     predicates:
//       - Path=/api/payments/**

// ② Gateway의 @ConfigurationProperties + @RefreshScope
@ConfigurationProperties(prefix = "gateway")
@RefreshScope  // Config Server 갱신 시 재바인딩
@Component
public class GatewayRouteProperties {
    private List<RouteConfig> routes = new ArrayList<>();

    @Data
    public static class RouteConfig {
        private String id;
        private String uri;
        private List<String> predicates;
    }
}

// ③ 동적 RouteLocator: Properties를 읽어 Route 생성
@Component
public class DynamicRouteLocator implements RouteLocator {

    private final GatewayRouteProperties properties;
    private final RouteLocatorBuilder builder;

    @Override
    public Flux<Route> getRoutes() {
        RouteLocatorBuilder.Builder routes = builder.routes();

        properties.getRoutes().forEach(config -> {
            routes.route(config.getId(), r -> r
                .path(config.getPredicates().get(0).replace("Path=", ""))
                .uri(config.getUri()));
        });

        return routes.build().getRoutes();
    }
}

// ④ @RefreshScope Bean이 재생성될 때 RefreshRoutesEvent 발행
@Component
public class RouteRefreshListener implements ApplicationListener<EnvironmentChangeEvent> {

    private final ApplicationEventPublisher publisher;

    @Override
    public void onApplicationEvent(EnvironmentChangeEvent event) {
        // 라우팅 관련 설정이 변경됐다면 라우트 갱신
        if (event.getKeys().stream().anyMatch(k -> k.startsWith("gateway.routes"))) {
            publisher.publishEvent(new RefreshRoutesEvent(this));
        }
    }
}
```

### 5. 런타임 라우트 CRUD (Actuator API)

```java
// Spring Cloud Gateway Actuator 엔드포인트
// /actuator/gateway/routes — 전체 라우트 조회
// POST /actuator/gateway/refresh — 라우트 강제 갱신

// RouteDefinitionRepository를 구현하면 CRUD API 활성화
@Component
public class InMemoryRouteDefinitionRepository implements RouteDefinitionRepository {

    private final Map<String, RouteDefinition> routes = new ConcurrentHashMap<>();

    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return Flux.fromIterable(routes.values());
    }

    @Override
    public Mono<Void> save(Mono<RouteDefinition> route) {
        return route.flatMap(rd -> {
            routes.put(rd.getId(), rd);
            return Mono.empty();
        });
    }

    @Override
    public Mono<Void> delete(Mono<String> routeId) {
        return routeId.flatMap(id -> {
            routes.remove(id);
            return Mono.empty();
        });
    }
}

// Actuator API 사용:
// POST /actuator/gateway/routes/{id}  → 새 라우트 추가
// DELETE /actuator/gateway/routes/{id} → 라우트 삭제
// POST /actuator/gateway/refresh      → 변경 적용 (RefreshRoutesEvent 발행)
```

---

## 💻 실전 구성

### Config Server + RefreshRoutesEvent 완전한 구성

```yaml
# config-repo/api-gateway.yml
gateway:
  routes:
    - id: order-route
      uri: lb://order-service
      predicates: Path=/api/orders/**
      filters:
        - RewritePath=/api(?<s>/?.*), /${s}
    
    - id: payment-route
      uri: lb://payment-service
      predicates: Path=/api/payments/**

# api-gateway application.yml
spring:
  application:
    name: api-gateway
  config:
    import: "configserver:"

management:
  endpoints:
    web:
      exposure:
        include: gateway, refresh, busrefresh
```

```java
// Gateway Auto-configuration 활성화 확인
// spring.cloud.gateway.enabled=true (기본값)

// 동적 라우팅 모니터링
@EventListener
public void onRoutesRefreshed(RefreshRoutesResultEvent event) {
    log.info("Routes refreshed: {} routes loaded", 
        routeLocator.getRoutes().count().block());
}
```

```bash
# 라우트 변경 → Config Server push → 갱신

# 1. config-repo에서 payment-route URI 변경 후 push
# uri: lb://payment-service-v2

# 2. Spring Cloud Bus로 전체 인스턴스 갱신
curl -X POST http://api-gateway:8080/actuator/busrefresh

# 3. 라우트 갱신 확인
curl http://api-gateway:8080/actuator/gateway/routes | jq '.[] | {id, uri}'
# → payment-route가 lb://payment-service-v2로 변경됨 확인
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 장애 발생 → 즉시 대체 서비스로 라우팅 전환

```
t=0:   payment-service 장애 감지
       AlertManager → Webhook → 자동화 스크립트 실행

t=5s:  Config Server의 api-gateway.yml 수정
       uri: lb://payment-service → lb://payment-service-fallback

t=10s: git commit + push

t=15s: GitHub Webhook → POST /actuator/busrefresh

t=20s: 모든 Gateway 인스턴스: RefreshRoutesEvent 수신
       → CachingRouteLocator 캐시 무효화
       → 새 라우트 로드
       → payment-fallback으로 트래픽 전환 완료

재배포 없이 약 20초 만에 트래픽 전환 완료
```

### 시나리오: 새 서비스 온보딩 (라우트 추가)

```bash
# 새 analytics-service 출시 → Gateway에 라우트 추가

# 방법 1: Config Server에 라우트 추가 후 refresh
# config-repo/api-gateway.yml에 analytics-route 추가 후 push
curl -X POST http://api-gateway:8080/actuator/gateway/refresh

# 방법 2: Actuator API 직접 호출
curl -X POST http://api-gateway:8080/actuator/gateway/routes/analytics-route \
  -H "Content-Type: application/json" \
  -d '{
    "id": "analytics-route",
    "predicates": [{"name": "Path", "args": {"pattern": "/api/analytics/**"}}],
    "filters": [{"name": "StripPrefix", "args": {"parts": "1"}}],
    "uri": "lb://analytics-service",
    "order": 0
  }'

# 라우트 활성화
curl -X POST http://api-gateway:8080/actuator/gateway/refresh
```

---

## ⚖️ 트레이드오프

| 방식 | 유연성 | 복잡도 | 적합한 케이스 |
|------|--------|--------|--------------|
| **YAML 정적** | 낮음 | 최소 | 단순한 고정 라우팅 |
| **Java DSL** | 높음 | 중간 | 조건부, 프로그래매틱 라우팅 |
| **Config Server 연동** | 높음 | 중간 | 운영 중 라우트 변경 |
| **Actuator CRUD API** | 최고 | 높음 | 완전 동적, 자동화 |
| **DB 기반 RouteLocator** | 최고 | 최고 | 대규모, Admin UI |

```
CachingRouteLocator의 중요성:
  캐시 없이 매 요청마다 RouteLocator.getRoutes() 호출:
    → RouteDefinitionLocator: YAML 파싱 반복
    → Custom RouteLocator: DB 조회 반복
    → 성능 영향 심각

  캐시 있음:
    RefreshRoutesEvent 전까지 메모리의 Route 목록 재사용
    → 라우트 조회 오버헤드 없음

  동적 라우팅 도입 시 항상 CachingRouteLocator와 함께
  RefreshRoutesEvent로만 갱신 → 예측 가능한 캐시 관리
```

---

## 📌 핵심 정리

```
RouteLocator 동적 라우팅 핵심:

계층 구조:
  CachingRouteLocator (캐시 관리)
    → CompositeRouteLocator (통합)
      → RouteDefinitionRouteLocator (YAML)
      → Custom RouteLocator Bean (Java DSL / DB)

갱신 메커니즘:
  설정 변경 → RefreshRoutesEvent 발행
  → CachingRouteLocator.onApplicationEvent() 실행
  → 캐시 무효화 → 하위 RouteLocator 재조회
  → 새 라우트 적용

Config Server 연동:
  api-gateway.yml에 라우트 설정
  → /actuator/busrefresh 로 모든 인스턴스 동시 갱신
  → @RefreshScope Bean 재생성 + RefreshRoutesEvent

Actuator API (RouteDefinitionRepository):
  POST /actuator/gateway/routes/{id} → 라우트 추가
  DELETE /actuator/gateway/routes/{id} → 라우트 삭제
  POST /actuator/gateway/refresh → 갱신 트리거

Java DSL:
  RouteLocatorBuilder로 타입 안전한 라우트 정의
  Lambda로 복잡한 조건 처리 가능
  @RefreshScope 또는 동적 RouteLocator로 런타임 갱신
```

---

## 🤔 생각해볼 문제

**Q1.** `CachingRouteLocator.fetch()`에서 `collectList().block()`을 호출한다. Gateway가 Non-Blocking인데 이 블로킹 호출은 문제가 없는가?

<details>
<summary>해설 보기</summary>

`fetch()`는 `RefreshRoutesEvent` 처리 시점에 호출됩니다. 이 이벤트는 Spring의 이벤트 리스너에서 처리되며, Gateway의 Netty 이벤트 루프 스레드가 아닌 일반 Spring 이벤트 스레드(또는 전용 스케줄러)에서 실행됩니다. 따라서 이벤트 루프 스레드를 블로킹하지 않아 문제가 없습니다. 만약 Netty 이벤트 루프 스레드에서 `block()`이 호출되면 `IllegalStateException`이 발생합니다. 최신 버전에서는 이를 `Schedulers.boundedElastic()`으로 개선하는 작업도 진행 중입니다.

</details>

---

**Q2.** Java DSL로 정의한 RouteLocator Bean과 YAML로 정의한 라우트가 동시에 존재할 때, 동일한 Path Predicate를 가진 경우 어느 것이 우선하는가?

<details>
<summary>해설 보기</summary>

`Order` 값에 따라 결정됩니다. `CompositeRouteLocator`는 모든 RouteLocator의 `getRoutes()` 결과를 합산한 뒤 `CachingRouteLocator`가 `AnnotationAwareOrderComparator`로 정렬합니다. Java DSL의 `.route()`에서 `order()`를 명시하지 않으면 기본값(0)이 사용되고, YAML 라우트도 `order: 0`이 기본값입니다. 동일한 Order에서는 리스트 순서에 의존하므로 예측하기 어렵습니다. Java DSL 라우트에 명시적으로 더 낮은 order 값을 지정하거나, 경로 패턴을 명확히 구분하여 충돌을 피하는 것이 권장됩니다.

</details>

---

**Q3.** DB에 라우트 설정을 저장하고 Admin UI에서 라우트를 추가/삭제할 수 있는 시스템을 구성하려면 어떤 컴포넌트를 구현해야 하는가?

<details>
<summary>해설 보기</summary>

다음 컴포넌트가 필요합니다.

1. **`RouteDefinitionRepository` 구현**: `getRouteDefinitions()`, `save()`, `delete()`를 DB(R2DBC 권장 — Reactive) 기반으로 구현합니다.

2. **Admin REST API**: `RouteDefinitionRepository.save()`/`delete()`를 호출하는 API를 Spring WebFlux Controller로 구현합니다. 라우트 변경 후 `ApplicationEventPublisher.publishEvent(new RefreshRoutesEvent(this))`로 Gateway에 갱신을 알립니다.

3. **다중 인스턴스 동기화**: Admin API가 하나의 Gateway 인스턴스에만 변경을 알리므로, 모든 인스턴스에 반영하려면 Spring Cloud Bus 또는 Redis Pub/Sub을 활용하여 `RefreshRoutesEvent`를 모든 인스턴스에 전파해야 합니다.

4. **초기 로드**: Gateway 시작 시 `RouteDefinitionRepository.getRouteDefinitions()`가 DB에서 모든 라우트를 로드합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Global Filter 체인 실행 순서](./04-global-filter-chain.md)** | **[홈으로 🏠](../README.md)** | **[다음: Gateway Timeout & Circuit Breaker 통합 ➡️](./06-gateway-timeout-circuit-breaker.md)**

</div>
