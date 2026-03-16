# Eureka Client 등록 과정 (Heartbeat) — DiscoveryClient 초기화부터 상태 전이까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `CloudEurekaClient`(Spring Cloud)가 `DiscoveryClient`(Netflix)를 초기화하는 시점은 언제인가?
- `register()` → `renew()` → `cancel()` 각 HTTP 요청의 메서드와 URL은 무엇인가?
- `InstanceInfo.InstanceStatus`가 `STARTING → UP` 으로 전이되는 조건과 시점은?
- Heartbeat 스케줄러가 `renew()` 실패 시 취하는 복구 동작은?
- Heartbeat 주기가 30초, 만료 시간이 90초로 설정된 배경은 무엇인가?
- 서비스가 Graceful Shutdown될 때 Eureka 등록 해제는 어떤 순서로 일어나는가?

---

## 🔍 왜 MSA에서 필요한가

### Heartbeat 없이 레지스트리를 신뢰할 수 없는 이유

```
레지스트리에 등록된 정보 = "지금 이 서비스가 이 주소에서 동작 중"이라는 주장

이 주장이 계속 유효한지 어떻게 확인하나?

방법 1: Active Health Check (서버가 주기적으로 클라이언트 호출)
  Eureka Server가 모든 인스턴스에 /health 호출
  단점:
    인스턴스 수 × 호출 주기 = Eureka Server 부하
    클라이언트 방화벽/네트워크 정책 고려 필요
    응답 없어도 네트워크 문제인지 서비스 문제인지 구분 어려움

방법 2: Passive Heartbeat (클라이언트가 주기적으로 서버에 갱신)
  클라이언트가 "나 아직 살아있어" 신호를 주기적으로 전송
  일정 시간 신호 없으면 서버가 레지스트리에서 제거
  Eureka의 선택: 이 방법
  
  장점:
    Eureka Server 부하 낮음 (HTTP PUT 1개 수신)
    클라이언트가 네트워크 문제 인지 가능
    분산 시스템에서 더 탄력적
```

---

## 😱 잘못된 구성

### Before: 너무 짧은 Heartbeat/Lease 설정

```yaml
# ❌ 지나치게 공격적인 설정
eureka:
  instance:
    lease-renewal-interval-in-seconds: 1    # 1초마다 Heartbeat
    lease-expiration-duration-in-seconds: 3  # 3초 만료
```

```
문제:
  1초마다 PUT /eureka/apps/... 요청
  수십 개 인스턴스 × 1초 = Eureka Server에 초당 수십 번 요청
  → Eureka Server CPU/네트워크 과부하

  JVM GC pause가 3초 이상 발생하면?
  → Heartbeat 3초 못 보냄 → 정상 인스턴스가 레지스트리에서 삭제!
  → 서비스 순간 중단

  배포 환경에서 잠깐의 네트워크 지연에도 인스턴스 삭제
  → 극도로 불안정한 레지스트리
```

### Before: register-with-eureka: false인데 다른 서비스에서 호출

```yaml
# ❌ 자신은 등록 안 하면서 다른 서비스에서 이 서비스를 직접 호출
eureka:
  client:
    register-with-eureka: false   # 레지스트리에 등록 안 함
    fetch-registry: true          # 다른 서비스는 조회

# 호출하는 서비스:
# @LoadBalanced RestTemplate → SERVICE-X 이름으로 호출
# → Eureka에서 SERVICE-X 조회 → 없음 → IllegalStateException
```

---

## ✨ 올바른 패턴

### After: DiscoveryClient 초기화 전체 흐름

```
Spring Boot 시작

  ① AutoConfiguration
     EurekaClientAutoConfiguration 로드
     → CloudEurekaClient Bean 생성
     → Netflix DiscoveryClient 초기화

  ② DiscoveryClient.initScheduledTasks()
     → HeartbeatThread 스케줄러 시작 (30초 주기)
     → CacheRefreshThread 스케줄러 시작 (30초 주기)

  ③ register() 호출 (초기 등록)
     POST /eureka/v2/apps/SERVICE-A
     Body: InstanceInfo (JSON)
     초기 상태: STARTING

  ④ Spring Boot HealthIndicator 모두 UP
     → ApplicationInfoManager.setInstanceStatus(UP)
     → InstanceInfo 상태 갱신
     → 다음 Heartbeat에서 UP 상태 전파

  ⑤ Heartbeat (30초마다)
     PUT /eureka/v2/apps/SERVICE-A/{instanceId}
     → 200 OK: 정상
     → 404 Not Found: 서버에서 제거됨 → 재등록(register) 실행

  서비스 종료 시:
  ⑥ ShutdownHook 실행
     → DiscoveryClient.shutdown()
     → cancel() 호출: DELETE /eureka/v2/apps/SERVICE-A/{instanceId}
```

