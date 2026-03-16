# Config Server High Availability — Config Server가 SPOF가 되지 않게 하는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Config Server가 단일 장애점(SPOF)이 되면 MSA 전체에 어떤 연쇄 장애가 발생하는가?
- Config Server를 다중화(Active-Active)할 때 상태 동기화 문제를 어떻게 해결하는가?
- Eureka에 Config Server 자체를 등록해서 클라이언트가 동적으로 Config Server를 찾게 하는 방법은?
- 로컬 Git clone 캐시를 활용한 Config Server 장애 복원력 전략은 무엇인가?
- 클라이언트 `fail-fast + retry` 설정만으로 충분한 HA를 달성할 수 있는가?
- Composite 저장소로 여러 Git 저장소를 하나의 Config Server로 서비스하는 방법은?

---

## 🔍 왜 MSA에서 필요한가

### Config Server SPOF 시나리오

```
Config Server가 다운됐을 때 발생하는 상황:

시나리오 A: 서비스 시작 시 Config Server 다운
  → fail-fast: true인 서비스들 전부 시작 실패
  → K8s Rolling Update 중이라면 모든 새 Pod 시작 불가
  → 구 버전 Pod들은 정상이지만 신 버전 배포 불가
  
시나리오 B: 서비스 운영 중 Config Server 다운
  → 이미 시작된 서비스들은 정상 운영 (메모리의 설정값 사용)
  → /actuator/refresh 불가 (Config Server 재호출 실패)
  → 새 인스턴스 추가(Scale-out) 불가 (시나리오 A)

시나리오 C: Config Server 재시작 시 Git 서버도 다운
  → Config Server 자체도 시작 불가
  → 모든 서비스 신규 배포 불가능

  MSA에서 Config Server는 모든 서비스의 공통 의존성
  → 단일 인스턴스 = 시스템 전체 단일 장애점
```

---

## 😱 잘못된 구성

### Before: Config Server 1개 + fail-fast + retry만 의존

```yaml
# ❌ 단일 Config Server에 의존하는 구성
services:
  config-server:
    replicas: 1    # 단일 인스턴스
  
  service-a:
    environment:
      - SPRING_CLOUD_CONFIG_URI=http://config-server:8888
      - SPRING_CLOUD_CONFIG_FAIL_FAST=true
      - SPRING_CLOUD_CONFIG_RETRY_MAX_ATTEMPTS=6

# retry가 있어도:
# Config Server가 장시간 다운 → retry 소진 → 서비스 시작 실패
# retry는 "일시적 장애"를 위한 것, "지속적 장애"는 다중화로 해결해야 함
```

### Before: 상태를 가진 Config Server 다중화 (잘못된 방식)

```yaml
# ❌ 각 Config Server 인스턴스가 독립적인 로컬 설정 파일 사용
services:
  config-server-1:
    volumes:
      - ./configs-node1:/config   # 노드1 전용 파일
  config-server-2:
    volumes:
      - ./configs-node2:/config   # 노드2 전용 파일 (다를 수 있음!)

# 문제:
# 두 인스턴스가 다른 설정 파일 → 어떤 서버에 붙느냐에 따라 다른 설정
# Git Backend를 쓰더라도 로컬 clone 시점이 다르면 일시적 불일치 발생
```

---

## ✨ 올바른 패턴

### After: Config Server HA 3계층 전략

```
Layer 1: Config Server 다중화
  ↑
  Config Server Instance 1 (replica)
  Config Server Instance 2 (replica)
  Config Server Instance 3 (replica)
  
  공통: 모두 같은 Git 저장소 참조 (상태 공유)

Layer 2: 클라이언트 Retry + Failover
  ↑
  클라이언트가 여러 Config Server URI 시도
  또는 Load Balancer 뒤에 Config Server 배치

Layer 3: 로컬 캐시 (마지막 방어선)
  ↑
  Config Server의 로컬 Git clone 영구 볼륨
  Git 서버 다운 시 캐시에서 응답

계층적 방어: 한 계층 실패해도 다음 계층이 동작
```

---

## 🔬 내부 동작 원리

### 1. Config Server 다중화 — Stateless 설계 활용

```yaml
# Config Server는 본질적으로 Stateless
# → "Git 저장소를 읽어서 HTTP로 응답하는 서버"
# → 각 인스턴스가 같은 Git 저장소를 바라보면 응답 동일

# docker-compose.yml
services:
  config-server:
    image: config-server:latest
    deploy:
      replicas: 2    # 2개 인스턴스
    environment:
      - SPRING_CLOUD_CONFIG_SERVER_GIT_URI=https://github.com/org/config-repo
      - ENCRYPT_KEY=${ENCRYPT_KEY}  # 모든 인스턴스 동일 암호화 키 필수!
    
  # Load Balancer (Nginx 또는 K8s Service)
  nginx:
    image: nginx:latest
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    ports:
      - "8888:8888"
```

