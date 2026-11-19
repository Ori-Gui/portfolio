# [2026-11-19] REST API 설계와 멱등성(Idempotency) — HTTP 의미론부터 Idempotency-Key 프로토콜, 재시도·중복 결제 방어까지

> **한 줄 핵심 요약**: 멱등성은 "같은 요청을 두 번 보내도 결과가 한 번"이라는 **HTTP 의미론**이고, 실무에서 필요한 이유는 서버가 느려서가 아니라 **타임아웃이 실패가 아니라 '결과 미확인'** 이기 때문입니다. 따라서 방어는 ① 클라이언트가 보내는 **Idempotency-Key**, ② 서버가 키와 응답을 기록하는 **DB 유니크 제약**, ③ 업무 연산 자체를 **조건부 상태 전이**로 만드는 세 층으로 겹쳐야 하고, 그중 **DB 유니크 제약만이 최종 방어선**입니다(조회 후 저장하는 check-then-act는 동시 요청에서 반드시 뚫립니다).

---

## 1. 개념 및 동작 원리

### 1.1 REST는 "예쁜 URL"이 아니라 자원·표현·무상태의 규약이다

| 요소 | 의미 | 실무에서 자주 어기는 지점 |
| --- | --- | --- |
| 자원(Resource) | URI는 **명사**로 식별자만 표현 | `/getOrder?id=1`, `/order/create` 같은 동사형 URI |
| 표현(Representation) | JSON·XML 등 **표현**을 주고받음 | 엔티티를 그대로 노출해 스키마 변경이 곧 API 파괴 |
| 무상태(Stateless) | 요청 하나에 처리에 필요한 정보가 다 들어감 | 서버 세션·로컬 캐시에 이전 요청 상태 의존 |
| 계층형(Hierarchical) | 컬렉션/단일 자원의 경로 규칙 | `/orders/{id}/items/{itemId}`처럼 소유 관계를 경로로 표현 |

```text
# 컬렉션과 단일 자원의 기본 골격
GET    /api/orders                 # 목록 (필터·정렬·페이지는 쿼리 파라미터)
POST   /api/orders                 # 생성 → 201 + Location: /api/orders/{id}
GET    /api/orders/{id}            # 단일 조회
PUT    /api/orders/{id}            # 전체 교체
PATCH  /api/orders/{id}            # 부분 수정
DELETE /api/orders/{id}            # 삭제
POST   /api/orders/{id}/cancel     # 상태 전이는 '동사'가 아니라 '하위 자원'으로
```

- 상태 전이(`cancel`, `approve`)는 REST 순수주의자 논쟁이 있지만, **표현 불가능한 동작을 억지로 PATCH 필드로 밀어 넣는 것보다 하위 자원 + POST가 명확**합니다. 중요한 건 URI 모양이 아니라 그 동작이 **멱등한가**입니다.
- 목록 API는 `?status=PAID&sort=createdAt,desc&page=0&size=20`처럼 **필터·정렬·페이지를 쿼리로** 분리합니다. 정렬 필드는 화이트리스트로 검증하지 않으면 SQL 인젝션·인덱스 미사용으로 이어집니다.

### 1.2 HTTP 메서드의 두 속성: Safe와 Idempotent

| 메서드 | Safe(읽기 전용) | Idempotent | 본문 | 대표 응답 |
| --- | --- | --- | --- | --- |
| GET | ✅ | ✅ | 없음(무시) | 200, 304, 404 |
| HEAD | ✅ | ✅ | 없음 | 200(헤더만) |
| OPTIONS | ✅ | ✅ | 없음 | 204 + Allow |
| PUT | ❌ | ✅ | 있음(전체) | 200/204, 201(신규 생성 시) |
| DELETE | ❌ | ✅ | 없음 | 204, 404(정책에 따라) |
| POST | ❌ | ❌ | 있음 | 201, 202, 200 |
| PATCH | ❌ | ❌ (구현 의존) | 있음(부분) | 200/204 |

- **Safe**는 "서버 상태를 바꾸지 않는다"는 뜻이고, **Idempotent**는 "여러 번 적용해도 서버 상태가 같다"는 뜻입니다(RFC 9110 §9.2.1~9.2.2). 서로 다른 개념이라, DELETE는 Safe가 아니지만 Idempotent합니다.
- **Idempotent는 응답이 같은 것이 아닙니다.** 첫 `DELETE /orders/1`은 204, 두 번째는 404일 수 있습니다. 서버 상태(주문이 삭제됨)는 동일하므로 멱등입니다. 이 구분을 못 하면 면접에서 바로 틀립니다.
- **PUT이 멱등인 이유**는 "리소스 전체를 주어진 표현으로 덮어쓰기"라서입니다. `PUT /orders/1 {status: PAID}`를 두 번 보내도 상태는 같습니다. 반면 **PATCH는 정의상 멱등이 아닙니다** — `{"op":"add","path":"/qty","value":1}` 같은 증분 연산이 가능하기 때문입니다. 실무에서는 PATCH도 `{"status":"PAID"}` 같은 **절대값 대입**으로 설계해 사실상 멱등하게 만듭니다.
- **POST가 멱등이 아닌 이유**는 서버가 새 식별자를 만들기 때문입니다. `POST /orders`를 두 번 보내면 주문 2건이 생깁니다. 결제·주문·발송처럼 POST로 만드는 모든 것은 여기에 해당합니다.

### 1.3 타임아웃은 실패가 아니라 "결과 미확인"이다

중복 결제의 전형적인 경로입니다.

```text
클라이언트                서버                     DB
   |  POST /payments       |                       |
   |---------------------->|  BEGIN                |
   |                       |---------------------->|
   |  (3s read timeout)    |  INSERT payment       |
   |  X--------------------|---------------------->|  커밋 (3.2s)
   |  "실패" 판단 → 재시도 |                       |
   |---------------------->|  BEGIN → INSERT       |
   |                       |---------------------->|  커밋 → 결제 2건
```

