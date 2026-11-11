# [2026-11-11] JWT 인증 보안과 Refresh Token Rotation(RTR) & Redis 무효화 전략

> **한 줄 핵심 요약**: Access Token(서명 검증만으로 무상태 처리)과 Refresh Token(상태를 가진 재발급 티켓)을 분리하고, 재발급 때마다 Refresh Token을 1회용으로 교체하는 **Refresh Token Rotation(RTR)** 과 `jti` 불일치 시 **Reuse Detection(연쇄 무효화)** 을 결합하면 JWT의 "즉시 무효화 불가" 한계를 실무 수준으로 메울 수 있다. 남는 구멍은 ①로그아웃된 Access Token의 잔여 수명 ②동시 재발급으로 인한 **RTR 레이스 컨디션** 두 가지이며, ①은 `Redis 블랙리스트(BL:{sha256(AT)})` + 짧은 AT TTL로, ②는 **Lua 기반 원자적 CAS 회전 + 짧은 유예(Grace) 창**으로 방어한다.

---

## 0. 코드 예제 (전체 실행 가능 버전)

### 0-1. 의존성 및 설정

```groovy
// build.gradle
// dependencies {
//     implementation 'org.springframework.boot:spring-boot-starter-web:3.3.5'
//     implementation 'org.springframework.boot:spring-boot-starter-security:3.3.5'
//     implementation 'org.springframework.boot:spring-boot-starter-data-redis:3.3.5'
//     implementation 'io.jsonwebtoken:jjwt-api:0.12.6'
//     runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.6'
//     runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.6'
//     implementation 'org.redisson:redisson-spring-boot-starter:3.35.0'   // 선택: 분산락(2-1-B)
//     testImplementation 'org.springframework.boot:spring-boot-starter-test:3.3.5'
//     testImplementation 'org.testcontainers:junit-jupiter:1.20.3'
// }
```

```yaml
# 파일 경로: src/main/resources/application.yml
auth:
  jwt:
    issuer: agzak-api
    secret-base64: ${JWT_SECRET_BASE64}   # HS256은 32바이트(256bit) 이상 필수
    access-ttl: 15m                       # 짧을수록 블랙리스트·탈취 창이 줄어든다
    refresh-ttl: 14d                      # 세션 슬라이딩 만료(회전 시마다 갱신)
    access-clock-skew: 30s                # AT 검증 허용 오차
    refresh-clock-skew: 0s                # RT는 만료에 관대할 이유가 없다
    grace-window: 10s                     # RTR 동시 요청 유예
    cookie:
      name: rt
      path: /api/auth                     # RT 쿠키는 재발급 경로로만 전송
      secure: true
      same-site: Strict

spring:
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2s
      # 주의: RT는 캐시가 아니라 '상태'다. allkeys-lru로 eviction되면 전 사용자 로그아웃 사고가 난다(2-3).
```

### 0-2. 설정 바인딩과 토큰 값 객체

```java
// 파일 경로: src/main/java/study/auth/jwt/JwtProperties.java
package study.auth.jwt;

import org.springframework.boot.context.properties.ConfigurationProperties;

import java.time.Duration;

@ConfigurationProperties(prefix = "auth.jwt")
public record JwtProperties(
        String issuer,
        String secretBase64,
        Duration accessTtl,
        Duration refreshTtl,
        Duration accessClockSkew,
        Duration refreshClockSkew,
        Duration graceWindow,
        Cookie cookie
) {
    public record Cookie(String name, String path, boolean secure, String sameSite) {
    }
}
```

```java
// 파일 경로: src/main/java/study/auth/jwt/TokenType.java
package study.auth.jwt;

public enum TokenType {
    ACCESS, REFRESH
}
```

```java
// 파일 경로: src/main/java/study/auth/jwt/TokenSpec.java
package study.auth.jwt;

import java.time.Instant;

/**
 * 토큰의 "내용"(서명 전 값). 유예 창에서 이미 발급한 토큰을 동일하게 재서명할 때 쓴다.
 * jti/iat/exp는 비밀이 아니며, 이것만으로는 서명을 만들 수 없다.
 */
public record TokenSpec(String jti, Instant issuedAt, Instant expiresAt) {
}
```

```java
// 파일 경로: src/main/java/study/auth/jwt/IssuedToken.java
package study.auth.jwt;

/**
 * @param value 실제 JWT 문자열
 * @param spec  jti/iat/exp (Redis에는 이 값만 저장한다)
 */
public record IssuedToken(String value, TokenSpec spec) {

    public String jti() {
        return spec.jti();
    }

    public java.time.Instant expiresAt() {
        return spec.expiresAt();
    }
}
```

```java
// 파일 경로: src/main/java/study/auth/jwt/TokenClaims.java
package study.auth.jwt;

import java.time.Instant;

/**
 * @param familyId REFRESH 토큰에만 존재. 세션(토큰 패밀리) 식별자.
 * @param role     ACCESS 토큰에만 존재. RT에는 권한을 넣지 않는다(재발급 시 최신 권한을 조회).
 */
public record TokenClaims(
        String subject,
        String jti,
        String familyId,
        String role,
        Instant issuedAt,
        Instant expiresAt
) {
}
```

```java
// 파일 경로: src/main/java/study/auth/jwt/TokenPair.java
package study.auth.jwt;

public record TokenPair(IssuedToken access, IssuedToken refresh) {
}
```

### 0-3. JwtProvider (JJWT 0.12.x)

```java
// 파일 경로: src/main/java/study/auth/jwt/JwtProvider.java
package study.auth.jwt;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtBuilder;
import io.jsonwebtoken.JwtException;
import io.jsonwebtoken.JwtParser;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.time.Duration;
import java.time.Instant;
import java.util.Date;
import java.util.UUID;

@Component
public class JwtProvider {

    private static final String CLAIM_TYPE = "typ";
    private static final String CLAIM_FAMILY = "fam";
    private static final String CLAIM_ROLE = "role";

    private final SecretKey key;
    private final JwtProperties props;
    /** JwtParser는 immutable + thread-safe → 매 요청 새로 만들지 않는다. */
    private final JwtParser accessParser;
    private final JwtParser refreshParser;

    public JwtProvider(JwtProperties props) {
        this.props = props;
        this.key = Keys.hmacShaKeyFor(Decoders.BASE64.decode(props.secretBase64()));
        this.accessParser = parser(props.accessClockSkew());
        this.refreshParser = parser(props.refreshClockSkew());
    }

    private JwtParser parser(Duration skew) {
        return Jwts.parser()
                .verifyWith(key)                      // 서명 검증(HS256)
                .requireIssuer(props.issuer())        // 다른 서비스가 발급한 토큰 거부
                .clockSkewSeconds(skew.toSeconds())   // 서버 간 시계 오차 허용
                .build();
    }

    public IssuedToken issueAccessToken(String userId, String role) {
        Instant now = Instant.now();
        TokenSpec spec = new TokenSpec(UUID.randomUUID().toString(), now, now.plus(props.accessTtl()));
        return new IssuedToken(build(TokenType.ACCESS, userId, role, null, spec), spec);
    }

    public IssuedToken issueRefreshToken(String userId, String familyId) {
        Instant now = Instant.now();
        TokenSpec spec = new TokenSpec(UUID.randomUUID().toString(), now, now.plus(props.refreshTtl()));
        return new IssuedToken(build(TokenType.REFRESH, userId, null, familyId, spec), spec);
    }

    /**
     * 유예 창 내 중복 요청에 "이미 발급한 것과 같은 문자열"을 돌려주기 위한 재서명.
     * 같은 클레임·같은 삽입 순서·같은 키로 서명하므로 HS256 결과가 동일하다(issue()와 같은 빌더 경로를 쓴다).
     */
    public IssuedToken reissue(TokenType type, String userId, String role, String familyId, TokenSpec spec) {
        return new IssuedToken(build(type, userId, role, familyId, spec), spec);
    }

    private String build(TokenType type, String userId, String role, String familyId, TokenSpec spec) {
        JwtBuilder builder = Jwts.builder()
                .issuer(props.issuer())
                .subject(userId)
                .id(spec.jti())
                .issuedAt(Date.from(spec.issuedAt()))
                .expiration(Date.from(spec.expiresAt()))
                .claim(CLAIM_TYPE, type.name());
        if (role != null) {
            builder.claim(CLAIM_ROLE, role);
        }
        if (familyId != null) {
            builder.claim(CLAIM_FAMILY, familyId);
        }
        return builder.signWith(key, Jwts.SIG.HS256).compact();
    }

    /** Access Token 검증. 실패 시 io.jsonwebtoken.JwtException 계열을 던진다. */
    public TokenClaims parseAccessToken(String token) {
        return toClaims(accessParser.parseSignedClaims(token).getPayload(), TokenType.ACCESS);
    }

    /** Refresh Token 검증. typ 클레임으로 AT/RT 혼용(alg confusion 류 공격)을 차단한다. */
    public TokenClaims parseRefreshToken(String token) {
        return toClaims(refreshParser.parseSignedClaims(token).getPayload(), TokenType.REFRESH);
    }

    private TokenClaims toClaims(Claims claims, TokenType expected) {
        String type = claims.get(CLAIM_TYPE, String.class);
        if (!expected.name().equals(type)) {
            throw new JwtException("unexpected token type: " + type);
        }
        return new TokenClaims(
                claims.getSubject(),
                claims.getId(),
                claims.get(CLAIM_FAMILY, String.class),
                claims.get(CLAIM_ROLE, String.class),
                claims.getIssuedAt().toInstant(),
                claims.getExpiration().toInstant());
    }
}
```

