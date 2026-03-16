# Config Server 동작 원리 (Git Backend) — EnvironmentController부터 PropertySource 반환까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Config Server의 `EnvironmentController`가 `/{application}/{profile}` 요청을 받아 어떤 순서로 처리하는가?
- `JGitEnvironmentRepository`는 Git 저장소에서 설정 파일을 어떻게 가져오는가? clone과 fetch의 차이는?
- `NativeEnvironmentRepository`가 yml 파일을 파싱해 `Environment` 객체로 만드는 과정은?
- `{application}-{profile}.yml`과 `application.yml`이 병합될 때 PropertySource 순서는 어떻게 결정되는가?
- `search-paths`와 `default-label` 설정이 파일 탐색에 어떤 영향을 주는가?
- Config Server가 Git 저장소를 로컬에 clone해 캐시하는 전략과 그 한계는?

---

## 🔍 왜 MSA에서 필요한가

### 단일 진입점으로서의 Config Server

```
MSA 환경에서 설정 요청 흐름:

Service A 시작
  → "내 이름은 service-a, 프로파일은 prod야"
  → Config Server에 GET /service-a/prod 요청
  → Config Server:
      1. Git에서 설정 파일 가져오기
      2. service-a-prod.yml, service-a.yml, application-prod.yml, application.yml 파싱
      3. PropertySource 목록으로 합쳐서 반환
  → Service A: 받은 설정을 Environment에 로드

핵심 설계 원칙:
  Config Server는 "설정을 읽어주는 HTTP 서버"
  자체 로직 없음 — Git 저장소의 파일을 HTTP로 노출할 뿐
  → 덕분에 Spring 없이 curl만으로도 설정 조회 가능
  → 설정 저장소(Git)와 설정 배포(Config Server) 역할 분리
```

---

## 😱 잘못된 구성

### Before: Git clone 없이 파일 직접 마운트

```yaml
# ❌ Docker Volume으로 설정 파일을 직접 마운트하는 구성
services:
  config-server:
    volumes:
      - ./config-files:/config    # 로컬 파일 직접 마운트
    environment:
      - SPRING_CLOUD_CONFIG_SERVER_NATIVE_SEARCH_LOCATIONS=file:/config

# 문제:
# 1. 변경 이력 없음 — 누가 언제 무엇을 바꿨는지 모름
# 2. CI/CD와 분리 불가 — 파일 변경 후 Config Server 재시작 필요
# 3. 다중 Config Server 시 파일 동기화 어려움
# 4. Rollback 불가 — 이전 설정값으로 돌아가려면 수동으로 파일 복구
```

```java
// ❌ Config Server 없이 각 서비스가 직접 Git에서 읽는 방식
// (구현 상상 버전 — 실제로는 이렇게 하면 안 됨)
@Service
public class ConfigLoader {
    public Properties loadFromGit(String repoUrl, String fileName) {
        Git.cloneRepository().setURI(repoUrl).call();
        // 모든 서비스가 각자 Git clone → 저장소 부하
        // Git 인증 정보가 각 서비스에 분산
    }
}
```

---

## ✨ 올바른 패턴

### After: Git Backend Config Server 전체 흐름

```
HTTP 요청 → EnvironmentController
              ↓
          EnvironmentEncryptorDecorator (암호화된 값 복호화)
              ↓
          JGitEnvironmentRepository.findOne()
              ↓
          Git 작업: clone / fetch / checkout
              ↓
          NativeEnvironmentRepository.findOne()
              ↓
          yml/properties 파일 파싱
              ↓
          Environment 객체 조립 (PropertySource 목록)
              ↓
          JSON 응답 반환

클라이언트가 받는 응답:
  {
    "name": "service-a",
    "profiles": ["prod"],
    "label": "main",
    "version": "a1b2c3d",   ← Git commit hash (불변성 보장)
    "propertySources": [...]
  }
```

---

## 🔬 내부 동작 원리

### 1. EnvironmentController — HTTP 요청 진입점