- 클라이언트가 보는 것은 **응답의 부재**이지 처리의 실패가 아닙니다. 네트워크 단절, 로드밸런서의 **자동 재시도**(idempotent 메서드에 한해 재시도하는 LB도 있고, 그렇지 않은 LB도 있습니다), 모바일 회선 전환, 게이트웨이 타임아웃, 사용자의 새로고침/더블클릭, 배치 재처리, Kafka의 **at-least-once** 소비(11-16 노트)가 모두 같은 상황을 만듭니다.
- 그래서 "재시도 금지"는 해법이 아닙니다. 재시도는 **네트워크에서 반드시 일어나야 하는 정상 동작**이고, 서버가 그것을 흡수해야 합니다.
- 타임아웃 예산도 함께 봐야 합니다. 클라이언트 3초 / 게이트웨이 5초 / 서버 10초 / DB 8초처럼 **바깥이 안쪽보다 짧아야** 의미가 있습니다. 반대로 두면 클라이언트가 포기한 요청을 서버가 계속 처리해 중복 창이 넓어집니다(11-18 노트의 GC pause도 같은 문제를 만듭니다).

### 1.4 방어 3계층 — 각 계층이 막는 것

| 계층 | 구현 | 막는 것 | 못 막는 것 |
| --- | --- | --- | --- |
| ① 클라이언트 키 | `Idempotency-Key` 헤더(UUID) | 같은 논리적 요청의 반복 전송 | 키를 안 보내거나 매번 새로 만드는 클라이언트 |
| ② 서버 기록 | `(scope, key)` **유니크 인덱스** + 응답 저장 | 동시 요청 경쟁까지 포함한 중복 | 서로 다른 키로 들어온 중복(사용자가 두 번 결제 버튼을 눌러 키 2개 생성) |
| ③ 업무 연산 | 조건부 UPDATE(`WHERE status='PAID'`), 상태 전이 검증 | 의미상 같은 결과를 만드는 서로 다른 요청 | 금액 계산·재고 차감처럼 매번 새 결과를 만드는 연산 |

- ③이 가장 견고합니다. "이미 취소된 주문은 취소할 수 없다"를 **DB 조건절**로 표현하면, 어떤 경로로 몇 번 들어와도 결과가 하나입니다.
- ②는 ①이 없어도 동작하는 **최종 방어선**입니다. 클라이언트 개발자가 키를 안 보내도, 서버가 요청 지문으로 유도 키를 만들 수 있습니다(단, 본문이 같아야 하므로 완전하지 않음).
- ①만 있고 ②가 없으면 아무것도 막지 못합니다. "Redis에 키를 저장했다"는 것도 **DB 유니크 제약이 없으면 동시 요청에서 뚫립니다**(1.7절).

### 1.5 Idempotency-Key 프로토콜 — 사실상 표준

`Idempotency-Key`는 IETF `draft-ietf-httpapi-idempotency-key-header`로 논의 중인 **초안**이며, 아직 RFC가 아닙니다. 그런데도 Stripe·Adyen·Square·PayPal이 같은 형태로 구현해서 **금융 API의 사실상 관례**가 되었습니다.

```text
요청 1: POST /payments   Idempotency-Key: 8f14e45f-...
        → 서버: 키 없음 → PROCESSING 기록 → 업무 처리 → 200 {paymentId: P1} 저장
        → 응답 유실

요청 2: POST /payments   Idempotency-Key: 8f14e45f-...   (같은 본문)
        → 서버: 키 있음 + 지문 일치 + COMPLETED → 저장된 200 {paymentId: P1} 재생
        → 결제는 1건, 클라이언트는 정상 응답 수신

요청 3: POST /payments   Idempotency-Key: 8f14e45f-...   (본문 다름)
        → 서버: 지문 불일치 → 422 (키 재사용 오류)
```

상태 기계와 응답 규칙입니다.

| 서버 상태 | 같은 키·같은 본문 | 같은 키·다른 본문 |
| --- | --- | --- |
| 기록 없음 | 새로 처리 → 결과 저장 | — |
| PROCESSING (처리 중) | **409 Conflict** + `Retry-After: 1` | 422 |
| COMPLETED | **저장된 상태코드·본문 재생** (재처리 없음) | 422 |
| FAILED | 재시도 허용(새 처리) | 422 |
| TTL 만료(예: 24h) | 키 폐기 → 새 요청으로 처리 | — |

- **409와 422의 역할이 다릅니다.** 409는 "지금 같은 키가 처리 중이니 잠시 후 다시"이고, 422는 "이 키는 이미 다른 요청에 쓰였다"입니다. 둘을 400 하나로 뭉개면 클라이언트가 재시도해도 되는지 판단할 수 없습니다.
- 저장 기간은 **비즈니스 재시도 창보다 길어야** 합니다. 24시간이 흔하고, 정산·배치 재처리까지 고려하면 7일을 두기도 합니다. TTL이 지난 키를 클라이언트가 재사용하면 **중복 결제가 다시 가능**해지므로, 키는 "한 번 쓰고 버리는 값"으로 취급해야 합니다.
- 키는 **추측 가능한 값이면 안 됩니다**(순번, 주문 ID). 타인이 키를 알아내 응답을 재생당할 수 있으므로 UUID v4/v7을 씁니다. 또한 키는 **권한 스코프 안에서만** 유효해야 합니다. `(user_id, key)`로 묶지 않으면 A 사용자의 키가 B 사용자의 응답을 재생할 수 있습니다.
- **요청 지문(fingerprint)** 은 `method + path + SHA-256(body)`입니다. 지문이 없으면 키만 같고 금액이 다른 요청이 첫 응답을 재생받아 **조용히 잘못된 결과**를 돌려줍니다.

### 1.6 멱등성을 "설계"하는 세 가지 형태

1. **자연 멱등(naturally idempotent)**: `PUT` 전체 교체, `DELETE`, `SET` 연산, 절대값 `PATCH`. 재시도해도 결과가 같으므로 추가 장치가 필요 없습니다.
2. **조건부 연산(conditional)**: `UPDATE orders SET status='CANCELED' WHERE id=? AND status='PAID'` → 영향 행 수 1이면 성공, 0이면 "이미 취소됨"으로 정상 응답. 락 없이도 동시 요청이 하나만 통과합니다.
3. **키 기반(key-based)**: POST처럼 매번 새 결과를 만드는 연산에 `Idempotency-Key`를 붙여 **결과를 재생**. 구현 비용이 가장 크고, 그래서 결제·주문·정산처럼 **중복이 금전 손실인 곳에만** 적용합니다.

