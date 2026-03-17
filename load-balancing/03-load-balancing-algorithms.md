# Load Balancing 알고리즘 — RoundRobin부터 가중치 기반까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `RoundRobinLoadBalancer`가 `AtomicInteger`를 사용하는 이유와 동시성 안전성 보장 방법은?
- `AtomicInteger` 카운터가 오버플로우될 때 어떻게 처리하는가?
- `RandomLoadBalancer`는 단순 랜덤인가, 아니면 가중치를 적용할 수 있는가?
- Spring Cloud LoadBalancer에서 Weighted Response Time 알고리즘을 구현하려면 무엇이 필요한가?
- `ReactorServiceInstanceLoadBalancer` 인터페이스를 구현해 커스텀 알고리즘을 등록하는 전체 과정은?
- `Request` 파라미터를 통해 요청 컨텍스트(헤더, 경로)를 알고리즘 결정에 활용하는 방법은?

---

## 🔍 왜 MSA에서 필요한가

### 알고리즘 선택이 서비스 품질에 미치는 영향

```
3개 인스턴스 상황에서 알고리즘 비교:

인스턴스:
  A: 응답 시간 10ms (고성능 서버)
  B: 응답 시간 100ms (일반 서버)
  C: 응답 시간 500ms (리소스 부족)

Round Robin 적용 시:
  A, B, C, A, B, C, A, B, C, ...
  평균 응답: (10+100+500)/3 = 203ms
  → C가 느려도 1/3 확률로 선택

Weighted Round Robin (가중치: A=10, B=3, C=1):
  A를 자주 선택, C를 드물게 선택
  평균 응답: (10×10 + 100×3 + 500×1)/14 ≈ 54ms
  → 성능에 비례한 트래픽 분배

Weighted Response Time (응답 시간 기반 자동 가중치):
  성능 좋은 인스턴스에 자동으로 더 많은 트래픽
  C가 느릴수록 C로 가는 트래픽 자동 감소
  → 헬스 상태에 따른 동적 조정

알고리즘 선택이 p99 레이턴시를 2-3배 차이 만들 수 있음
```

---

## 😱 잘못된 구성

### Before: 스레드 안전하지 않은 카운터

```java
// ❌ 스레드 안전하지 않은 커스텀 LB 구현
public class UnsafeRoundRobinLoadBalancer implements ReactorServiceInstanceLoadBalancer {
    private int position = 0;  // ❌ 공유 가변 상태, 동기화 없음

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        return serviceInstanceListSupplier.get().next()
            .map(instances -> {
                if (instances.isEmpty()) return new EmptyResponse();
                // ❌ 경쟁 조건: 여러 스레드가 동시에 position 읽기/쓰기
                int pos = position % instances.size();
                position++;  // non-atomic: read-modify-write 경쟁
                return new DefaultResponse(instances.get(pos));
            });
    }
}
```

### Before: 인스턴스 목록 변경 시 AtomicInteger 리셋 없음

```java
// ❌ position이 인스턴스 수보다 클 때 처리 안 됨
// position = 5, instances = 2개
// 5 % 2 = 1 → index 1 (정상)
// 하지만 position이 Integer.MAX_VALUE에 도달하면?
// 다음 incrementAndGet → Integer.MIN_VALUE (음수!)
// -2147483648 % 2 = 0 (Java에서 음수 모듈로는 음수가 나올 수 있음)
// Math.abs(-2147483648) = -2147483648 (오버플로우!)
```

---

## ✨ 올바른 패턴

### After: RoundRobinLoadBalancer 소스 분석

