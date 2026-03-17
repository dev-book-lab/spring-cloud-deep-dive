<div align="center">

# ☁️ Spring Cloud Deep Dive

**"분산 시스템의 복잡성을 Spring Cloud가 어떻게 해결하는가"**

<br/>

> *"Config Server를 쓰는 것과, 설정이 어떻게 동적으로 리프레시되는지 아는 것은 다르다"*

Config Server의 `@RefreshScope` 동작 원리부터 Gateway 필터 체인 실행 순서, Eureka Heartbeat 메커니즘, Circuit Breaker의 CLOSED → OPEN → HALF_OPEN 상태 전이까지  
**왜 이렇게 설계됐는가** 라는 질문으로 Spring Cloud 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![Java](https://img.shields.io/badge/Java-17%2B-orange?style=flat-square&logo=openjdk)](https://www.java.com)
[![Spring Cloud](https://img.shields.io/badge/Spring_Cloud-2023.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://spring.io/projects/spring-cloud)
[![Docs](https://img.shields.io/badge/Docs-40개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

Spring Cloud에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "`@RefreshScope`를 붙이면 설정이 갱신됩니다" | `ContextRefresher.refresh()` → `RefreshScope.refreshAll()` → Proxy Bean 재생성 전 과정, Bus 이벤트 전파 메커니즘 |
| "Eureka에 서비스를 등록하면 됩니다" | `DiscoveryClient` 초기화 → Heartbeat 스케줄러 → `InstanceInfo` 상태 전이 → Self-Preservation Mode 발동 조건 |
| "Gateway로 라우팅하면 됩니다" | `RoutePredicateHandlerMapping`이 요청을 받아 `FilteringWebHandler`에서 Global Filter + Route Filter를 정렬·실행하는 Reactive 체인 |
| "Circuit Breaker로 장애를 격리합니다" | `CircuitBreakerStateMachine`의 SlidingWindow 기반 실패율 계산, OPEN 상태 진입·탈출 조건, `HalfOpenState`에서 허용 호출 수 관리 |
| "LoadBalancer로 부하를 분산합니다" | `ReactorLoadBalancer`의 `choose()` 호출 흐름, `ServiceInstanceListSupplier` 체인, Round Robin vs Weighted Random 알고리즘 비교 |
| 이론 나열 | 실행 가능한 코드 + Spring Cloud 소스코드 직접 추적 + Docker Compose 전체 환경 + 장애 시뮬레이션 실험 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Config](https://img.shields.io/badge/🔹_Config_Server-Externalized_Configuration_원리-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./config-server/01-externalized-configuration.md)
[![Discovery](https://img.shields.io/badge/🔹_Service_Discovery-Service_Registry_패턴-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./service-discovery/01-service-registry-pattern.md)
[![Gateway](https://img.shields.io/badge/🔹_API_Gateway-Gateway_vs_Zuul_비교-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./api-gateway/01-gateway-vs-zuul.md)
[![LB](https://img.shields.io/badge/🔹_Load_Balancing-Ribbon_vs_Spring_Cloud_LB-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./load-balancing/01-ribbon-vs-spring-cloud-lb.md)
[![CB](https://img.shields.io/badge/🔹_Circuit_Breaker-Circuit_Breaker_패턴과_필요성-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./circuit-breaker/01-circuit-breaker-pattern.md)
[![Tracing](https://img.shields.io/badge/🔹_Distributed_Tracing-Sleuth_Trace_ID_원리-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./distributed-tracing/01-sleuth-trace-span.md)
[![Advanced](https://img.shields.io/badge/🔹_Advanced_Patterns-Event_Driven_Architecture-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./advanced-cloud-patterns/01-event-driven-architecture.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: Config Server

> **핵심 질문:** 설정 파일이 Git에 있는데, 서비스는 어떻게 그 값을 읽어오고, 변경됐을 때 어떻게 반영하는가?

<details>
<summary><b>Externalized Configuration부터 동적 갱신 원리까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Externalized Configuration 필요성](./config-server/01-externalized-configuration.md) | 12-Factor App의 Config 원칙, 환경마다 다른 설정값을 코드와 분리해야 하는 이유, 프로퍼티 소스 우선순위 체계 |
| [02. Config Server 동작 원리 (Git Backend)](./config-server/02-config-server-git-backend.md) | `EnvironmentController`가 Git clone/fetch → `NativeEnvironmentRepository`에서 yml 파싱 → `Environment` 객체 반환하는 흐름, `/config/{app}/{profile}` 엔드포인트 내부 |
| [03. Config Client 자동 설정](./config-server/03-config-client-auto-configuration.md) | `ConfigClientAutoConfiguration`이 부트스트랩 컨텍스트에서 `ConfigServicePropertySourceLocator`를 등록하는 과정, `bootstrap.yml`이 `application.yml`보다 먼저 로딩되는 이유 |
| [04. @RefreshScope와 동적 갱신](./config-server/04-refresh-scope-dynamic-reload.md) | `ContextRefresher.refresh()` 호출 흐름, `RefreshScope`가 Proxy Bean을 무효화·재생성하는 메커니즘, `/actuator/refresh` vs Spring Cloud Bus 이벤트 전파 비교 |
| [05. Encryption / Decryption](./config-server/05-encryption-decryption.md) | `{cipher}` 접두사가 붙은 값을 Config Server가 복호화하는 시점, 대칭키(AES) vs 비대칭키(RSA) 설정 방법, JCE Unlimited Strength 이슈 |
| [06. Config Server High Availability](./config-server/06-config-server-high-availability.md) | Config Server 다중화 전략(Active-Active), Eureka에 Config Server 자체 등록, Git 미러링과 Git LFS, 클라이언트 Retry 설정과 폴백 전략 |

</details>

<br/>

### 🔹 Chapter 2: Service Discovery with Eureka

> **핵심 질문:** 서비스 인스턴스가 늘고 줄어드는 MSA 환경에서, 클라이언트는 어떻게 서버의 주소를 알아내는가?

<details>
<summary><b>Service Registry 패턴부터 Self-Preservation Mode까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Service Registry 패턴](./service-discovery/01-service-registry-pattern.md) | DNS 기반 서비스 디스커버리의 한계, Client-Side vs Server-Side 디스커버리 비교, Eureka가 선택된 이유와 CAP 트레이드오프 |
| [02. Eureka Server 내부 구조](./service-discovery/02-eureka-server-internals.md) | `PeerAwareInstanceRegistry`의 인스턴스 등록·조회·만료 로직, `AbstractInstanceRegistry`의 ConcurrentHashMap 기반 레지스트리 구조, 피어 간 복제 메커니즘 |
| [03. Eureka Client 등록 과정 (Heartbeat)](./service-discovery/03-eureka-client-heartbeat.md) | `DiscoveryClient` 초기화 → `register()` 호출 → Heartbeat 스케줄러 동작, `InstanceInfo.InstanceStatus` 전이(STARTING → UP → DOWN), 30초 갱신 인터벌이 결정된 배경 |
| [04. Service Instance Metadata](./service-discovery/04-service-instance-metadata.md) | `EurekaInstanceConfigBean`으로 커스텀 메타데이터 등록하는 방법, `metadataMap`을 통한 카나리 배포·버전 라우팅 전략, `preferIpAddress` 설정이 필요한 이유 |
| [05. Client-Side Load Balancing (LoadBalancerClient)](./service-discovery/05-client-side-load-balancing.md) | `@LoadBalanced` `RestTemplate`이 `LoadBalancerInterceptor`를 통해 서비스 이름을 실제 URL로 치환하는 흐름, `ServiceInstanceChooser`가 Eureka 레지스트리에서 인스턴스를 선택하는 과정 |
| [06. Self-Preservation Mode](./service-discovery/06-self-preservation-mode.md) | 네트워크 파티션 상황에서 Eureka가 인스턴스를 삭제하지 않는 이유, 임계값 계산 공식(`renewalPercentThreshold`), 개발 환경에서 비활성화해야 하는 이유 |

</details>

<br/>

### 🔹 Chapter 3: API Gateway (Spring Cloud Gateway)

> **핵심 질문:** 클라이언트 요청이 Gateway에 도달했을 때, Predicate 평가 → Filter 체인 실행 → 업스트림 라우팅이 어떤 순서로 일어나는가?

<details>
<summary><b>Reactive 라우팅 아키텍처부터 Custom Filter 작성까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Gateway vs Zuul (Reactive vs Blocking)](./api-gateway/01-gateway-vs-zuul.md) | Zuul 1.x Thread-per-Request 모델의 한계, Spring Cloud Gateway의 Netty + Project Reactor 기반 Non-Blocking 아키텍처, `WebFlux`와의 관계 |
| [02. Route Predicate Factory](./api-gateway/02-route-predicate-factory.md) | `RoutePredicateFactory` 인터페이스 구조, Path/Method/Header/Query/Weight Predicate 동작 방식, `AsyncPredicate`가 Reactive 체인에 통합되는 원리 |
| [03. Gateway Filter Factory](./api-gateway/03-gateway-filter-factory.md) | `GatewayFilterFactory` 계층 구조, `AddRequestHeader` · `RewritePath` · `Retry` 필터 내부 구현, `apply()` 메서드가 `GatewayFilter` 람다를 반환하는 패턴 |
| [04. Global Filter 체인 실행 순서](./api-gateway/04-global-filter-chain.md) | `FilteringWebHandler`가 Global Filter와 Route Filter를 `@Order` 기반으로 정렬·병합하는 과정, `NettyRoutingFilter`와 `NettyWriteResponseFilter`의 실행 위치가 중요한 이유 |
| [05. Route Locator 동적 라우팅](./api-gateway/05-route-locator-dynamic-routing.md) | `RouteLocator` Bean 정의 방식 vs YAML 라우트 비교, `RefreshRoutesEvent`로 라우트를 런타임에 갱신하는 흐름, Config Server 연동으로 라우트 동적 변경 |
| [06. Gateway Timeout & Circuit Breaker 통합](./api-gateway/06-gateway-timeout-circuit-breaker.md) | `connect-timeout` · `response-timeout` 설정이 Netty 레벨에서 적용되는 시점, `SpringCloudCircuitBreakerFilterFactory`가 Resilience4j를 호출하는 방식, Fallback URI 처리 |
| [07. Custom Filter 작성](./api-gateway/07-custom-filter.md) | `GlobalFilter` vs `GatewayFilterFactory` 선택 기준, JWT 검증 Global Filter 구현 전 과정, `exchange.mutate()`로 요청·응답을 불변 체인에서 수정하는 패턴 |

</details>

<br/>

### 🔹 Chapter 4: Load Balancing

> **핵심 질문:** `@LoadBalanced`가 붙은 RestTemplate은 서비스 이름을 어떻게 실제 IP/Port로 바꾸고, 어떤 알고리즘으로 인스턴스를 선택하는가?

<details>
<summary><b>Ribbon 대체부터 Custom LoadBalancer 구현까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Ribbon vs Spring Cloud LoadBalancer](./load-balancing/01-ribbon-vs-spring-cloud-lb.md) | Netflix Ribbon이 Maintenance 상태로 전환된 배경, `BlockingLoadBalancerClient` vs `ReactorLoadBalancer` 비교, `spring-cloud-starter-loadbalancer` 의존성 전환 가이드 |
| [02. LoadBalancerClient 동작 원리](./load-balancing/02-load-balancer-client-internals.md) | `LoadBalancerInterceptor` → `BlockingLoadBalancerClient.choose()` → `ServiceInstanceListSupplier` → Eureka 레지스트리 조회 전체 흐름, 캐싱 레이어 동작 |
| [03. Load Balancing 알고리즘](./load-balancing/03-load-balancing-algorithms.md) | `RoundRobinLoadBalancer`의 AtomicInteger 기반 구현, `RandomLoadBalancer` 비교, Weighted Response Time 전략, Spring Cloud LoadBalancer에서 커스텀 알고리즘 플러그인 방법 |
| [04. Retry 전략](./load-balancing/04-retry-strategy.md) | `RetryLoadBalancerInterceptor`가 동일 인스턴스 vs 다른 인스턴스 재시도를 구분하는 로직, `spring.cloud.loadbalancer.retry` 설정과 Circuit Breaker와의 경계 설정 |
| [05. Custom LoadBalancer 구현](./load-balancing/05-custom-load-balancer.md) | `ReactorServiceInstanceLoadBalancer` 인터페이스 구현, 메타데이터 기반 카나리 라우팅(버전 태그 필터링), `LoadBalancerClientConfiguration`으로 서비스별 로드밸런서 격리 |

</details>

<br/>

### 🔹 Chapter 5: Circuit Breaker (Resilience4j)

> **핵심 질문:** Circuit Breaker는 어떻게 실패를 감지하고, OPEN 상태를 언제 진입·탈출하며, Fallback은 어떤 시점에 실행되는가?

<details>
<summary><b>상태 전이 원리부터 Bulkhead · Rate Limiter 통합까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Circuit Breaker 패턴과 필요성](./circuit-breaker/01-circuit-breaker-pattern.md) | 분산 환경에서 연쇄 장애(Cascading Failure)가 발생하는 원리, Circuit Breaker가 없을 때 Thread Pool 고갈 시나리오, Resilience4j vs Hystrix 설계 철학 비교 |
| [02. CLOSED → OPEN → HALF_OPEN 상태 전이](./circuit-breaker/02-state-machine-transitions.md) | `CircuitBreakerStateMachine` 내부 구현, `SlidingWindowType` (COUNT_BASED vs TIME_BASED) 비교, 상태 전이 트리거 조건과 `StateTransition` 이벤트 발행 |
| [03. Failure Rate Threshold 계산](./circuit-breaker/03-failure-rate-threshold.md) | `FixedSizeSlidingWindow`와 `SlidingTimeWindow`의 버킷 구조, `failureRateThreshold` 계산 공식, `minimumNumberOfCalls` 설정이 없을 때 발생하는 오탐 문제 |
| [04. Slow Call 탐지](./circuit-breaker/04-slow-call-detection.md) | `slowCallDurationThreshold` · `slowCallRateThreshold` 설정이 적용되는 시점, Slow Call도 OPEN 상태를 유발할 수 있는 이유, 실제 네트워크 지연 시뮬레이션 실험 |
| [05. Fallback 메서드 작성](./circuit-breaker/05-fallback-methods.md) | `@CircuitBreaker(fallbackMethod=...)` AOP 프록시가 예외를 가로채는 시점, Fallback 메서드 시그니처 규칙, Fallback 체이닝(Fallback의 Fallback), `CallNotPermittedException` vs 실제 예외 구분 |
| [06. Bulkhead 패턴 (격리)](./circuit-breaker/06-bulkhead-pattern.md) | `Semaphore Bulkhead`(동시 호출 수 제한) vs `ThreadPool Bulkhead`(격리 스레드 풀) 비교, `BulkheadFullException` 발생 조건, Circuit Breaker와 함께 사용할 때 실행 순서 |
| [07. Rate Limiter 통합](./circuit-breaker/07-rate-limiter.md) | `AtomicRateLimiter`의 토큰 버킷 갱신 주기, `SemaphoreBasedRateLimiter`와의 비교, `RequestNotPermittedException` 예외 처리, Circuit Breaker → Bulkhead → Rate Limiter 조합 데코레이터 순서 |

</details>

<br/>

### 🔹 Chapter 6: Distributed Tracing

> **핵심 질문:** 마이크로서비스 A → B → C 호출 체인에서, 하나의 요청을 전 구간에 걸쳐 어떻게 추적하는가?

<details>
<summary><b>Trace ID 전파부터 Zipkin 시각화까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Sleuth와 Trace ID / Span ID](./distributed-tracing/01-sleuth-trace-span.md) | `TraceContext`의 traceId · spanId · parentSpanId 구조, `Propagator`가 HTTP 헤더(`X-B3-TraceId` 등)에 컨텍스트를 주입·추출하는 방법, Micrometer Tracing으로의 전환 배경 |
| [02. Zipkin Server 연동](./distributed-tracing/02-zipkin-integration.md) | `ZipkinSpanHandler`가 Span을 완료 시점에 수집해 Zipkin HTTP API로 전송하는 흐름, `management.tracing.sampling.probability` 샘플링 전략, Docker Compose로 Zipkin UI 구성 |
| [03. MDC를 통한 로그 추적](./distributed-tracing/03-mdc-log-tracing.md) | `MDC`에 traceId · spanId가 자동으로 주입되는 시점, Logback 패턴에 `%X{traceId}` 추가 설정, 비동기 처리(`@Async`, Reactor)에서 MDC 컨텍스트 전파 문제와 해결 |
| [04. Baggage Propagation](./distributed-tracing/04-baggage-propagation.md) | `Baggage`가 Trace Context와 함께 서비스 간에 전파되는 원리, 사용자 ID · 테넌트 ID 전파 실전 패턴, `BaggageField`의 remote vs local 구분 |
| [05. Custom Span Tags](./distributed-tracing/05-custom-span-tags.md) | `Tracer.currentSpan().tag()` API로 비즈니스 데이터를 Span에 추가하는 방법, `@NewSpan` · `@ContinueSpan` · `@SpanTag` 어노테이션 동작 원리, Zipkin에서 태그로 검색하는 방법 |

</details>

<br/>

### 🔹 Chapter 7: Advanced Cloud Patterns

> **핵심 질문:** 분산 트랜잭션, 이벤트 기반 통신, 데이터 조회 최적화를 MSA에서 어떻게 설계하는가?

<details>
<summary><b>Event-Driven부터 CQRS까지 분산 시스템 고급 패턴 (4개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Event-Driven Architecture (Spring Cloud Stream)](./advanced-cloud-patterns/01-event-driven-architecture.md) | `@EnableBinding` 대신 함수형 프로그래밍 모델(`Consumer<T>`, `Supplier<T>`, `Function<T,R>`)로 Kafka · RabbitMQ 바인딩하는 방법, `MessageChannel`과 `Binder` 추상화 계층 |
| [02. Saga Pattern (분산 트랜잭션)](./advanced-cloud-patterns/02-saga-pattern.md) | 2PC(Two-Phase Commit)의 가용성 문제, Choreography Saga vs Orchestration Saga 비교, 보상 트랜잭션(Compensating Transaction) 설계 원칙과 멱등성 보장 전략 |
| [03. API Composition](./advanced-cloud-patterns/03-api-composition.md) | 여러 서비스를 조합해 단일 응답을 반환하는 API Composition 패턴, Gateway에서 병렬 호출 후 집계하는 `WebClient` + `Mono.zip()` 구현, 부분 실패(Partial Failure) 처리 전략 |
| [04. CQRS (Command Query Responsibility Segregation)](./advanced-cloud-patterns/04-cqrs.md) | 단일 모델의 읽기/쓰기 경합 문제, Command 모델과 Query 모델 분리 전략, Event Sourcing과의 조합, Spring Data의 Projection으로 Query 모델을 경량화하는 실전 예시 |

</details>

---

## 🗺️ 목적별 학습 경로

<details>
<summary><b>🟢 "MSA 면접 질문에 막힌다" — 핵심 개념 파악 (1주)</b></summary>

<br/>

```
Day 1  Ch1-04  @RefreshScope와 동적 갱신 ← Config의 핵심
Day 2  Ch2-03  Eureka Client 등록 과정 (Heartbeat) ← Discovery의 핵심
Day 3  Ch3-04  Global Filter 체인 실행 순서 ← Gateway의 핵심
Day 4  Ch5-02  CLOSED → OPEN → HALF_OPEN 상태 전이 ← Circuit Breaker의 핵심
Day 5  Ch5-03  Failure Rate Threshold 계산
Day 6  Ch6-01  Sleuth와 Trace ID / Span ID ← Tracing의 핵심
Day 7  Ch2-06  Self-Preservation Mode
```

</details>

<details>
<summary><b>🔵 "실제 MSA 환경을 구성하고 장애를 시뮬레이션해야 한다" — 실전 실험 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch1-02  Config Server 동작 원리 → Docker로 Config Server 띄우기
Day 2  Ch2-02  Eureka Server 내부 구조 → Eureka UI에서 인스턴스 등록 확인
Day 3  Ch3-07  Custom Filter 작성 → JWT 검증 필터 직접 구현
Day 4  Ch5-01  Circuit Breaker 패턴 → Service 다운 후 OPEN 상태 전환 실험
Day 5  Ch5-05  Fallback 메서드 작성 → Fallback 응답 흐름 확인
Day 6  Ch6-02  Zipkin Server 연동 → Zipkin UI로 전체 요청 추적
Day 7  Ch4-05  Custom LoadBalancer 구현 → 카나리 배포 라우팅 실험
```

</details>

<details>
<summary><b>🔴 "Spring Cloud 소스코드를 직접 읽고 내부를 완전히 이해하고 싶다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — Config Server 완전 분해
        → ContextRefresher.refresh() 에 브레이크포인트 걸고 Bean 재생성 확인

2주차  Chapter 2 전체 — Eureka 내부 구조
        → AbstractInstanceRegistry.register() 소스에서 ConcurrentHashMap 직접 읽기

3주차  Chapter 3 전체 — Spring Cloud Gateway
        → FilteringWebHandler.handle() 에서 필터 정렬 순서 디버거로 확인

4주차  Chapter 4 전체 — Load Balancing
        → RoundRobinLoadBalancer.choose() 의 AtomicInteger 동작 확인

5주차  Chapter 5 전체 — Circuit Breaker
        → CircuitBreakerStateMachine 상태 전이를 Actuator /circuitbreakers 로 실시간 모니터링

6주차  Chapter 6 전체 — Distributed Tracing
        → Zipkin UI에서 Trace를 시각화하며 서비스 간 Span 연결 확인

7주차  Chapter 7 전체 — Advanced Patterns
        → Saga Pattern 보상 트랜잭션을 직접 구현하며 원자성 보장 전략 실험
```

</details>

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 MSA에서 필요한가** | 분산 시스템 문제 상황과 설계 배경 |
| 😱 **잘못된 구성** | Before — 단일 장애점 또는 흔한 실수 |
| ✨ **올바른 패턴** | After — 원리를 알고 난 후의 올바른 접근 |
| 🔬 **내부 동작 원리** | Spring Cloud 소스코드 직접 추적 + ASCII 구조도 |
| 💻 **실전 구성** | Docker Compose로 전체 MSA 환경 구성 |
| 🌐 **분산 시스템 시나리오** | 장애 상황 시뮬레이션 (서비스 다운, 네트워크 지연) |
| ⚖️ **트레이드오프** | 이 패턴의 장단점, 언제 다른 방법을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🔬 핵심 분석 대상 — MSA 전체 호출 흐름

이 레포의 모든 챕터는 아래 호출 흐름을 완전히 이해하는 것을 목표로 합니다.

```
Client
  │
  ▼
┌─────────────────────────────────┐
│  API Gateway  (Ch3)             │
│  RoutePredicateHandlerMapping   │
│  → GlobalFilter 체인             │
│  → NettyRoutingFilter           │
└─────────────┬───────────────────┘
              │ Service 이름으로 라우팅
              ▼
┌─────────────────────────────────┐
│  Eureka Server  (Ch2)           │  ←── Config Server (Ch1)
│  PeerAwareInstanceRegistry      │      설정 동적 리프레시
└─────────────┬───────────────────┘
              │ 인스턴스 목록 반환
              ▼
┌─────────────────────────────────┐
│  LoadBalancer  (Ch4)            │
│  RoundRobinLoadBalancer.choose()│
└─────────────┬───────────────────┘
              │ 실제 IP:Port 선택
              ▼
┌─────────────────────────────────┐
│  Service A  (Port 8081)         │
│  Circuit Breaker (Ch5)          │
│  → CLOSED / OPEN / HALF_OPEN    │
└─────────────┬───────────────────┘
              │ 내부 호출
              ▼
┌─────────────────────────────────┐
│  Service B  (Port 8082)         │
└─────────────────────────────────┘

전 구간에 걸쳐:
  Trace ID / Span ID 전파 (Ch6)
  → Zipkin에서 전체 흐름 시각화

장애 시나리오:
  1. Service B 다운 → Circuit Breaker OPEN → Fallback 응답
  2. Config 값 변경 → /actuator/refresh → @RefreshScope Bean 재생성
  3. Service A 인스턴스 3개 → Round Robin Load Balancing
  4. Trace ID로 A → B 전체 구간 추적 (Zipkin UI)
```

---

## 🐳 실험 환경 (Docker Compose)

모든 챕터의 실험은 아래 환경에서 재현 가능합니다.

```yaml
# docker-compose.yml (전체 MSA 환경)
services:
  config-server:    # Ch1 — Git Backend 설정 서버
  eureka-server:    # Ch2 — Service Registry
  api-gateway:      # Ch3 — Reactive Gateway (Port 8080)
  service-a:        # Ch4, Ch5 — LoadBalancer, Circuit Breaker 실험 대상
  service-b:        # Ch5 — 장애 시뮬레이션 대상
  zipkin:           # Ch6 — 분산 추적 UI (Port 9411)
```

각 문서에서 해당 시나리오에 필요한 서비스만 선택적으로 실행하는 방법을 안내합니다.

---

## 🔗 선행 학습 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [spring-core-deep-dive](https://github.com/dev-book-lab/spring-core-deep-dive) | IoC 컨테이너, AOP, Bean 생명주기, Proxy | Ch1(@RefreshScope Proxy 재생성), Ch5(Circuit Breaker AOP 프록시) |
| [spring-boot-internals](https://github.com/dev-book-lab/spring-boot-internals) | Auto-configuration, Actuator | Ch1(Config Client 자동 설정), Ch5(Actuator circuitbreakers 엔드포인트) |
| [spring-mvc-deep-dive](https://github.com/dev-book-lab/spring-mvc-deep-dive) | DispatcherServlet, Filter, Interceptor | Ch3(Gateway Filter vs MVC Interceptor 비교) |

> 💡 Spring Cloud는 독립적인 영역이므로 선행 레포 없이도 학습 가능합니다. 단, `AOP / Proxy` 개념이 Ch5(Circuit Breaker 어노테이션 동작)에서 핵심적으로 활용됩니다.

---

## 🙏 Reference

- [Spring Cloud Reference Documentation](https://docs.spring.io/spring-cloud/docs/current/reference/html/)
- [Spring Cloud Gateway Reference](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/)
- [Resilience4j User Guide](https://resilience4j.readme.io/docs)
- [Netflix Eureka Wiki](https://github.com/Netflix/eureka/wiki)
- [Micrometer Tracing Docs](https://micrometer.io/docs/tracing)
- [Spring Cloud Config Reference](https://docs.spring.io/spring-cloud-config/docs/current/reference/html/)
- [12-Factor App — Config](https://12factor.net/config)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"Config Server를 쓰는 것과, 설정이 어떻게 동적으로 리프레시되는지 아는 것은 다르다"*

</div>