### 0-4. Redis 키 설계

```java
// 파일 경로: src/main/java/study/auth/jwt/AuthRedisKeys.java
package study.auth.jwt;

/**
 * 키 네이밍: auth:{userId}:... — 중괄호는 Redis Cluster의 hash tag다.
 * 한 사용자의 RT 키들이 같은 슬롯에 모여야 Lua 다중 키 연산과 DEL 다건이 클러스터에서도 동작한다.
 *
 * auth:{userId}:rt:family:{familyId}  HASH { rtJti, rtIat, rtExp, ua, createdAt }  TTL=refreshTtl(슬라이딩)
 * auth:{userId}:rt:families          SET  { familyId... }                          TTL=refreshTtl
 * auth:{userId}:rt:grace:{familyId}  HASH { prevJti, succRt*, succAt* }            TTL=graceWindow(10s)
 * auth:bl:{sha256(accessToken)}      STRING "logout"                               TTL=AT 잔여 수명
 */
public final class AuthRedisKeys {

    private AuthRedisKeys() {
    }

    public static String family(String userId, String familyId) {
        return "auth:{" + userId + "}:rt:family:" + familyId;
    }

    public static String familyIndex(String userId) {
        return "auth:{" + userId + "}:rt:families";
    }

    public static String grace(String userId, String familyId) {
        return "auth:{" + userId + "}:rt:grace:" + familyId;
    }

    /** 블랙리스트 키는 토큰 원문이 아니라 SHA-256을 쓴다(원문 저장 시 Redis 덤프/MONITOR에 비밀이 남는다). */
    public static String blacklist(String tokenHash) {
        return "auth:bl:" + tokenHash;
    }
}
```

### 0-5. RefreshTokenRepository (원자적 회전 + 재사용 감지 + 유예)

```java
// 파일 경로: src/main/java/study/auth/jwt/GraceRecord.java
package study.auth.jwt;

/**
 * 유예 창에 남겨두는 "직전 토큰 → 이미 발급한 다음 세대 토큰" 정보.
 * 토큰 문자열이 아니라 jti/iat/exp만 저장한다.
 */
public record GraceRecord(String prevJti, TokenSpec successorAccess, TokenSpec successorRefresh) {
}
```

```java
// 파일 경로: src/main/java/study/auth/jwt/RefreshTokenRepository.java
package study.auth.jwt;

import org.springframework.data.redis.core.RedisScript;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Repository;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;

@Repository
public class RefreshTokenRepository {

    public enum Rotation {
        /** 정상 회전: 제시된 jti가 현재 세대였다. */
        ROTATED,
        /** 유예 창 안의 중복 요청: 재사용 공격이 아니라 "먼저 도착한 요청과 같은 응답"을 재생한다. */
        GRACE_HIT,
        /** 현재 세대도, 유예 창의 직전 세대도 아님 → 탈취 후 재사용으로 판단. */
        REUSE_DETECTED,
        /** 패밀리 자체가 없다(만료·로그아웃·강제 무효화). */
        NO_FAMILY
    }

    /**
     * 회전을 "읽고-비교하고-쓰기" 3단계로 나누면 두 요청이 모두 통과해 서로 덮어쓴다.
     * Lua는 Redis 단일 스레드에서 원자 실행되므로 비교와 쓰기 사이에 다른 명령이 끼어들 수 없다.
     */
    private static final String ROTATE_LUA = """
            local current = redis.call('HGET', KEYS[1], 'rtJti')
            if not current then
                return 'NO_FAMILY'
            end
            if current == ARGV[1] then
                redis.call('HSET', KEYS[1], 'rtJti', ARGV[4], 'rtIat', ARGV[5], 'rtExp', ARGV[6])
                redis.call('EXPIRE', KEYS[1], ARGV[2])
                redis.call('HSET', KEYS[2],
                        'prevJti', ARGV[1],
                        'succRtJti', ARGV[4], 'succRtIat', ARGV[5], 'succRtExp', ARGV[6],
                        'succAtJti', ARGV[7], 'succAtIat', ARGV[8], 'succAtExp', ARGV[9])
                redis.call('EXPIRE', KEYS[2], ARGV[3])
                return 'ROTATED'
            end
            local prev = redis.call('HGET', KEYS[2], 'prevJti')
            if prev and prev == ARGV[1] then
                return 'GRACE_HIT'
            end
            return 'REUSE_DETECTED'
            """;

    private static final RedisScript<String> ROTATE = new DefaultRedisScript<>(ROTATE_LUA, String.class);

    private final StringRedisTemplate redis;
    private final JwtProperties props;

    public RefreshTokenRepository(StringRedisTemplate redis, JwtProperties props) {
        this.redis = redis;
        this.props = props;
    }

    /** 로그인 시 새 세션(패밀리) 등록. */
    public void registerSession(String userId, String familyId, IssuedToken refresh, String userAgent) {
        String familyKey = AuthRedisKeys.family(userId, familyId);
        redis.opsForHash().putAll(familyKey, Map.of(
                "rtJti", refresh.jti(),
                "rtIat", Long.toString(refresh.spec().issuedAt().getEpochSecond()),
                "rtExp", Long.toString(refresh.expiresAt().getEpochSecond()),
                "ua", userAgent == null ? "" : truncate(userAgent, 200),
                "createdAt", Long.toString(Instant.now().getEpochSecond())));
        redis.expire(familyKey, props.refreshTtl());

        String indexKey = AuthRedisKeys.familyIndex(userId);
        redis.opsForSet().add(indexKey, familyId);
        redis.expire(indexKey, props.refreshTtl());
    }

    public Rotation rotate(String userId, String familyId, String presentedJti,
                           IssuedToken newRefresh, IssuedToken newAccess) {
        List<String> keys = List.of(
                AuthRedisKeys.family(userId, familyId),
                AuthRedisKeys.grace(userId, familyId));
        String result = redis.execute(ROTATE, keys,
                presentedJti,
                Long.toString(props.refreshTtl().toSeconds()),
                Long.toString(props.graceWindow().toSeconds()),
                newRefresh.jti(),
                Long.toString(newRefresh.spec().issuedAt().getEpochSecond()),
                Long.toString(newRefresh.expiresAt().getEpochSecond()),
                newAccess.jti(),
                Long.toString(newAccess.spec().issuedAt().getEpochSecond()),
                Long.toString(newAccess.expiresAt().getEpochSecond()));
        return Rotation.valueOf(result == null ? Rotation.NO_FAMILY.name() : result);
    }

    public Optional<GraceRecord> findGrace(String userId, String familyId) {
        Map<Object, Object> hash = redis.opsForHash().entries(AuthRedisKeys.grace(userId, familyId));
        if (hash.isEmpty()) {
            return Optional.empty();
        }
        return Optional.of(new GraceRecord(
                String.valueOf(hash.get("prevJti")),
                new TokenSpec(String.valueOf(hash.get("succAtJti")),
                        Instant.ofEpochSecond(Long.parseLong(String.valueOf(hash.get("succAtIat")))),
                        Instant.ofEpochSecond(Long.parseLong(String.valueOf(hash.get("succAtExp"))))),
                new TokenSpec(String.valueOf(hash.get("succRtJti")),
                        Instant.ofEpochSecond(Long.parseLong(String.valueOf(hash.get("succRtIat")))),
                        Instant.ofEpochSecond(Long.parseLong(String.valueOf(hash.get("succRtExp")))))));
    }

    /** 패밀리의 현재 세대 jti. 유예 재생 직전에 "그 사이 또 회전하지 않았는지" 확인한다. */
    public boolean isCurrentGeneration(String userId, String familyId, String expectedJti) {
        Object current = redis.opsForHash().get(AuthRedisKeys.family(userId, familyId), "rtJti");
        return expectedJti.equals(current);
    }

    /** 특정 세션(기기)만 무효화. RT 재사용 감지 시 연쇄 무효화에 쓴다. */
    public void revokeFamily(String userId, String familyId) {
        redis.delete(List.of(
                AuthRedisKeys.family(userId, familyId),
                AuthRedisKeys.grace(userId, familyId)));
        redis.opsForSet().remove(AuthRedisKeys.familyIndex(userId), familyId);
    }

    /** 모든 기기 로그아웃(비밀번호 변경·탈취 의심 시). */
    public void revokeAllSessions(String userId) {
        Set<String> families = redis.opsForSet().members(AuthRedisKeys.familyIndex(userId));
        List<String> keys = new ArrayList<>();
        if (families != null) {
            for (String familyId : families) {
                keys.add(AuthRedisKeys.family(userId, familyId));
                keys.add(AuthRedisKeys.grace(userId, familyId));
            }
        }
        keys.add(AuthRedisKeys.familyIndex(userId));
        redis.delete(keys);   // hash tag 덕분에 클러스터에서도 단일 슬롯 → 한 번에 삭제
    }

    public Duration refreshTtl() {
        return props.refreshTtl();
    }

    private static String truncate(String value, int max) {
        return value.length() <= max ? value : value.substring(0, max);
    }
}
```

### 0-6. Access Token 블랙리스트

