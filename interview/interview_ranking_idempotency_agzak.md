# 면접 답변 가이드 — 아그작 랭킹·게임 멱등 설계

> 작성: 2026-05-08 / 검증 기준: 04_agzak 실제 코드 + 블로그 `agzak_ranking_db_split.md`·`agzak_scheduler_deadlock.md`
> 대상: 백엔드·서비스 개발 면접 (동시성·분산락·캐시·멱등 깊이 질문 대응)
> 함께 보기: [interview_auth_4projects.md](./interview_auth_4projects.md)

---

## 0. 코드 사실관계 한눈에 (면접관 검증 대비)

| 주장 | 파일 | 핵심 메서드/구문 |
|---|---|---|
| Redis ZSET 3종 동시 운영 | `domain/ranking/support/RedisRankingStore.java` | `zTotal/zOriginal/zGain`, `rank:{yyyyMMdd}:{suffix}` 키 |
| ZSET + HASH 분리 적재 | 같은 파일 | `bulkUpsertTodayUsers` 안의 `executePipelined` |
| 즉시 반영 (O(log N)) | `RankingService.refreshTodayForMember` | `redisStore.upsertTodayUser(...)` |
| TOP N 조회 | `RedisRankingStore.readTop` | `z.reverseRangeWithScores(zset, 0, limit-1)` |
| 내 등수 조회 | `RedisRankingStore.zrevRank1Base` | `z.reverseRank(zset, memberId)` |
| 시간당 reseed | `global/scheduler/RankingScheduler.hourlyReseedRedis` | `cron = "0 55 * * * *"`, ZCARD 비교 후 noop |
| 매시간 스냅샷 | `RankingScheduler.todayRebuild` | `cron = "55 59 * * * *"`, `rebuildTodaySnapshot()` |
| Blue/Green 분산락 | 위 두 메서드 | `@SchedulerLock(name=..., lockAtMostFor=PT4M, lockAtLeastFor=PT10S/PT20S)` |
| 읽기 fallback | `RankingService.getTodayTopN` | Redis 비거나 예외 → `getSnapshotTopN(d, limit)` |
| 고아 데이터 prune | `RankingService.rebuildTodaySnapshot` + `pruneTodayOrphans` | Redis ZSET ids ∩ MySQL alive ids 차집합 삭제 |
| 게임 멱등 + 일일캡 | `domain/game/service/RedisQuotaService.claimOne` | **Lua 스크립트**로 `SETNX` + `INCR` 원자 실행 |
| 자정 KST TTL | `RedisQuotaService.ensureCapTtl` / `nextMidnightKSTDate` | 내일 00:00:05 KST `expireAt` |
| 세션 재개 패턴 | `domain/game/service/CookieGameService.startForCurrentUser` | 진행 중 세션 있으면 마지막 attempt에서 이어 계산 |
| 인벤토리 동시성 | 같은 파일 `lockInventoryAndCountAvailable` | `inventoryRepository.findWithPessimisticWriteByMember` (비관적 락) |
| 재현 가능한 액운 재분배 | `CookieGameService.applyMisfortuneRedistribution` | `SecureRandom.setSeed(seedHint ^ magnitude)` |

---

## 1. 랭킹: "왜 MySQL ORDER BY 만으로는 안 됐는가"

### 처음 시도와 한계 (블로그 근거)

> *"초기에는 `Fortune` 테이블을 그 날짜로 조회한 뒤 메모리 정렬로 응답했습니다. 운영에 올린 후 네 가지 문제를 만났습니다:*
> *① 매 요청마다 풀스캔 + ORDER BY,*
> *② Total·Original·Gain 세 정렬을 매번 메모리에서 새로 함,*
> *③ 내 등수 조회가 별도 카운트 쿼리,*
> *④ 가장 결정적으로 — Fortune 점수가 게임으로 계속 변하기 때문에 어제 랭킹이 어제 그대로 보존되지 않았습니다."*

