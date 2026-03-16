# Service Registry 패턴 — 동적 환경에서 서비스를 찾는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- DNS 기반 서비스 디스커버리가 MSA 환경에서 부적합한 구체적인 이유는 무엇인가?
- Client-Side Discovery와 Server-Side Discovery의 실행 주체와 책임 분리는 어떻게 다른가?
- Eureka가 CP가 아닌 AP를 선택한 이유는 무엇이며, 이로 인해 발생하는 트레이드오프는?
- Service Registry에 등록되는 정보의 구조(InstanceInfo)는 어떻게 구성되는가?
- Consul, ZooKeeper와 비교했을 때 Eureka의 설계 철학 차이는 무엇인가?
- 서비스가 비정상 종료됐을 때 레지스트리에서 제거되기까지 얼마나 걸리는가?

---

## 🔍 왜 MSA에서 필요한가

### 단일 서비스 → MSA 전환 시 발생하는 주소 관리 문제

```
모놀리식 환경:
  클라이언트 → nginx(고정 IP) → 서버
  nginx.conf에 upstream 하드코딩
  서버 IP 바뀌면 nginx.conf 수정 후 reload

MSA + 클라우드 환경:
  서비스 인스턴스 수: 수십 ~ 수백 개
  인스턴스 IP: 컨테이너/Pod 재시작마다 변경
  인스턴스 수: Auto Scaling으로 동적 변동
  
  문제:
    "Order Service를 호출하려는데 IP가 뭐지?"
    10분 전에는 10.0.1.5였는데 지금은 Pod가 재시작됐을 수도 있음
    Scale-out으로 인스턴스가 3개가 됐는데 어느 쪽으로 보내야 하나?
    
  하드코딩 + nginx로는 해결 불가능
  → 동적으로 서비스 위치를 알아내는 메커니즘 필요
```

---

## 😱 잘못된 구성

### Before 1: 설정 파일에 서비스 URL 하드코딩

```yaml
# ❌ application.yml
services:
  order-service:
    url: http://10.0.1.5:8080    # Pod 재시작하면 IP 바뀜
  payment-service:
    url: http://10.0.1.8:8081    # Scale-out하면 어느 인스턴스?
  inventory-service:
    url: http://10.0.1.12:8082
```

```java
// ❌ 코드에서 직접 URL 사용
@Value("${services.order-service.url}")
private String orderServiceUrl;

// 문제:
// IP 변경 → 모든 호출 서비스의 설정 파일 수정 + 재배포
// 수십 개 서비스가 서로 의존 → 변경 연쇄 전파
// Auto Scaling 불가 → 새 인스턴스의 IP를 모름
```

### Before 2: DNS만으로 해결 시도

```
DNS 기반 접근:
  order-service.internal → 10.0.1.5 (DNS A 레코드)

문제 1: DNS TTL
  Pod 재시작 → IP 변경 → DNS 레코드 업데이트
  그러나 클라이언트가 DNS를 캐시 (TTL 60~300초)
  → 구 IP로 계속 요청 → 연결 실패
  → TTL을 1초로 낮추면? → DNS 서버 과부하

문제 2: 로드 밸런싱 불가
  DNS Round Robin: 인스턴스 health 무관하게 분배
  → 죽은 인스턴스로 요청 → 클라이언트가 직접 재시도 로직 구현 필요

문제 3: 메타데이터 없음
  어떤 인스턴스가 v2인지, 어느 AZ에 있는지 DNS에서 알 방법 없음
  → 카나리 배포, Zone-Aware 라우팅 불가
```

---

## ✨ 올바른 패턴

### After: Service Registry 패턴

```
Service Registry 핵심 개념:

Registry (등록부):
  모든 서비스 인스턴스의 현재 상태를 실시간으로 관리
  - 어떤 서비스가 어떤 IP:Port로 운영 중인가
  - 각 인스턴스의 health 상태
  - 인스턴스 메타데이터 (버전, AZ, 가중치 등)

두 가지 상호작용:
  ① 등록 (Registration): 서비스 시작 시 자신의 정보를 Registry에 등록
  ② 조회 (Discovery): 다른 서비스를 호출하기 전 Registry에서 주소 조회
```

### Client-Side vs Server-Side Discovery

