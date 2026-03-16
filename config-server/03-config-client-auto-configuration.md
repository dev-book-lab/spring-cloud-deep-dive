# Config Client 자동 설정 — 서비스가 시작할 때 설정을 가져오는 과정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ConfigClientAutoConfiguration`이 등록하는 Bean들은 무엇이며, 언제 활성화되는가?
- Bootstrap Context란 무엇이고, Application Context와 어떤 부모-자식 관계를 갖는가?
- `ConfigServicePropertySourceLocator.locate()`가 Config Server에 HTTP 요청을 보내는 정확한 시점은?
- `bootstrap.yml` 방식과 `spring.config.import` 방식(Spring Boot 2.4+)의 내부 동작 차이는?
- `fail-fast: true` 설정이 없을 때 Config Server 연결 실패는 어떻게 처리되는가?
- Retry 설정이 동작하려면 어떤 의존성이 추가로 필요한가?

---

## 🔍 왜 MSA에서 필요한가

### Config Client가 없다면

```
Config Server가 아무리 잘 동작해도,
클라이언트 서비스가 "시작 시점에 설정을 가져오는" 메커니즘이 없다면:

선택지 1: @Value로 Config Server URL 하드코딩
  @Value("${config.server.url}")  // 이 값 자체를 어디서 읽어오나?
  → 닭이 먼저냐 달걀이 먼저냐 문제

선택지 2: 애플리케이션 코드가 직접 Config Server 호출
  @PostConstruct
  public void loadConfig() {
      RestTemplate rt = new RestTemplate();
      Map props = rt.getForObject("http://config-server:8888/...", Map.class);
      // Spring Environment에 어떻게 주입?
      // @Value 바인딩은 이미 완료된 상태 → 너무 늦음
  }
  → @Value 주입 전에 설정이 로드되어야 한다는 근본 문제

해결:
  Spring이 Bean을 만들기 전,
  Environment가 구성되는 시점에 Config Server 설정을 로드해야 함
  → Bootstrap Context 패턴
```

---

## 😱 잘못된 구성

### Before: bootstrap.yml에 일반 설정도 함께 넣는 실수

```yaml
# ❌ bootstrap.yml에 너무 많은 설정
spring:
  application:
    name: service-a        # ✅ 여기에 있어야 함
  cloud:
    config:
      uri: http://config-server:8888  # ✅ 여기에 있어야 함

# 아래는 ❌ bootstrap.yml에 있으면 안 됨
server:
  port: 8080               # application.yml 또는 Config Server에서 관리해야 함
logging:
  level:
    root: DEBUG
spring:
  datasource:
    url: jdbc:h2:mem:test  # 로컬 기본값도 application.yml에
```

```
문제:
  bootstrap.yml은 Bootstrap Context에서 로드됨
  Bootstrap Context는 부모 컨텍스트 → 자식(Application Context)보다 먼저 생성
  
  bootstrap.yml에 있는 설정은 Config Server 설정으로 오버라이드 안 됨
  → Config Server에서 server.port를 바꾸려 해도 반영 안 됨
  → 일반 설정은 application.yml에 두고 Config Server가 오버라이드하는 구조여야 함
```

### Before: spring.config.import 사용 시 순서 혼동

```yaml
# ❌ spring.config.import 위치 실수
# application.yml

server:
  port: 8080           # 이 설정이 Config Server보다 먼저 처리됨

spring:
  config:
    import: "configserver:http://config-server:8888"
  application:
    name: service-a
```

```
문제:
  spring.config.import는 파일 내 위치와 무관하게 동작하지만,
  application.yml 자체는 Config Server 설정보다 낮은 우선순위
  
  그러나 spring.application.name이 없으면 Config Server가
  어떤 서비스 설정을 반환해야 할지 모름 → 기본값 "application"으로 요청
```

---

## ✨ 올바른 패턴

### After: Spring Boot 2.4+ 권장 설정 방식

