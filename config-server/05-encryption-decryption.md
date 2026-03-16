# Encryption / Decryption — Config Server에서 민감 정보를 안전하게 관리하는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `{cipher}` 접두사가 붙은 값을 Config Server가 언제, 어느 레이어에서 복호화하는가?
- 대칭키(AES)와 비대칭키(RSA) 방식의 내부 구현 차이는 무엇인가?
- `TextEncryptorLocator`는 Bean별로 다른 암호화 키를 적용할 때 어떻게 동작하는가?
- Config Client 측에서 복호화하는 방식(`spring.cloud.config.server.encrypt.enabled=false`)의 장단점은?
- JCE(Java Cryptography Extension) Unlimited Strength가 Java 8에서 필요했던 이유와 Java 9+에서 해결된 방식은?
- `/encrypt`, `/decrypt` 엔드포인트로 암호화 값을 생성하는 과정은?

---

## 🔍 왜 MSA에서 필요한가

### Git 저장소에 민감 정보를 평문으로 저장하면 안 되는 이유

```
현실적인 위협:
  1. 실수로 public 저장소에 push
     → 수 분 내에 GitHub Secret Scanning 또는 악의적 봇이 탐지
     → 키 무효화해도 Git 히스토리에 영구 기록

  2. 내부자 위협
     → Config 저장소 read 권한이 있는 모든 개발자가 prod DB 비밀번호를 볼 수 있음
     → 퇴사 후에도 Git clone된 로컬 복사본에 남아있음

  3. Git 저장소 토큰 탈취
     → 저장소 토큰이 유출되면 모든 민감 정보가 한 번에 노출

이상적인 상태:
  Git 저장소 (public이어도 안전):
    spring.datasource.password: {cipher}ABCDEF123456...  ← 암호화된 값
  
  Config Server만 복호화 키 보유 (환경 변수로 주입)
  
  클라이언트는 평문 값 수신 (TLS 적용 전제)
```

---

## 😱 잘못된 구성

### Before: 평문 비밀번호가 Git에

```yaml
# ❌ config-repo/service-a-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-db.internal:3306/myapp
    username: app_user
    password: MySecretProd#2024   # Git에 평문으로 노출

aws:
  credentials:
    access-key: AKIAIOSFODNN7EXAMPLE
    secret-key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

### Before: 환경 변수로만 관리 (Config Server 미사용)

```yaml
# ❌ application.yml에서 환경 변수 참조
spring:
  datasource:
    password: ${DB_PASSWORD}  # 각 서비스 인스턴스에 환경 변수 개별 설정

# 문제:
# 수십 개 서비스 × 수십 개 민감 설정 = 환경 변수 관리 지옥
# K8s Secret은 base64 인코딩 (암호화 ≠ 인코딩) → 실질적 보안 없음
# 변경 시 모든 인스턴스 환경 변수 수정 + 재시작 필요
```

---

## ✨ 올바른 패턴

### After: {cipher} 접두사로 암호화 저장

```yaml
# ✅ config-repo/service-a-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-db.internal:3306/myapp
    username: app_user
    password: '{cipher}AQBhBhJ4...(암호화된 긴 문자열)...'

aws:
  credentials:
    access-key: '{cipher}AQCA...'
    secret-key: '{cipher}AQB9...'

# Git에는 암호화된 값만 존재
# 복호화 키는 Config Server 환경 변수로만 보유
# Config Server가 클라이언트에 응답 시 평문으로 복호화하여 전달
```

---

## 🔬 내부 동작 원리

### 1. {cipher} 복호화 타이밍 — EnvironmentEncryptorDecorator

```java
// EnvironmentEncryptorDecorator.java
// EnvironmentController의 응답을 가로채서 복호화

public class EnvironmentEncryptorDecorator implements EnvironmentRepository {

    private final EnvironmentRepository delegate;
    private final EnvironmentEncryptor encryptor;

    @Override
    public Environment findOne(String application, String profile, String label) {
        // ① 먼저 실제 Repository에서 Environment 조회
        Environment environment = this.delegate.findOne(application, profile, label);

        // ② 암호화된 값이 있으면 복호화
        return this.encryptor.decrypt(environment);
    }
}

