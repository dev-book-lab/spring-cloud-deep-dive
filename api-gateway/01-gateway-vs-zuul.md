# Gateway vs Zuul — Reactive vs Blocking 아키텍처의 근본 차이

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Zuul 1.x의 Thread-per-Request 모델이 높은 동시성 환경에서 병목이 되는 이유는 무엇인가?
- Spring Cloud Gateway가 Netty + Project Reactor를 선택한 이유는 무엇인가?
- `WebFlux`와 Spring Cloud Gateway의 관계는 무엇인가? Gateway가 WebFlux 위에서 동작하는 이유는?
- 동일 하드웨어에서 Zuul 1.x 대비 Spring Cloud Gateway의 성능 차이가 나는 구체적 이유는?
- Zuul 2.x도 Netty 기반인데 Spring Cloud가 Zuul 2.x 대신 자체 Gateway를 만든 이유는?
- Gateway에서 `Mono<Void>`를 반환하는 Filter 메서드 시그니처의 의미는 무엇인가?

---

## 🔍 왜 MSA에서 필요한가

### API Gateway가 MSA의 관문이 되는 이유

```
MSA 없이 클라이언트가 직접 각 서비스를 호출한다면:

  클라이언트 앱:
    주문 조회  → http://order-service:8080/orders/1
    재고 조회  → http://inventory-service:8081/inventory/1
    결제 조회  → http://payment-service:8082/payments/1
    
  문제:
    ① 서비스 주소가 클라이언트에 노출 (보안)
    ② 서비스 수만큼 CORS, 인증, 로깅 중복 구현
    ③ 클라이언트가 서비스 내부 구조에 종속
    ④ 모바일 앱에서 10개 API를 병렬 호출 → 배터리, 데이터 낭비

API Gateway 도입:
  
  모든 요청 → Gateway(단일 진입점)
    → 인증/인가 (한 곳에서)
    → 라우팅 (서비스명으로 분기)
    → 로드밸런싱
    → 로깅/추적

  Gateway는 초당 수천 ~ 수만 요청을 처리해야 함
  → 성능이 MSA 전체 처리량을 결정
  → Blocking vs Non-Blocking 선택이 핵심
```

---

## 😱 잘못된 구성

### Before: Zuul 1.x의 Thread-per-Request 한계

```
Zuul 1.x 아키텍처 (Netflix, Servlet 기반):

요청 1 → Tomcat 스레드 1 → 업스트림 서비스 호출 (200ms 대기) → 응답
요청 2 → Tomcat 스레드 2 → 업스트림 서비스 호출 (150ms 대기) → 응답
요청 3 → Tomcat 스레드 3 → 업스트림 서비스 호출 (300ms 대기) → 응답
...
요청 N → 스레드 없음! → 큐 대기 또는 거절

스레드 풀 (기본 200개):
  각 스레드: ~1MB 스택 메모리
  200개 스레드 → 200MB 메모리 상시 점유
  
  동시 요청 200개 초과:
  → 모든 스레드가 업스트림 응답 대기 중
  → 새 요청은 큐에서 대기
  → 큐도 꽉 차면 → 503 Service Unavailable

  업스트림 서비스가 느릴수록 더 심각:
  → 각 스레드가 더 오래 점유 → 더 빨리 고갈
  → "업스트림 서비스 지연 → Gateway 스레드 고갈 → 전체 시스템 응답 불가"
```

```java
// Zuul 1.x 필터 (Servlet 기반, 블로킹)
public class AuthFilter extends ZuulFilter {
    @Override
    public String filterType() { return "pre"; }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        // ❌ 이 스레드가 블로킹되는 지점들:
        String token = request.getHeader("Authorization");
        User user = userService.findByToken(token);  // DB 조회 - 블로킹!
        // 이 스레드는 DB 응답 올 때까지 아무것도 못 함
        // Tomcat 스레드 1개 소진 중

        if (user == null) {
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
        }
        return null;
    }
}
```

---

## ✨ 올바른 패턴

### After: Spring Cloud Gateway — Non-Blocking Reactive 아키텍처

