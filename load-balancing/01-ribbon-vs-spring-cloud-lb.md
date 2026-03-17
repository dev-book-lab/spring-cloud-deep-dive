# Ribbon vs Spring Cloud LoadBalancer — Netflix에서 Spring으로의 전환

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Netflix Ribbon이 Maintenance 상태로 전환된 기술적·조직적 이유는 무엇인가?
- `BlockingLoadBalancerClient`와 `ReactorLoadBalancer`는 각각 어떤 상황에 사용되는가?
- Ribbon에서 Spring Cloud LoadBalancer로 전환 시 변경해야 할 의존성과 설정은 무엇인가?
- Spring Cloud LoadBalancer가 Ribbon보다 Reactive 환경에 더 적합한 구체적인 이유는?
- `spring-cloud-starter-loadbalancer`가 자동으로 구성하는 Bean 목록은?
- 두 구현체의 캐시 전략(서버 목록 갱신 주기)은 어떻게 다른가?

---

## 🔍 왜 MSA에서 필요한가

### Client-Side Load Balancing이 필요한 이유

```
Server-Side LB (L4/L7 로드밸런서):
  클라이언트 → LB(고정 주소) → 서버 선택 → 업스트림

  문제:
  LB 자체가 SPOF 및 병목 → 수평 확장 어려움
  LB를 거치는 추가 네트워크 홉 → 지연 증가
  세밀한 라우팅 전략 적용 어려움 (카나리, Zone-Aware 등)

Client-Side LB (Ribbon / Spring Cloud LoadBalancer):
  클라이언트가 서버 목록을 직접 관리
  → LB 홉 없음 (직접 연결)
  → 클라이언트가 알고리즘 선택 가능
  → 메타데이터 기반 세밀한 라우팅

MSA에서 Client-Side LB의 가치:
  수십 개 서비스 × 수십 개 인스턴스
  서비스 간 호출마다 최적 인스턴스 선택
  중간 LB 없이 직접 통신 → 레이턴시 최소화
```

---

## 😱 잘못된 구성

### Before: Ribbon 의존성을 유지하면서 WebFlux 사용

```xml
<!-- ❌ WebFlux 환경에서 Ribbon 사용 시도 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
    <!-- Ribbon: 블로킹 기반 설계 -->
    <!-- WebFlux의 Non-Blocking 모델과 근본적 불일치 -->
</dependency>
```

```java
// ❌ WebClient + Ribbon의 문제
// Ribbon은 내부적으로 블로킹 방식으로 서버 목록 갱신
// Ribbon의 ILoadBalancer.getAllServers()는 동기 반환
// WebClient(Reactive)와 통합 시 Schedulers.boundedElastic()으로 격리 필요
// → Non-Blocking의 이점 상쇄
```

### Before: Ribbon 설정이 Spring Cloud LoadBalancer에서 동작하지 않음

```yaml
# ❌ Ribbon 전용 설정 (Spring Cloud LoadBalancer에서 무시됨)
service-order:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule
    ConnectTimeout: 3000
    ReadTimeout: 60000
    MaxAutoRetries: 1
    MaxAutoRetriesNextServer: 1
    # Spring Cloud LoadBalancer로 전환 시 이 설정들은 아무 효과 없음!
```

---

## ✨ 올바른 패턴

### After: 의존성 전환 가이드

```xml
<!-- Spring Boot 2.4+ / Spring Cloud 2020.0+ 권장 -->

<!-- ❌ 이전: Ribbon -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <!-- eureka-client가 ribbon을 전이 의존성으로 포함 -->
</dependency>

<!-- ✅ 이후: Spring Cloud LoadBalancer -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <exclusions>
        <!-- Ribbon 제외 -->
        <exclusion>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- Spring Cloud LoadBalancer 명시적 추가 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>

<!-- Caffeine 캐시 (선택, 성능 최적화) -->
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

---

## 🔬 내부 동작 원리

### 1. Netflix Ribbon — Maintenance 상태가 된 이유

```
Ribbon의 설계 (2013년대):
  Netflix 내부 자바 라이브러리
  동기(Blocking) 기반 설계
  ILoadBalancer, IRule, IPing 등 복잡한 인터페이스 체계
  자체 서버 목록 관리 (Eureka와 연동하지만 독립적)