---

## 2. 실무 적용 및 주의사항 (Edge Cases)

### 2.1 테이블 설계 — 유니크 제약이 곧 멱등성이다

```sql
-- db/migration/V15__create_idempotency_key.sql
CREATE TABLE idempotency_key (
    id            BIGINT       NOT NULL AUTO_INCREMENT,
    scope         VARCHAR(120) NOT NULL COMMENT 'user:{userId}:{METHOD}:{path}',
    idem_key      VARCHAR(128) NOT NULL COMMENT '클라이언트가 보낸 Idempotency-Key',
    request_hash  CHAR(64)     NOT NULL COMMENT 'SHA-256(method + path + body)',
    state         VARCHAR(20)  NOT NULL COMMENT 'PROCESSING / COMPLETED / FAILED',
    status_code   SMALLINT     NULL,
    content_type  VARCHAR(120) NULL,
    response_body MEDIUMBLOB   NULL,
    created_at    DATETIME(6)  NOT NULL,
    expires_at    DATETIME(6)  NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uk_idem_scope_key (scope, idem_key),
    KEY idx_idem_expires (expires_at)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_0900_ai_ci;
```

- 유니크 키는 `(scope, idem_key)`입니다. `scope`에 사용자와 메서드·경로를 넣어 **다른 사용자·다른 엔드포인트의 키 충돌을 차단**합니다.
- `response_body`는 `MEDIUMBLOB`(최대 16MB)으로 두되, **저장 상한을 코드로 강제**합니다(예: 64KB 초과 시 저장하지 않고 재처리 허용). 대용량 응답을 저장하면 이 테이블이 순식간에 커집니다.
- `expires_at`은 TTL 정리용 인덱스입니다. MySQL에는 TTL 삭제가 없으므로 **ShedLock이 걸린 정리 잡**으로 지웁니다(11-06 노트의 분산 스케줄러 패턴).
- 같은 목적을 `INSERT ... ON DUPLICATE KEY UPDATE`로 처리할 수도 있지만, `INSERT IGNORE`는 **데이터 오류(길이 초과 등)까지 조용히 삼키므로** 쓰지 않는 편이 좋습니다.

### 2.2 엔티티와 리포지토리

```java
// src/main/java/com/example/order/idempotency/IdempotencyRecord.java
package com.example.order.idempotency;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Lob;
import jakarta.persistence.Table;
import jakarta.persistence.UniqueConstraint;

import java.time.LocalDateTime;

@Entity
@Table(
        name = "idempotency_key",
        uniqueConstraints = @UniqueConstraint(
                name = "uk_idem_scope_key",
                columnNames = {"scope", "idem_key"}
        )
)
public class IdempotencyRecord {

    public enum State { PROCESSING, COMPLETED, FAILED }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 120)
    private String scope;

    @Column(name = "idem_key", nullable = false, length = 128)
    private String key;

    @Column(name = "request_hash", nullable = false, length = 64)
    private String requestHash;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private State state;

    @Column(name = "status_code")
    private Integer statusCode;

    @Column(name = "content_type", length = 120)
    private String contentType;

    @Lob
    @Column(name = "response_body")
    private byte[] responseBody;

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;

    @Column(name = "expires_at", nullable = false)
    private LocalDateTime expiresAt;

    protected IdempotencyRecord() {
    }

    public static IdempotencyRecord claim(String scope, String key, String requestHash,
                                          LocalDateTime now, LocalDateTime expiresAt) {
        IdempotencyRecord record = new IdempotencyRecord();
        record.scope = scope;
        record.key = key;
        record.requestHash = requestHash;
        record.state = State.PROCESSING;
        record.createdAt = now;
        record.expiresAt = expiresAt;
        return record;
    }

    public void complete(int statusCode, String contentType, byte[] body) {
        this.state = State.COMPLETED;
        this.statusCode = statusCode;
        this.contentType = contentType;
        this.responseBody = body;
    }

    public void fail() {
        this.state = State.FAILED;
    }

    public boolean isSameRequest(String requestHash) {
        return this.requestHash.equals(requestHash);
    }

    public Long getId() { return id; }
    public State getState() { return state; }
    public Integer getStatusCode() { return statusCode; }
    public String getContentType() { return contentType; }
    public byte[] getResponseBody() { return responseBody; }
    public LocalDateTime getCreatedAt() { return createdAt; }
}
```

```java
// src/main/java/com/example/order/idempotency/IdempotencyRecordRepository.java
package com.example.order.idempotency;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.LocalDateTime;
import java.util.Optional;

public interface IdempotencyRecordRepository extends JpaRepository<IdempotencyRecord, Long> {

    Optional<IdempotencyRecord> findByScopeAndKey(String scope, String key);

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("delete from IdempotencyRecord r where r.expiresAt < :now")
    int deleteExpired(@Param("now") LocalDateTime now);
}
```

### 2.3 서비스 — "선점(claim) → 처리 → 완료"에서 트랜잭션을 분리하는 이유

여기서 가장 많이 틀리는 부분입니다. **유니크 제약 위반 예외를 같은 트랜잭션 안에서 catch해 계속 쓰면 안 됩니다.** JPA는 제약 위반이 난 트랜잭션을 rollback-only로 표시하므로, 이후 조회·저장이 `UnexpectedRollbackException`으로 터집니다. 그래서 **선점(INSERT)은 별도 트랜잭션(REQUIRES_NEW)**, 판정과 저장은 바깥에서 수행합니다.

```java
// src/main/java/com/example/order/idempotency/IdempotencyClaimer.java
package com.example.order.idempotency;

import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

import java.time.Clock;
import java.time.Duration;
import java.time.LocalDateTime;

@Component
public class IdempotencyClaimer {

    private static final Duration TTL = Duration.ofHours(24);

    private final IdempotencyRecordRepository repository;
    private final Clock clock;

    public IdempotencyClaimer(IdempotencyRecordRepository repository, Clock clock) {
        this.repository = repository;
        this.clock = clock;
    }

    /** 새 키를 선점한다. 유니크 제약 위반은 호출자에게 DataIntegrityViolationException으로 전파된다. */
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public Long claim(String scope, String key, String requestHash) {
        LocalDateTime now = LocalDateTime.now(clock);
        IdempotencyRecord saved = repository.saveAndFlush(
                IdempotencyRecord.claim(scope, key, requestHash, now, now.plus(TTL))
        );
        return saved.getId();
    }
}
```

