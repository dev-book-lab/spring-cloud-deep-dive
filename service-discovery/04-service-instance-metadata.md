# Service Instance Metadata — 인스턴스에 라우팅 힌트를 심는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `EurekaInstanceConfigBean.metadataMap`에 저장된 값은 어디에 쓰이는가?
- `preferIpAddress: true` 없이 컨테이너 환경에서 무슨 문제가 발생하는가?
- 메타데이터로 카나리 배포를 구현할 때 LoadBalancer Filter 코드는 어떻게 작성하는가?
- `instance-id`를 커스터마이징하지 않으면 다중 인스턴스 환경에서 어떤 문제가 생기는가?
- Zone-Aware 라우팅에서 `zone` 메타데이터가 어떻게 활용되는가?
- Actuator `/actuator/serviceregistry`로 인스턴스 상태를 런타임에 변경하는 방법은?

---

## 🔍 왜 MSA에서 필요한가

### 단순 IP:Port 이상의 정보가 필요한 이유

```
기본 Service Discovery로 알 수 있는 것:
  SERVICE-A의 인스턴스 주소 목록
  → [10.0.1.5:8080, 10.0.1.6:8080, 10.0.1.7:8080]

이것만으로 부족한 케이스:

카나리 배포:
  10.0.1.5 = v1.0 (90% 트래픽)
  10.0.1.6 = v2.0 (10% 트래픽, 새 버전 검증 중)
  → "이 인스턴스가 어느 버전인지" 정보가 필요

Zone-Aware 라우팅:
  10.0.1.5 = ap-northeast-2a AZ
  10.0.1.6 = ap-northeast-2c AZ
  → "같은 AZ의 인스턴스를 우선 호출" (레이턴시 최소화, 데이터 전송 비용 절감)

A/B 테스트:
  10.0.1.5 = 실험군 (feature-flag: new-algorithm)
  10.0.1.6 = 대조군
  → 특정 사용자 그룹은 실험군 인스턴스로만 라우팅

메타데이터(metadataMap):
  이 "추가 정보"를 Eureka 인스턴스에 심는 메커니즘
  LoadBalancer Filter에서 이 메타데이터를 읽어 라우팅 결정
```

---

## 😱 잘못된 구성

### Before: preferIpAddress 없이 컨테이너 환경에서 사용

```yaml
# ❌ 컨테이너(Docker/K8s) 환경에서 기본 설정
eureka:
  instance:
    prefer-ip-address: false  # 기본값
    # hostname으로 등록됨
```

```
문제:
  Docker 컨테이너 hostname: "a1b2c3d4e5f6" (컨테이너 ID)
  Eureka에 등록: hostname = "a1b2c3d4e5f6"
  
  클라이언트가 이 hostname으로 연결 시도:
  → DNS 조회: "a1b2c3d4e5f6" → 해석 실패
  → 연결 불가!
  
  K8s Pod hostname: "service-a-7d8f9b6c5-xk2pq"
  같은 클러스터 내에서는 DNS 해석 가능하지만
  클러스터 외부에서 접근 시 동일한 문제
  
해결: prefer-ip-address: true
  → IP 주소 직접 등록
  → 어느 환경에서나 연결 가능
```

### Before: instance-id 기본값으로 다중 인스턴스 중복

```yaml
# ❌ instance-id를 설정하지 않은 경우
eureka:
  instance:
    # 기본 instance-id: ${spring.cloud.client.hostname}:${spring.application.name}:${server.port}
```