문제들:
  ① Reactive(WebFlux) 지원 불가
    Ribbon의 핵심 인터페이스가 블로킹 반환 타입
    WebClient + Ribbon = 이벤트 루프 스레드 블로킹 위험

  ② Netflix 자체적으로 사용 중단
    Netflix는 내부적으로 gRPC + Envoy로 전환
    Ribbon 유지보수 인력 없음 → GitHub Issues 방치

  ③ Spring Cloud의 설계 방향과 불일치
    Spring이 추구하는 Reactive 퍼스트 방향
    Ribbon의 인터페이스 확장이 어려움

  ④ 보안 취약점 패치 지연
    Maintenance 상태 → CVE 패치도 느림

결정 (2020년):
  Spring Cloud 2020.0 (Ilford) 릴리즈에서
  Ribbon 기본값 비활성화
  → spring-cloud-starter-loadbalancer가 기본값
  Spring Cloud 2022.0 이후 Ribbon 완전 제거
```

### 2. Ribbon vs Spring Cloud LoadBalancer 아키텍처 비교

```
Ribbon 아키텍처:
  ┌────────────────────────────────────┐
  │  @LoadBalanced RestTemplate        │
  │    ↓ RibbonInterceptor             │
  │    ↓ ILoadBalancer.chooseServer()  │  ← 블로킹 반환
  │    ↓ IRule (WeightedResponseTime)  │
  │    ↓ IPing (상태 확인)               │
  │    ↓ ServerList (Eureka 연동)       │
  └────────────────────────────────────┘

  특징:
  - 모든 반환 타입이 동기 (Server, List<Server>)
  - 자체 서버 목록 갱신 타이머 (30초)
  - 자체 Ping 스레드 (IPing → 각 서버 Health Check)
  - 복잡한 설정 체계 (NFLoadBalancerRuleClassName 등)

Spring Cloud LoadBalancer 아키텍처:
  ┌────────────────────────────────────────────┐
  │  @LoadBalanced RestTemplate                │
  │    ↓ LoadBalancerInterceptor               │
  │    ↓ BlockingLoadBalancerClient.choose()   │
  │        → ReactorLoadBalancer.choose()      │  ← Mono<Response>
  │           → ServiceInstanceListSupplier    │  ← Flux<List<Instance>>
  │              → (Caching → Eureka/Discovery)│
  └────────────────────────────────────────────┘

  @LoadBalanced WebClient:
  ┌────────────────────────────────────────────┐
  │  @LoadBalanced WebClient                   │
  │    ↓ ReactorLoadBalancerExchangeFilter     │
  │    ↓ ReactorLoadBalancer.choose()          │  ← 완전 Reactive
  │        → ServiceInstanceListSupplier       │
  └────────────────────────────────────────────┘
```

### 3. LoadBalancerAutoConfiguration — 자동 구성 Bean

```java
// LoadBalancerAutoConfiguration.java
// spring-cloud-starter-loadbalancer 추가 시 자동 활성화

@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerClientsProperties.class)
public class LoadBalancerAutoConfiguration {

    // ① BlockingLoadBalancerClient: RestTemplate용 동기 클라이언트
    @Bean
    @ConditionalOnMissingBean
    public LoadBalancerClient blockingLoadBalancerClient(
            LoadBalancerClientFactory loadBalancerClientFactory,
            LoadBalancerClientsProperties properties) {
        return new BlockingLoadBalancerClient(loadBalancerClientFactory, properties);
    }

    // ② @LoadBalanced RestTemplate에 인터셉터 주입
    @Bean
    public LoadBalancerInterceptor loadBalancerInterceptor(
            LoadBalancerClient loadBalancerClient,
            LoadBalancerRequestFactory requestFactory) {
        return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
    }

