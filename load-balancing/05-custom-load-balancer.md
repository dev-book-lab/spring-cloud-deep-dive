# Custom LoadBalancer 구현 — 메타데이터 기반 카나리 라우팅과 서비스별 격리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ReactorServiceInstanceLoadBalancer` 인터페이스를 구현해 Spring이 자동 사용하게 하는 등록 방법은?
- `@LoadBalancerClient`와 `@LoadBalancerClients`로 서비스별로 다른 LoadBalancer를 격리하는 방법은?
- Eureka 인스턴스 메타데이터(`version`, `zone`, `canary`)를 LoadBalancer 알고리즘에서 읽는 방법은?
- `ServiceInstanceListSupplier`를 커스터마이징해 인스턴스 목록을 필터링하는 방법은?
- Custom LoadBalancer 설정 클래스에 `@Configuration`을 붙이면 안 되는 이유는?
- `ObjectProvider<ServiceInstanceListSupplier>`를 통해 지연 초기화를 해야 하는 이유는?

---

## 🔍 왜 MSA에서 필요한가

### 기본 LoadBalancer로 해결 안 되는 케이스

```
기본 Round Robin의 한계:

1. 버전 기반 라우팅:
   service-a v1 (3개) + v2 (1개, 카나리)
   → Round Robin: 25%가 v2로 감 (균등 분배)
   → 원하는 것: v2는 내부 테스터 또는 X-Version: v2 헤더를 가진 요청만

2. Zone-Aware 라우팅:
   ap-northeast-2a: 3개, ap-northeast-2c: 3개
   → Round Robin: AZ 무관 50/50
   → 원하는 것: 같은 AZ 인스턴스 우선 (레이턴시 최소화, 전송 비용 절감)

3. 테넌트별 전용 인스턴스:
   tenant-premium 요청 → 고성능 전용 인스턴스
   tenant-standard 요청 → 일반 인스턴스
   → JWT의 tenantTier를 기반으로 인스턴스 선택

4. 스티키 세션:
   같은 사용자는 항상 같은 인스턴스로
   → 세션/캐시 일관성 유지
   → userId 해시 기반 일관성 있는 선택

이 모든 케이스가 Custom LoadBalancer로 해결 가능
```

---

## 😱 잘못된 구성

### Before: 설정 클래스에 @Configuration 추가

```java
// ❌ @LoadBalancerClient configuration에 @Configuration 추가 금지!
@Configuration  // ← 이것이 문제
@LoadBalancerClient(name = "order-service",
    configuration = OrderLoadBalancerConfig.class)
public class OrderLoadBalancerConfig {
    // @Configuration이 붙으면 Spring이 이 클래스를
    // 컴포넌트 스캔에서 발견해 전역으로 등록할 수 있음
    // → 모든 서비스에 이 LoadBalancer가 적용될 수 있음
    // → 격리 실패!
}
```

```java
// ✅ 올바른 방법: @Configuration 없음
// @LoadBalancerClient의 configuration 파라미터로만 사용
// 컴포넌트 스캔 범위 밖에 두는 것이 더 안전
public class OrderLoadBalancerConfig {
    @Bean
    public ReactorServiceInstanceLoadBalancer orderLoadBalancer(...) {
        return new CanaryLoadBalancer(...);
    }
}
```

### Before: ServiceInstanceListSupplier를 직접 생성

```java
// ❌ Supplier를 new로 직접 생성 → Child Context의 Bean 활용 못함
@Bean
public ReactorServiceInstanceLoadBalancer loadBalancer() {
    // ❌ EurekaClient를 직접 생성하면 캐싱, 설정 격리 등 놓침
    EurekaServiceInstanceListSupplier supplier =
        new EurekaServiceInstanceListSupplier(...);
    return new CustomLoadBalancer(supplier, serviceId);
}

// ✅ ObjectProvider를 통해 Child Context의 Supplier 사용
@Bean
public ReactorServiceInstanceLoadBalancer loadBalancer(
        Environment environment,
        LoadBalancerClientFactory factory) {
    String serviceId = environment.getProperty(
        LoadBalancerClientFactory.PROPERTY_NAME);
    return new CustomLoadBalancer(
        // Lazy Provider: Child Context에서 Supplier 지연 조회
        factory.getLazyProvider(serviceId, ServiceInstanceListSupplier.class),
        serviceId);
}
```