```java
// EnvironmentController.java
// spring-cloud-config-server 모듈

@RestController
@RequestMapping(method = RequestMethod.GET,
    path = "${spring.cloud.config.server.prefix:}")
public class EnvironmentController {

    private EnvironmentRepository repository;
    private EnvironmentEncryptor encryptor;

    // GET /{application}/{profile}
    @RequestMapping("/{name}/{profiles:.*[^-].*}")
    public Environment defaultLabel(
            @PathVariable String name,
            @PathVariable String profiles) {
        return labelled(name, profiles, null);
    }

    // GET /{application}/{profile}/{label}
    @RequestMapping("/{name}/{profiles}/{label:.*}")
    public Environment labelled(
            @PathVariable String name,
            @PathVariable String profiles,
            @PathVariable String label) {

        // ① 쉼표로 구분된 다중 프로파일 지원 ("prod,cloud")
        if (name != null && name.contains("(_)")) {
            name = name.replace("(_)", "/");
        }

        // ② Repository에 위임
        Environment environment = this.repository.findOne(name, profiles, label);

        // ③ 암호화된 값 복호화 ({cipher}xxx → 실제 값)
        if (!this.encryptor.equals(NOOP)) {
            environment = this.encryptor.decrypt(environment);
        }

        return environment;
    }
}
```

### 2. JGitEnvironmentRepository — Git 연동의 핵심

```java
// JGitEnvironmentRepository.java
public class JGitEnvironmentRepository extends AbstractScmEnvironmentRepository {

    private String uri;           // Git 저장소 URI
    private String defaultLabel;  // 기본 브랜치 (main)
    private boolean cloneOnStart; // 시작 시 즉시 clone

    @Override
    public synchronized Locations getLocations(String application, String profile, String label) {
        // label = null이면 defaultLabel 사용
        if (label == null) {
            label = this.defaultLabel;
        }

        // ① 로컬 Git 저장소 경로 준비
        File basedir = getWorkingDirectory();

        // ② Git 저장소 상태 확인
        if (shouldPull(basedir)) {
            // 이미 clone된 경우: fetch + checkout
            fetch(basedir, label);
            checkout(basedir, label);
        } else {
            // 최초 또는 저장소 없음: clone
            clone(basedir, label);
        }

        // ③ 체크아웃된 실제 파일 경로 반환
        return new Locations(application, profile, label,
            this.getVersion(basedir, label),     // Git commit hash
            new String[]{ basedir.getAbsolutePath() }
        );
    }

    private void fetch(File basedir, String label) {
        try (Git git = openGitRepository(basedir)) {
            FetchCommand fetch = git.fetch();
            fetch.setRemote("origin");
            // 인증 설정 (SSH key 또는 username/password)
            configureAuthentication(fetch);
            fetch.call();

            // 원격 브랜치로 reset (로컬 변경사항 덮어쓰기)
            git.reset()
               .setMode(ResetCommand.ResetType.HARD)
               .setRef("origin/" + label)
               .call();
        }
    }

    private void clone(File basedir, String label) {
        CloneCommand clone = Git.cloneRepository()
            .setURI(this.uri)
            .setDirectory(basedir)
            .setBranchesToClone(Collections.singleton("refs/heads/" + label))
            .setBranch("refs/heads/" + label);
        configureAuthentication(clone);
        clone.call();
    }
}
```

### 3. NativeEnvironmentRepository — 파일 파싱 엔진

