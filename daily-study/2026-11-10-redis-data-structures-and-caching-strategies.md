# [2026-11-10] Redis 핵심 자료구조 실무 활용(ZSET 랭킹)과 캐싱 전략 — Cache Stampede 방어까지

> **한 줄 핵심 요약**: Redis는 "빠른 캐시"가 아니라 **자료구조 서버**이고, 자료구조 선택이 곧 성능·메모리 설계다. ZSET(SkipList + HashTable)은 `O(log N)` 삽입·순위 조회로 실시간 랭킹의 표준이고, 캐시는 **Cache-Aside + TTL**로 시작하되 만료 순간의 **Cache Stampede(동시 재계산 폭주)**를 분산락·TTL 지터·PER(확률적 조기 재계산) 세 가지로 나눠 방어해야 한다. 세 방법은 "정합성 vs 지연 vs 구현 복잡도"가 서로 다르므로, 캐시 키의 트래픽 성격(hot key 여부)에 따라 선택해야 한다.

---

## 0. 코드 예제 (전체 실행 가능 버전)

### 0-1. 의존성 및 설정

```groovy
// build.gradle
// dependencies {
//     implementation 'org.springframework.boot:spring-boot-starter-web:3.3.x'
//     implementation 'org.springframework.boot:spring-boot-starter-data-redis:3.3.x'
//     testImplementation 'org.springframework.boot:spring-boot-starter-test:3.3.x'
//     testImplementation 'org.testcontainers:testcontainers:1.20.x'
//     testImplementation 'org.testcontainers:junit-jupiter:1.20.x'
// }

// 파일 경로: src/main/resources/application.yml
// spring:
//   data:
//     redis:
//       host: localhost
//       port: 6379
//       timeout: 2s
//       lettuce:
//         pool:
//           max-active: 16
//           max-idle: 8
//           min-idle: 2
//   cache:
//     type: redis
```

### 0-2. 랭킹 DTO (record)

```java
// 파일 경로: src/main/java/study/redis/rank/LeaderboardEntry.java
package study.redis.rank;

/**
 * 랭킹 한 줄. rank는 1-based.
 */
public record LeaderboardEntry(int rank, String userId, double score) {
}
```

```java
// 파일 경로: src/main/java/study/redis/rank/SubmitScoreCommand.java
package study.redis.rank;

public record SubmitScoreCommand(String userId, long points) {
}
```

### 0-3. 랭킹 키 설계 (주간 롤링 + TTL)

```java
// 파일 경로: src/main/java/study/redis/rank/LeaderboardKeys.java
package study.redis.rank;

import java.time.Duration;
import java.time.LocalDate;
import java.time.temporal.WeekFields;

/**
 * 키 네이밍 규칙: {도메인}:{집계주기}:{식별자}
 * - 조회/장애 대응 시 SCAN 패턴으로 묶어서 관리할 수 있게 계층을 고정한다.
 */
public final class LeaderboardKeys {

    private static final String WEEKLY_PREFIX = "leaderboard:weekly";

    private LeaderboardKeys() {
    }

    /** 예: leaderboard:weekly:2026-W46 */
    public static String weekly(LocalDate date) {
        WeekFields iso = WeekFields.ISO;
        int year = date.get(iso.weekBasedYear());
        int week = date.get(iso.weekOfWeekBasedYear());
        return WEEKLY_PREFIX + ":" + year + "-W" + String.format("%02d", week);
    }

    /** 이번 주 랭킹은 다음 주에도 조회될 수 있으므로 8일 TTL을 준다(무기한 키 = 메모리 누수). */
    public static final Duration WEEKLY_TTL = Duration.ofDays(8);

    /** 랭킹 키의 스코어보드와 별도로, 원본 점수/이력은 DB를 신뢰한다. Redis는 파생 데이터. */
    public static String weeklySnapshot(LocalDate date) {
        return weekly(date) + ":snapshot";
    }
}
```

### 0-4. LeaderboardService (ZSetOperations)