```
Docker Compose에서 같은 이름의 서비스 2개 실행:
  service-a-1: hostname=container1, port=8080
  service-a-2: hostname=container2, port=8080

  기본 instance-id:
  container1:service-a:8080
  container2:service-a:8080
  → 서로 다른 ID, 정상 등록

  그런데 K8s에서 같은 포트:
  Pod1: hostname=service-a-pod1, port=8080
  Pod2: hostname=service-a-pod2, port=8080
  → 역시 다르지만...

  prefer-ip-address: true + instance-id 미설정:
  IP1:service-a:8080 → "10.0.1.5:service-a:8080"
  IP2:service-a:8080 → "10.0.1.6:service-a:8080"
  → IP가 고유하므로 일반적으로 OK

  문제 발생 케이스:
  같은 호스트에서 2개 인스턴스 실행 (개발 환경):
  10.0.1.5:service-a:8080 (중복!)
  → 하나가 다른 하나를 덮어씀
```

---

## ✨ 올바른 패턴

### After: 컨테이너 환경 완전한 메타데이터 설정

```yaml
# application.yml
eureka:
  instance:
    # 컨테이너 환경 필수: IP로 등록
    prefer-ip-address: true
    
    # 고유한 instance-id: IP + 앱이름 + 포트
    instance-id: ${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}
    
    # 비즈니스 메타데이터
    metadata-map:
      version: "v2.1.0"           # 버전 라우팅용
      zone: "ap-northeast-2a"     # Zone-Aware 라우팅용
      weight: "10"                 # 가중치 (카나리: 낮게)
      canary: "true"               # 카나리 인스턴스 식별자
    
    # Health Check URL (Actuator)
    health-check-url-path: /actuator/health
    status-page-url-path: /actuator/info
```

---

## 🔬 내부 동작 원리

### 1. EurekaInstanceConfigBean — 메타데이터 설정

```java
// EurekaInstanceConfigBean.java (Spring Cloud)
// application.yml의 eureka.instance.* 바인딩

@ConfigurationProperties("eureka.instance")
public class EurekaInstanceConfigBean implements CloudEurekaInstanceConfig {

    // eureka.instance.prefer-ip-address
    private boolean preferIpAddress = false;

    // eureka.instance.instance-id
    // 기본값: ${spring.cloud.client.hostname}:${spring.application.name}:${server.port}
    private String instanceId;

    // eureka.instance.metadata-map
    // Key-Value 자유 형식
    private Map<String, String> metadataMap = new HashMap<>();

    // preferIpAddress: true이면 hostname 대신 IP 반환
    @Override
    public String getHostName(boolean refresh) {
        if (this.preferIpAddress) {
            return this.ipAddress;  // InetAddress.getLocalHost().getHostAddress()
        }
        return this.hostname;
    }
}
```

### 2. InstanceInfo.metadataMap — Eureka 레지스트리에 저장

```java
// 클라이언트가 register() 호출 시 전송하는 InstanceInfo JSON 예시
{
  "instance": {
    "instanceId": "10.0.1.5:service-a:8080",
    "hostName": "10.0.1.5",
    "app": "SERVICE-A",
    "ipAddr": "10.0.1.5",
    "status": "UP",
    "port": {"$": 8080, "@enabled": "true"},
    "metadata": {
      "version": "v2.1.0",
      "zone": "ap-northeast-2a",
      "weight": "10",
      "canary": "true",
      "management.port": "8080"  // Spring Boot Actuator 자동 추가
    },
    "leaseInfo": {
      "renewalIntervalInSecs": 30,
      "durationInSecs": 90
    }
  }
}
```

### 3. 메타데이터 기반 카나리 LoadBalancer 구현

