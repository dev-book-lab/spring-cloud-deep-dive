# Custom Filter 작성 — JWT 검증 Global Filter와 exchange.mutate() 완전 가이드

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `GlobalFilter`와 `GatewayFilterFactory` 중 어느 것을 선택해야 하는 기준은?
- JWT 검증 Global Filter에서 토큰 파싱 후 사용자 정보를 업스트림에 전달하는 방법은?
- `exchange.mutate().request(newRequest).build()` 패턴에서 각 단계의 역할은?
- Reactive 체인에서 인증 실패 시 즉시 응답을 반환하고 체인을 중단하는 방법은?
- 응답(Post)을 수정할 때 응답 본문(Body)을 변환하려면 어떻게 해야 하는가?
- Custom Filter를 테스트하는 방법은?

---

## 🔍 왜 MSA에서 필요한가

### Custom Filter가 필요한 실제 케이스

```
내장 필터로 해결 안 되는 케이스들:

① JWT 파싱 + 사용자 정보 헤더 주입
  AddRequestHeader: 정적 값만 가능
  → 실제로는 JWT에서 사용자 ID, 역할 동적 추출 필요
  → Custom Filter 필수

② 요청 로깅 + 추적 ID 생성
  기존 트레이싱과 통합하거나
  게이트웨이 전용 X-Request-Id 생성
  → 모든 요청에 적용 (Global Filter)

③ IP 기반 Rate Limiting (Redis 연동)
  특정 IP에서 초당 N개 이상 요청 시 429
  → Redis에서 카운터 조회/갱신 (Reactive)
  → Custom GatewayFilterFactory

④ 응답 본문 마스킹
  업스트림 응답의 민감 데이터(카드번호, 주민번호) 마스킹
  → 응답 본문 스트림 변환
  → Custom Global Filter (Post)

⑤ 멀티테넌트 라우팅
  JWT의 tenantId를 기반으로 다른 URI로 라우팅
  → Dynamic URI 결정
  → Custom Filter에서 GATEWAY_REQUEST_URL_ATTR 수정
```

---

## 😱 잘못된 구성

### Before: 이벤트 루프에서 블로킹 JWT 검증

```java
// ❌ 블로킹 JWT 파싱 (동기 라이브러리 직접 사용)
@Component
@Order(-100)
public class JwtFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders()
            .getFirst("Authorization");

        // ❌ 블로킹 DB 조회 → 이벤트 루프 스레드 점유
        User user = userRepository.findByToken(token);  // JDBC 블로킹!

        if (user == null) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }
}
```

### Before: 응답 헤더를 너무 늦게 수정

```java
// ❌ 응답이 이미 커밋된 후 헤더 수정 시도
@Component
public class ResponseHeaderFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return chain.filter(exchange)
            .doOnSuccess(v -> {
                // ❌ NettyWriteResponseFilter가 이미 응답 전송 시작
                // isCommitted() = true → 헤더 변경 무시됨
                exchange.getResponse().getHeaders()
                    .add("X-Response-Header", "value");
            });
    }
}
```

---

## ✨ 올바른 패턴

### After: GlobalFilter vs GatewayFilterFactory 선택 기준

```
GlobalFilter:
  모든 Route에 자동 적용
  @Component 등록으로 즉시 활성화
  설정 파라미터 없음 (또는 @Value, @ConfigurationProperties)

  사용 케이스:
  ✅ 인증/인가 (모든 요청에 적용)
  ✅ 요청/응답 로깅
  ✅ 분산 추적 (Trace ID 주입)
  ✅ 요청 크기 제한 (글로벌 정책)

GatewayFilterFactory:
  특정 Route에만 선택적으로 적용
  YAML에서 설정 파라미터 전달 가능
  필터 체인 내 순서 명시 가능

  사용 케이스:
  ✅ 경로별 다른 인증 (일부 Route는 Public)
  ✅ 특정 Route의 요청 변환
  ✅ Route별 다른 Rate Limit 설정
  ✅ 재사용 가능한 로직 (여러 Route에 다른 파라미터로 적용)
```

---

## 🔬 내부 동작 원리

### 1. JWT 검증 Global Filter — 완전한 구현