```
Client-Side Discovery (Eureka 방식):
  
  Service A → Eureka에서 Order Service 인스턴스 목록 조회
  Service A → 목록에서 직접 인스턴스 선택 (로드밸런싱)
  Service A → 선택한 인스턴스로 직접 호출

  장점:
    클라이언트가 로드밸런싱 전략 자유롭게 선택
    중간 LB 서버 없음 → 지연 없음
    서비스별 맞춤 라우팅 가능 (카나리, Zone-Aware)
  
  단점:
    클라이언트에 로드밸런싱 로직 포함 → 언어/프레임워크마다 구현 필요
    Registry 클라이언트 라이브러리 의존

Server-Side Discovery (AWS ALB + ECS 방식):

  Service A → LB(로드밸런서)에 서비스 이름으로 요청
  LB → Registry 조회 → 인스턴스 선택 → 포워딩
  
  장점:
    클라이언트 단순 (LB 주소만 알면 됨)
    언어/프레임워크 무관
  
  단점:
    LB가 추가 홉 → 지연 증가
    LB 자체가 SPOF 또는 병목
    세밀한 라우팅 전략 제한
```

---

## 🔬 내부 동작 원리

### 1. Eureka의 InstanceInfo — 레지스트리에 저장되는 정보

```java
// InstanceInfo.java (netflix-eureka-client)
// Service Registry의 한 엔트리

public class InstanceInfo {
    // 식별 정보
    private String instanceId;      // "service-a:8080" 또는 "host:service-a:8080"
    private String appName;         // "SERVICE-A" (대문자로 저장)
    private String ipAddr;          // "10.0.1.5"
    private String hostName;        // "pod-abc123.cluster.local"
    private int port;               // 8080
    private int securePort;         // 8443 (HTTPS)

    // 상태
    private InstanceStatus status;          // UP, DOWN, STARTING, OUT_OF_SERVICE, UNKNOWN
    private InstanceStatus overriddenStatus; // 운영자가 강제로 설정한 상태

    // Heartbeat 추적
    private volatile long lastUpdatedTimestamp;   // 마지막 갱신 시각
    private volatile long lastDirtyTimestamp;     // 마지막 변경 시각

    // 메타데이터 (자유 형식 key-value)
    private Map<String, String> metadata;
    // 예: {"version": "v2", "zone": "ap-northeast-2a", "weight": "10"}

    // Health Check URL
    private String healthCheckUrl;  // http://10.0.1.5:8080/actuator/health
    private String homePageUrl;     // http://10.0.1.5:8080/
    private String statusPageUrl;   // http://10.0.1.5:8080/actuator/info

    public enum InstanceStatus {
        UP,              // 정상, 트래픽 받을 수 있음
        DOWN,            // 비정상
        STARTING,        // 초기화 중
        OUT_OF_SERVICE,  // 운영자가 수동으로 비활성화
        UNKNOWN          // 상태 불명
    }
}
```

### 2. CAP 정리와 Eureka의 선택

```
CAP 정리:
  분산 시스템은 Consistency(일관성), Availability(가용성),
  Partition Tolerance(분단 내성) 중 2개만 동시에 보장 가능

  네트워크 분단(P)은 분산 시스템에서 피할 수 없음
  → CP 또는 AP 중 선택해야 함

CP 선택 (ZooKeeper, Consul):
  네트워크 분단 시 → 일관성 보장을 위해 일부 노드 응답 중단
  레지스트리 데이터는 항상 정확
  단점: 분단 중 새 서비스 등록/조회 불가
  
  MSA에서의 문제:
    일시적 네트워크 이슈 → Registry 일부 응답 불가
    → 서비스 간 호출 전체 실패
    → 이미 알고 있는 구 주소로라도 호출하는 게 낫지 않을까?

AP 선택 (Eureka):
  네트워크 분단 시 → 가용성 우선 (모든 노드가 응답 계속)
  단, 각 노드가 다른 레지스트리 데이터를 가질 수 있음 (일시적 불일치)
  
  Netflix의 판단:
    "이미 알고 있는 오래된 주소 정보로 호출 시도하는 것이
     아예 호출하지 못하는 것보다 낫다"
    
    실제로 서비스 인스턴스는 자주 바뀌지 않음
    일시적 불일치는 몇 초 안에 해소됨 (Heartbeat로 갱신)
    
  결과: 네트워크 분단 중에도 캐시된 레지스트리 정보로 서비스 호출 가능
```

