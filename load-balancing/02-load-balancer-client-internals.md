# LoadBalancerClient 동작 원리 — 서비스 이름이 실제 IP:Port로 바뀌는 전체 흐름

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `LoadBalancerInterceptor.intercept()`에서 서비스 이름을 추출하고 URI를 치환하는 정확한 코드 경로는?
- `BlockingLoadBalancerClient.choose()`가 `ReactorLoadBalancer`를 Blocking으로 래핑하는 이유와 한계는?
- `ServiceInstanceListSupplier` 체인에서 각 Supplier가 담당하는 역할은 무엇인가?
- `EurekaServiceInstanceListSupplier`가 Eureka 클라이언트의 로컬 캐시에서 인스턴스를 가져오는 방법은?
- `CachingServiceInstanceListSupplier`의 캐시는 어떤 자료구조로 구현되며 만료 시 어떻게 동작하는가?
- `ServiceInstance`를 선택한 후 URI를 재구성하는 `reconstructURI()` 메서드의 역할은?

---

## 🔍 왜 MSA에서 필요한가

### 호출 흐름이 복잡한 이유

```
단순해 보이는 코드:
  restTemplate.getForObject("http://order-service/orders/1", Order.class)

내부에서 일어나는 일:
  ① URI에서 "order-service" 추출
  ② "order-service" 인스턴스 목록 조회
     → Eureka 로컬 캐시 접근
     → 캐시 없으면 Eureka 서버 fetch
  ③ 알고리즘으로 인스턴스 하나 선택
     → 10.0.1.5:8080
  ④ URI 재조립
     → "http://10.0.1.5:8080/orders/1"
  ⑤ 실제 HTTP 요청 전송

이 과정에서:
  캐시 레이어 2단계 (LoadBalancer 캐시 + Eureka 캐시)
  Reactive → Blocking 변환 (RestTemplate 호환성)
  URI 재구성 (스킴, 호스트, 포트 조합)
  
  각 단계를 이해해야 성능 튜닝 및 장애 디버깅 가능
```

---

## 😱 잘못된 구성

### Before: 서비스 이름 대소문자 혼용으로 인한 불일치

```java
// ❌ 대소문자 불일치
// Eureka에 등록된 이름: "ORDER-SERVICE" (대문자)

restTemplate.getForObject(
    "http://Order-Service/orders/1",  // 혼합 케이스
    Order.class);
// → LoadBalancerInterceptor: host = "Order-Service"
// → choose("Order-Service") 호출
// → EurekaServiceInstanceListSupplier:
//    getInstancesByVipAddress("Order-Service") → 대문자 변환 후 조회
// → 실제로는 동작하지만 케이스 의존 → 명시적으로 소문자 권장
```

### Before: WebFlux 이벤트 루프에서 BlockingLoadBalancerClient 호출

```java
// ❌ WebFlux Controller에서 RestTemplate(@LoadBalanced) 사용
@RestController
public class ReactiveController {

    @Autowired
    private RestTemplate restTemplate;  // @LoadBalanced

    @GetMapping("/proxy")
    public Mono<Order> proxy() {
        return Mono.fromCallable(() ->
            // ❌ Mono.fromCallable 내부지만 subscribeOn 없으면 이벤트 루프 스레드
            restTemplate.getForObject("http://order-service/orders/1", Order.class)
            // BlockingLoadBalancerClient.choose()에서 block() 호출
            // → 이벤트 루프 스레드 블로킹 → 성능 저하 또는 예외
        );
        // 해결: .subscribeOn(Schedulers.boundedElastic()) 추가
    }
}
```

---

## ✨ 올바른 패턴

### After: 전체 호출 체인 시각화

