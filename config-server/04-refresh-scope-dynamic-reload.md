# @RefreshScope와 동적 갱신 — 재배포 없이 설정값을 바꾸는 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@RefreshScope`가 붙은 Bean이 실제로 어떻게 "Refresh"되는가? Proxy 패턴이 어떻게 사용되는가?
- `ContextRefresher.refresh()`의 실행 순서는 무엇인가?
- `/actuator/refresh` 호출 후 `@Value`가 바뀌지 않는 이유는 무엇인가? `@RefreshScope`와의 차이는?
- Spring Cloud Bus를 쓰면 왜 Config Server에 변경이 있을 때 모든 서비스가 자동으로 갱신되는가?
- `@RefreshScope` Bean의 생명주기와 일반 Singleton Bean의 생명주기는 어떻게 다른가?
- `RefreshScope.refreshAll()`이 호출될 때 Bean 캐시는 어떻게 무효화되는가?

---

## 🔍 왜 MSA에서 필요한가

### 재배포 없는 설정 갱신의 필요성

```
운영 중 설정 변경 시나리오:

1. 기능 플래그 (Feature Toggle)
   "신규 결제 시스템 점진적 롤아웃"
   app.feature.new-payment: false → true
   → 재배포하면 전체 롤아웃 — 점진적 전환 불가
   → 즉시 반영으로 1% 사용자에게만 활성화, 모니터링 후 확대

2. 임계값 튜닝
   "Circuit Breaker가 너무 민감함"
   resilience4j.circuitbreaker.failureRateThreshold: 50 → 70
   → 재배포 대기 15분 동안 장애 지속

3. 외부 서비스 URL 변경
   "3rd party API가 URL 변경 공지"
   payment.gateway.url: https://old.api.com → https://new.api.com
   → 재배포 없이 즉시 전환

4. 로그 레벨 임시 조정
   "장애 분석을 위해 DEBUG 레벨 임시 활성화"
   logging.level.com.myapp: INFO → DEBUG
   → 문제 해결 후 다시 INFO로

@RefreshScope가 없다면:
  모든 케이스에서 코드 수정 → 배포 필요
  서비스 수 × 배포 시간 = 운영 지옥
```

---

## 😱 잘못된 구성

### Before: @RefreshScope 없이 @Value 사용

```java
// ❌ Singleton Bean에서 @Value
@Service
public class PaymentService {

    @Value("${payment.gateway.url}")
    private String gatewayUrl;  // 최초 주입 후 변경 불가

    public void processPayment() {
        // Config Server에서 payment.gateway.url을 바꿔도
        // 이 Bean은 Spring Context 시작 시 한 번만 생성됨
        // gatewayUrl 필드는 영원히 처음 주입된 값
        callGateway(gatewayUrl);
    }
}
```

```
/actuator/refresh를 호출해도:
  Environment의 PropertySource는 갱신됨
  그러나 이미 생성된 PaymentService Bean은 재생성되지 않음
  → gatewayUrl 필드는 그대로
  
  @Value는 Bean 생성 시 한 번 주입
  → Singleton Bean은 Context 재시작 없이 필드 변경 불가
```

### Before: @RefreshScope를 남발

```java
// ❌ 모든 Bean에 @RefreshScope 적용
@RefreshScope
@Service
public class OrderService {  // 설정값을 전혀 쓰지 않는 서비스

    private final OrderRepository repository;
    // 생성자 주입

    public Order createOrder(OrderDto dto) {
        return repository.save(dto.toEntity());
    }
}
```

```
문제:
  @RefreshScope Bean은 Lazy 초기화 Proxy
  → 매 호출 시 실제 Bean 존재 여부 확인 오버헤드
  → refresh 발생 시 재생성 → 상태 초기화 위험
  → 불필요한 Bean 재생성

  @RefreshScope는 설정값을 직접 사용하는 Bean에만 적용해야 함
```

---

## ✨ 올바른 패턴

### After: @RefreshScope의 올바른 적용 범위

