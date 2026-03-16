# Client-Side Load Balancing — @LoadBalanced RestTemplate이 서비스 이름을 IP로 바꾸는 과정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@LoadBalanced`가 `RestTemplate`에 무엇을 추가하며, 어떻게 동작 방식을 바꾸는가?
- `LoadBalancerInterceptor.intercept()`에서 서비스 이름 → 실제 URL 치환이 일어나는 정확한 위치는?
- `BlockingLoadBalancerClient.choose()`는 어떤 과정으로 인스턴스를 선택하는가?
- `ServiceInstanceListSupplier` 체인에서 캐싱 레이어는 어떻게 동작하는가?
- `@LoadBalanced WebClient`와 `@LoadBalanced RestTemplate`의 내부 구현 차이는?
- LoadBalancer가 선택한 인스턴스에서 연결 실패가 발생하면 어떻게 처리되는가?

---

## 🔍 왜 MSA에서 필요한가

### 서비스 이름으로 호출하는 것의 의미

```
@LoadBalanced 없이 서비스 호출:
  String url = "http://10.0.1.5:8080/orders";  // 직접 IP
  → IP 변경, Scale-out 대응 불가

@LoadBalanced 있는 서비스 호출:
  String url = "http://service-order/orders";   // 서비스 이름
  → LoadBalancer가 Eureka에서 인스턴스 목록 조회
  → Round Robin으로 하나 선택
  → http://10.0.1.5:8080/orders 로 실제 요청

개발자가 신경 쓸 것:
  ① 서비스 이름 (Eureka에 등록된 spring.application.name)
  ② HTTP 메서드와 경로
  
개발자가 신경 안 써도 되는 것:
  ✅ 현재 인스턴스 IP:Port
  ✅ 인스턴스가 몇 개인지
  ✅ 어느 인스턴스로 보낼지 (로드밸런싱)
  ✅ 인스턴스 추가/제거 시 대응
```

---

## 😱 잘못된 구성

### Before: @LoadBalanced 없이 서비스 이름으로 RestTemplate 사용

```java
// ❌ @LoadBalanced 없는 일반 RestTemplate
@Bean
public RestTemplate restTemplate() {
    return new RestTemplate();  // @LoadBalanced 없음
}

@Service
public class OrderService {
    @Autowired
    private RestTemplate restTemplate;

    public Order getOrder(Long id) {
        // ❌ DNS가 "service-inventory"를 해석하지 못함
        return restTemplate.getForObject(
            "http://service-inventory/inventory/" + id, Order.class);
        // → UnknownHostException: service-inventory
    }
}
```

### Before: 여러 RestTemplate 중 @LoadBalanced 혼용

```java
// ❌ 혼용으로 인한 혼란
@Configuration
public class RestTemplateConfig {

    @Bean
    @LoadBalanced              // 이 Bean만 로드밸런싱됨
    public RestTemplate loadBalancedRestTemplate() {
        return new RestTemplate();
    }

    @Bean                      // 이 Bean은 일반 RestTemplate
    public RestTemplate plainRestTemplate() {
        return new RestTemplate();
    }
}

@Service
public class SomeService {
    @Autowired
    private RestTemplate restTemplate;  // ❌ 어느 Bean이 주입되는지 불명확
    // NoUniqueBeanDefinitionException 또는 의도와 다른 Bean 주입
}
```

---

## ✨ 올바른 패턴

### After: @LoadBalanced RestTemplate 전체 동작 흐름

```
개발자 코드:
  restTemplate.getForObject("http://service-inventory/inventory/1", Item.class)

@LoadBalanced 처리 체인:

  ① LoadBalancerInterceptor.intercept()
     URI에서 "service-inventory" 추출
     → BlockingLoadBalancerClient.execute() 호출

  ② BlockingLoadBalancerClient
     ReactorLoadBalancer.choose("service-inventory") 호출 (블로킹 래핑)
     → 선택된 ServiceInstance 반환 (예: 10.0.1.5:8080)

  ③ ReactorLoadBalancer (RoundRobinLoadBalancer)
     ServiceInstanceListSupplier에서 인스턴스 목록 조회
     → Round Robin으로 선택

  ④ ServiceInstanceListSupplier 체인
     CachingServiceInstanceListSupplier
     → EurekaServiceInstanceListSupplier
        → Eureka 레지스트리에서 "SERVICE-INVENTORY" 인스턴스 목록

  ⑤ URI 치환
     "http://service-inventory/inventory/1"
     → "http://10.0.1.5:8080/inventory/1"

  ⑥ 실제 HTTP 요청 전송
```