### 본질 재정의

> *"같은 '랭킹'이지만 두 요구의 자료구조 특성이 다릅니다.*
> *오늘 랭킹 = 점수 변동이 잦고 즉시 정렬된 상태가 필요 → in-memory 정렬 자료구조 (Redis ZSET).*
> *과거 일자 랭킹 = 이미 확정된 결과, 영속 보관 + 가끔 조회 → RDB 스냅샷 테이블 (MySQL).*
> *두 책임을 단일 테이블에 끼워 맞추려 한 것 자체가 비용을 키운 원인이었습니다."*

### 결정: Redis ZSET (실시간) + MySQL `ranking` (스냅샷) 이원화

---

## 2. ZSET을 선택한 이유 — "정렬을 자료구조로 갖는다"

> *"MySQL `ORDER BY`는 매 요청마다 정렬 비용을 지불합니다. ZSET은 ZADD 시점에 O(log N)으로 정렬 상태를 유지하고, ZREVRANGE·ZREVRANK는 그 상태를 그대로 꺼내기만 하면 됩니다. 즉, '정렬 결과를 자료구조로 가지고 있다'는 게 ZSET의 본질입니다."*

### 키 설계 (RedisRankingStore.java)

```
rank:20251119:total      ZSET (memberId → totalScore)
rank:20251119:original   ZSET (memberId → originalScore)
rank:20251119:gain       ZSET (memberId → 증가율 %)
rank:20251119:user:42    HASH (nickname/total/original)
```

**ZSET은 점수만, HASH에 부가정보 분리**한 이유:

> *"ZSET 원소를 작게 유지해야 메모리·네트워크 비용이 줄어듭니다. 닉네임 같은 보조 정보는 HASH로 분리해서 TOP N 조회 시 ZSET은 정렬·등수 계산만 책임지고, 화면 표시용 메타데이터는 HASH에서 별도로 보강합니다."*

### 점수 변경 → 즉시 반영 (`refreshTodayForMember` → `upsertTodayUser`)

```java
redis.executePipelined((RedisCallback<Object>) connection -> {
    h.put(hUser, "nickname", nickname);
    h.put(hUser, "total",    String.valueOf(total));
    h.put(hUser, "original", String.valueOf(original));
    z.add(zTotal,    String.valueOf(memberId), total.doubleValue());
    z.add(zOriginal, String.valueOf(memberId), original.doubleValue());
    z.add(zGain,     String.valueOf(memberId), gain);
    return null;
});
```

> *"파이프라이닝으로 7개 명령을 한 번의 왕복으로 묶었습니다. 네트워크 RTT를 한 번만 소비하면서 ZSET 3종·HASH 3필드를 동시에 일관성 있게 갱신합니다."*

### 세 가지 기준 동시 노출 — 같은 사용자, 정렬만 다른 결과

> *"Total·Original·Gain 세 가지 랭킹은 각각 별도 ZSET입니다. 화면에서 탭만 바꿔도 정렬 기준이 달라지는 UX를 별도 계산 없이 제공할 수 있습니다."*

---

## 3. 동기화 전략 — Redis가 1차 저장소일 때 가져야 할 3가지

Redis가 빠르지만 단일 장애점이 되면 곤란합니다. 세 가지 안전장치를 함께 뒀습니다.

### ① 시간당 Reseed (MySQL → Redis): 정합성 자동 회복

`@Scheduled(cron = "0 55 * * * *", zone = "Asia/Seoul")`

```java
Long zcard = redisStore.zcard(redisStore.zTotal(today));
boolean needsReseed = (zcard == null) || (zcard.intValue() != fortunes.size());
if (!needsReseed) return;     // noop
redisStore.bulkUpsertTodayUsers(rows);
```