---

## 🔬 내부 동작 원리

### 1. EurekaClientAutoConfiguration — Spring Cloud 진입점

```java
// EurekaClientAutoConfiguration.java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties
@ConditionalOnClass(EurekaClientConfig.class)
@ConditionalOnProperty(value = "eureka.client.enabled", matchIfMissing = true)
public class EurekaClientAutoConfiguration {

    @Bean(destroyMethod = "shutdown")  // shutdown()을 빈 소멸자로 등록
    @ConditionalOnMissingBean(value = EurekaClient.class, ...)
    public EurekaClient eurekaClient(ApplicationInfoManager manager,
            EurekaClientConfig config, ...) {
        return new CloudEurekaClient(manager, config, this.optionalArgs, this.context);
    }

    @Bean
    @ConditionalOnMissingBean(value = ApplicationInfoManager.class, ...)
    public ApplicationInfoManager eurekaApplicationInfoManager(
            EurekaInstanceConfig config) {
        // InstanceInfo 생성 (레지스트리에 등록될 정보)
        InstanceInfo instanceInfo = new InstanceInfoFactory().create(config);
        return new ApplicationInfoManager(config, instanceInfo);
    }
}
```

### 2. DiscoveryClient 초기화와 스케줄러 시작

```java
// DiscoveryClient.java (Netflix eureka-client)
DiscoveryClient(ApplicationInfoManager applicationInfoManager,
        final EurekaClientConfig config, ...) {

    // ① 설정 초기화
    this.config = config;
    this.applicationInfoManager = applicationInfoManager;

    // ② Eureka Server URL 목록 초기화
    this.eurekaServiceUrls = new AtomicReference<>();

    // ③ 초기 레지스트리 fetch (Full fetch)
    if (clientConfig.shouldFetchRegistry()) {
        fetchRegistry(false);  // false = full fetch (delta 아님)
    }

    // ④ Eureka Server에 등록
    if (clientConfig.shouldRegisterWithEureka()) {
        register();
    }

    // ⑤ 스케줄러 시작
    initScheduledTasks();
}

private void initScheduledTasks() {
    // 레지스트리 갱신 스케줄러
    if (clientConfig.shouldFetchRegistry()) {
        scheduler.schedule(new TimedSupervisorTask(
            "cacheRefresh",
            scheduler,
            cacheRefreshExecutor,
            registryFetchIntervalSeconds,  // 기본 30초
            TimeUnit.SECONDS,
            expBackOffBound,
            new CacheRefreshThread()
        ), registryFetchIntervalSeconds, TimeUnit.SECONDS);
    }

    // Heartbeat 스케줄러
    if (clientConfig.shouldRegisterWithEureka()) {
        heartbeatTask = new TimedSupervisorTask(
            "heartbeat",
            scheduler,
            heartbeatExecutor,
            renewalIntervalInSecs,         // 기본 30초
            TimeUnit.SECONDS,
            expBackOffBound,               // 실패 시 지수 백오프
            new HeartbeatThread()
        );
        scheduler.schedule(heartbeatTask, renewalIntervalInSecs, TimeUnit.SECONDS);
    }
}
```

### 3. register() — 초기 등록 HTTP 요청

```java
// DiscoveryClient.register()
boolean register() throws Throwable {
    logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
    EurekaHttpResponse<Void> httpResponse;
    try {
        // POST /eureka/v2/apps/{appName}
        // Body: InstanceInfo (JSON)
        httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
    } catch (Exception e) {
        logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier, e.getMessage(), e);
        throw e;
    }
    boolean isSuccessful = httpResponse.getStatusCode() == 204;
    logger.info(PREFIX + "{} - registration status: {}", appPathIdentifier, httpResponse.getStatusCode());
    return isSuccessful;
}

// 실제 HTTP 요청 (AbstractJerseyEurekaHttpClient)
@Override
public EurekaHttpResponse<Void> register(InstanceInfo info) {
    String urlPath = "apps/" + info.getAppName();
    ClientResponse response = null;
    response = jerseyClient.resource(serviceUrl)
        .path(urlPath)
        .accept(MediaType.APPLICATION_JSON_TYPE)
        .type(MediaType.APPLICATION_JSON_TYPE)
        .post(ClientResponse.class, info);  // POST 요청
    return anEurekaHttpResponse(response.getStatus())
        .headers(headersOf(response))
        .build();
}
```