```java
// NativeEnvironmentRepository.java
// JGitEnvironmentRepository가 checkout한 로컬 파일을 실제로 파싱

public class NativeEnvironmentRepository implements EnvironmentRepository {

    @Override
    public Environment findOne(String config, String profile, String label) {

        // ① Spring Application을 "설정 로더"로 사용하는 독특한 패턴
        SpringApplicationBuilder builder = new SpringApplicationBuilder(
            PropertyPlaceholderAutoConfiguration.class
        );

        // ② 로컬 파일 경로를 PropertySource로 설정
        ConfigurableEnvironment environment = getEnvironment(profile);
        builder.environment(environment);

        // ③ 검색 경로 설정
        //    searchPaths: {application} 치환 → service-a 디렉토리 탐색
        String[] searchLocations = getSearchLocations(config, label, profile);

        // ④ SpringApplication 실행 (실제 HTTP 서버는 시작 안 됨)
        //    목적: application.yml 로더 메커니즘 재사용
        try (ConfigurableApplicationContext context =
                builder.web(WebApplicationType.NONE)
                       .run("--spring.config.location=" + String.join(",", searchLocations))) {

            // ⑤ 로드된 PropertySource 추출
            environment = context.getEnvironment();
            return clean(new PassthruEnvironmentRepository(environment)
                .findOne(config, profile, label));
        }
    }

    private String[] getSearchLocations(String application, String label, String profile) {
        // search-paths: classpath:/config-repo, /data/config
        // {application} 플레이스홀더를 실제 서비스 이름으로 치환
        return Arrays.stream(this.searchPaths)
            .map(path -> path
                .replace("{application}", application)
                .replace("{profile}", profile)
                .replace("{label}", label))
            .toArray(String[]::new);
    }
}
```

### 4. Environment 조립 — PropertySource 병합 순서

```
findOne("service-a", "prod", "main") 처리 결과:

NativeEnvironmentRepository가 로드하는 파일 순서:
  ① service-a-prod.yml   (application + profile 조합, 가장 구체적)
  ② service-a.yml        (application 전용)
  ③ application-prod.yml (profile 전용)
  ④ application.yml      (전체 공통, 가장 일반적)

Environment.propertySources 구성:
  [0] "configserver:service-a-prod.yml"   ← 최고 우선순위
  [1] "configserver:service-a.yml"
  [2] "configserver:application-prod.yml"
  [3] "configserver:application.yml"      ← 최저 우선순위

동일 키가 여러 파일에 있을 때:
  spring.datasource.url:
    service-a-prod.yml: jdbc:mysql://prod.db:3306/app  ← 이 값 사용
    application-prod.yml: jdbc:mysql://default.db:3306/app  ← 무시
```

### 5. 실제 Git 캐시 동작

```java
// shouldPull() — 매 요청마다 Git fetch 여부 결정
private boolean shouldPull(File basedir) {
    // refreshRate 설정에 따라 캐시 사용
    // 기본값: 0 (매 요청마다 fetch)
    // spring.cloud.config.server.git.refresh-rate=30  ← 30초 캐시

    if (this.refreshRate > 0) {
        // 마지막 fetch로부터 refreshRate초 이내면 캐시 사용
        return System.currentTimeMillis() - this.lastRefresh > this.refreshRate * 1000L;
    }
    return true; // 항상 fetch
}

// 캐시 디렉토리 위치
// basedir: /tmp/config-repo-{hash}/
//   매 Config Server 인스턴스마다 별도 로컬 clone
//   Config Server 재시작 시 재clone
```

---

## 💻 실전 구성

### Git Backend 상세 설정

```yaml
# Config Server application.yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          
          # 기본 브랜치 (Git flow에서는 develop/main 분리 가능)
          default-label: main
          
          # 서비스별 서브디렉토리 탐색
          # service-a 요청 → config-repo/service-a/ 디렉토리 탐색
          search-paths: '{application}'
          
          # 시작 시 즉시 clone (실패 시 서버 시작 실패 → HA 구성 필수)
          clone-on-start: true
          
          # Git 인증 (HTTPS)
          username: ${GIT_USERNAME}
          password: ${GIT_TOKEN}     # GitHub Personal Access Token
          
          # Git 인증 (SSH)
          # private-key: ${GIT_PRIVATE_KEY}
          # ignore-local-ssh-settings: true
          
          # 캐시: 30초간 재fetch 안 함 (성능 최적화)
          refresh-rate: 30
          
          # 강제 pull (로컬 변경사항 무시)
          force-pull: true
          
          # 타임아웃
          timeout: 10
          
          # 다중 저장소 패턴 (서비스별 다른 Git 저장소)
          repos:
            payment:
              pattern: payment-*           # payment-service, payment-worker 등
              uri: https://github.com/org/payment-config
              search-paths: '{application}'
```