---

## 🔬 내부 동작 원리

### 1. @LoadBalanced — RestTemplate에 Interceptor 추가

```java
// LoadBalancerAutoConfiguration.java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
public class LoadBalancerAutoConfiguration {

    // @LoadBalanced가 붙은 RestTemplate Bean들만 수집
    @LoadBalanced
    @Autowired(required = false)
    private List<RestTemplate> restTemplates = Collections.emptyList();

    @Bean
    public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
            final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
        return () -> restTemplateCustomizers.ifAvailable(customizers -> {
            for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
                for (RestTemplateCustomizer customizer : customizers) {
                    // 각 @LoadBalanced RestTemplate에 커스터마이저 적용
                    customizer.customize(restTemplate);
                }
            }
        });
    }

    @Bean
    @ConditionalOnMissingBean
    public LoadBalancerRequestFactory loadBalancerRequestFactory(
            LoadBalancerClient loadBalancerClient) {
        return new LoadBalancerRequestFactory(loadBalancerClient, this.transformers);
    }

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
    static class LoadBalancerInterceptorConfig {

        @Bean
        public LoadBalancerInterceptor loadBalancerInterceptor(
                LoadBalancerClient loadBalancerClient,
                LoadBalancerRequestFactory requestFactory) {
            return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
        }

        @Bean
        @ConditionalOnMissingBean
        public RestTemplateCustomizer restTemplateCustomizer(
                final LoadBalancerInterceptor loadBalancerInterceptor) {
            return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                    restTemplate.getInterceptors());
                // @LoadBalanced RestTemplate에 LoadBalancerInterceptor 추가
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
        }
    }
}
```

### 2. LoadBalancerInterceptor — 서비스 이름 추출 및 치환

```java
// LoadBalancerInterceptor.java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

    private LoadBalancerClient loadBalancer;
    private LoadBalancerRequestFactory requestFactory;

    @Override
    public ClientHttpResponse intercept(
            final HttpRequest request,
            final byte[] body,
            final ClientHttpRequestExecution execution) throws IOException {

        // ① URI에서 서비스 이름 추출
        final URI originalUri = request.getURI();
        String serviceName = originalUri.getHost();
        // "http://service-inventory/inventory/1" → host = "service-inventory"

        Assert.state(serviceName != null,
            "Request URI does not contain a valid hostname: " + originalUri);

        // ② LoadBalancerClient.execute() 호출
        //    내부에서 인스턴스 선택 → URI 치환 → 실제 HTTP 요청
        return this.loadBalancer.execute(serviceName,
            this.requestFactory.createRequest(request, body, execution));
    }
}
```

### 3. BlockingLoadBalancerClient — Reactive → Blocking 브리지

```java
// BlockingLoadBalancerClient.java
// ReactorLoadBalancer(비동기)를 동기 RestTemplate과 연결하는 브리지

public class BlockingLoadBalancerClient implements LoadBalancerClient {

    private final LoadBalancerClientFactory loadBalancerClientFactory;

    @Override
    public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
        // ① ReactorLoadBalancer에서 인스턴스 선택 (비동기 → 블로킹 대기)
        ServiceInstance serviceInstance = choose(serviceId);
        if (serviceInstance == null) {
            throw new IllegalStateException("No instances available for " + serviceId);
        }
        return execute(serviceId, serviceInstance, request);
    }

    @Override
    public ServiceInstance choose(String serviceId) {
        // LoadBalancerClientFactory에서 서비스별 ReactorLoadBalancer 가져오기
        ReactorLoadBalancer<ServiceInstance> loadBalancer =
            this.loadBalancerClientFactory.getInstance(serviceId,
                ReactorServiceInstanceLoadBalancer.class);

        if (loadBalancer == null) {
            return null;
        }

        // Reactive → Blocking: Mono.block()으로 동기 변환
        Response<ServiceInstance> loadBalancerResponse =
            Mono.from(loadBalancer.choose(buildRequest(serviceId))).block();

        if (loadBalancerResponse == null || !loadBalancerResponse.hasServer()) {
            return null;
        }
        return loadBalancerResponse.getServer();
    }

    @Override
    public <T> T execute(String serviceId, ServiceInstance serviceInstance,
            LoadBalancerRequest<T> request) throws IOException {
        try {
            // ② 선택된 인스턴스로 실제 HTTP 요청 실행
            T response = request.apply(serviceInstance);
            // ③ 성능 메트릭 기록
            Object clientResponse = getClientResponse(response);
            supportedLifecycleProcessors.forEach(lifecycle ->
                lifecycle.onComplete(new CompletionContext<>(
                    CompletionContext.Status.SUCCESS, serviceInstance, ...)));
            return response;
        } catch (Exception ex) {
            supportedLifecycleProcessors.forEach(lifecycle ->
                lifecycle.onComplete(new CompletionContext<>(
                    CompletionContext.Status.FAILED, ex, serviceInstance, ...)));
            throw ex;
        }
    }
}
```

