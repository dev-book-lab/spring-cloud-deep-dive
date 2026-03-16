# Eureka Server 내부 구조 — PeerAwareInstanceRegistry와 레지스트리 관리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `AbstractInstanceRegistry`가 인스턴스 정보를 저장하는 자료구조는 무엇이며 왜 이 구조를 선택했는가?
- `register()`, `renew()`, `cancel()`이 각각 어떤 자료구조를 어떻게 조작하는가?
- Eureka Server가 인스턴스 만료(eviction)를 체크하는 주기와 판단 기준은 무엇인가?
- `ResponseCache`가 레지스트리 조회 성능을 어떻게 최적화하는가?
- `PeerAwareInstanceRegistry`가 `AbstractInstanceRegistry`와 다른 점은 무엇인가?
- Eureka Server 다중화 시 피어 간 복제(Peer Replication)는 어떻게 동작하는가?

---

## 🔍 왜 MSA에서 필요한가

### Eureka Server가 단순 Map이 아닌 이유

```
단순 HashMap으로 구현한다면:

  Map<String, InstanceInfo> registry = new HashMap<>();
  
  문제 1: 동시성
    수십 개 서비스가 동시에 register, renew, cancel 요청
    → 동시 쓰기 → ConcurrentModificationException
  
  문제 2: 다중 인스턴스
    "SERVICE-A"가 3개 인스턴스일 때 구조?
    Map<String, InstanceInfo> → 마지막 하나만 저장
    Map<String, List<InstanceInfo>> → List 조작의 동시성 문제
  
  문제 3: 조회 성능
    수천 번/초의 fetch 요청에 매번 전체 레지스트리 직렬화?
    → CPU 과부하
  
  Eureka가 실제로 사용하는 구조:
    이중 ConcurrentHashMap + 읽기 전용 캐시 + Delta 변경분 추적
```

---

## 😱 잘못된 구성

### Before: Eureka Server 단일 인스턴스 + 레지스트리 영속성 없음

```yaml
# ❌ 단일 인스턴스 Eureka Server
services:
  eureka-server:
    replicas: 1
    # 재시작하면 레지스트리 전부 초기화
    # 모든 서비스가 재등록할 때까지 Discovery 불가
```

```
문제:
  Eureka Server 재시작 (배포, OOM, 노드 교체)
  → 레지스트리 비어있음
  → 모든 클라이언트: "레지스트리 비어있음" 수신
  → 기존 캐시는 유지되지만, 신규 인스턴스 정보는 없음
  → 서비스 간 호출 일시 중단 가능성

  Eureka 자체가 없어도 클라이언트 캐시로 버틸 수 있지만,
  서버가 재시작되면 Self-Preservation 없이 기존 등록 정보 유실
```

---

## ✨ 올바른 패턴

### After: Eureka Server 레지스트리 자료구조 전체 구조

```
AbstractInstanceRegistry 핵심 자료구조:

registry:
  ConcurrentHashMap<String,              // appName ("SERVICE-A")
    Map<String,                          // instanceId ("10.0.1.5:service-a:8080")
      Lease<InstanceInfo>>>              // Lease: 만료 시간을 감싼 래퍼

recentlyChangedQueue:
  ConcurrentLinkedQueue<RecentlyChangedItem>
  → Delta 변경분: 등록·갱신·삭제된 항목의 최근 3분 이력
  → 클라이언트 delta fetch 시 사용 (전체 대신 변경분만 전송)

responseCache:
  ConcurrentMap<Key, Value>
  → 직렬화된 레지스트리 스냅샷 캐시 (JSON/XML)
  → 조회 성능 최적화

overriddenInstanceStatusMap:
  ConcurrentHashMap<instanceId, InstanceStatus>
  → 운영자가 강제로 설정한 인스턴스 상태
  → OUT_OF_SERVICE 강제 설정 등
```

---

## 🔬 내부 동작 원리

### 1. 레지스트리 핵심 자료구조