```java
// 파일 경로: src/main/java/study/auth/jwt/AccessTokenBlacklist.java
package study.auth.jwt;

import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.time.Duration;
import java.time.Instant;
import java.util.HexFormat;

@Component
public class AccessTokenBlacklist {

    private static final String LOGOUT = "logout";

    private final StringRedisTemplate redis;

    public AccessTokenBlacklist(StringRedisTemplate redis) {
        this.redis = redis;
    }

    /**
     * 로그아웃된 AT를 "남은 유효시간"만큼만 저장한다. AT가 만료되면 서명 검증에서 이미 거부되므로
     * 그 뒤로는 저장할 필요가 없다 → 블랙리스트 크기가 (로그아웃 수 × AT TTL)로 자연히 제한된다.
     */
    public void revoke(String accessToken, Instant expiresAt, Duration clockSkew) {
        long ttlSeconds = Duration.between(Instant.now(), expiresAt).plus(clockSkew).toSeconds();
        if (ttlSeconds <= 0) {
            return;   // 이미 만료: 저장해도 검증 단계에서 걸리지 않는다
        }
        redis.opsForValue().set(AuthRedisKeys.blacklist(hash(accessToken)), LOGOUT, Duration.ofSeconds(ttlSeconds));
    }

    public boolean isRevoked(String accessToken) {
        return Boolean.TRUE.equals(redis.hasKey(AuthRedisKeys.blacklist(hash(accessToken))));
    }

    public static String hash(String token) {
        try {
            byte[] digest = MessageDigest.getInstance("SHA-256").digest(token.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(digest);
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException("SHA-256 not available", e);
        }
    }
}
```

### 0-7. AuthService (로그인 · 재발급 · 로그아웃)

```java
// 파일 경로: src/main/java/study/auth/UserAccount.java
package study.auth;

/** 조회 전용. role은 "ROLE_USER"처럼 접두어를 포함한 문자열로 둔다. */
public record UserAccount(String id, String loginId, String passwordHash, String role) {
}
```

```java
// 파일 경로: src/main/java/study/auth/UserAccountRepository.java
package study.auth;

import java.util.Optional;

/** JPA 구현체(UserAccountJpaRepository)가 있다고 가정한다. */
public interface UserAccountRepository {

    Optional<UserAccount> findByLoginId(String loginId);

    Optional<String> findRole(String userId);
}
```

```java
// 파일 경로: src/main/java/study/auth/AuthError.java
package study.auth;

import org.springframework.http.HttpStatus;

public enum AuthError {
    INVALID_CREDENTIALS(HttpStatus.UNAUTHORIZED, "아이디 또는 비밀번호가 올바르지 않습니다."),
    REFRESH_MISSING(HttpStatus.UNAUTHORIZED, "Refresh Token 쿠키가 없습니다."),
    REFRESH_INVALID(HttpStatus.UNAUTHORIZED, "Refresh Token이 유효하지 않습니다. 다시 로그인해 주세요."),
    REFRESH_REUSED(HttpStatus.UNAUTHORIZED, "이미 사용된 Refresh Token입니다. 보안을 위해 세션을 종료했습니다."),
    USER_GONE(HttpStatus.UNAUTHORIZED, "사용자를 찾을 수 없습니다.");

    private final HttpStatus status;
    private final String message;

    AuthError(HttpStatus status, String message) {
        this.status = status;
        this.message = message;
    }

    public HttpStatus status() {
        return status;
    }

    public String message() {
        return message;
    }
}
```

```java
// 파일 경로: src/main/java/study/auth/AuthException.java
package study.auth;

public class AuthException extends RuntimeException {

    private final AuthError error;

    public AuthException(AuthError error) {
        super(error.message());
        this.error = error;
    }

    public AuthError error() {
        return error;
    }
}
```

```java
// 파일 경로: src/main/java/study/auth/AuthService.java
package study.auth;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import study.auth.dto.LoginRequest;
import study.auth.jwt.AccessTokenBlacklist;
import study.auth.jwt.GraceRecord;
import study.auth.jwt.IssuedToken;
import study.auth.jwt.JwtProperties;
import study.auth.jwt.JwtProvider;
import study.auth.jwt.RefreshTokenRepository;
import study.auth.jwt.TokenClaims;
import study.auth.jwt.TokenPair;
import study.auth.jwt.TokenType;

import java.util.UUID;

import io.jsonwebtoken.JwtException;   // 표준 라이브러리 이후 배치(가독성 목적)

@Service
public class AuthService {

    private static final Logger log = LoggerFactory.getLogger(AuthService.class);

    private final JwtProvider jwtProvider;
    private final RefreshTokenRepository refreshTokenRepository;
    private final AccessTokenBlacklist blacklist;
    private final UserAccountRepository userAccountRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtProperties props;

    public AuthService(JwtProvider jwtProvider,
                       RefreshTokenRepository refreshTokenRepository,
                       AccessTokenBlacklist blacklist,
                       UserAccountRepository userAccountRepository,
                       PasswordEncoder passwordEncoder,
                       JwtProperties props) {
        this.jwtProvider = jwtProvider;
        this.refreshTokenRepository = refreshTokenRepository;
        this.blacklist = blacklist;
        this.userAccountRepository = userAccountRepository;
        this.passwordEncoder = passwordEncoder;
        this.props = props;
    }

    /** 로그인: 기기(세션)마다 독립된 패밀리를 발급한다 → 한 기기 로그아웃이 다른 기기를 끊지 않는다. */
    public TokenPair login(LoginRequest request, String userAgent) {
        UserAccount user = userAccountRepository.findByLoginId(request.loginId())
                .filter(found -> passwordEncoder.matches(request.password(), found.passwordHash()))
                .orElseThrow(() -> new AuthException(AuthError.INVALID_CREDENTIALS));

        String familyId = UUID.randomUUID().toString();
        IssuedToken access = jwtProvider.issueAccessToken(user.id(), user.role());
        IssuedToken refresh = jwtProvider.issueRefreshToken(user.id(), familyId);
        refreshTokenRepository.registerSession(user.id(), familyId, refresh, userAgent);
        return new TokenPair(access, refresh);
    }

    /**
     * 재발급 + 회전.
     * 순서가 중요하다: ① 서명/만료 검증(무상태) → ② Redis 원자 회전(상태) → ③ 결과별 분기.
     */
    public TokenPair refresh(String refreshTokenValue, String userAgent) {
        TokenClaims claims;
        try {
            claims = jwtProvider.parseRefreshToken(refreshTokenValue);
        } catch (JwtException | IllegalArgumentException e) {
            throw new AuthException(AuthError.REFRESH_INVALID);
        }

        String userId = claims.subject();
        String familyId = claims.familyId();
        if (familyId == null || familyId.isBlank()) {
            throw new AuthException(AuthError.REFRESH_INVALID);
        }
        // RT에 권한을 넣지 않았으므로 최신 권한을 조회한다(권한 회수·승격이 다음 AT부터 반영).
        String role = userAccountRepository.findRole(userId)
                .orElseThrow(() -> new AuthException(AuthError.USER_GONE));

        IssuedToken newAccess = jwtProvider.issueAccessToken(userId, role);
        IssuedToken newRefresh = jwtProvider.issueRefreshToken(userId, familyId);

        RefreshTokenRepository.Rotation rotation =
                refreshTokenRepository.rotate(userId, familyId, claims.jti(), newRefresh, newAccess);

        return switch (rotation) {
            case ROTATED -> new TokenPair(newAccess, newRefresh);
            case GRACE_HIT -> replaySuccessor(userId, role, familyId, claims.jti());
            case REUSE_DETECTED -> {
                // 탈취된 RT가 재사용된 것으로 판단: 이 세션의 모든 토큰을 즉시 무효화(연쇄 무효화).
                refreshTokenRepository.revokeFamily(userId, familyId);
                log.warn("refresh token reuse detected. userId={}, familyId={}, presentedJti={}, ua={}",
                        userId, familyId, claims.jti(), userAgent);
                throw new AuthException(AuthError.REFRESH_REUSED);
            }
            case NO_FAMILY -> throw new AuthException(AuthError.REFRESH_INVALID);
        };
    }

    /** 유예 창: 프론트가 동시에 재발급을 두 번 보낸 경우, 두 번째에게 "첫 번째와 같은" 토큰 쌍을 재생한다. */
    private TokenPair replaySuccessor(String userId, String role, String familyId, String presentedJti) {
        GraceRecord grace = refreshTokenRepository.findGrace(userId, familyId)
                .orElseThrow(() -> new AuthException(AuthError.REFRESH_INVALID));

        boolean sameGeneration = grace.prevJti().equals(presentedJti)
                && refreshTokenRepository.isCurrentGeneration(userId, familyId, grace.successorRefresh().jti());
        if (!sameGeneration) {
            // 유예 창 안에서 이미 다음 세대가 또 회전했다 = 같은 RT를 서로 다른 주체가 쓰고 있다.
            refreshTokenRepository.revokeFamily(userId, familyId);
            log.warn("refresh token reuse detected inside grace window. userId={}, familyId={}", userId, familyId);
            throw new AuthException(AuthError.REFRESH_REUSED);
        }

        IssuedToken access = jwtProvider.reissue(TokenType.ACCESS, userId, role, null, grace.successorAccess());
        IssuedToken refresh = jwtProvider.reissue(TokenType.REFRESH, userId, null, familyId, grace.successorRefresh());
        return new TokenPair(access, refresh);
    }

    /**
     * 로그아웃: AT는 블랙리스트(남은 TTL), RT는 패밀리 삭제.
     * AT 블랙리스트가 없으면 로그아웃 후에도 AT 만료(최대 access-ttl)까지 API 호출이 가능하다.
     */
    public void logout(String accessToken, String refreshTokenValue) {
        if (accessToken != null) {
            try {
                TokenClaims claims = jwtProvider.parseAccessToken(accessToken);
                blacklist.revoke(accessToken, claims.expiresAt(), props.accessClockSkew());
            } catch (JwtException | IllegalArgumentException ignored) {
                // 이미 만료·위조된 AT: 블랙리스트에 넣어도 의미가 없다(검증 단계에서 거부된다).
            }
        }
        if (refreshTokenValue != null) {
            try {
                TokenClaims claims = jwtProvider.parseRefreshToken(refreshTokenValue);
                refreshTokenRepository.revokeFamily(claims.subject(), claims.familyId());
            } catch (JwtException | IllegalArgumentException ignored) {
                // 쿠키가 이미 무효: 삭제만 하고 정상 응답(멱등 로그아웃).
            }
        }
    }

    public void logoutAll(String accessToken) {
        if (accessToken == null) {
            return;
        }
        TokenClaims claims = jwtProvider.parseAccessToken(accessToken);
        blacklist.revoke(accessToken, claims.expiresAt(), props.accessClockSkew());
        refreshTokenRepository.revokeAllSessions(claims.subject());
    }
}
```