### 4. HeartbeatThread — 갱신과 재등록

```java
// DiscoveryClient.HeartbeatThread
private class HeartbeatThread implements Runnable {
    public void run() {
        if (renew()) {
            lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
        }
    }
}

boolean renew() {
    EurekaHttpResponse<InstanceInfo> httpResponse;
    try {
        // PUT /eureka/v2/apps/{appName}/{instanceId}
        httpResponse = eurekaTransport.registrationClient.sendHeartBeat(
            instanceInfo.getAppName(),
            instanceInfo.getId(),
            instanceInfo,
            null
        );

        if (httpResponse.getStatusCode() == 404) {
            // 서버에서 인스턴스를 모름 (서버 재시작 등)
            // → 재등록 필요
            REREGISTER_COUNTER.increment();
            long timestamp = instanceInfo.setIsDirtyWithTime();
            boolean success = register();  // 재등록 시도
            if (success) {
                instanceInfo.unsetIsDirty(timestamp);
            }
            return success;
        }
        return httpResponse.getStatusCode() == 200;
    } catch (Throwable e) {
        logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
        return false;
    }
}
```

### 5. InstanceStatus 전이 — STARTING → UP

```java
// EurekaAutoServiceRegistration.java (Spring Cloud)
// SmartLifecycle 구현 → Spring ApplicationContext 준비 완료 후 호출

@Override
public void start() {
    // Spring Context가 완전히 준비된 후 호출됨
    if (this.port.get() != 0) {
        if (this.registration.getNonSecurePort() == 0) {
            this.registration.setNonSecurePort(this.port.get());
        }
        if (this.registration.getSecurePort() == 0 && this.registration.isSecure()) {
            this.registration.setSecurePort(this.port.get());
        }
    }
    // EurekaServiceRegistry.register() → ApplicationInfoManager.setInstanceStatus(UP)
    this.serviceRegistry.register(this.registration);
    this.context.publishEvent(new InstanceRegisteredEvent<>(this, this.registration.getInstanceConfig()));
    this.running.set(true);
}

// ApplicationInfoManager.setInstanceStatus()
public synchronized void setInstanceStatus(InstanceStatus status) {
    InstanceStatus prev = instanceInfo.getStatus();
    if (prev != status) {
        // InstanceInfo 상태 변경
        instanceInfo.setStatus(status);
        // 상태 변경 리스너에게 통보 (Heartbeat 스레드가 이를 감지해 서버에 전달)
        if (listeners != null) {
            for (StatusChangeListener listener : listeners.values()) {
                listener.notify(new StatusChangeEvent(prev, status));
            }
        }
    }
}
```

### 6. Graceful Shutdown 등록 해제

```java
// DiscoveryClient.shutdown()
@PreDestroy
public synchronized void shutdown() {
    if (isShutdown.compareAndSet(false, true)) {
        logger.info("Shutting down DiscoveryClient...");

        // ① 스케줄러 중단 (Heartbeat 스레드 종료)
        cancelScheduledTasks();

        // ② Eureka Server에 등록 해제 요청
        if (shouldRegisterWithEureka() && shouldUnregisterOnShutdown()) {
            unregister();
        }
        // ③ HTTP 클라이언트 종료
        eurekaTransport.shutdown();
    }
}

void unregister() {
    // DELETE /eureka/v2/apps/{appName}/{instanceId}
    EurekaHttpResponse<Void> httpResponse = eurekaTransport.registrationClient.cancel(
        instanceInfo.getAppName(), instanceInfo.getId());
    logger.info("DiscoveryClient_{} - deregister  status: {}", appPathIdentifier, httpResponse.getStatusCode());
}
```

### 7. 30초 / 90초의 배경

```
Netflix 운영 경험에서 도출된 값:

Heartbeat 주기 30초:
  너무 짧으면: 불필요한 네트워크 트래픽 + Eureka Server 부하
  너무 길면: 인스턴스 상태 변경이 늦게 반영
  30초: "서비스 상태가 1분 이내에 전파됨" 이라는 SLO에서 역산
  
Lease 만료 90초 (= Heartbeat × 3):
  일시적 네트워크 이슈 허용:
    30초: 1번 실패 허용 → 아직 만료 안 됨
    60초: 2번 실패 허용 → 아직 만료 안 됨
    90초: 3번 연속 실패 → "이 인스턴스는 진짜 죽었다"로 판단
  
  GC pause, 네트워크 일시 불안정 등으로
  Heartbeat 1~2번 못 보내는 상황을 허용하는 설계
  
  "3번 연속 실패면 레지스트리에서 제거해도 된다"는 운영 경험치
```