```
RestTemplate.getForObject("http://order-service/orders/1")
  │
  ▼ ClientHttpRequestInterceptor 체인
LoadBalancerInterceptor.intercept()
  │  host 추출: "order-service"
  │
  ▼
BlockingLoadBalancerClient.execute(serviceId, request)
  │
  ▼ ServiceId로 Child Context에서 LoadBalancer 조회
ReactorLoadBalancer = RoundRobinLoadBalancer
  │
  ▼ choose(Request) → Mono<Response<ServiceInstance>>
ServiceInstanceListSupplier 체인:
  CachingServiceInstanceListSupplier (TTL: 35s)
    ↓ 캐시 미스 or 만료
  HealthCheckServiceInstanceListSupplier (선택적)
    ↓
  EurekaServiceInstanceListSupplier
    ↓ Eureka Client 로컬 캐시
  [10.0.1.5:8080, 10.0.1.6:8080, 10.0.1.7:8080]
  │
  ▼ RoundRobin 선택
ServiceInstance: 10.0.1.5:8080
  │
  ▼ URI 재구성
reconstructURI(instance, originalURI)
  → "http://10.0.1.5:8080/orders/1"
  │
  ▼ 실제 HTTP 요청 전송
HttpResponse(200 OK, Body: {...})
```

---

## 🔬 내부 동작 원리

### 1. LoadBalancerInterceptor — 진입점

```java
// LoadBalancerInterceptor.java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

    private LoadBalancerClient loadBalancer;
    private LoadBalancerRequestFactory requestFactory;

    @Override
    public ClientHttpResponse intercept(final HttpRequest request,
            final byte[] body,
            final ClientHttpRequestExecution execution) throws IOException {

        // ① URI에서 host = 서비스 이름 추출
        final URI originalUri = request.getURI();
        String serviceName = originalUri.getHost();
        // "http://order-service/orders/1" → "order-service"

        Assert.state(serviceName != null,
            "Request URI does not contain a valid hostname: " + originalUri);

        // ② LoadBalancerClient에 위임
        //    내부에서 인스턴스 선택 + URI 치환 + 실제 HTTP 요청
        return this.loadBalancer.execute(serviceName,
            this.requestFactory.createRequest(request, body, execution));
    }
}
```

### 2. BlockingLoadBalancerClient — 인스턴스 선택과 실행

```java
// BlockingLoadBalancerClient.java
public class BlockingLoadBalancerClient implements LoadBalancerClient {

    private final LoadBalancerClientFactory loadBalancerClientFactory;

    @Override
    public <T> T execute(String serviceId, LoadBalancerRequest<T> request)
            throws IOException {
        // ① ReactorLoadBalancer로부터 인스턴스 선택
        ServiceInstance serviceInstance = choose(serviceId);
        if (serviceInstance == null) {
            throw new IllegalStateException(
                "No instances available for " + serviceId);
        }
        // ② 선택된 인스턴스로 실제 HTTP 요청 실행
        return execute(serviceId, serviceInstance, request);
    }

    @Override
    public ServiceInstance choose(String serviceId) {
        // ③ Child ApplicationContext에서 서비스별 LoadBalancer 조회
        ReactorLoadBalancer<ServiceInstance> loadBalancer =
            this.loadBalancerClientFactory.getInstance(
                serviceId, ReactorServiceInstanceLoadBalancer.class);

        if (loadBalancer == null) { return null; }

        // ④ Reactive → Blocking 변환
        //    Servlet 환경(RestTemplate 스레드)에서만 안전
        //    Netty 이벤트 루프 스레드에서 절대 호출 금지!
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
            // ⑤ request.apply(serviceInstance):
            //    LoadBalancerRequestFactory가 생성한 래퍼
            //    내부에서 reconstructURI() 호출 후 실제 HTTP 전송
            T response = request.apply(serviceInstance);

            // ⑥ 성공/실패 메트릭 기록
            Object clientResponse = getClientResponse(response);
            supportedLifecycleProcessors.forEach(lifecycle ->
                lifecycle.onComplete(new CompletionContext<>(
                    CompletionContext.Status.SUCCESS,
                    serviceInstance,
                    buildRequestData(request),
                    clientResponse)));
            return response;
        } catch (Exception ex) {
            supportedLifecycleProcessors.forEach(lifecycle ->
                lifecycle.onComplete(new CompletionContext<>(
                    CompletionContext.Status.FAILED,
                    ex, serviceInstance,
                    buildRequestData(request))));
            throw ex;
        }
    }

    // URI 재구성: 서비스 이름 → 실제 IP:Port
    @Override
    public URI reconstructURI(ServiceInstance serviceInstance, URI original) {
        return LoadBalancerUriTools.reconstructURI(serviceInstance, original);
    }
}
```