```java
// JwtAuthenticationGlobalFilter.java
@Component
@Order(-100)  // 가장 먼저 실행
public class JwtAuthenticationGlobalFilter implements GlobalFilter {

    private final JwtTokenParser jwtParser;
    // JwtTokenParser: io.jsonwebtoken.Jwts 사용, CPU 바운드이므로 Non-Blocking OK
    // 단, 키 로딩이 DB/네트워크 필요 시 Reactive 처리 필요

    private static final Set<String> PUBLIC_PATHS = Set.of(
        "/api/auth/login",
        "/api/auth/register",
        "/actuator/health"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getURI().getPath();

        // ① Public 경로는 인증 건너뜀
        if (PUBLIC_PATHS.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }

        // ② Authorization 헤더 추출
        String authHeader = request.getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return onAuthError(exchange, "Missing or invalid Authorization header");
        }

        String token = authHeader.substring(7);

        // ③ JWT 파싱 (CPU 바운드 → Mono.fromCallable로 감싸기)
        return Mono.fromCallable(() -> jwtParser.parse(token))
            // CPU 바운드 작업이지만 빠르므로 현재 스레드 사용
            // DB 조회가 필요하다면: .subscribeOn(Schedulers.boundedElastic())
            .flatMap(claims -> {
                String userId = claims.getSubject();
                String roles = String.join(",", claims.get("roles", List.class));

                // ④ JWT 클레임을 요청 헤더에 주입 (업스트림으로 전달)
                ServerHttpRequest mutatedRequest = request.mutate()
                    .header("X-User-Id", userId)
                    .header("X-User-Roles", roles)
                    .header("X-Authenticated", "true")
                    // 원본 JWT는 제거 (업스트림 서비스 보안)
                    .headers(headers -> headers.remove(HttpHeaders.AUTHORIZATION))
                    .build();

                // ⑤ mutate된 exchange로 체인 계속
                return chain.filter(
                    exchange.mutate()
                        .request(mutatedRequest)
                        .build()
                );
            })
            .onErrorResume(JwtException.class, e ->
                onAuthError(exchange, "Invalid or expired token: " + e.getMessage()))
            .onErrorResume(Exception.class, e ->
                onAuthError(exchange, "Authentication error"));
    }

    private Mono<Void> onAuthError(ServerWebExchange exchange, String message) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON);

        // JSON 에러 응답 작성
        String body = """
            {"error": "Unauthorized", "message": "%s"}
            """.formatted(message);
        DataBuffer buffer = response.bufferFactory()
            .wrap(body.getBytes(StandardCharsets.UTF_8));

        // chain.filter() 호출 없이 응답 직접 반환 → 체인 중단
        return response.writeWith(Mono.just(buffer));
    }
}
```

### 2. exchange.mutate() 패턴 상세

```java
// ServerWebExchange, ServerHttpRequest, ServerHttpResponse 모두 불변
// 수정하려면 반드시 mutate() 사용

// ServerHttpRequest 수정 패턴:
ServerHttpRequest originalRequest = exchange.getRequest();

ServerHttpRequest newRequest = originalRequest.mutate()
    // 헤더 추가
    .header("X-New-Header", "value")
    // 헤더 제거
    .headers(headers -> headers.remove("X-Remove-This"))
    // 특정 헤더 값 변경
    .headers(headers -> headers.set("X-Existing", "new-value"))
    // 경로 변경
    .path("/new/path")
    // URI 전체 변경
    .uri(URI.create("http://new-host:8080/path"))
    .build();

// ServerWebExchange에 새 Request 적용:
ServerWebExchange newExchange = exchange.mutate()
    .request(newRequest)
    .build();

// 체인에 새 Exchange 전달:
return chain.filter(newExchange);

// 주의: 원본 exchange는 불변이므로 newExchange를 chain에 전달해야 함
// chain.filter(exchange) ← 잘못됨 (원본 전달)
// chain.filter(newExchange) ← 올바름 (수정된 버전 전달)
```

### 3. 응답 본문(Body) 변환 — ModifyResponseBodyFilter 패턴