```java
// RoundRobinLoadBalancer.java (Spring Cloud LoadBalancer)
public class RoundRobinLoadBalancer
        implements ReactorServiceInstanceLoadBalancer {

    private static final Log log = LogFactory.getLog(RoundRobinLoadBalancer.class);

    final AtomicInteger position;  // 스레드 안전한 카운터
    final String serviceId;
    ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;

    // position은 랜덤 초기값 사용 (매번 0부터 시작하지 않음)
    // → 모든 인스턴스가 동등하게 선택될 기회
    public RoundRobinLoadBalancer(
            ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider,
            String serviceId) {
        this(serviceInstanceListSupplierProvider, serviceId,
            new Random().nextInt(1000));  // 랜덤 시작점
    }

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier =
            serviceInstanceListSupplierProvider
                .getIfAvailable(NoopServiceInstanceListSupplier::new);

        return supplier.get(request)  // Request 컨텍스트 전달
            .next()                    // Flux → Mono (첫 번째 항목)
            .map(serviceInstances ->
                processInstanceResponse(supplier, serviceInstances));
    }

    private Response<ServiceInstance> processInstanceResponse(
            ServiceInstanceListSupplier supplier,
            List<ServiceInstance> serviceInstances) {

        Response<ServiceInstance> serviceInstanceResponse =
            getInstanceResponse(serviceInstances);

        // 선택 결과를 Supplier에 통보 (통계, 가중치 조정 등)
        if (supplier instanceof SelectedInstanceCallback callback) {
            callback.selectedServiceInstance(serviceInstanceResponse);
        }
        return serviceInstanceResponse;
    }

    private Response<ServiceInstance> getInstanceResponse(
            List<ServiceInstance> instances) {

        if (instances.isEmpty()) {
            log.warn("No servers available for service: " + this.serviceId);
            return new EmptyResponse();
        }

        // ① 원자적 증가 (CAS 연산 → 스레드 안전)
        int pos = this.position.incrementAndGet();

        // ② 음수 및 오버플로우 방어
        //    & Integer.MAX_VALUE: MSB(부호 비트) 제거 → 항상 양수
        //    Integer.MIN_VALUE & Integer.MAX_VALUE = 0 (안전)
        ServiceInstance instance =
            instances.get(Math.abs(pos) % instances.size());

        return new DefaultResponse(instance);
    }
}
```

### RandomLoadBalancer 구현

```java
// RandomLoadBalancer.java
public class RandomLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private static final Log log = LogFactory.getLog(RandomLoadBalancer.class);

    final String serviceId;
    ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier =
            serviceInstanceListSupplierProvider
                .getIfAvailable(NoopServiceInstanceListSupplier::new);

        return supplier.get(request).next()
            .map(serviceInstances ->
                getInstanceResponse(serviceInstances));
    }

    private Response<ServiceInstance> getInstanceResponse(
            List<ServiceInstance> instances) {
        if (instances.isEmpty()) {
            return new EmptyResponse();
        }
        // ThreadLocalRandom: 스레드별 독립 Random → 경쟁 없음
        int idx = ThreadLocalRandom.current().nextInt(instances.size());
        return new DefaultResponse(instances.get(idx));
    }
}
```

### Weighted Response Time — 커스텀 구현

```java
// Spring Cloud LoadBalancer는 Weighted Response Time을 기본 제공하지 않음
// → 직접 구현 필요

@Component
public class WeightedResponseTimeLoadBalancer
        implements ReactorServiceInstanceLoadBalancer {

    private final String serviceId;
    private final ObjectProvider<ServiceInstanceListSupplier> supplierProvider;

    // 인스턴스별 평균 응답 시간 추적 (지수 이동 평균)
    private final ConcurrentHashMap<String, Double> responseTimes =
        new ConcurrentHashMap<>();
    private final double alpha = 0.1;  // EMA 감쇠 계수

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier =
            supplierProvider.getIfAvailable(NoopServiceInstanceListSupplier::new);

        return supplier.get(request).next()
            .map(instances -> {
                if (instances.isEmpty()) return new EmptyResponse();

                // ① 각 인스턴스의 가중치 계산 (응답 시간 역수)
                double totalWeight = 0;
                List<Double> weights = new ArrayList<>();

                for (ServiceInstance instance : instances) {
                    String key = instance.getInstanceId();
                    // 응답 시간 없으면 기본값 100ms
                    double avgTime = responseTimes.getOrDefault(key, 100.0);
                    // 가중치 = 응답 시간의 역수 (빠를수록 높은 가중치)
                    double weight = 1.0 / avgTime;
                    weights.add(weight);
                    totalWeight += weight;
                }

                // ② 가중치 기반 랜덤 선택
                double random = ThreadLocalRandom.current().nextDouble(totalWeight);
                double cumulative = 0;
                for (int i = 0; i < instances.size(); i++) {
                    cumulative += weights.get(i);
                    if (random < cumulative) {
                        return new DefaultResponse(instances.get(i));
                    }
                }

                // 폴백: 마지막 인스턴스
                return new DefaultResponse(
                    instances.get(instances.size() - 1));
            });
    }

    // 호출 완료 후 응답 시간 업데이트 (LifecycleProcessor로 호출됨)
    public void recordResponseTime(String instanceId, long responseTimeMs) {
        responseTimes.merge(instanceId, (double) responseTimeMs,
            (oldVal, newVal) -> alpha * newVal + (1 - alpha) * oldVal);
        // EMA: 최근 응답 시간에 더 높은 가중치
    }
}
```

