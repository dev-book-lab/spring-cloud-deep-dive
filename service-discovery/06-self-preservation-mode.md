# Self-Preservation Mode — Eureka가 정상 인스턴스를 보호하는 메커니즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Self-Preservation Mode가 발동되는 정확한 조건과 `renewalPercentThreshold` 계산 공식은?
- Self-Preservation Mode 중 Eureka가 eviction을 중단하는 코드 경로는?
- 네트워크 파티션 상황에서 Self-Preservation이 없었다면 어떤 재앙이 벌어지는가?
- Self-Preservation Mode가 개발 환경에서 오히려 문제가 되는 이유는 무엇인가?
- `expectedNumberOfClientsSendingRenews`는 언제 계산되고, 어떻게 업데이트되는가?
- Self-Preservation 상태를 모니터링하고 알람을 설정하는 방법은?

---

## 🔍 왜 MSA에서 필요한가

### 네트워크 파티션 없이 Self-Preservation이 없다면

```
네트워크 파티션 시나리오:
  정상 인스턴스: 50개
  Eureka Server와 클라이언트 사이 네트워크 불안정 (30초간)
  → 50개 인스턴스 모두 Heartbeat 전송 실패
  → 30초 후: 아무도 죽지 않았지만 Heartbeat 미수신

Self-Preservation 없다면:
  t=90s: 50개 인스턴스 모두 lease 만료
  → Eureka Server: 전부 eviction!
  → 레지스트리 텅 빔
  → 모든 서비스 간 호출: "No instances available"
  → 시스템 전체 서비스 불능

  그런데 사실 50개 인스턴스는 다 살아있음
  네트워크만 잠깐 불안정했을 뿐

Self-Preservation 있다면:
  Eureka: "갑자기 Heartbeat가 너무 많이 줄었다 → 내 문제인가? 서비스 문제가 아닌가?"
  → eviction 중단
  → 기존 레지스트리 유지
  → 네트워크 복구 후 Heartbeat 재개 → 정상 복귀

  "불확실할 때는 아무것도 삭제하지 않는다" 원칙
```

---

## 😱 잘못된 구성

### Before: 개발 환경에서 Self-Preservation 활성화

```yaml
# ❌ 개발 환경에서 기본값(Self-Preservation 활성화) 사용
eureka:
  server:
    enable-self-preservation: true  # 기본값
```

```
개발 환경에서의 문제:

상황 1: 로컬에서 서비스 A 테스트 종료 (Ctrl+C)
  → Eureka: DELETE 수신 → 즉시 제거 (Graceful)
  잠깐... 다른 서비스 재시작
  → Heartbeat 잠시 끊김

상황 2: 개발자가 서비스를 자주 재시작
  → Heartbeat 불규칙 → Self-Preservation 발동
  → 이미 종료된 서비스가 레지스트리에 남아있음
  → 다른 서비스가 죽은 인스턴스로 계속 요청 시도
  → "왜 계속 오류가 나지? 서비스 올렸는데..."

  EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP
  WHEN THEY'RE NOT. RENEWALS ARE LESSER THAN THRESHOLD AND HENCE
  THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE.

  이 빨간 경고가 Eureka UI에 표시됨
  → 개발자들이 혼란에 빠짐
```

### Before: 프로덕션에서 Self-Preservation 비활성화

```yaml
# ❌ 프로덕션에서 Self-Preservation 비활성화
eureka:
  server:
    enable-self-preservation: false
```

```
네트워크 파티션 발생:
  → 30초간 Heartbeat 불안정
  → 정상 인스턴스 50개가 만료되어 레지스트리에서 삭제
  → 전체 서비스 장애
  → 네트워크 복구 후에도 재등록에 최대 수분 필요

  enable-self-preservation: false는 개발 환경에서만 사용!
```

---

## ✨ 올바른 패턴

### After: 환경별 Self-Preservation 설정

```yaml
# 프로덕션 (기본값 권장)
eureka:
  server:
    enable-self-preservation: true  # 기본값, 생략 가능
    renewal-percent-threshold: 0.85  # 기본값: 수신된 Heartbeat가 기대값의 85% 미만이면 발동
    renewal-threshold-update-interval-ms: 900000  # 15분마다 기대값 재계산

---
# 개발 환경
eureka:
  server:
    enable-self-preservation: false  # 개발 환경에서 비활성화
    eviction-interval-timer-in-ms: 3000  # 만료 체크 주기도 단축 (기본 60초 → 3초)

eureka:
  instance:
    lease-renewal-interval-in-seconds: 5   # 개발: Heartbeat 빠르게
    lease-expiration-duration-in-seconds: 15  # 개발: 빠른 만료
```

