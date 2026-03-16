# Externalized Configuration 필요성 — 설정을 코드에서 분리해야 하는 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 12-Factor App의 3번 원칙(Config)이 말하는 "환경(environment)에 의존하는 모든 것"이란 무엇인가?
- 설정값을 코드(`application.yml`)에 하드코딩했을 때 MSA에서 발생하는 실제 문제는 무엇인가?
- Spring의 `PropertySource` 추상화는 어떤 계층 구조로 프로퍼티를 탐색하는가?
- `Environment`가 프로퍼티 키를 조회할 때 우선순위가 높은 소스부터 탐색하는 과정은?
- Config Server 도입 이전에 팀이 보통 쓰는 임시방편들의 공통 단점은 무엇인가?
- `spring.config.import`와 `bootstrap.yml` 방식의 차이는 무엇인가?

---

## 🔍 왜 MSA에서 필요한가

### 문제 1: 환경마다 달라지는 설정값이 코드에 박혀 있다

```
단일 서비스일 때:
  application.yml 하나로 dev / staging / prod 프로파일 분기
  → 그나마 관리 가능

MSA에서:
  Service A, B, C, D, E, F ... (수십 개 서비스)
  각 서비스마다 application.yml 존재
  각 서비스마다 dev / staging / prod 프로파일 존재

  변경이 필요한 설정 하나 (예: DB 커넥션 풀 크기):
    → Service A application.yml 수정
    → Service B application.yml 수정
    → ...
    → 배포 파이프라인 N번 실행
    → 변경 이력 추적 불가

  결과:
    어떤 서비스가 어떤 설정값을 쓰는지 아무도 모름
    "이 서비스 prod에서 DB URL이 뭐였더라?"
    → 배포된 JAR 파일 압축 해제 후 확인
```

### 문제 2: 민감한 정보(비밀번호, API 키)가 코드 저장소에 노출된다

```yaml
# ❌ 실제로 많이 보이는 패턴
spring:
  datasource:
    url: jdbc:mysql://prod-db.internal:3306/myapp
    username: app_user
    password: SuperSecret123!   # Git에 올라가 있음

aws:
  access-key: AKIAIOSFODNN7EXAMPLE   # AWS 키도 Git에
  secret-key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

```
결과:
  Git 히스토리에 영구 기록
  퇴사한 직원도 조회 가능
  실수로 public 저장소에 push → 보안 사고
  GitHub secret scanning에 걸려도 이미 늦음
```

### 문제 3: 설정 변경에 재배포가 필요하다

```
현재 운영 중인 서비스:
  "Circuit Breaker threshold를 50%에서 70%로 올려야 합니다"

application.yml 수정 → commit → push → CI/CD 실행 → 배포
  소요 시간: 최소 10분 ~ 수십 분
  중단 없는 배포(rolling update)도 완료까지 대기 필요
  급한 장애 상황에서 설정 하나 바꾸려고 30분을 기다림

이상적인 흐름:
  설정값 변경 → 운영 중인 서비스에 즉시 반영
  재배포 없음
  변경 이력 Git에 자동 기록
```

---

## 😱 잘못된 구성

### Before: 환경별 프로파일 분리 — 임시방편의 한계

```yaml
# ❌ application-prod.yml이 코드 저장소에 있는 구조
src/main/resources/
├── application.yml
├── application-dev.yml
├── application-staging.yml
└── application-prod.yml    # prod DB 비밀번호가 여기에
```

```java
// ❌ 코드에서 직접 프로파일 분기
@Value("${spring.profiles.active}")
private String activeProfile;

public String getDbUrl() {
    if ("prod".equals(activeProfile)) {
        return "jdbc:mysql://prod-db:3306/app";
    }
    return "jdbc:mysql://localhost:3306/app";
}
// 코드가 환경을 알고 있음 → 12-Factor 위반
```

```
임시방편들의 공통 단점:
  1. 환경 변수 직접 관리: 수십 개 서비스 × 수십 개 변수 = 관리 지옥
  2. Kubernetes ConfigMap: 암호화 없음, 변경 이력 추적 어려움
  3. Vault만 사용: Spring과의 통합 설정이 복잡, 동적 갱신 추가 구현 필요
  4. 각 팀이 제각각: 서비스마다 설정 관리 방식 다름