### 0-8. Controller (HttpOnly 쿠키로 RT 전달)

```java
// 파일 경로: src/main/java/study/auth/dto/LoginRequest.java
package study.auth.dto;

import jakarta.validation.constraints.NotBlank;

public record LoginRequest(@NotBlank String loginId, @NotBlank String password) {
}
```

```java
// 파일 경로: src/main/java/study/auth/dto/TokenResponse.java
package study.auth.dto;

import java.time.Instant;

/** AT는 응답 본문으로(메모리 보관), RT는 Set-Cookie로만 내려준다. */
public record TokenResponse(String accessToken, String tokenType, Instant expiresAt) {
}
```

```java
// 파일 경로: src/main/java/study/auth/AuthController.java
package study.auth;

import jakarta.validation.Valid;
import org.springframework.http.HttpHeaders;
import org.springframework.http.ResponseCookie;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.CookieValue;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestHeader;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import study.auth.dto.LoginRequest;
import study.auth.dto.TokenResponse;
import study.auth.jwt.JwtProperties;
import study.auth.jwt.TokenPair;

import java.time.Duration;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    /** application.yml의 auth.jwt.cookie.name과 반드시 같아야 한다. */
    private static final String REFRESH_COOKIE = "rt";

    private final AuthService authService;
    private final JwtProperties props;

    public AuthController(AuthService authService, JwtProperties props) {
        this.authService = authService;
        this.props = props;
    }

    @PostMapping("/login")
    public ResponseEntity<TokenResponse> login(@RequestBody @Valid LoginRequest request,
                                               @RequestHeader(value = HttpHeaders.USER_AGENT, required = false) String userAgent) {
        TokenPair pair = authService.login(request, userAgent);
        return ResponseEntity.ok()
                .header(HttpHeaders.SET_COOKIE, refreshCookie(pair.refresh().value(), props.refreshTtl()).toString())
                .body(toBody(pair));
    }

    @PostMapping("/refresh")
    public ResponseEntity<TokenResponse> refresh(
            @CookieValue(name = REFRESH_COOKIE, required = false) String refreshToken,
            @RequestHeader(value = HttpHeaders.USER_AGENT, required = false) String userAgent) {
        if (refreshToken == null || refreshToken.isBlank()) {
            throw new AuthException(AuthError.REFRESH_MISSING);
        }
        TokenPair pair = authService.refresh(refreshToken, userAgent);
        return ResponseEntity.ok()
                // 슬라이딩 만료: 회전할 때마다 쿠키 Max-Age도 새 RT의 TTL로 갱신한다.
                .header(HttpHeaders.SET_COOKIE, refreshCookie(pair.refresh().value(), props.refreshTtl()).toString())
                .body(toBody(pair));
    }

    @PostMapping("/logout")
    public ResponseEntity<Void> logout(
            @RequestHeader(value = HttpHeaders.AUTHORIZATION, required = false) String authorization,
            @CookieValue(name = REFRESH_COOKIE, required = false) String refreshToken) {
        authService.logout(bearer(authorization), refreshToken);
        return ResponseEntity.noContent()
                .header(HttpHeaders.SET_COOKIE, refreshCookie("", Duration.ZERO).toString())   // 쿠키 삭제
                .build();
    }

    private TokenResponse toBody(TokenPair pair) {
        return new TokenResponse(pair.access().value(), "Bearer", pair.access().expiresAt());
    }

    private ResponseCookie refreshCookie(String value, Duration maxAge) {
        return ResponseCookie.from(props.cookie().name(), value)
                .httpOnly(true)                     // JS에서 읽을 수 없게 → XSS로 RT 탈취 차단
                .secure(props.cookie().secure())    // HTTPS에서만 전송
                .sameSite(props.cookie().sameSite())// CSRF 완화(Strict 권장, UX상 Lax도 허용)
                .path(props.cookie().path())        // /api/auth 에서만 전송
                .maxAge(maxAge)
                .build();
    }

    private static String bearer(String authorizationHeader) {
        if (authorizationHeader == null || !authorizationHeader.startsWith("Bearer ")) {
            return null;
        }
        return authorizationHeader.substring(7);
    }
}
```

### 0-9. 인증 필터 · 블랙리스트 필터 · 예외 응답

```java
// 파일 경로: src/main/java/study/auth/jwt/JwtAuthenticationFilter.java
package study.auth.jwt;

import io.jsonwebtoken.JwtException;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpHeaders;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

/** AT 서명만으로 인증을 완성한다(Redis 조회 없음). 실패해도 인증을 비운 채 통과시키고, 인가 단계에서 401/403이 난다. */
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtProvider jwtProvider;

    public JwtAuthenticationFilter(JwtProvider jwtProvider) {
        this.jwtProvider = jwtProvider;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        String header = request.getHeader(HttpHeaders.AUTHORIZATION);
        if (header != null && header.startsWith("Bearer ")) {
            try {
                TokenClaims claims = jwtProvider.parseAccessToken(header.substring(7));
                var authentication = new UsernamePasswordAuthenticationToken(
                        claims.subject(), null, List.of(new SimpleGrantedAuthority(claims.role())));
                SecurityContextHolder.getContext().setAuthentication(authentication);
            } catch (JwtException | IllegalArgumentException e) {
                SecurityContextHolder.clearContext();   // 만료·위조: 비로그인으로 취급
            }
        }
        chain.doFilter(request, response);
    }
}
```

```java
// 파일 경로: src/main/java/study/auth/jwt/AccessTokenBlacklistFilter.java
package study.auth.jwt;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

/**
 * 서명 검증은 이미 통과한 AT라도, 로그아웃된 AT면 거부해야 한다.
 * 여기서 Redis 1회 조회(RTT)가 추가되므로 ① AT TTL을 짧게 ② 민감 API에만 적용 ③ 결과를 짧게 로컬 캐시
 * 중 하나로 비용을 조절한다(2-3).
 */
@Component
public class AccessTokenBlacklistFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(AccessTokenBlacklistFilter.class);

    private final AccessTokenBlacklist blacklist;

    public AccessTokenBlacklistFilter(AccessTokenBlacklist blacklist) {
        this.blacklist = blacklist;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        String header = request.getHeader(HttpHeaders.AUTHORIZATION);
        String token = (header != null && header.startsWith("Bearer ")) ? header.substring(7) : null;

        if (token != null) {
            boolean revoked;
            try {
                revoked = blacklist.isRevoked(token);
            } catch (RuntimeException e) {
                // Redis 장애: AT 서명은 이미 유효하므로 fail-open(가용성 우선)으로 통과시키고 경보를 남긴다.
                log.error("blacklist lookup failed. fail-open applied.", e);
                revoked = false;
            }
            if (revoked) {
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                response.setContentType(MediaType.APPLICATION_JSON_VALUE);
                response.setCharacterEncoding("UTF-8");
                response.getWriter().write("{\"code\":\"ACCESS_TOKEN_REVOKED\",\"message\":\"로그아웃된 토큰입니다.\"}");
                return;
            }
        }
        chain.doFilter(request, response);
    }
}
```

```java
// 파일 경로: src/main/java/study/auth/AuthExceptionHandler.java
package study.auth;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.Map;

@RestControllerAdvice
public class AuthExceptionHandler {

    @ExceptionHandler(AuthException.class)
    public ResponseEntity<Map<String, String>> handle(AuthException e) {
        return ResponseEntity.status(e.error().status())
                .body(Map.of("code", e.error().name(), "message", e.error().message()));
    }
}
```