```java
// CanaryLoadBalancer.java
// 메타데이터의 'canary' 키를 읽어 트래픽 분배

@Component
public class CanaryLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private final String serviceId;
    private final ObjectProvider<ServiceInstanceListSupplier> supplierObjectProvider;
    private final Random random = new Random();

    // 카나리 트래픽 비율 (10%)
    private static final double CANARY_TRAFFIC_RATIO = 0.1;

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier = supplierObjectProvider
            .getIfAvailable(NoopServiceInstanceListSupplier::new);

        return supplier.get(request).next().map(instances -> {
            if (instances.isEmpty()) {
                return new EmptyResponse();
            }

            // ① 인스턴스를 카나리 / 일반으로 분리
            List<ServiceInstance> canaryInstances = instances.stream()
                .filter(i -> "true".equals(i.getMetadata().get("canary")))
                .collect(Collectors.toList());

            List<ServiceInstance> normalInstances = instances.stream()
                .filter(i -> !"true".equals(i.getMetadata().get("canary")))
                .collect(Collectors.toList());

            // ② 확률에 따라 카나리 또는 일반 풀에서 선택
            List<ServiceInstance> pool;
            if (!canaryInstances.isEmpty() && random.nextDouble() < CANARY_TRAFFIC_RATIO) {
                pool = canaryInstances;   // 10% → 카나리
            } else {
                pool = normalInstances.isEmpty() ? instances : normalInstances;
            }

            // ③ 선택된 풀에서 라운드 로빈
            int index = Math.abs(counter.incrementAndGet()) % pool.size();
            return new DefaultResponse(pool.get(index));
        });
    }
}

// LoadBalancer Bean 등록 (서비스별 격리)
@Configuration
@LoadBalancerClient(name = "service-a", configuration = ServiceALoadBalancerConfig.class)
public class LoadBalancerConfig {}

@Configuration
public class ServiceALoadBalancerConfig {
    @Bean
    public ReactorServiceInstanceLoadBalancer loadBalancer(
            Environment environment,
            LoadBalancerClientFactory factory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new CanaryLoadBalancer(name,
            factory.getLazyProvider(name, ServiceInstanceListSupplier.class));
    }
}
```

### 4. Zone-Aware 라우팅

```java
// ZonePreferenceServiceInstanceListSupplier.java (Spring Cloud LoadBalancer)
// 같은 Zone의 인스턴스를 우선적으로 반환

public class ZonePreferenceServiceInstanceListSupplier
        extends DelegatingServiceInstanceListSupplier {

    // 현재 서비스의 zone (메타데이터에서 읽거나 환경변수에서 주입)
    private final String zone;

    @Override
    public Flux<List<ServiceInstance>> get(Request request) {
        return getDelegate().get(request).map(instances -> {
            // ① 같은 Zone 인스턴스 필터링
            List<ServiceInstance> sameZoneInstances = instances.stream()
                .filter(instance -> zone.equals(instance.getMetadata().get("zone")))
                .collect(Collectors.toList());

            // ② 같은 Zone 인스턴스 있으면 우선 사용
            //    없으면 전체 인스턴스 사용 (fallback)
            return sameZoneInstances.isEmpty() ? instances : sameZoneInstances;
        });
    }
}
```

### 5. Actuator serviceregistry — 런타임 상태 변경

```java
// ServiceRegistryEndpoint.java (Spring Cloud)
// /actuator/serviceregistry 엔드포인트

@Endpoint(id = "serviceregistry")
public class ServiceRegistryEndpoint {

    private final ServiceRegistry serviceRegistry;
    private Registration registration;

    // GET /actuator/serviceregistry
    @ReadOperation
    public Map<String, Object> getStatus() {
        return Collections.singletonMap("status",
            this.serviceRegistry.getStatus(this.registration));
    }

    // POST /actuator/serviceregistry
    // Body: {"status": "OUT_OF_SERVICE"}
    @WriteOperation
    public void setStatus(@Selector String status) {
        // Eureka에서: InstanceStatus.OUT_OF_SERVICE
        // → 이 인스턴스는 레지스트리에 있지만 트래픽 받지 않음
        this.serviceRegistry.setStatus(this.registration,
            InstanceStatus.valueOf(status));
    }
}
```

---

## 💻 실전 구성

### 카나리 배포 실전 구성