### Request 컨텍스트 활용 — 헤더 기반 선택

```java
// Request 객체에서 원본 HTTP 요청 컨텍스트 추출
public class HeaderAwareLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        // RequestDataContext: HTTP 요청의 헤더, 쿠키, URL 정보 포함
        RequestDataContext context = (RequestDataContext) request.getContext();
        HttpHeaders headers = context.getClientRequest().getHeaders();

        // X-Region 헤더 기반 지역 선호 라우팅
        String preferredRegion = headers.getFirst("X-Preferred-Region");

        ServiceInstanceListSupplier supplier = ...;
        return supplier.get(request).next()
            .map(instances -> {
                // 선호 지역 인스턴스 먼저 선택 시도
                if (preferredRegion != null) {
                    List<ServiceInstance> regionInstances = instances.stream()
                        .filter(i -> preferredRegion.equals(
                            i.getMetadata().get("region")))
                        .collect(Collectors.toList());

                    if (!regionInstances.isEmpty()) {
                        int idx = ThreadLocalRandom.current()
                            .nextInt(regionInstances.size());
                        return new DefaultResponse(regionInstances.get(idx));
                    }
                }
                // 선호 지역 없으면 전체에서 랜덤
                int idx = ThreadLocalRandom.current().nextInt(instances.size());
                return new DefaultResponse(instances.get(idx));
            });
    }
}
```

---

## 💻 실전 구성

### 서비스별 알고리즘 설정

```java
// 서비스별 로드밸런서 설정 분리
@LoadBalancerClient(name = "order-service",
    configuration = OrderLoadBalancerConfig.class)
@LoadBalancerClient(name = "payment-service",
    configuration = PaymentLoadBalancerConfig.class)
@Configuration
public class GlobalLoadBalancerConfig {}

// order-service: Round Robin (기본 설정)
@Configuration
public class OrderLoadBalancerConfig {
    @Bean
    public ReactorServiceInstanceLoadBalancer orderLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory factory) {
        String serviceId = environment.getProperty(
            LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(
            factory.getLazyProvider(serviceId,
                ServiceInstanceListSupplier.class),
            serviceId);
    }
}

// payment-service: 가중치 기반 (커스텀)
@Configuration
public class PaymentLoadBalancerConfig {
    @Bean
    public ReactorServiceInstanceLoadBalancer paymentLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory factory) {
        String serviceId = environment.getProperty(
            LoadBalancerClientFactory.PROPERTY_NAME);
        return new WeightedResponseTimeLoadBalancer(serviceId,
            factory.getLazyProvider(serviceId,
                ServiceInstanceListSupplier.class));
    }
}
```

### 알고리즘 동작 확인