```java
// 파일 경로: src/main/java/study/redis/rank/LeaderboardService.java
package study.redis.rank;

import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ZSetOperations;
import org.springframework.stereotype.Service;

import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;
import java.util.Set;

@Service
public class LeaderboardService {

    private final StringRedisTemplate redis;
    private final ZSetOperations<String, String> zSet;

    public LeaderboardService(StringRedisTemplate redis) {
        this.redis = redis;
        this.zSet = redis.opsForZSet();
    }

    /**
     * 점수 반영. ZINCRBY는 원자적이므로 동시 요청에도 합산이 유실되지 않는다.
     * 단, "재시도"가 일어나면 중복 가산되므로 멱등성이 필요하면 이벤트 ID 기반 선차단을 둔다.
     */
    public double submit(SubmitScoreCommand command) {
        String key = LeaderboardKeys.weekly(LocalDate.now());
        Double updated = zSet.incrementScore(key, command.userId(), command.points());
        // 키가 처음 생성된 경우에만 TTL이 붙으므로, 매 호출에서 갱신해도 비용이 크지 않다(EXPIRE O(1)).
        redis.expire(key, LeaderboardKeys.WEEKLY_TTL);
        return updated == null ? 0d : updated;
    }

    /** 상위 N명. reverseRangeWithScores는 점수 내림차순, 동점이면 member 사전순. */
    public List<LeaderboardEntry> topN(int n) {
        String key = LeaderboardKeys.weekly(LocalDate.now());
        Set<ZSetOperations.TypedTuple<String>> tuples = zSet.reverseRangeWithScores(key, 0, n - 1L);
        if (tuples == null || tuples.isEmpty()) {
            return List.of();
        }
        List<LeaderboardEntry> result = new ArrayList<>(tuples.size());
        int rank = 1;
        for (ZSetOperations.TypedTuple<String> tuple : tuples) {
            result.add(new LeaderboardEntry(rank++, tuple.getValue(), toScore(tuple.getScore())));
        }
        return result;
    }

    /** 내 순위(1-based). 미참여면 -1. ZREVRANK는 O(log N). */
    public long myRank(String userId) {
        String key = LeaderboardKeys.weekly(LocalDate.now());
        Long zeroBased = zSet.reverseRank(key, userId);
        return zeroBased == null ? -1L : zeroBased + 1L;
    }

    /** 내 앞뒤 window명. 전체를 읽지 않고 필요한 구간만 O(log N + M)으로 가져온다. */
    public List<LeaderboardEntry> aroundMe(String userId, int window) {
        String key = LeaderboardKeys.weekly(LocalDate.now());
        Long zeroBased = zSet.reverseRank(key, userId);
        if (zeroBased == null) {
            return List.of();
        }
        long start = Math.max(0L, zeroBased - window);
        long end = zeroBased + window;
        Set<ZSetOperations.TypedTuple<String>> tuples = zSet.reverseRangeWithScores(key, start, end);
        if (tuples == null) {
            return List.of();
        }
        List<LeaderboardEntry> result = new ArrayList<>(tuples.size());
        long rank = start + 1L;
        for (ZSetOperations.TypedTuple<String> tuple : tuples) {
            result.add(new LeaderboardEntry((int) rank++, tuple.getValue(), toScore(tuple.getScore())));
        }
        return result;
    }

    private static double toScore(Double score) {
        return score == null ? 0d : score;
    }
}
```

### 0-5. Cache-Aside + TTL 지터 + 단일 재계산(Mutex) + PER

```java
// 파일 경로: src/main/java/study/redis/cache/ProductView.java
package study.redis.cache;

public record ProductView(long id, String name, long price) {
}
```

```java
// 파일 경로: src/main/java/study/redis/cache/CacheEntry.java
package study.redis.cache;

/**
 * 값과 함께 "논리 만료 시각"을 저장한다. 물리 TTL보다 먼저 만료시켜
 * 만료 직전에 미리 재계산(PER)하거나, 재계산 실패 시 stale을 계속 서빙할 수 있다.
 */
public record CacheEntry(ProductView value, long logicalExpireAtMillis, long staleUntilMillis) {
}
```

```java
// 파일 경로: src/main/java/study/redis/cache/ProductCacheService.java
package study.redis.cache;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.concurrent.ThreadLocalRandom;
import java.util.function.Supplier;

@Service
public class ProductCacheService {

    private static final Duration PHYSICAL_TTL = Duration.ofMinutes(10);
    /** 논리 만료는 물리 TTL의 80% 지점: 남은 20%가 stale 서빙 여유분이 된다. */
    private static final double LOGICAL_RATIO = 0.8;
    /** 재계산에 걸리는 시간 추정치(ms). PER의 delta. */
    private static final long DELTA_MILLIS = 200L;
    private static final double BETA = 1.0;

    private final StringRedisTemplate redis;
    private final ObjectMapper objectMapper;

    public ProductCacheService(StringRedisTemplate redis, ObjectMapper objectMapper) {
        this.redis = redis;
        this.objectMapper = objectMapper;
    }

    /**
     * Cache-Aside + PER(XFetch) + Mutex.
     * - 만료 전: 캐시 반환
     * - 논리 만료 후: 확률적으로 한 요청만 재계산, 나머지는 stale 반환
     * - 물리 만료 후(락 획득 실패 시): 짧게 대기 후 DB 직접 조회(스탬피드 폭주 상한을 락 1개로 제한)
     */
    public ProductView get(long productId, Supplier<ProductView> loader) {
        String key = key(productId);
        String raw = redis.opsForValue().get(key);
        long now = System.currentTimeMillis();

        if (raw != null) {
            CacheEntry entry = read(raw);
            if (entry != null && now < entry.logicalExpireAtMillis()) {
                return entry.value();
            }
            if (entry != null && shouldRecompute(now, entry)) {
                // 확률적으로 뽑힌 소수만 재계산. 실패해도 stale로 버틴다.
                tryRecompute(key, loader);
                return entry.value();
            }
            if (entry != null) {
                return entry.value();
            }
        }

        // 캐시 자체가 없음(콜드 스타트/물리 만료): 락으로 재계산을 1개로 직렬화
        String lockKey = key + ":lock";
        String token = Long.toString(ThreadLocalRandom.current().nextLong());
        Boolean acquired = redis.opsForValue().setIfAbsent(lockKey, token, Duration.ofSeconds(3));

        if (Boolean.TRUE.equals(acquired)) {
            try {
                // 락 획득 후 double-check: 다른 스레드가 이미 채웠을 수 있다.
                String rechecked = redis.opsForValue().get(key);
                if (rechecked != null) {
                    CacheEntry entry = read(rechecked);
                    if (entry != null) {
                        return entry.value();
                    }
                }
                ProductView loaded = loader.get();
                put(key, loaded);
                return loaded;
            } finally {
                releaseLock(lockKey, token);
            }
        }

        // 락을 못 잡은 요청: 캐시를 한 번 더 확인하고, 없으면 DB 직접 조회(짧은 지연 + 백오프)
        sleepQuietly(30L + ThreadLocalRandom.current().nextLong(30L));
        String second = redis.opsForValue().get(key);
        if (second != null) {
            CacheEntry entry = read(second);
            if (entry != null) {
                return entry.value();
            }
        }
        return loader.get();
    }

    private boolean shouldRecompute(long now, CacheEntry entry) {
        double random = ThreadLocalRandom.current().nextDouble();
        // XFetch: now - delta * beta * ln(rand) >= logicalExpire 이면 재계산
        double xfetch = now - DELTA_MILLIS * BETA * Math.log(random);
        return xfetch >= entry.logicalExpireAtMillis();
    }

    private void tryRecompute(String key, Supplier<ProductView> loader) {
        String lockKey = key + ":lock";
        String token = Long.toString(ThreadLocalRandom.current().nextLong());
        Boolean acquired = redis.opsForValue().setIfAbsent(lockKey, token, Duration.ofSeconds(3));
        if (!Boolean.TRUE.equals(acquired)) {
            return;
        }
        try {
            put(key, loader.get());
        } finally {
            releaseLock(lockKey, token);
        }
    }

    private void put(String key, ProductView value) {
        long now = System.currentTimeMillis();
        long ttlMillis = PHYSICAL_TTL.toMillis();
        // TTL 지터: 같은 시각에 대량 생성된 키가 동시에 만료되지 않도록 ±10% 흩뿌린다.
        long jitter = (long) (ttlMillis * 0.1 * (ThreadLocalRandom.current().nextDouble() * 2 - 1));
        CacheEntry entry = new CacheEntry(value, now + (long) (ttlMillis * LOGICAL_RATIO), now + ttlMillis + jitter);
        try {
            redis.opsForValue().set(key, objectMapper.writeValueAsString(entry), PHYSICAL_TTL.plusMillis(jitter));
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("cache serialize failed", e);
        }
    }

    private CacheEntry read(String raw) {
        try {
            return objectMapper.readValue(raw, CacheEntry.class);
        } catch (JsonProcessingException e) {
            return null; // 스키마 변경/오염된 값은 미스로 취급하고 덮어쓴다.
        }
    }

    /** 락 해제는 반드시 "내 토큰일 때만" 지운다(남의 락을 지우면 상호배제가 깨진다). */
    private void releaseLock(String lockKey, String token) {
        String script = """
                if redis.call('get', KEYS[1]) == ARGV[1] then
                    return redis.call('del', KEYS[1])
                else
                    return 0
                end
                """;
        redis.execute(new org.springframework.data.redis.core.script.DefaultRedisScript<>(script, Long.class),
                java.util.List.of(lockKey), token);
    }

    private void sleepQuietly(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private String key(long productId) {
        return "cache:product:" + productId;
    }
}
```