### 4. ServiceInstanceListSupplier 체인

```java
// ServiceInstanceListSupplierBuilder (Spring Cloud LoadBalancer)
// ServiceInstanceListSupplier를 체인으로 구성

// 기본 체인 구성 (자동 설정):
//   CachingServiceInstanceListSupplier
//     → HealthCheckServiceInstanceListSupplier (선택적)
//       → EurekaServiceInstanceListSupplier (또는 DiscoveryClientServiceInstanceListSupplier)

// CachingServiceInstanceListSupplier:
// Eureka fetch는 비용이 있음 → 결과를 캐시

public class CachingServiceInstanceListSupplier extends DelegatingServiceInstanceListSupplier {

    // 캐시: 서비스별 인스턴스 목록
    private final Flux<List<ServiceInstance>> serviceInstances;

    public CachingServiceInstanceListSupplier(ServiceInstanceListSupplier delegate,
            ReactiveLoadBalancerProperties.Cache cacheConfig) {
        super(delegate);
        // cache.ttl(기본 35초) 동안 캐시 유지
        // Eureka 레지스트리 fetch 없이 캐시에서 응답
        this.serviceInstances = delegate.get()
            .take(1)
            .cache(Duration.ofSeconds(cacheConfig.getTtl().getSeconds()),
                Schedulers.boundedElastic());
    }

    @Override
    public Flux<List<ServiceInstance>> get() {
        return this.serviceInstances;  // 캐시된 목록 반환
    }
}

// EurekaServiceInstanceListSupplier:
// Eureka 레지스트리에서 실제 인스턴스 목록 조회

public class EurekaServiceInstanceListSupplier extends DelegatingServiceInstanceListSupplier {

    private final EurekaClient eurekaClient;

    @Override
    public Flux<List<ServiceInstance>> get() {
        return Flux.defer(() -> {
            // EurekaClient의 로컬 캐시에서 인스턴스 목록 조회
            List<InstanceInfo> instances = this.eurekaClient
                .getInstancesByVipAddress(getServiceId(), false);
            // InstanceInfo → EurekaServiceInstance 변환
            return Flux.just(instances.stream()
                .filter(i -> i.getStatus() == InstanceStatus.UP)
                .map(EurekaServiceInstance::new)
                .collect(Collectors.toList()));
        });
    }
}
```

### 5. @LoadBalanced WebClient 방식

```java
// WebClient는 Reactive이므로 다른 방식으로 구성

@Configuration
public class WebClientConfig {

    @Bean
    @LoadBalanced  // WebClient.Builder에 LoadBalancer ExchangeFilter 추가
    public WebClient.Builder loadBalancedWebClientBuilder() {
        return WebClient.builder();
    }
}

// 사용
@Service
public class ReactiveOrderService {

    private final WebClient webClient;

    public ReactiveOrderService(@LoadBalanced WebClient.Builder builder) {
        this.webClient = builder.build();
    }

    public Mono<Order> getOrder(Long id) {
        return webClient.get()
            .uri("http://service-order/orders/{id}", id)
            // 내부적으로 ReactorLoadBalancer.choose() 직접 호출
            // 블로킹 없음
            .retrieve()
            .bodyToMono(Order.class);
    }
}

// WebClient LoadBalancer Filter:
// LoadBalancerClientRequestTransformer → URI 치환
// LoadBalancerExchangeFilterFunction.filter() → ReactorLoadBalancer.choose() 호출
```