```java
// AbstractInstanceRegistry.java
public abstract class AbstractInstanceRegistry implements InstanceRegistry {

    // 레지스트리: appName → instanceId → Lease<InstanceInfo>
    // ConcurrentHashMap: 동시 읽기/쓰기 안전
    // 중첩 Map: 서비스명 → 인스턴스별 Lease
    private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry
        = new ConcurrentHashMap<>();

    // Lease: 만료 시간 래퍼
    public class Lease<T> {
        private T holder;                    // InstanceInfo
        private long evictionTimestamp;      // 만료 시각 (0이면 활성)
        private long registrationTimestamp;  // 등록 시각
        private long lastUpdateTimestamp;    // 마지막 Heartbeat 시각
        private long duration;               // lease 유효 기간 (기본 90초)

        public boolean isExpired() {
            return isExpired(0L);
        }

        public boolean isExpired(long additionalLeaseMs) {
            // 마지막 갱신으로부터 duration 이상 경과 → 만료
            return (evictionTimestamp > 0
                || System.currentTimeMillis() > (lastUpdateTimestamp + duration + additionalLeaseMs));
        }

        public void renew() {
            // Heartbeat 수신 시 호출
            lastUpdateTimestamp = System.currentTimeMillis() + duration;
            // duration을 더하는 이유: 시계 오차(clock skew) 허용
        }
    }
}
```

### 2. register() — 인스턴스 등록

```java
// AbstractInstanceRegistry.register()
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
    // ① appName으로 해당 서비스의 인스턴스 Map 조회
    Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
    if (gMap == null) {
        // ② 새 서비스: Map 생성 후 원자적으로 등록
        ConcurrentHashMap<String, Lease<InstanceInfo>> newMap = new ConcurrentHashMap<>();
        Map<String, Lease<InstanceInfo>> existingMap =
            registry.putIfAbsent(registrant.getAppName(), newMap);
        gMap = (existingMap != null) ? existingMap : newMap;
    }

    // ③ 기존 Lease 확인 (재등록 케이스: 서비스 재시작)
    Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());

    if (existingLease != null && existingLease.getHolder() != null) {
        // 재등록: 기존 lastDirtyTimestamp 유지 (더 최신인 경우)
        Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
        Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
        if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
            // 서버가 더 최신 상태를 알고 있음 → 클라이언트 상태 오버라이드
            registrant = existingLease.getHolder();
        }
    }

    // ④ 새 Lease 생성 및 등록
    Lease<InstanceInfo> lease = new Lease<>(registrant, leaseDuration);
    if (existingLease != null) {
        lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
    }
    gMap.put(registrant.getId(), lease);

    // ⑤ 최근 변경 큐에 추가 (Delta fetch용)
    recentlyChangedQueue.add(new RecentlyChangedItem(lease));

    // ⑥ ResponseCache 무효화 (새 인스턴스가 생겼으므로 캐시 갱신 필요)
    invalidateCache(registrant.getAppName(), ...);
}
```

### 3. renew() — Heartbeat 처리

```java
// AbstractInstanceRegistry.renew()
// 클라이언트가 30초마다 호출 (PUT /eureka/apps/{appName}/{instanceId})
public boolean renew(String appName, String id, boolean isReplication) {
    // ① 레지스트리에서 해당 인스턴스의 Lease 조회
    Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
    Lease<InstanceInfo> leaseToRenew = null;
    if (gMap != null) {
        leaseToRenew = gMap.get(id);
    }

    // ② Lease가 없으면 false 반환 → 클라이언트가 재등록
    if (leaseToRenew == null) {
        RENEW_NOT_FOUND.increment();
        return false;
    }

    // ③ Heartbeat 카운터 증가 (Self-Preservation 계산에 사용)
    renewsLastMin.increment();

    // ④ lastUpdateTimestamp 갱신
    leaseToRenew.renew();  // = System.currentTimeMillis() + duration

    return true;
}
```

### 4. evict() — 만료 인스턴스 정리