> *"매시간 55분에 ZCARD와 MySQL 행 수를 비교합니다. 같으면 noop — 강제로 매시간 갈아엎지 않습니다. 다를 때만 bulk upsert로 채우고, 실패해도 예외를 던지지 않고 로그만 남깁니다. Redis가 잠깐 문제여도 다음 요청에서 자연 회복됩니다."*

### ② 매시간 스냅샷 (Redis → MySQL `ranking`): 영속화 + 고아 prune

`@Scheduled(cron = "55 59 * * * *", zone = "Asia/Seoul")`

```java
Set<Long> ids   = all.stream().map(UserRow::memberId).collect(toSet());
Set<Long> alive = mysqlStore.existingMemberIds(ids);
// 차집합 = 탈퇴 회원, ZSET에서 제거
for (Long mid : ids) if (!alive.contains(mid)) redisStore.removeTodayUser(mid);
// 남은 회원만 rank_no 1부터 채워 MySQL ranking 테이블에 영속
```

> *"스냅샷 단계에서 ZSET이 시간이 지날수록 부풀어 오르는 문제를 같이 해결했습니다. 탈퇴 회원 등으로 발생한 고아 데이터를 차집합 연산으로 prune하고, 살아있는 회원만 rank_no를 1부터 매겨 MySQL `ranking` 테이블에 저장합니다. 이 시점 이후로 Fortune 점수가 다시 변해도 과거 일자 랭킹은 흔들리지 않습니다."*

### ③ 읽기 Fallback — Redis 죽어도 5xx 없음

```java
try {
    var total = redisStore.readTop(d, redisStore.zTotal(d), limit);
    // ... 모두 비어있으면 ↓
    return getSnapshotTopN(d, limit);   // MySQL ranking 테이블 대체
} catch (Exception e) {
    return getSnapshotTopN(d, limit);   // Redis 자체 예외도 동일 분기
}
```

> *"Redis가 죽거나 비어있을 때 사용자에게 5xx를 내보내는 대신, 마지막 정시 스냅샷을 반환합니다. 사용자 입장에서는 '잠시 갱신이 멈춘 것' 정도로 보이고 서비스가 죽지는 않습니다."*

---

## 4. ShedLock 분산락 — Blue/Green 환경에서 스케줄러 단일 실행

```java
@SchedulerLock(name = "rankingSnapshot",
               lockAtMostFor = "PT4M", lockAtLeastFor = "PT20S")
```

> *"04_agzak는 Blue/Green 무중단 배포 중 두 컨테이너가 동시에 떠 있는 시간대가 존재합니다. 자정에 같은 스케줄러가 두 번 실행되면 ranking 테이블에 같은 (date, member_id) 키로 두 번 INSERT가 발생해 InnoDB Deadlock이 났습니다. ShedLock으로 단일 인스턴스만 실행되도록 분산락을 걸어 해결했습니다."*

핵심 파라미터의 의미:

- `lockAtMostFor = PT4M`: 인스턴스가 죽어도 4분 후엔 락이 풀려 다른 인스턴스가 다음 회차를 받음 (좀비 락 방지)
- `lockAtLeastFor = PT20S`: 작업이 너무 빨리 끝나도 20초간 락 유지 → **시계 오차로 두 인스턴스가 같은 회차를 두 번 가져가는 사고 방지**

> *"이 두 파라미터의 트레이드오프 이해가 중요합니다. lockAtMostFor가 짧으면 가용성이 좋지만 좀비 락 위험이 줄고, 길면 안전하지만 실패 복구가 느립니다. 작업이 평소 30초~1분 안에 끝나는 걸 측정한 뒤 4분으로 잡았습니다."*

---

## 5. 게임 멱등 설계 — Lua 스크립트로 원자성 보장

### 문제 상황