```java
// 파일 경로: src/main/java/study/auth/SecurityConfig.java
package study.auth;

import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import study.auth.jwt.AccessTokenBlacklistFilter;
import study.auth.jwt.JwtAuthenticationFilter;
import study.auth.jwt.JwtProperties;

@Configuration
@EnableWebSecurity
@EnableConfigurationProperties(JwtProperties.class)
public class SecurityConfig {

    // PasswordEncoder(BCryptPasswordEncoder) 빈은 별도 구성 클래스에 두고 AuthService에 주입한다.

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http,
                                           JwtAuthenticationFilter jwtAuthenticationFilter,
                                           AccessTokenBlacklistFilter blacklistFilter) throws Exception {
        http
                // 쿠키 기반 refresh를 쓰므로 CSRF 토큰은 끄되, SameSite=Strict + Origin/Referer 검증으로 보완한다(2-4).
                .csrf(csrf -> csrf.disable())
                .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(a -> a
                        .requestMatchers("/api/auth/login", "/api/auth/refresh").permitAll()
                        .anyRequest().authenticated())
                // 순서: AT 인증 → 블랙리스트 확인 → 인가
                .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
                .addFilterAfter(blacklistFilter, JwtAuthenticationFilter.class);
        return http.build();
    }
}
```

### 0-10. Testcontainers 테스트 (회전 · 재사용 감지 · 유예 재생)

```java
// 파일 경로: src/test/java/study/auth/jwt/RefreshTokenRotationTest.java
package study.auth.jwt;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.data.redis.connection.RedisCallback;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.util.Base64;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
class RefreshTokenRotationTest {

    @Container
    static final GenericContainer<?> REDIS =
            new GenericContainer<>(DockerImageName.parse("redis:7.2-alpine")).withExposedPorts(6379);

    private StringRedisTemplate redis;
    private JwtProvider jwtProvider;
    private RefreshTokenRepository repository;

    @BeforeEach
    void setUp() {
        LettuceConnectionFactory factory =
                new LettuceConnectionFactory(REDIS.getHost(), REDIS.getMappedPort(6379));
        factory.afterPropertiesSet();
        redis = new StringRedisTemplate(factory);

        JwtProperties props = new JwtProperties(
                "test-issuer",
                Base64.getEncoder().encodeToString("0123456789abcdef0123456789abcdef".getBytes(StandardCharsets.UTF_8)),
                Duration.ofMinutes(15),
                Duration.ofDays(14),
                Duration.ofSeconds(30),
                Duration.ZERO,
                Duration.ofSeconds(10),
                new JwtProperties.Cookie("rt", "/api/auth", true, "Strict"));

        jwtProvider = new JwtProvider(props);
        repository = new RefreshTokenRepository(redis, props);
    }

    @AfterEach
    void tearDown() {
        redis.execute((RedisCallback<Object>) connection -> {
            connection.serverCommands().flushAll();
            return null;
        });
    }

    @Test
    void 정상_회전시_세대가_바뀐다() {
        String userId = "u1";
        String familyId = "f1";
        IssuedToken first = jwtProvider.issueRefreshToken(userId, familyId);
        repository.registerSession(userId, familyId, first, "JUnit");

        IssuedToken next = jwtProvider.issueRefreshToken(userId, familyId);
        IssuedToken nextAccess = jwtProvider.issueAccessToken(userId, "ROLE_USER");

        RefreshTokenRepository.Rotation result =
                repository.rotate(userId, familyId, first.jti(), next, nextAccess);

        assertThat(result).isEqualTo(RefreshTokenRepository.Rotation.ROTATED);
        assertThat(repository.isCurrentGeneration(userId, familyId, next.jti())).isTrue();
        assertThat(repository.isCurrentGeneration(userId, familyId, first.jti())).isFalse();
    }

    @Test
    void 유예_창_안의_중복_요청은_같은_다음_세대를_재생한다() {
        String userId = "u2";
        String familyId = "f2";
        IssuedToken first = jwtProvider.issueRefreshToken(userId, familyId);
        repository.registerSession(userId, familyId, first, "JUnit");

        IssuedToken next = jwtProvider.issueRefreshToken(userId, familyId);
        IssuedToken nextAccess = jwtProvider.issueAccessToken(userId, "ROLE_USER");
        repository.rotate(userId, familyId, first.jti(), next, nextAccess);

        // 같은 RT로 동시에 한 번 더 도착
        IssuedToken competitor = jwtProvider.issueRefreshToken(userId, familyId);
        IssuedToken competitorAccess = jwtProvider.issueAccessToken(userId, "ROLE_USER");
        RefreshTokenRepository.Rotation replay =
                repository.rotate(userId, familyId, first.jti(), competitor, competitorAccess);

        assertThat(replay).isEqualTo(RefreshTokenRepository.Rotation.GRACE_HIT);
        GraceRecord grace = repository.findGrace(userId, familyId).orElseThrow();
        assertThat(grace.successorRefresh().jti()).isEqualTo(next.jti());   // 첫 응답과 동일한 세대
        assertThat(repository.isCurrentGeneration(userId, familyId, next.jti())).isTrue();
    }

    @Test
    void 유예_창이_지난_뒤의_재사용은_감지된다() {
        String userId = "u3";
        String familyId = "f3";
        IssuedToken first = jwtProvider.issueRefreshToken(userId, familyId);
        repository.registerSession(userId, familyId, first, "JUnit");

        IssuedToken next = jwtProvider.issueRefreshToken(userId, familyId);
        repository.rotate(userId, familyId, first.jti(), next, jwtProvider.issueAccessToken(userId, "ROLE_USER"));
        redis.delete(AuthRedisKeys.grace(userId, familyId));   // 유예 창 만료를 즉시 재현

        RefreshTokenRepository.Rotation reuse = repository.rotate(userId, familyId, first.jti(),
                jwtProvider.issueRefreshToken(userId, familyId),
                jwtProvider.issueAccessToken(userId, "ROLE_USER"));

        assertThat(reuse).isEqualTo(RefreshTokenRepository.Rotation.REUSE_DETECTED);

        repository.revokeFamily(userId, familyId);
        assertThat(repository.isCurrentGeneration(userId, familyId, next.jti())).isFalse();
    }
}
```

---

## 1. 개념 및 동작 원리

### 1-1. 왜 Access Token과 Refresh Token을 나누는가

| | **Access Token** | **Refresh Token** |
|---|---|---|
| 수명 | 짧다(5~30분) | 길다(7~30일, 슬라이딩) |
| 전달 위치 | `Authorization: Bearer` 헤더(JS 메모리 보관) | `HttpOnly` 쿠키(스크립트 접근 불가) |
| 서버 검증 | **서명 검증만**(Redis 조회 없음, 무상태) | 서명 + **Redis 저장값 대조**(상태 있음) |
| 탈취 시 피해 | 짧은 시간 동안 API 호출 | 지속적인 재발급(→ 회전으로 차단) |
| 무효화 | 블랙리스트 또는 TTL 만료 대기 | Redis 삭제로 **즉시** 무효화 |

- **핵심 트레이드오프**: AT를 무상태로 두면 모든 API가 Redis/DB 없이 검증되어 확장이 쉽다. 대신 **발급 후 취소가 불가능**하다. RT는 반대로 상태를 갖지만, 발급 빈도가 낮아(로그인·재발급) 상태 저장 비용이 작다. **자주 쓰는 것은 무상태, 드물게 쓰는 것은 상태**로 두는 분업이다.
- **AT 수명을 짧게 잡는 이유**: AT 탈취의 유효 창을 줄인다. 하지만 너무 짧으면 재발급 트래픽이 늘고 RTR 레이스가 잦아진다. 실무 기본값은 15~30분, 민감 서비스는 5분 + 재발급 요청 집중 완화(클라이언트 single-flight).

### 1-2. Refresh Token Rotation (RTR) — 1회용 티켓

```
[로그인]  AT(15m) + RT#1 ── Redis: auth:{u}:rt:family:{f} = { rtJti: #1 }
   │
   ├─ AT 만료 → POST /api/auth/refresh (쿠키: RT#1)
   │      Redis 현재 jti(#1) == 제시된 jti(#1)  ✅
   │      → RT#2 발급, Redis jti = #2, 유예 창에 {prev: #1, succ: #2} 10초 보관
   │      → 응답: AT' + Set-Cookie RT#2
   │
   ├─ (정상 클라이언트) 다음 갱신은 RT#2로 → 계속 성공
   │
   └─ (공격자) 탈취한 RT#1로 갱신 시도
          Redis 현재 jti(#2) ≠ 제시된 jti(#1), 유예 창에도 없음
          → REUSE_DETECTED → 패밀리 전체 삭제 → 정상 사용자도 재로그인, 공격자 토큰도 무효
```

- **회전의 본질**: RT는 "값"이 아니라 **세대(generation)를 가진 일회용 티켓**이다. 매 사용마다 다음 세대로 교체되므로, 같은 티켓이 두 번 나타나면 **둘 중 하나는 반드시 복제본**이다. 이것이 재사용 감지(Reuse Detection)의 전부다.
- **연쇄 무효화(family revocation)**: 재사용을 감지하면 그 RT 하나만이 아니라 **패밀리(세션) 전체**를 지운다. 탈취자가 먼저 갱신해 새 RT를 가져갔다면 정상 사용자의 다음 요청이 재사용으로 감지되는데, 이때도 패밀리를 지워 **양쪽 모두 로그아웃**시킨다. "누가 진짜인지" 판별하는 것보다 **둘 다 끊고 재로그인**시키는 편이 안전하다.
- **RT에 `jti`를 넣는 이유**: 토큰 문자열 전체를 저장·비교하면 Redis에 비밀값이 남고 키가 커진다. `jti`(랜덤 UUID)는 토큰의 **세대 식별자** 역할만 하며, 그 자체로는 서명을 만들 수 없다.