```yaml
# 카나리 인스턴스 (신버전 10%만 배포)
# application-canary.yml
eureka:
  instance:
    metadata-map:
      version: "v2.0.0"
      canary: "true"
      weight: "10"     # 상대적 가중치 (일반 인스턴스: 100)

# docker-compose.yml
services:
  service-a-stable:
    environment:
      - SPRING_PROFILES_ACTIVE=default
      # canary 메타데이터 없음 → 일반 인스턴스
    deploy:
      replicas: 9    # 90% 트래픽

  service-a-canary:
    environment:
      - SPRING_PROFILES_ACTIVE=canary
    deploy:
      replicas: 1    # 10% 트래픽 (CanaryLoadBalancer가 메타데이터로 분리)
```

### 배포 중 Graceful Traffic Drain

```bash
# 1. 인스턴스를 OUT_OF_SERVICE로 전환 (트래픽 중단)
curl -X POST http://service-a-1:8080/actuator/serviceregistry \
  -H "Content-Type: application/json" \
  -d '{"status": "OUT_OF_SERVICE"}'

# 2. Eureka 캐시 갱신 대기 (최대 30초)
sleep 30

# 3. 진행 중인 요청 완료 대기
sleep 10

# 4. 서비스 배포 실행
docker-compose up --no-deps service-a-1

# 5. UP으로 복구 (자동으로 됨 — 재시작 후 STARTING → UP)
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 버전 메타데이터로 하위 호환성 보장

```
상황: service-a v1 → v2 마이그레이션
     v2의 API 스펙이 일부 변경됨

Eureka 메타데이터 활용:
  service-a v1 인스턴스: metadata.version = "v1"
  service-a v2 인스턴스: metadata.version = "v2"

service-b (호출자):
  "아직 v2 API 지원 안 됨" → version: v1 인스턴스로만 라우팅
  "v2 API 사용 가능" → version: v2 인스턴스로 라우팅

VersionAwareLoadBalancer:
  RequestContext에서 "required-version" 헤더 읽기
  → 메타데이터의 version 값과 비교
  → 매칭되는 인스턴스로 라우팅

점진적 마이그레이션:
  1단계: v1 9개 + v2 1개 (10% 검증)
  2단계: v1 5개 + v2 5개 (50/50)
  3단계: v2만 유지
  각 단계에서 Eureka 메타데이터로 라우팅 제어
```

---

## ⚖️ 트레이드오프

| 접근법 | 유연성 | 복잡도 | 적합한 케이스 |
|--------|--------|--------|--------------|
| **메타데이터 기반 필터** | 높음 | 중간 | 카나리, 버전, Zone-Aware |
| **별도 서비스 이름** | 낮음 (인스턴스 분리) | 낮음 | 완전 독립 배포 |
| **가중치 Round Robin** | 중간 | 낮음 | 단순 트래픽 비율 제어 |

```
메타데이터 주의사항:
  metadataMap의 크기 제한:
    Eureka에는 공식 크기 제한 없지만
    매 Heartbeat마다 전송 → 너무 많으면 네트워크 부하
    권장: 5~10개 이내, 값은 짧은 문자열

  값 변경 불가:
    metadataMap은 인스턴스 등록 시 설정
    런타임에 변경하려면 인스턴스 재등록 필요
    → 자주 바뀌는 값은 메타데이터보다 설정 서버로 관리

  타입 없음:
    모든 값은 String
    숫자 비교 시 파싱 필요 (weight: "10" → Integer.parseInt())
```

---

## 📌 핵심 정리

```
Service Instance Metadata 핵심:

컨테이너 환경 필수 설정:
  prefer-ip-address: true  → hostname 대신 IP 등록
  instance-id: ${ip}:${appName}:${port}  → 고유 ID 보장

metadataMap 활용:
  Eureka InstanceInfo에 자유 형식 key-value 저장
  클라이언트가 레지스트리 조회 시 메타데이터도 함께 수신
  LoadBalancer Filter에서 메타데이터 읽어 라우팅 결정

