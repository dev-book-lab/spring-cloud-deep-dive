# API Composition — 여러 서비스 응답을 하나로 합치는 패턴

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- API Composition이 필요한 이유와 MSA에서 발생하는 조인 문제를 어떻게 해결하는가?
- `WebClient`와 `Mono.zip()`으로 여러 서비스를 병렬 호출해 집계하는 코드 패턴은?
- 순차 호출과 병렬 호출의 성능 차이와 각각 선택해야 하는 기준은?
- 부분 실패(Partial Failure) — 하나의 서비스가 실패할 때 나머지 응답을 어떻게 처리하는가?
- `flatMap`과 `zipWith` 중 어느 것을 언제 사용해야 하는가?
- API Composition에서 N+1 문제와 그 해결 방법은?

---

## 🔍 왜 MSA에서 필요한가

### MSA에서 조인이 사라진 자리

```
단일 DB 시절:
  SELECT o.*, u.name, p.amount, i.stock
  FROM orders o
  JOIN users u ON o.user_id = u.id
  JOIN payments p ON o.id = p.order_id
  JOIN inventory i ON o.product_id = i.product_id
  WHERE o.id = 123

MSA에서:
  Order DB (Order Service): orders 테이블
  User DB (User Service): users 테이블
  Payment DB (Payment Service): payments 테이블
  Inventory DB (Inventory Service): inventory 테이블

  → DB 간 JOIN 불가능!
  → 클라이언트가 4개 API를 각각 호출?
     → 4번의 네트워크 왕복 (레이턴시 증가)
     → 클라이언트 로직 복잡 (여러 언어/플랫폼)
     → CORS, 인증 각각 처리 필요

API Composition 패턴:
  BFF(Backend for Frontend) 또는 API Gateway에서
  여러 서비스를 병렬 호출 → 하나의 응답으로 합침
  클라이언트: 단 1번의 API 호출로 완전한 응답 수신
```

---

## 😱 잘못된 구성

### Before: 순차 호출로 인한 레이턴시 누적

```java
// ❌ 순차 호출 - 각 서비스가 직렬로 대기
@GetMapping("/orders/{id}/detail")
public OrderDetailResponse getOrderDetail(@PathVariable Long id) {
    Order order = orderClient.getOrder(id);           // 50ms 대기
    User user = userClient.getUser(order.getUserId()); // 30ms 대기 (order 완료 후)
    Payment payment = paymentClient.getPayment(id);    // 40ms 대기 (user 완료 후)

    return new OrderDetailResponse(order, user, payment);
    // 총 레이턴시: 50 + 30 + 40 = 120ms
    // 병렬이면: max(50, 30, 40) = 50ms
}
```

### Before: 오류 시 전체 응답 실패

```java
// ❌ 하나의 서비스 실패 → 전체 응답 실패
public Mono<OrderDetailResponse> getOrderDetail(Long id) {
    return Mono.zip(
        orderClient.getOrder(id),
        userClient.getUser(id),
        paymentClient.getPayment(id)
    ).map(tuple -> new OrderDetailResponse(
        tuple.getT1(), tuple.getT2(), tuple.getT3()));
    // payment-service가 다운되면 → 전체 응답 실패
    // 주문 정보와 사용자 정보는 있는데도 불구하고
}
```

---

## ✨ 올바른 패턴

### After: 병렬 호출 + 부분 실패 허용

```java
// ✅ 병렬 호출 + 부분 실패 시 기본값으로 대체
@Service
public class OrderCompositionService {

    private final WebClient orderWebClient;
    private final WebClient userWebClient;
    private final WebClient paymentWebClient;

    public Mono<OrderDetailResponse> getOrderDetail(Long orderId) {
        // ① 각 서비스 호출 Mono 정의 (아직 실행 안 됨)
        Mono<OrderDto> orderMono = orderWebClient.get()
            .uri("/orders/{id}", orderId)
            .retrieve()
            .bodyToMono(OrderDto.class)
            .timeout(Duration.ofSeconds(2));

        Mono<UserDto> userMono = orderMono.flatMap(order ->
            userWebClient.get()
                .uri("/users/{id}", order.getUserId())
                .retrieve()
                .bodyToMono(UserDto.class)
                .timeout(Duration.ofSeconds(1))
                // 사용자 정보 실패 시 기본값
                .onErrorReturn(UserDto.unknown())
        );

        Mono<PaymentDto> paymentMono = paymentWebClient.get()
            .uri("/payments/order/{id}", orderId)
            .retrieve()
            .bodyToMono(PaymentDto.class)
            .timeout(Duration.ofSeconds(2))
            // 결제 정보 실패 시 빈 응답
            .onErrorReturn(PaymentDto.empty());

        // ② 병렬 실행 + 결과 집계
        return Mono.zip(orderMono, userMono, paymentMono)
            .map(tuple -> OrderDetailResponse.builder()
                .order(tuple.getT1())
                .user(tuple.getT2())     // 실패 시 UserDto.unknown()
                .payment(tuple.getT3())  // 실패 시 PaymentDto.empty()
                .build());
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Mono.zip() — 병렬 실행 메커니즘

```java
// Mono.zip()의 내부 동작:

// 구독(subscribe) 시점에:
// ① orderMono 구독 → HTTP 요청 1 시작 (Non-Blocking, 스레드 반환)
// ② userMono 구독  → HTTP 요청 2 시작 (Non-Blocking, 스레드 반환)
// ③ paymentMono 구독 → HTTP 요청 3 시작 (Non-Blocking, 스레드 반환)

// 세 요청이 병렬로 진행 중...

// 응답 수신 (이벤트 루프):
// 요청 2 완료 → 내부 버퍼에 저장
// 요청 3 완료 → 내부 버퍼에 저장
// 요청 1 완료 → 세 개 모두 완료 → tuple 생성 → 다운스트림 실행

// Mono.zip() 완료 시각 = max(각 Mono 완료 시각)

// 구현:
public static <T1, T2, T3> Mono<Tuple3<T1, T2, T3>> zip(
        Mono<? extends T1> p1,
        Mono<? extends T2> p2,
        Mono<? extends T3> p3) {

    return onAssembly(new MonoZip<>(
        false,  // 실패 시 다른 Mono도 취소
        a -> Tuples.of(a[0], a[1], a[2]),
        p1, p2, p3
    ));
}

// MonoZip 내부:
// 각 Mono에 대해 ZipInner 구독자 등록
// ZipInner.onNext() 호출 시: 내부 배열에 값 저장
// 모든 ZipInner가 값 받으면 → combinator 함수 호출 → 결과 발행
```

### 2. 순차 vs 병렬 선택 기준

```java
// 순차 호출이 필요한 경우:
// - 다음 호출이 이전 호출 결과에 의존할 때

Mono<OrderDetailResponse> sequential() {
    return orderClient.getOrder(orderId)
        // ① order 결과로 userId 획득
        .flatMap(order -> userClient.getUser(order.getUserId())
            // ② user 결과와 order 결합
            .map(user -> new OrderDetailResponse(order, user))
        );
}

// 병렬 호출이 가능한 경우:
// - 호출 간 의존성 없을 때 (orderId만 알면 됨)

Mono<OrderDetailResponse> parallel() {
    Mono<Order> orderMono = orderClient.getOrder(orderId);
    Mono<Payment> paymentMono = paymentClient.getPayment(orderId);

    // orderId만 있으면 두 호출 모두 가능 → 병렬!
    return Mono.zip(orderMono, paymentMono)
        .map(t -> new OrderDetailResponse(t.getT1(), t.getT2()));
}

// 혼합: 일부 의존, 일부 독립
Mono<OrderDetailResponse> mixed() {
    // orderId → order (필요)
    return orderClient.getOrder(orderId)
        .flatMap(order -> {
            // order.userId 알게 됨 → user 호출
            Mono<User> userMono = userClient.getUser(order.getUserId());
            // payment는 orderId만 있으면 됨 → 병렬!
            Mono<Payment> paymentMono = paymentClient.getPayment(orderId);

            return Mono.zip(userMono, paymentMono)
                .map(t -> new OrderDetailResponse(order, t.getT1(), t.getT2()));
        });
}
```

### 3. 부분 실패 처리 전략

```java
// 전략 1: 기본값으로 대체 (onErrorReturn)
Mono<UserDto> userMono = userClient.getUser(userId)
    .onErrorReturn(UserDto.anonymous());  // 실패 시 익명 사용자