```
Spring Cloud Gateway 아키텍처 (Netty + Project Reactor):

이벤트 루프 (CPU 코어 수 × 2개 스레드):
  
  요청 1 도착 → 이벤트 루프 스레드 → 처리 시작
    업스트림 호출 시작 → 스레드 반환 (대기 안 함!)
    ← 200ms 후 업스트림 응답 도착 → 이벤트로 처리 재개 → 응답

  요청 2 도착 → 같은 이벤트 루프 스레드 → 처리 시작
    (요청 1이 대기 중이지만 스레드는 자유)
    업스트림 호출 시작 → 스레드 반환

  결과:
  스레드 8개(4코어 × 2)로 수천 개 동시 요청 처리 가능
  메모리 사용량 극적으로 감소
  업스트림 서비스 지연 → Gateway 성능에 영향 없음
```

---

## 🔬 내부 동작 원리

### 1. Netty 이벤트 루프 기반 처리

```java
// Spring Cloud Gateway는 Netty Server 위에서 동작
// DispatcherHandler(WebFlux) → RoutePredicateHandlerMapping → FilteringWebHandler

// Gateway 요청 처리 흐름 (Reactive 체인)
// 모든 반환 타입이 Mono<T> 또는 Flux<T>
// → 실제 실행은 구독(subscribe) 시점에
// → 스레드를 블로킹하지 않고 콜백으로 연결

// NettyRoutingFilter가 업스트림 호출하는 방식
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    // Reactive HTTP 클라이언트 (Non-Blocking)
    return this.httpClient
        .request(method)
        .uri(url)
        .send(requestBodyFlux)
        .responseConnection((res, connection) -> {
            // 응답이 도착했을 때 실행될 콜백
            // 이 시점까지 스레드를 점유하지 않음
            exchange.getAttributes().put(CLIENT_RESPONSE_ATTR, res);
            exchange.getAttributes().put(CLIENT_RESPONSE_CONN_ATTR, connection);
            return chain.filter(exchange);
        });
    // 여기서 바로 반환 (응답을 기다리지 않음)
    // Netty 이벤트 루프가 응답을 받으면 위 콜백 실행
}
```

### 2. WebFlux와의 관계

```java
// Spring Cloud Gateway는 Spring WebFlux 위에 구축됨
// WebFlux: Reactive Web Framework (Spring MVC의 Non-Blocking 버전)
// Gateway: WebFlux를 사용해 라우팅·필터링 레이어를 추가한 것

// WebFlux 핵심 인터페이스
public interface WebHandler {
    Mono<Void> handle(ServerWebExchange exchange);
}

// Gateway의 WebHandler 구현
public class FilteringWebHandler implements WebHandler {
    @Override
    public Mono<Void> handle(ServerWebExchange exchange) {
        // ① Route에서 필터 목록 가져오기
        Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
        List<GatewayFilter> gatewayFilters = route.getFilters();

        // ② Global Filter + Route Filter 병합 및 정렬
        List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
        combined.addAll(gatewayFilters);
        AnnotationAwareOrderComparator.sort(combined);

        // ③ Filter 체인 생성 및 실행
        return new DefaultGatewayFilterChain(combined).filter(exchange);
        // Mono<Void>: 비동기 완료 신호
    }
}

// ServerWebExchange: WebFlux의 요청/응답 컨테이너
// (Spring MVC의 HttpServletRequest + HttpServletResponse 역할)
// 불변(Immutable) 설계 → mutate()로만 수정 가능
public interface ServerWebExchange {
    ServerHttpRequest getRequest();
    ServerHttpResponse getResponse();
    // 수정 시: exchange.mutate().request(newRequest).build()
    Builder mutate();
}
```

### 3. Zuul 2.x vs Spring Cloud Gateway