// EnvironmentEncryptor.decrypt() 동작
public Environment decrypt(Environment environment) {
    List<PropertySource> propertySources = new ArrayList<>();

    for (PropertySource source : environment.getPropertySources()) {
        Map<String, Object> decryptedMap = new LinkedHashMap<>();

        for (Map.Entry<String, Object> entry : source.getSource().entrySet()) {
            String key = entry.getKey();
            String value = entry.getValue().toString();

            // {cipher} 접두사 확인
            if (value.startsWith("{cipher}")) {
                try {
                    // 실제 암호화된 부분 추출 및 복호화
                    String cipherText = value.substring("{cipher}".length());
                    String decrypted = this.encryptor.decrypt(cipherText);
                    decryptedMap.put(key, decrypted);
                } catch (Exception e) {
                    // 복호화 실패 시 에러 표시값으로 대체 (평문 노출 방지)
                    decryptedMap.put(key, "<n/a>");
                    log.error("Cannot decrypt key: " + key);
                }
            } else {
                decryptedMap.put(key, value);
            }
        }
        propertySources.add(new PropertySource(source.getName(), decryptedMap));
    }
    return new Environment(/* ... */);
}
```

### 2. 대칭키(AES) 암호화 설정

```java
// TextEncryptorUtils.java
// encrypt.key가 설정되면 AesTextEncryptor 사용

// Config Server 환경 변수
// ENCRYPT_KEY=MySecretSymmetricKey32Characters!

// application.yml
// encrypt:
//   key: ${ENCRYPT_KEY}

// 내부적으로 사용하는 암호화 알고리즘:
// AES/CBC/PKCS5Padding with PBKDF2WithHmacSHA1 키 파생
// 각 암호화마다 다른 IV(Initialization Vector) 사용
//   → 같은 평문도 암호화할 때마다 다른 결과
//   → rainbow table 공격 방어

// 예시:
// "MyPassword123" 첫 번째 암호화: AQB9x3k...abc
// "MyPassword123" 두 번째 암호화: AQB2m7p...xyz  (다른 값!)
// 둘 다 복호화하면 "MyPassword123"
```

### 3. 비대칭키(RSA) 암호화 설정

```java
// KeyStoreTextEncryptor.java
// RSA 키페어 사용: 공개키로 암호화, 개인키로 복호화

// keystore 생성
// keytool -genkeypair \
//   -alias config-server-key \
//   -keyalg RSA \
//   -keystore config-server.jks \
//   -storepass storepass123 \
//   -keypass keypass123

// application.yml
// encrypt:
//   key-store:
//     location: classpath:/config-server.jks
//     password: ${KEYSTORE_PASSWORD}
//     alias: config-server-key
//     secret: ${KEY_PASSWORD}

// 비대칭키 장점:
//   암호화(공개키)와 복호화(개인키) 권한 분리
//   개발자가 공개키만으로 {cipher} 값 생성 가능
//   Config Server(개인키)만 복호화 가능
//   → 개발자가 암호화된 값을 Git에 커밋 가능, 복호화는 불가

public class RsaSecretEncryptor implements TextEncryptor {
    private final KeyPair keyPair;

    @Override
    public String encrypt(String text) {
        // RSA/ECB/OAEPWithSHA-256AndMGF1Padding
        Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
        cipher.init(Cipher.ENCRYPT_MODE, keyPair.getPublic());
        byte[] encrypted = cipher.doFinal(text.getBytes(StandardCharsets.UTF_8));
        return Base64.getEncoder().encodeToString(encrypted);
    }

    @Override
    public String decrypt(String encryptedText) {
        Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
        cipher.init(Cipher.DECRYPT_MODE, keyPair.getPrivate());
        byte[] decrypted = cipher.doFinal(Base64.getDecoder().decode(encryptedText));
        return new String(decrypted, StandardCharsets.UTF_8);
    }
}
```

### 4. /encrypt, /decrypt 엔드포인트

```java
// EncryptionController.java
// Config Server가 노출하는 암호화 유틸리티 엔드포인트

@RestController
public class EncryptionController {

    @PostMapping("/encrypt")
    public String encrypt(@RequestBody String data) {
        // 평문 → 암호화된 값 반환
        // Config 파일에 {cipher}암호화된값 으로 붙여넣기
        return this.encryptor.encrypt(data);
    }

    @PostMapping("/decrypt")
    public String decrypt(@RequestBody String data) {
        // 암호화된 값 → 평문 반환
        // 복호화 확인 용도
        return this.encryptor.decrypt(data);
    }
}
```

### 5. Client-side Decryption 옵션

```yaml
# Config Server에서 복호화하지 않고 암호화된 채로 클라이언트에 전달
# spring.cloud.config.server.encrypt.enabled=false

# 클라이언트가 직접 복호화
# 클라이언트 application.yml:
# encrypt:
#   key: ${ENCRYPT_KEY}  # 클라이언트도 키 보유

# 장점:
#   TLS 없이도 전송 구간 안전 (이미 암호화된 값 전달)
#   Config Server가 복호화 키를 알 필요 없음

# 단점:
#   모든 클라이언트가 복호화 키 보유 → 키 관리 복잡
#   클라이언트에 spring-security-crypto 의존성 필요
```

---

## 💻 실전 구성

### 대칭키 암호화 전체 실습

```bash
# 1. Config Server에 암호화 키 설정
# docker-compose.yml
services:
  config-server:
    environment:
      - ENCRYPT_KEY=ThisIsA32CharacterSecretKeyForAES!
      - MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE=encrypt,decrypt,health