주요 활용 패턴:
  카나리 배포: canary=true → 10% 트래픽 분배
  버전 라우팅: version=v2 → 특정 버전 인스턴스 선택
  Zone-Aware: zone=ap-northeast-2a → 같은 AZ 우선
  가중치: weight=10 → 상대적 트래픽 비율

런타임 상태 제어:
  POST /actuator/serviceregistry → OUT_OF_SERVICE
  → 재배포 없이 트래픽 즉시 차단
  → Graceful Traffic Drain에 활용
```

---

## 🤔 생각해볼 문제

**Q1.** `prefer-ip-address: true`로 설정했는데 같은 호스트에서 여러 포트로 인스턴스를 실행하면 `instance-id`가 중복되는 문제가 발생한다. 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

`instance-id`를 포트를 포함한 고유한 값으로 설정해야 합니다.

```yaml
eureka:
  instance:
    instance-id: ${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}
```

또는 랜덤 값을 포함시켜 완전히 고유하게 만들 수도 있습니다.

```yaml
eureka:
  instance:
    instance-id: ${spring.cloud.client.ip-address}:${spring.application.name}:${random.value}
```

K8s 환경에서는 Pod IP가 고유하므로 `${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}` 조합이 일반적으로 충분합니다.

</details>

---

**Q2.** Zone-Aware 라우팅에서 현재 서비스가 속한 Zone을 어떻게 자동으로 알아내는가? 코드에 하드코딩하지 않는 방법은?

<details>
<summary>해설 보기</summary>

AWS에서는 EC2 인스턴스 메타데이터 서비스(IMDS)를 통해 AZ 정보를 가져올 수 있습니다.

```yaml
# EC2 환경
eureka:
  instance:
    metadata-map:
      zone: ${EC2_AVAILABILITY_ZONE:unknown}  # EC2 인스턴스 메타데이터에서 주입
```

K8s 환경에서는 Pod spec에서 노드 레이블을 주입할 수 있습니다.

```yaml
# k8s Pod spec
env:
- name: AVAILABILITY_ZONE
  valueFrom:
    fieldRef:
      fieldPath: metadata.labels['topology.kubernetes.io/zone']
```

```yaml
# application.yml
eureka:
  instance:
    metadata-map:
      zone: ${AVAILABILITY_ZONE:default}
```

Spring Cloud의 `ZonePreferenceServiceInstanceListSupplier`는 환경 변수 `EUREKA_INSTANCE_METADATA_MAP_ZONE` 또는 메타데이터의 `zone` 키를 자동으로 읽어 Zone-Aware 라우팅에 활용합니다.

</details>

---

**Q3.** `OUT_OF_SERVICE`로 전환한 인스턴스는 Eureka 레지스트리에는 남아있지만 트래픽을 받지 않는다. 그런데 LoadBalancer가 이 인스턴스를 제외하는 원리는 무엇인가?

<details>
<summary>해설 보기</summary>

Spring Cloud LoadBalancer의 `HealthCheckServiceInstanceListSupplier`는 `ServiceInstance.getMetadata()`와 `ServiceInstance`가 구현하는 상태 확인 로직에서 `status == UP`인 인스턴스만 반환합니다.

Netflix Ribbon 기반의 `ServerList`는 `EurekaNotificationServerListUpdater`를 통해 Eureka 이벤트를 구독하고, `InstanceStatus.UP`인 인스턴스만 `ServerList`에 포함시킵니다.

Spring Cloud LoadBalancer에서는 `EurekaServiceInstance.getInstanceInfo().getStatus() == InstanceStatus.UP` 조건으로 필터링합니다. `OUT_OF_SERVICE` 상태로 전환하면 다음 레지스트리 fetch(최대 30초) 이후부터 LoadBalancer 목록에서 제외됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: Eureka Client 등록 과정 (Heartbeat)](./03-eureka-client-heartbeat.md)** | **[홈으로 🏠](../README.md)** | **[다음: Client-Side Load Balancing ➡️](./05-client-side-load-balancing.md)**

</div>