// 전략 2: 빈 Optional로 대체 (onErrorResume)
Mono<Optional<PaymentDto>> paymentMono = paymentClient.getPayment(orderId)
    .map(Optional::of)
    .onErrorResume(e -> {
        log.warn("Payment service unavailable: {}", e.getMessage());
        return Mono.just(Optional.empty());
    });

// 전략 3: 타임아웃 + 폴백
Mono<InventoryDto> inventoryMono = inventoryClient.getInventory(productId)
    .timeout(Duration.ofMillis(500))       // 500ms 타임아웃
    .onErrorResume(TimeoutException.class,
        e -> Mono.just(InventoryDto.unavailable()));  // 타임아웃 시 기본값

// 전략 4: 응답 품질 표시
@Data
public class PartialOrderDetailResponse {
    private OrderDto order;
    private UserDto user;
    private PaymentDto payment;
    private Set<String> unavailableServices;  // 실패한 서비스 목록
    private boolean isComplete;               // 모든 데이터 완전한지
}

// 조합 시 실패 서비스 추적
Mono<PartialOrderDetailResponse> safeGetOrderDetail(Long orderId) {
    Set<String> failures = new ConcurrentHashSet<>();

    Mono<OrderDto> orderMono = orderClient.getOrder(orderId)
        .onErrorResume(e -> {
            failures.add("order-service");
            return Mono.just(OrderDto.empty());
        });

    Mono<PaymentDto> paymentMono = paymentClient.getPayment(orderId)
        .onErrorResume(e -> {
            failures.add("payment-service");
            return Mono.just(PaymentDto.empty());
        });

    return Mono.zip(orderMono, paymentMono)
        .map(t -> PartialOrderDetailResponse.builder()
            .order(t.getT1())
            .payment(t.getT2())
            .unavailableServices(failures)
            .isComplete(failures.isEmpty())
            .build());
}
```

### 4. N+1 문제와 해결 방법

```java
// N+1 문제: 목록 조회 시 각 항목마다 개별 호출
// ❌ N+1 발생
public Flux<OrderSummary> getOrderList(String userId) {
    return orderClient.getOrdersByUser(userId)  // 주문 N개 조회
        .flatMap(order -> userClient.getUser(order.getUserId())  // N번 호출!
            .map(user -> new OrderSummary(order, user)));
}

// ✅ 배치 조회로 N+1 해결
public Flux<OrderSummary> getOrderListOptimized(String userId) {
    return orderClient.getOrdersByUser(userId)
        .collectList()  // N개 주문을 List로 수집
        .flatMap(orders -> {
            // userId 목록 추출
            Set<String> userIds = orders.stream()
                .map(Order::getUserId)
                .collect(Collectors.toSet());

            // 배치 조회: N개 userId → 1번 호출
            return userClient.getUsersByIds(userIds)  // 1번만 호출!
                .collectMap(User::getId)  // Map<userId, User>
                .map(userMap -> orders.stream()
                    .map(order -> new OrderSummary(
                        order,
                        userMap.getOrDefault(order.getUserId(), User.unknown())))
                    .collect(Collectors.toList()));
        })
        .flatMapMany(Flux::fromIterable);
}
```

### 5. 응답 캐싱으로 성능 최적화

```java
// 같은 요청 내에서 동일한 서비스를 여러 번 호출하는 경우 캐싱
// Mono.cache(): 첫 번째 구독 결과를 캐시

@Service
public class CachedCompositionService {

    private final WebClient userWebClient;

    // 요청 스코프 캐시 (같은 요청 내 중복 호출 방지)
    public Mono<UserDto> getCachedUser(String userId) {
        return userWebClient.get()
            .uri("/users/{id}", userId)
            .retrieve()
            .bodyToMono(UserDto.class)
            .cache();  // 첫 구독 결과를 캐시 → 같은 Mono 재구독 시 재사용
    }