---

## 💻 실전 구성

### Actuator로 Eureka 상태 확인

```bash
# 현재 인스턴스 상태 확인
curl http://localhost:8080/actuator/info
# 응답:
# {
#   "app": {...},
#   ...
# }

# Eureka 등록 정보 확인
curl http://localhost:8080/actuator/env | jq '.'

# Eureka Server에서 직접 확인
curl http://localhost:8761/eureka/apps/SERVICE-A | \
  python3 -c "import sys,json; d=json.load(sys.stdin); \
  [print(i['instanceId'], i['status'], i['lastUpdatedTimestamp']) \
  for i in d['application']['instance']]"
```

### Graceful Shutdown 설정 (Spring Boot 2.3+)

```yaml
# application.yml
server:
  shutdown: graceful              # 진행 중인 요청 완료 후 종료

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # 종료 대기 시간

# Eureka 등록 해제 + 진행 요청 완료 순서:
# 1. SIGTERM 수신
# 2. DiscoveryClient.shutdown() → Eureka DELETE 요청
# 3. 기존 요청 처리 완료 대기 (최대 30초)
# 4. 프로세스 종료
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: Heartbeat 실패 → 재등록 흐름

```
t=0s:   Eureka Server 재시작 (레지스트리 초기화)

t=30s:  service-a HeartbeatThread 실행
        PUT /eureka/apps/SERVICE-A/{id}
        → 404 Not Found (서버 레지스트리에 없음)
        → renew() 내부: register() 재호출
        → POST /eureka/apps/SERVICE-A (재등록)
        → 200 OK (등록 성공)

t=60s:  service-b HeartbeatThread 실행 (30초 주기)
        → 동일하게 재등록

결과:
  Eureka Server 재시작 후 최대 30초 × 서비스 수 안에
  모든 인스턴스가 자동으로 재등록됨
  레지스트리 자동 복구
```

### 시나리오: 배포 중 일시적 Heartbeat 실패

```
상황: 네트워크 설정 변경으로 30~45초간 Heartbeat 전달 불가

t=0s:   Heartbeat 차단 시작
t=30s:  1차 Heartbeat 실패 (renew() false)
t=60s:  2차 Heartbeat 실패
t=90s:  lease-expiration 도달 → Eureka Server가 인스턴스 제거
        클라이언트들: 다음 fetch에서 인스턴스 사라짐
        → 이 인스턴스로의 요청 없음

t=95s:  네트워크 정상화
t=120s: 3차 Heartbeat 시도 → 404 → register() 재등록
        → 클라이언트들: 30초 후 fetch에서 다시 인식

총 서비스 중단 영향: 약 30초 (t=90s ~ t=120s)
```

---

## ⚖️ 트레이드오프

| 설정 | 빠른 실패 감지 | 안정성 | 적합 환경 |
|------|--------------|--------|----------|
| `renewal: 5s / expiration: 15s` | 매우 빠름 (15초) | 낮음 (GC pause에 취약) | 개발/테스트 |
| `renewal: 10s / expiration: 30s` | 빠름 (30초) | 중간 | 스테이징 |
| `renewal: 30s / expiration: 90s` | 중간 (90초) | 높음 (기본값) | 프로덕션 |

```
Graceful Shutdown vs 강제 종료:

Graceful (SIGTERM):
  Eureka DELETE → 즉시 레지스트리에서 제거
  → 다음 클라이언트 fetch (최대 30초)에서 반영
  → 이후 이 인스턴스로 요청 없음 ✅

강제 종료 (SIGKILL, OOM Kill):
  Eureka DELETE 없음
  → lease-expiration (90초) 경과 후 서버에서 제거
  → 최대 90s + 30s = 120초간 죽은 인스턴스로 요청 가능 ⚠️
  → Retry가 필수인 이유
```

---

## 📌 핵심 정리

```
Eureka Client 등록 핵심:

초기화 순서:
  DiscoveryClient 생성
  → fetchRegistry() (전체 레지스트리 로드)
  → register() (자신 등록, 상태: STARTING)
  → initScheduledTasks() (Heartbeat + CacheRefresh 스케줄러)
  → Spring Context 준비 완료 → setInstanceStatus(UP)