```java
// AbstractInstanceRegistry.evict()
// 스케줄러가 60초마다 호출 (evictionIntervalTimerInMs)

public void evict(long additionalLeaseMs) {
    // ① Self-Preservation Mode 확인
    if (!isLeaseExpirationEnabled()) {
        // Self-Preservation 활성화 중 → eviction 중단
        logger.debug("DS: lease expiration is currently disabled.");
        return;
    }

    // ② 만료된 Lease 수집
    List<Lease<InstanceInfo>> expiredLeases = new ArrayList<>();
    for (Map.Entry<String, Map<String, Lease<InstanceInfo>>> groupEntry : registry.entrySet()) {
        for (Map.Entry<String, Lease<InstanceInfo>> leaseEntry : groupEntry.getValue().entrySet()) {
            Lease<InstanceInfo> lease = leaseEntry.getValue();
            if (lease.isExpired(additionalLeaseMs) && lease.getHolder().getStatus() != InstanceStatus.OUT_OF_SERVICE) {
                expiredLeases.add(lease);
            }
        }
    }

    // ③ 한 번에 전체 삭제하지 않고 일부만 삭제 (Self-Preservation 보완)
    int registrySize = (int) getLocalRegistrySize();
    int registrySizeThreshold = (int) (registrySize * serverConfig.getRenewalPercentThreshold());
    int evictionLimit = registrySize - registrySizeThreshold;

    // 만료 인스턴스와 삭제 한도 중 작은 값만큼만 삭제
    int toEvict = Math.min(expiredLeases.size(), evictionLimit);
    if (toEvict > 0) {
        // ④ 랜덤 순서로 삭제 (특정 서비스가 집중 삭제되는 편향 방지)
        Collections.shuffle(expiredLeases);
        for (int i = 0; i < toEvict; i++) {
            internalCancel(expiredLeases.get(i).getHolder().getAppName(),
                expiredLeases.get(i).getHolder().getId(), false);
        }
    }
}
```

### 5. ResponseCache — 조회 성능 최적화

```java
// ResponseCacheImpl.java
// 레지스트리 전체 또는 서비스별 조회 결과를 직렬화해 캐시

public class ResponseCacheImpl implements ResponseCache {

    // 두 레벨 캐시
    private final ConcurrentMap<Key, Value> readOnlyCacheMap  = new ConcurrentHashMap<>();  // 읽기 전용 (클라이언트 응답용)
    private final LoadingCache<Key, Value> readWriteCacheMap;  // 읽기/쓰기 (Guava Cache)

    // Key: (EntityType, entityName, KeyType, version, EurekaAccept)
    // 예: (APPLICATION, "SERVICE-A", JSON, V2, COMPACT)

    public ResponseCacheImpl(...) {
        // readWriteCacheMap: 30초 TTL, 최대 1000개 항목
        this.readWriteCacheMap = CacheBuilder.newBuilder()
            .initialCapacity(serverConfig.getInitialCapacityOfResponseCache())
            .expireAfterWrite(serverConfig.getResponseCacheAutoExpirationInSeconds(), TimeUnit.SECONDS)
            .build(new CacheLoader<Key, Value>() {
                @Override
                public Value load(Key key) {
                    // 캐시 미스 시 레지스트리에서 직렬화
                    return generatePayload(key);
                }
            });

        // readOnlyCacheMap → readWriteCacheMap 동기화 (30초마다)
        scheduler.schedule(new TimerTask() {
            @Override
            public void run() {
                updateReadOnlyMap();  // 읽기 전용 캐시 갱신
            }
        }, responseCacheUpdateIntervalMs, responseCacheUpdateIntervalMs);
    }

    // 레지스트리 조회: readOnlyCacheMap 우선
    public String get(final Key key, boolean useReadOnlyCache) {
        Value payload = getValue(key, useReadOnlyCache);
        return payload == null ? null : payload.getPayload();
    }

    // 레지스트리 변경 시 관련 캐시 무효화
    public void invalidate(Key... keys) {
        for (Key key : keys) {
            readWriteCacheMap.invalidate(key);  // 다음 접근 시 재생성
        }
    }
}
```

### 6. PeerAwareInstanceRegistry — 피어 간 복제

```java
// PeerAwareInstanceRegistryImpl.java (AbstractInstanceRegistry 확장)
// Eureka Server 다중화 시 피어 간 레지스트리 동기화

public class PeerAwareInstanceRegistryImpl extends AbstractInstanceRegistry {

    @Override
    public void register(InstanceInfo info, final boolean isReplication) {
        // ① 기본 등록 (AbstractInstanceRegistry.register())
        int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
        if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
            leaseDuration = info.getLeaseInfo().getDurationInSecs();
        }
        super.register(info, leaseDuration, isReplication);

        // ② 피어에게 복제 (isReplication=false일 때만 — 무한 루프 방지)
        replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
    }

    private void replicateToPeers(Action action, String appName, String id,
            InstanceInfo info, InstanceStatus newStatus, boolean isReplication) {
        // 다른 서버로부터 복제받은 것이면 재복제 안 함
        if (isReplication) {
            return;
        }
        // 모든 Eureka 피어에게 비동기 전송
        for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
            // 자기 자신 제외
            if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                continue;
            }
            // 비동기 HTTP 요청으로 피어에게 전달
            replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
        }
    }
}
```