### 0-6. Testcontainers 통합 테스트 (쿼리 대신 순위/동점/만료 검증)

```java
// 파일 경로: src/test/java/study/redis/rank/LeaderboardServiceTest.java
package study.redis.rank;

import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import java.time.LocalDate;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
@SpringBootTest
class LeaderboardServiceTest {

    @Container
    static final GenericContainer<?> REDIS =
            new GenericContainer<>(DockerImageName.parse("redis:7.2-alpine")).withExposedPorts(6379);

    @DynamicPropertySource
    static void redisProps(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", REDIS::getHost);
        registry.add("spring.data.redis.port", () -> REDIS.getMappedPort(6379));
    }

    @Autowired LeaderboardService leaderboardService;
    @Autowired StringRedisTemplate redis;

    @AfterEach
    void tearDown() {
        redis.delete(LeaderboardKeys.weekly(LocalDate.now()));
    }

    @Test
    void 점수_합산과_순위_조회() {
        leaderboardService.submit(new SubmitScoreCommand("user-a", 10));
        leaderboardService.submit(new SubmitScoreCommand("user-b", 30));
        leaderboardService.submit(new SubmitScoreCommand("user-a", 25)); // 35점으로 역전

        List<LeaderboardEntry> top = leaderboardService.topN(2);

        assertThat(top).hasSize(2);
        assertThat(top.get(0).userId()).isEqualTo("user-a");
        assertThat(top.get(0).score()).isEqualTo(35d);
        assertThat(leaderboardService.myRank("user-b")).isEqualTo(2L);
        assertThat(leaderboardService.myRank("nobody")).isEqualTo(-1L);
    }

    @Test
    void 주간_키에_TTL이_걸린다() {
        leaderboardService.submit(new SubmitScoreCommand("user-a", 1));

        Long ttl = redis.getExpire(LeaderboardKeys.weekly(LocalDate.now()));

        assertThat(ttl).isNotNull();
        assertThat(ttl).isGreaterThan(0L);
        assertThat(ttl).isLessThanOrEqualTo(LeaderboardKeys.WEEKLY_TTL.toSeconds());
    }
}
```

---

## 1. 개념 및 동작 원리

### 1-1. 자료구조 한눈 비교 (선택 기준표)