Heartbeat 흐름:
  30초마다 PUT /eureka/apps/{appName}/{instanceId}
  → 200 OK: 정상
  → 404: 서버에서 제거됨 → 자동 재등록

InstanceStatus 전이:
  STARTING → UP (Spring Context 준비 완료 시)
  UP → OUT_OF_SERVICE (운영자 강제 설정)
  → DOWN (Health Check 실패, 간접적)

만료 타이밍:
  lease-renewal: 30초 (Heartbeat 주기)
  lease-expiration: 90초 (3회 연속 실패 허용)
  이 값들이 기본값인 이유: Netflix 운영 경험치

Graceful Shutdown:
  PreDestroy → unregister() → DELETE /eureka/apps/...
  → 즉시 레지스트리에서 제거
```

---

## 🤔 생각해볼 문제

**Q1.** `InstanceStatus`가 `STARTING`인 동안에는 해당 인스턴스로 트래픽이 라우팅되지 않는다. Eureka 클라이언트는 어떻게 이를 구현하는가?

<details>
<summary>해설 보기</summary>

Eureka 클라이언트의 `DiscoveryClient.getInstancesByVipAddress()`를 통해 인스턴스 목록을 가져올 때, 내부적으로 `InstanceInfo.InstanceStatus.UP` 상태인 인스턴스만 반환합니다. `ApplicationsUtils.getInstancesByVipAddress()`에서 `status == InstanceStatus.UP` 조건으로 필터링합니다. 따라서 `STARTING` 상태의 인스턴스는 조회 결과에 포함되지 않아 LoadBalancer가 선택하지 않습니다. Spring Cloud LoadBalancer의 `HealthCheckServiceInstanceListSupplier`도 마찬가지로 `UP` 상태만 반환합니다.

</details>

---

**Q2.** `TimedSupervisorTask`로 HeartbeatThread를 감싸는 이유는 무엇인가? 일반 ScheduledExecutorService.scheduleAtFixedRate()와 무엇이 다른가?

<details>
<summary>해설 보기</summary>

`scheduleAtFixedRate()`은 태스크가 예외로 종료되면 이후 스케줄이 영구적으로 취소됩니다. Heartbeat 스레드에서 네트워크 예외가 발생하면 Heartbeat 자체가 멈춰버리는 치명적 문제가 생깁니다.

`TimedSupervisorTask`는 이를 보완합니다. 태스크 실패 시 지수 백오프(exponential backoff)로 재시도 주기를 늘리되 최대값(`expBackOffBound` × 기본 주기)을 넘지 않게 합니다. 또한 태스크가 타임아웃되면 강제 중단하고 다음 주기에 새 태스크를 시작합니다. 이를 통해 일시적 장애 시 Eureka Server에 과도한 재시도 폭풍을 방지하면서, Heartbeat가 영구적으로 멈추는 것도 방지합니다.

</details>

---

**Q3.** K8s 환경에서 `PreDestroy`(Eureka unregister)와 K8s `readinessProbe` 실패가 거의 동시에 발생할 때 문제가 없는가? 어떤 순서로 트래픽이 차단되어야 이상적인가?

<details>
<summary>해설 보기</summary>

이상적인 순서는 다음과 같습니다.

1. K8s가 SIGTERM 전송
2. **readinessProbe를 먼저 실패**시켜 K8s Service에서 Pod 제거 (K8s 로드밸런서 레벨)
3. Eureka에서도 등록 해제 (Spring Cloud LoadBalancer 레벨)
4. 진행 중인 요청 완료 대기 (Graceful Shutdown)
5. 프로세스 종료

실제로 `PreDestroy`와 readinessProbe 실패가 동시에 일어나면 K8s Service가 Pod를 제거하기 전에 Eureka에서 먼저 제거될 수 있습니다. 이를 보완하려면 `lifecycle.preStop` 훅에 `sleep`을 추가하거나, `EurekaInstanceConfigBean.setInstanceEnabledOnIt(false)` → Spring Actuator `/actuator/serviceregistry` API로 `OUT_OF_SERVICE` 전환을 먼저 하고, K8s가 트래픽을 차단한 후 종료하는 순서를 만들어야 합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Eureka Server 내부 구조](./02-eureka-server-internals.md)** | **[홈으로 🏠](../README.md)** | **[다음: Service Instance Metadata ➡️](./04-service-instance-metadata.md)**

</div>