```java
// 응답 본문 변환은 가장 복잡한 패턴
// DataBuffer 스트림을 직접 처리해야 함

@Component
@Order(100)  // Post-filter, 나중 실행 (높은 Order → Post 먼저)
public class SensitiveDataMaskingFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 원본 Response를 가로채는 Decorator 생성
        ServerHttpResponseDecorator decoratedResponse =
            new ServerHttpResponseDecorator(exchange.getResponse()) {

                @Override
                public Mono<Void> writeWith(Publisher<? extends DataBuffer> body) {
                    // 응답 Content-Type이 JSON일 때만 처리
                    MediaType contentType = getHeaders().getContentType();
                    if (MediaType.APPLICATION_JSON.isCompatibleWith(contentType)) {

                        // DataBuffer 스트림 수집 → 문자열 변환 → 마스킹 → 재직렬화
                        Flux<DataBuffer> transformedBody = Flux.from(body)
                            .collectList()
                            .map(dataBuffers -> {
                                // ① 모든 DataBuffer를 하나로 합침
                                DataBufferUtils.join(Flux.fromIterable(dataBuffers));
                                byte[] bytes = dataBuffers.stream()
                                    .reduce(new byte[0], (a, b) -> {
                                        byte[] combined = new byte[a.length + b.remaining()];
                                        System.arraycopy(a, 0, combined, 0, a.length);
                                        b.get(combined, a.length, b.remaining());
                                        return combined;
                                    }, (a, b) -> b);

                                // ② 마스킹 처리
                                String originalBody = new String(bytes, StandardCharsets.UTF_8);
                                String maskedBody = maskSensitiveData(originalBody);

                                // ③ 새 DataBuffer로 래핑
                                return getDelegate().bufferFactory()
                                    .wrap(maskedBody.getBytes(StandardCharsets.UTF_8));
                            })
                            .flux();

                        return super.writeWith(transformedBody);
                    }
                    return super.writeWith(body);  // JSON 아니면 그대로
                }
            };

        // Decorator를 exchange에 적용
        return chain.filter(exchange.mutate().response(decoratedResponse).build());
    }

    private String maskSensitiveData(String json) {
        // 카드번호 마스킹: "1234-5678-9012-3456" → "****-****-****-3456"
        return json.replaceAll("\"cardNumber\":\"(\\d{4}-\\d{4}-\\d{4}-)(\\d{4})\"",
            "\"cardNumber\":\"****-****-****-$2\"");
    }
}
```

### 4. Custom GatewayFilterFactory — 파라미터 있는 필터

```java
// RequestIdGatewayFilterFactory: 요청마다 고유 ID 추가
// YAML: filters: - name: RequestId, args: prefix: "order-"
@Component
public class RequestIdGatewayFilterFactory
        extends AbstractGatewayFilterFactory<RequestIdGatewayFilterFactory.Config> {

    public RequestIdGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return List.of("prefix");  // shortcut: - RequestId=order-
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            // 요청마다 고유 ID 생성
            String requestId = config.getPrefix() + UUID.randomUUID();

            ServerHttpRequest newRequest = exchange.getRequest().mutate()
                .header("X-Request-Id", requestId)
                .build();

            // 응답에도 같은 ID 반환 (추적용)
            ServerHttpResponse response = exchange.getResponse();

            return chain.filter(exchange.mutate().request(newRequest).build())
                .doFirst(() -> response.getHeaders()
                    .add("X-Request-Id", requestId));
            // doFirst: 체인 실행 전에 응답 헤더 예약
            // 응답이 커밋되기 전에 추가됨
        };
    }

    @Data
    public static class Config {
        private String prefix = "";
    }
}
```

### 5. Custom Filter 테스트

```java
// JwtAuthenticationGlobalFilter 단위 테스트
@ExtendWith(MockitoExtension.class)
class JwtAuthenticationGlobalFilterTest {

    @Mock private JwtTokenParser jwtParser;
    @Mock private GatewayFilterChain chain;
    private JwtAuthenticationGlobalFilter filter;

    @BeforeEach
    void setUp() {
        filter = new JwtAuthenticationGlobalFilter(jwtParser);
        when(chain.filter(any())).thenReturn(Mono.empty());
    }

    @Test
    void validToken_shouldPassAndInjectHeaders() {
        // Given
        Claims claims = mock(Claims.class);
        when(claims.getSubject()).thenReturn("user123");
        when(claims.get("roles", List.class)).thenReturn(List.of("ROLE_USER"));
        when(jwtParser.parse("valid-token")).thenReturn(claims);

        MockServerHttpRequest request = MockServerHttpRequest
            .get("/api/orders")
            .header("Authorization", "Bearer valid-token")
            .build();
        MockServerWebExchange exchange = MockServerWebExchange.from(request);

        // When
        StepVerifier.create(filter.filter(exchange, chain))
            .verifyComplete();

        // Then: chain.filter()가 호출됐고 헤더가 주입됐는지 확인
        verify(chain).filter(argThat(e ->
            "user123".equals(e.getRequest().getHeaders().getFirst("X-User-Id"))));
    }

    @Test
    void missingToken_shouldReturn401() {
        // Given
        MockServerHttpRequest request = MockServerHttpRequest
            .get("/api/orders")
            .build();
        MockServerWebExchange exchange = MockServerWebExchange.from(request);

        // When
        StepVerifier.create(filter.filter(exchange, chain))
            .verifyComplete();

        // Then: 401 상태 코드, chain은 호출 안 됨
        assertThat(exchange.getResponse().getStatusCode())
            .isEqualTo(HttpStatus.UNAUTHORIZED);
        verify(chain, never()).filter(any());
    }
}

// 통합 테스트 (WebTestClient)
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class GatewayIntegrationTest {

    @Autowired private WebTestClient webTestClient;

    @Test
    void withValidToken_shouldRouteToUpstream() {
        String token = generateTestToken("user123", List.of("ROLE_USER"));

        webTestClient.get()
            .uri("/api/orders/1")
            .header("Authorization", "Bearer " + token)
            .exchange()
            .expectStatus().isOk()
            .expectHeader().exists("X-Request-Id");
    }
}
```