```
Netflix Zuul 2.x (2018):
  Netty 기반으로 Zuul 1.x 문제 해결
  비동기 Non-Blocking 아키텍처

  왜 Spring Cloud가 채택 안 했는가:
    ① Spring Cloud Gateway 개발을 이미 시작한 상태
    ② Zuul 2.x의 Netflix 필터 API가 Spring 생태계와 이질적
    ③ Project Reactor / WebFlux 통합이 더 자연스럽게 가능
    ④ Spring 특유의 Auto-configuration, 컨벤션 적용 어려움
    ⑤ Netflix가 Zuul 2.x 개발을 간헐적으로 진행 (불확실한 로드맵)

  결정: Spring 자체적으로 Gateway 개발
        WebFlux + Project Reactor 완전 통합
        Spring Boot Auto-configuration 지원
        Spring Security 통합 용이

Spring Cloud Gateway vs Zuul 1.x 성능 (벤치마크):
  같은 하드웨어 기준:
  Zuul 1.x: ~10,000 RPS (Thread Pool 한계)
  Spring Cloud Gateway: ~35,000 RPS (이벤트 루프 기반)
  → 약 3배 이상 처리량 차이
  → 특히 업스트림 지연이 높을수록 차이 극대화
```

### 4. Reactive Filter 체인 실행 방식

```java
// GatewayFilter 인터페이스
@FunctionalInterface
public interface GatewayFilter extends ShortcutConfigurable, Ordered {
    Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}

// Filter 체인 구현
private static class DefaultGatewayFilterChain implements GatewayFilterChain {
    private final List<GatewayFilter> filters;
    private final int index;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange) {
        return Mono.defer(() -> {
            if (this.index < filters.size()) {
                GatewayFilter filter = filters.get(this.index);
                DefaultGatewayFilterChain chain = new DefaultGatewayFilterChain(
                    this.filters, this.index + 1);
                // 현재 필터 실행 → 다음 필터로 체인
                return filter.filter(exchange, chain);
            } else {
                return Mono.empty();  // 체인 끝
            }
        });
    }
}

// 실제 Filter 구현 패턴
public class SampleFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 요청 처리 (Pre-filter 로직)
        log.info("Pre-filter: {}", exchange.getRequest().getPath());

        return chain.filter(exchange)
            .then(Mono.fromRunnable(() -> {
                // 응답 처리 (Post-filter 로직)
                // chain.filter() 완료 후 실행
                log.info("Post-filter: status={}",
                    exchange.getResponse().getStatusCode());
            }));
    }

    @Override
    public int getOrder() { return -1; }
}
```

### 5. Backpressure — Reactive의 핵심 이점

```java
// Reactive Streams Backpressure:
// 클라이언트가 처리할 수 있는 속도 이상으로 데이터를 보내지 않음

// Zuul 1.x (블로킹):
// 업스트림이 100MB 파일을 전송 중
// → Gateway 스레드: 100MB 전부 메모리에 버퍼링 후 클라이언트에 전달
// → 메모리 폭발 가능성

// Spring Cloud Gateway (Reactive):
// 업스트림 → 청크 단위로 Flux로 수신
// → 클라이언트가 받을 준비됐을 때만 다음 청크 요청 (Backpressure)
// → 메모리 상수 사용 (전체 파일을 버퍼링 안 함)

Flux<DataBuffer> responseBody = ...;  // 업스트림 응답 스트림
exchange.getResponse().writeWith(responseBody);
// writeWith: 클라이언트 속도에 맞춰 자동 Backpressure 적용
```

---

## 💻 실전 구성

### Spring Cloud Gateway 기본 설정

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
    <!-- spring-boot-starter-webflux 자동 포함 -->
    <!-- spring-boot-starter-web과 함께 사용 불가! -->
</dependency>

<!-- spring-boot-starter-web (Servlet)과 공존 불가:
     Gateway = WebFlux(Non-Blocking)
     Web Starter = Tomcat(Blocking Servlet)
     둘 다 있으면 Gateway 비활성화됨
     spring.main.web-application-type=reactive 로 강제 설정 필요 -->
```

```yaml
# application.yml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      # Netty 로깅 (개발용)
      httpserver:
        wiretap: true    # 요청/응답 바이트 로깅
      httpclient:
        wiretap: true    # 업스트림 요청/응답 바이트 로깅