### 3. ServiceInstanceListSupplier 체인 상세

```java
// ① CachingServiceInstanceListSupplier
// 가장 바깥 레이어 — 인스턴스 목록을 캐시
public class CachingServiceInstanceListSupplier
        extends DelegatingServiceInstanceListSupplier {

    // Caffeine 또는 기본 캐시
    private final LoadBalancerCache<String, List<ServiceInstance>> cache;
    private final String serviceId;

    @Override
    public Flux<List<ServiceInstance>> get() {
        return Flux.defer(() -> {
            // 캐시 키: serviceId
            List<ServiceInstance> cached = this.cache.get(serviceId);
            if (cached != null) {
                return Flux.just(cached);  // 캐시 히트
            }
            // 캐시 미스: 하위 Supplier에서 로드
            return getDelegate().get()
                .take(1)
                .map(instances -> {
                    // TTL과 함께 캐시 저장
                    this.cache.put(serviceId, instances);
                    return instances;
                });
        });
    }

    // RefreshCacheEvent: 캐시 강제 무효화
    @EventListener
    public void onCacheRefresh(RefreshCacheEvent event) {
        this.cache.evict(serviceId);
    }
}

// ② EurekaServiceInstanceListSupplier
// Eureka Client 로컬 캐시에서 인스턴스 조회
public class EurekaServiceInstanceListSupplier
        extends DelegatingServiceInstanceListSupplier {

    private final EurekaClient eurekaClient;
    private final EurekaClientConfig clientConfig;

    @Override
    public Flux<List<ServiceInstance>> get() {
        return Flux.defer(() -> {
            // EurekaClient의 로컬 캐시에서 조회 (네트워크 요청 없음)
            // Eureka Client가 30초마다 서버에서 delta fetch하여 캐시 갱신
            List<InstanceInfo> instances = Optional.ofNullable(
                eurekaClient.getInstancesByVipAddress(
                    getServiceId(), false))  // false = non-secure
                .orElse(Collections.emptyList());

            return Flux.just(instances.stream()
                // InstanceStatus.UP인 것만 반환
                .filter(i -> i.getStatus() == InstanceStatus.UP)
                // InstanceInfo → EurekaServiceInstance 변환
                .map(i -> new EurekaServiceInstance(i))
                .collect(Collectors.toList()));
        });
    }
}

// ③ DiscoveryClientServiceInstanceListSupplier
// Spring의 DiscoveryClient 추상화 사용 (Eureka 외 다른 Discovery도 지원)
public class DiscoveryClientServiceInstanceListSupplier
        extends DelegatingServiceInstanceListSupplier {

    private final DiscoveryClient discoveryClient;

    @Override
    public Flux<List<ServiceInstance>> get() {
        return Flux.defer(() ->
            Flux.just(discoveryClient.getInstances(getServiceId())));
    }
}
```

### 4. reconstructURI — URI 재구성

```java
// LoadBalancerUriTools.reconstructURI()
public static URI reconstructURI(ServiceInstance instance, URI original) {
    // ServiceInstance: { host: "10.0.1.5", port: 8080, secure: false }
    // original: "http://order-service/orders/1"

    String host = instance.getHost();      // "10.0.1.5"
    int port = instance.getPort();          // 8080
    String scheme = instance.isSecure()    // "http" or "https"
        ? "https" : "http";

    // 결과: "http://10.0.1.5:8080/orders/1"
    return UriComponentsBuilder.fromUri(original)
        .scheme(scheme)
        .host(host)
        .port(port)
        .build(true)  // encoded: URI 인코딩 유지
        .toUri();
}
```

### 5. LoadBalancerClientFactory — 서비스별 Context 관리

```java
// LoadBalancerClientFactory.java
// 서비스별 독립적인 Child ApplicationContext 관리
public class LoadBalancerClientFactory
        extends NamedContextFactory<LoadBalancerClientSpecification>
        implements ReactiveLoadBalancer.Factory<ServiceInstance> {

    // contexts: Map<serviceId, ApplicationContext>
    // 각 서비스별로 독립적인 Bean 공간 → 설정 격리

    @Override
    public <T> T getInstance(String serviceId, Class<T> type) {
        // serviceId에 해당하는 Child Context에서 Bean 조회
        // Child Context 없으면 생성 (Lazy Initialization)
        AnnotationConfigApplicationContext context = getContext(serviceId);
        return context.getBean(type);
    }

    // 기본 설정: DefaultLoadBalancerConfiguration
    // → RoundRobinLoadBalancer Bean 생성
    // → CachingServiceInstanceListSupplier Bean 생성
    // → (Eureka 있으면) EurekaServiceInstanceListSupplier Bean 생성
}
```