```java
// src/main/java/com/example/order/idempotency/IdempotencyService.java
package com.example.order.idempotency;

import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

import java.time.Clock;
import java.time.Duration;
import java.time.LocalDateTime;

@Service
public class IdempotencyService {

    /** 처리 중(PROCESSING) 기록이 이 시간을 넘으면 죽은 요청으로 보고 회수한다. */
    private static final Duration STALE_AFTER = Duration.ofSeconds(30);
    private static final int MAX_RESPONSE_BYTES = 64 * 1024;

    public enum Outcome { PROCEED, REPLAY, IN_FLIGHT, MISMATCH }

    public record Begin(Outcome outcome, Long recordId, Integer statusCode,
                        String contentType, byte[] body) {

        static Begin proceed(Long id) { return new Begin(Outcome.PROCEED, id, null, null, null); }
        static Begin inFlight() { return new Begin(Outcome.IN_FLIGHT, null, null, null, null); }
        static Begin mismatch() { return new Begin(Outcome.MISMATCH, null, null, null, null); }
        static Begin replay(IdempotencyRecord r) {
            return new Begin(Outcome.REPLAY, r.getId(), r.getStatusCode(),
                    r.getContentType(), r.getResponseBody());
        }
    }

    private final IdempotencyClaimer claimer;
    private final IdempotencyRecordRepository repository;
    private final Clock clock;

    public IdempotencyService(IdempotencyClaimer claimer,
                              IdempotencyRecordRepository repository,
                              Clock clock) {
        this.claimer = claimer;
        this.repository = repository;
        this.clock = clock;
    }

    public Begin begin(String scope, String key, String requestHash) {
        try {
            Long recordId = claimer.claim(scope, key, requestHash);
            return Begin.proceed(recordId);
        } catch (DataIntegrityViolationException duplicate) {
            // 별도 트랜잭션이 롤백됐으므로 이 트랜잭션은 오염되지 않는다.
            return interpretExisting(scope, key, requestHash);
        }
    }

    // 같은 클래스 내부 호출이므로 @Transactional을 붙여도 프록시를 타지 않는다(11-08 노트).
    // 여기서는 "읽기 → 판정 → 필요 시 별도 빈의 claim" 이라 단일 트랜잭션이 필요 없다.
    private Begin interpretExisting(String scope, String key, String requestHash) {
        IdempotencyRecord existing = repository.findByScopeAndKey(scope, key)
                .orElseThrow(() -> new IllegalStateException(
                        "유니크 위반인데 기존 레코드가 없습니다: " + scope + "/" + key));

        if (!existing.isSameRequest(requestHash)) {
            return Begin.mismatch();                       // 422
        }
        if (existing.getState() == IdempotencyRecord.State.COMPLETED) {
            return Begin.replay(existing);                 // 저장 응답 재생
        }
        if (existing.getState() == IdempotencyRecord.State.PROCESSING) {
            boolean stale = existing.getCreatedAt().isBefore(LocalDateTime.now(clock).minus(STALE_AFTER));
            if (!stale) {
                return Begin.inFlight();                   // 409 + Retry-After
            }
            // 앞선 요청이 죽었다고 판정 → 회수(takeover). 이 UPDATE 자체도 조건부여야 한다.
            int taken = repository.markFailedIfStale(existing.getId(), existing.getCreatedAt());
            if (taken == 0) {
                return Begin.inFlight();                   // 다른 스레드가 먼저 회수함
            }
        }
        // FAILED → 재시도 허용
        Long recordId = claimer.claim(scope, key, requestHash);
        return Begin.proceed(recordId);
    }

    /** 업무 트랜잭션이 커밋된 뒤 호출된다. 응답 재생용으로만 저장한다. */
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void finish(Long recordId, int statusCode, String contentType, byte[] body) {
        if (recordId == null) {
            return;
        }
        if (statusCode >= 500 || body == null || body.length > MAX_RESPONSE_BYTES) {
            // 서버 오류·대용량 응답은 캐싱하지 않는다 → 같은 키 재시도로 다시 처리 가능
            repository.findById(recordId).ifPresent(IdempotencyRecord::fail);
            return;
        }
        repository.findById(recordId)
                .ifPresent(record -> record.complete(statusCode, contentType, body));
    }

    /** TTL 정리. 분산 환경에서는 ShedLock으로 한 노드만 실행한다. */
    @Transactional
    public int purgeExpired() {
        return repository.deleteExpired(LocalDateTime.now(clock));
    }
}
```

```java
// IdempotencyRecordRepository에 추가할 조건부 회수 쿼리
// 오래된 PROCESSING 기록만 FAILED로 바꾼다. 영향 행 0이면 다른 노드가 먼저 가져간 것.
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("""
        update IdempotencyRecord r
           set r.state = com.example.order.idempotency.IdempotencyRecord.State.FAILED
         where r.id = :id
           and r.state = com.example.order.idempotency.IdempotencyRecord.State.PROCESSING
           and r.createdAt = :createdAt
        """)
int markFailedIfStale(@Param("id") Long id, @Param("createdAt") java.time.LocalDateTime createdAt);
```

> `assert` 대신 **조건부 UPDATE의 영향 행 수**로 승자를 결정하는 것이 핵심입니다. `SELECT` 후 `if (stale)` 판정은 두 노드가 동시에 통과합니다.

### 2.4 필터 — 본문을 캐싱하고 응답을 재생한다