```bash
# 여러 번 호출해서 Round Robin 확인
for i in {1..6}; do
  curl -s http://service-a:8080/api/echo | jq '.serverIp'
done
# 10.0.1.5, 10.0.1.6, 10.0.1.7, 10.0.1.5, 10.0.1.6, 10.0.1.7
# → 순환 확인

# LoadBalancer 선택 로그 활성화
logging:
  level:
    org.springframework.cloud.loadbalancer.core: TRACE
# [TRACE] Selected instance: 10.0.1.5:8080 (Round Robin, position=1)
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: GC Pause로 인한 응답 지연 인스턴스 처리

```
상황: service-a 인스턴스 3개
  A: 정상 (10ms)
  B: GC Pause로 일시적 200ms 지연
  C: 정상 (12ms)

Round Robin:
  A, B, C, A, B, C ...
  B가 느려도 1/3 확률로 선택 → 전체 p99 영향

Weighted Response Time:
  A: 가중치 1/10 = 0.1
  B: 가중치 1/200 = 0.005 (GC pause 반영됨)
  C: 가중치 1/12 = 0.083

  총 가중치: 0.1 + 0.005 + 0.083 = 0.188
  A 선택 확률: 0.1/0.188 = 53%
  B 선택 확률: 0.005/0.188 = 2.7%  ← 거의 선택 안 됨
  C 선택 확률: 0.083/0.188 = 44%

  B의 GC Pause가 해소되면 자동으로 가중치 회복
  → 동적 상태에 맞는 라우팅 자동 조정
```

---

## ⚖️ 트레이드오프

| 알고리즘 | 공정성 | 성능 최적화 | 구현 복잡도 | 상태 유지 |
|---------|--------|------------|------------|---------|
| **Round Robin** | 완벽 (순환) | 없음 | 최소 | 없음 (AtomicInteger) |
| **Random** | 확률적 | 없음 | 최소 | 없음 |
| **Weighted RR** | 가중치 비례 | 정적 가중치 | 낮음 | 가중치 설정 |
| **Weighted Response Time** | 동적 | 높음 | 높음 | 응답 시간 히스토리 |
| **Least Connections** | 부하 균등 | 높음 | 중간 | 연결 수 카운터 |

```
AtomicInteger 오버플로우 안전 처리:
  pos & Integer.MAX_VALUE 는 비트 AND 연산
  Integer.MAX_VALUE = 0x7FFFFFFF (최상위 부호 비트 0)
  
  어떤 int 값 & 0x7FFFFFFF = 항상 양수 또는 0
  Integer.MIN_VALUE (-2147483648) & 0x7FFFFFFF = 0
  → 오버플로우 후에도 안전하게 동작

  Math.abs(Integer.MIN_VALUE) = Integer.MIN_VALUE (오버플로우!)
  → Math.abs 사용 시 MIN_VALUE 예외 처리 필요
  → Spring이 & MAX_VALUE를 선택한 이유
```

---

## 📌 핵심 정리

```
Load Balancing 알고리즘 핵심:

RoundRobinLoadBalancer:
  AtomicInteger.incrementAndGet() → pos & Integer.MAX_VALUE
  스레드 안전 (CAS 연산)
  오버플로우 안전 (비트 AND)
  랜덤 초기값으로 첫 번째 인스턴스 편향 방지

RandomLoadBalancer:
  ThreadLocalRandom.current().nextInt(size)
  스레드별 독립 Random (경쟁 없음)
  단순하고 공정한 분배

커스텀 알고리즘 구현:
  ReactorServiceInstanceLoadBalancer 인터페이스 구현
  choose(Request) → Mono<Response<ServiceInstance>>
  ServiceInstanceListSupplier에서 인스턴스 목록 받아 선택

Request 컨텍스트 활용:
  RequestDataContext → HttpHeaders, URL 접근
  헤더/쿠키/경로 기반 라우팅 결정 가능

서비스별 격리:
  @LoadBalancerClient(name = "service-name", configuration = Config.class)
  → LoadBalancerClientFactory의 Child Context에 Bean 등록
  → 서비스마다 다른 알고리즘 독립적으로 적용