---

## 💻 실전 구성

### 실무 Custom Filter 조합

```java
// 요청 로깅 + 응답 시간 측정
@Component
@Order(-50)
public class AccessLogGlobalFilter implements GlobalFilter {

    private static final Logger log = LoggerFactory.getLogger("ACCESS_LOG");

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest req = exchange.getRequest();
        String requestId = req.getHeaders().getFirst("X-Request-Id");
        long start = System.currentTimeMillis();

        return chain.filter(exchange)
            .doFinally(signalType -> {
                long elapsed = System.currentTimeMillis() - start;
                HttpStatus status = exchange.getResponse().getStatusCode();
                String routeId = exchange.<Route>getAttribute(GATEWAY_ROUTE_ATTR) != null
                    ? exchange.<Route>getAttribute(GATEWAY_ROUTE_ATTR).getId()
                    : "unknown";

                log.info("[{}] {} {} → {} {} {}ms",
                    requestId,
                    req.getMethod(),
                    req.getURI().getPath(),
                    routeId,
                    status != null ? status.value() : "?",
                    elapsed);
            });
    }
}
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 멀티테넌트 라우팅 — JWT의 tenantId로 다른 URI로 분기

```java
@Component
@Order(-80)
public class TenantRoutingFilter implements GlobalFilter {

    private final Map<String, String> tenantUriMap = Map.of(
        "tenant-a", "lb://service-tenant-a",
        "tenant-b", "lb://service-tenant-b",
        "default", "lb://service-default"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // JwtAuthFilter가 이미 실행되어 X-Tenant-Id 헤더가 있다고 가정
        String tenantId = exchange.getRequest().getHeaders()
            .getFirst("X-Tenant-Id");

        String targetUri = tenantUriMap.getOrDefault(tenantId, tenantUriMap.get("default"));

        // 라우팅 URI 동적 변경
        URI uri = URI.create(targetUri + exchange.getRequest().getURI().getPath());
        exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, uri);

        return chain.filter(exchange);
    }
}
```

---

## ⚖️ 트레이드오프

| 방식 | 복잡도 | 재사용성 | 파라미터 |
|------|--------|---------|---------|
| **GlobalFilter** | 낮음 | 모든 Route 자동 적용 | 없음 (설정 클래스) |
| **GatewayFilterFactory** | 중간 | 선택적 적용 | YAML에서 전달 |
| **AbstractGatewayFilterFactory** | 중간 | Shortcut 문법 지원 | 타입 안전 Config |

```
응답 본문 변환의 비용:
  DataBuffer를 수집하고 문자열로 변환하면:
    대용량 응답 → 전체 본문을 메모리에 로드 필요
    → 메모리 사용량 급증

  대안:
  1. 스트리밍 처리: DataBuffer를 청크 단위로 변환 (복잡)
  2. 변환 대상 한정: Content-Type 필터링, 크기 제한
  3. 업스트림에서 마스킹 처리 (Gateway에서 안 하기)
  
  권장: 응답 본문 변환은 업스트림에서 처리
        Gateway는 헤더 레벨 변환에 집중
```

---

## 📌 핵심 정리

```
Custom Filter 작성 핵심:

GlobalFilter vs GatewayFilterFactory 선택:
  GlobalFilter: 모든 Route, 공통 인증/로깅
  GatewayFilterFactory: 특정 Route, 파라미터 필요