    // ③ 캐시 설정 (Caffeine 사용 시)
    @Bean
    @ConditionalOnClass(name = "com.github.benmanes.caffeine.cache.Caffeine")
    public LoadBalancerCacheManager caffeineLoadBalancerCacheManager() {
        return new CaffeineBasedLoadBalancerCacheManager();
    }
}

// LoadBalancerClientFactory: 서비스별 ApplicationContext 관리
// 각 서비스(service-a, service-b...)마다 별도 Child ApplicationContext 생성
// → 서비스별 독립적인 LoadBalancer Bean 격리
@Bean
public LoadBalancerClientFactory loadBalancerClientFactory(
        LoadBalancerClientsProperties properties) {
    LoadBalancerClientFactory clientFactory = new LoadBalancerClientFactory();
    clientFactory.setConfigurations(
        properties.getClients().values().stream()
            .map(c -> new LoadBalancerClientSpecification(
                c.getName(), new Class[]{c.getConfiguration()}))
            .collect(Collectors.toList()));
    return clientFactory;
}
```

### 4. BlockingLoadBalancerClient vs ReactorLoadBalancer

```java
// BlockingLoadBalancerClient: RestTemplate(동기)용 어댑터
public class BlockingLoadBalancerClient implements LoadBalancerClient {

    @Override
    public ServiceInstance choose(String serviceId) {
        ReactorLoadBalancer<ServiceInstance> loadBalancer =
            loadBalancerClientFactory.getInstance(serviceId,
                ReactorServiceInstanceLoadBalancer.class);

        // Reactive → Blocking 변환 (RestTemplate 호환용)
        Response<ServiceInstance> response =
            Mono.from(loadBalancer.choose(buildRequest(serviceId))).block();
        //                                                          ^^^^^^
        // 이 block()이 RestTemplate 스레드에서 호출됨
        // Netty 이벤트 루프에서 호출 금지!

        return response != null && response.hasServer()
            ? response.getServer() : null;
    }
}

// ReactorLoadBalancer: WebClient(비동기)용 네이티브 인터페이스
public interface ReactorServiceInstanceLoadBalancer {
    // 완전 Reactive: 블로킹 없음
    Mono<Response<ServiceInstance>> choose(Request request);
}

// ReactorLoadBalancerExchangeFilterFunction: WebClient 통합
// block() 없이 Reactive 체인 내에서 직접 choose() 호출
public class ReactorLoadBalancerExchangeFilterFunction
        implements ExchangeFilterFunction {

    @Override
    public Mono<ClientResponse> filter(ClientRequest request,
            ExchangeFunction next) {
        URI url = request.url();
        String serviceId = url.getHost();

        return choose(serviceId, request)
            .flatMap(response -> {
                // 선택된 인스턴스로 URI 재작성
                ServiceInstance instance = response.getServer();
                URI newUri = reconstructURI(instance, url);
                ClientRequest newRequest = ClientRequest.from(request)
                    .url(newUri)
                    .build();
                return next.exchange(newRequest);
            });
        // 완전 Non-Blocking: 이벤트 루프 스레드 블로킹 없음
    }
}
```

### 5. 서버 목록 갱신 전략 비교

```java
// Ribbon의 서버 목록 갱신:
// PollingServerListUpdater: 30초마다 전체 목록 재조회
// IPing: 각 서버마다 Ping 요청 → Health Check
// → 타이머 스레드 N개 항상 실행 중 (리소스 소모)

// Spring Cloud LoadBalancer의 서버 목록 캐시:
// CachingServiceInstanceListSupplier:
spring:
  cloud:
    loadbalancer:
      cache:
        enabled: true          # 기본값 true
        ttl: 35s               # 캐시 TTL (기본 35초)
        capacity: 256          # 캐시 최대 항목 수

// 차이:
// Ribbon: 백그라운드 타이머가 능동적으로 갱신 (Push)
// Spring Cloud LB: TTL 만료 시 다음 요청에 재조회 (Pull on demand)
// → 타이머 스레드 없음 → 리소스 효율적
```

---

## 💻 실전 구성

### 전환 체크리스트

```yaml
# application.yml: Ribbon 비활성화 (전환 과도기)
spring:
  cloud:
    loadbalancer:
      ribbon:
        enabled: false   # Ribbon이 클래스패스에 있을 때 명시적 비활성화