---

## 🔬 내부 동작 원리

### 1. Self-Preservation 발동 조건 계산

```java
// AbstractInstanceRegistry.java
// Self-Preservation 발동 여부 결정

public boolean isLeaseExpirationEnabled() {
    if (!isSelfPreservationModeEnabled()) {
        // enable-self-preservation: false → 항상 eviction 허용
        return true;
    }

    // 현재 수신 중인 Heartbeat 비율이 임계값 이상인지 확인
    return numberOfRenewsPerMinThreshold > 0
        && getNumOfRenewsInLastMin() > numberOfRenewsPerMinThreshold;
    // true: 정상 → eviction 허용
    // false: Self-Preservation 발동 → eviction 중단
}
```

```java
// numberOfRenewsPerMinThreshold 계산:
// "분당 기대 Heartbeat 수 × renewalPercentThreshold"

// 기대 Heartbeat 수 계산:
private int updateRenewsPerMinThreshold(
        int stubRenewsPerMinute,
        double renewalPercentThreshold) {
    return (int) (stubRenewsPerMinute * renewalPercentThreshold);
}

// stubRenewsPerMinute = expectedNumberOfClientsSendingRenews × (60 / 30)
// = 등록된 클라이언트 수 × 분당 Heartbeat 횟수 (30초 주기 → 2회/분)

// 예시:
//   등록된 서비스 인스턴스: 50개
//   expectedNumberOfClientsSendingRenews: 50
//   분당 기대 Heartbeat: 50 × 2 = 100회
//   renewalPercentThreshold: 0.85
//   numberOfRenewsPerMinThreshold: 100 × 0.85 = 85

// 실제 수신 Heartbeat가 85회/분 미만 → Self-Preservation 발동
```

### 2. expectedNumberOfClientsSendingRenews 업데이트 시점

```java
// PeerAwareInstanceRegistryImpl.java

// ① 서비스 등록 시 증가
@Override
public void register(InstanceInfo info, final boolean isReplication) {
    super.register(info, leaseDuration, isReplication);
    replicateToPeers(Action.Register, ...);
}

// AbstractInstanceRegistry.register() 내부
// 새 인스턴스 등록 시:
if (existingLease == null) {
    // 새 등록 → expectedNumberOfClientsSendingRenews 증가
    this.expectedNumberOfClientsSendingRenews.incrementAndGet();
    updateRenewsPerMinThreshold();  // 임계값 재계산
}

// ② 서비스 취소(해제) 시 감소
@Override
public boolean cancel(String appName, String id, boolean isReplication) {
    boolean cancelled = super.cancel(appName, id, isReplication);
    if (cancelled) {
        // 인스턴스 해제 → 기대값 감소
        this.expectedNumberOfClientsSendingRenews.decrementAndGet();
        updateRenewsPerMinThreshold();
    }
    return cancelled;
}

// ③ 주기적 재계산 (15분마다)
// eviction 중에 비정상 삭제로 expectedNumberOfClientsSendingRenews가
// 실제 등록 수와 달라질 경우를 보정
private void scheduleRenewalThresholdUpdateTask() {
    timer.schedule(new TimerTask() {
        @Override
        public void run() {
            updateRenewalThreshold();  // 레지스트리 실제 크기로 재계산
        }
    }, serverConfig.getRenewalThresholdUpdateIntervalMs(),
       serverConfig.getRenewalThresholdUpdateIntervalMs());  // 기본 15분
}
```

### 3. evict() — Self-Preservation 발동 시 중단