> *"강화게임은 게스트 → 회원 전환 직후, FCM 푸시 알림 클릭 → 딥링크 복귀 직후 같은 '재진입' 시점에 같은 요청이 두 번 들어올 수 있습니다. 같은 사용자가 같은 요청 ID로 cap 증가를 두 번 받으면 게임 균형이 무너집니다. 그래서 **클라이언트가 idempotency-key를 발급**하고, 서버는 그 키가 처음 들어온 요청만 처리하도록 했습니다."*

### Redis Lua 스크립트 (RedisQuotaService.java:73-93)

```lua
local cap = redis.call('GET', KEYS[1])
if not cap then cap = tostring(ARGV[1]) end
cap = tonumber(cap)
local maxCap = tonumber(ARGV[2])
if cap >= maxCap then
    return cap                              -- 상한 도달, 추가 차감 없음
end
if redis.call('SETNX', KEYS[2], ARGV[3]) == 0 then
    return cap                              -- 멱등키 이미 사용됨, 무효
end
cap = cap + 1
if cap > maxCap then cap = maxCap end
redis.call('SET', KEYS[1], tostring(cap))
redis.call('PEXPIRE', KEYS[2], ARGV[4])      -- 멱등키 24h TTL
return cap
```

### 왜 Lua인가 — 원자성

> *"`SETNX` + `INCR` + `SET`을 자바 쪽에서 따로 호출하면, 첫 두 호출 사이에 같은 idempotency-key가 다시 들어올 수 있습니다. Lua 스크립트는 Redis 안에서 **단일 명령처럼 원자적으로 실행**됩니다. 이 안에서 (1) cap 조회·기본값 (2) 상한 체크 (3) 멱등키 SETNX (4) cap +1 (5) 멱등키 TTL 부여까지 한 번에 끝나서 race condition을 원천 차단합니다."*

### 핵심 동작 분기

| 시나리오 | 결과 |
|---|---|
| 첫 호출 (멱등키 신규) | SETNX 성공 → cap +1, 새 값 반환 |
| 같은 idempotency-key 재시도 | SETNX 실패 (0 반환) → **cap 그대로 반환** = 멱등 |
| cap이 이미 상한(10) | 멱등키 SETNX 시도조차 안 함 → cap 그대로 |
| 멱등키 24h 만료 후 같은 키 재전송 | SETNX 다시 성공 → 정상 +1 (사실상 새 요청) |

### 자정 KST TTL — Cap은 매일 리셋

```java
private void ensureCapTtl(String key) {
    Long ttlSec = redis.getExpire(key, TimeUnit.SECONDS);
    if (ttlSec < 0) redis.expireAt(key, nextMidnightKSTDate());
    // nextMidnightKSTDate = 내일 00:00:05 KST (5초 버퍼)
}
```

> *"cap 키에는 자정 KST 만료를 보장합니다. -1(무기한) 또는 -2(없음)일 때만 보강하고, 이미 유효한 TTL이 있으면 그대로 둡니다. 5초 버퍼를 둔 이유는 자정 정각에 호출되는 일일 리셋 배치와의 시계 오차를 흡수하기 위해서입니다."*

---

## 6. 강화 게임의 동시성·재현성

`CookieGameService.java`에 멱등성과 별개로 두 가지 동시성·재현성 보장 장치가 더 있습니다.

### 비관적 락 — 인벤토리 동시성

```java
Inventory inventory = inventoryRepository
    .findWithPessimisticWriteByMember(member)
    .orElseGet(...);
```

> *"인벤토리 정원이 50개로 제한되어 있고, 게임 완료 시점에 새 쿠키를 굽기 직전에 '오늘 굽기 가능한가'를 결정합니다. 같은 사용자가 다중 탭이나 PWA + 모바일로 동시에 마지막 시도를 하면 정원 초과 가능성이 있어서 `findWithPessimisticWriteByMember`로 SELECT … FOR UPDATE를 걸어 직렬화했습니다."*