# Spring Cloud LoadBalancer 캐시 설정
    cache:
      ttl: 35s
      capacity: 256

# Ribbon 전용 설정 제거 필요:
# (아래 설정들은 Spring Cloud LoadBalancer에서 무시됨)
# ribbon:
#   ConnectTimeout: 3000      → httpclient.connect-timeout으로 대체
#   ReadTimeout: 60000        → httpclient.response-timeout으로 대체
#   MaxAutoRetries: 1         → spring.cloud.loadbalancer.retry로 대체
```

```java
// Ribbon IRule → Spring Cloud LoadBalancer 알고리즘 대체 매핑
// Ribbon: RoundRobinRule → Spring: RoundRobinLoadBalancer (기본값)
// Ribbon: WeightedResponseTimeRule → Spring: Custom LoadBalancer (Ch4-03)
// Ribbon: ZoneAvoidanceRule → Spring: ZonePreferenceServiceInstanceListSupplier
// Ribbon: RandomRule → Spring: RandomLoadBalancer

// 서비스별 LoadBalancer 설정 (Ch4-05에서 상세)
@LoadBalancerClient(name = "service-order",
    configuration = OrderServiceLoadBalancerConfig.class)
public class LoadBalancerConfig {}
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: Ribbon → Spring Cloud LoadBalancer 무중단 마이그레이션

```
단계별 전환 전략:

1단계: 의존성 추가 (Ribbon 유지)
  pom.xml에 spring-cloud-starter-loadbalancer 추가
  spring.cloud.loadbalancer.ribbon.enabled=false 설정
  → Ribbon 클래스패스 있지만 비활성화
  → Spring Cloud LB 동작 확인

2단계: 설정 이전
  Ribbon 전용 설정 → Spring Cloud LB 설정으로 변환
  타임아웃, 재시도 설정 확인

3단계: Ribbon 의존성 제거
  pom.xml에서 Ribbon 제외
  → Spring Cloud LB만 남음

4단계: 검증
  curl 또는 테스트로 로드밸런싱 동작 확인
  Actuator /actuator/loadbalancer로 인스턴스 목록 확인

롤백:
  ribbon.enabled=true 로 즉시 롤백 (의존성 있는 경우)
```

---

## ⚖️ 트레이드오프

| 관점 | Netflix Ribbon | Spring Cloud LoadBalancer |
|------|----------------|--------------------------|
| **Reactive 지원** | 없음 (블로킹) | 완전 지원 (Mono/Flux) |
| **유지보수** | Maintenance (사실상 종료) | Spring 팀이 활발하게 개발 |
| **알고리즘** | 다양 (WeightedResponseTime 등) | 기본 2개 (RoundRobin, Random), 확장 가능 |
| **설정 복잡도** | 높음 (Netflix 전용 프로퍼티) | 낮음 (Spring 표준) |
| **Background 스레드** | 있음 (Ping, Polling) | 없음 (TTL 기반 캐시) |
| **WebFlux 통합** | 불가 (구조적 한계) | 네이티브 지원 |

---

## 📌 핵심 정리

```
Ribbon → Spring Cloud LoadBalancer 전환 핵심:

전환 이유:
  Ribbon: Netflix Maintenance 상태, Reactive 지원 불가
  Spring Cloud LB: 완전 Reactive, Spring 생태계 통합

핵심 인터페이스 매핑:
  Ribbon ILoadBalancer → ReactorServiceInstanceLoadBalancer
  Ribbon ServerList → ServiceInstanceListSupplier
  Ribbon IRule → choose() 내 선택 알고리즘
  Ribbon IPing → HealthCheckServiceInstanceListSupplier

두 클라이언트:
  BlockingLoadBalancerClient: RestTemplate용 (Reactive → Blocking)
  ReactorLoadBalancerExchangeFilterFunction: WebClient용 (순수 Reactive)

캐시 전략:
  Ribbon: 백그라운드 타이머 Poll (30초)
  Spring Cloud LB: TTL 만료 시 on-demand 재조회 (35초 기본)

의존성 전환:
  eureka-client에서 ribbon 제외
  spring-cloud-starter-loadbalancer 추가
  ribbon 전용 설정 제거 (무시되므로)
```