# 2. 평문을 암호화 (개발자가 실행)
curl -X POST http://localhost:8888/encrypt \
  -d "MyDatabasePassword123"
# 응답: AQBhBhJ4Xk9...긴_암호화_문자열...

# 3. config-repo/service-a-prod.yml에 저장
# spring:
#   datasource:
#     password: '{cipher}AQBhBhJ4Xk9...'
# 주의: YAML에서 중괄호는 따옴표로 감싸야 함

# 4. 복호화 확인
curl -X POST http://localhost:8888/decrypt \
  -d "AQBhBhJ4Xk9..."
# 응답: MyDatabasePassword123

# 5. 서비스 실행 후 복호화 적용 확인
curl http://localhost:8080/actuator/env/spring.datasource.password
# 응답에 실제 비밀번호가 나오면 안 됨 — Actuator가 민감 필드 마스킹
# {"property":{"source":"configService:service-a-prod.yml","value":"******"}}
```

### RSA 키스토어 설정

```bash
# 1. JKS 키스토어 생성
keytool -genkeypair \
  -alias config-server \
  -keyalg RSA \
  -keysize 2048 \
  -dname "CN=Config Server,OU=Dev,O=MyOrg,C=KR" \
  -keystore config-server.jks \
  -storepass changeit \
  -keypass changeit \
  -validity 3650

# 2. 공개키만 추출 (개발자 배포용)
keytool -export \
  -alias config-server \
  -keystore config-server.jks \
  -rfc \
  -file config-server-public.cer

# 3. Config Server에 키스토어 마운트
services:
  config-server:
    volumes:
      - ./config-server.jks:/secrets/config-server.jks
    environment:
      - ENCRYPT_KEYSTORE_LOCATION=file:/secrets/config-server.jks
      - ENCRYPT_KEYSTORE_PASSWORD=${KEYSTORE_PASSWORD}
      - ENCRYPT_KEYSTORE_ALIAS=config-server
      - ENCRYPT_KEYSTORE_SECRET=${KEY_PASSWORD}
```

---

## 🌐 분산 시스템 시나리오

### 시나리오: 암호화 키 로테이션

```
보안 정책: 6개월마다 암호화 키 갱신

현재 상황:
  config-repo에 {cipher}구버전키로암호화된값 이 저장됨
  Config Server에 구버전 키 설정됨

키 로테이션 절차:

1단계: 새 키 생성
  # 새 키로 모든 {cipher} 값 재암호화 스크립트
  OLD_KEY=OldKey32Characters...
  NEW_KEY=NewKey32Characters...

2단계: 점진적 전환
  # Config Server를 새 키로 재시작하기 전에
  # 모든 {cipher} 값을 새 키로 재암호화 후 Git commit

  # 방법 1: Config Server에서 재암호화
  # 임시로 양쪽 키 지원하는 MultiTextEncryptor 구현 (커스텀)
  
  # 방법 2: 스크립트로 일괄 재암호화
  for file in config-repo/**/*.yml; do
    # {cipher} 패턴 찾기
    # 구 키로 복호화
    # 신 키로 암호화
    # 파일 업데이트
  done

3단계: Config Server 키 교체 + 재시작
  ENCRYPT_KEY=NewKey32Characters...

주의:
  키 교체 전에 모든 파일의 {cipher} 값이 새 키로 변환 완료되어야 함
  → 변환 중 서비스가 Config Server 재시작 전에 refresh하면 복호화 실패
  → 유지보수 시간대에 진행 권장
```

---

## ⚖️ 트레이드오프

| 방식 | 보안 | 관리 편의 | 적합한 상황 |
|------|------|-----------|------------|
| **대칭키(AES)** | 중간 (키 하나로 암/복호화) | 단순 | 소규모, 단일 Config Server |
| **비대칭키(RSA)** | 높음 (암호화/복호화 권한 분리) | 복잡 (키스토어 관리) | 대규모, 암호화 권한 분리 필요 |
| **Vault 연동** | 최고 (동적 시크릿, 자동 로테이션) | 인프라 복잡 | 엔터프라이즈, 컴플라이언스 요건 |
| **K8s Secret** | 중간 (etcd 암호화 설정 필요) | K8s 환경에서 자연스러움 | K8s 환경, 외부 도구 최소화 |

```
Config Server 암호화 vs HashiCorp Vault:
  Config Server 암호화:
    장점: 추가 인프라 불필요, Spring과 자연스러운 통합
    단점: 정적 키 (자동 로테이션 없음), 감사 로그 제한적
  
  HashiCorp Vault:
    장점: 동적 시크릿 (매 요청마다 새 자격증명), 자동 만료, 세밀한 감사
    단점: Vault 인프라 별도 운영 필요
    Spring Cloud Vault: spring-cloud-vault-config 의존성으로 통합 가능