```java
// src/main/java/com/example/order/idempotency/CachedBodyHttpServletRequest.java
package com.example.order.idempotency;

import jakarta.servlet.ReadListener;
import jakarta.servlet.ServletInputStream;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletRequestWrapper;

import java.io.BufferedReader;
import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStreamReader;
import java.nio.charset.Charset;
import java.nio.charset.StandardCharsets;

/** 본문을 한 번 읽어 보관하고, 이후 컨트롤러가 다시 읽을 수 있게 한다. */
public class CachedBodyHttpServletRequest extends HttpServletRequestWrapper {

    private final byte[] body;

    public CachedBodyHttpServletRequest(HttpServletRequest request) throws IOException {
        super(request);
        this.body = request.getInputStream().readAllBytes();
    }

    public byte[] getBody() {
        return body;
    }

    @Override
    public ServletInputStream getInputStream() {
        ByteArrayInputStream in = new ByteArrayInputStream(body);
        return new ServletInputStream() {
            @Override public boolean isFinished() { return in.available() == 0; }
            @Override public boolean isReady() { return true; }
            @Override public void setReadListener(ReadListener listener) { /* 비동기 미지원 */ }
            @Override public int read() { return in.read(); }
        };
    }

    @Override
    public BufferedReader getReader() {
        Charset charset = getCharacterEncoding() == null
                ? StandardCharsets.UTF_8
                : Charset.forName(getCharacterEncoding());
        return new BufferedReader(new InputStreamReader(getInputStream(), charset));
    }
}
```

```java
// src/main/java/com/example/order/idempotency/IdempotencyFilter.java
package com.example.order.idempotency;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.boot.autoconfigure.security.SecurityProperties;
import org.springframework.core.annotation.Order;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;
import org.springframework.web.util.ContentCachingResponseWrapper;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.HexFormat;
import java.util.Set;

@Component
@Order(SecurityProperties.DEFAULT_FILTER_ORDER + 10)   // 보안 필터(-100) 뒤, 컨트롤러 앞
public class IdempotencyFilter extends OncePerRequestFilter {

    private static final String HEADER = "Idempotency-Key";
    private static final Set<String> GUARDED_METHODS = Set.of("POST", "PATCH");

    private final IdempotencyService idempotencyService;

    public IdempotencyFilter(IdempotencyService idempotencyService) {
        this.idempotencyService = idempotencyService;
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        // 키를 안 보내는 요청은 대상이 아니다. GET/DELETE는 이미 멱등이므로 제외.
        return !GUARDED_METHODS.contains(request.getMethod())
                || request.getHeader(HEADER) == null
                || request.getContentType() == null
                || !request.getContentType().startsWith("application/json");
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String key = request.getHeader(HEADER).trim();
        if (key.isEmpty() || key.length() > 128) {
            response.sendError(HttpServletResponse.SC_BAD_REQUEST, "invalid Idempotency-Key");
            return;
        }

        CachedBodyHttpServletRequest cached = new CachedBodyHttpServletRequest(request);
        String scope = scopeOf(request);
        String requestHash = sha256Hex(request.getMethod() + "\n" + request.getRequestURI()
                + "\n" + new String(cached.getBody(), StandardCharsets.UTF_8));

        IdempotencyService.Begin begin = idempotencyService.begin(scope, key, requestHash);

        switch (begin.outcome()) {
            case MISMATCH -> write(response, 422, "application/json",
                    "{\"code\":\"IDEMPOTENCY_KEY_REUSED\",\"message\":\"같은 키로 다른 요청이 접수되었습니다.\"}"
                            .getBytes(StandardCharsets.UTF_8));
            case IN_FLIGHT -> {
                response.setHeader("Retry-After", "1");
                write(response, 409, "application/json",
                        "{\"code\":\"REQUEST_IN_PROGRESS\",\"message\":\"동일 요청이 처리 중입니다.\"}"
                                .getBytes(StandardCharsets.UTF_8));
            }
            case REPLAY -> write(response,
                    begin.statusCode() == null ? 200 : begin.statusCode(),
                    begin.contentType(), begin.body());
            case PROCEED -> {
                ContentCachingResponseWrapper wrapped = new ContentCachingResponseWrapper(response);
                try {
                    chain.doFilter(cached, wrapped);
                } finally {
                    // 업무 트랜잭션은 이미 커밋된 상태. 여기서 실패해도 응답은 나가야 한다.
                    try {
                        idempotencyService.finish(begin.recordId(), wrapped.getStatus(),
                                wrapped.getContentType(), wrapped.getContentAsByteArray());
                    } catch (RuntimeException ignored) {
                        // 저장 실패 시 키가 PROCESSING으로 남고 STALE_AFTER 뒤 회수된다.
                    }
                    wrapped.copyBodyToResponse();
                }
            }
        }
    }

    private String scopeOf(HttpServletRequest request) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        String principal = (auth == null || !auth.isAuthenticated()) ? "anonymous" : auth.getName();
        return "user:" + principal + ":" + request.getMethod() + ":" + request.getRequestURI();
    }

    private void write(HttpServletResponse response, int status, String contentType, byte[] body)
            throws IOException {
        response.setStatus(status);
        if (contentType != null) {
            response.setContentType(contentType);
        }
        if (body != null) {
            response.getOutputStream().write(body);
        }
        response.flushBuffer();
    }

    private String sha256Hex(String value) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            return HexFormat.of().formatHex(digest.digest(value.getBytes(StandardCharsets.UTF_8)));
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException("SHA-256 미지원 JVM", e);
        }
    }
}
```

- **본문을 읽는 순서**가 중요합니다. 필터에서 `getInputStream()`을 읽고 그대로 넘기면 컨트롤러가 빈 본문을 받습니다. 반드시 래퍼로 감싸 재읽기를 보장합니다.
- **응답 버퍼링은 JSON 소형 응답에만** 적용합니다. 파일 다운로드·SSE·스트리밍은 `ContentCachingResponseWrapper`에 담기지 않거나 메모리를 터뜨립니다. `shouldNotFilter`의 `application/json` 조건이 그 방어입니다.
- `copyBodyToResponse()`를 빠뜨리면 **응답 본문이 비어 나갑니다.** 실제로 가장 자주 나는 버그입니다.
- 필터 순서는 Security 뒤여야 인증 주체를 얻을 수 있고, `RequestContextHolder`·`@Async`보다 앞이어야 합니다. 보안 필터보다 앞에 두면 `anonymous` 스코프로 키가 섞입니다.