> *"멀티 디바이스 사용이 일상화된 PWA 환경에서 분산락(Redis) 대신 RDB 비관적 락을 쓴 이유는, 인벤토리는 회원당 1행이고 트랜잭션 안에서 자연스럽게 닫혀서 RDB 락의 비용이 충분히 낮았기 때문입니다."*

### 세션 재개 패턴 — 멱등성과 다른 결의 '이어하기'

```java
var inProgressOpt = gameSessionRepository
    .findTopByMemberAndGameDateAndStatusOrderBySessionNoDesc(
        member, today, GameSession.SessionStatus.IN_PROGRESS);

if (inProgressOpt.isPresent()) {
    // 마지막 attempt에서 nextAttemptNo, currentBaseProb, snapshot 5종 복원
    return StartGameResponse.builder()...nextAttemptNo(...).currentBaseProb(last.getNextBaseProb())...
}
```

> *"앱을 끄거나 푸시 알림으로 외부에 갔다 돌아왔을 때 진행 중 세션이 있으면 '새로 시작'이 아니라 마지막 시도 이후 상태에서 정확히 이어집니다. nextAttemptNo, currentBaseProb, snapshot 5종(Love/Money/Achievement/Health/Misfortune) 모두 마지막 GameAttempt에서 복원합니다. 멱등성이 '같은 요청을 두 번 받아도 결과 1번'을 보장한다면, 세션 재개는 '연속된 요청이 끊겨도 진행상태를 잃지 않음'을 보장하는 다른 결의 설계입니다."*

### 재현 가능한 액운 재분배 — seed 고정

```java
var rnd = new SecureRandom();
rnd.setSeed((seedHint == null ? 0L : seedHint) ^ (long) magnitude);
```

> *"액운 점수를 4종 행운에 무작위 분배하는 로직인데, seed를 `세션ID ^ magnitude`로 고정했습니다. 같은 세션은 같은 결과 — 클라이언트가 결과를 재요청하거나 디버깅 시점에 동일 결과를 재현할 수 있습니다. SecureRandom을 쓰면서도 결정성을 확보한 사례입니다."*

---

## 7. 면접 시뮬레이션 질문 답변

### Q1. "Redis ZSET을 랭킹에 쓸 때 어떤 점이 가장 좋은가요?"

> *"세 가지 답을 드릴 수 있습니다.*
> *첫째, 정렬 결과를 자료구조로 들고 있어 ZREVRANGE는 정렬 비용이 0입니다.*
> *둘째, ZREVRANK로 '내 등수'를 O(log N)에 단건 조회할 수 있어 별도 카운트 쿼리가 필요 없습니다.*
> *셋째, ZADD 한 번이면 점수 갱신이 끝나기 때문에 점수가 자주 바뀌는 도메인에 적합합니다.*
> *반면 RDB ORDER BY는 매 요청마다 정렬을 다시 수행하고, 인덱스가 있어도 페이지 캐시 미스가 나면 디스크 I/O가 발생합니다."*

### Q2. "Redis가 단일 장애점이 되지 않으려면?"

> *"04_agzak에서는 세 가지 안전장치를 함께 뒀습니다.*
> *① 시간당 reseed로 MySQL → Redis 정합성을 자동 회복. ZCARD와 MySQL 행 수가 일치하면 noop이라 평소엔 비용 없음.*
> *② 매시간 스냅샷으로 Redis → MySQL ranking 테이블 영속화. 과거 일자는 RDB에서 그대로 서빙.*
> *③ 읽기 fallback — Redis 비어있거나 예외 시 MySQL 스냅샷으로 자동 대체. 5xx가 사용자에게 노출되지 않습니다.*
> *이 세 가지를 함께 두지 않은 Redis 1차 저장소는 실제로 단일 장애점입니다."*

### Q3. "스케줄러를 분산환경에서 단일 실행시키는 방법은?"