```nginx
# nginx.conf: Config Server 로드 밸런싱
upstream config-servers {
    server config-server-1:8888;
    server config-server-2:8888;
    # health check: 응답 없는 서버는 자동 제외
}

server {
    listen 8888;
    location / {
        proxy_pass http://config-servers;
    }
}
```

```
다중화 시 필수 일치 조건:
  ① Git URI — 모든 인스턴스가 같은 저장소
  ② 암호화 키(ENCRYPT_KEY) — 어느 인스턴스가 복호화해도 동일한 결과
  ③ Spring 프로파일 — 같은 설정으로 동작

암호화 키 불일치 시:
  인스턴스 A가 암호화한 값을 인스턴스 B가 복호화 시도 → 실패
  → 클라이언트는 '{cipher}값' 대신 '<n/a>' 수신
```

### 2. Eureka에 Config Server 등록 — Service Discovery 연동

```yaml
# Config Server: Eureka에 자신을 등록
spring:
  application:
    name: config-server   # Eureka 등록 이름
eureka:
  client:
    service-url:
      defaultZone: http://eureka:8761/eureka/
  instance:
    prefer-ip-address: true
```

```yaml
# Config Client: Eureka에서 Config Server를 찾는 방식
# bootstrap.yml (또는 application.yml)
spring:
  cloud:
    config:
      discovery:
        enabled: true              # Eureka로 Config Server 위치 탐색
        service-id: config-server  # Eureka에서 찾을 서비스 이름
      fail-fast: true
      retry:
        max-attempts: 10           # Eureka 등록 전에 시도할 수 있어 충분히 설정

eureka:
  client:
    service-url:
      defaultZone: http://eureka:8761/eureka/
```

```java
// ConfigServerInstanceProvider.java (내부)
// Eureka에서 "config-server" 이름으로 인스턴스 목록 조회
// → 인스턴스 중 하나를 선택하여 URI 동적 결정

// 동작 순서:
// 1. Eureka에서 config-server 인스턴스 목록 조회
// 2. 랜덤 또는 Round Robin으로 하나 선택
// 3. 선택된 인스턴스의 IP:Port를 Config Server URI로 사용
// 4. 해당 인스턴스 장애 시 다음 요청에서 다른 인스턴스 선택
```

### 3. 로컬 Git clone 영구 캐시

```yaml
# Config Server application.yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/org/config-repo
          
          # 로컬 clone 저장 위치를 명시적으로 지정
          # 기본값: /tmp/config-repo-{random} → 재시작 시 삭제됨
          basedir: /var/config-repo-cache  # 영구 볼륨 마운트 필요
          
          # 클론이 있으면 fetch, 없으면 clone
          # clone-on-start: false → 첫 요청 시 clone
          clone-on-start: true
          
          # Git 서버 다운 시 로컬 clone으로 서비스 (stale-if-error 패턴)
          force-pull: false  # 로컬 캐시가 있으면 그것 사용 허용
```

```yaml
# docker-compose.yml
services:
  config-server:
    volumes:
      - config-repo-cache:/var/config-repo-cache   # 영구 볼륨

volumes:
  config-repo-cache:    # Docker 명명 볼륨 (컨테이너 재시작에도 유지)
```

```
로컬 캐시 동작:

정상 상태:
  요청 → refresh-rate 확인 → Git fetch → 로컬 clone에서 파일 읽기

Git 서버 다운:
  요청 → refresh-rate 확인 → Git fetch 시도 → 실패
  → 마지막 성공한 로컬 clone 파일에서 응답 (stale 설정)
  → 설정이 최신은 아니지만 서비스 가용성 유지

Config Server 재시작 (Git 서버 정상):
  영구 볼륨에 기존 clone 존재 → clone-on-start여도 즉시 사용 가능
  → Git 서버 일시적 불가 상태에서도 시작 가능
```

### 4. Composite 저장소 — 멀티 Git 저장소 통합

```yaml
# 서비스별로 다른 Git 저장소를 사용하는 엔터프라이즈 구성
spring:
  cloud:
    config:
      server:
        composite:
          - type: git
            uri: https://github.com/org/global-config    # 전체 공통 설정
            search-paths: global

          - type: git
            uri: https://github.com/org/payment-config   # 결제 도메인 설정
            search-paths: '{application}'
            # pattern으로 라우팅: payment-*만 이 저장소에서
            # repos에서 pattern 지정 가능

          - type: native
            search-locations: classpath:/emergency-configs  # 긴급 오버라이드
            # 모든 요청에 항상 적용 (우선순위 최고)
```