---

## 🤔 생각해볼 문제

**Q1.** Ribbon의 `IPing`은 각 서버에 주기적으로 Ping을 보내 헬스 체크를 했다. Spring Cloud LoadBalancer는 이에 해당하는 기능이 없는가? 어떻게 죽은 인스턴스를 걸러내는가?

<details>
<summary>해설 보기</summary>

Spring Cloud LoadBalancer는 기본적으로 별도의 Ping 스레드가 없습니다. 대신 두 가지 방식으로 죽은 인스턴스를 처리합니다.

1. **Eureka 레지스트리 기반**: `EurekaServiceInstanceListSupplier`는 Eureka에서 `InstanceStatus.UP`인 인스턴스만 반환합니다. Eureka의 Heartbeat(30초)와 만료(90초) 메커니즘이 죽은 인스턴스를 자동으로 제거합니다. 최대 120초 지연이 있습니다.

2. **HealthCheckServiceInstanceListSupplier**: `/actuator/health` 엔드포인트를 주기적으로 호출해 인스턴스 상태를 확인합니다. Ribbon의 IPing과 유사하지만 선택적으로 활성화해야 합니다:
```yaml
spring.cloud.loadbalancer.health-check.enabled=true
```

3. **Retry + Circuit Breaker**: 죽은 인스턴스 호출 실패 → Retry → 다른 인스턴스 → Circuit Breaker로 자동 격리. 이것이 실무에서 더 일반적인 방어 전략입니다.

</details>

---

**Q2.** `LoadBalancerClientFactory`가 서비스별로 별도의 Child ApplicationContext를 생성하는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

서비스별로 다른 LoadBalancer 설정(알고리즘, 서버 목록, 캐시 등)을 격리하기 위해서입니다. 예를 들어 `service-a`는 Round Robin, `service-b`는 커스텀 카나리 알고리즘을 사용할 수 있습니다. 공유 ApplicationContext에 두면 Bean 이름 충돌이 발생하거나 설정이 서로 영향을 줍니다. Child Context는 부모 Context의 Bean을 상속하지만 자신만의 Bean을 별도로 가집니다. `@LoadBalancerClient(name = "service-a", configuration = ServiceAConfig.class)`로 특정 서비스에만 별도 설정을 주입할 수 있습니다. 자세한 내용은 Ch4-05에서 다룹니다.

</details>

---

**Q3.** Spring Cloud LoadBalancer의 캐시 TTL을 0으로 설정하면 어떻게 되는가? 이것이 실제로 유용한 케이스가 있는가?

<details>
<summary>해설 보기</summary>

TTL 0 또는 `cache.enabled=false`로 설정하면 매 요청마다 `ServiceInstanceListSupplier`에서 인스턴스 목록을 새로 조회합니다. Eureka 클라이언트의 로컬 캐시(30초 갱신)에서 읽지만, LoadBalancer 자체의 캐시는 없으므로 항상 최신 Eureka 캐시를 기준으로 선택합니다.

유용한 케이스:
- 인스턴스 변경이 매우 빈번한 환경 (Auto Scaling이 수 초 단위로 발생)
- 테스트 환경 (인스턴스 추가/제거를 즉시 반영해야 하는 경우)
- 인스턴스 수가 매우 적어 캐시 갱신 지연이 체감되는 경우

단점: 모든 요청이 Eureka 클라이언트 캐시를 매번 읽음 → 약간의 CPU 오버헤드. 대부분의 프로덕션 환경에서는 기본값(35초)이 적합합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: LoadBalancerClient 동작 원리 ➡️](./02-load-balancer-client-internals.md)**

</div>