```java
// AbstractInstanceRegistry.evict()
public void evict(long additionalLeaseMs) {
    logger.debug("Running the evict task");

    // ① isLeaseExpirationEnabled() 확인
    if (!isLeaseExpirationEnabled()) {
        // Self-Preservation Mode 발동 중
        // → eviction 완전 중단
        logger.debug("DS: lease expiration is currently disabled.");
        return;   // ← 여기서 바로 리턴, 만료 인스턴스 삭제 안 함
    }

    // ② 만료 인스턴스 수집
    List<Lease<InstanceInfo>> expiredLeases = new ArrayList<>();
    for (Entry<String, Map<String, Lease<InstanceInfo>>> groupEntry : registry.entrySet()) {
        Map<String, Lease<InstanceInfo>> leaseMap = groupEntry.getValue();
        if (leaseMap != null) {
            for (Entry<String, Lease<InstanceInfo>> leaseEntry : leaseMap.entrySet()) {
                Lease<InstanceInfo> lease = leaseEntry.getValue();
                if (lease.isExpired(additionalLeaseMs)
                        && lease.getHolder().getStatus() != InstanceStatus.OUT_OF_SERVICE) {
                    expiredLeases.add(lease);
                }
            }
        }
    }

    // ③ 안전 한도: 전체의 (1 - renewalPercentThreshold) 비율만 삭제
    int registrySize = (int) getLocalRegistrySize();
    int registrySizeThreshold = (int) (registrySize * serverConfig.getRenewalPercentThreshold());
    int evictionLimit = registrySize - registrySizeThreshold;
    // 50개 중 85%인 42개는 유지 → 최대 8개만 삭제

    int toEvict = Math.min(expiredLeases.size(), evictionLimit);
    // ...
}
```

### 4. Eureka UI의 Self-Preservation 경고

```
Eureka UI (http://eureka-server:8761)에 빨간 경고 표시:

정상 상태:
  "No instances are currently registered with Eureka"
  또는 인스턴스 목록만 표시 (경고 없음)

Self-Preservation 발동 시:
  "EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES
   ARE UP WHEN THEY'RE NOT. RENEWALS ARE LESSER THAN THRESHOLD
   AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE."

경고 발생 조건:
  getNumOfRenewsInLastMin() <= numberOfRenewsPerMinThreshold
  → 최근 1분 Heartbeat 수 ≤ 기대값의 85%

모니터링 API:
  GET /eureka/v2/apps                    → 전체 레지스트리 (XML/JSON)
  GET /actuator/metrics/eureka.server.*  → Actuator 메트릭
```

---

## 💻 실전 구성

### Self-Preservation 상태 모니터링

```yaml
# Config Server에서 Actuator 노출
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  metrics:
    export:
      prometheus:
        enabled: true  # Prometheus 연동
```

```bash
# Self-Preservation 관련 메트릭 확인
curl http://eureka-server:8761/actuator/metrics/eureka.server.renewals-in-last-min
# {"name":"eureka.server.renewals-in-last-min","measurements":[{"statistic":"VALUE","value":98.0}]}

curl http://eureka-server:8761/actuator/metrics/eureka.server.renewal-threshold-per-min
# {"name":"eureka.server.renewal-threshold-per-min","measurements":[{"statistic":"VALUE","value":85.0}]}

# Prometheus + Grafana 알람 규칙
# eureka_server_renewals_in_last_min < eureka_server_renewal_threshold_per_min
# → Self-Preservation 발동 시 알람
```

```java
// 커스텀 HealthIndicator로 Self-Preservation 상태 노출
@Component
public class EurekaSelfPreservationHealthIndicator implements HealthIndicator {

    private final PeerAwareInstanceRegistry registry;

    @Override
    public Health health() {
        boolean isSelfPreservationActive = !registry.isLeaseExpirationEnabled();
        if (isSelfPreservationActive) {
            return Health.status("SELF_PRESERVATION")
                .withDetail("message", "Self-preservation mode is active")
                .withDetail("renewals-last-min", registry.getNumOfRenewsInLastMin())
                .withDetail("renewal-threshold", registry.getNumOfRenewsPerMinThreshold())
                .build();
        }
        return Health.up()
            .withDetail("renewals-last-min", registry.getNumOfRenewsInLastMin())
            .withDetail("renewal-threshold", registry.getNumOfRenewsPerMinThreshold())
            .build();
    }
}
```

### Docker Compose: 개발 vs 프로덕션 분리

```yaml
# docker-compose.yml (개발 환경)
services:
  eureka-server:
    environment:
      - EUREKA_SERVER_ENABLE_SELF_PRESERVATION=false
      - EUREKA_SERVER_EVICTION_INTERVAL_TIMER_IN_MS=3000

  service-a:
    environment:
      - EUREKA_INSTANCE_LEASE_RENEWAL_INTERVAL_IN_SECONDS=5
      - EUREKA_INSTANCE_LEASE_EXPIRATION_DURATION_IN_SECONDS=15
```