| 자료구조 | 내부 구조 | 핵심 명령 / 복잡도 | 실무 유즈케이스 | 피해야 할 사용 |
|---|---|---|---|---|
| **String** | SDS(Simple Dynamic String) | `GET/SET` O(1), `INCR` O(1), `SET NX PX` | 단순 캐시(JSON), 카운터, 분산락 토큰, 세션 | 큰 값(수 MB) 반복 갱신 — 매번 전체 전송 |
| **Hash** | listpack(구 ziplist) → hashtable | `HSET/HGET` O(1), `HINCRBY` O(1) | 객체 필드 단위 부분 갱신, 사용자 프로필, 장바구니 | 필드 수천 개 + `HGETALL` (응답 지연·O(N)) |
| **List** | quicklist(listpack 노드 연결) | `LPUSH/RPUSH` O(1), `LRANGE` O(S+N) | 최근 본 상품, 단순 큐, 로그 버퍼 | 중간 삭제·순위 조회(불가) |
| **Set** | listpack → hashtable | `SADD/SISMEMBER` O(1), `SMEMBERS` O(N) | 좋아요 사용자 집합, 중복 제거, 교집합 | 수백만 원소 `SMEMBERS`(블로킹) |
| **ZSet** | **SkipList + HashTable** | `ZADD/ZINCRBY` O(log N), `ZREVRANK` O(log N), `ZRANGE` O(log N+M) | 실시간 랭킹, 리더보드, 지연 큐(score=실행시각) | 동점 다발 + 잦은 `ZRANGE` 전체 스캔 |
| **Bitmap** | String 위 비트 배열 | `SETBIT/BITCOUNT` O(N/wordsize) | DAU(일 단위), 출석체크(31비트), 온라인 상태 | 듬성듬성한 ID(메모리 낭비) |
| **HyperLogLog** | 2^14 레지스터(6bit) ≈ 12KB | `PFADD/PFCOUNT` O(1), `PFMERGE` | 대규모 UV/DAU 근사 카운팅 | 정확한 값이 필요한 정산 |

### 1-2. String — 가장 단순하지만 결제·락까지 쓰인다

- **동작**: 키 하나에 바이너리 안전한 바이트 배열. 내부적으로는 길이/여유공간을 가진 SDS라 `APPEND` 시에도 재할당이 상각 O(1).
- **캐시로 쓸 때의 핵심**: 값이 크면(예: 상품 목록 JSON 1MB) 네트워크 전송과 역직렬화가 병목이 된다. 캐시 히트율이 99%여도 **값이 크면 DB보다 느릴 수 있다**. 그래서 캐시는 "화면에 필요한 최소 필드"만 담는 DTO로 설계한다.
- **원자 카운터**: `INCR`은 단일 명령이라 멀티 인스턴스에서도 유실이 없다. 좋아요 수·조회수를 DB `UPDATE ... SET count = count + 1`로 처리하면 행 락 경합이 생기는데, Redis 카운터 + 주기적 DB flush 구조가 이를 회피한다(단, 유실 허용 범위를 먼저 정해야 한다).
- **분산락의 기본형**: `SET key token NX PX 3000`. `SETNX` + `EXPIRE`를 두 번 호출하면 그 사이 프로세스가 죽어 **TTL 없는 락 = 영구 데드락**이 된다. 반드시 한 명령으로.

### 1-3. Hash — "필드 단위 갱신"이 필요할 때

- **동작**: 필드-값 쌍의 맵. 필드 수와 값 길이가 작으면 **listpack**(연속 메모리 배열)으로 저장되어 1개 키당 오버헤드가 거의 없다. 임계치(`hash-max-listpack-entries 128`, `hash-max-listpack-value 64` — Redis 7 기본값)를 넘으면 hashtable로 자동 전환된다.
- **실무 이득**: `String`에 JSON을 통째로 넣으면 "닉네임만 바꾸는" 갱신에도 전체를 읽고·파싱하고·다시 써야 한다. Hash는 `HSET user:1 nickname "새닉"`으로 끝나고, 동시에 다른 필드를 만지는 요청과도 충돌 범위가 작다.
- **주의**: 필드가 수만 개인 Hash에서 `HGETALL`은 O(N)이며 Redis 이벤트 루프를 점유한다. 목록 조회는 `HSCAN` 커서 또는 필드 단위 `HMGET`으로 쪼갠다. 또한 Hash에는 **필드별 TTL이 없다**(키 단위 TTL만) — 필드 만료가 필요하면 키를 나누거나 ZSET 인덱스를 병행한다.

### 1-4. ZSet — 실시간 랭킹의 표준, 내부는 두 개의 자료구조

- **구조**: ZSet은 **① member → score 매핑용 HashTable(dict)** 과 **② score 정렬용 SkipList** 를 동시에 유지한다. 그래서 `ZSCORE`(해시 조회 O(1))와 `ZRANGE`(정렬 순회 O(log N + M))가 둘 다 빠르다. 대신 **같은 원소를 두 구조에 중복 저장**하므로 같은 데이터를 Set/List로 둘 때보다 메모리를 더 쓴다(대략 2배 감각).
- **SkipList를 쓰는 이유**: 균형 트리(Red-Black)와 달리 **범위 조회(range)가 단순 연결 포인터 순회**로 끝나고, 구현이 단순해 동시성/메모리 지역성이 유리하다. Redis는 노드 레벨을 확률 1/4로 올려 평균 1.33개 포인터, 기대 높이 O(log N)을 얻는다(최대 레벨 32).
- **랭킹 명령 세트**:
  - `ZADD key score member` — 추가/갱신 O(log N). 6.2+ 에서 `NX/XX/GT/LT/CH/INCR` 옵션.
  - `ZINCRBY key delta member` — 원자적 점수 합산. 게임 점수처럼 "누적"이면 이걸 쓴다.
  - `ZREVRANGE key 0 9 WITHSCORES` — 상위 10명. `ZREVRANK` — 내 순위 O(log N).
  - `ZREVRANGEBYSCORE`, `ZCOUNT`, `ZREMRANGEBYRANK` — 구간 집계·오래된 항목 정리.
  - `ZRANGEBYLEX` — **모든 score가 같을 때만** member 사전순 조회(자동완성에 사용).
- **왜 DB `ORDER BY` 대신 ZSet인가**: `ORDER BY score DESC LIMIT 10`은 인덱스가 있어도 매 요청 정렬 비용·커넥션을 쓰고, 초당 수천 요청이면 DB가 먼저 죽는다. ZSet은 정렬 상태를 **쓰기 시점에 유지**하므로 조회가 O(log N + 10)이다. 대신 원본 데이터의 신뢰원은 DB이고 Redis는 **파생 인덱스**로 두는 것이 원칙이다(Redis 유실 시 DB에서 재구축).