```yaml
# application.yml (spring.config.import 방식 - 권장)
spring:
  application:
    name: service-a            # Config Server에 이 이름으로 설정 요청
  profiles:
    active: dev                # {profile} 값
  config:
    import: "configserver:"    # Config Server URI는 별도 설정에서

  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true
      retry:
        max-attempts: 6
        initial-interval: 1000
        multiplier: 1.1
        max-interval: 2000

# 개발 환경 로컬 기본값 (Config Server가 없어도 동작하게)
server:
  port: 8080    # Config Server에서 오버라이드되면 그 값 사용
```

```yaml
# bootstrap.yml 방식 (Spring Boot 2.3 이하 또는 레거시)
spring:
  application:
    name: service-a
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true
```

---

## 🔬 내부 동작 원리

### 1. ConfigClientAutoConfiguration — 자동 설정 진입점

```java
// ConfigClientAutoConfiguration.java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(ConfigClientProperties.class)
public class ConfigClientAutoConfiguration {

    // Config Server 연결 정보를 담는 Properties Bean 등록
    // spring.cloud.config.* 프로퍼티 바인딩

    @Bean
    @ConditionalOnMissingBean(ConfigServicePropertySourceLocator.class)
    @ConditionalOnProperty(value = "spring.cloud.config.enabled", matchIfMissing = true)
    public ConfigServicePropertySourceLocator configServicePropertySource(
            ConfigClientProperties properties) {
        return new ConfigServicePropertySourceLocator(properties);
    }
}

// ConfigClientProperties.java
@ConfigurationProperties("spring.cloud.config")
public class ConfigClientProperties {
    private String uri = "http://localhost:8888";   // 기본값
    private String name;       // spring.application.name에서 가져옴
    private String profile = "default";
    private String label;
    private boolean failFast = false;
    private Retry retry = new Retry();
    // ...
}
```

### 2. Bootstrap Context 생성 과정 (Spring Boot 2.3 이하 / bootstrap.yml 방식)

```java
// BootstrapApplicationListener.java
// spring.factories에 ApplicationListener로 등록됨
// SpringApplication이 실행될 때 가장 먼저 호출되는 리스너 중 하나

public class BootstrapApplicationListener
        implements ApplicationListener<ApplicationEnvironmentPreparedEvent>, Ordered {

    @Override
    public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
        ConfigurableEnvironment environment = event.getEnvironment();

        // bootstrap.yml / bootstrap.properties 로드 여부 확인
        if (!environment.getProperty("spring.cloud.bootstrap.enabled", Boolean.class, true)) {
            return;
        }

        // 이미 Bootstrap 완료 여부 확인 (중복 실행 방지)
        if (environment.getPropertySources().contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
            return;
        }

        // ① Bootstrap 전용 SpringApplication 생성
        ConfigurableApplicationContext context = bootstrapServiceContext(environment, event.getSpringApplication());

        // ② Bootstrap Context의 PropertySources를 메인 Environment 앞에 삽입
        //    (높은 우선순위 = Config Server 설정이 로컬 설정보다 우선)
        mergePropertySources(environment, context.getEnvironment());
    }

    private ConfigurableApplicationContext bootstrapServiceContext(
            ConfigurableEnvironment environment, SpringApplication application) {

        // bootstrap.yml을 설정으로 하는 별도 SpringApplication 실행
        SpringApplicationBuilder builder = new SpringApplicationBuilder()
            .profiles(environment.getActiveProfiles())
            .bannerMode(Mode.OFF)
            .environment(bootstrapEnvironment)
            // Bootstrap Context는 웹 서버 없이 실행
            .web(WebApplicationType.NONE)
            .sources(BootstrapImportSelectorConfiguration.class);

        // PropertySourceLocator들이 여기서 호출됨
        // → ConfigServicePropertySourceLocator.locate() 실행
        return builder.run();
    }
}
```

### 3. ConfigServicePropertySourceLocator — Config Server HTTP 호출