exchange.mutate() 패턴:
  1. request.mutate().header(...).build() → 새 Request
  2. exchange.mutate().request(newRequest).build() → 새 Exchange
  3. chain.filter(newExchange) → 수정된 Exchange 전달

인증 실패 시 체인 중단:
  response.setStatusCode(UNAUTHORIZED)
  return response.writeWith(errorBodyMono)
  // chain.filter() 호출 없음 → 나머지 필터 실행 안 됨

Post-filter 응답 헤더 추가:
  chain.filter(exchange).doFirst(() →
    response.getHeaders().add(...))
  // 응답 커밋 전에 헤더 추가 보장

응답 본문 변환:
  ServerHttpResponseDecorator 서브클래스
  writeWith() 오버라이드
  → DataBuffer 스트림 변환
  → exchange.mutate().response(decorator).build()

블로킹 금지:
  이벤트 루프 스레드에서 블로킹 I/O 절대 금지
  블로킹 필요 시: .subscribeOn(Schedulers.boundedElastic())
```

---

## 🤔 생각해볼 문제

**Q1.** `chain.filter(exchange)` 대신 `chain.filter(exchange.mutate().build())`를 호출해도 동작하는가? `exchange.mutate().build()`는 무엇을 반환하는가?

<details>
<summary>해설 보기</summary>

동작합니다. `exchange.mutate().build()`는 아무것도 변경하지 않고 동일한 속성을 가진 새 `ServerWebExchange` 인스턴스를 생성합니다. 얕은 복사이므로 원본과 내용이 같습니다. 실제로는 `exchange.mutate().request(newRequest).build()`처럼 변경할 내용을 지정해야 의미가 있습니다. 단순히 `chain.filter(exchange)`를 호출하는 것과 `chain.filter(exchange.mutate().build())`는 기능상 동일하지만 불필요한 객체 생성이 발생합니다. 변경 사항이 없으면 원본 exchange를 그대로 전달하는 것이 더 효율적입니다.

</details>

---

**Q2.** JWT 필터에서 `onErrorResume(JwtException.class, ...)`으로 토큰 파싱 실패를 처리했다. 그런데 `jwtParser.parse(token)`이 RuntimeException 서브클래스가 아닌 `Error`를 던진다면 처리되는가?

<details>
<summary>해설 보기</summary>

처리되지 않습니다. `onErrorResume(JwtException.class, ...)`은 `Throwable` 중 `JwtException` 타입인 것만 처리합니다. `Error`는 `Exception`의 부모가 아닌 `Throwable`의 별도 계층이므로 `JwtException.class` 필터에 걸리지 않습니다. `onErrorResume(Exception.class, ...)`도 `Error`는 포함하지 않습니다. `Error`까지 처리하려면 `onErrorResume(Throwable.class, ...)` 또는 `.onErrorResume(e -> e instanceof Error || ..., ...)`를 사용해야 합니다. 실제로 JWT 라이브러리에서 `Error`를 던지는 경우는 드물지만, `OutOfMemoryError` 같은 JVM 레벨 오류는 처리하지 않는 것이 일반적입니다.

</details>

---

**Q3.** Global Filter에서 Exchange 속성(`exchange.getAttributes().put(...)`)에 값을 저장했다. 이 값을 다음 필터나 업스트림 서비스에서 읽을 수 있는가? HTTP 헤더와의 차이는?

<details>
<summary>해설 보기</summary>

**다음 필터에서**: 읽을 수 있습니다. `exchange.getAttributes()`는 요청 처리 중 필터 체인 내에서 공유되는 `ConcurrentHashMap`입니다. 앞 단계의 필터가 저장한 값을 뒤 필터에서 `exchange.getAttribute(key)`로 읽을 수 있습니다.

**업스트림 서비스에서**: 직접은 읽을 수 없습니다. `exchange.getAttributes()`는 Gateway JVM 내부의 메모리 구조로, HTTP 요청과 함께 업스트림으로 전송되지 않습니다. 업스트림에 값을 전달하려면 반드시 HTTP 헤더(`exchange.getRequest().mutate().header(key, value)`)로 변환해야 합니다.

**활용 패턴**: JWT 필터에서 파싱된 Claims를 `exchange.getAttributes()`에 저장하고, 이후 필터에서 꺼내어 헤더로 주입하거나 로깅에 활용합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Gateway Timeout & Circuit Breaker 통합](./06-gateway-timeout-circuit-breaker.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 — Load Balancing ➡️](../load-balancing/01-ribbon-vs-spring-cloud-lb.md)**

</div>