---

## ✨ 올바른 패턴

### After: 카나리 LoadBalancer 전체 구현

```java
// 메타데이터 기반 카나리 라우팅:
// X-Canary: true 헤더가 있으면 canary=true 인스턴스로
// 그 외에는 canary=true 인스턴스 제외

public class CanaryLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private static final String CANARY_KEY = "canary";
    private static final String CANARY_HEADER = "X-Canary";

    private final String serviceId;
    private final ObjectProvider<ServiceInstanceListSupplier> supplierObjectProvider;
    private final AtomicInteger position;

    public CanaryLoadBalancer(
            ObjectProvider<ServiceInstanceListSupplier> supplierObjectProvider,
            String serviceId) {
        this.serviceId = serviceId;
        this.supplierObjectProvider = supplierObjectProvider;
        this.position = new AtomicInteger(new Random().nextInt(1000));
    }

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier =
            supplierObjectProvider.getIfAvailable(
                NoopServiceInstanceListSupplier::new);

        return supplier.get(request).next()
            .map(instances -> getInstanceResponse(instances, request));
    }

    private Response<ServiceInstance> getInstanceResponse(
            List<ServiceInstance> instances, Request request) {

        if (instances.isEmpty()) {
            return new EmptyResponse();
        }

        // ① 요청에서 카나리 헤더 확인
        boolean isCanaryRequest = isCanaryRequest(request);

        // ② 카나리 여부에 따라 인스턴스 필터링
        List<ServiceInstance> targetInstances;
        if (isCanaryRequest) {
            // 카나리 요청: canary=true 인스턴스 우선
            List<ServiceInstance> canaryInstances = instances.stream()
                .filter(i -> "true".equals(i.getMetadata().get(CANARY_KEY)))
                .collect(Collectors.toList());
            // 카나리 인스턴스 없으면 전체 (폴백)
            targetInstances = canaryInstances.isEmpty()
                ? instances : canaryInstances;
        } else {
            // 일반 요청: canary=true 인스턴스 제외
            List<ServiceInstance> stableInstances = instances.stream()
                .filter(i -> !"true".equals(i.getMetadata().get(CANARY_KEY)))
                .collect(Collectors.toList());
            // 안정 버전 없으면 전체 (폴백)
            targetInstances = stableInstances.isEmpty()
                ? instances : stableInstances;
        }

        // ③ Round Robin으로 선택
        int pos = Math.abs(this.position.incrementAndGet())
            % targetInstances.size();
        return new DefaultResponse(targetInstances.get(pos));
    }

    private boolean isCanaryRequest(Request request) {
        if (request.getContext() instanceof RequestDataContext context) {
            HttpHeaders headers = context.getClientRequest().getHeaders();
            return "true".equalsIgnoreCase(headers.getFirst(CANARY_HEADER));
        }
        return false;
    }
}
```

---

## 🔬 내부 동작 원리

### 1. @LoadBalancerClient — 서비스별 Child Context에 Bean 등록

```java
// @LoadBalancerClient: 특정 서비스에만 적용되는 설정
// 설정 클래스의 Bean들이 서비스별 Child ApplicationContext에 등록됨

// 주의: 설정 클래스는 @Configuration 없이 일반 클래스로
// 컴포넌트 스캔에 걸리지 않는 위치에 두거나 @ComponentScan 범위 밖에 두기
// 권장: 별도 패키지에 분리

// 단일 서비스 설정:
@LoadBalancerClient(
    name = "order-service",
    configuration = OrderLoadBalancerConfig.class)
@Configuration
public class AppConfig {}

// 여러 서비스 설정:
@LoadBalancerClients({
    @LoadBalancerClient(name = "order-service",
        configuration = OrderLoadBalancerConfig.class),
    @LoadBalancerClient(name = "payment-service",
        configuration = PaymentLoadBalancerConfig.class)
})
@Configuration
public class AppConfig {}

// 기본값 설정 (모든 서비스에 적용):
@LoadBalancerClients(defaultConfiguration = DefaultLoadBalancerConfig.class)
@Configuration
public class AppConfig {}
```