```java
// ConfigServicePropertySourceLocator.java
// PropertySourceLocator 인터페이스 구현
// 역할: Config Server에서 설정을 가져와 PropertySource로 반환

@Order(0)
public class ConfigServicePropertySourceLocator implements PropertySourceLocator {

    @Override
    public PropertySource<?> locate(Environment environment) {
        // ① 현재 application name, profiles, label 결정
        ConfigClientProperties properties = this.defaultProperties.override(environment);

        // ② CompositePropertySource 생성 (여러 파일 병합 컨테이너)
        CompositePropertySource composite =
            new CompositePropertySource("configService");

        // ③ HTTP 요청으로 Config Server 호출
        RestTemplate restTemplate = this.restTemplate != null
            ? this.restTemplate
            : getSecureRestTemplate(properties);

        Exception error = null;
        String errorBody = null;

        try {
            // 프로파일이 여러 개일 수 있음 (쉼표 구분)
            String[] labels = new String[]{""};
            if (StringUtils.hasText(properties.getLabel())) {
                labels = StringUtils.commaDelimitedListToStringArray(properties.getLabel());
            }

            for (String label : labels) {
                // ④ Config Server 응답(Environment JSON) → PropertySource 변환
                Environment result = getRemoteEnvironment(restTemplate, properties,
                    label.trim(), null);

                if (result != null) {
                    // ⑤ 응답의 propertySources를 CompositePropertySource에 추가
                    for (PropertySource source : result.getPropertySources()) {
                        Map<String, Object> map = (Map<String, Object>) source.getSource();
                        composite.addPropertySource(
                            new MapPropertySource(source.getName(), map));
                    }
                    return composite;
                }
            }
        } catch (HttpServerErrorException e) {
            error = e;
            // ...
        }

        // ⑥ fail-fast 처리
        if (properties.isFailFast()) {
            throw new IllegalStateException(
                "Could not locate PropertySource and the fail fast property is set, failing",
                error);
        }
        // fail-fast=false: 경고 로그만 남기고 빈 PropertySource 반환
        return null;
    }

    private Environment getRemoteEnvironment(RestTemplate restTemplate,
            ConfigClientProperties properties, String label, String state) {
        // URL 패턴: {uri}/{name}/{profile}/{label}
        String path = "/{name}/{profile}";
        String name = properties.getName();
        String profile = properties.getProfile();
        String uri = properties.getRawUri();

        Object[] args = new String[]{name, profile};
        if (StringUtils.hasText(label)) {
            path = path + "/{label}";
            args = new String[]{name, profile, label};
        }

        ResponseEntity<Environment> response = restTemplate.exchange(
            uri + path, HttpMethod.GET, null, Environment.class, args);

        return response.getBody();
    }
}
```

### 4. Spring Boot 2.4+ `spring.config.import` 동작 방식

```java
// ConfigServerConfigDataLoader.java (Spring Boot 2.4+)
// spring.config.import: "configserver:" 처리

public class ConfigServerConfigDataLoader implements ConfigDataLoader<ConfigServerConfigDataResource> {

    @Override
    public ConfigData load(ConfigDataLoaderContext context,
            ConfigServerConfigDataResource resource) {

        // Spring Boot의 ConfigData 로딩 메커니즘 활용
        // → BootstrapContext 없이도 동작 (경량화)

        ConfigClientProperties properties = resource.getProperties();
        List<PropertySource<?>> propertySources = new ArrayList<>();

        // Config Server 호출 (내부 로직은 동일)
        Environment environment = getRemoteEnvironment(...);

        if (environment != null) {
            for (PropertySource source : environment.getPropertySources()) {
                propertySources.add(new MapPropertySource(...));
            }
        }

        // 오버라이드 우선순위 설정
        // OVERRIDE_AND_REPLACE: Config Server 설정이 로컬 application.yml보다 우선
        return new ConfigData(propertySources,
            propertySource -> ConfigData.Option.OVERRIDE_AND_REPLACE);
    }
}
```

### 5. Bootstrap Context vs Application Context 계층 구조

```
Bootstrap Context (부모)
  PropertySources:
    - configService (Config Server에서 로드)   ← 최고 우선순위
    - bootstrapProperties (bootstrap.yml)

  ↓ 자식 컨텍스트로 PropertySources 전달

Application Context (자식)
  PropertySources:
    - configService                             ← 부모에서 상속
    - systemProperties (JVM -D 옵션)
    - systemEnvironment (OS 환경 변수)
    - applicationConfig: application-{profile}.yml
    - applicationConfig: application.yml        ← 최저 우선순위

  @Value 바인딩, @ConfigurationProperties 바인딩은
  Application Context의 모든 PropertySource가 준비된 후 실행
```