---

## 💻 실전 구성

### Eureka Server 다중화 구성

```yaml
# Peer 1: application-peer1.yml
server:
  port: 8761

spring:
  application:
    name: eureka-server

eureka:
  instance:
    hostname: eureka-peer1
  client:
    # 다른 Eureka 피어에 등록하고 레지스트리 공유
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka-peer2:8762/eureka/
```

```yaml
# Peer 2: application-peer2.yml
server:
  port: 8762

eureka:
  instance:
    hostname: eureka-peer2
  client:
    service-url:
      defaultZone: http://eureka-peer1:8761/eureka/
```

```yaml
# Client: 두 Eureka에 모두 등록·조회
eureka:
  client:
    service-url:
      defaultZone: http://eureka-peer1:8761/eureka/,http://eureka-peer2:8762/eureka/
```

### ResponseCache 튜닝

```yaml
eureka:
  server:
    # ResponseCache TTL (기본 30초)
    response-cache-auto-expiration-in-seconds: 10  # 더 빠른 갱신 반영
    
    # 읽기 전용 캐시 동기화 주기 (기본 30초)
    response-cache-update-interval-ms: 5000
    
    # 읽기 전용 캐시 비활성화 (항상 최신, 성능 저하)
    use-read-only-response-cache: false
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: Eureka Peer 1 다운 중 서비스 등록

```
상황: Eureka Peer1(8761) 다운, Peer2(8762) 정상

새 service-b 시작:
  → Peer1:8761 연결 시도 → 실패
  → Peer2:8762 연결 시도 → 성공
  → service-b: Peer2에만 등록

기존 service-a (Peer1 캐시 보유):
  → 로컬 캐시에 service-b 없음
  → 다음 fetch에서 Peer2에서 받음 (defaultZone에 Peer2 포함 시)

Peer1 복구:
  → Peer2로부터 레지스트리 동기화
  → service-b 정보 수신
  → Peer1도 service-b 인식

결과: 일시적 불일치 있었지만 결국 동기화됨 (AP 특성)
```

### 시나리오: ResponseCache로 인한 갱신 지연

```
t=0s:   service-a 새 인스턴스 등록 → registry에 즉시 반영
                                    → readWriteCacheMap 무효화

t=0~30s: readOnlyCacheMap에는 아직 구 버전
         클라이언트 fetch → readOnlyCacheMap 응답
         → 새 인스턴스 아직 포함 안 됨

t=30s:  readOnlyCacheMap 동기화 스케줄러 실행
        → readWriteCacheMap에서 최신 데이터 fetch
        → readOnlyCacheMap 갱신

t=30s~60s: 클라이언트 registry-fetch-interval(30초) 경과
           → 최신 레지스트리 수신
           → 새 인스턴스 인식

최악의 경우: 30s(cache sync) + 30s(client fetch) = 60초 지연
```

---

## ⚖️ 트레이드오프

| 설계 선택 | 장점 | 단점 |
|-----------|------|------|
| **ConcurrentHashMap** | 동시 읽기/쓰기 안전, 세분화된 락 | 메모리 사용량 높음 |
| **ResponseCache 두 레벨** | 읽기 성능 극대화 | 변경 반영 최대 30초 지연 |
| **인메모리 레지스트리** | 빠른 읽기/쓰기 | 서버 재시작 시 초기화 |
| **랜덤 eviction 순서** | 특정 서비스 집중 삭제 방지 | 예측 불가능한 삭제 순서 |

```
ResponseCache 지연 vs 일관성:
  use-read-only-response-cache: false
    → 항상 readWriteCacheMap 직접 조회
    → 변경 즉시 반영
    → 높은 트래픽에서 직렬화 CPU 부하 증가

  response-cache-update-interval-ms 줄이기
    → 캐시 동기화 더 자주
    → 지연 감소, CPU 소모 증가

  실험: 레지스트리 변경 후 바로 fetch
  → 30초 cache sync + 30초 client fetch 실제로 확인