### 1-5. Bitmap / HyperLogLog — "몇 명인가"를 싸게 세는 법

- **Bitmap**: String의 비트를 직접 조작. `SETBIT dau:2026-11-10 {userId} 1` 로 방문 표시, `BITCOUNT`로 일별 DAU. userId를 0부터 조밀하게 매핑하면 100만 명 = 125KB. **듬성듬성한 ID(예: 64비트 snowflake)** 를 그대로 오프셋으로 쓰면 최대 512MB까지 낭비되므로, 별도 정수 매핑 테이블이나 버킷(일 단위 키 분리)이 필요하다. 비트 연산 `BITOP AND`로 "7일 연속 방문"도 한 번에 계산된다.
- **HyperLogLog**: 카디널리티 추정 알고리즘. 12KB 고정 메모리로 **표준 오차 0.81%**, `PFADD/PFCOUNT/PFMERGE`. 1억 명 UV도 12KB로 끝나지만 **개별 원소를 알 수 없고**(집합 연산·목록 조회 불가), 오차가 있으므로 정산·과금에는 쓰지 않는다. "대시보드용 UV"와 "정확한 결제 대상자"는 다른 자료구조를 써야 한다.
- **선택 기준**: 정확도 100% + 원소 열거 필요 → Set/Bitmap, 근사 + 메모리 극한 절약 → HLL.

### 1-6. 캐싱 패턴 — Cache-Aside가 기본값인 이유

| 패턴 | 읽기 | 쓰기 | 장점 | 단점/언제 쓰나 |
|---|---|---|---|---|
| **Cache-Aside (Look-Aside)** | 캐시 미스 시 앱이 DB 조회 후 캐시 적재 | DB 갱신 후 **캐시 삭제** | 구현 단순, 캐시 장애가 DB 장애로 번지지 않음(우회 가능) | 정합성 창 존재. 대부분의 서비스 기본값 |
| **Write-Through** | 캐시에서만 읽음 | 캐시와 DB를 동시에 갱신 | 읽기 정합성 좋음, 캐시 미스가 드묾 | 쓰기 지연 증가, 안 쓰는 데이터까지 캐시 적재 |
| **Write-Behind (Back)** | 캐시에서 읽음 | 캐시에 먼저 쓰고 비동기 DB 반영 | 쓰기 폭발 흡수(카운터·로그) | 캐시 유실 시 데이터 유실 → 유실 허용 데이터만 |
| **Write-Around** | 캐시 미스 시 DB | DB만 갱신, 캐시 무효화 | 캐시 오염 방지(한 번 읽는 데이터) | 읽기 시 매번 미스 |

- **정합성의 핵심 규칙**: ① 가능하면 **DB 커밋 이후에 캐시를 지운다**(커밋 전에 지우면 그 사이 다른 요청이 옛 값을 다시 채운다). ② 캐시 갱신이 아니라 **삭제**를 기본으로 한다(두 곳에 쓰면 순서 문제가 두 배). ③ TTL은 정합성 안전장치이므로 "무한 TTL + 이벤트 무효화"는 이벤트 유실 시 영구 불일치가 된다.
- **Cache Penetration(관통)**: 존재하지 않는 키를 계속 조회 → 매번 DB. 해결: null/empty도 짧은 TTL로 캐싱(negative caching), 또는 Bloom Filter 선차단.
- **Cache Avalanche(눈사태)**: 대량 키가 같은 시각에 만료되거나 Redis가 다운 → DB로 트래픽 집중. 해결: TTL 지터, Redis HA(Sentinel/Cluster), 서킷브레이커·폴백.

### 1-7. Cache Stampede — 만료 순간의 "동시 재계산" 폭주

- **정의**: 인기 키 하나가 만료된 직후, 그 키를 동시에 요청한 수백~수천 스레드가 **모두 캐시 미스**를 맞고 각자 DB/외부 API를 호출하는 현상(Thundering Herd, Dog-pile). 결과적으로 DB 커넥션 풀이 고갈되고, 재계산 쿼리 자체가 느려 **자기 강화 루프**(느린 쿼리 → 더 많은 대기 → 타임아웃 재시도)로 번진다. 단일 스레드가 10ms면 끝날 조회가, 1000 스레드 동시 재계산으로 10초가 되는 것이 전형이다.
- **원인 3요소**: ① 높은 동시성(hot key), ② 동일 만료 시각(모든 요청이 같은 TTL을 받음), ③ 비싼 재계산(집계 쿼리·외부 호출).
- **해결 3종**:
  1. **Mutual Exclusion(분산락)**: 재계산 권한을 1개 스레드로 직렬화. `SET NX PX` 로 락을 잡고, 실패한 요청은 ①짧게 대기 후 캐시 재확인 ②stale 값 반환 ③DB 직접 조회 중 하나로 정책 결정. **DB 부하 상한이 1회**로 고정되는 대신, 락 대기 지연과 락 TTL(작업이 TTL보다 길면 중복 재계산) 관리가 필요하다.
  2. **TTL 지터(Jitter/Randomization)**: 같은 시점에 적재된 키들이 동시에 만료되지 않도록 TTL에 ±5~10% 난수를 준다. **구현 비용 0**으로 눈사태 규모를 분산시키는 가장 먼저 할 일. 다만 hot key 하나의 스탬피드는 막지 못한다.
  3. **PER(Probabilistic Early Recomputation, XFetch)**: 논리 만료 시각을 두고, 만료 **전에** 확률적으로 미리 재계산한다. 남은 시간이 줄수록 확률이 커지고, 재계산이 느릴수록(delta↑) 더 일찍 시작한다. 공식은 `now - delta * β * ln(rand) ≥ logicalExpire`(rand ∈ (0,1), β≈1). 지연이 거의 없고 락도 필요 없지만, **stale을 잠깐 서빙**하는 것을 허용해야 하고 구현이 가장 복잡하다.