### 3. 서비스 등록·조회·만료 흐름

```
서비스 시작 ~ 정상 운영 ~ 종료 전체 흐름:

① 시작 (STARTING)
   DiscoveryClient 초기화
   → POST /eureka/apps/{appName} (InstanceInfo 등록)
   → 상태: STARTING

② 준비 완료 (UP)
   Spring Boot: HealthIndicator 모두 UP
   → InstanceInfo.status = UP으로 갱신
   → Eureka Server에 상태 업데이트

③ 정상 운영 (Heartbeat)
   30초마다 PUT /eureka/apps/{appName}/{instanceId}
   → "아직 살아있다" 갱신
   → Eureka Server: lastUpdatedTimestamp 갱신

④ 인스턴스 목록 캐시 (클라이언트)
   Eureka Client: 30초마다 전체 레지스트리 fetch
   또는 delta fetch (변경분만)
   → 로컬 캐시에 보관

⑤ 정상 종료 (Graceful Shutdown)
   DELETE /eureka/apps/{appName}/{instanceId}
   → Eureka Server: 즉시 레지스트리에서 제거

⑥ 비정상 종료 (프로세스 강제 종료)
   DELETE 요청 없음
   → Eureka Server: 마지막 Heartbeat로부터 90초 경과 시 제거
   → 최악의 경우 클라이언트가 90초간 죽은 인스턴스 주소 보유
```

### 4. Eureka vs 경쟁 기술 비교

```
Eureka (Netflix OSS):
  언어: Java
  프로토콜: HTTP REST
  일관성 모델: AP (가용성 우선)
  저장소: 인메모리 (영속성 없음)
  Spring Cloud 통합: 최고 수준
  특징: Self-Preservation Mode, 클라이언트 캐시

Consul (HashiCorp):
  언어: Go
  프로토콜: HTTP + gRPC
  일관성 모델: CP (Raft 합의 알고리즘)
  저장소: KV Store (영속성)
  추가 기능: Service Mesh, Secret 관리, DNS 인터페이스
  특징: Health Check를 서버가 직접 수행

ZooKeeper (Apache):
  언어: Java
  프로토콜: 자체 바이너리
  일관성 모델: CP (ZAB 프로토콜)
  저장소: ZNode 트리 (영속성)
  원래 목적: 분산 코디네이션 (Lock, Election)
  Service Discovery는 부가 용도

선택 기준:
  Spring Boot MSA, Java 팀 → Eureka (Spring Cloud 통합 최고)
  Service Mesh, 멀티 언어, 컴플라이언스 → Consul
  이미 ZooKeeper 운영 중, 강한 일관성 필요 → ZooKeeper
```

---

## 💻 실전 구성

### Docker Compose: Eureka Server + Client 기본 환경

```yaml
# docker-compose.yml
services:
  eureka-server:
    image: openjdk:17-jdk-slim
    container_name: eureka-server
    volumes:
      - ./eureka-server/target/eureka-server.jar:/app.jar
    entrypoint: ["java", "-jar", "/app.jar"]
    ports:
      - "8761:8761"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8761/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 10

  service-a:
    image: openjdk:17-jdk-slim
    container_name: service-a
    volumes:
      - ./service-a/target/service-a.jar:/app.jar
    entrypoint: ["java", "-jar", "/app.jar"]
    environment:
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/
      - SPRING_APPLICATION_NAME=service-a
    ports:
      - "8080:8080"
    depends_on:
      eureka-server:
        condition: service_healthy
```

```java
// Eureka Server 설정
@SpringBootApplication
@EnableEurekaServer   // Eureka Server 활성화
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# Eureka Server application.yml
server:
  port: 8761

spring:
  application:
    name: eureka-server

eureka:
  instance:
    hostname: eureka-server
  client:
    # 서버 자신은 레지스트리에 등록하지 않음
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    # 개발 환경: Self-Preservation 비활성화
    enable-self-preservation: false
    eviction-interval-timer-in-ms: 3000  # 만료 체크 주기 (기본 60초 → 개발 3초)
```