```

---

## ✨ 올바른 패턴

### After: Externalized Configuration 원칙

```
12-Factor App Rule #3 — Config:
  "코드베이스와 설정의 완전한 분리"

  환경에 의존하는 모든 것 = Config:
    ✅ 데이터베이스, 메시지 큐, 외부 서비스 URL/인증정보
    ✅ 서드파티 API 키
    ✅ 배포 환경별로 달라지는 값 (로그 레벨, 스레드 풀 크기 등)
    
    ❌ Config가 아닌 것:
    내부 라우팅 설정 (예: Spring MVC URL 패턴)
    앱 모듈 간 의존성 설정 (Spring Bean 정의)

판단 기준:
  "이 값을 오픈소스로 공개해도 문제없는가?"
  → Yes: 코드에 있어도 됨
  → No: Config로 분리해야 함
```

```
Spring Cloud Config Server 도입 후:

Git Repository (설정 전용)
  ├── application.yml          ← 모든 서비스 공통 설정
  ├── service-a.yml            ← Service A 전용
  ├── service-b.yml            ← Service B 전용
  ├── application-prod.yml     ← prod 공통 오버라이드
  └── service-a-prod.yml       ← Service A prod 전용

Config Server (단일 진입점)
  ↑ Git에서 설정 읽기
  ↓ 각 서비스에 제공

Service A, B, C ...
  → 시작 시 Config Server에서 설정 로드
  → /actuator/refresh 호출 시 재배포 없이 갱신
```

---

## 🔬 내부 동작 원리

### 1. Spring의 PropertySource 추상화

```java
// PropertySource.java
// 모든 설정 소스의 추상 기반 클래스
public abstract class PropertySource<T> {
    protected final String name;    // 소스 이름 (예: "applicationConfig: [classpath:/application.yml]")
    protected final T source;       // 실제 소스 객체

    public abstract Object getProperty(String name);

    public boolean containsProperty(String name) {
        return getProperty(name) != null;
    }
}

// 구현체 예시
public class MapPropertySource extends EnumerablePropertySource<Map<String, Object>> {
    @Override
    public Object getProperty(String name) {
        return this.source.get(name);
    }
}

// Config Server에서 받아온 설정을 담는 구현체
public class CompositePropertySource extends EnumerablePropertySource<Object> {
    private final Collection<PropertySource<?>> propertySources = new LinkedHashSet<>();
    // Config Server가 여러 파일에서 읽은 설정을 하나로 병합
}
```

### 2. Environment의 PropertySource 우선순위 체계

```java
// StandardEnvironment에서 초기화되는 기본 PropertySource 스택
// 위에 있을수록 높은 우선순위 (아래를 오버라이드)

MutablePropertySources propertySources:
  [0] systemProperties          // JVM 시스템 프로퍼티 (-D 옵션)
  [1] systemEnvironment         // OS 환경 변수
  [2] random                    // random.* 값
  [3] Config Server 설정        // Spring Cloud Config가 여기에 삽입
      └── service-a-prod.yml    // 가장 구체적인 것이 먼저
      └── service-a.yml
      └── application-prod.yml
      └── application.yml       // 가장 일반적인 것이 나중
  [4] application-prod.yml      // 로컬 프로파일별 설정
  [5] application.yml           // 로컬 기본 설정
```

```java
// Environment.getProperty() 동작
public String getProperty(String key) {
    // PropertySource 스택을 순서대로 탐색
    for (PropertySource<?> propertySource : this.propertySources) {
        Object value = propertySource.getProperty(key);
        if (value != null) {
            return convertValueIfNecessary(value, String.class);
            // 첫 번째로 찾은 값 반환 → 높은 우선순위 소스가 이긴다
        }
    }
    return null;
}
```

### 3. Config Server가 PropertySource 스택에 삽입되는 시점

```
Spring Boot 애플리케이션 시작 순서:

1. JVM 시작
2. SpringApplication.run() 호출
3. Bootstrap Context 생성 (Spring Cloud 전용 초기 컨텍스트)
   └── ConfigServicePropertySourceLocator.locate() 실행
       → Config Server HTTP 요청
       → 응답을 PropertySource로 변환
       → Bootstrap PropertySources에 등록
4. Application Context 생성
   └── Bootstrap Context의 PropertySources를 부모로 병합
5. @Value, @ConfigurationProperties 바인딩

핵심:
  Config Server 설정이 application.yml보다 먼저 로드됨
  → Config Server 설정이 로컬 설정보다 높은 우선순위를 가짐
  → 로컬 application.yml은 개발용 기본값 역할
```

### 4. PropertySource 우선순위 직접 확인

```java
// Actuator /actuator/env 엔드포인트로 확인 가능
// GET http://localhost:8080/actuator/env/spring.datasource.url

{
  "property": {
    "source": "configService:service-a-prod.yml",  // 어떤 소스에서 왔는지
    "value": "jdbc:mysql://prod-db:3306/app"
  },
  "activeProfiles": ["prod"],
  "propertySources": [
    {
      "name": "configService:service-a-prod.yml",
      "property": { "value": "jdbc:mysql://prod-db:3306/app" }
    },
    {
      "name": "configService:application-prod.yml",
      "property": { "value": "jdbc:mysql://default-prod-db:3306/app" }
      // 위에서 이미 찾았으므로 이 값은 사용 안 됨
    }
  ]
}
```

---

## 💻 실전 구성

### Docker Compose: Config Server 기본 환경

```yaml
# docker-compose.yml
version: '3.9'

services:
  config-server:
    image: openjdk:17-jdk-slim
    container_name: config-server
    volumes:
      - ./config-server/target/config-server.jar:/app.jar
    environment:
      - SPRING_PROFILES_ACTIVE=native  # 로컬 파일 사용 (Git 없이 테스트)
      - SPRING_CLOUD_CONFIG_SERVER_NATIVE_SEARCH_LOCATIONS=classpath:/config-repo
    ports:
      - "8888:8888"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8888/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5

  service-a:
    image: openjdk:17-jdk-slim
    container_name: service-a
    volumes:
      - ./service-a/target/service-a.jar:/app.jar
    environment:
      - SPRING_CONFIG_IMPORT=configserver:http://config-server:8888
      - SPRING_APPLICATION_NAME=service-a
      - SPRING_PROFILES_ACTIVE=dev
    ports:
      - "8080:8080"
    depends_on:
      config-server:
        condition: service_healthy
```

```yaml
# Config Server: application.yml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          default-label: main
          search-paths: '{application}'  # service-a/ 디렉토리에서 탐색
          clone-on-start: true           # 시작 시 즉시 clone
```

```yaml
# Config Client (service-a): application.yml
spring:
  application:
    name: service-a          # Config Server에서 service-a.yml을 찾는 키
  config:
    import: "configserver:"  # Spring Boot 2.4+ 방식
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true        # Config Server 연결 실패 시 앱 시작 중단
      retry:
        max-attempts: 6      # 재시도 횟수
        initial-interval: 1000
        multiplier: 1.1
```

### Config Server API 직접 호출로 동작 확인

```bash
# 설정 조회: /{application}/{profile}[/{label}]
curl http://localhost:8888/service-a/dev
curl http://localhost:8888/service-a/prod
curl http://localhost:8888/service-a/dev/main  # label(브랜치) 지정