```

---

## 🤔 생각해볼 문제

**Q1.** `RoundRobinLoadBalancer`의 `AtomicInteger position`은 여러 요청이 동시에 같은 인스턴스를 선택하는 상황이 가능한가? 어떤 경우에 발생하는가?

<details>
<summary>해설 보기</summary>

이론상으로는 항상 다른 위치를 선택합니다. `AtomicInteger.incrementAndGet()`은 원자적 연산이므로 두 스레드가 동시에 호출해도 각각 다른 값을 받습니다. 하지만 `pos % instances.size()`에서 두 연속된 값이 같은 인덱스를 가리킬 수는 없습니다(같은 나머지가 나오려면 size만큼 차이가 나야 함).

그러나 예외적 상황이 있습니다. `position`이 오버플로우 직전(`Integer.MAX_VALUE`)에서 여러 스레드가 `incrementAndGet()`을 하면 음수 → `& Integer.MAX_VALUE` 처리 → 0, 1, 2, ... 로 리셋되어 짧은 시간 동안 낮은 인덱스 인스턴스들이 연속 선택될 수 있습니다. 실용적으로는 문제가 되지 않지만, 이것이 `position`을 0이 아닌 랜덤 초기값으로 시작하게 하는 이유 중 하나입니다.

</details>

---

**Q2.** Weighted Response Time 알고리즘에서 새로 추가된 인스턴스(응답 시간 기록 없음)는 어떻게 처리해야 하는가?

<details>
<summary>해설 보기</summary>

새 인스턴스에 기본값(예: 100ms)을 설정하는 것이 일반적입니다. 하지만 기본값 선택이 중요합니다.

- 기본값이 너무 높으면(1000ms): 새 인스턴스가 처음에 거의 선택되지 않아 워밍업이 느립니다.
- 기본값이 너무 낮으면(1ms): 새 인스턴스로 트래픽이 갑자기 몰려 Cold Start 문제(JVM 워밍업, 커넥션 풀 초기화)로 성능이 저하될 수 있습니다.

실무적 접근:
1. **Gradual Warm-up**: 초기 기본값을 다른 인스턴스 평균 응답 시간으로 설정하고 트래픽이 쌓이면서 EMA로 실제값으로 수렴
2. **별도 워밍업 단계**: 인스턴스가 UP 상태가 되더라도 일정 시간(`OUT_OF_SERVICE`나 낮은 가중치)을 두고 점진적으로 트래픽 증가
3. **최소 트래픽 보장**: 응답 시간이 아무리 높아도 최소한의 선택 확률(예: 1%)은 유지해 지속적으로 상태 측정

</details>

---

**Q3.** 인스턴스 목록이 3개였다가 2개로 줄었을 때 `RoundRobinLoadBalancer`의 `position`은 리셋되지 않는다. `position=5`, 인스턴스 2개일 때 선택되는 인스턴스 인덱스는 무엇인가?

<details>
<summary>해설 보기</summary>

`5 % 2 = 1`이므로 인덱스 1(두 번째 인스턴스)이 선택됩니다. `position`이 리셋되지 않아도 `% instances.size()` 연산으로 항상 유효한 인덱스가 반환됩니다. 인스턴스 수가 줄어도 Round Robin은 올바르게 동작합니다.

다만 단기적으로 편향이 발생할 수 있습니다. 예를 들어 인스턴스 3개에서 `position=0,1,2,3,...`으로 순환하다가 2개로 줄면 `0%2=0, 1%2=1, 2%2=0, 3%2=1, ...`으로 0과 1이 균등하게 선택됩니다. 이전 3개 시절의 position 값이 어떻든 간에 이후부터는 2개 기준으로 올바르게 순환합니다. Spring의 구현이 position을 리셋하지 않는 것은 의도적입니다 — 리셋 시 AtomicInteger에 추가 동기화가 필요하고 실용적 이점이 없기 때문입니다.

</details>

---

<div align="center">

**[⬅️ 이전: LoadBalancerClient 동작 원리](./02-load-balancer-client-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: Retry 전략 ➡️](./04-retry-strategy.md)**

</div>