```yaml
# Eureka Client (service-a) application.yml
spring:
  application:
    name: service-a

eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
    # 레지스트리 fetch 주기 (기본 30초)
    registry-fetch-interval-seconds: 5   # 개발 환경에서 빠른 반영
  instance:
    # 호스트명 대신 IP로 등록 (컨테이너 환경에서 필수)
    prefer-ip-address: true
    # Heartbeat 주기 (기본 30초)
    lease-renewal-interval-in-seconds: 5
    # 만료 대기 시간 (기본 90초)
    lease-expiration-duration-in-seconds: 15
```

### Eureka Dashboard 확인

```bash
# Eureka UI: http://localhost:8761
# 등록된 인스턴스 목록 확인

# REST API로 레지스트리 조회
curl http://localhost:8761/eureka/apps | jq '.'

# 특정 서비스 조회
curl http://localhost:8761/eureka/apps/SERVICE-A | jq '.'

# 응답 예시:
# {
#   "application": {
#     "name": "SERVICE-A",
#     "instance": [{
#       "instanceId": "10.0.1.5:service-a:8080",
#       "hostName": "10.0.1.5",
#       "app": "SERVICE-A",
#       "ipAddr": "10.0.1.5",
#       "status": "UP",
#       "port": {"$": 8080, "@enabled": "true"},
#       "lastUpdatedTimestamp": 1710000000000
#     }]
#   }
# }
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 인스턴스 비정상 종료 후 레지스트리 정리까지

```
t=0s:   service-a Pod OOM Kill → 프로세스 강제 종료
        (DELETE 요청 없음)

t=0~30s: Eureka Server: 마지막 Heartbeat 기록 유지
         다른 서비스들: 로컬 캐시에 service-a 주소 보유
         → service-a로 요청하면 실패 (Connection Refused)

t=30s:  Eureka Server: "30초 이상 Heartbeat 없음" 감지
         → 아직 레지스트리에서 제거 안 됨 (eviction 주기 대기)

t=90s:  lease-expiration-duration 초과
         → Eureka Server: service-a 레지스트리에서 제거
         → 다른 서비스들: 30초 후 다음 fetch에서 제거 반영

실제 클라이언트 영향:
  service-a 종료 후 최대 120초간 (90s + 30s 캐시)
  죽은 인스턴스로 요청이 라우팅될 수 있음

대응:
  - Retry 설정 (한 번 실패해도 다른 인스턴스로 재시도)
  - Circuit Breaker (연속 실패 시 자동 차단)
  - lease 시간을 줄여 빠른 감지 (개발/테스트 환경에서 유용)
```

### 시나리오: Scale-out 후 새 인스턴스가 트래픽을 받기까지

```
t=0s:   service-a Pod 3개 → 4개로 Scale-out
        새 Pod: service-a-4 시작

t=~5s:  service-a-4: STARTING 상태로 Eureka 등록
         아직 트래픽 받지 않음 (STARTING은 제외)

t=~10s: Spring Boot HealthIndicator 모두 UP
         → service-a-4: UP 상태로 갱신

t=~40s: 다른 서비스들: 30초 후 레지스트리 fetch
         → 로컬 캐시에 service-a-4 추가
         → 이제 service-a-4에도 트래픽 라우팅 시작

registry-fetch-interval-seconds: 5 로 줄이면:
  t=~15s: 다른 서비스들이 새 인스턴스 인식
  → Scale-out 반영 지연 단축
```

---

## ⚖️ 트레이드오프

| 관점 | Client-Side Discovery (Eureka) | Server-Side Discovery (ALB) |
|------|-------------------------------|----------------------------|
| **클라이언트 복잡도** | 높음 (Registry 클라이언트 필요) | 낮음 (LB 주소만 알면 됨) |
| **지연** | 낮음 (직접 호출) | 중간 (LB 홉 추가) |
| **로드밸런싱 유연성** | 높음 (클라이언트가 전략 선택) | 중간 (LB 설정에 의존) |
| **언어 독립성** | 낮음 (언어별 클라이언트 필요) | 높음 |
| **세밀한 라우팅** | 가능 (메타데이터 활용) | 제한적 |

```
Eureka AP 선택의 현실적 함의:
  장점:
    네트워크 파티션 중에도 캐시된 레지스트리로 서비스 호출 가능
    Eureka Server 재시작 중에도 클라이언트 캐시로 동작

  단점:
    레지스트리 불일치 허용 → 죽은 인스턴스 주소가 잠시 남아있을 수 있음
    이를 보완하기 위해 Retry, Circuit Breaker가 필수적으로 필요

  "Eureka는 Retry와 Circuit Breaker 없이는 반쪽짜리다"