### 2.5 업무 코드 — 조건부 상태 전이를 함께 둔다

```java
// src/main/java/com/example/order/order/OrderRepository.java
package com.example.order.order;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.LocalDateTime;

public interface OrderRepository extends JpaRepository<Order, Long> {

    /** 이미 취소된 주문은 다시 취소되지 않는다 → 영향 행 수 0 = 이미 처리됨 */
    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("""
            update Order o
               set o.status = com.example.order.order.OrderStatus.CANCELED,
                   o.canceledAt = :now
             where o.id = :id
               and o.status = com.example.order.order.OrderStatus.PAID
            """)
    int cancelIfPaid(@Param("id") Long id, @Param("now") LocalDateTime now);
}
```

```java
// src/main/java/com/example/order/order/OrderCancelService.java
package com.example.order.order;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Clock;
import java.time.LocalDateTime;

@Service
public class OrderCancelService {

    private final OrderRepository orderRepository;
    private final Clock clock;

    public OrderCancelService(OrderRepository orderRepository, Clock clock) {
        this.orderRepository = orderRepository;
        this.clock = clock;
    }

    /** 멱등: 이미 취소된 주문을 다시 취소해도 200과 같은 결과를 돌려준다. */
    @Transactional
    public CancelResult cancel(Long orderId) {
        int affected = orderRepository.cancelIfPaid(orderId, LocalDateTime.now(clock));
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId));

        // affected == 0 이면서 이미 CANCELED → 정상(멱등 재요청)
        // affected == 0 이면서 CANCELED가 아님 → 상태 충돌(409)
        return new CancelResult(order.getId(), order.getStatus(), affected == 1);
    }
}
```

- **키 저장과 업무 처리를 같은 트랜잭션에 넣으면** 업무가 롤백될 때 키 기록도 사라져 재시도가 가능해집니다. 반대로 **응답 저장을 업무 커밋 뒤에 하면** 그 사이에 죽었을 때 재시도가 업무를 다시 실행합니다(2.6절).
- 금전 연산은 **업무 트랜잭션 안에서 키 레코드를 COMPLETED로 바꾸는 방식**이 가장 안전합니다. 키 갱신과 업무 효과가 원자적이므로 "커밋됐는데 키가 PROCESSING"인 창이 없습니다. 대신 응답 본문 저장은 커밋 후로 미루고, 재생 시 상태코드만이라도 정확히 맞춥니다.
- 위처럼 **업무 연산 자체를 조건부**로 만들어 두면, 키 기록이 유실돼도 중복 효과가 생기지 않습니다. 두 장치를 같이 두는 이유입니다.

### 2.6 실패 응답을 캐싱할지 판단하는 표

| 응답 | 저장 | 이유 |
| --- | --- | --- |
| 200 / 201 / 202 | ✅ 저장·재생 | 정상 결과. 재시도 시 같은 리소스를 가리켜야 함 |
| 204 | ✅ 저장 | 본문 없음, 상태코드만 재생 |
| 400 (JSON 파싱 오류) | ❌ 저장 안 함 | 클라이언트가 본문을 고쳐 **새 요청**으로 보내야 함 |
| 401 / 403 | ❌ 저장 안 함 | 토큰 갱신 후 같은 키로 재시도하는 것이 정상 |
| 409 (업무 충돌) | ✅ 저장 | "이미 처리됨"은 결정적 결과 |
| 422 (검증·키 재사용) | ✅ 저장(검증 실패는 ❌) | 키 재사용은 키를 바꿔야 하므로 캐싱해도 무해 |
| 429 / 503 | ❌ 저장, FAILED | 재시도가 정상 경로 |
| 500 | ❌ 저장, FAILED | 버그 수정 후 재시도 허용. 캐싱하면 영구 실패로 굳음 |
| 502 / 504 | ❌ 저장, FAILED | 게이트웨이 오류는 서버 처리 결과를 알 수 없음 |

- 5xx를 캐싱하는 순간 **일시 장애가 영구 실패로 바뀝니다.** 반대로 4xx 검증 실패를 캐싱하면 클라이언트가 본문을 고쳐도 같은 오류만 받습니다.
- 502/504는 특히 조심해야 합니다. 서버는 실제로 성공했는데 게이트웨이가 먼저 끊었을 수 있으므로, **키를 FAILED로 내려 재시도**시키고 업무 연산의 조건부 성질로 중복을 흡수합니다.

### 2.7 동시 중복 요청 — 유니크 제약이 유일한 승자 결정자

```text
스레드 A: INSERT (scope, key)  → 성공 (PROCESSING)
스레드 B: INSERT (scope, key)  → 유니크 위반 → 기존 행 조회
          ├ PROCESSING & 신선 → 409 + Retry-After: 1
          └ PROCESSING & 오래됨 → 조건부 UPDATE로 회수 시도 (영향 행 1인 쪽만 진행)
```

- **check-then-act는 반드시 뚫립니다.** "SELECT로 없음을 확인 → INSERT"는 두 트랜잭션이 모두 없음을 보고 둘 다 INSERT합니다. MySQL InnoDB에서 유니크 인덱스가 없으면 막을 방법이 없습니다.
- Redis `SET key NX PX`로 선점하면 빠르지만, **Redis가 유일한 방어선이 되면 안 됩니다.** 장애 조치·키 축출·TTL 만료 시 중복이 열립니다. Redis는 1차 속도 제한, DB 유니크는 최종 보증으로 역할을 나눕니다(11-10 노트의 캐시 원칙과 같은 구조).
- 락을 길게 잡지 않습니다. 처리 시간이 긴 API에서 409를 바로 주고 클라이언트가 폴링하게 하는 편이, 커넥션을 붙잡고 대기시키는 것보다 안전합니다.
- **409를 받은 클라이언트는 "같은 키 + 같은 본문"으로 재시도**해야 합니다. 키를 새로 만들면 중복이 됩니다.

### 2.8 PUT/PATCH/DELETE와 조건부 요청