- **세 방법 비교**: 지터는 "만료 시각 분산"(전체 부하), 락은 "재계산 직렬화"(정합성 우선), PER은 "만료 전 선제 갱신"(지연 우선). 실무에서는 **지터(전역) + 락 또는 PER(핵심 hot key)** 조합이 표준이다.

---

## 2. 실무 적용 및 주의사항 (Edge Cases & 동시성)

### 2-1. 랭킹 구현의 함정

- **동점(tie) 처리 — score는 double이다**: ZSet score는 IEEE-754 double이라 **정수 정밀도는 2^53(약 9×10^15)까지만 보장**된다. 점수를 `누적점수 * 10^12 + 보정값` 같은 식으로 합성하면 정밀도가 깨져 순위가 뒤섞인다. 실무 해법은 두 가지.
  - (a) **member에 tie-breaker를 인코딩**: ZSet은 score가 같으면 **member 사전순(lexicographic) 오름차순**으로 정렬된다. member를 `"0001234567890:userId"`처럼 **19자리 zero-padding한 역시간**(`Long.MAX_VALUE - endedAtMillis`)으로 만들면, 동점일 때 **먼저 달성한 사람이 위**로 온다. 표시할 때 userId를 파싱해 내려준다.
  - (b) **score를 정수 도메인으로 제한 + 보조 ZSet**: score는 포인트만, 동점 정렬은 별도 인덱스로. 관리 포인트가 늘어나므로 (a)를 권장.
- **주간/시즌 키 롤링**: `leaderboard:weekly:{year}-W{week}`처럼 **키 이름에 집계 주기를 넣고** 매 요청 `EXPIRE`로 TTL을 보장한다. TTL을 안 걸면 시즌이 바뀔 때마다 키가 영구히 쌓여 메모리를 잠식한다(`maxmemory` 도달 → eviction으로 **다른 캐시까지 밀려나는** 2차 장애).
- **랭킹 페이지네이션**: `ZREVRANGE key (page*size) (page*size+size-1)`은 offset이 커질수록 O(log N + offset)으로 느려진다. 심층 페이지는 커서 방식(`ZREVRANGEBYSCORE key (lastScore -inf LIMIT 0 size`)로 바꾸거나, 상위 N(예: 1000)만 노출하고 그 아래는 "내 주변" 뷰로 대체한다.
- **원본과 파생의 분리**: Redis 유실(AOF/RDB 유실, failover) 시 랭킹이 사라진다. 점수의 **신뢰원은 DB**로 두고, Redis는 조회용 인덱스로만 쓴다. 재구축 배치(`ZADD` 파이프라인)를 반드시 만들어 둔다. Redis를 유일 저장소로 쓰면 "재시작 후 랭킹 초기화" 사고가 난다.
- **`ZINCRBY` 재시도와 멱등성**: 네트워크 타임아웃 후 앱이 재시도하면 **점수가 두 번 더해진다**. 결제·포인트 적립처럼 중복 가산이 치명적이면 `SETNX event:{eventId}`(TTL 1일)로 선차단하거나 DB 유니크 제약을 함께 둔다.
- **파이프라인/배치**: 사용자 100명의 점수를 개별 `ZSCORE`로 읽으면 RTT 100번. `ZRANGE ... WITHSCORES` 한 번 또는 `executePipelined`로 묶는다. (Redis 6.2+는 `ZMSCORE`로 다건 조회 가능.)

### 2-2. 캐시 정합성: "커밋 후 삭제"와 그 사이의 경쟁

- **대표 시나리오(갱신 유실)**: ① T1 읽기 미스 → DB에서 v1 조회 ② T2가 v2로 갱신·커밋, 캐시 삭제 ③ T1이 뒤늦게 v1을 캐시에 SET → **TTL까지 v1이 계속 서빙**된다. DB는 v2인데 캐시는 v1.
- **완화책**: (a) TTL을 짧게(정합성 손실 시간 상한을 명시), (b) **지연 이중 삭제**(커밋 직후 삭제 + 300~500ms 후 한 번 더 삭제), (c) 캐시 값에 **버전/타임스탬프**를 넣고 DB 버전보다 오래된 값은 무시, (d) 갱신 빈도가 높은 데이터는 캐시하지 않는다.
- **삭제 실패 처리**: 캐시 삭제가 예외로 실패하면 DB만 갱신되어 불일치가 남는다. 삭제 실패는 **재시도 큐**(또는 아웃박스)에 적재하고, 실패를 삼키지 않는다. "캐시는 최선노력, 정합성은 TTL로 수렴"을 팀 규칙으로 문서화해야 논쟁이 줄어든다.
- **트랜잭션 경계**: `@Transactional` 안에서 캐시를 지우면 **커밋 전에 캐시가 비고**, 그 사이 다른 요청이 옛 DB 값을 다시 채운다. `TransactionSynchronization.afterCommit()`(또는 `@TransactionalEventListener(AFTER_COMMIT)`)으로 커밋 이후 삭제한다.

### 2-3. Stampede 방어를 실무에 넣을 때