> *"04_agzak는 Blue/Green 무중단 배포라 두 컨테이너가 동시에 떠 있는 시간대가 존재합니다. ShedLock을 도입해 `@SchedulerLock(name=..., lockAtMostFor=PT4M, lockAtLeastFor=PT20S)`로 락을 걸었고, 락은 별도 ShedLock 테이블에 INSERT로 보관됩니다. lockAtMostFor는 좀비 락 방지, lockAtLeastFor는 시계 오차로 두 인스턴스가 같은 회차를 두 번 가져가는 사고 방지 — 두 파라미터의 의미를 분리해서 이해하는 게 핵심입니다.*
> *덕분에 운영 14일 동안 새벽 ranking 테이블 InnoDB Deadlock은 0건이었습니다."*

### Q4. "멱등성(idempotency)은 어떻게 구현했나요?"

> *"강화게임에서 클라이언트가 idempotency-key를 발급하고 서버는 Lua 스크립트 하나로 (1) cap 조회·기본값 (2) 상한 체크 (3) 멱등키 SETNX (4) cap +1 (5) 멱등키 TTL 부여를 원자적으로 실행했습니다. Lua를 쓴 이유는 자바에서 SETNX와 INCR을 따로 호출하면 그 사이에 같은 idempotency-key가 또 들어올 수 있기 때문입니다. SETNX가 0을 반환하면 이미 처리된 요청이므로 cap을 그대로 반환 — 같은 요청을 N번 받아도 결과는 1번 처리된 상태로 수렴합니다."*

### Q5. "그 멱등키는 어떻게 청소하나요?"

> *"멱등키는 24h TTL을 부여해 Redis가 자동으로 만료시킵니다. cap 키는 자정 KST에 만료되도록 expireAt으로 보강하고, 5초 버퍼를 둬서 자정 일일 리셋 배치와의 시계 오차를 흡수했습니다. 별도 청소 배치가 필요 없다는 게 Redis TTL의 장점입니다."*

### Q6. "왜 멱등은 Redis인데 인벤토리 동시성은 RDB 비관적 락인가요?"

> *"두 락의 비용 구조가 다릅니다.*
> *멱등은 '같은 키로 들어오는 짧은 시점 요청을 거르는' 일이라 짧은 TTL과 RTT가 본질 → Redis SETNX가 압도적으로 가볍습니다.*
> *인벤토리 정원 50 체크는 같은 사용자의 동시 트랜잭션 안에서 '쿠키 추가'와 함께 직렬화돼야 의미가 있어서, JPA 트랜잭션 안에서 닫히는 RDB 비관적 락이 자연스럽습니다.*
> *분산락은 '여러 인스턴스에 걸친 동시 실행을 막을 때' 쓰는 도구지, 모든 동시성 문제의 답은 아닙니다."*

---

## 8. 한 줄 무기 (외워두면 좋음)

| 상황 | 한 줄 |
|---|---|
| ZSET vs ORDER BY | *"ORDER BY는 매번 정렬 비용을 지불하지만, ZSET은 ZADD 시점에 O(log N)으로 정렬 상태를 유지합니다 — 정렬 결과를 자료구조로 갖는다는 게 본질입니다."* |
| Redis SPOF 방지 | *"reseed + 스냅샷 + read fallback 세 가지를 함께 두지 않은 Redis 1차 저장소는 단일 장애점입니다."* |
| ShedLock 두 파라미터 | *"lockAtMostFor는 좀비 락 방지, lockAtLeastFor는 시계 오차로 두 번 실행되는 사고 방지 — 의미가 분리된 두 안전장치입니다."* |
| 멱등 vs 트랜잭션 | *"멱등은 '같은 요청을 N번 받아도 결과는 1번', 트랜잭션은 '여러 작업을 ACID로 묶음' — 다른 결의 보장이라 둘 다 필요합니다."* |
| 멱등 Redis 선택 이유 | *"Lua 스크립트로 SETNX + INCR을 원자 실행 — Java 쪽에서 따로 호출하면 그 사이가 race window가 됩니다."* |
| 분산락 vs 비관적 락 | *"인스턴스 경계를 넘는 동시성은 분산락(Redis/ShedLock), 트랜잭션 안에서 닫히는 동시성은 RDB 비관적 락. 도구를 문제의 경계에 맞춰 선택해야 합니다."* |
| 자정 TTL 5초 버퍼 | *"일일 리셋 배치와의 시계 오차를 흡수하려고 자정 00:00:05 KST로 잡았습니다. 분산 시스템에선 시계 동기화 가정을 최소화하는 게 안전합니다."* |