    // 여러 곳에서 같은 userId를 참조해도 API 1번만 호출
    public Mono<ComplexResponse> getComplexData(String userId) {
        Mono<UserDto> cachedUser = getCachedUser(userId);

        return Mono.zip(
            cachedUser,                    // 첫 번째 사용
            getOrderHistory(userId, cachedUser),  // 두 번째 사용 (캐시 히트)
            getRecommendations(userId, cachedUser) // 세 번째 사용 (캐시 히트)
        ).map(tuple -> new ComplexResponse(
            tuple.getT1(), tuple.getT2(), tuple.getT3()));
    }
}
```

---

## 💻 실전 구성

### WebClient 공통 설정

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient orderWebClient(WebClient.Builder builder) {
        return builder
            .baseUrl("http://order-service")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .filter(ExchangeFilterFunctions.basicAuthentication(
                "gateway", "secret"))   // 서비스 간 인증
            .build();
    }

    // @LoadBalanced: Eureka에서 인스턴스 자동 선택
    @Bean
    @LoadBalanced
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder()
            .filter(traceIdPropagationFilter())  // Trace 헤더 자동 전파
            .codecs(configurer ->
                configurer.defaultCodecs()
                    .maxInMemorySize(256 * 1024));  // 256KB 응답 크기 제한
    }
}
```

### 완전한 API Composition 엔드포인트

```java
@RestController
@RequestMapping("/api/v2/orders")
public class OrderCompositionController {

    @GetMapping("/{id}/detail")
    public Mono<OrderDetailResponse> getDetail(@PathVariable Long id) {
        return compositionService.getOrderDetail(id)
            .doOnSuccess(r -> log.info("Composition success for order {}", id))
            .doOnError(e -> log.error("Composition failed for order {}", id, e));
    }

    @GetMapping("/user/{userId}/summary")
    public Flux<OrderSummary> getUserOrderSummary(@PathVariable String userId) {
        return compositionService.getOrderListOptimized(userId)
            .timeout(Duration.ofSeconds(5))
            .onErrorResume(TimeoutException.class, e ->
                Flux.just(OrderSummary.timeoutFallback()));
    }
}
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 결제 서비스 다운 시 주문 상세 응답

```
payment-service 다운 상황:

응답 없는 경우 (부분 실패 미처리):
  GET /orders/123/detail
  → orderMono: 성공 (50ms)
  → paymentMono: 타임아웃 (2s) → 전체 응답 실패
  → 클라이언트: 502 오류 수신

부분 실패 허용 처리:
  GET /orders/123/detail
  → orderMono: 성공 (50ms)
  → paymentMono: 타임아웃 → PaymentDto.empty() 대체
  → 클라이언트:
  {
    "orderId": 123,
    "status": "CONFIRMED",
    "user": { "name": "홍길동", ... },
    "payment": null,          ← 결제 정보 없음 표시
    "paymentAvailable": false ← UI에서 "결제 정보 조회 중" 표시
  }

결과:
  결제 서비스 다운이어도 주문 기본 정보 정상 제공
  클라이언트 UX: "결제 정보를 잠시 후 확인해주세요" 메시지 표시
  완전한 실패 대신 부분적 서비스 제공
```

---

## ⚖️ 트레이드오프

| 방식 | 레이턴시 | 데이터 신선도 | 복잡도 |
|------|----------|-------------|--------|
| **순차 호출** | 높음 (누적) | 실시간 | 낮음 |
| **병렬 호출 (zip)** | 낮음 (max) | 실시간 | 중간 |
| **병렬 + 캐시** | 최저 | 캐시 만료까지 | 높음 |
| **이벤트 기반 조합** | 즉각 | 최종 일관성 | 매우 높음 |

```
API Gateway에서 Composition vs 서비스 내부에서:

Gateway에서 (BFF 패턴):
  장점: 클라이언트별 맞춤 응답 (Mobile vs Web 다른 필드)
  단점: Gateway가 각 서비스 스키마를 알아야 함

전용 Composition 서비스:
  장점: 재사용 가능, 캐싱 적용 용이
  단점: 추가 서비스 = 추가 운영 비용

GraphQL:
  클라이언트가 필요한 필드만 선택적 쿼리
  Composition 로직을 GraphQL 리졸버로 분산
  학습 곡선 높지만 N+1 해결 (DataLoader)
```

---

## 📌 핵심 정리

```
API Composition 핵심:

필요 이유:
  MSA에서 DB JOIN 불가 → 여러 서비스 호출 + 응답 집계

병렬 호출 (Mono.zip):
  모든 Mono를 동시에 구독 → Non-Blocking 병렬 실행
  완료 시각 = max(각 Mono 완료 시각)
  순차 대비: (50+30+40)ms → max(50,30,40)ms = 2.4배 빠름