---

## 💻 실전 구성

### 캐시 계층별 갱신 주기 설정

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        enabled: true
        ttl: 35s              # LoadBalancer 캐시 TTL
        capacity: 256         # 캐시 최대 항목 수

eureka:
  client:
    registry-fetch-interval-seconds: 30  # Eureka 로컬 캐시 갱신 주기

# 총 캐시 지연: LB 캐시(35s) + Eureka 캐시(30s) = 최대 65초
# 인스턴스 변경 → 최대 65초 후 LoadBalancer가 인식
```

### 호출 체인 디버깅

```yaml
logging:
  level:
    org.springframework.cloud.loadbalancer: TRACE
    com.netflix.discovery: DEBUG
```

```bash
# 인스턴스 목록 확인 (Actuator)
curl http://service-a:8080/actuator/loadbalancer
# 응답:
# {
#   "instances": {
#     "order-service": [
#       {"serviceId": "order-service", "host": "10.0.1.5", "port": 8080, "secure": false},
#       {"serviceId": "order-service", "host": "10.0.1.6", "port": 8080, "secure": false}
#     ]
#   }
# }

# 캐시 강제 갱신 (인스턴스 변경 후 즉시 반영)
curl -X POST http://service-a:8080/actuator/loadbalancer/refresh
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 인스턴스 다운 후 캐시 갱신 타이밍

```
t=0s:   order-service 인스턴스 2개 → 캐시에 저장 (TTL: 35s)
t=10s:  인스턴스 하나 다운 (OOM Kill)
t=10~40s: LB 캐시 유효 기간 → 다운된 인스턴스도 목록에 있음
           → 50% 요청이 다운 인스턴스로 → ConnectException
t=35s:  LB 캐시 TTL 만료 → Eureka 로컬 캐시 재조회
         → Eureka: 아직 다운 인스턴스 있음 (90초 만료 안 됨)
         → LB 캐시: 두 인스턴스 모두 있음

t=90s:  Eureka: 인스턴스 만료 → 레지스트리에서 제거
t=90s~125s: Eureka 클라이언트: 30초 후 fetch → 로컬 캐시 갱신
t=125s~160s: LB 캐시 TTL 만료 후 재조회 → 정상 인스턴스만 남음

결론:
  인스턴스 다운 후 최대 160초간 오류 가능
  → Retry가 필수 (다운 인스턴스 실패 → 즉시 다른 인스턴스 재시도)
```

---

## ⚖️ 트레이드오프

| 캐시 설정 | 응답 속도 | 최신성 | 적합한 환경 |
|----------|----------|--------|------------|
| `ttl=0` (비활성) | 낮음 | 최고 | 테스트, 인스턴스 변동 잦음 |
| `ttl=35s` (기본) | 높음 | 보통 | 프로덕션 일반 |
| `ttl=300s` | 최고 | 낮음 | 인스턴스 변동 거의 없음 |

```
두 캐시 레이어의 의미:
  Eureka 로컬 캐시 (30s 갱신):
    Eureka Server와 통신 빈도 감소
    Discovery 서버 부하 감소

  LoadBalancer 캐시 (35s TTL):
    Eureka 로컬 캐시 조회 빈도 감소
    ServiceInstanceListSupplier 체인 호출 최소화
    요청 처리 스레드에서 캐시 조회 빠름

  두 캐시를 별도로 조정:
    변경 빠른 반영: Eureka 30s → 15s, LB 35s → 10s
    성능 최적화: Eureka 30s 유지, LB 300s로 증가
```

---

## 📌 핵심 정리