```java
// ✅ @RefreshScope는 설정값을 직접 읽는 Bean에만
@RefreshScope
@Service
public class CircuitBreakerConfigService {

    @Value("${resilience4j.circuitbreaker.failure-rate-threshold:50}")
    private int failureRateThreshold;

    // refresh 발생 시 이 Bean이 재생성되며 새 값이 주입됨
    public int getThreshold() {
        return failureRateThreshold;
    }
}

// ✅ @ConfigurationProperties + @RefreshScope 조합
@RefreshScope
@ConfigurationProperties(prefix = "app.feature")
@Component
public class FeatureProperties {
    private boolean newPaymentEnabled;
    private int maxRetryCount;
    // getter/setter
}
// → refresh 시 FeatureProperties Bean 재생성 → 새 설정 바인딩
```

---

## 🔬 내부 동작 원리

### 1. @RefreshScope의 핵심 — CGLIB Proxy

```java
// RefreshScope.java (GenericScope 상속)
// Scope 인터페이스 구현 → Spring IoC 컨테이너에 "refresh" 스코프 등록

@Component
@ManagedResource
public class RefreshScope extends GenericScope implements ApplicationContextAware,
        ApplicationListener<ContextRefreshedEvent>, Ordered {

    // refresh 스코프의 Bean들을 캐시하는 구조
    // Map<beanName, Bean 인스턴스>
    // GenericScope가 이 캐시를 관리

    @Override
    public Object get(String name, ObjectFactory<?> objectFactory) {
        // ① 캐시에 Bean이 있으면 반환 (일반 요청 경로)
        BeanLifecycleWrapper value = this.cache.get(name);
        if (value == null) {
            // ② 없으면 objectFactory로 생성 후 캐시에 저장
            value = new BeanLifecycleWrapper(name, objectFactory);
            this.cache.put(name, value);
        }
        return value.getBean();
    }

    // refresh 발생 시 호출
    public void refreshAll() {
        // ③ 캐시를 전부 비움
        super.destroy();
        // ④ RefreshScopeRefreshedEvent 발행
        this.context.publishEvent(new RefreshScopeRefreshedEvent());
        // 다음 요청 시 ②번 경로로 Bean이 재생성됨
    }
}
```

```java
// @RefreshScope Bean의 실제 동작 방식
// Spring이 @RefreshScope Bean을 주입할 때 실제로 하는 일:

// PaymentService에 CircuitBreakerConfigService를 주입할 때:
// 실제 CircuitBreakerConfigService 인스턴스 X
// → CGLIB Proxy를 주입

// Proxy 내부:
public class CircuitBreakerConfigService$$EnhancerBySpringCGLIB extends CircuitBreakerConfigService {

    private final RefreshScope scope;
    private final String beanName;

    @Override
    public int getThreshold() {
        // ① scope.get(beanName)으로 실제 Bean 조회
        CircuitBreakerConfigService realBean =
            (CircuitBreakerConfigService) scope.get(beanName, ...);
        // ② 실제 Bean에 위임
        return realBean.getThreshold();
    }
}

// refresh 발생 → scope.refreshAll() → 캐시 비움
// 다음 getThreshold() 호출 시:
//   scope.get()이 캐시에서 못 찾음
//   → 새 CircuitBreakerConfigService 인스턴스 생성
//   → 새 @Value 주입
//   → 새 설정값 반환
```

### 2. ContextRefresher.refresh() 전체 실행 순서