server:
  port: 8080

# Reactive 스레드 모델 확인
# 실행 시 로그:
# Netty started on port(s): 8080
# → Netty(Non-Blocking) 서버 확인
# Tomcat started on port(s): ...  ← 이게 나오면 설정 잘못된 것
```

### 성능 비교 실험

```bash
# 업스트림 서비스에 인위적 지연 추가
# Thread.sleep(500) → 500ms 응답 지연

# wrk로 부하 테스트
wrk -t12 -c400 -d30s http://localhost:8080/api/test

# Zuul 1.x (예상):
# Requests/sec: ~1,000 (Thread Pool 200개, 500ms 점유 → 200/0.5 = 400 RPS 이론치)
# Latency 99th: >1000ms (큐 대기)

# Spring Cloud Gateway (예상):
# Requests/sec: ~8,000+ (이벤트 루프, 500ms 대기 중 스레드 반환)
# Latency 99th: ~600ms (업스트림 지연만)
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 업스트림 서비스 응답 지연 시 Gateway 동작

```
상황: 업스트림 service-a 응답 시간 5초로 증가 (DB 쿼리 지연)

Zuul 1.x 에서:
  동시 요청 200개 도달
  → 200개 스레드 전부 service-a 응답 대기 중 (5초 × 200개)
  → 201번째 요청 → 큐 대기
  → 큐 꽉 참 → 503 반환
  → Gateway 전체 응답 불가
  → service-a만 느린데 Gateway도 다운 상태

Spring Cloud Gateway에서:
  동시 요청 1000개 도달
  → 이벤트 루프 8개 스레드가 모든 요청 처리 시작
  → 각 요청에 대해 service-a 호출 후 스레드 반환 (대기 안 함)
  → Timeout 설정 내에서 응답 대기
  → service-a 느려도 다른 서비스 라우팅은 정상
  → Timeout 초과 시 503 반환 (격리)

결론:
  Gateway 자체가 Blocking I/O로 인해 전체 장애로 이어지는 Zuul 1.x와 달리
  Spring Cloud Gateway는 업스트림 지연이 다른 라우트에 영향 없음
```

---

## ⚖️ 트레이드오프

| 관점 | Spring Cloud Gateway | Zuul 1.x |
|------|---------------------|----------|
| **동시성** | 이벤트 루프 (수천 동시 처리) | Thread Pool (수백 동시 처리) |
| **메모리** | 낮음 (스레드 수 적음) | 높음 (스레드당 ~1MB) |
| **지연 격리** | 업스트림 지연이 Gateway 영향 없음 | 업스트림 지연 → 스레드 고갈 |
| **프로그래밍 모델** | Reactive (학습 필요) | 명령형 (직관적) |
| **디버깅** | 복잡 (비동기 스택 트레이스) | 쉬움 (동기 스택 트레이스) |
| **블로킹 코드** | 절대 사용 금지 | 자유롭게 사용 가능 |

```
언제 Reactive 모델이 더 필요한가:
  ✅ I/O 집약적 (많은 업스트림 호출, 파일 전송)
  ✅ 높은 동시성 요구 (초당 수천 요청 이상)
  ✅ 업스트림 서비스 응답 시간 편차가 큼
  
  ❌ 블로킹 코드가 많은 레거시 필터 로직
  ❌ 팀이 Reactive 프로그래밍에 익숙하지 않음
  ❌ CPU 집약적 연산 (Reactive의 이점 없음)

Gateway에서 절대 하면 안 되는 것:
  이벤트 루프 스레드에서 블로킹 호출
  Thread.sleep(), JDBC 쿼리, 동기 HTTP 호출
  → 이벤트 루프 스레드가 블로킹되면 모든 요청 처리 중단
  → 해결: Schedulers.boundedElastic()으로 블로킹 작업 별도 스레드 풀에서 실행
```

---

## 📌 핵심 정리