```

---

## 📌 핵심 정리

```
Config Server 암호화 핵심:

복호화 타이밍:
  Config Server → EnvironmentEncryptorDecorator.decrypt()
  → 클라이언트에게 평문으로 응답
  (client-side decrypt 옵션 사용 시 암호화된 채로 전달)

{cipher} 값 생성:
  POST /encrypt → 암호화된 값 반환
  config-repo yml에 '{cipher}암호화된값' 저장
  YAML: 중괄호는 반드시 따옴표로 감싸기

대칭키 vs 비대칭키:
  대칭키(AES): encrypt.key 환경 변수 하나로 설정, 간단
  비대칭키(RSA): 키스토어 필요, 암호화/복호화 권한 분리 가능
               개발자: 공개키로 암호화, Config Server: 개인키로 복호화

Java 9+: JCE Unlimited Strength 기본 활성화
  → AES-256 등 강력한 알고리즘 추가 설치 없이 사용 가능

보안 원칙:
  복호화 키는 Config Server 환경 변수로만 (Git 절대 금지)
  TLS 적용 (평문 전송 구간 암호화)
  /encrypt, /decrypt 엔드포인트 접근 제한 (IP 화이트리스트)
```

---

## 🤔 생각해볼 문제

**Q1.** 같은 평문 "password123"을 `/encrypt` 엔드포인트로 두 번 호출했더니 서로 다른 암호화 값이 나왔다. 이것이 버그인가? 왜 이렇게 설계됐는가?

<details>
<summary>해설 보기</summary>

버그가 아니라 의도된 설계입니다. AES 암호화 시 매번 다른 IV(Initialization Vector)를 랜덤 생성하여 사용합니다. 같은 평문도 IV가 다르면 암호화 결과가 달라집니다. 이는 Rainbow Table 공격을 방어하기 위한 표준 보안 관행입니다. 만약 항상 같은 결과가 나온다면 공격자가 자주 사용되는 비밀번호의 암호화 값을 미리 계산해두고 Git 저장소에서 찾아낼 수 있습니다. IV는 암호화된 값에 포함되어 있어 Config Server는 이를 이용해 정상적으로 복호화할 수 있습니다.

</details>

---

**Q2.** `spring.cloud.config.server.encrypt.enabled=false`로 설정해서 클라이언트 측에서 복호화하는 방식을 도입했다. 그런데 클라이언트의 `/actuator/env`에서 해당 설정 값이 `{cipher}ABC...` 형태로 그대로 보인다. 무슨 문제인가?

<details>
<summary>해설 보기</summary>

`spring.cloud.config.server.encrypt.enabled=false`는 Config Server가 복호화하지 않고 암호화된 값을 그대로 클라이언트에 전달하는 설정입니다. 클라이언트가 복호화하려면 클라이언트 서비스에도 `encrypt.key` 또는 `encrypt.key-store` 설정이 있어야 합니다. 설정이 없으면 `{cipher}` 값이 그대로 `@Value`나 `@ConfigurationProperties`에 바인딩됩니다. 또한 `spring-security-crypto`가 의존성에 포함되어 있어야 클라이언트 측 복호화가 동작합니다. 해결 방법은 클라이언트 서비스의 bootstrap.yml에 `encrypt.key: ${ENCRYPT_KEY}`를 추가하는 것입니다.

</details>

---

**Q3.** Config Server의 `/decrypt` 엔드포인트가 외부에 노출되어 있다면 어떤 보안 위협이 있는가? 어떻게 보호해야 하는가?

<details>
<summary>해설 보기</summary>

`/decrypt` 엔드포인트가 노출되면 공격자가 Git 저장소에서 획득한 `{cipher}` 값을 Config Server에 직접 복호화 요청하여 평문 비밀번호를 얻을 수 있습니다. 보호 방법:

1. Spring Security로 `/encrypt`, `/decrypt` 엔드포인트에 인증 추가
2. Config Server를 내부 네트워크에만 노출 (K8s 내부 Service 또는 VPC 내부)
3. API Gateway에서 `/encrypt`, `/decrypt` 경로 차단 (외부 트래픽 필터링)
4. IP 화이트리스트 적용 (Spring Security `HttpSecurity.requestMatchers().ipAddress()`)

일반적으로 Config Server는 내부 서비스만 접근 가능하도록 구성하고, `/encrypt`, `/decrypt`는 운영자가 개발 환경에서만 사용하도록 제한합니다.

</details>

---

<div align="center">

**[⬅️ 이전: @RefreshScope와 동적 갱신](./04-refresh-scope-dynamic-reload.md)** | **[홈으로 🏠](../README.md)** | **[다음: Config Server High Availability ➡️](./06-config-server-high-availability.md)**

</div>