- **락 TTL > 재계산 최대 시간**이어야 한다. 재계산이 TTL보다 길면 락이 먼저 풀려 다른 스레드가 또 재계산 → 스탬피드 재발. 재계산 시간을 계측해 TTL을 2~3배로 잡고, 길어질 수 있는 작업은 애초에 요청 경로에서 빼서 배치로 만든다.
- **락 획득 실패 시 정책을 코드로 명시**한다: ㉠ 짧은 sleep 후 캐시 재확인(권장, 지연 수십 ms) ㉡ stale 반환(가장 빠르지만 오래된 값 허용) ㉢ DB 직접 조회(부하 상한이 깨지므로 최후 수단). "그냥 loader.get()"을 실패 경로에 두면 락을 만든 의미가 없다.
- **stale-while-revalidate**: 논리 만료를 물리 TTL보다 앞에 두면, 재계산 중에도 **stale을 즉시 서빙**할 수 있다. 뉴스 피드·상품 목록처럼 초 단위 정합성이 필요 없는 화면에 적합. 결제·재고는 적용 금지.
- **로컬 캐시 2단(Caffeine + Redis)**: hot key는 프로세스 로컬 캐시에 1~5초 두고 Redis를 2차로. Redis QPS와 네트워크 홉이 급감하지만 **인스턴스 간 값 불일치**가 생기므로 TTL을 매우 짧게. 변경 이벤트 브로드캐스트(Redis Pub/Sub)로 무효화할 수 있으나, Pub/Sub은 유실 가능성이 있어 TTL과 병행해야 한다.
- **PER의 함정**: `delta`(재계산 소요 시간) 추정이 틀리면 조기 재계산이 안 되거나 과도해진다. 실제 계측값(최근 p95)을 주기적으로 갱신하고, `beta`로 민감도를 조절한다. 또한 **논리 만료가 물리 TTL보다 반드시 앞서야** 한다(뒤면 값이 사라진 뒤에야 재계산을 시도해 효과가 없다).

### 2-4. Redis 운용에서 자주 나오는 사고

- **싱글 스레드 이벤트 루프**: Redis는 명령을 단일 스레드로 처리한다. `KEYS *`, 큰 `SMEMBERS`, 큰 `HGETALL`, `ZRANGE 0 -1`(전체) 같은 **O(N) 명령 하나가 전체 지연을 만든다**. 운영에서는 `SCAN`(커서) 사용, 목록은 `LIMIT`, `UNLINK`(비동기 삭제) 사용.
- **키 설계**: `user:profile:{id}`, `leaderboard:weekly:2026-W46`처럼 **콜론 계층 + 도메인 접두어**를 고정하고, 모든 키에 TTL을 기본으로 붙인다. TTL 없는 키는 "언젠가 터지는 메모리 누수"다.
- **maxmemory 정책**: `allkeys-lru`(캐시 전용) / `volatile-lru`(TTL 있는 키만) / `noeviction`(저장소로 쓸 때). 캐시와 영속 데이터를 **같은 인스턴스에 섞지 않는다** — eviction이 업무 데이터를 밀어낼 수 있다.
- **클러스터**: 다중 키 명령(`MGET`, `ZUNIONSTORE`, Lua)은 **같은 슬롯**이어야 한다. `{user:1}:cart`처럼 hash tag로 슬롯을 고정하면 해결되지만, 슬롯 편중(hot slot) 위험이 생긴다.
- **직렬화**: 기본 `JdkSerializationRedisSerializer`는 바이트가 크고 역직렬화가 느리다. `StringRedisTemplate` + JSON(또는 `GenericJackson2JsonRedisSerializer`)을 쓰고, record DTO에는 기본 생성자가 없으므로 Jackson `ParameterNamesModule`/`@JsonCreator` 지원을 확인한다. 스키마 변경 시 캐시는 **미스로 취급**하고 덮어쓴다(버전 접미어 `:v2`로 키를 교체하는 방법도 안전).
- **Lettuce vs Jedis**: Lettuce는 Netty 기반 비동기·논블로킹, 커넥션 1개로 다중화(Spring Boot 기본). Jedis는 동기 + 커넥션 풀. 트래픽이 큰 서비스에서 기본 Lettuce를 쓰되, **블로킹 명령(`BLPOP`)은 별도 커넥션**이 필요하다.

### 2-5. 아그작(AGZAK) 관점 메모

- 아그작은 Redis ZSET 랭킹과 ShedLock 기반 스케줄링을 쓰는 구조이므로, 랭킹은 **"DB 원본 + ZSet 파생"** 관계를 명시해 두는 것이 좋다. Redis 유실 시 재구축 배치가 없으면 랭킹이 초기화된다.
- 점수 적립이 이벤트/재시도로 중복 반영될 수 있는 경로(예: RAG 결과 저장 후 점수 부여)에는 `SETNX` 멱등 키를 붙이고, `ZINCRBY` 재시도 정책을 문서화한다.
- 랭킹 조회 API가 상위 N + 내 순위 구조라면 `ZREVRANGE` + `ZREVRANK` 2회 호출로 끝나므로 DB 조회가 사라진다. 응답 DTO는 `record`로 두고 Redis score(double) → 표시용 정수 변환은 경계에서 한 번만 한다.
- 캐시는 `daily-study` 11-05에서 정리한 "화면 단위 DTO" 원칙과 결합한다: 캐시 키 하나가 화면 하나, 값은 표시에 필요한 필드만.

---

## 3. 기술 면접 단골 꼬리 질문 & 모범 답변 팁