```java
// ContextRefresher.java
// /actuator/refresh 엔드포인트가 호출하는 핵심 클래스

public class ContextRefresher {

    private final ConfigurableApplicationContext context;
    private final RefreshScope scope;

    public synchronized Set<String> refresh() {
        // ① 현재 Environment의 모든 PropertySource 스냅샷 저장
        Map<String, Object> before = extract(
            this.context.getEnvironment().getPropertySources());

        // ② Config Server에서 최신 설정 다시 로드
        //    ConfigServicePropertySourceLocator.locate() 재실행
        addConfigFilesToEnvironment();

        // ③ 이전과 달라진 PropertySource 키 계산
        Set<String> keys = changes(before,
            extract(this.context.getEnvironment().getPropertySources()));

        // ④ EnvironmentChangeEvent 발행
        //    → @ConfigurationProperties 재바인딩 트리거
        this.context.publishEvent(new EnvironmentChangeEvent(this.context, keys));

        // ⑤ RefreshScope.refreshAll() 호출
        //    → @RefreshScope Bean 캐시 전체 무효화
        this.scope.refreshAll();

        // ⑥ 변경된 키 목록 반환 (actuator/refresh 응답 body)
        return keys;
    }

    private void addConfigFilesToEnvironment() {
        // 임시 ApplicationContext를 생성해 Config Server 재호출
        // 새 PropertySources를 메인 Context의 Environment에 적용
        ConfigurableApplicationContext capture = null;
        try {
            StandardEnvironment environment = copyEnvironment(this.context.getEnvironment());
            SpringApplicationBuilder builder = new SpringApplicationBuilder(Empty.class)
                .bannerMode(Mode.OFF)
                .web(WebApplicationType.NONE)
                .environment(environment);
            capture = builder.run();
            // 새로 로드된 PropertySources로 메인 Environment 교체
            MutablePropertySources target = this.context.getEnvironment().getPropertySources();
            // ... PropertySource 교체 로직
        } finally {
            if (capture != null) { capture.close(); }
        }
    }
}
```

### 3. @ConfigurationProperties 재바인딩

```java
// ConfigurationPropertiesRebinder.java
// EnvironmentChangeEvent를 받아 @ConfigurationProperties Bean들을 재바인딩

@Component
public class ConfigurationPropertiesRebinder
        implements ApplicationContextAware, ApplicationListener<EnvironmentChangeEvent> {

    private Map<String, Object> beans = new HashMap<>();  // @ConfigurationProperties Bean 목록

    @Override
    public void onApplicationEvent(EnvironmentChangeEvent event) {
        if (this.applicationContext.equals(event.getSource())) {
            // ① 등록된 @ConfigurationProperties Bean 전체 재바인딩
            rebind();
        }
    }

    public void rebind() {
        for (Map.Entry<String, Object> entry : this.beans.entrySet()) {
            String name = entry.getKey();
            Object bean = entry.getValue();

            // ② BeanDefinition 가져오기
            BeanDefinition bd = this.applicationContext.getBeanFactory()
                .getBeanDefinition(name);

            // ③ PropertyValues 초기화 후 재적용
            // → @ConfigurationProperties의 필드값이 새 Environment 값으로 업데이트
            this.binder.bind(bean);
        }
    }
}

// 중요:
// @ConfigurationProperties는 @RefreshScope 없어도 EnvironmentChangeEvent로 재바인딩됨
// @Value는 둘 다 필요 없음 → @RefreshScope Bean이 재생성될 때만 새 값 주입
```

### 4. Spring Cloud Bus — 다중 인스턴스 동시 Refresh

```
단일 인스턴스 refresh:
  POST /actuator/refresh → 해당 인스턴스만 갱신
  
  MSA에서 서비스 인스턴스가 N개라면:
  → N번의 curl 호출 필요
  → 일부는 갱신, 일부는 미갱신 → 불일치 상태 발생

Spring Cloud Bus 도입:
  
  Git 변경 → GitHub Webhook → Config Server
    → POST /actuator/busrefresh
    → MessageBroker (Kafka / RabbitMQ)에 RefreshRemoteApplicationEvent 발행
    → 모든 서비스 인스턴스가 이벤트 수신
    → 각자 ContextRefresher.refresh() 실행
    → 전체 인스턴스 동시 갱신
  
  의존성:
    spring-cloud-starter-bus-kafka
    또는
    spring-cloud-starter-bus-amqp (RabbitMQ)
```

```java
// Spring Cloud Bus 설정
// application.yml
spring:
  cloud:
    bus:
      enabled: true
  kafka:
    bootstrap-servers: kafka:9092

// GitHub Webhook 설정:
// Payload URL: http://config-server:8888/actuator/busrefresh
// Content type: application/json
// Events: Push events

// 또는 특정 서비스만 갱신 (destination 파라미터)
// POST /actuator/busrefresh?destination=service-a:**
// → service-a 서비스의 모든 인스턴스만 갱신
```

---

## 💻 실전 구성

### Docker Compose: Refresh 실험 환경

```yaml
# docker-compose.yml
services:
  config-server:
    environment:
      - MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE=refresh,busrefresh,health
  
  service-a:
    environment:
      - MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE=refresh,env,health

  # Spring Cloud Bus 사용 시
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
```