```

---

## 📌 핵심 정리

```
Eureka Server 자료구조 핵심:

registry:
  ConcurrentHashMap<appName, Map<instanceId, Lease<InstanceInfo>>>
  이중 Map: 서비스명 → 인스턴스별 만료 정보

Lease<T>:
  isExpired(): lastUpdateTimestamp + duration < now
  renew(): lastUpdateTimestamp = now + duration (시계 오차 허용)

ResponseCache (2단계):
  readWriteCacheMap(Guava, 30s TTL) ← 레지스트리 변경 시 무효화
  readOnlyCacheMap(동기화 주기 30s) ← 클라이언트 조회 응답
  변경 반영 지연: 최대 60초 (cache sync + client fetch)

PeerAwareInstanceRegistry:
  register/renew/cancel 후 다른 Eureka 피어에 비동기 복제
  isReplication=true 플래그로 무한 복제 루프 방지

evict() 핵심:
  전체 삭제 안 함 → registrySize × (1-renewalPercentThreshold) 한도
  랜덤 순서 삭제 → 특정 서비스 집중 삭제 방지
```

---

## 🤔 생각해볼 문제

**Q1.** `Lease.renew()`가 `lastUpdateTimestamp = System.currentTimeMillis() + duration`으로 구현된 이유는 무엇인가? 단순히 `System.currentTimeMillis()`로 설정하면 안 되는가?

<details>
<summary>해설 보기</summary>

`duration`을 더하는 것은 시계 오차(clock skew)를 허용하기 위한 의도적 설계입니다. 클라이언트가 Heartbeat를 보낸 시각과 서버가 수신한 시각 사이에 네트워크 지연이 있을 수 있습니다. 또한 분산 환경에서 서버와 클라이언트의 시스템 시계가 완전히 동일하지 않을 수 있습니다. `lastUpdateTimestamp + duration`으로 설정하면 `isExpired()`에서 `lastUpdateTimestamp + duration + duration < now`가 되어, 실제로는 `2 × duration` 시간이 지나야 만료됩니다. 이는 약간의 관대함을 두어 정상 인스턴스가 일시적 지연으로 잘못 만료되는 것을 방지합니다.

</details>

---

**Q2.** `evict()`에서 만료된 인스턴스를 한 번에 모두 삭제하지 않고 `evictionLimit`만큼만 삭제하는 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

Self-Preservation Mode의 보완적 동작입니다. 네트워크 파티션이 발생하면 많은 인스턴스가 동시에 Heartbeat를 보내지 못해 한꺼번에 만료될 수 있습니다. 이때 모두 삭제하면 실제로는 살아있는 인스턴스들이 레지스트리에서 사라져 서비스 전체 장애로 이어집니다. `evictionLimit = registrySize × (1 - renewalPercentThreshold)`만큼만 삭제함으로써, 전체 인스턴스의 일정 비율은 반드시 유지합니다. 이것이 Self-Preservation의 핵심 아이디어이며, Ch2-06에서 상세히 다룹니다.

</details>

---

**Q3.** Eureka Server의 레지스트리는 인메모리에만 존재한다. 서버가 재시작되면 레지스트리가 비어있는 상태에서 어떻게 빠르게 복구되는가?

<details>
<summary>해설 보기</summary>

두 가지 복구 메커니즘이 있습니다.

1. **Peer Eureka 동기화**: 다중 Eureka 구성에서 재시작된 서버는 다른 피어로부터 전체 레지스트리를 fetch합니다. `PeerEurekaNodes`의 초기화 시 피어에게 `GET /eureka/apps`를 호출하여 전체 레지스트리를 받아옵니다.

2. **클라이언트 재등록**: 단일 Eureka 서버가 재시작된 경우, 살아있는 모든 클라이언트들이 `register-with-eureka: true`이므로 다음 Heartbeat 시점(최대 30초)에 서버에서 404를 받으면 자동으로 재등록합니다. `DiscoveryClient`의 `HeartbeatThread`가 `renew()` 실패(false 반환) 시 `register()`를 재호출하도록 구현되어 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Service Registry 패턴](./01-service-registry-pattern.md)** | **[홈으로 🏠](../README.md)** | **[다음: Eureka Client 등록 과정 (Heartbeat) ➡️](./03-eureka-client-heartbeat.md)**

</div>