```

---

## 📌 핵심 정리

```
Service Registry 패턴 핵심:

해결하는 문제:
  동적으로 변하는 서비스 인스턴스의 IP:Port를
  실시간으로 추적하고 클라이언트가 조회할 수 있게 함

Eureka의 선택:
  AP (가용성 + 분단 내성) → 일시적 불일치 허용
  이유: "구 주소로 시도라도 하는 것이 아무것도 못하는 것보다 낫다"

InstanceInfo 핵심 필드:
  appName: 서비스 이름 (레지스트리 조회 키)
  status: UP / DOWN / STARTING / OUT_OF_SERVICE
  ipAddr + port: 실제 호출 주소
  metadata: 커스텀 라우팅 정보

비정상 종료 후 레지스트리 정리까지:
  lease-expiration(기본 90s) + fetch 주기(기본 30s) = 최대 120초
  → Retry + Circuit Breaker로 이 기간을 견뎌야 함
```

---

## 🤔 생각해볼 문제

**Q1.** Eureka가 AP를 선택했다는 것은 네트워크 분단 시 레지스트리 데이터가 각 노드마다 다를 수 있다는 의미다. 이 상황에서 Service A가 Service B를 호출할 때 어떤 일이 일어날 수 있는가?

<details>
<summary>해설 보기</summary>

네트워크 분단으로 Eureka Node 1과 Node 2가 분리됐다고 가정합니다. 이 사이 Service B의 새 인스턴스가 Node 1에만 등록됐다면, Node 1에 연결된 Service A는 새 인스턴스를 알지만 Node 2에 연결된 Service A는 새 인스턴스를 모릅니다. Service A들이 서로 다른 인스턴스 목록으로 로드밸런싱하는 상황이 발생합니다. 하지만 Eureka는 이를 허용합니다. 네트워크 분단이 해소되면 Peer Replication으로 결국 동기화됩니다. 이 짧은 불일치 기간 동안 일부 요청이 실패할 수 있으므로, 클라이언트 측 Retry가 필수적입니다.

</details>

---

**Q2.** `lease-expiration-duration-in-seconds`를 5초로 매우 짧게 설정했다. 어떤 장점과 위험이 있는가?

<details>
<summary>해설 보기</summary>

**장점**: 비정상 종료된 인스턴스가 5초 안에 레지스트리에서 제거됩니다. 죽은 인스턴스로 라우팅되는 기간이 극적으로 줄어듭니다.

**위험**: Eureka Server가 일시적으로 GC 멈춤(Stop-the-World GC)을 겪거나 잠깐의 네트워크 지연이 발생하면, 실제로는 정상인 인스턴스가 Heartbeat를 5초 안에 보내지 못해 레지스트리에서 삭제될 수 있습니다. 이후 Self-Preservation Mode가 없다면 다수의 정상 인스턴스가 일시적으로 제거되어 심각한 장애로 이어집니다. 프로덕션에서는 기본값(90초)에 가까운 값을 유지하고, 짧은 lease-expiration은 개발/테스트 환경에서만 사용해야 합니다.

</details>

---

**Q3.** MSA 팀의 절반은 Java/Spring, 절반은 Node.js로 개발한다. Eureka Client는 Java에 최적화되어 있는데, Node.js 서비스들도 Eureka에 등록하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

Eureka는 HTTP REST API 기반이므로 언어에 무관하게 직접 HTTP 요청으로 등록·갱신·해제가 가능합니다.

Node.js 옵션:
1. **eureka-js-client** 같은 서드파티 라이브러리 사용
2. HTTP 클라이언트로 직접 구현: `POST /eureka/apps/{appName}` (등록), `PUT /eureka/apps/{appName}/{instanceId}` (Heartbeat), `DELETE /eureka/apps/{appName}/{instanceId}` (해제)

그러나 언어 다양성이 높은 환경에서는 Consul이 더 적합할 수 있습니다. Consul은 언어 독립적인 DNS 인터페이스와 HTTP API를 제공하고, 다양한 언어의 공식 SDK가 있습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Eureka Server 내부 구조 ➡️](./02-eureka-server-internals.md)**

</div>