**Q1. Sorted Set(ZSET)이 `O(log N)`으로 순위를 조회할 수 있는 이유는 무엇인가요?**
- **A.** ZSet은 내부에 **① member→score 해시 테이블**과 **② score 기준 SkipList** 두 구조를 동시에 유지합니다. `ZSCORE`는 해시로 O(1), `ZRANK`는 SkipList의 각 레벨 포인터를 따라 내려가며 노드마다 서브트리 크기를 더해 순위를 계산하므로 O(log N)입니다. 삽입도 레벨을 확률 1/4로 뽑아 평균 높이를 O(log N)으로 유지하기 때문에 O(log N)입니다. 같은 원소를 두 번 저장하므로 메모리를 더 쓰는 것이 트레이드오프입니다.

**Q2. Redis는 싱글 스레드인데 왜 빠른가요? 그리고 그 구조의 위험은 무엇인가요?**
- **A.** ① 모든 데이터가 메모리에 있고, ② 컨텍스트 스위칭·락 경합이 없으며, ③ 이벤트 루프 기반 논블로킹 I/O(6.0+는 I/O 스레드로 네트워크 읽기/쓰기까지 병렬화)로 처리하기 때문입니다. 위험은 **명령이 원자적이고 순차적**이라는 점입니다. `KEYS *`나 큰 Hash의 `HGETALL` 같은 O(N) 명령 하나가 전체 인스턴스를 수 초 블로킹할 수 있어, 운영에서는 `SCAN`·`LIMIT`·`UNLINK`를 쓰고 느린 명령은 `SLOWLOG`로 감시합니다. 또한 연산은 싱글 스레드지만 전체 처리량은 코어를 늘려도 1코어 기준이라는 확장 한계가 있어, 샤딩/클러스터로 확장합니다.

**Q3. Cache-Aside에서 DB와 캐시의 정합성이 깨지는 시나리오와 해결책은?**
- **A.** 대표적으로 읽기 미스 후 DB를 읽은 스레드가, 그 사이 다른 스레드의 갱신·캐시 삭제보다 늦게 옛 값을 캐시에 쓰는 경우입니다. 해결은 ① 캐시 갱신이 아니라 **커밋 이후 삭제**(`afterCommit`)를 기본으로 하고, ② TTL로 수렴 시간을 보장하며, ③ 필요하면 지연 이중 삭제나 값에 버전을 넣어 오래된 값을 무시합니다. 근본적으로는 "캐시는 최선노력, 원본은 DB"라는 경계를 정하고, 강한 정합성이 필요한 데이터(재고·잔액)는 캐시하지 않거나 짧은 TTL + 무효화 이벤트로 다룹니다.

**Q4. Cache Stampede란 무엇이고, 어떻게 막나요?**
- **A.** 인기 키가 만료된 순간 동시에 들어온 요청들이 모두 미스를 맞고 각자 DB를 조회해 폭주하는 현상입니다. 세 가지로 나눠 방어합니다. ① **분산락(SET NX PX)** 으로 재계산을 1회로 직렬화 — DB 부하 상한은 확실하지만 락 대기 지연과 TTL 관리가 필요합니다. ② **TTL 지터**로 만료 시각을 흩뿌려 전체 부하를 분산 — 비용이 거의 없지만 단일 hot key에는 효과가 없습니다. ③ **PER(XFetch)** 로 만료 전 확률적으로 미리 재계산 — 지연이 없지만 stale 서빙을 허용해야 합니다. 실무에서는 지터를 전역 기본값으로 두고, 핵심 hot key만 락이나 PER로 보강합니다.

**Q5. 객체를 캐시할 때 String(JSON)과 Hash 중 무엇을 선택하시겠습니까?**
- **A.** 갱신 단위와 조회 단위로 결정합니다. **필드 하나만 자주 바뀌고**(닉네임, 상태값) 부분 조회가 많으면 Hash — `HSET`/`HMGET`으로 전체 직렬화 비용과 충돌 범위를 줄일 수 있습니다. 반면 **한 번에 통째로 읽고 쓰는 화면 DTO**나 TTL을 필드별로 다르게 가져야 하는 경우, 원자적 교체가 필요한 경우에는 String + JSON이 단순합니다. Hash는 필드별 TTL이 없다는 점과 큰 Hash의 `HGETALL`이 이벤트 루프를 막는다는 점을 함께 고려합니다.

**Q6. 랭킹에서 동점자가 생기면 순위를 어떻게 처리하시겠습니까?**
- **A.** ZSet은 score가 double이라 점수에 보정값을 곱해 넣는 방식은 2^53 정밀도 한계로 위험합니다. 대신 **동점 시 member 사전순 정렬**을 이용해 member에 역시간을 zero-padding으로 인코딩하면 "먼저 달성한 사람이 상위"가 되고, 정렬 규칙이 명시적·재현 가능해집니다. 노출 순위는 `RANK`가 아니라 **동점 동순위**(예: 공동 2위) 정책인지 결정해 API 스펙에 명시해야 합니다.

---

## 4. 오늘의 체크리스트 (내일 이어서)

- [ ] 아그작 랭킹 키에 TTL이 실제로 걸려 있는지 `TTL leaderboard:*`로 점검, 주간 롤링 키 전략 정리
- [ ] `ZINCRBY` 재시도 중복 가산 재현 → 멱등 키(`SETNX event:{}`) 적용 데모
- [ ] Cache-Aside + 지터 + 분산락 조합을 부하 테스트(k6)로 재현: 락 없음 vs 락 vs PER의 p99·DB QPS 비교
- [ ] `@Transactional` + `@TransactionalEventListener(AFTER_COMMIT)` 캐시 무효화 패턴 코드로 정리
- [ ] Redis `SLOWLOG GET 10`으로 O(N) 명령 습관 점검(`KEYS`, `HGETALL`, 전체 `ZRANGE`)