### 1-3. 무효화 전략: 블랙리스트 vs 화이트리스트

| | **블랙리스트(deny-list)** | **화이트리스트(allow-list)** |
|---|---|---|
| 저장 대상 | 로그아웃된 AT의 해시 + 남은 TTL | 발급된 AT의 jti(또는 세션 ID) |
| 요청당 비용 | Redis 조회 1회(키 없으면 통과) | Redis 조회 1회(키 없으면 거부) |
| 저장량 | **로그아웃 수 × AT TTL**로 자연 감소 | **활성 사용자 수 × AT TTL** |
| 장점 | Redis 장애 시 fail-open 가능(가용성), 저장량 작음 | Redis 장애 시 fail-close(보안), 즉시 전체 차단 가능 |
| 단점 | 로그아웃 전 탈취 토큰은 여전히 유효 | 저장량·조회 부하 큼, Redis가 SPOF |

- **실무 기본은 블랙리스트 + 짧은 AT TTL**이다. AT가 15분이면 "로그아웃 후 최대 15분"이 아니라 **로그아웃 시점 이후의 AT는 즉시 차단**되고, 저장되는 키는 15분 뒤 스스로 사라진다.
- **화이트리스트가 필요한 순간**: 강제 로그아웃·계정 정지·비밀번호 변경 시 "모든 기기의 AT를 지금 당장" 무효화해야 할 때. 이때는 AT에 `sessionId`(패밀리 ID)를 넣고 **`auth:{u}:rt:family:{f}` 존재 여부로 AT까지 검증**하는 하이브리드가 깔끔하다(패밀리 삭제 = 그 세션의 AT·RT 동시 무효).

### 1-4. 토큰 보관 위치: LocalStorage vs Cookie

| 보관 위치 | XSS | CSRF | 비고 |
|---|---|---|---|
| `localStorage` | ❌ 스크립트로 읽힘 | ✅ 자동 전송 없음 | 편하지만 XSS 1건이면 RT 영구 탈취 |
| `sessionStorage` | ❌ 동일 | ✅ | 탭 종료 시 사라짐 |
| **HttpOnly Cookie** | ✅ JS 접근 불가 | ❌ 자동 전송됨 | `SameSite` + CSRF 토큰으로 보완 |
| 메모리 변수(AT) | △ 새로고침 시 소실 | ✅ | AT는 메모리, RT는 쿠키 조합이 표준 |

- **결론**: **AT는 JS 메모리, RT는 `HttpOnly; Secure; SameSite=Strict` 쿠키**.
- **`SameSite` 선택**: `Strict`는 외부 사이트에서 우리 사이트로 이동할 때도 쿠키가 빠져 "로그인 화면으로 튕기는" UX가 생긴다. 이때는 RT 쿠키를 `Path=/api/auth`로 좁혀 **재발급 경로에서만** 전송되게 하고(외부 링크 진입 시 최초 페이지 요청에는 쿠키가 필요 없다), `Lax`를 쓰는 편이 실용적이다.
- **`__Host-` 접두어**: `__Host-rt`로 이름을 지으면 ①Secure ②Path=/ ③Domain 없음이 강제되어 서브도메인 하위에서 쿠키를 덮어쓰는 공격(subdomain cookie tossing)을 막는다. 단 **Path=/가 강제**되므로 위의 `Path=/api/auth` 최소화와는 함께 쓸 수 없다 → 둘 중 하나를 정책으로 선택한다.

### 1-5. 상태를 어디에 둘 것인가: Redis vs DB

| | **Redis** | **RDB(토큰 테이블)** |
|---|---|---|
| 지연 | 서브밀리초 | 수 ms + 커넥션 |
| TTL | 키 단위 자동 만료 | 별도 삭제 배치 필요 |
| 원자성 | Lua로 CAS 가능 | 트랜잭션·행 락(UNIQUE(jti)) |
| 유실 위험 | eviction·failover 시 소실 → 전 사용자 재로그인 | 내구성 높음 |
| 적합 | 대부분의 서비스, 세션성 데이터 | 금융권·감사 요건, 감사 로그 필수 |

- **Redis 유실 = 전원 재로그인**. 이는 "가용성 사고"이지 "보안 사고"는 아니며, 대부분 허용 가능한 실패 모드다. 반대로 **DB를 쓰면 재발급마다 DB 왕복**이 생기고, 트래픽이 몰리면 인증이 병목이 된다.
- 실무 절충: **RT는 Redis, 세션 이력(로그인 기록·기기 목록)은 DB**. Redis는 "현재 유효한 세대"만, DB는 "감사·표시용".

---

## 2. 실무 적용 및 주의사항 (Edge Cases & 동시성)

### 2-1. RTR 레이스 컨디션 — 가장 많이 터지는 실사고

- **시나리오**: 화면 진입 시 API 3개가 동시에 401을 받고, 각자 `/refresh`를 호출한다. RT#1이 3번 도착.
  - ①번이 먼저 회전 → Redis jti = #2.
  - ②③번은 jti가 #1이라 불일치 → **재사용 공격으로 오인 → 패밀리 삭제 → 정상 사용자 로그아웃**.
  - 사용자 입장에서는 "가만히 있었는데 로그아웃됨"이고, 로그에는 "보안 사고"가 찍힌다.
- **원인은 두 층으로 나뉜다**:
  1. **클라이언트**: 재발급 요청이 직렬화되지 않음(각 API 클라이언트가 개별적으로 refresh).
  2. **서버**: 회전이 원자적이지 않으면(읽고→비교→쓰기) 두 요청이 모두 "일치" 판정을 통과해 서로 덮어쓴다. 이 경우 첫 응답의 RT는 즉시 무효가 되어 **첫 요청자도 다음에 실패**한다.
- **해결 A — 클라이언트 single-flight(가장 먼저 할 일)**: 앱 전체에 **하나의 refresh Promise**를 공유한다. 401을 받은 요청들은 진행 중인 refresh를 기다렸다가 새 AT를 받는다.
  ```javascript
  // 파일 경로: src/api/authClient.js
  let refreshing = null;

  export async function requestWithAuth(path, options = {}) {
    let res = await fetch(path, { ...options, credentials: 'include' });
    if (res.status !== 401) return res;

    refreshing ??= fetch('/api/auth/refresh', { method: 'POST', credentials: 'include' })
      .then(r => (r.ok ? r.json() : Promise.reject(new Error('refresh failed'))))
      .finally(() => { refreshing = null; });   // 성공·실패 모두 다음 갱신을 위해 초기화

    const tokens = await refreshing;             // 동시에 401을 맞은 요청들은 이 Promise를 공유
    return fetch(path, {
      ...options,
      headers: { ...options.headers, Authorization: `Bearer ${tokens.accessToken}` },
    });
  }
  ```
- **해결 B — 서버 유예 창(Grace Period)**: 회전 직후 `graceWindow`(예: 10초) 동안 **직전 세대 jti를 "정상"으로 인정**하고, 그 요청에는 **이미 발급한 다음 세대 토큰을 그대로 재생**한다. 재생을 위해 유예 창에 저장하는 것은 토큰 문자열이 아니라 `jti/iat/exp`이며, 같은 값으로 다시 서명하면 동일한 JWT가 나온다(HS256은 결정적).
  - **왜 "새로 발급"이 아니라 "재생"인가**: 새로 발급하면 첫 요청이 받은 RT가 곧바로 무효가 되어, 응답 순서가 뒤바뀐 클라이언트가 또 로그아웃된다. 재생하면 **두 응답이 같은 세대**이므로 어느 것이 쿠키에 남아도 다음 갱신이 성공한다.
  - **보안 비용**: 유예 창 안에서는 **직전 RT도 아직 통한다**. 탈취자가 그 10초 안에 먼저 재발급하면 탐지되지 않는다. 완화책은 ① 창을 짧게(5~10초) ② `/refresh`에 **Origin/Referer 검증** ③ 같은 RT가 **서로 다른 IP·UA**에서 오면 즉시 재사용 처리(그 경우 유예 적용 제외) ④ 레이트 리밋.