```yaml
# docker-compose.prod.yml (프로덕션 오버라이드)
services:
  eureka-server:
    environment:
      - EUREKA_SERVER_ENABLE_SELF_PRESERVATION=true
      # 나머지는 기본값 사용

  service-a:
    environment:
      - EUREKA_INSTANCE_LEASE_RENEWAL_INTERVAL_IN_SECONDS=30
      - EUREKA_INSTANCE_LEASE_EXPIRATION_DURATION_IN_SECONDS=90
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: Self-Preservation 발동부터 해제까지

```
운영 중 상황:
  50개 인스턴스 등록, 분당 기대 Heartbeat: 100회
  numberOfRenewsPerMinThreshold: 85회

t=0s:   데이터센터 A ↔ Eureka Server 네트워크 순간 불안정
        DC-A의 30개 인스턴스 Heartbeat 전달 안 됨
        나머지 20개는 정상

t=60s:  최근 1분 Heartbeat 수: 약 40회 (20개 × 2회)
        40 < 85 → Self-Preservation 발동
        → eviction 완전 중단
        → Eureka UI에 경고 표시

t=0~90s: DC-A 인스턴스들: 정상 운영 중 (Heartbeat 못 보내는 것 뿐)
          레지스트리에 살아있음 → 트래픽 계속 수신 가능

t=90s:  네트워크 복구
        DC-A 인스턴스들 Heartbeat 재개

t=150s: 최근 1분 Heartbeat 수 정상화: 100회 > 85회
         → Self-Preservation 해제
         → 정상 eviction 재개

결과: 네트워크 파티션 90초간 발생했지만
      DC-A 30개 인스턴스는 레지스트리에서 삭제되지 않음
      → 전체 서비스 가용성 유지
```

### 시나리오: Self-Preservation과 진짜 장애의 혼재

```
문제 케이스:
  30개 인스턴스 중 15개가 진짜 다운 (메모리 부족)
  나머지 15개는 정상

Self-Preservation 발동:
  Heartbeat 수 감소 → Self-Preservation 활성화
  → 진짜 다운된 15개도 레지스트리에 유지
  → 전체 요청의 50%가 죽은 인스턴스로 라우팅
  → 오류율 50%

AP의 한계:
  Eureka는 "이게 네트워크 문제인지 실제 장애인지 모른다"
  → 보수적으로 아무것도 삭제 안 함
  
대응:
  클라이언트 Retry: 죽은 인스턴스로 가면 즉시 다른 인스턴스 재시도
  Circuit Breaker: 죽은 인스턴스 자동 격리
  
  Self-Preservation은 "데이터 손실 방지"
  실제 장애 대응은 Retry + Circuit Breaker의 역할
  → 두 메커니즘이 서로 보완
```

---

## ⚖️ 트레이드오프

| 환경 | Self-Preservation | 이유 |
|------|------------------|------|
| **프로덕션** | `true` (기본값 유지) | 네트워크 파티션에서 정상 인스턴스 보호 |
| **스테이징** | `true` | 프로덕션 동작 검증 |
| **개발** | `false` | 빠른 인스턴스 상태 반영, 개발 속도 |
| **테스트** | `false` | 결정론적 동작, 테스트 예측 가능성 |

```
renewalPercentThreshold 조정:

0.85 (기본값):
  85% 미만 수신 시 발동
  약 15% 이상 인스턴스가 동시에 Heartbeat 실패하면 Self-Preservation

0.5로 낮추면:
  더 보수적: 50% 미만 수신 시 발동
  네트워크 장애 허용 범위 넓음
  단, 실제 장애 시에도 오래 레지스트리 유지

0.95로 높이면:
  더 민감: 5% 감소에도 Self-Preservation
  개발 환경 재시작에도 바로 발동 → 개발 불편

권장: 기본값(0.85) 유지
개발 환경에서는 enable-self-preservation: false로 해결
```

---

## 📌 핵심 정리

```
Self-Preservation Mode 핵심:

목적:
  네트워크 파티션 시 정상 인스턴스가 잘못 삭제되는 것 방지
  "불확실할 때는 삭제하지 않는다" 원칙