---

## 💻 실전 구성

### Retry 설정 (Config Server 일시 불가 대응)

```xml
<!-- pom.xml: Retry 동작을 위한 필수 의존성 -->
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  cloud:
    config:
      fail-fast: true          # true여야 retry가 의미 있음
      retry:
        initial-interval: 1000  # 첫 재시도 간격 (ms)
        max-attempts: 6         # 최대 시도 횟수
        multiplier: 1.1         # 간격 증가 배율
        max-interval: 2000      # 최대 간격 (ms)

# 총 대기 시간 계산:
# 1000 + 1100 + 1210 + 1331 + 1464 = 약 7초 동안 재시도
```

```yaml
# 로컬 개발: Config Server 없이도 실행 가능하게
spring:
  cloud:
    config:
      fail-fast: false    # 연결 실패해도 로컬 설정으로 계속
      enabled: false      # 완전히 비활성화 옵션
      
# 또는 profile로 분리
---
spring:
  config:
    activate:
      on-profile: local
  cloud:
    config:
      enabled: false
```

### 설정 로드 확인

```bash
# 서비스 시작 로그에서 Config Server 연결 확인
# [INFO] Fetching config from server at : http://config-server:8888
# [INFO] Located environment: name=service-a, profiles=[dev], label=main, version=abc123

# Actuator로 로드된 PropertySource 확인
curl http://localhost:8080/actuator/env | jq '.propertySources[] | .name'
# "configService:service-a-dev.yml"    ← Config Server에서 로드됨
# "configService:application.yml"
# "systemEnvironment"
# "applicationConfig: [classpath:/application.yml]"
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: Config Server 다운 중 새 서비스 인스턴스 배포

```
운영 상황:
  Config Server가 일시적으로 응답 없음
  새 service-a 인스턴스를 rolling update로 배포 중

fail-fast: true + retry 설정 시:
  새 인스턴스 시작
  → Config Server 호출 시도 (1차)
  → 1000ms 대기
  → Config Server 호출 시도 (2차)
  → 1100ms 대기
  → ...
  → 6회 모두 실패
  → 서비스 시작 실패 (ContextRefreshFailedEvent 발생)
  → K8s가 시작 실패를 감지하고 이전 버전 유지
  
결과: 배포는 실패했지만 기존 인스턴스는 정상 운영 중 → 장애 전파 차단
```

```
fail-fast: false 시:
  새 인스턴스 시작
  → Config Server 호출 실패
  → 경고 로그만 남기고 진행
  → 로컬 application.yml의 기본값만으로 시작
  
  문제:
    prod DB URL이 Config Server에 있었다면 → dev DB URL로 연결 시도
    암호화된 비밀번호가 Config Server에 있었다면 → 복호화 없이 {cipher}xxx 그대로 사용
    
  결론: prod 환경에서는 반드시 fail-fast: true
```

---

## ⚖️ 트레이드오프

| 설정 | fail-fast: true | fail-fast: false |
|------|-----------------|------------------|
| **Config Server 다운 시** | 서비스 시작 실패 | 기본값으로 시작 |
| **안전성** | 잘못된 설정으로 시작 방지 | 서비스 가용성 우선 |
| **적합 환경** | prod, staging | local, dev |
| **추가 필요 사항** | Retry 설정 + Config Server HA | - |

```
bootstrap.yml vs spring.config.import 비교:

bootstrap.yml:
  장점: Spring Boot 2.3 이하와 호환
  단점: spring-cloud-starter-bootstrap 의존성 필요
       Bootstrap Context 생성 오버헤드

spring.config.import (Boot 2.4+):
  장점: 의존성 추가 없이 동작
       ConfigData API 활용 (더 유연한 우선순위 제어)
  단점: 구버전 Spring Cloud와 호환성 확인 필요
  
  권장: 신규 프로젝트는 spring.config.import 방식
```

---

## 📌 핵심 정리

```
Config Client 자동 설정 핵심:

설정 로드 타이밍이 핵심:
  ApplicationContext 생성 전에 Config Server 설정이 Environment에 있어야 함
  → @Value 바인딩은 Bean 생성 시점 → 그 전에 Environment 구성 완료 필요

두 가지 방식:
  1. bootstrap.yml + Bootstrap Context (레거시)
     BootstrapApplicationListener → Bootstrap SpringApplication 실행
     → ConfigServicePropertySourceLocator.locate() → Config Server 호출
     → Bootstrap Context PropertySources를 메인 Environment에 삽입
  
  2. spring.config.import (Spring Boot 2.4+, 권장)
     ConfigServerConfigDataLoader → Config Server 호출
     → ConfigData로 PropertySource 반환 → 높은 우선순위로 삽입

핵심 설정:
  spring.application.name = Config Server에 요청할 서비스 이름
  fail-fast: true = Config Server 없으면 시작 거부 (prod 필수)
  retry.max-attempts = 일시적 네트워크 오류 대응
```

---

## 🤔 생각해볼 문제

**Q1.** `spring.application.name`을 `bootstrap.yml`에 설정하지 않고 `application.yml`에만 설정했다면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`bootstrap.yml` 방식에서는 Bootstrap Context가 `application.yml`을 로드하기 전에 Config Server 요청을 시도합니다. 이 시점에는 `spring.application.name`이 아직 로드되지 않았으므로 이름을 알 수 없어 기본값인 `"application"`으로 요청합니다. 결과적으로 `application.yml`과 `application-{profile}.yml`만 로드되고, `service-a.yml` 같은 서비스 전용 설정은 로드되지 않습니다. `spring.config.import` 방식에서도 `spring.application.name`은 Config Server 호출보다 먼저 인식될 수 있도록 파일 상단에 위치시키는 것이 안전합니다.

</details>

---

**Q2.** Config Server가 `service-a-dev.yml`을 반환했고 그 안에 `spring.datasource.url`이 있다. 서비스의 로컬 `application-dev.yml`에도 같은 키가 있다. 어느 값이 최종적으로 사용되는가? 이 우선순위를 역전시킬 수 있는가?

<details>
<summary>해설 보기</summary>

Config Server에서 가져온 값이 사용됩니다. `ConfigServicePropertySourceLocator`가 반환하는 PropertySource가 `application-dev.yml`보다 높은 순서로 Environment PropertySources에 삽입되기 때문입니다.

역전 방법: `spring.cloud.config.override-system-properties=false`와 `spring.cloud.config.override-none=true`를 설정하면 Config Server 설정이 로컬 설정보다 낮은 우선순위를 가지게 됩니다. 단, 이는 Config Server의 중앙 관리 목적을 무력화하므로 일반적으로 권장하지 않습니다. 로컬 개발에서만 사용하는 패턴입니다.

</details>

---

**Q3.** Kubernetes 환경에서 Config Server 주소를 하드코딩하지 않고 동적으로 발견하게 하려면 어떻게 구성해야 하는가?

<details>
<summary>해설 보기</summary>

두 가지 방법이 있습니다.

1. **Kubernetes Service DNS 활용**: Kubernetes는 Service 이름을 DNS로 자동 등록합니다. `spring.cloud.config.uri=http://config-server:8888`처럼 서비스 이름을 그대로 사용하면 Kubernetes DNS가 해석해줍니다. 환경 변수 `SPRING_CLOUD_CONFIG_URI`로 외부 주입도 가능합니다.

2. **Eureka를 통한 Config Server 디스커버리**: `spring.cloud.config.discovery.enabled=true`로 설정하면 Eureka에서 `configserver` 이름으로 등록된 인스턴스를 찾아 URI를 자동으로 결정합니다. 단, Eureka 서버 주소는 여전히 bootstrap.yml에 필요합니다. Ch1-06에서 이 HA 구성을 상세히 다룹니다.

</details>

---

<div align="center">

**[⬅️ 이전: Config Server 동작 원리 (Git Backend)](./02-config-server-git-backend.md)** | **[홈으로 🏠](../README.md)** | **[다음: @RefreshScope와 동적 갱신 ➡️](./04-refresh-scope-dynamic-reload.md)**

</div>