```bash
# 설정 저장소(config-repo) 구조 예시
config-repo/
├── application.yml              # 전체 공통
├── application-dev.yml          # dev 공통
├── application-prod.yml         # prod 공통
├── service-a/
│   ├── service-a.yml            # search-paths: '{application}' 사용 시
│   ├── service-a-dev.yml
│   └── service-a-prod.yml
└── service-b/
    ├── service-b.yml
    └── service-b-prod.yml
```

### 동작 확인

```bash
# Config Server 로그에서 Git 작업 확인
# [INFO] Fetching config from server at: https://github.com/org/config-repo
# [INFO] Located environment: name=service-a, profiles=[prod], label=main, version=a1b2c3d

# 특정 commit(label)의 설정 조회 → 롤백 시나리오에 유용
curl http://localhost:8888/service-a/prod/a1b2c3d   # commit hash로 조회
curl http://localhost:8888/service-a/prod/v1.2.0    # Git tag로 조회
curl http://localhost:8888/service-a/prod/feature/new-config  # 브랜치로 조회

# 파일 형식으로도 조회 가능
curl http://localhost:8888/service-a-prod.yml        # YAML 형식 직접 반환
curl http://localhost:8888/service-a-prod.properties # Properties 형식
```

---

## 🌐 분산 시스템 시나리오

### 시나리오 1: Config Server 시작 시 Git clone 실패

```
상황: 회사 Git 서버 점검 중 Config Server 재시작 필요

clone-on-start: true 일 때:
  Config Server 시작 → Git clone 시도 → 실패
  → Config Server 자체가 시작 안 됨
  → 모든 서비스 시작 불가 (SPOF!)

대응 전략:
  1. clone-on-start: false (요청 시점에 clone) — 단, 첫 요청 지연
  2. basedir을 영구 볼륨에 마운트 → 재시작 시 기존 clone 재사용
  3. Config Server 다중화 (Ch1-06에서 상세 다룸)

docker-compose에서 basedir 영구화:
  volumes:
    - config-repo-cache:/tmp/config-repo   # 재시작해도 유지
  
  → 이전에 clone된 파일이 남아있어 Git 서버 없이도 동작 가능
  → 단, 설정이 최신이 아닐 수 있음 (스테일 캐시)
```

### 시나리오 2: 설정 변경 후 즉시 반영 확인

```bash
# 1. Git에서 설정 파일 수정
# service-a-prod.yml:
# app.circuit-breaker.threshold: 70  (50에서 변경)

# 2. commit & push
git add service-a-prod.yml
git commit -m "fix: circuit breaker threshold 50 → 70"
git push origin main

# 3. Config Server 캐시 즉시 무효화 (refresh-rate 기다리지 않으려면)
curl -X POST http://config-server:8888/actuator/refresh

# 4. 변경 확인
curl http://config-server:8888/service-a/prod | jq '.propertySources[0].source'
# → "app.circuit-breaker.threshold": "70"  확인

# 5. 서비스 RefreshScope 갱신 (Ch1-04에서 상세)
curl -X POST http://service-a:8080/actuator/refresh
```

---

## ⚖️ 트레이드오프

| 관점 | Git Backend | Native (파일 시스템) |
|------|-------------|---------------------|
| **변경 이력** | Git 히스토리 자동 관리 | 별도 이력 관리 필요 |
| **롤백** | commit hash / tag로 즉시 롤백 | 파일 수동 복구 |
| **가용성** | Git 서버 다운 → 캐시로 동작 | 파일 시스템 항상 가용 |
| **보안** | Git 인증 (SSH/HTTPS) | 파일 권한으로 관리 |
| **운영 복잡도** | Git 저장소 관리 필요 | 단순, 로컬 개발에 적합 |
| **적합한 용도** | 프로덕션 환경 | 로컬 개발, CI 테스트 |

```
refresh-rate 설정 트레이드오프:
  0 (매 요청마다 fetch):
    장점: 항상 최신 설정
    단점: Git 서버에 매 요청마다 부하 발생, 응답 지연

  30~60초:
    장점: Git 서버 부하 감소, 빠른 응답
    단점: 설정 변경 후 최대 refresh-rate초 지연

  권장: 30초 (운영 환경)
  refresh 즉시 필요: POST /actuator/refresh 수동 호출
```