### 2. 설정 클래스 내부 구조

```java
// OrderLoadBalancerConfig.java
// @Configuration 없음! 컴포넌트 스캔 제외
public class OrderLoadBalancerConfig {

    // ① 커스텀 LoadBalancer Bean
    @Bean
    public ReactorServiceInstanceLoadBalancer orderLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory factory) {

        // PROPERTY_NAME = "loadbalancer.client.name"
        // Child Context에 "order-service" 값으로 설정됨
        String serviceId = environment.getProperty(
            LoadBalancerClientFactory.PROPERTY_NAME);

        return new CanaryLoadBalancer(
            // getLazyProvider: Child Context에서 지연 조회
            // 순환 의존성 방지를 위해 Lazy 사용 필수
            factory.getLazyProvider(serviceId,
                ServiceInstanceListSupplier.class),
            serviceId);
    }

    // ② 커스텀 ServiceInstanceListSupplier (선택)
    // 기본: CachingServiceInstanceListSupplier → EurekaSupplier 체인
    // 커스터마이징 필요 시 오버라이드
    @Bean
    public ServiceInstanceListSupplier serviceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
            .withDiscoveryClient()    // DiscoveryClient 기반
            .withCaching()           // 캐싱 적용
            // .withHealthChecks()   // 헬스 체크 (Actuator 연동)
            // .withSameInstancePreference() // 같은 인스턴스 선호
            .build(context);
    }
}
```

### 3. Custom ServiceInstanceListSupplier — 인스턴스 필터링

```java
// 메타데이터로 인스턴스를 필터링하는 커스텀 Supplier
// Decorator 패턴으로 기존 Supplier 체인에 추가

public class MetadataBasedInstanceFilter
        extends DelegatingServiceInstanceListSupplier {

    private final String requiredVersion;

    public MetadataBasedInstanceFilter(
            ServiceInstanceListSupplier delegate,
            String requiredVersion) {
        super(delegate);
        this.requiredVersion = requiredVersion;
    }

    @Override
    public Flux<List<ServiceInstance>> get() {
        return getDelegate().get()
            .map(instances -> filterByVersion(instances));
    }

    @Override
    public Flux<List<ServiceInstance>> get(Request request) {
        // Request 컨텍스트에서 원하는 버전 추출
        String requestedVersion = extractVersionFromRequest(request);

        return getDelegate().get(request)
            .map(instances -> {
                if (requestedVersion != null) {
                    List<ServiceInstance> versionInstances = instances.stream()
                        .filter(i -> requestedVersion.equals(
                            i.getMetadata().get("version")))
                        .collect(Collectors.toList());
                    return versionInstances.isEmpty() ? instances : versionInstances;
                }
                return filterByVersion(instances);  // 기본 버전 필터
            });
    }

    private List<ServiceInstance> filterByVersion(List<ServiceInstance> instances) {
        List<ServiceInstance> filtered = instances.stream()
            .filter(i -> requiredVersion.equals(i.getMetadata().get("version")))
            .collect(Collectors.toList());
        return filtered.isEmpty() ? instances : filtered;
    }

    private String extractVersionFromRequest(Request request) {
        if (request.getContext() instanceof RequestDataContext ctx) {
            return ctx.getClientRequest().getHeaders().getFirst("X-API-Version");
        }
        return null;
    }
}
```

### 4. ObjectProvider 지연 초기화의 이유