순차 vs 병렬 기준:
  의존성 있음 (다음 호출이 이전 결과 필요): flatMap 순차
  의존성 없음 (orderId만 있으면 됨): zip 병렬

부분 실패 처리:
  onErrorReturn(기본값): 실패 시 기본값
  onErrorResume: 실패 시 대체 로직
  timeout + onErrorReturn: 타임아웃 제한 + 폴백

N+1 해결:
  목록 → collectList() → 배치 ID 추출 → 배치 조회 API

캐싱:
  Mono.cache(): 같은 Mono 재구독 시 캐시 결과 재사용
  같은 요청 내 중복 API 호출 방지
```

---

## 🤔 생각해볼 문제

**Q1.** `Mono.zip()`에서 하나의 Mono가 `onErrorReturn()`으로 처리됐는데도 다른 Mono가 계속 실행된다. `onErrorReturn()` 없이 예외를 던지면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

`Mono.zip()`의 기본 동작은 하나의 Mono에서 오류가 발생하면 다른 Mono들을 즉시 취소(cancel)하고 오류를 전파합니다. 즉, `onErrorReturn()` 없이 예외가 전파되면 `zip`이 즉시 종료되고 다운스트림에 오류 신호가 전달됩니다. 클라이언트는 오류 응답을 받게 됩니다.

`onErrorReturn()`을 사용하면 해당 Mono의 오류를 기본값으로 변환해 오류 신호가 `zip`에 전달되지 않습니다. 다른 Mono들은 취소 없이 계속 실행됩니다.

`Mono.zipDelayError()`를 사용하면 모든 Mono를 실행 완료할 때까지 기다린 후 발생한 모든 오류를 집계해서 전달합니다. 부분 실패를 모두 수집하고 싶을 때 유용합니다.

</details>

---

**Q2.** `Mono.zip(orderMono, userMono, paymentMono)`에서 userMono가 orderMono 결과에 의존하는데 어떻게 병렬 처리하는가?

<details>
<summary>해설 보기</summary>

의존성이 있다면 완전한 병렬 처리는 불가능합니다. 그러나 부분적 병렬화는 가능합니다:

```java
return orderClient.getOrder(orderId)  // ① order 먼저 가져옴
    .flatMap(order -> {
        // ② order 있으면 user와 payment 병렬 조회
        Mono<User> userMono = userClient.getUser(order.getUserId()); // order.userId 필요
        Mono<Payment> paymentMono = paymentClient.getPayment(orderId); // orderId만 필요

        // user와 payment는 서로 독립 → 병렬!
        return Mono.zip(userMono, paymentMono)
            .map(t -> new OrderDetail(order, t.getT1(), t.getT2()));
    });
```

이 패턴에서 총 레이턴시 = order 조회 시간 + max(user 조회, payment 조회) 시간. 완전 직렬(order + user + payment)보다 빠르지만, 완전 병렬(모두 동시)보다는 느립니다. 의존성 구조에 따라 최적의 조합을 선택해야 합니다.

</details>

---

**Q3.** API Composition에서 각 서비스 응답에 서로 다른 타임아웃을 설정해야 할 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

각 서비스의 특성이 다르기 때문입니다.

1. **서비스별 SLA 다름**: Order 서비스는 빠른 DB 조회(100ms), Payment 서비스는 외부 API 연동(500ms).
2. **중요도 차이**: 핵심 데이터(주문 정보)는 긴 타임아웃(응답 필수), 부가 데이터(추천 상품)는 짧은 타임아웃(없어도 됨).
3. **전체 응답 시간 제어**: `Mono.zip()`의 완료 시각은 가장 느린 Mono. 타임아웃을 설정하면 최악의 케이스를 제한.

설정 예:
```java
Mono<Order> orderMono = orderClient.getOrder(id)
    .timeout(Duration.ofMillis(500));  // 핵심: 500ms

Mono<Recommendation> recMono = recClient.getRecommendations(userId)
    .timeout(Duration.ofMillis(100))  // 부가: 100ms
    .onErrorReturn(Recommendation.empty());
```

이렇게 하면 추천 서비스가 느려도 100ms만 기다리고 기본값 사용 → 전체 응답 지연이 추천 서비스에 종속되지 않습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Saga Pattern](./02-saga-pattern.md)** | **[홈으로 🏠](../README.md)** | **[다음: CQRS ➡️](./04-cqrs.md)**

</div>