---

## 📌 핵심 정리

```
Config Server Git Backend 동작 요약:

HTTP 요청 /{application}/{profile}/{label}
  ↓
EnvironmentController
  ↓
JGitEnvironmentRepository
  ├── shouldPull()? → fetch + checkout
  └── 최초? → clone
  ↓
NativeEnvironmentRepository
  ├── SpringApplication으로 yml 파싱
  └── {application}-{profile}.yml 우선 탐색
  ↓
Environment 객체
  ├── name, profiles, label
  ├── version (Git commit hash)
  └── propertySources (우선순위 순서대로)
      [0] {application}-{profile}.yml  ← 최고 우선순위
      [1] {application}.yml
      [2] application-{profile}.yml
      [3] application.yml              ← 최저 우선순위

성능 최적화 키:
  refresh-rate: Git fetch 주기 (기본 0 = 매번)
  clone-on-start: 시작 시 즉시 clone 여부
  basedir: 캐시 저장 위치 (영구 볼륨 권장)
```

---

## 🤔 생각해볼 문제

**Q1.** `search-paths: '{application}'`으로 설정되어 있고, 서비스 이름이 `payment-service`라면 Config Server는 어떤 경로에서 파일을 탐색하는가? `payment-service-prod.yml`과 `application-prod.yml` 중 무엇이 먼저 로드되는가?

<details>
<summary>해설 보기</summary>

`{application}`이 `payment-service`로 치환되어 `config-repo/payment-service/` 디렉토리를 탐색합니다. 로드 순서는 다음과 같습니다.
1. `payment-service/payment-service-prod.yml`
2. `payment-service/payment-service.yml`
3. `application-prod.yml` (search-paths 루트)
4. `application.yml`

`payment-service-prod.yml`이 더 구체적이므로 먼저 PropertySource에 배치되고, 동일 키가 있을 때 이 값이 최종적으로 사용됩니다.

</details>

---

**Q2.** Config Server에 `refresh-rate: 0` (기본값)이 설정된 상태에서 트래픽이 많은 MSA 환경에서 어떤 문제가 발생할 수 있는가? 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

`refresh-rate: 0`이면 서비스 시작 시와 `/actuator/refresh` 호출 시마다 Git 서버에 fetch 요청이 발생합니다. 서비스 인스턴스가 수십 개이고 동시에 refresh를 호출하면 Git 서버에 과부하가 걸릴 수 있습니다.

해결책:
1. `refresh-rate: 30` 이상으로 설정 → Git fetch 빈도 줄이기
2. Spring Cloud Bus 도입 → Config Server가 하나의 이벤트를 Kafka/RabbitMQ로 발행하면 모든 서비스가 동시에 refresh (Git 서버 요청 1번)
3. Config Server 앞에 HTTP 캐시 레이어(Nginx) 추가

</details>

---

**Q3.** `label`(Git 브랜치/태그)을 지정해서 설정을 조회하는 기능을 어떻게 활용하면 Blue-Green 배포나 카나리 배포에서 설정 관리를 개선할 수 있는가?

<details>
<summary>해설 보기</summary>

`label`을 브랜치로 활용하면 다음과 같은 전략이 가능합니다.

**Blue-Green 배포:**
- Blue 환경: `spring.cloud.config.label=main` (현재 안정 설정)
- Green 환경: `spring.cloud.config.label=release/v2` (새 설정 브랜치)
- 검증 완료 후 `release/v2`를 `main`에 merge → Blue도 동일 설정 적용

**카나리 배포:**
- 전체 트래픽(95%): label=main
- 카나리 인스턴스(5%): label=canary
- 카나리 설정에서 feature flag 활성화

Git commit hash를 label로 사용하면 정확히 어떤 시점의 설정을 사용하는지 보장할 수 있어 재현 가능한 배포가 됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: Externalized Configuration 필요성](./01-externalized-configuration.md)** | **[홈으로 🏠](../README.md)** | **[다음: Config Client 자동 설정 ➡️](./03-config-client-auto-configuration.md)**

</div>