- **PUT**: 전체 표현 교체. 부분 필드를 빠뜨리면 null로 덮여 데이터가 사라집니다. 부분 수정 의도라면 PATCH 또는 별도 하위 자원을 씁니다.
- **PATCH**: `application/merge-patch+json`이면 "없는 필드는 유지, null은 삭제"라는 규칙이 정해져 있습니다. 직접 만든 DTO라면 **null과 미전송을 구분**해야 합니다(`Optional` 래퍼, `JsonNullable`, 또는 별도 커맨드 객체).
- **DELETE**: 첫 요청 204, 두 번째 요청은 204(정책) 또는 404. REST 관점에서 404도 멱등이지만, **클라이언트가 "이미 없음"을 오류로 처리하지 않도록 204로 통일**하는 팀이 많습니다. 어느 쪽이든 문서에 고정합니다.
- **lost update 방지**: `If-Match: "<ETag>"` + 서버의 `412 Precondition Failed`, 또는 `@Version` 낙관적 락 + 409. 두 요청이 같은 버전을 읽고 각자 수정하면 하나는 반드시 실패해야 합니다. 조건부 요청은 **캐시 검증(If-None-Match → 304)** 과 같은 헤더 체계를 씁니다.

```java
// src/main/java/com/example/order/order/OrderController.java (발췌)
@PutMapping("/api/orders/{id}")
public ResponseEntity<OrderResponse> replace(@PathVariable Long id,
                                             @RequestHeader("If-Match") String etag,
                                             @RequestBody @Valid OrderReplaceRequest request) {
    OrderResponse updated = orderService.replace(id, etag, request);  // 버전 불일치 시 412
    return ResponseEntity.ok()
            .eTag(String.valueOf(updated.version()))
            .body(updated);
}
```

### 2.9 외부 PG·비동기 작업에서의 멱등성

- **PG에 우리 키를 그대로 전달**합니다. 대부분의 PG가 자체 멱등 키를 받으므로, 우리 쪽 재시도와 PG 쪽 중복 차단이 같은 키로 이어집니다.
- **타임아웃 시 재결제보다 조회가 먼저**입니다. 결제 생성이 타임아웃이면 `GET /payments?merchantOrderId=...`로 실제 상태를 확인하고, 없을 때만 같은 키로 재시도합니다. "취소 API를 무조건 호출"하는 망취소는 이미 취소된 결제에 또 취소를 걸어 오류를 만듭니다.
- **202 Accepted**로 접수만 하고 비동기 처리하는 경우, 키 → `jobId` 매핑을 남겨 같은 키 재요청이 **같은 jobId**를 받게 합니다. 새 job을 만들면 큐에 중복 작업이 쌓입니다.
- **웹훅**은 PG가 재전송하는 것이 정상입니다. 이벤트 ID를 유니크 키로 저장하고, 처리 후 200을 돌려줍니다. 200을 주지 않으면 PG가 계속 재전송합니다.
- **Kafka 소비**는 11-16 노트의 멱등 소비 패턴과 같은 문제입니다. HTTP 계층의 키는 "요청"의 중복을, 컨슈머의 키는 "메시지"의 중복을 막습니다. 두 계층 모두 필요합니다.

### 2.10 함정 체크리스트

- [ ] `(scope, key)` **유니크 인덱스**가 실제 DB에 존재하는가 (JPA `uniqueConstraints`만 믿지 말고 `SHOW CREATE TABLE`로 확인)
- [ ] 키 스코프에 **사용자 ID**가 들어 있는가 (다른 사용자의 응답 재생 차단)
- [ ] **요청 지문**을 비교하는가 (같은 키·다른 금액)
- [ ] 필터에서 본문을 읽은 뒤 **컨트롤러가 본문을 다시 읽을 수 있는가**
- [ ] `copyBodyToResponse()`를 호출하는가 (응답 본문 유실)
- [ ] 5xx를 캐싱하지 않는가 (일시 장애 → 영구 실패)
- [ ] PROCESSING 좀비를 회수하는가 (`STALE_AFTER` + 조건부 UPDATE)
- [ ] TTL 정리 잡이 **ShedLock**으로 한 노드만 도는가
- [ ] 대용량·스트리밍 응답을 버퍼링하지 않는가
- [ ] 409를 받은 클라이언트가 **같은 키로** 재시도하도록 문서화했는가
- [ ] 재생 응답에 `Content-Type`·상태코드가 함께 저장되는가 (본문만 저장하면 JSON이 text로 나감)
- [ ] 키 테이블 증가율과 재생률을 모니터링하는가 (재생률 급증 = 클라이언트 타임아웃 문제)

### 2.11 테스트로 증명하는 방법

| 시나리오 | 기대 |
| --- | --- |
| 같은 키·같은 본문 2회 (순차) | 2번째는 저장 응답 재생, **부수효과 1회** |
| 같은 키·같은 본문 2회 (동시, 스레드 20개) | 성공 1건 + 409 19건, DB 행 1건 |
| 같은 키·다른 본문 | 422 |
| 처리 중 3초 지연 + 동시 재요청 | 409 + `Retry-After` |
| 업무가 500으로 실패 후 같은 키 재시도 | 재처리되어 성공 가능 |
| 502 응답 후 같은 키 재시도 | 중복 결제 없이 1건 |
| TTL 만료 후 같은 키 | 새 요청으로 처리(중복 가능) — 문서화된 동작 |

- 동시성 테스트는 **`CountDownLatch` + `ExecutorService`** 로 실제 스레드를 겹쳐야 합니다. 순차 테스트만으로는 유니크 제약 경로를 검증할 수 없습니다.
- `@SpringBootTest` + Testcontainers MySQL을 쓰면 유니크 제약과 격리수준을 실제 DB로 검증할 수 있습니다(H2는 락·유니크 동작이 달라 대체가 안 됩니다).

---

## 3. 기술 면접 단골 꼬리 질문 & 답변 팁

- **Q. 멱등성이 무엇인가요? PUT과 POST의 차이로 설명해 보세요.**
  - **A.** "같은 요청을 여러 번 보내도 서버 상태가 한 번 보낸 것과 같은 성질입니다. 응답이 같은 것은 아닙니다 — 두 번째 DELETE가 404여도 상태는 같습니다. PUT은 리소스 전체를 주어진 표현으로 덮어쓰므로 멱등이고, POST는 서버가 새 식별자를 만들어 리소스를 추가하므로 멱등이 아닙니다. 그래서 결제·주문처럼 POST로 생성하는 API에는 별도 멱등 키가 필요합니다."