---

## 9. 만약 더 깊은 질문이 들어온다면

### "ZSET 키가 일자별로 쌓이는데 메모리 문제는 없나요?"

> *"키 네임스페이스가 `rank:yyyyMMdd:*` 이라 일자별로 격리되어 있고, MySQL 스냅샷이 끝난 뒤에는 과거 일자 ZSET은 EXPIRE를 거는 방식으로 청소할 수 있습니다. 다만 04_agzak 운영 14일 규모에서는 일자당 ZSET이 수백~수천 원소 수준이라 메모리 압박은 없었습니다."*

### "ZSET이 정확히 같은 점수면 어떻게 정렬되나요?"

> *"ZSET은 동점 시 사전순 lex 정렬을 합니다. 04_agzak는 member_id를 문자열로 넣어서 사실상 가입 순으로 정렬되는데, 동점자를 '먼저 도달한 순'으로 보고 싶다면 score를 `score * 1e9 - timestamp`처럼 인코딩해서 자연 정렬을 깰 수도 있습니다. 다만 그러면 사람이 score를 디버깅하기 어려워지는 트레이드오프가 생깁니다."*

### "Lua 스크립트는 클러스터 모드에서 어떻게 동작하나요?"

> *"Lua 스크립트의 모든 KEYS는 같은 hash slot에 속해야 합니다. RedisQuotaService에서는 `agzak:quota:{memberId}:{day}:cap`와 `agzak:quota:{memberId}:{day}:idem:{key}`로 prefix가 같아서 클러스터 모드에서도 같은 노드로 라우팅됩니다 — `{memberId}:{day}` 부분이 hash tag로 작동하도록 키 설계를 한 점이 의도된 부분입니다."* (실제 코드는 `{}` hash tag 명시는 없으니, 면접에서는 "단일 인스턴스 운영이었지만, 클러스터로 가려면 hash tag를 명시할 것"이라고 정직하게 답하기)

### "스냅샷 도중에 점수가 더 들어오면?"

> *"`rebuildTodaySnapshot`이 ZSET을 한 번에 readTop(MAX)로 읽어와서 그 시점 스냅을 만들고 MySQL에 INSERT합니다. 그 뒤에 들어오는 점수 갱신은 다음 시각 스냅샷에 반영됩니다. 즉, 'rank_no가 정해지는 순간'은 매시 59분 55초이고, 그 외 시각의 ZSET은 실시간이라 답이 자연스럽게 분리됩니다."*

---

## 10. 정리 — 면접관에게 남길 메시지

> *"같은 '랭킹'이라도 요구가 다르면 저장소를 분리한다.*
> *Redis가 1차 저장소가 되려면 reseed + 스냅샷 + fallback 세 가지가 함께 있어야 한다.*
> *동시성·멱등은 도구가 다르다 — 인스턴스 경계엔 분산락, 트랜잭션 안엔 비관적 락, 같은 요청 거름엔 Lua + SETNX.*
> *이 세 가지를 실제 코드와 운영 14일 측정치로 함께 말할 수 있는 게 04_agzak의 자산입니다."*

> 면접관이 "그래서 그 코드 직접 보여줄 수 있나요?"라고 하면 §0의 인덱스로 즉시 파일·메서드까지 댈 것.