발동 조건:
  실제 수신 Heartbeat/분 < expectedRenewals/분 × renewalPercentThreshold(0.85)
  
  예: 50개 인스턴스 → 기대 100회/분 → 임계 85회/분
      실수신 80회 < 85 → Self-Preservation 발동

발동 결과:
  evict() 완전 중단 → 만료 인스턴스 삭제 안 함
  Eureka UI 빨간 경고 표시

해제 조건:
  실수신 Heartbeat가 임계값 이상 회복

환경별 전략:
  프로덕션: enable-self-preservation=true (기본값)
  개발: enable-self-preservation=false + 짧은 lease 시간

AP 특성 보완:
  Self-Preservation: 네트워크 파티션 시 레지스트리 보호
  Retry + Circuit Breaker: 죽은 인스턴스 요청 실패 대응
  두 메커니즘의 조합이 완전한 탄력성을 제공
```

---

## 🤔 생각해볼 문제

**Q1.** Self-Preservation이 발동된 상태에서 진짜 장애 인스턴스 10개가 있다. 운영자가 이 인스턴스들을 즉시 레지스트리에서 제거하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

Self-Preservation 중 자동 eviction은 중단되지만, 수동 조작은 여전히 가능합니다.

1. **각 인스턴스의 Actuator 사용** (Graceful): `DELETE /eureka/apps/{appName}/{instanceId}` 또는 `POST /actuator/serviceregistry`로 `OUT_OF_SERVICE` 전환
2. **Eureka REST API 직접 호출**: `DELETE http://eureka-server:8761/eureka/apps/{appName}/{instanceId}`
3. **임시로 Self-Preservation 비활성화**: Eureka Server의 `enable-self-preservation=false`로 변경 후 재시작 → 자동 eviction 재개 → 다시 활성화 (위험, 권장하지 않음)

실제 장애 대응에서 Self-Preservation이 문제가 된다면 Retry와 Circuit Breaker로 죽은 인스턴스 요청을 차단하는 것이 더 현실적입니다.

</details>

---

**Q2.** `expectedNumberOfClientsSendingRenews`는 어떤 경우에 실제 등록 인스턴스 수와 어긋날 수 있는가? 이때 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

어긋나는 경우:
1. **비정상 종료**: 인스턴스가 `DELETE` 없이 죽으면 `decrementAndGet()`이 호출되지 않습니다. `expectedNumberOfClientsSendingRenews`는 그대로인데 실제 Heartbeat 수는 줄어, 기대값이 과대평가됩니다.
2. **Eureka Server 재시작**: 인메모리 상태가 초기화되어 `expectedNumberOfClientsSendingRenews = 0`에서 시작합니다. 클라이언트들이 재등록하면서 증가하지만 타이밍 차이가 발생합니다.
3. **Peer Replication 지연**: 다중 Eureka에서 피어 간 등록 복제가 지연되면 일시적으로 값이 다를 수 있습니다.

문제: 기대값이 과대평가되면 정상적인 Heartbeat 비율이 임계값 아래로 떨어져 Self-Preservation이 불필요하게 발동됩니다. 이를 보정하기 위해 `renewalThresholdUpdateIntervalMs`(기본 15분)마다 레지스트리 실제 크기로 재계산합니다.

</details>

---

**Q3.** Self-Preservation Mode가 활성화된 채로 새 서비스 배포가 완료되어 인스턴스가 추가됐다. 이때 `expectedNumberOfClientsSendingRenews`는 어떻게 변화하는가?

<details>
<summary>해설 보기</summary>

새 인스턴스가 `register()`를 호출하면 `register()` 내부에서 `expectedNumberOfClientsSendingRenews.incrementAndGet()`이 호출되고 `updateRenewsPerMinThreshold()`로 임계값이 재계산됩니다. Self-Preservation Mode가 활성화되어 있어도 새 등록은 정상적으로 처리됩니다 — Self-Preservation은 eviction(삭제)만 막을 뿐, 신규 등록(register)은 막지 않습니다.

단, 기대값이 증가하면서 임계값도 올라가므로, Self-Preservation 해제를 위해 필요한 최소 Heartbeat 수도 증가합니다. 인스턴스가 늘어날수록 전체 Heartbeat 수도 늘어나므로 결국은 임계값을 넘어 Self-Preservation이 해제됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: Client-Side Load Balancing](./05-client-side-load-balancing.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 3 — API Gateway ➡️](../api-gateway/01-gateway-vs-zuul.md)**

</div>