### Refresh 실험 순서

```bash
# 1. 현재 설정값 확인
curl http://localhost:8080/actuator/env/app.feature.enabled
# {"property":{"source":"configService:service-a-dev.yml","value":"false"}}

# 2. Git 저장소에서 설정 변경 (편집기로 수정 후 commit/push)
# app.feature.enabled: true

# 3. Config Server 캐시 무효화 (refresh-rate가 있는 경우)
curl -X POST http://localhost:8888/actuator/refresh

# 4. Config Server에서 변경 확인
curl http://localhost:8888/service-a/dev | jq '.propertySources[0].source'
# → "app.feature.enabled": "true" 확인

# 5. 서비스 refresh
curl -X POST http://localhost:8080/actuator/refresh
# 응답: ["app.feature.enabled"]  ← 변경된 키 목록

# 6. 새 값 적용 확인
curl http://localhost:8080/actuator/env/app.feature.enabled
# {"property":{"source":"configService:service-a-dev.yml","value":"true"}}

# 7. @RefreshScope Bean 재생성 확인
# 서비스 로그: "Refreshed beans: [featureProperties, circuitBreakerConfigService]"
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: @RefreshScope Bean의 상태 손실

```java
// @RefreshScope Bean이 내부 상태를 가질 때 위험
@RefreshScope
@Service
public class RateLimiterService {

    @Value("${app.rate-limit.max-requests:100}")
    private int maxRequests;

    // ❌ 위험한 상태
    private final Map<String, Integer> requestCount = new HashMap<>();
    private final AtomicInteger counter = new AtomicInteger(0);

    public boolean isAllowed(String userId) {
        // refresh 발생 시 이 Bean이 재생성됨
        // → requestCount와 counter가 초기화됨
        // → 카운터 리셋 → 레이트 리미팅 우회 가능
        return requestCount.getOrDefault(userId, 0) < maxRequests;
    }
}
```

```java
// ✅ 올바른 패턴: 설정만 @RefreshScope, 상태는 분리
@RefreshScope
@ConfigurationProperties(prefix = "app.rate-limit")
@Component
public class RateLimitProperties {
    private int maxRequests = 100;
    // getter/setter
}

@Service  // @RefreshScope 없음
public class RateLimiterService {

    private final RateLimitProperties properties;  // @RefreshScope Bean 주입 (Proxy)
    private final Map<String, Integer> requestCount = new ConcurrentHashMap<>();

    public boolean isAllowed(String userId) {
        // properties.getMaxRequests()는 항상 최신 값 (Proxy를 통해 조회)
        // requestCount 상태는 refresh 영향 없음
        return requestCount.getOrDefault(userId, 0) < properties.getMaxRequests();
    }
}
```

---

## ⚖️ 트레이드오프

| 방식 | 장점 | 단점 |
|------|------|------|
| `@RefreshScope + @Value` | 필드 레벨 세밀한 제어 | Bean 재생성 → 상태 초기화 위험 |
| `@RefreshScope + @ConfigurationProperties` | 관련 설정 그룹화 | 클래스 전체 재생성 |
| `@ConfigurationProperties` (Bus 없이) | `EnvironmentChangeEvent`로 재바인딩 | 복잡한 설정 변경 시 재바인딩 타이밍 이슈 |
| Spring Cloud Bus | 다중 인스턴스 동시 갱신 | Kafka/RabbitMQ 인프라 필요 |

```
@RefreshScope 사용 시 주의사항:
  1. Bean이 내부 상태(캐시, 카운터)를 가지면 refresh 후 초기화됨
  2. @RefreshScope Bean을 주입받는 곳은 항상 Proxy를 통해 접근
     → Singleton Bean이 @RefreshScope Bean을 필드로 가지면 OK
     → 역방향(@RefreshScope Bean이 Singleton을 필드로)도 OK
  3. 무거운 Bean(DB 커넥션 풀 등)에 @RefreshScope 적용 주의
     → refresh마다 재생성 → 성능 영향
```

---

## 📌 핵심 정리

```
@RefreshScope 동작 메커니즘:

1. CGLIB Proxy로 감쌈
   실제 Bean 대신 Proxy가 주입됨
   모든 메서드 호출 → Proxy가 RefreshScope 캐시에서 실제 Bean 조회

2. /actuator/refresh 호출 시
   ContextRefresher.refresh():
   ① Config Server 재호출 → 새 PropertySources로 Environment 교체
   ② EnvironmentChangeEvent → @ConfigurationProperties 재바인딩
   ③ RefreshScope.refreshAll() → Bean 캐시 비움
   
   다음 메서드 호출 시:
   캐시 miss → 새 Bean 생성 → 새 @Value 주입 → 새 설정값 사용

3. Spring Cloud Bus
   단일 /actuator/busrefresh 호출
   → 메시지 브로커 경유
   → 전체 인스턴스 동시 ContextRefresher.refresh() 실행

핵심 규칙:
  설정값을 직접 사용하는 Bean에만 @RefreshScope
  내부 상태가 있는 Bean에는 @RefreshScope 주의
  @ConfigurationProperties는 @RefreshScope 없이도 재바인딩됨
```

---

## 🤔 생각해볼 문제

**Q1.** `@RefreshScope`를 붙인 `DataSource` Bean이 있다. refresh가 발생했을 때 어떤 문제가 생길 수 있는가?

<details>
<summary>해설 보기</summary>

DataSource Bean이 재생성되면 기존 커넥션 풀이 닫히고 새 풀이 생성됩니다. 이 시점에 DB 커넥션을 사용 중이던 스레드들은 연결이 갑자기 끊길 수 있습니다. 또한 새 DataSource 생성 시 DB 연결 확립에 시간이 걸려 일시적인 응답 지연이 발생합니다. 일반적으로 DataSource는 `@RefreshScope`를 적용하지 않고, DB URL 같은 설정은 환경 변수로 주입하거나 설정 변경 시 재배포하는 방식을 선택합니다. 실시간 변경이 꼭 필요하다면 DataSource Proxy 패턴(LazyDataSource)을 활용해 재생성 타이밍을 제어해야 합니다.

</details>

---

**Q2.** `/actuator/refresh`를 호출했는데 응답 body가 `[]`(빈 배열)로 반환됐다. 이것이 의미하는 바는 무엇인가? 설정이 실제로 바뀌었는지 확인하는 방법은?

<details>
<summary>해설 보기</summary>

응답 `[]`는 ContextRefresher가 변경된 PropertySource 키를 하나도 감지하지 못했다는 의미입니다. 원인은 다음 중 하나입니다.
1. Git push가 완료되지 않았거나 Config Server가 아직 캐시를 가지고 있음 (`refresh-rate` 대기 중)
2. Config Server에는 변경됐지만 서비스가 연결하는 Config Server URI가 다름
3. 파일 수정은 했지만 실제로 해당 서비스/프로파일에 영향 없는 파일을 수정함

확인 방법:
```bash
curl http://config-server:8888/service-a/dev | jq '.propertySources[0].source'
```
Config Server API를 직접 조회해서 예상한 값이 반환되는지 먼저 확인한 뒤, 서비스의 `/actuator/refresh`를 호출합니다.

</details>

---

**Q3.** Spring Cloud Bus에서 `POST /actuator/busrefresh?destination=service-a:**` 처럼 destination을 지정하는 패턴에서 `**`의 의미는 무엇인가?

<details>
<summary>해설 보기</summary>

`destination` 파라미터는 `{serviceId}:{instanceId}` 패턴을 사용합니다. `service-a:**`는 서비스 이름이 `service-a`인 모든 인스턴스(`**`)를 대상으로 합니다. Ant 패턴이 적용되어 `**`는 모든 문자열을 의미합니다. 특정 인스턴스만 갱신하려면 `service-a:8080:service-a:abc123`처럼 인스턴스 ID까지 지정할 수 있습니다. destination 없이 호출하면 Bus에 연결된 모든 서비스의 모든 인스턴스가 갱신됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: Config Client 자동 설정](./03-config-client-auto-configuration.md)** | **[홈으로 🏠](../README.md)** | **[다음: Encryption / Decryption ➡️](./05-encryption-decryption.md)**

</div>