```
Composite 저장소 병합 순서:
  [0] native (emergency-configs)   ← 가장 높은 우선순위
  [1] payment-config (Git)
  [2] global-config (Git)          ← 가장 낮은 우선순위

payment-service/prod 요청 시:
  native에서 매칭되는 값 먼저
  payment-config에서 payment-service-prod.yml
  global-config에서 application-prod.yml
  모두 합쳐서 하나의 Environment로 반환
```

---

## 💻 실전 구성

### K8s 환경에서 Config Server HA

```yaml
# k8s/config-server-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-server
spec:
  replicas: 2          # 2개 이상 항상 유지
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0     # 업데이트 중 0개 다운 (항상 1개 이상 운영)
      maxSurge: 1
  template:
    spec:
      containers:
      - name: config-server
        image: config-server:latest
        env:
        - name: ENCRYPT_KEY
          valueFrom:
            secretKeyRef:
              name: config-server-secret
              key: encrypt-key
        volumeMounts:
        - name: git-cache
          mountPath: /var/config-repo-cache
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8888
          initialDelaySeconds: 30   # Git clone 시간 고려
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8888
          initialDelaySeconds: 60
      volumes:
      - name: git-cache
        persistentVolumeClaim:
          claimName: config-server-git-cache
---
apiVersion: v1
kind: Service
metadata:
  name: config-server
spec:
  selector:
    app: config-server
  ports:
  - port: 8888
    targetPort: 8888
  # ClusterIP: K8s 내부에서만 접근 가능 (보안)
```

### 클라이언트 다중 Config Server URI 설정

```yaml
# 여러 Config Server URI 콤마 구분으로 지정
spring:
  cloud:
    config:
      uri: http://config-server-1:8888,http://config-server-2:8888
      # 첫 번째 URI 실패 시 두 번째 시도
      fail-fast: true
      retry:
        max-attempts: 6
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: Config Server 1번 인스턴스 장애 + 새 서비스 배포 동시 발생

```
상황:
  config-server-1: DOWN (OOM)
  config-server-2: 정상
  service-a 새 버전 K8s Rolling Update 시작

클라이언트 요청 흐름:
  새 service-a Pod 시작
  → spring.cloud.config.uri=http://config-server:8888 (K8s Service)
  → K8s Service가 config-server-2로 라우팅
  → 설정 정상 로드
  → Pod 시작 성공

config-server-1 복구 중:
  K8s Deployment: replicas: 2 → 새 config-server Pod 자동 생성
  readinessProbe 통과 후 Service에 등록
  → 완전 복구

감지 방법:
  Actuator /actuator/health 응답에서 Config Server 상태 확인 가능
  config-server-1 다운 시 config-server-1의 /health: DOWN
  Grafana 대시보드에서 Config Server 인스턴스별 health 모니터링
```

### 시나리오: GitHub 점검 시간 중 새 서비스 배포

```
GitHub 점검 (1시간):
  Git 저장소 접근 불가

Config Server with 영구 Git 캐시:
  점검 전 마지막 fetch된 설정이 /var/config-repo-cache에 존재
  → GET /service-a/prod 요청 → Git fetch 실패
  → force-pull: false → 로컬 캐시 파일 사용
  → 설정 반환 성공 (최신 아닐 수 있지만 서비스 가용)

새 서비스 배포:
  Config Client → Config Server → 로컬 캐시 응답
  → 서비스 시작 성공 (점검 시간대 배포 가능)

GitHub 점검 종료:
  다음 요청에서 Git fetch 성공
  → 자동으로 최신 설정 반영
```

---

## ⚖️ 트레이드오프

| 전략 | 복잡도 | 내장 결함 허용성 | 적합한 환경 |
|------|--------|-----------------|------------|
| **단일 인스턴스 + Retry** | 낮음 | 낮음 | 개발, 소규모 |
| **다중 인스턴스 + LB** | 중간 | 높음 | 스테이징, 소규모 prod |
| **Eureka 연동** | 중간 | 높음 | Eureka 이미 사용 중 |
| **K8s Deployment + PVC** | 높음 | 매우 높음 | K8s 환경 prod |
| **Composite + 여러 Git** | 매우 높음 | 높음 | 대규모 엔터프라이즈 |

```
캐시 신선도 vs 가용성 트레이드오프:
  force-pull: true  → 항상 최신 설정, Git 다운 시 장애 전파
  force-pull: false → 구 설정 서비스 가능, 최신 설정 반영 지연

refresh-rate:
  0      → 매 요청 Git fetch → 최신, Git 부하 높음
  30s    → 30초 캐시 → 균형
  3600s  → 1시간 캐시 → Git 부하 최소, 최신 반영 느림
  
선택 기준:
  설정 변경 빈도가 낮고 안정성이 중요 → refresh-rate 높게
  실시간 설정 변경이 중요 → refresh-rate 낮게 + Spring Cloud Bus