---

## 💻 실전 구성

### RestTemplate vs WebClient 비교 설정

```java
// RestTemplate (동기, 스레드 블로킹)
@Configuration
public class RestTemplateConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .setConnectTimeout(Duration.ofSeconds(2))
            .setReadTimeout(Duration.ofSeconds(5))
            .build();
    }
}

// WebClient (비동기, 논블로킹) - 권장
@Configuration
public class WebClientConfig {

    @Bean
    @LoadBalanced
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder()
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE);
    }
}
```

### 인스턴스 선택 로그로 동작 확인

```yaml
# LoadBalancer 디버그 로그
logging:
  level:
    org.springframework.cloud.loadbalancer: DEBUG
    com.netflix.loadbalancer: DEBUG
```

```bash
# 로그 확인
# [DEBUG] No LoadBalancer for id [service-inventory]
# [DEBUG] Using existing LoadBalancer for service [service-inventory]
# [DEBUG] 10.0.1.5:8080  ← 선택된 인스턴스
# [DEBUG] 10.0.1.6:8080  ← 다음 요청에서 Round Robin
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 인스턴스 다운 중 요청 처리

```
현재 상태:
  service-inventory 인스턴스 2개:
  10.0.1.5:8080 (UP)
  10.0.1.6:8080 (DOWN - Heartbeat 끊겼지만 아직 레지스트리에 있음)

LoadBalancer의 행동:
  Round Robin: 10.0.1.5 → 10.0.1.6 → 10.0.1.5 → ...

  10.0.1.6에 요청 → Connection Refused
  → IOException 발생
  → 호출자에게 예외 전파

  LoadBalancer는 자동으로 10.0.1.5로 재시도하지 않음!
  → spring-retry + RetryLoadBalancerInterceptor 필요 (Ch4 참고)

가장 현실적인 대응:
  1. Retry (spring-cloud-starter-loadbalancer에 retry 설정)
  2. Circuit Breaker (연속 실패 시 자동 차단 → Ch5)
  3. Actuator를 통한 health check로 빠른 DOWN 감지
```

### 시나리오: CachingServiceInstanceListSupplier TTL 동안 구 목록 사용

```
t=0s:   service-inventory 인스턴스 3개 → 캐시에 3개 저장

t=10s:  service-inventory-2 다운
        Eureka 레지스트리에서 90초 후 제거 예정

t=0~35s: CachingServiceInstanceListSupplier TTL 내
         캐시에 여전히 3개 인스턴스
         → 죽은 인스턴스로 요청 라우팅 가능
         → 약 1/3 요청 실패

t=35s:  캐시 TTL 만료 → EurekaServiceInstanceListSupplier에서 재조회
        → Eureka 로컬 캐시 기준 (아직 3개일 수 있음)
        → Eureka 서버에서 90초 경과 후 레지스트리 제거
        → 이후 클라이언트 fetch에서 2개로 갱신

결론:
  Retry + Circuit Breaker가 없으면
  인스턴스 다운 후 최대 수분간 오류 지속 가능
```

---

## ⚖️ 트레이드오프

| 방식 | 장점 | 단점 |
|------|------|------|
| **@LoadBalanced RestTemplate** | 기존 코드 변경 최소 | 동기/블로킹, Spring WebFlux와 혼용 문제 |
| **@LoadBalanced WebClient** | 논블로킹, Reactor 생태계 통합 | Reactive 학습 곡선 |
| **Feign Client** | 선언적 인터페이스, 최소 코드 | 추가 의존성, 설정 복잡 |

```
CachingServiceInstanceListSupplier TTL 설정:
  spring.cloud.loadbalancer.cache.ttl=35s (기본값)
  
  줄이면:
    빠른 인스턴스 변경 반영
    Eureka 조회 빈도 증가

  늘리면:
    Eureka 조회 감소 (성능 향상)
    다운된 인스턴스 오래 유지

  cache.enabled=false:
    매 요청마다 Eureka 조회
    항상 최신, 성능 저하

  실무: 기본값(35s) 유지 + Retry + Circuit Breaker 조합