```java
// ObjectProvider<ServiceInstanceListSupplier> 사용 이유:

// Child ApplicationContext 초기화 순서 문제:
// LoadBalancer Bean 초기화 시점에
// ServiceInstanceListSupplier Bean이 아직 생성 안 됐을 수 있음

// ObjectProvider: 실제 사용 시점에 Bean 조회 (지연 초기화)
ObjectProvider<ServiceInstanceListSupplier> provider;

// 첫 번째 요청이 올 때:
ServiceInstanceListSupplier supplier =
    provider.getIfAvailable(NoopServiceInstanceListSupplier::new);
// → 이 시점에 Supplier Bean 조회
// → 초기화 순서 문제 없음

// 직접 주입 시 발생하는 문제:
// @Autowired
// ServiceInstanceListSupplier supplier;
// → 순환 의존성 또는 초기화 순서 오류 발생 가능

// getLazyProvider: 특정 서비스의 Child Context에서 지연 조회
ObjectProvider<ServiceInstanceListSupplier> lazyProvider =
    factory.getLazyProvider("order-service", ServiceInstanceListSupplier.class);
// → order-service의 Child Context에서 Supplier 찾음
// → 다른 서비스의 Supplier와 격리됨
```

### 5. 스티키 세션 LoadBalancer 구현

```java
// 같은 사용자는 항상 같은 인스턴스로 (ConsistentHash 기반)
public class StickySessionLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private final String serviceId;
    private final ObjectProvider<ServiceInstanceListSupplier> supplierObjectProvider;

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier =
            supplierObjectProvider.getIfAvailable(
                NoopServiceInstanceListSupplier::new);

        return supplier.get(request).next()
            .map(instances -> {
                if (instances.isEmpty()) return new EmptyResponse();

                // userId를 스티키 키로 사용
                String stickyKey = extractStickyKey(request);

                if (stickyKey != null) {
                    // ConsistentHash: stickyKey → 항상 같은 인스턴스
                    // 인스턴스 추가/제거 시 최소한의 재분배
                    int hash = Math.abs(stickyKey.hashCode());
                    int idx = hash % instances.size();
                    return new DefaultResponse(instances.get(idx));
                }

                // 스티키 키 없으면 랜덤
                int idx = ThreadLocalRandom.current().nextInt(instances.size());
                return new DefaultResponse(instances.get(idx));
            });
    }

    private String extractStickyKey(Request request) {
        if (request.getContext() instanceof RequestDataContext ctx) {
            // X-User-Id 헤더 또는 쿠키에서 추출
            String userId = ctx.getClientRequest()
                .getHeaders().getFirst("X-User-Id");
            if (userId != null) return userId;

            // 쿠키에서 세션 ID 추출
            return ctx.getClientRequest().getCookies()
                .getFirst("JSESSIONID") != null
                ? ctx.getClientRequest().getCookies()
                    .getFirst("JSESSIONID").getValue()
                : null;
        }
        return null;
    }
}
```

---

## 💻 실전 구성

### 카나리 배포 완전한 설정

```yaml
# 카나리 인스턴스 Eureka 메타데이터 설정
# (카나리 서비스의 application.yml)
eureka:
  instance:
    metadata-map:
      canary: "true"
      version: "v2.0.0"
```

```java
// 설정 클래스 (컴포넌트 스캔 제외 패키지에 위치)
// src/main/java/com/example/lb/config/ (별도 패키지)
public class CanaryLoadBalancerConfig {

    @Bean
    public ReactorServiceInstanceLoadBalancer canaryLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory factory) {
        String serviceId = environment.getProperty(
            LoadBalancerClientFactory.PROPERTY_NAME);
        return new CanaryLoadBalancer(
            factory.getLazyProvider(serviceId,
                ServiceInstanceListSupplier.class),
            serviceId);
    }
}

// 적용 대상 서비스 지정
@SpringBootApplication
@LoadBalancerClients({
    @LoadBalancerClient(name = "order-service",
        configuration = CanaryLoadBalancerConfig.class),
    @LoadBalancerClient(name = "product-service",
        configuration = CanaryLoadBalancerConfig.class)
})
public class Application { ... }
```