```
Gateway vs Zuul 핵심:

Zuul 1.x:
  Servlet 기반 + Tomcat Thread Pool
  요청 1개 = 스레드 1개 점유 (응답 올 때까지)
  동시 처리 = Thread Pool 크기
  업스트림 지연 → 스레드 고갈 → Gateway 장애

Spring Cloud Gateway:
  Netty + Project Reactor (Non-Blocking)
  요청 1개 = 이벤트 루프 스레드 (업스트림 대기 중 스레드 반환)
  동시 처리 = 이론상 무제한 (I/O bound의 경우)
  업스트림 지연 → 해당 요청만 대기, 다른 요청 영향 없음

WebFlux와의 관계:
  Gateway = WebFlux 위에 구축된 특수 목적 애플리케이션
  모든 처리: Mono<Void> / Flux<T> Reactive 타입
  ServerWebExchange: 불변 객체 → mutate()로만 수정

핵심 제약:
  이벤트 루프 스레드에서 블로킹 코드 금지
  블로킹 필요 시: Schedulers.boundedElastic() 사용
```

---

## 🤔 생각해볼 문제

**Q1.** Spring Cloud Gateway 필터에서 실수로 `Thread.sleep(100)`을 호출하면 어떤 일이 벌어지는가?

<details>
<summary>해설 보기</summary>

Netty 이벤트 루프 스레드가 100ms 동안 블로킹됩니다. 이 시간 동안 해당 스레드가 처리해야 할 모든 요청이 멈춥니다. 이벤트 루프 스레드는 보통 CPU 코어 수 × 2개뿐이므로, 예를 들어 8개 스레드 중 하나가 100ms 블로킹되면 전체 처리량의 12.5%가 100ms 동안 멈춥니다. 모든 스레드가 동시에 블로킹 코드를 실행하면 Gateway가 완전히 응답 불능 상태가 됩니다. 해결 방법은 `Mono.delay(Duration.ofMillis(100))`(Non-Blocking)을 사용하거나, 블로킹이 불가피하다면 `Mono.fromCallable(() -> blockingOperation()).subscribeOn(Schedulers.boundedElastic())`으로 별도 스레드 풀에서 실행해야 합니다.

</details>

---

**Q2.** `spring-boot-starter-web`과 `spring-cloud-starter-gateway`를 둘 다 의존성에 추가했다. 어떤 일이 발생하는가?

<details>
<summary>해설 보기</summary>

`spring-boot-starter-web`이 Tomcat(Servlet 컨테이너)을 포함하고, `spring-cloud-starter-gateway`가 Netty(Reactive)를 요구하는 충돌이 발생합니다. Spring Boot는 Servlet 환경을 감지하면 `spring.main.web-application-type=servlet`으로 설정하여 WebFlux(Reactive) 대신 Spring MVC를 활성화합니다. 이 경우 Gateway의 `RoutePredicateHandlerMapping` Bean이 생성되지 않아 라우팅이 동작하지 않습니다. 해결 방법은 `spring-boot-starter-web` 의존성을 제거하거나 `spring.main.web-application-type=reactive`를 명시적으로 설정하는 것입니다.

</details>

---

**Q3.** Gateway의 이벤트 루프 스레드 수는 기본적으로 `CPU 코어 수 × 2`다. 4코어 서버에서 동시에 100개의 업스트림 HTTP 호출이 진행 중이라면 사용 중인 이벤트 루프 스레드는 몇 개인가?

<details>
<summary>해설 보기</summary>

8개(4 × 2)입니다. Non-Blocking I/O의 핵심은 I/O 대기 중에 스레드를 반환한다는 것입니다. 100개의 업스트림 호출이 진행 중이라는 것은 100개의 비동기 작업이 등록된 상태이지, 100개 스레드가 대기 중인 것이 아닙니다. Netty의 NIO Selector가 소켓 이벤트(응답 도착, 연결 완료 등)를 감시하다가 이벤트 발생 시 8개의 이벤트 루프 스레드가 처리합니다. 따라서 이론적으로 8개 스레드로 수천 개의 동시 연결을 처리할 수 있습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Route Predicate Factory ➡️](./02-route-predicate-factory.md)**

</div>