```

---

## 📌 핵심 정리

```
@LoadBalanced RestTemplate 동작 핵심:

@LoadBalanced = RestTemplate에 LoadBalancerInterceptor 추가
  → 모든 요청이 인터셉터를 통해 라우팅됨

요청 처리 체인:
  RestTemplate.getForObject("http://service-name/path")
  → LoadBalancerInterceptor: host 추출 ("service-name")
  → BlockingLoadBalancerClient.choose()
  → RoundRobinLoadBalancer: 인스턴스 선택
  → CachingServiceInstanceListSupplier: 캐시된 목록 반환
  → 선택된 IP:Port로 URI 치환
  → 실제 HTTP 요청 전송

ServiceInstanceListSupplier 캐시:
  기본 TTL: 35초
  Eureka 로컬 캐시 + LoadBalancer 캐시 = 2단계 캐시
  → 인스턴스 변경 반영 지연: 최대 수십 초

WebClient(@LoadBalanced):
  ExchangeFilterFunction으로 구현
  ReactorLoadBalancer.choose() 직접 호출 (블로킹 없음)
  → Reactive 환경에서 권장
```

---

## 🤔 생각해볼 문제

**Q1.** `@LoadBalanced RestTemplate`이 주입된 Bean에서 Eureka에 등록되지 않은 외부 API(예: `https://api.github.com/...`)를 호출하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`LoadBalancerInterceptor`는 URI의 host를 서비스 이름으로 간주하여 `loadBalancerClient.choose("api.github.com")`을 호출합니다. Eureka에 `api.github.com`으로 등록된 인스턴스가 없으므로 `choose()`가 null을 반환하고, `IllegalStateException: No instances available for api.github.com`이 발생합니다.

해결 방법:
1. 외부 API 호출용 `@LoadBalanced` 없는 별도 `RestTemplate` Bean 사용
2. `@Qualifier`로 구분하여 주입
3. `spring.cloud.loadbalancer.clients.api-github-com.enabled=false`로 특정 서비스 LB 비활성화

일반적으로 외부 API와 내부 서비스 호출을 별도 Bean으로 분리하는 것이 권장됩니다.

</details>

---

**Q2.** `BlockingLoadBalancerClient.choose()`에서 `Mono.block()`을 호출한다. Project Reactor 환경(Spring WebFlux)에서 이를 사용하면 어떤 문제가 발생하는가?

<details>
<summary>해설 보기</summary>

WebFlux의 이벤트 루프 스레드(Netty의 EventLoop)에서 `Mono.block()`을 호출하면 `IllegalStateException: block()/blockFirst()/blockLast() are blocking, which is not supported in thread...` 예외가 발생합니다. WebFlux는 블로킹 작업이 이벤트 루프를 점유하는 것을 금지합니다.

WebFlux 환경에서는 `@LoadBalanced RestTemplate` 대신 `@LoadBalanced WebClient`를 사용해야 합니다. WebClient의 LoadBalancer 구현은 `ReactorLoadBalancer.choose()`를 비동기로 직접 호출하므로 이벤트 루프를 블로킹하지 않습니다.

</details>

---

**Q3.** `@LoadBalanced RestTemplate`으로 `http://SERVICE-A/api/data`를 호출했을 때 서비스 이름을 대문자로 써야 하는가? 소문자로 써도 동작하는가?

<details>
<summary>해설 보기</summary>

대소문자 구분 없이 동작합니다. `LoadBalancerInterceptor`에서 host를 추출한 후, Eureka 클라이언트의 `getInstancesByVipAddress()` 내부에서 서비스 이름을 대문자로 정규화하여 조회합니다. Eureka는 서비스 이름을 대문자(`APP NAME`)로 저장하므로, 클라이언트에서 소문자(`service-a`)로 요청해도 대문자로 변환되어 매칭됩니다. 관례적으로 소문자 케밥케이스(`service-a`)를 사용하는 것이 읽기 쉽습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Service Instance Metadata](./04-service-instance-metadata.md)** | **[홈으로 🏠](../README.md)** | **[다음: Self-Preservation Mode ➡️](./06-self-preservation-mode.md)**

</div>