- **해결 C — 분산락(Redisson)**: 회전 구간을 사용자 단위로 직렬화한다.
  ```java
  // 파일 경로: src/main/java/study/auth/jwt/RefreshRotationLock.java
  package study.auth.jwt;

  import org.redisson.api.RLock;
  import org.redisson.api.RedissonClient;
  import org.springframework.stereotype.Component;

  import java.util.concurrent.TimeUnit;
  import java.util.function.Supplier;

  @Component
  public class RefreshRotationLock {

      private final RedissonClient redisson;

      public RefreshRotationLock(RedissonClient redisson) {
          this.redisson = redisson;
      }

      /** 락 키도 hash tag로 사용자 슬롯에 고정한다. waitTime은 짧게(요청 지연 상한), leaseTime은 작업 상한. */
      public <T> T withUserLock(String userId, Supplier<T> action) {
          RLock lock = redisson.getLock("auth:{" + userId + "}:lock:rotate");
          boolean acquired = false;
          try {
              acquired = lock.tryLock(500, 3000, TimeUnit.MILLISECONDS);
              if (!acquired) {
                  // 락 대기 초과: 회전이 이미 진행 중이라는 뜻이므로 클라이언트에 409/재시도를 유도한다.
                  throw new AuthException(AuthError.REFRESH_INVALID);
              }
              return action.get();
          } catch (InterruptedException e) {
              Thread.currentThread().interrupt();
              throw new AuthException(AuthError.REFRESH_INVALID);
          } finally {
              if (acquired && lock.isHeldByCurrentThread()) {
                  lock.unlock();
              }
          }
      }
  }
  ```
  - **언제 필요한가**: 회전이 **Redis 쓰기 1회로 끝나면 Lua CAS만으로 충분**하다. 락이 필요한 건 ①회전과 함께 DB 쓰기(세션 이력)까지 묶어야 할 때 ②회전 결과에 따라 외부 호출(알림)이 있을 때다. 락은 처리량을 사용자 단위로 직렬화하고 추가 RTT(락 획득+해제 = 2회)를 만든다.
  - **락 TTL 함정**: `tryLock(wait, lease)`에서 `leaseTime`을 주면 워치독이 비활성화되어 작업이 lease보다 길면 락이 먼저 풀린다. 회전처럼 짧은 작업은 명시 lease가 낫고, 길어질 수 있는 작업은 워치독(`tryLock()` 무인자)을 쓴다.
- **정리**: **Lua CAS(정합성) + 유예 창(중복 요청 멱등) + 클라이언트 single-flight(요청 자체를 줄임)** 조합이 표준. 락은 위 두 조건이 추가로 필요할 때만.

### 2-2. Clock Skew(서버 간 시계 불일치)

- **무엇이 문제인가**: `exp`는 epoch 초 단위 절대 시각이다. 인증 서버가 만료 1초 전에 토큰을 발급하고, API 서버 시계가 2초 빠르면 **정상 토큰이 즉시 만료**로 판정된다(반대면 최대 오차만큼 더 오래 유효). 다중 인스턴스·컨테이너 환경에서 흔하다.
- **설정**: JJWT `clockSkewSeconds(n)`은 `exp`(만료)와 `nbf`(활성 시작) 판정에 허용 오차를 준다.
  - **AT: 30~60초** — 경계 401 폭주(사용자 체감 "갑자기 로그아웃")를 막는다. 보안 손실은 최대 30초.
  - **RT: 0초(또는 5초 이하)** — RT는 수명이 길어 경계 문제가 드물고, 오차를 주면 **세션 수명을 늘려주는 것**이므로 관대할 이유가 없다.
- **블랙리스트 TTL과의 관계**: AT 검증에 skew를 허용하면 그 AT는 `exp + skew`까지 살아있을 수 있다. 그러므로 블랙리스트 TTL을 **남은 시간 + skew**로 잡아야 한다(코드 0-6). skew를 빼먹으면 마지막 30초 동안 로그아웃된 AT가 통과한다.
- **근본 대책**: 모든 인스턴스에서 **NTP/chrony 동기화**, `iat`·`exp`는 **발급 시점 한 곳에서만** 계산(게이트웨이가 발급하고 각 서비스가 재계산하지 않기), 배포 직후 대량 401 알람은 시계 문제인지 먼저 확인.

### 2-3. Redis 운용 함정 (RT는 캐시가 아니다)

- **eviction**: `allkeys-lru`면 메모리 압박 시 RT 키가 밀려나 **전 사용자 로그아웃**이 된다. RT·블랙리스트를 캐시와 같은 인스턴스에 두지 말고, 최소한 `volatile-lru`(TTL 있는 키만 evict) + `maxmemory` 여유를 확보한다.
- **영속성**: AOF `everysec`이면 failover 시 최대 1초 유실 — 유실된 회전 기록 때문에 "재사용" 오탐이 날 수 있다. RT 저장소는 RDB/AOF를 켜고, **재사용 감지로 세션을 끊을 때는 로그·메트릭을 남겨** 오탐과 실제 공격을 구분한다.
- **클러스터 슬롯**: Lua 다중 키(`family`, `grace`)와 `DEL` 다건은 **모두 같은 슬롯**이어야 한다. `auth:{userId}:...` hash tag로 사용자 단위 고정(코드 0-4). 태그가 없으면 `CROSSSLOT` 오류가 난다.
- **블랙리스트 조회 비용**: 모든 API에 Redis 1 RTT가 붙는다. ①AT TTL을 5~15분으로 줄이면 블랙리스트 필요성 자체가 줄고 ②민감 API(결제·비밀번호 변경·관리자)에만 필터를 걸거나 ③"차단 목록"을 **로컬 캐시(Caffeine, TTL 5초)** 에 미러링한다(전파 지연 5초 허용).
- **fail-open vs fail-close**: AT 서명 검증은 로컬이라 Redis 장애와 무관하다. 블랙리스트 조회가 실패할 때 **fail-open(통과)** 하면 가용성↑·보안↓(로그아웃된 AT가 잠깐 통과), **fail-close(거부)** 하면 보안↑·전면 장애. 서비스 성격으로 결정하고, 기본은 fail-open + 경보(로그아웃된 토큰은 어차피 AT TTL로 소멸).
- **키 위생**: RT 키에 TTL이 없으면 영구 누적. `EXPIRE`를 회전마다 갱신(슬라이딩)하되, **절대 세션 수명 상한**(예: `createdAt + 30일`)을 두어 무한 연장을 막는다.
- **토큰 원문 로깅 금지**: 액세스 로그·APM·에러 트레이스에 `Authorization` 헤더나 쿠키가 찍히지 않게 마스킹. `MONITOR`/`SLOWLOG`에도 값이 노출되므로 블랙리스트 키는 해시(코드 0-4)로.

### 2-4. XSS · CSRF · 쿠키 세부

- **XSS**: RT가 `HttpOnly`면 스크립트가 읽지 못한다. 하지만 **XSS가 있으면 그 페이지 컨텍스트에서 `/refresh`를 호출해 AT를 얻을 수 있다**(읽지 못할 뿐 사용은 가능). 결국 XSS 방어가 1차 방어선이고, `HttpOnly`는 "토큰 유출 후 재사용"을 막는 2차 방어선이다. CSP(`script-src 'self'`), 출력 이스케이프, `X-Content-Type-Options: nosniff` 병행.
- **CSRF**: RT 쿠키는 자동 전송되므로 `/refresh`가 CSRF 표적이 된다. 대책 ①`SameSite=Strict/Lax` ②**Origin/Referer 검증**(`Origin`이 우리 도메인이 아니면 403) ③커스텀 헤더 요구(단순 폼 전송으로는 불가) ④CSRF 토큰(쿠키 기반 세션과 동일하게).
  - **주의**: `SameSite=None`은 서드파티 컨텍스트(모바일 웹뷰·임베디드)에서 필요할 수 있지만 CSRF 노출이 커지므로 그때는 CSRF 토큰을 필수로.
- **경로 최소화**: `Path=/api/auth`면 다른 API 요청에는 RT가 실리지 않아 노출 면적이 줄고, 로그·프록시에 쿠키가 남을 확률도 낮다.
- **HTTPS 전용**: `Secure` 없이 배포하면 평문 구간에서 RT가 그대로 노출된다. 로컬 개발에서만 `secure: false`, 운영은 강제 + HSTS.
- **로그아웃 응답**: `Set-Cookie: rt=; Max-Age=0; Path=/api/auth` — **Path/도메인이 발급 때와 같아야** 브라우저가 실제로 삭제한다(다르면 "삭제했는데 계속 로그인됨" 버그).

### 2-5. 그 밖의 실무 엣지 케이스

- **AT·RT 혼용 공격**: `typ` 클레임을 검사해 REFRESH 토큰을 AT로 쓰지 못하게 한다(코드 0-3). RT는 수명이 길기 때문에 이 검사가 없으면 "14일짜리 AT"가 된다.
- **알고리즘 혼동(alg confusion)**: `alg: none`이나 HS/RS 혼용을 막으려면 **파서에 키를 고정**(`verifyWith(key)`)하고 헤더의 `alg`를 신뢰하지 않는다. JJWT 0.12는 `verifyWith`로 검증하므로 헤더 alg 변조는 서명 불일치로 실패한다.
- **비밀키 관리·회전**: HS256 단일 키는 유출 시 전면 재발급이 필요하다. ①키를 `kid`로 구분해 **이전 키/현재 키를 동시에 검증**(무중단 회전) ②키는 시크릿 매니저에서 주입(코드·이미지에 넣지 않음) ③32바이트 미만 키는 `WeakKeyException`으로 기동 실패하게 둔다.
- **권한 변경 반영**: RT에 role을 넣지 않으면 권한 회수는 **다음 AT 발급(최대 access-ttl)** 부터 반영된다. 즉시 반영이 필요하면 ①AT에 `ver`(권한 버전)를 넣고 사용자별 버전을 Redis에서 비교 ②권한 변경 시 `revokeAllSessions`로 강제 재로그인.
- **비밀번호 변경·탈취 의심**: `revokeAllSessions(userId)` + AT 블랙리스트(현재 요청)로 **모든 기기 즉시 로그아웃**. RT 재사용 감지 시에도 정책에 따라 전체 세션 종료를 선택할 수 있다(기본은 해당 패밀리만).
- **다중 기기 정책**: `auth:{u}:rt:families` 인덱스로 기기별 세션 목록을 유지하면 "기기별 로그아웃"·"활성 세션 목록" UI가 가능하다. 인덱스 SET에도 TTL을 걸어야 하며, 패밀리가 만료되면 SET에서 지워지지 않고 남으므로 **주기적 정리**(ShedLock 배치) 또는 조회 시 존재 확인이 필요하다.
- **재발급 멱등성과 응답 유실**: 첫 `/refresh` 응답이 네트워크에서 유실되면 클라이언트는 RT#2를 받지 못하고 RT#1만 갖는다. RT#1은 유예 창(10초) 안에서만 유효하므로, 그 뒤에는 재로그인이 필요하다. 개선: 유예 창을 클라이언트 재시도 시간(수 초)에 맞추고, 쿠키는 **응답이 도착한 뒤에만** 갱신되므로 `fetch` 타임아웃을 짧게(3~5초) 유지.
- **토큰 크기**: 쿠키는 도메인당 수 KB 제한이 있다. 클레임을 늘리면(권한 목록·조직 정보) RT가 커져 `Set-Cookie`가 잘린다. RT에는 **식별자만**, 상세 정보는 서버 조회.
- **시간대·문자열 비교**: `exp`/`iat`는 epoch 초(UTC)로만 다룬다. `LocalDateTime`으로 변환해 비교하면 서버 타임존에 따라 최대 9시간 오차가 난다.