```bash
# 카나리 요청 (X-Canary 헤더)
curl -H "X-Canary: true" http://localhost:8080/api/orders/1
# → canary=true 인스턴스로 라우팅 (v2.0.0)

# 일반 요청 (카나리 제외)
curl http://localhost:8080/api/orders/1
# → canary=true 인스턴스 제외, stable 인스턴스로 라우팅

# 응답 확인 (인스턴스 버전 확인)
curl -H "X-Canary: true" http://localhost:8080/api/orders/1 | jq '.serverVersion'
# → "v2.0.0" (카나리)
curl http://localhost:8080/api/orders/1 | jq '.serverVersion'
# → "v1.0.0" (stable)
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 단계적 카나리 배포 롤아웃

```
Phase 1: 내부 테스터 (X-Canary: true 헤더)
  → 카나리 인스턴스 1개 배포
  → 내부 QA팀이 X-Canary: true로 검증
  → 일반 사용자: 영향 없음

Phase 2: 특정 사용자 그룹 (X-User-Group: beta)
  → BetaUserGroupLoadBalancer 구현:
    X-User-Group: beta 헤더 → 카나리
    기타 → stable
  → 베타 사용자 5000명 대상 검증

Phase 3: 가중치 기반 점진적 확대
  → WeightedCanaryLoadBalancer 구현:
    95:5 → 90:10 → 70:30 → 50:50 → 0:100
  → 각 단계별 에러율, 레이턴시 모니터링

Phase 4: 전체 전환 (카나리 메타데이터 제거)
  → 모든 인스턴스 v2 → canary=true 메타데이터 제거
  → CanaryLoadBalancer는 그대로 (모든 인스턴스 선택 대상)
  → stable 버전 인스턴스 점진적 제거

롤백:
  카나리 인스턴스 종료 → Eureka에서 제거 → LB 캐시 갱신
  일반 인스턴스만 남음 → 자동 롤백 완료
```

---

## ⚖️ 트레이드오프

| 커스텀 방식 | 유연성 | 복잡도 | 유지보수 |
|-----------|--------|--------|---------|
| **메타데이터 필터링** | 높음 | 중간 | 메타데이터 관리 필요 |
| **헤더 기반 라우팅** | 높음 | 낮음 | 클라이언트 헤더 관리 필요 |
| **ConsistentHash (스티키)** | 중간 | 높음 | 인스턴스 변경 시 분배 변경 |
| **ServiceInstanceListSupplier 커스텀** | 최고 | 높음 | 복잡한 테스트 필요 |

```
@LoadBalancerClient configuration 주의사항:

✅ 올바른 위치:
  - @SpringBootApplication 클래스가 있는 패키지의 하위 패키지
    (단, @ComponentScan 범위 밖)
  - 별도의 "loadbalancer.config" 패키지
  - @ComponentScan excludeFilters로 제외

✅ 올바른 클래스 선언:
  - public class (패키지 접근 가능)
  - @Configuration 없음
  - @Bean 메서드만 포함

❌ 잘못된 패턴:
  - 메인 Application 클래스와 같은 패키지 → 컴포넌트 스캔에서 전역 적용
  - @Configuration 추가 → 전역 Bean으로 등록될 위험
  - static inner class로 사용 → Child Context에서 접근 어려울 수 있음
```

---

## 📌 핵심 정리

```
Custom LoadBalancer 구현 핵심:

인터페이스:
  ReactorServiceInstanceLoadBalancer
    choose(Request) → Mono<Response<ServiceInstance>>
  
  Request.getContext() → RequestDataContext
    → HttpHeaders, Cookies, URI 접근 가능

등록 방법:
  @LoadBalancerClient(name = "service-name",
    configuration = MyConfig.class)
  
  설정 클래스: @Configuration 없는 일반 클래스
               컴포넌트 스캔 범위 밖에 위치

ObjectProvider 사용 이유:
  Child Context 초기화 순서 문제 방지
  factory.getLazyProvider(serviceId, ServiceInstanceListSupplier.class)
  → 실제 요청 시점에 Supplier 조회

커스텀 Supplier:
  DelegatingServiceInstanceListSupplier 상속
  get(Request) 오버라이드 → 인스턴스 필터링
  ServiceInstanceListSupplier.builder()로 체인 구성