# 응답 구조
{
  "name": "service-a",
  "profiles": ["dev"],
  "label": "main",
  "version": "abc1234",    // Git commit hash
  "state": null,
  "propertySources": [
    {
      "name": "https://github.com/org/config-repo/service-a-dev.yml",
      "source": {
        "spring.datasource.url": "jdbc:mysql://dev-db:3306/app",
        "app.feature.enabled": "true"
      }
    },
    {
      "name": "https://github.com/org/config-repo/application.yml",
      "source": {
        "server.port": "8080",
        "management.endpoints.web.exposure.include": "health,info,refresh"
      }
    }
  ]
}
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 운영 중 설정값 긴급 변경

```
장애 상황:
  오전 2시, Circuit Breaker가 너무 민감하게 OPEN 상태로 전환
  resilience4j.circuitbreaker.instances.order-service.failureRateThreshold=50
  → 50%가 너무 낮음, 70%로 올려야 함

Config Server 없을 때:
  1. 코드 수정 (2분)
  2. commit/push (1분)
  3. CI/CD 파이프라인 대기 (5~15분)
  4. 배포 완료 대기 (5~10분)
  총 소요: 13~28분 / 이 시간 동안 장애 지속

Config Server 있을 때:
  1. Git 저장소에서 설정 파일 수정 (30초)
  2. commit/push (30초)
  3. /actuator/refresh 호출 (1초)  ← 재배포 없음!
  총 소요: 약 1분
```

```bash
# 긴급 변경 절차
# 1. Git에서 설정 수정 후 push

# 2. Config Server 캐시 무효화 (선택사항)
curl -X POST http://config-server:8888/actuator/refresh

# 3. 각 서비스 RefreshScope Bean 갱신
curl -X POST http://service-a:8080/actuator/refresh

# 결과: 재배포 없이 설정 반영 완료
```

### 시나리오: 환경별 설정 분리 효과

```
Git 저장소 구조:
  config-repo/
    application.yml           ← 모든 서비스, 모든 환경 공통
    application-prod.yml      ← 모든 서비스, prod 오버라이드
    service-a.yml             ← service-a 전용 공통
    service-a-dev.yml         ← service-a dev 전용
    service-a-prod.yml        ← service-a prod 전용

적용 우선순위 (service-a, prod 프로파일):
  service-a-prod.yml  > service-a.yml  > application-prod.yml  > application.yml

실제 값 결정:
  spring.datasource.url:
    service-a-prod.yml: jdbc:mysql://prod-db.internal:3306/app  ← 이 값 사용
    application-prod.yml: (없음)
    application.yml: (없음)
  
  logging.level.root:
    service-a-prod.yml: (없음)
    application-prod.yml: WARN  ← 이 값 사용 (prod 전체 공통)
    application.yml: DEBUG
```

---

## ⚖️ 트레이드오프

| 관점 | 장점 | 단점 / 주의사항 |
|------|------|----------------|
| **운영 편의성** | 재배포 없이 설정 변경 가능 | Config Server 자체가 SPOF(단일 장애점)가 될 수 있음 |
| **보안** | 암호화 지원, 코드와 설정 분리 | Git 저장소 접근 권한 관리 필요 |
| **이력 관리** | Git 히스토리로 변경 이력 자동 기록 | 설정과 코드 변경의 동기화 타이밍 주의 |
| **일관성** | 모든 서비스가 동일한 설정 소스 사용 | 네트워크 이슈로 Config Server 응답 지연 시 서비스 시작 지연 |
| **복잡도** | 중앙화된 설정 관리 | 로컬 개발 시 Config Server 실행 필요 (또는 `fail-fast=false`) |

```
언제 Config Server를 도입하지 않아도 되는가:
  - 서비스가 2~3개 이하인 초기 단계
  - 설정 변경 빈도가 매우 낮고 재배포 비용이 크지 않을 때
  - Kubernetes Secret + ConfigMap으로 이미 충분한 경우

언제 반드시 도입해야 하는가:
  - 마이크로서비스 5개 이상
  - 설정 변경에 즉각적인 반영이 필요할 때
  - 여러 환경(dev/staging/prod)의 설정을 체계적으로 관리해야 할 때
  - 감사(Audit) 요건 — 누가 언제 어떤 설정을 바꿨는지 추적 필요
```