### 2-6. 아그작(AGZAK) 관점 메모

- 아그작은 JWT 인증과 Redis(ZSET 랭킹)·ShedLock 스케줄링을 쓰는 구조이므로, **인증 Redis와 랭킹 Redis를 같은 인스턴스에 두되 eviction 정책을 분리**하는 것이 안전하다(랭킹은 재구축 가능, RT는 불가).
- 현재 AT 단일 구조라면 RTR 도입은 ①`jti` 클레임 추가 ②RT 패밀리 저장소 ③`/refresh` 교체 ④블랙리스트 필터 네 단계로 쪼갤 수 있고, ②~④는 기존 인증 로직을 건드리지 않아 **점진 배포**가 가능하다.
- 랭킹 조회처럼 **초당 호출이 많은 API**에 블랙리스트 필터를 전부 걸면 Redis QPS가 인증 때문에 늘어난다. public 조회 API는 AT를 요구하지 않게 설계하고(비로그인 허용), 인증이 필요한 쓰기 API에만 필터를 건다.
- ShedLock 배치로 ①만료된 패밀리 인덱스 정리 ②비정상 재사용 감지 로그 집계(같은 userId에서 반복 감지 → 계정 잠금 알림)를 돌리면 운영 부담이 줄어든다.

---

## 3. 기술 면접 단골 꼬리 질문 & 모범 답변 팁

**Q1. JWT는 무상태(Stateless)인데 왜 Redis 같은 상태 저장소를 함께 사용하나요?**
- **A.** "모든 것을 무상태로"는 선택이 아니라 비용 배분입니다. Access Token은 수명이 짧고 호출 빈도가 높으므로 **서명 검증만으로 처리해 Redis/DB 조회를 없애는 것이 이득**입니다. 반면 Refresh Token은 (a) 재발급이라는 드문 이벤트에서만 쓰이고 (b) **회전·재사용 감지·즉시 무효화**라는 상태가 반드시 필요합니다. JWT는 발급 후 취소가 불가능하다는 것이 본질적 한계이므로, "취소가 필요한 지점"에만 상태를 둡니다. 결과적으로 요청당 비용은 늘리지 않고(AT는 무상태), 보안상 반드시 필요한 상태(RT 세대·블랙리스트)만 Redis에 두는 구조입니다. Redis는 TTL로 자동 정리되고 원자적 연산(Lua)으로 회전 경합을 다룰 수 있어 이 용도에 잘 맞습니다.

**Q2. Access Token이 탈취당했을 때 서버는 어떻게 대응할 수 있나요?**
- **A.** 세 가지로 나눠 답합니다. ①**사전 축소**: AT 수명을 5~30분으로 짧게 두어 탈취 성공 시 피해 창을 제한합니다. ②**사후 차단**: 로그아웃·비밀번호 변경 시 해당 AT의 해시를 `BL:{sha256(AT)}`에 **남은 TTL만큼** 저장해 즉시 거부합니다. AT가 만료되면 서명 검증에서 걸리므로 키가 자연 소멸해 저장량이 (로그아웃 수 × AT TTL)로 제한됩니다. ③**근본 차단**: AT 탈취는 보통 XSS·기기 침해에서 오므로 `HttpOnly` 쿠키(RT)·CSP로 유출 경로를 줄이고, RT는 RTR로 재사용을 감지해 **세션 자체를 끊습니다**. 즉 "AT는 짧게 살리고, 이상 징후는 RT 계층에서 잡는다"가 정답 구조입니다.

**Q3. Refresh Token Rotation(RTR)의 탈취 감지(Reuse Detection) 원리는 무엇인가요?**
- **A.** RT를 **세대를 가진 일회용 티켓**으로 취급하는 것입니다. 서버는 세션(패밀리)마다 "현재 유효한 jti"를 Redis에 저장하고, 재발급 요청이 오면 ①제시된 jti가 현재 세대면 → 새 세대로 교체하고 ②아니면 재사용으로 판단합니다. 정상 사용자는 항상 최신 세대를 갖고 있으므로, **같은 세대가 두 번 등장했다는 것은 복제본이 존재한다는 증거**입니다. 감지 시 그 토큰만이 아니라 패밀리 전체를 삭제해 공격자와 사용자 모두 재로그인시킵니다(누가 진짜인지 판별하지 않고 둘 다 끊는 것이 안전). 이때 정상 사용자의 다음 요청도 재사용으로 잡히는데, 그것이 의도된 동작입니다.

**Q4. 프론트엔드에서 Refresh Token을 안전하게 저장하는 방법(LocalStorage vs Cookie)과 XSS/CSRF 방어책은?**
- **A.** **RT는 `HttpOnly; Secure; SameSite` 쿠키, AT는 JS 메모리**가 표준입니다. localStorage는 XSS 한 번에 토큰이 그대로 읽혀 장기 탈취로 이어지므로 RT에는 부적합합니다. HttpOnly 쿠키는 JS가 읽지 못하지만 **자동 전송**되므로 CSRF가 새 위협이 되고, `SameSite=Strict/Lax` + **Origin/Referer 검증** + 필요 시 CSRF 토큰으로 막습니다. 쿠키는 `Path=/api/auth`로 좁혀 재발급 경로에서만 전송되게 하고, `__Host-` 접두어는 서브도메인 쿠키 덮어쓰기를 막습니다. 다만 HttpOnly도 XSS가 존재하면 페이지 컨텍스트에서 재발급 API를 호출해 AT를 얻을 수 있으므로, **CSP·출력 이스케이프가 1차 방어선**이고 HttpOnly는 2차 방어선이라고 설명하는 것이 정확합니다.

**Q5. 동시 다발적인 토큰 재발급 요청 시 발생하는 레이스 컨디션은 어떻게 방어하나요?**
- **A.** 세 층으로 방어합니다. ①**클라이언트 single-flight**: 401을 받은 요청들이 하나의 refresh Promise를 공유해 재발급 호출 자체를 1회로 줄입니다(가장 효과적). ②**서버 원자성**: "읽고-비교하고-쓰기"를 Lua 스크립트로 묶어 **비교와 쓰기 사이에 다른 요청이 끼어들지 못하게** 합니다. 이게 없으면 두 요청이 모두 통과해 서로 덮어쓰고 첫 응답의 토큰이 무효가 됩니다. ③**유예 창(Grace Period)**: 회전 직후 5~10초 동안 직전 세대를 정상으로 인정하되, **새로 발급하지 않고 이미 발급한 다음 세대를 재생**해 두 응답이 같은 세대가 되게 합니다. 트래픽이 몰리는 환경이나 여러 서버가 회전에 관여하면 **Redisson 분산락**으로 사용자 단위 직렬화를 추가합니다. 유예 창은 보안을 조금 양보하는 선택이므로 창을 짧게, Origin 검증·IP/UA 불일치 시 재사용 처리로 보완해야 한다는 점까지 말하면 좋습니다.

---

## 4. 오늘의 체크리스트 (내일 이어서)

- [ ] `/api/auth/refresh`에 동시 요청 2건을 던져 **유예 창 없음 vs 있음**의 결과(로그아웃 여부) 재현
- [ ] 회전 Lua 스크립트를 `read-compare-write` 3단계로 바꿔 **첫 응답 RT가 무효화되는 현상** 재현해 비교
- [ ] 아그작 인증 코드에 `typ` 클레임 검사가 있는지 확인(RT를 AT로 사용하는 혼용 차단)
- [ ] 로그아웃 후 AT로 API 호출 → 401 확인, `TTL auth:bl:*`가 남은 수명과 일치하는지 확인
- [ ] Redis `maxmemory-policy`가 RT 키를 evict하지 않는지 점검(`volatile-lru`/분리 인스턴스)
- [ ] 패밀리 인덱스(`auth:{u}:rt:families`) 고아 항목 정리 배치(ShedLock) 설계