카나리 패턴:
  메타데이터 metadata-map.canary: "true"
  X-Canary 헤더 여부로 대상 인스턴스 풀 분리
  카나리 없을 때 전체 폴백 (안전)
```

---

## 🤔 생각해볼 문제

**Q1.** 스티키 세션 LoadBalancer에서 인스턴스가 2개에서 3개로 Scale-out될 때 어떤 문제가 발생하는가? ConsistentHash가 이를 어떻게 완화하는가?

<details>
<summary>해설 보기</summary>

단순 `userId.hashCode() % instances.size()`를 사용하면 인스턴스 수가 변경될 때 대부분의 사용자가 다른 인스턴스로 재분배됩니다. 인스턴스 2개→3개 추가 시 이론적으로 2/3의 사용자가 다른 인스턴스로 이동합니다. 이로 인해 세션 정보나 로컬 캐시가 유실됩니다.

ConsistentHash(일관성 해시):
- 인스턴스들을 해시 링(ring)에 배치
- 각 요청은 해시 링에서 가장 가까운 인스턴스로 라우팅
- 인스턴스 1개 추가 시 해당 인스턴스의 "구간"만 재분배 → 전체 중 1/N만 이동

Guava의 `Hashing.consistentHash(userId.hashCode(), instances.size())`로 구현 가능합니다. 단, 인스턴스 순서가 일관성 있어야 하므로 인스턴스 목록을 instanceId로 정렬한 후 사용합니다.

</details>

---

**Q2.** `@LoadBalancerClient` 설정 클래스에 정의한 Bean이 모든 서비스에 적용되는 버그가 발생했다. 원인과 해결 방법은?

<details>
<summary>해설 보기</summary>

설정 클래스가 Spring의 컴포넌트 스캔 범위 안에 `@Configuration`과 함께 위치하면 전역 ApplicationContext에 Bean으로 등록됩니다. `LoadBalancerClientFactory`가 Child Context를 만들 때 부모 Context에서 Bean을 상속하므로, 전역에 등록된 Bean이 모든 서비스의 Child Context에서도 사용됩니다.

해결 방법:
1. **설정 클래스에서 `@Configuration` 제거** (가장 중요)
2. **컴포넌트 스캔 범위 밖으로 이동**: `@SpringBootApplication`의 basePackages 밖의 패키지에 위치
3. **`@ComponentScan` 명시적 제외**:
```java
@SpringBootApplication
@ComponentScan(excludeFilters = {
    @ComponentScan.Filter(type = FilterType.REGEX,
        pattern = "com.example.lb.config.*")
})
```
4. **확인 방법**: `actuator/beans`에서 해당 Bean이 루트 컨텍스트가 아닌 특정 서비스의 컨텍스트에만 있는지 확인

</details>

---

**Q3.** Custom LoadBalancer가 `EmptyResponse()`를 반환했을 때 `BlockingLoadBalancerClient.choose()`는 어떻게 처리하는가? 이때 클라이언트가 받는 오류는?

<details>
<summary>해설 보기</summary>

`EmptyResponse.hasServer()` = false를 반환하므로, `BlockingLoadBalancerClient.choose()`에서 다음 로직이 실행됩니다.

```java
if (loadBalancerResponse == null || !loadBalancerResponse.hasServer()) {
    return null;  // null 반환
}
```

`choose()`가 null을 반환하면 `execute()` 메서드에서:
```java
ServiceInstance serviceInstance = choose(serviceId);
if (serviceInstance == null) {
    throw new IllegalStateException(
        "No instances available for " + serviceId);
}
```

`IllegalStateException`이 발생합니다. 이는 Spring MVC에서 500 Internal Server Error로, Spring WebFlux에서도 500 오류로 클라이언트에 전달됩니다.

개선: `EmptyResponse` 대신 명시적 예외를 던지거나, `GlobalExceptionHandler`에서 `IllegalStateException`을 잡아 503 Service Unavailable로 변환하는 것이 사용자 경험 측면에서 더 좋습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Retry 전략](./04-retry-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — Circuit Breaker ➡️](../circuit-breaker/01-circuit-breaker-pattern.md)**

</div>