---

## 📌 핵심 정리

```
Externalized Configuration의 핵심:

1. 12-Factor App Rule #3
   코드와 설정의 완전한 분리
   "오픈소스로 공개해도 문제없는가?" → No이면 Config로

2. Spring PropertySource 우선순위
   시스템 환경 변수 > Config Server > 로컬 profile yml > application.yml
   높은 우선순위 소스가 낮은 소스를 오버라이드

3. Config Server가 해결하는 문제
   ✅ 설정 중앙화 (하나의 Git 저장소)
   ✅ 재배포 없는 동적 갱신 (@RefreshScope)
   ✅ 암호화/복호화 (민감 정보 보호)
   ✅ 환경별 설정 분리 (profile 기반)
   ✅ 변경 이력 추적 (Git 히스토리)

4. Config 파일 탐색 순서
   {application}-{profile}.yml
   {application}.yml
   application-{profile}.yml
   application.yml
   → 위에서부터 적용, 아래로 갈수록 더 일반적
```

---

## 🤔 생각해볼 문제

**Q1.** `service-a-prod.yml`에 `spring.datasource.password`가 있고, `application-prod.yml`에도 같은 키가 있다면, 어느 값이 사용되는가? 이 우선순위는 어떤 규칙에서 비롯되는가?

<details>
<summary>해설 보기</summary>

`service-a-prod.yml`의 값이 사용됩니다. Config Server는 요청된 `{application}`과 `{profile}`에 맞게 여러 파일을 로드한 뒤, 더 구체적인 파일(application 이름 포함)의 PropertySource를 더 앞쪽에 배치합니다. `Environment.getProperty()`는 PropertySource 목록을 순서대로 탐색하다가 첫 번째로 찾은 값을 반환하므로, 더 구체적인 `service-a-prod.yml`이 이깁니다.

</details>

---

**Q2.** Config Server가 갑자기 다운됐을 때, 이미 시작된 서비스는 어떻게 동작하는가? 반면 이 상태에서 새 서비스 인스턴스를 시작하려 하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

이미 시작된 서비스는 정상 동작합니다. Config Server에서 가져온 설정값은 이미 Spring `Environment`의 PropertySource로 메모리에 로드된 상태이기 때문입니다. Config Server 연결이 끊겨도 기존 값은 유지됩니다.

반면 새 인스턴스를 시작할 때는 Bootstrap Context가 Config Server에 연결을 시도합니다. `fail-fast=true`로 설정된 경우 Config Server에 연결할 수 없으면 애플리케이션 시작이 실패합니다. `fail-fast=false`(기본값)라면 Config Server 없이 로컬 `application.yml`의 기본값만으로 시작합니다. 이것이 Config Server의 고가용성(HA)이 중요한 이유입니다.

</details>

---

**Q3.** 환경 변수 `SPRING_DATASOURCE_URL`과 Config Server에 있는 `spring.datasource.url` 중 어느 것이 우선순위가 높은가? 이를 이용해 어떤 전략을 취할 수 있는가?

<details>
<summary>해설 보기</summary>

OS 환경 변수(`systemEnvironment`)가 Config Server 설정보다 높은 우선순위를 가집니다. 이를 이용해 Kubernetes Secret을 환경 변수로 주입하면 Config Server에는 민감하지 않은 설정만 관리하고, DB 비밀번호처럼 민감한 값은 K8s Secret → 환경 변수로 오버라이드하는 전략을 쓸 수 있습니다. 또는 Config Server의 암호화 기능을 활용해 Config Server가 복호화하여 제공하는 방법도 있습니다(Ch1-05에서 다룹니다).

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Config Server 동작 원리 (Git Backend) ➡️](./02-config-server-git-backend.md)**

</div>