```

---

## 📌 핵심 정리

```
Config Server HA 핵심:

Config Server = Stateless
  → 같은 Git URI를 바라보는 다중 인스턴스 = 상태 공유 없이 HA 가능
  → 단, 암호화 키는 모든 인스턴스에서 동일해야 함

3계층 방어:
  1. Config Server 다중 인스턴스 + LB (인스턴스 장애 대응)
  2. 클라이언트 Retry (일시적 네트워크 오류 대응)
  3. 로컬 Git clone 영구 캐시 (Git 서버 장애 대응)

Eureka 연동:
  Config Server를 Eureka에 등록
  클라이언트: discovery.enabled=true
  → Config Server URI 하드코딩 불필요
  → Config Server 추가/제거 시 자동 반영

K8s 환경 권장 구성:
  Deployment replicas: 2+
  maxUnavailable: 0 (항상 하나 이상 정상)
  PVC로 Git cache 영구화
  ClusterIP Service (외부 비노출)

감시 포인트:
  Config Server /actuator/health
  Config Client: config.server.health (Actuator env에서 확인)
```

---

## 🤔 생각해볼 문제

**Q1.** Config Server를 Eureka에 등록하고 클라이언트가 `spring.cloud.config.discovery.enabled=true`로 설정했다. 그런데 Eureka 자체도 재시작 중이라면 어떤 상황이 발생하는가? 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

Eureka가 재시작 중이면 Config Client가 Eureka에 Config Server 위치를 조회할 수 없습니다. 결과적으로 Config Server를 찾지 못해 `fail-fast: true`인 경우 서비스 시작 실패가 됩니다. 이는 "닭이 먼저냐 달걀이 먼저냐" 문제입니다.

해결 전략:
1. Eureka도 다중화 (Eureka Server 2개 이상) — Eureka 자체의 HA
2. 부트스트랩 Config Server URI를 명시적으로 하나 고정 + Eureka로 추가 확장 (하이브리드)
3. K8s 환경이라면 Eureka 대신 K8s Service DNS 사용 → 단순하고 안정적

일반적으로 Eureka, Config Server, 서비스 시작 순서를 K8s `initContainers`나 `depends_on` + healthcheck로 보장합니다.

</details>

---

**Q2.** Config Server 2개 인스턴스가 있는데, 인스턴스 A가 Git fetch를 완료한 직후 인스턴스 B는 아직 fetch 중이다. 이 사이에 클라이언트 요청이 오면 어떤 상황이 발생할 수 있는가?

<details>
<summary>해설 보기</summary>

일시적인 불일치가 발생할 수 있습니다. LB가 A로 라우팅하면 새 설정을 받고, B로 라우팅하면 구 설정을 받습니다. 특히 `/actuator/refresh` 후 연속 요청에서 이 현상이 나타납니다.

실제 영향: 대부분의 설정 변경은 결과적 일관성(Eventual Consistency)으로 충분합니다. B도 `refresh-rate` 이내에 최신화됩니다. 즉각적인 일관성이 필요하다면:
1. `refresh-rate: 0` (매 요청마다 fetch)
2. 또는 L7 LB에서 sticky session 적용 (같은 클라이언트는 같은 인스턴스)

그러나 대부분의 경우 이 잠깐의 불일치는 문제가 되지 않으며, 설정 변경 자체가 빈번하지 않기 때문에 허용 가능한 트레이드오프입니다.

</details>

---

**Q3.** Config Server의 로컬 Git clone 캐시가 부패(corrupted)된 경우를 어떻게 탐지하고 복구할 수 있는가?

<details>
<summary>해설 보기</summary>

Config Server는 `force-pull: true` 설정 시 Git repository 상태 이상을 감지하면 삭제 후 재clone을 시도합니다. `JGitEnvironmentRepository.deleteBaseDirIfExists()`가 이를 처리합니다.

수동 복구 방법:
1. Config Server 인스턴스 중지
2. Git cache 디렉토리 삭제 (`rm -rf /var/config-repo-cache`)
3. 재시작 → `clone-on-start: true`이면 자동으로 재clone

예방 방법:
1. 영구 볼륨 대신 emptyDir(K8s) 사용 — 재시작마다 새로 clone (캐시 이점 없음)
2. health check에 Git 저장소 상태 포함 (커스텀 HealthIndicator)
3. `spring.cloud.config.server.git.force-pull: true` — 항상 원격 상태로 reset

감지 방법: Config Server 로그에서 `InvalidRemoteException`, `TransportException`, `JGitInternalException` 등 Git 관련 예외를 모니터링하고 알람 설정합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Encryption / Decryption](./05-encryption-decryption.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 — Service Discovery with Eureka ➡️](../service-discovery/01-service-registry-pattern.md)**

</div>