- **Q. 타임아웃이 나면 클라이언트는 어떻게 해야 하나요?**
  - **A.** "타임아웃은 실패가 아니라 결과 미확인입니다. 무작정 재시도하면 중복 결제가 됩니다. 같은 Idempotency-Key로 재시도하고, 서버는 저장된 응답을 재생해야 합니다. 결제처럼 외부 PG가 끼면 재시도 전에 조회 API로 실제 상태를 확인하는 것이 우선입니다."

- **Q. Idempotency-Key를 서버에서 어떻게 구현하나요?**
  - **A.** "(scope, key) 유니크 인덱스를 가진 테이블에 PROCESSING으로 INSERT해 선점하고, 업무 처리 후 상태코드·본문을 저장합니다. 같은 키가 오면 요청 지문을 비교해 같으면 응답을 재생, 다르면 422, 처리 중이면 409를 돌려줍니다. 핵심은 **조회 후 삽입(check-then-act)이 아니라 유니크 제약으로 승자를 결정**하는 것입니다."

- **Q. Redis에 키를 저장하면 안 되나요?**
  - **A.** "빠른 1차 차단으로는 좋지만 최종 보증이 될 수 없습니다. Redis 장애·축출·TTL 만료 시 중복이 열리고, 장애 조치 중 쓰기 유실도 가능합니다. DB 유니크 제약을 최종 방어선으로 두고 Redis는 앞단에서 부하를 줄이는 역할로 씁니다."

- **Q. 키를 같은데 본문이 다르면 어떻게 처리해야 하나요?**
  - **A.** "422로 거부해야 합니다. 그냥 첫 응답을 재생하면 클라이언트는 다른 요청의 결과를 받고도 성공으로 오인합니다. 그래서 요청 지문(method + path + body 해시)을 저장하고 비교합니다. 지문이 없으면 멱등 키가 오히려 버그를 숨깁니다."

- **Q. 5xx 응답도 캐싱해야 하나요?**
  - **A.** "아니요. 500을 저장하면 일시적 버그가 그 키의 영구 실패로 굳습니다. 5xx는 FAILED로 두고 재시도를 허용합니다. 반대로 4xx 검증 실패는 캐싱하지 않아 클라이언트가 본문을 고쳐 보낼 수 있게 하고, 409 같은 결정적 결과는 저장합니다."

- **Q. 동시에 같은 키로 두 요청이 오면?**
  - **A.** "유니크 인덱스 INSERT에서 한 쪽만 성공합니다. 진 쪽은 기존 행을 읽어 처리 중이면 409와 Retry-After를 주고, 완료됐으면 응답을 재생합니다. 앞선 요청이 죽어 PROCESSING으로 남으면, 타임스탬프 조건을 건 조건부 UPDATE로 한 노드만 회수합니다."

- **Q. 트랜잭션 경계는 어떻게 잡나요?**
  - **A.** "키 선점은 REQUIRES_NEW로 분리해야 합니다. 같은 트랜잭션에서 유니크 위반을 catch하면 rollback-only가 되어 이후 작업이 터집니다. 금전 연산은 업무 트랜잭션 안에서 키를 COMPLETED로 바꿔 원자성을 확보하고, 응답 본문 저장은 커밋 후로 미룹니다. 그래도 창이 남으므로 업무 연산 자체를 조건부 상태 전이로 만들어 이중 방어합니다."

- **Q. REST API에서 멱등성을 보장하려면 어떤 설계를 하나요?**
  - **A.** "세 층으로 겹칩니다. 자연 멱등 연산(PUT·DELETE·절대값 PATCH)을 기본으로 쓰고, 상태 전이는 WHERE 조건을 건 UPDATE로 만들어 영향 행 수로 결과를 판정하며, 매번 새 결과를 만드는 생성 API에만 Idempotency-Key를 붙입니다."

- **Q. PATCH는 멱등인가요?**
  - **A.** "정의상 아닙니다. 증분 연산(add, increment)이 가능하기 때문입니다. 다만 실무에서는 절대값 대입으로 설계해 사실상 멱등하게 만듭니다. JSON Merge Patch도 '없는 필드는 유지'라서 같은 문서를 두 번 적용하면 결과가 같습니다."

- **Q. 409와 422 중 무엇을 써야 하나요?**
  - **A.** "409는 현재 리소스 상태와 요청이 충돌할 때, 즉 같은 키가 처리 중이라 지금은 처리할 수 없을 때입니다. 422는 문법은 맞지만 의미상 처리할 수 없을 때로, 키를 다른 요청에 재사용한 경우가 여기 해당합니다. 400은 문법 오류에 씁니다. 이 구분이 있어야 클라이언트가 재시도 가능 여부를 판단할 수 있습니다."

- **Q. 낙관적 락과 멱등성은 같은 문제인가요?**
  - **A.** "다릅니다. 낙관적 락(@Version, If-Match)은 동시에 같은 리소스를 수정하는 lost update를 막고, 멱등성은 같은 요청의 반복 적용을 흡수합니다. 다만 조건부 요청(If-Match → 412)은 둘을 동시에 만족시키는 수단이라 함께 설계하는 경우가 많습니다."

---

## 참고

- RFC 9110 §9.2.1 Safe Methods / §9.2.2 Idempotent Methods: https://www.rfc-editor.org/rfc/rfc9110#section-9.2.2
- Idempotency-Key HTTP Header Field (IETF draft): https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
- RFC 6585 (428 Precondition Required, 429 Too Many Requests): https://www.rfc-editor.org/rfc/rfc6585
- Stripe — Idempotent requests: https://docs.stripe.com/api/idempotent_requests
- Spring `OncePerRequestFilter`: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/filter/OncePerRequestFilter.html
- Spring `ContentCachingResponseWrapper`: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/util/ContentCachingResponseWrapper.html
- MySQL `INSERT ... ON DUPLICATE KEY UPDATE`: https://dev.mysql.com/doc/refman/8.0/en/insert-on-duplicate.html