```
LoadBalancerClient 동작 핵심:

호출 체인:
  LoadBalancerInterceptor → BlockingLoadBalancerClient
  → LoadBalancerClientFactory (서비스별 Child Context)
  → ReactorLoadBalancer.choose()
  → ServiceInstanceListSupplier 체인
    CachingSupplier → EurekaSupplier → [인스턴스 목록]
  → 선택된 ServiceInstance
  → reconstructURI() → 실제 URI
  → HTTP 요청 전송

캐시 2단계:
  ① LB 캐시 (CachingServiceInstanceListSupplier, TTL 35s)
  ② Eureka 로컬 캐시 (30s 갱신)
  최대 지연: 35 + 30 = 65초

reconstructURI:
  ServiceInstance.host + port + scheme
  + original URI의 path + query
  = 완전한 실제 URI

서비스별 격리:
  LoadBalancerClientFactory → 서비스마다 Child Context
  서비스별 다른 알고리즘/설정 가능
  @LoadBalancerClient(name, configuration) 어노테이션
```

---

## 🤔 생각해볼 문제

**Q1.** `BlockingLoadBalancerClient.choose()` 내부의 `Mono.from(...).block()`이 `null`을 반환하는 경우는 어떤 상황인가?

<details>
<summary>해설 보기</summary>

두 가지 경우에 `null`이 반환됩니다.

1. **인스턴스 없음**: `ServiceInstanceListSupplier`가 빈 목록을 반환하거나 Eureka에 해당 서비스가 등록되지 않은 경우. `RoundRobinLoadBalancer`는 빈 목록에서 `EmptyResponse()`를 반환하고 `loadBalancerResponse.hasServer()`가 false가 됩니다.

2. **LoadBalancer 없음**: `loadBalancerClientFactory.getInstance(serviceId, ...)`가 null을 반환하는 경우(해당 서비스 ID에 대한 LoadBalancer 설정이 없는 경우).

`choose()`가 null을 반환하면 `execute()`에서 `IllegalStateException: No instances available for {serviceId}`가 발생합니다. 이는 Eureka가 완전히 다운됐거나 서비스가 아직 등록되지 않은 상황에서 발생합니다.

</details>

---

**Q2.** `LoadBalancerClientFactory`가 생성하는 Child ApplicationContext는 언제 소멸되는가?

<details>
<summary>해설 보기</summary>

Child ApplicationContext는 명시적으로 닫히거나 부모 ApplicationContext가 닫힐 때 소멸됩니다. 일반적으로 서비스 종료 시(`@PreDestroy`, JVM Shutdown Hook)에 부모 Context가 닫히며 자동으로 Child Context도 닫힙니다. 런타임에 특정 서비스의 Child Context를 재생성해야 한다면 `NamedContextFactory.destroy(serviceId)`를 호출해 해당 서비스의 Context를 제거하고 다음 요청 시 재생성하게 할 수 있습니다. 이는 로드밸런서 설정을 런타임에 변경할 때 유용합니다.

</details>

---

**Q3.** `reconstructURI`가 `http://order-service/orders/1`에서 `order-service`를 실제 IP:Port로 바꾼다고 했다. 그런데 `order-service` 서비스가 HTTPS를 사용한다면 URI 스킴은 어떻게 결정되는가?

<details>
<summary>해설 보기</summary>

`ServiceInstance.isSecure()` 메서드의 반환값에 따라 스킴이 결정됩니다. Eureka에서는 인스턴스의 `securePort.@enabled`가 `true`이면 `isSecure() = true`가 되어 `https`를 사용합니다. Eureka 클라이언트 설정:

```yaml
eureka:
  instance:
    secure-port-enabled: true
    secure-port: 8443
    non-secure-port-enabled: false
```

또는 스프링 설정에서:
```yaml
spring:
  cloud:
    loadbalancer:
      use-raw-status-codes-in-response-data: false
```

클라이언트에서 원본 URI의 스킴(`http://`)이 선언됐어도 `ServiceInstance.isSecure()`가 true면 `https`로 바뀝니다. 이 동작이 의도치 않다면 `reconstructURI()`를 오버라이드하는 커스텀 `LoadBalancerClient`를 구현할 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Ribbon vs Spring Cloud LoadBalancer](./01-ribbon-vs-spring-cloud-lb.md)** | **[홈으로 🏠](../README.md)** | **[다음: Load Balancing 알고리즘 ➡️](./03-load-balancing-algorithms.md)**

</div>
