# [2026-11-09] MySQL InnoDB MVCC(다중 버전 동시성 제어) 동작 원리와 트랜잭션 격리 수준(Isolation Level)

> **한 줄 핵심 요약**: InnoDB는 데이터를 **덮어쓰기 전에 이전 버전을 undo log에 남기고**, 각 row에 숨은 컬럼 `DB_TRX_ID`(마지막으로 수정한 트랜잭션 ID)와 `DB_ROLL_PTR`(undo log 포인터)를 붙여 두었다가, 읽기 시점의 **Read View**(활성 트랜잭션 목록 스냅샷)와 비교해 "내가 봐야 할 버전"까지 undo를 거슬러 올라가 복원한다. 이 구조 덕분에 **읽기는 락을 잡지 않고(비잠금 일관성 읽기), 쓰기는 읽기를 막지 않는다.** 격리 수준의 차이는 결국 **Read View를 언제 만드는가**(READ COMMITTED = 문장마다, REPEATABLE READ = 트랜잭션의 첫 일관성 읽기 한 번) 하나로 정리되고, 그 대가로 RC는 팬텀에 가까워지고 RR은 갭 락으로 교착이 늘어난다. 실무 사고의 대부분은 "MVCC가 있으니 동시성 문제는 DB가 해결해 준다"는 오해 — 특히 **RR에서도 애플리케이션 레벨 lost update는 그대로 발생한다** — 에서 나온다.

---

## 0. 재현 예제 (MySQL 8.0, 세션 2개로 직접 확인)

### 0-1. 실습 스키마와 준비

```sql
-- 파일 경로: src/main/resources/db/migration/V1__product.sql
-- InnoDB, 기본 격리 수준은 REPEATABLE READ (default_transaction_isolation)
CREATE TABLE product (
    id          BIGINT      NOT NULL AUTO_INCREMENT,
    category    VARCHAR(20) NOT NULL,
    stock       INT         NOT NULL,          -- 재고: 동시성 이슈의 단골
    price       INT         NOT NULL,
    updated_at  DATETIME(6) NOT NULL,
    PRIMARY KEY (id),
    KEY idx_category (category)
) ENGINE = InnoDB;

INSERT INTO product (category, stock, price, updated_at) VALUES
    ('A', 10, 1000, NOW(6)),
    ('A', 10, 2000, NOW(6)),
    ('A', 10, 3000, NOW(6)),
    ('B', 10, 4000, NOW(6));
```

```sql
-- 현재 세션/전역 격리 수준 확인
SELECT @@transaction_isolation, @@global.transaction_isolation;   -- REPEATABLE-READ
```

준비: 세션 A·B를 각각 다른 커넥션(MySQL CLI 두 개, 또는 DBeaver 두 탭)으로 연다. **같은 커넥션에서 두 세션을 흉내 내면 아무것도 재현되지 않는다.**

### 0-2. REPEATABLE READ: 같은 SELECT를 두 번 해도 값이 바뀌지 않는다

```sql
-- ── 세션 A ─────────────────────────────────────────────
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT stock FROM product WHERE id = 1;      -- 결과: 10
                                             -- ★ 이 순간 Read View가 생성된다 (START TRANSACTION 시점이 아니다)

-- ── 세션 B ─────────────────────────────────────────────
UPDATE product SET stock = 5, updated_at = NOW(6) WHERE id = 1;
SELECT stock FROM product WHERE id = 1;      -- 결과: 5 (자기 변경은 자기 눈에 보인다)
COMMIT;

-- ── 세션 A ─────────────────────────────────────────────
SELECT stock FROM product WHERE id = 1;      -- 결과: 여전히 10  ← 스냅샷 읽기 (undo로 이전 버전 복원)
SELECT stock FROM product WHERE id = 1 FOR UPDATE;  -- 결과: 5   ← 잠금 읽기(현재 읽기)
COMMIT;

-- 커밋 후 다시 조회
SELECT stock FROM product WHERE id = 1;      -- 결과: 5 (새 트랜잭션 = 새 Read View)
```

핵심은 마지막 두 줄의 대비다. **같은 트랜잭션 안에서, 같은 row에 대해, 일반 `SELECT`는 10을, `SELECT ... FOR UPDATE`는 5를 돌려준다.** MVCC는 "읽기용 스냅샷"과 "쓰기 직전의 최신 확인"을 분리한 구조이지, 트랜잭션 전체가 과거에 갇히는 구조가 아니다.

### 0-3. READ COMMITTED: 문장마다 새 스냅샷

```sql
-- ── 세션 A ─────────────────────────────────────────────
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT stock FROM product WHERE id = 1;      -- 결과: 5

-- ── 세션 B ─────────────────────────────────────────────
UPDATE product SET stock = 3, updated_at = NOW(6) WHERE id = 1;
COMMIT;

-- ── 세션 A ─────────────────────────────────────────────
SELECT stock FROM product WHERE id = 1;      -- 결과: 3  ← 문장마다 Read View를 새로 만든다
COMMIT;
```

두 격리 수준의 차이는 **Read View 생성 시점** 하나뿐이다. 데이터 구조·undo log·숨은 컬럼은 완전히 동일하다.

### 0-4. 팬텀 리드: 스냅샷 읽기에서는 안 보이지만, DML은 본다

```sql
-- ── 세션 A (REPEATABLE READ) ───────────────────────────
START TRANSACTION;
SELECT COUNT(*) FROM product WHERE category = 'A';   -- 결과: 3 (Read View 고정)

-- ── 세션 B ─────────────────────────────────────────────
INSERT INTO product (category, stock, price, updated_at) VALUES ('A', 10, 5000, NOW(6));
COMMIT;

-- ── 세션 A ─────────────────────────────────────────────
SELECT COUNT(*) FROM product WHERE category = 'A';   -- 결과: 3  ← 스냅샷이라 팬텀 없음
UPDATE product SET price = price + 100 WHERE category = 'A';
-- 결과: Rows matched: 4, Changed: 4   ← ★ DML은 현재 읽기라 방금 INSERT된 row까지 잡는다
COMMIT;
```

이것이 "InnoDB의 RR은 팬텀 리드를 막는다"는 말이 **일관성 읽기에 한정된 이야기**인 이유다. `UPDATE`/`DELETE`/`FOR UPDATE`는 항상 최신 커밋 데이터를 기준으로 대상을 찾는다. 같은 `WHERE category = 'A'`가 `SELECT`에서는 3건, `UPDATE`에서는 4건을 대상으로 삼는다.

### 0-5. 갭 락과 교착: RR에서만 생기는 대표 패턴

```sql
-- ── 세션 A ─────────────────────────────────────────────
START TRANSACTION;
SELECT * FROM product WHERE id = 5 FOR UPDATE;     -- id=5 없음 → 갭 (1, 4] 또는 (4, ...) 구간에 갭 락

-- ── 세션 B ─────────────────────────────────────────────
START TRANSACTION;
SELECT * FROM product WHERE id = 10 FOR UPDATE;    -- 다른 갭에 갭 락

-- ── 세션 A ─────────────────────────────────────────────
INSERT INTO product (id, category, stock, price, updated_at) VALUES (6, 'C', 1, 100, NOW(6));
-- 대기: 6이 B가 잡은 갭 안에 있다

-- ── 세션 B ─────────────────────────────────────────────
INSERT INTO product (id, category, stock, price, updated_at) VALUES (9, 'C', 1, 100, NOW(6));
-- ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

두 트랜잭션이 각자 갭을 잡고, 상대 갭 안에 INSERT하려다 서로를 기다린다. **같은 코드가 READ COMMITTED에서는 갭 락이 비활성이라 교착 없이 지나간다.** "RR은 교착이 잦다"는 말의 실체가 이 지점이다.

### 0-6. 손에 잡히는 관측: 활성 트랜잭션 · 락 · Read View

```sql
-- 1) 지금 열려 있는 트랜잭션과 그 격리 수준
SELECT trx_id, trx_state, trx_started, TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS age_sec,
       trx_mysql_thread_id, trx_isolation_level, trx_rows_locked, trx_rows_modified
FROM information_schema.INNODB_TRX
ORDER BY trx_started;

-- 2) 누가 누구를 기다리는가 (MySQL 8.0)
SELECT * FROM performance_schema.data_lock_waits;
SELECT ENGINE_TRANSACTION_ID, OBJECT_NAME, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_STATUS, LOCK_DATA
FROM performance_schema.data_locks;

-- 3) undo 로그 적체(긴 트랜잭션의 흔적)와 Read View
SHOW ENGINE INNODB STATUS\G
-- TRANSACTIONS 섹션에서 확인할 줄:
--   Trx id counter 12345
--   Purge done for trx's n:o < 12300 undo n:o < 0
--   History list length 821                       ← undo 적체 길이
--   Trx read view will not see trx with id >= 12310, sees < 12301
--   ↑ max_trx_id(12310) / min_trx_id(12301) — 이 트랜잭션이 undo를 붙잡고 있는 범위
```

`History list length`가 계속 증가하면 **아무것도 안 하는 것처럼 보이는 조회 트랜잭션 하나가 undo purge를 막고 있다**는 뜻이다. 2-2에서 다룬다.

### 0-7. Spring에서 격리 수준을 다루는 코드

```yaml
# 파일 경로: src/main/resources/application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/study?serverTimezone=Asia/Seoul&characterEncoding=UTF-8
    username: study
    password: study
    hikari:
      maximum-pool-size: 10
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
    properties:
      hibernate:
        # SQL 로그에 트랜잭션/커넥션 정보를 붙여 격리 수준을 눈으로 확인할 때만 켠다
        session.events.log.LOG_QUERIES_SLOWER_THAN_MS: 100
logging:
  level:
    org.springframework.transaction.interceptor: TRACE
    org.hibernate.engine.transaction.internal: DEBUG
```

```java
// 파일 경로: src/main/java/study/product/domain/Product.java
package study.product.domain;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import jakarta.persistence.Version;
import java.time.LocalDateTime;

@Entity
@Table(name = "product")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 20)
    private String category;

    @Column(nullable = false)
    private int stock;

    @Column(nullable = false)
    private int price;

    @Column(nullable = false)
    private LocalDateTime updatedAt;

    /**
     * 낙관적 락용 버전 컬럼. MVCC의 Read View와는 무관한 "애플리케이션 레벨 충돌 감지" 장치다.
     * (2-1의 RR lost update를 막는 수단 중 하나)
     */
    @Version
    private Long version;

    protected Product() {
    }

    public Product(String category, int stock, int price) {
        this.category = category;
        this.stock = stock;
        this.price = price;
        this.updatedAt = LocalDateTime.now();
    }

    /** 도메인 레벨 차감: 음수 재고를 만들지 않는다. 단, 이 검증만으로는 동시성을 보장하지 못한다. */
    public void decrease(int quantity) {
        if (this.stock < quantity) {
            throw new IllegalStateException("재고 부족: " + this.stock + " < " + quantity);
        }
        this.stock -= quantity;
        this.updatedAt = LocalDateTime.now();
    }

    public Long getId() {
        return id;
    }

    public int getStock() {
        return stock;
    }

    public Long getVersion() {
        return version;
    }
}
```

```java
// 파일 경로: src/main/java/study/product/repository/ProductRepository.java
package study.product.repository;

import jakarta.persistence.LockModeType;
import java.util.Optional;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import study.product.domain.Product;

public interface ProductRepository extends JpaRepository<Product, Long> {

    /**
     * 비관적 락 (MySQL: SELECT ... FOR UPDATE). 현재 읽기이므로 최신 커밋 값을 본다.
     * 같은 row를 노리는 다른 트랜잭션은 커밋/롤백까지 대기한다 → 처리량은 줄고 정합성은 올라간다.
     */
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select p from Product p where p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);

    /**
     * 조건부 원자 UPDATE. 가장 저렴하고 안전한 재고 차감 방식.
     * "읽고 → 계산 → 쓰기"를 하지 않고 DB가 한 문장으로 판정·차감한다.
     * 반환값이 0이면 재고 부족(또는 row 없음) → 호출부에서 예외 처리.
     */
    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("""
            update Product p
               set p.stock = p.stock - :quantity,
                   p.updatedAt = CURRENT_TIMESTAMP
             where p.id = :id
               and p.stock >= :quantity
            """)
    int decreaseStock(@Param("id") Long id, @Param("quantity") int quantity);
}
```

```java
// 파일 경로: src/main/java/study/product/service/StockService.java
package study.product.service;

import java.util.NoSuchElementException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;
import study.product.domain.Product;
import study.product.repository.ProductRepository;

@Service
public class StockService {

    private final ProductRepository productRepository;

    public StockService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    /**
     * ❌ 가장 흔한 안티패턴: 스냅샷 읽기로 재고를 읽고, 계산해서, 다시 쓴다.
     *    RR에서도 두 요청이 동시에 들어오면 둘 다 stock=10을 읽고 각자 9를 써서
     *    최종 재고가 8이 아니라 9가 된다 (lost update). 2-1 참고.
     */
    @Transactional
    public void decreaseNaive(Long productId, int quantity) {
        Product product = productRepository.findById(productId)
                .orElseThrow(() -> new NoSuchElementException("상품 없음: " + productId));
        product.decrease(quantity);   // ★ 읽은 시점의 값을 기준으로 계산 → 덮어쓰기
    }

    /**
     * ✅ 조건부 원자 UPDATE: DB 한 문장 안에서 판정과 차감이 끝난다.
     *    영향 행 수 0 → 재고 부족으로 해석. 별도 락·재시도가 필요 없다.
     */
    @Transactional
    public void decreaseConditional(Long productId, int quantity) {
        int updated = productRepository.decreaseStock(productId, quantity);
        if (updated == 0) {
            throw new IllegalStateException("재고 부족 또는 상품 없음: " + productId);
        }
    }

    /**
     * ✅ 비관적 락: FOR UPDATE로 현재 읽기 + row 락을 잡고, 그 상태에서 엔티티를 수정한다.
     *    락 대기가 길어질 수 있으므로 짧은 트랜잭션에서만 사용한다.
     */
    @Transactional
    public void decreasePessimistic(Long productId, int quantity) {
        Product product = productRepository.findByIdForUpdate(productId)
                .orElseThrow(() -> new NoSuchElementException("상품 없음: " + productId));
        product.decrease(quantity);
    }

    /**
     * 격리 수준을 낮추는 예: 갭 락을 줄이고 싶을 때.
     * 주의) REQUIRED + 이미 진행 중인 트랜잭션에 참여하는 경우 이 isolation은 무시된다(2-5).
     */
    @Transactional(isolation = Isolation.READ_COMMITTED)
    public void decreaseReadCommitted(Long productId, int quantity) {
        int updated = productRepository.decreaseStock(productId, quantity);
        if (updated == 0) {
            throw new IllegalStateException("재고 부족: " + productId);
        }
    }
}
```

```java
// 파일 경로: src/test/java/study/product/IsolationScenarioTest.java
// 두 스레드로 "동시에 읽고 쓰기"를 재현한다. 실제 재현율을 높이려면 두 스레드의 시작을 배리어로 맞춘다.
package study.product;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import study.product.service.StockService;

@SpringBootTest
class IsolationScenarioTest {

    @Autowired
    private StockService stockService;

    @Test
    @DisplayName("조건부 UPDATE는 동시 10건 차감에서도 재고가 정확히 줄어든다")
    void conditionalUpdateIsSafe() throws Exception {
        // given: 재고 10인 상품 1건
        long productId = 1L;

        int threads = 10;
        ExecutorService pool = Executors.newFixedThreadPool(threads);
        CountDownLatch start = new CountDownLatch(1);
        CountDownLatch done = new CountDownLatch(threads);

        for (int i = 0; i < threads; i++) {
            pool.submit(() -> {
                try {
                    start.await();
                    stockService.decreaseConditional(productId, 1);
                } catch (Exception ignored) {
                    // 재고 부족 예외는 정상 시나리오(2-1의 반환값 0 처리)
                } finally {
                    done.countDown();
                }
                return null;
            });
        }

        start.countDown();               // 동시에 출발
        assertThat(done.await(10, TimeUnit.SECONDS)).isTrue();
        pool.shutdown();

        // then: 성공 10건 → 재고 0. (naive 방식으로 바꾸면 9~10 사이 어딘가로 어긋난다)
    }
}
```

---

## 1. 개념 및 동작 원리

### 1-1. 격리 수준과 이상 현상: 표준 SQL과 InnoDB의 실제 동작은 다르다

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read | InnoDB 구현 방식 |
|---|---|---|---|---|
| READ UNCOMMITTED | 발생 | 발생 | 발생 | MVCC를 쓰지 않고 **가장 최신 버전(커밋 안 된 것 포함)** 을 그대로 읽음 |
| READ COMMITTED | 없음 | 발생 | 발생 | **문장마다** Read View 생성 |
| REPEATABLE READ (InnoDB 기본) | 없음 | 없음 | **일관성 읽기에서는 없음**(잠금 읽기에서는 발생 가능) | **트랜잭션 첫 일관성 읽기에서 한 번** Read View 생성 + 갭 락 |
| SERIALIZABLE | 없음 | 없음 | 없음 | (autocommit이 아닐 때) 모든 일반 `SELECT`를 `SELECT ... FOR SHARE`로 암묵 변환 |

세 가지 이상 현상을 먼저 정확히 구분한다.

- **Dirty Read**: 다른 트랜잭션이 **아직 커밋하지 않은** 변경을 읽는 것. 그 트랜잭션이 롤백하면 읽은 값은 존재한 적이 없는 값이 된다.
- **Non-Repeatable Read**: 같은 트랜잭션 안에서 같은 row를 두 번 읽었는데 **값이 달라지는 것**(중간에 다른 트랜잭션이 UPDATE + COMMIT).
- **Phantom Read**: 같은 범위 조건으로 두 번 읽었는데 **행의 개수가 달라지는 것**(중간에 다른 트랜잭션이 INSERT + COMMIT).

주의할 점이 두 가지다.

1. **표준 SQL은 RR에서 팬텀이 발생한다고 정의**하지만, InnoDB의 RR은 갭 락(next-key lock) 덕분에 **잠금 읽기 기준으로는 팬텀을 막는다.** 대신 "1-6에서 보듯 DML은 현재 읽기"라서, `SELECT` 결과를 믿고 `UPDATE ... WHERE 범위`를 실행하면 스냅샷에 없던 row가 갱신되는 형태의 팬텀이 남는다.
2. **성능 문제로 READ COMMITTED를 쓰는 회사가 많다.** 갭 락이 사라져 교착이 줄고, undo를 오래 붙잡지 않는다. 대가는 "같은 트랜잭션 안에서 읽은 값이 바뀔 수 있다"는 전제를 애플리케이션이 감당해야 한다는 것이다. 어느 쪽을 고르든 **격리 수준은 정합성 보증의 상한이지, 비즈니스 규칙의 대체물이 아니다.**

### 1-2. MVCC가 필요한 이유: 읽기와 쓰기가 서로를 막지 않게

MVCC가 없으면 두 가지 방법밖에 없다.

```text
[락 기반 동시성 — MVCC 없음]
읽기: S-Lock(공유 락) → 다른 읽기는 허용, 쓰기는 대기
쓰기: X-Lock(배타 락) → 모든 읽기·쓰기 대기

→ 조회 트래픽이 많은 서비스에서 읽기가 쓰기를, 쓰기가 읽기를 계속 막는다.
→ 격리를 올릴수록 동시성이 급락한다. (전형적인 "읽기 많은 서비스가 느려짐")
```

```text
[MVCC — InnoDB]
읽기(일관성 읽기): 락 없음. undo log로 "내 스냅샷 시점의 버전"을 복원해서 읽는다.
쓰기: 필요한 row에만 X-Lock. 읽기와 충돌하지 않는다.

→ 읽기는 항상 논블로킹, 쓰기끼리만 경합한다.
→ 대신 "이전 버전을 보관하는 비용"(undo log, purge)을 지불한다.
```

즉 MVCC는 **공짜 최적화가 아니라 저장 공간·purge 스레드·복잡도를 읽기 동시성과 맞바꾼 설계**다. 그래서 "MVCC를 지원하는 DB라서 읽기가 안전하다"가 아니라, **"읽기는 스냅샷을 보고, 쓰기는 최신을 본다"** 는 비대칭을 이해해야 한다.

### 1-3. 숨은 컬럼 3개: row마다 붙는 MVCC의 좌표

InnoDB의 모든 row에는 사용자가 정의하지 않은 컬럼이 붙는다.

| 숨은 컬럼 | 크기 | 의미 | 용도 |
|---|---|---|---|
| `DB_TRX_ID` | 6바이트 | 이 row를 **마지막으로 INSERT/UPDATE한 트랜잭션 ID** | Read View와 비교해 "이 버전이 내 스냅샷에 보이는가"를 판정 |
| `DB_ROLL_PTR` | 7바이트 | undo log의 **직전 버전 레코드**를 가리키는 포인터 | 안 보이면 이 포인터를 따라 이전 버전을 찾아간다 |
| `DB_ROW_ID` | 6바이트 | PK가 없을 때 InnoDB가 만드는 **숨은 클러스터드 키** | PK가 있으면 생성되지 않는다 |

```text
[클러스터드 인덱스 리프의 row — 사용자 컬럼 + 숨은 컬럼]

 row(id=1) : [ id=1 | stock=5 | price=1000 | DB_TRX_ID=900 | DB_ROLL_PTR=→① ]
                                                            │
                                    ┌───────────────────────┘
                                    ▼
        [undo log ①]  이전 버전 이미지: stock=10, price=1000, DB_TRX_ID=850
                                    │
                                    ▼
        [undo log ②]  이전 버전 이미지: stock=10, price=900,  DB_TRX_ID=700
                                    │
                                    ▼
                              (INSERT 시점의 undo → 체인의 끝)
```

- **UPDATE는 "원본을 고쳐 쓰기"가 아니라 "새 버전을 만들고, 헤더에 이전 버전 포인터를 다는 것"** 이다. 그래서 같은 row를 100번 수정하면 undo 체인이 100단계가 된다.
- `DB_TRX_ID`는 6바이트라 표현 범위가 약 2^48이다. 내부적으로는 8바이트 트랜잭션 ID를 쓰고 row에는 하위 48비트만 저장한다. 그래서 트랜잭션 ID가 계속 증가하는 장기 운영 DB에서는 **과거 버전과 현재 버전의 ID 비교가 모호해질 수 있고, 이를 막기 위해 MySQL은 `trx_id` 랩어라운드 시점에 안전하게 처리(강제 종료·오프라인)한다.** 신입 면접에서 이 지점까지 요구되지는 않지만, "왜 6바이트인가"라는 질문의 답은 여기에 있다.
- `DB_ROLL_PTR`이 7바이트인 이유는 undo 레코드의 **테이블스페이스 ID + 페이지 번호 + 레코드 오프셋**을 한 워드에 담기 위해서다.

### 1-4. undo log의 두 종류: insert undo와 update undo

| 종류 | 생성 시점 | 커밋 직후 | MVCC 관여 |
|---|---|---|---|
| **Insert undo** | `INSERT` | **즉시 버려도 됨** (롤백에만 필요) | 관여하지 않는다 (INSERT된 버전은 커밋 전까지 아무에게도 안 보임) |
| **Update undo** | `UPDATE` / `DELETE` | **보관** | 일관성 읽기와 롤백 양쪽에 필요. **아무도 안 볼 때까지** purge가 정리 |

- `DELETE`도 실제로는 **delete-mark**(`deleted_flag = 1`)만 세우고 row를 남긴다. 그래서 "삭제된 row를 스냅샷 시점에는 살아 있던 것으로 보여줘야" 하는 읽기가 가능하다. 실제 물리 삭제는 purge 스레드의 몫이다.
- **purge 스레드**는 "현재 존재하는 가장 오래된 Read View"보다 과거의 undo만 지울 수 있다. 그래서 **오래 열려 있는 트랜잭션 하나가 전체 undo 정리를 막는다**(2-2). 이 한 줄이 "긴 트랜잭션이 왜 위험한가"의 전부다.

```text
[purge가 막히는 구조]

T1 ── 시작 ──(8분째 살아있음, Read View 고정: max_trx_id = 1000)──────────────▶
T2 ── 시작/종료 (trx 1005)  ─ undo 필요 없음, 하지만 지울 수 없다
T3 ── 시작/종료 (trx 1006)  ─ 위와 동일

purge 입장: "T1이 trx 1000 이전 버전을 볼 수 있으므로 그 이후 undo는 못 지운다"
→ History list length 증가 → undo 테이블스페이스 팽창 → 디스크·버퍼 풀 압박
```

### 1-5. Read View: 네 개의 값으로 가시성을 판정한다

Read View는 "스냅샷을 만든 시점에 **아직 커밋되지 않은** 트랜잭션 ID 목록"이다.

| 필드 | 의미 |
|---|---|
| `m_ids` | Read View 생성 시점에 **활성(미커밋) 상태였던 트랜잭션 ID 집합** |
| `min_trx_id` | `m_ids` 중 최솟값 (소스 코드의 `up_limit_id`) |
| `max_trx_id` | 생성 시점에 **다음에 부여될** 트랜잭션 ID (= 현재 최대 커밋 ID + 1, 소스의 `low_limit_id`) |
| `creator_trx_id` | 이 Read View를 만든 트랜잭션 자신의 ID |

row 버전의 `DB_TRX_ID = trx_id`에 대한 판정 순서는 다음과 같다.

```text
1) trx_id == creator_trx_id
   → 내가 쓴 변경이다. 보인다. (자기 변경은 항상 보인다)

2) trx_id < min_trx_id
   → Read View 생성 전에 "이미 커밋이 끝난" 트랜잭션이다. 보인다.

3) trx_id >= max_trx_id
   → Read View 생성 후에 시작된 트랜잭션이다. 안 보인다.
     (미래의 트랜잭션이 만든 버전)

4) min_trx_id <= trx_id < max_trx_id
   → m_ids 에 포함되어 있으면 "그때 아직 살아 있던 트랜잭션" → 안 보인다.
   → 포함되어 있지 않으면 "그때 이미 커밋된 트랜잭션"     → 보인다.

안 보이면? DB_ROLL_PTR이 가리키는 undo 레코드로 이동해 위 판정을 반복한다.
체인 끝(INSERT 시점)까지 가도 안 보이면 → "내 스냅샷 시점에는 이 row가 없었다" → 결과에서 제외.
```

이 판정이 **row 단위로, 읽는 순간마다** 일어난다는 점이 중요하다. 트랜잭션 단위로 "테이블을 복사해 두는" 방식이 아니라서, 수천만 건 테이블에서도 스냅샷 읽기가 가능하다. 대신 각 row가 "체인을 몇 단계 거슬러 올라가야 하는지"가 달라 성능이 들쭉날쭉해질 수 있다(2-2의 undo 적체가 여기에 영향을 준다).

### 1-6. 일관성 읽기(Consistent Read) vs 잠금 읽기(Locking Read)

| 구분 | 일반 `SELECT` | `SELECT ... FOR UPDATE` / `FOR SHARE` | `UPDATE` / `DELETE` |
|---|---|---|---|
| 읽는 데이터 | **Read View 시점의 버전** | **최신 커밋 버전** (현재 읽기) | **최신 커밋 버전** |
| 락 | 없음 | X-Lock / S-Lock (+ RR에서는 갭/넥스트키 락) | X-Lock (+ 갭/넥스트키 락) |
| 다른 트랜잭션을 막는가 | 막지 않는다 | 막는다 (대기 발생) | 막는다 |
| 팬텀 | 스냅샷 기준이므로 없음 | RR에서는 갭 락으로 방지 | 범위 조건이면 새로 들어온 row도 대상 |
| Read View 생성 | RC: 문장마다 / RR: 트랜잭션 첫 일관성 읽기 | 생성하지 않음 | 생성하지 않음 |

여기서 실무적으로 가장 중요한 줄은 **마지막 행**이다. `UPDATE`/`DELETE`는 Read View를 만들지도, 참조하지도 않는다. 그래서 다음 코드가 위험해진다.

```java
// 위험한 패턴: 스냅샷으로 판단하고, 현재 읽기로 실행한다
@Transactional   // REPEATABLE READ
public void cancelIfNoItems(Long orderId) {
    int itemCount = itemRepository.countByOrderId(orderId);   // ① 일관성 읽기 (스냅샷)
    if (itemCount == 0) {
        orderRepository.deleteById(orderId);                  // ② 현재 읽기 (최신 기준)
    }
}
```

①과 ② 사이에 다른 트랜잭션이 항목을 추가하면, "항목이 0개라고 판단했는데 삭제는 실행되는" 상태가 된다. **판단 근거와 실행 대상이 서로 다른 시점을 보는 구조**이므로, 판단이 필요하면 같은 트랜잭션 안에서 `FOR UPDATE`로 현재 읽기를 하거나, `DELETE ... WHERE NOT EXISTS (...)`처럼 **한 문장으로 판단과 실행을 합쳐야** 한다.

### 1-7. 갭 락 / 넥스트키 락: RR이 팬텀을 막는 방법

```text
인덱스 리프:  [ 1 ]      [ 5 ]        [ 10 ]
갭:        (-∞,1)  (1,5)       (5,10)        (10,+∞)

Record Lock : id = 5 인 레코드만 잠금
Gap Lock    : (1,5) 구간만 잠금 (레코드 자체는 안 잠금)
Next-Key Lock: Record Lock + 그 레코드 앞의 Gap Lock → (1,5] 형태
Insert Intention Lock: INSERT 직전에 "이 갭에 넣겠다"는 의도 락 (충돌 시에만 대기)
```

- **갭 락의 목적은 "그 범위에 새 row가 들어오지 못하게" 하는 것**이다. 팬텀을 막는 수단이 락이지 MVCC가 아니다. MVCC는 이미 존재하는 row의 버전만 다룰 수 있다.
- **갭 락은 READ COMMITTED에서 비활성**이다(외래키 제약 검사·중복키 검사에서만 사용). 이것이 RC가 교착에 강한 이유이자, RC에서 팬텀이 나타나는 이유다.
- **갭 락은 "인덱스 레코드 사이"에 걸린다.** 인덱스가 없으면 스캔한 전체 범위(사실상 테이블 전체)에 락이 걸려 동시성이 붕괴한다. "인덱스 없는 컬럼으로 `FOR UPDATE`를 하면 테이블이 잠긴다"는 말이 여기서 나온다.
- **유니크 인덱스의 동등 조건**은 갭 락이 필요 없다(`WHERE id = 5 FOR UPDATE`는 record lock만). 정확히 한 행만 존재할 수 있으므로 팬텀이 생길 여지가 없기 때문이다. 반면 범위 조건(`BETWEEN`, `>`)이나 비유니크 인덱스는 넥스트키 락이 걸린다.
- `SERIALIZABLE`은 (autocommit이 아닌 트랜잭션에서) 모든 일반 `SELECT`를 `FOR SHARE`로 암묵 변환한다. 읽기가 락을 잡는 순간 MVCC의 이점(읽기 논블로킹)이 사라지므로, 실무에서는 거의 쓰지 않고 **필요한 지점만 `FOR UPDATE`/`FOR SHARE`를 명시적으로 거는 방식**을 택한다.

### 1-8. 정리: 격리 수준 선택 매트릭스

| 상황 | 권장 | 이유 |
|---|---|---|
| 기본값 그대로 두는 일반 웹 서비스 | REPEATABLE READ | 한 트랜잭션 안에서 읽은 값이 바뀌지 않아 코드가 단순해진다 |
| 교착이 잦은 고동시성 쓰기 구간 | READ COMMITTED (또는 조건부 UPDATE로 회피) | 갭 락 비활성 → 교착 감소. 단 `binlog_format=ROW` 전제 |
| 재고·포인트·쿠폰 차감 | **격리 수준에 의존하지 않는다** | 조건부 UPDATE / `FOR UPDATE` / 낙관적 락으로 명시적으로 처리 (2-1) |
| 통계·리포트용 대량 조회 | REPEATABLE READ + `START TRANSACTION WITH CONSISTENT SNAPSHOT` | 조회 중 값이 흔들리지 않고, 여러 테이블이 같은 시점을 본다 |
| 여러 테이블을 넘나드는 정합성 검사 | SERIALIZABLE 또는 명시적 락 + 유니크 제약 | 격리 수준보다 **제약 조건**이 마지막 방어선이다 |

---

## 2. 실무 적용 및 주의사항 (Edge Cases)

### 2-1. RR에서도 lost update는 발생한다 — 가장 많이 틀리는 지점

"REPEATABLE READ면 갱신 손실이 없다"는 **틀렸다.** MVCC는 읽기 충돌을 해결하지만, **"읽은 값을 기반으로 계산해서 쓰는" 패턴**은 막지 않는다.

```text
[RR, 재고 10, 두 요청이 동시에 1개씩 차감]

요청A: BEGIN → SELECT stock (스냅샷: 10) → 애플리케이션이 9 계산
요청B: BEGIN → SELECT stock (스냅샷: 10) → 애플리케이션이 9 계산
요청A: UPDATE product SET stock = 9 WHERE id = 1  → X-Lock 획득, 커밋
요청B: UPDATE product SET stock = 9 WHERE id = 1  → 락 대기 후 실행, 커밋
                                                 ↑ UPDATE는 최신 row를 잠그지만
                                                   "9"라는 값은 이미 계산된 상수다

결과: stock = 9  (2개 팔렸는데 1개만 차감됨) = lost update
```

핵심은 **UPDATE 문이 `stock = stock - 1`이 아니라 `stock = 9`(애플리케이션이 계산한 절대값)** 이라는 점이다. InnoDB는 "최신 row에 락을 걸고 값을 써 넣는" 일만 하므로, 값 자체가 낡았는지는 알 수 없다.

해결 수단 비교:

| 방법 | 구현 | 장점 | 단점 / 주의 |
|---|---|---|---|
| **조건부 원자 UPDATE** (권장) | `UPDATE ... SET stock = stock - 1 WHERE id = ? AND stock >= 1` | 락 대기 없음, 한 문장, 반환 행 수로 결과 판정 | 반환값 0을 반드시 예외로 처리. `clearAutomatically`/`flushAutomatically` 주의 |
| **비관적 락** | `SELECT ... FOR UPDATE` 후 엔티티 수정 | 도메인 로직(검증·이벤트)을 그대로 쓸 수 있음 | 락 대기 → 처리량 저하, 긴 트랜잭션이면 교착·타임아웃 |
| **낙관적 락** | `@Version` + 충돌 시 재시도 | 락을 안 잡아 처리량 높음, 충돌이 드문 경우 최적 | 충돌 시 `OptimisticLockException` → **재시도 로직 필수**. 재시도가 없으면 그냥 실패 |
| **분산 락 (Redis/Redisson)** | 키 단위 락 | 다중 인스턴스·다중 리소스 조합에 유용 | DB 락보다 비싸고, 락 만료·펜싱 토큰 등 추가 설계 필요 |
| **Redis 원자 연산** | `DECR`, Lua 스크립트, ZSET | 초고속, 경합에 강함 | RDB와의 정합성은 별도 문제(2-7) |

```java
// 낙관적 락 재시도 — 재시도가 없으면 낙관적 락은 "가끔 실패하는 기능"이 된다
@Transactional
public void decreaseWithRetry(Long productId, int quantity) {
    int maxAttempts = 3;
    for (int attempt = 1; attempt <= maxAttempts; attempt++) {
        try {
            Product product = productRepository.findById(productId)
                    .orElseThrow(() -> new NoSuchElementException("상품 없음"));
            product.decrease(quantity);
            return;   // 커밋은 프록시가 담당
        } catch (ObjectOptimisticLockingFailureException e) {
            if (attempt == maxAttempts) {
                throw e;
            }
            // 같은 트랜잭션 안에서 재시도하면 영속성 컨텍스트가 이미 오염되어 있다.
            // 실무에서는 REQUIRES_NEW 또는 트랜잭션 밖 재시도 루프(별도 빈)로 분리한다.
        }
    }
}
```

**낙관적 락의 함정**: 위 코드처럼 `@Transactional` 메서드 **안에서** 재시도 루프를 돌리면, 첫 실패 이후 영속성 컨텍스트와 트랜잭션이 rollback-only가 되어 재시도가 무의미해진다. 재시도는 (1) 트랜잭션 경계 바깥(컨트롤러/파사드), 또는 (2) `REQUIRES_NEW`로 분리한 별도 빈에서 해야 한다.

### 2-2. 긴 트랜잭션 = undo 적체 = DB 전체가 느려짐

"조회만 하는 트랜잭션이니 괜찮다"가 가장 위험한 착각이다. **일관성 읽기도 Read View를 고정하므로 purge를 막는다.**

증상과 원인:

| 증상 | 원인 |
|---|---|
| `History list length` 지속 증가 | 오래 열린 Read View가 있어 undo purge가 진행되지 못함 |
| `ibdata`/undo 테이블스페이스 급증 | purge 지연으로 undo 페이지가 계속 쌓임 |
| 갑자기 모든 쿼리가 느려짐 | undo 체인이 길어져 스냅샷 읽기가 여러 버전을 거슬러 올라감 + 버퍼 풀 오염 |
| 디스크 사용량만 늘고 데이터는 그대로 | delete-mark만 된 row와 undo가 물리 공간을 점유 |

대응:

```sql
-- 1) 범인 찾기: 오래 살아 있는 트랜잭션
SELECT trx_id, trx_started, TIMESTAMPDIFF(MINUTE, trx_started, NOW()) AS age_min,
       trx_mysql_thread_id, trx_state, trx_isolation_level, trx_query
FROM information_schema.INNODB_TRX
ORDER BY trx_started
LIMIT 10;

-- 2) 필요하면 종료 (애플리케이션이 아니라 "사람이 연 세션"인지 확인 후)
KILL <trx_mysql_thread_id>;

-- 3) 배치/리포트는 반드시 짧게 쪼갠다
--    ❌ SELECT * FROM orders WHERE ... (수백만 건, 애플리케이션에서 스트리밍하며 8분)
--    ✅ 키셋 페이징으로 5,000건씩, 각 배치는 독립 트랜잭션
SELECT id, ... FROM orders WHERE id > :lastId ORDER BY id LIMIT 5000;
```

- 커넥션 풀(HikariCP)에 `leakDetectionThreshold`를 걸어 "커넥션을 오래 쥔 코드"를 조기에 발견한다.
- `spring.jpa.open-in-view=false`로 두면 OSIV가 트랜잭션 종료 후까지 커넥션·영속성 컨텍스트를 붙잡는 문제를 피할 수 있다.
- **읽기 전용 Replica**로 리포트를 보내도 Replica의 purge가 막힌다. "읽기니까 무해하다"는 성립하지 않는다.

### 2-3. RC로 바꿀 때의 대가

READ COMMITTED는 갭 락을 없애 교착을 줄여 주지만, 공짜가 아니다.

- **`binlog_format = ROW`가 사실상 필수**다. STATEMENT 기반 바이너리 로그는 "실행된 SQL 문"을 그대로 복제하는데, RC에서는 문장마다 스냅샷이 달라져 마스터와 레플리카의 결과가 어긋날 수 있다(비결정적 UPDATE). MySQL 8.0 기본값은 `ROW`라 대개 문제없지만, 레거시 설정을 물려받았다면 확인해야 한다.
- **"트랜잭션 안에서 같은 값을 두 번 읽어도 같다"는 보장이 사라진다.** 서비스 로직이 그 보장에 기대고 있었다면(예: 1-6의 `cancelIfNoItems`), RC로 바꾸는 순간 조용히 깨진다.
- **팬텀이 잠금 읽기에서도 나타난다.** RR에서는 `FOR UPDATE`가 갭 락으로 새 row 유입을 막았지만, RC에서는 막지 않는다.
- 그래서 **격리 수준을 바꾸는 변경은 성능 튜닝이 아니라 의미 변경**으로 취급해야 한다. 코드 리뷰·회귀 테스트 없이 전역 설정만 바꾸는 것은 위험하다.

### 2-4. 갭 락이 만드는 교착, 그리고 회피 순서

```text
[대표 패턴 1 — 순서가 다른 두 범위 잠금]
A: SELECT ... WHERE id BETWEEN 1 AND 5 FOR UPDATE
B: SELECT ... WHERE id BETWEEN 3 AND 8 FOR UPDATE
A: (5 이후 구간에 INSERT 시도)
B: (3 이전 구간에 INSERT 시도)     → 서로의 갭에서 대기 → 교착

[대표 패턴 2 — "조회 후 없으면 INSERT" (동시 INSERT)]
A: SELECT ... WHERE email = 'x@y.com' FOR UPDATE  → 없음 → 갭 락
B: SELECT ... WHERE email = 'x@y.com' FOR UPDATE  → 없음 → 같은 갭 락 (공유 가능)
A: INSERT ...                                     → B의 갭 때문에 대기
B: INSERT ...                                     → A의 갭 때문에 대기 → 교착
```

패턴 2는 **유니크 제약 + INSERT 예외 처리**로 푸는 것이 정석이다. 조회로 중복을 판정하지 않고, `UNIQUE(email)` 인덱스에 맡기고 `DuplicateKeyException`을 잡아 멱등 처리한다. 갭 락을 아예 만들지 않는 것이 최선이다.

교착이 발생했을 때의 처리 원칙:

1. **애플리케이션은 교착을 "정상적인 재시도 대상"으로 취급**한다(MySQL은 교착 시 한쪽을 `ERROR 1213`으로 즉시 롤백한다). 최대 2~3회, 짧은 지연을 두고 재시도.
2. **락 획득 순서를 코드 전체에서 통일**한다(ID 오름차순 등). 순서만 맞춰도 대부분의 교착이 사라진다.
3. **트랜잭션 안에서 외부 API 호출을 하지 않는다.** 락을 쥔 시간이 외부 응답 시간만큼 늘어난다.
4. 교착이 잦은 구간은 **갭 락을 만들지 않는 방식(조건부 UPDATE, 유니크 제약, `SKIP LOCKED`)** 으로 재설계한다.

```sql
-- 작업 큐/재고 예약처럼 "여러 워커가 같은 테이블에서 서로 다른 row를 가져가는" 패턴
-- SKIP LOCKED: 이미 잠긴 row는 기다리지 않고 건너뛴다 → 갭 락 대기·교착을 회피
START TRANSACTION;
SELECT id FROM coupon_stock
 WHERE status = 'AVAILABLE'
 ORDER BY id
 LIMIT 10
 FOR UPDATE SKIP LOCKED;
-- 가져온 id들만 처리
COMMIT;
```

### 2-5. Spring에서 격리 수준이 조용히 무시되는 지점

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void outer() {
    inner();   // 같은 빈 안의 호출 → 프록시 우회(자가 호출). 격리 수준도 전파도 적용 안 됨
}
```

- **`@Transactional`은 새 물리 트랜잭션이 시작될 때만** `setTransactionIsolation()`을 호출한다. `REQUIRED`로 기존 트랜잭션에 **참여**하면 이미 정해진 격리 수준을 바꾸지 못한다. 이때 Spring의 `validateExistingTransaction`이 `false`(기본값)이면 **조용히 무시**되고, `true`면 `IllegalTransactionStateException`이 발생한다. "격리 수준을 바꿨는데 동작이 그대로"라면 이 지점을 먼저 확인한다.
- `PlatformTransactionManager`가 `DataSourceTransactionManager`든 `JpaTransactionManager`든, 격리 수준은 결국 **JDBC `Connection.setIsolationLevel()`** 로 내려간다. 커넥션 풀은 `setIsolationLevel` 호출 시점에 해당 커넥션을 풀에서 분리(reset)하므로, **격리 수준이 다른 트랜잭션이 섞이면 커넥션 재사용 효율이 떨어진다.**
- **Hibernate 2차 캐시는 MVCC를 우회한다.** 2차 캐시에 있는 엔티티는 DB의 Read View와 무관하게 캐시에서 반환된다. MVCC 스냅샷 일관성이 중요한 로직에서는 2차 캐시를 끄거나 `@Cache(usage = NONSTRICT_READ_WRITE)`의 의미를 정확히 알고 써야 한다.
- **`Isolation.SERIALIZABLE`을 JPA에서 쓰면** 격리 수준이 커넥션에 내려가고, InnoDB가 일반 `SELECT`를 공유 락 읽기로 바꾼다. SQL 자체는 그대로여도 실제로는 광범위한 공유 락이 걸려 동시성이 크게 떨어지고 교착도 늘어난다. 예상보다 훨씬 많은 락이 잡히므로 특정 메서드에만 좁게 적용한다.

### 2-6. Replica·복제 지연과 스냅샷

- Replica에서 `START TRANSACTION WITH CONSISTENT SNAPSHOT`을 하면 **Replica에 적용된 시점의 스냅샷**을 본다. Primary보다 과거일 수 있으므로 "방금 쓴 데이터를 바로 읽기" 요구에는 부적합하다.
- 읽기/쓰기 분리 구조에서 **쓰기 직후 읽기를 Replica로 보내면** 복제 지연만큼 값이 안 보인다. 이때 (1) 쓰기 직후 일정 시간은 Primary로 라우팅, (2) 세션 단위 고정, (3) 사용자에게 "반영 중"을 표시하는 방식 중 하나를 명시적으로 선택해야 한다.
- `START TRANSACTION WITH CONSISTENT SNAPSHOT`은 **RR에서만 의미가 있다.** RC에서는 어차피 문장마다 Read View를 만들므로 이 구문이 스냅샷을 고정하지 못한다(에러는 나지 않는다 — 조용히 기대만 어긋난다).

### 2-7. 아그작(AGZAK) 관점 메모

- **Redis ZSET 랭킹과 RDB 점수 저장은 하나의 트랜잭션으로 묶을 수 없다.** MVCC는 같은 스토리지 엔진 안에서만 동작한다. RDB 커밋 후 `@TransactionalEventListener(AFTER_COMMIT)`로 Redis를 갱신하고, 실패 시 재처리(보상·재시도 큐·아웃박스)를 두는 구조가 현실적이다.
- **점수 중복 반영 방지**는 격리 수준이 아니라 **유니크 제약(예: `UNIQUE(game_id, user_id, round)`)** 으로 막는다. "조회해서 없으면 INSERT"는 2-4의 교착 패턴에 빠진다.
- **랭킹 스냅샷 적재 배치**는 여러 워커가 동시에 돌 수 있으므로 ShedLock으로 한 인스턴스만 실행하고, 배치 자체는 **청크 단위 짧은 트랜잭션**으로 쪼갠다. 배치 전체를 한 트랜잭션으로 감싸면 undo가 끝없이 쌓인다.
- **동시 로그인/중복 요청**처럼 "한 사용자에 대해 순서가 중요한" 처리는 MVCC로 해결되지 않는다. 사용자 단위 분산 락 또는 조건부 UPDATE로 명시적으로 직렬화한다.

### 2-8. 그 밖의 엣지 케이스

- **`COUNT(*)`와 스냅샷**: RR의 `SELECT COUNT(*)`는 스냅샷 기준이다. 그래서 "삭제했는데 카운트가 그대로"처럼 보일 수 있다(다른 트랜잭션이 아직 커밋 전이거나, 내 트랜잭션이 오래됐거나). 대부분은 **긴 트랜잭션**이 원인이다.
- **`SELECT ... FOR UPDATE`의 대상이 없을 때**: RR에서는 갭 락이 걸려 "없는 row"를 잠근다. 이 락은 `performance_schema.data_locks`에 `LOCK_DATA`가 비어 있는 형태로 나타나 원인 파악이 어렵다. "아무것도 잠그지 않았다"고 착각하지 말 것.
- **자동 증가 PK와 갭 락**: `AUTO_INCREMENT` 삽입은 삽입 위치가 항상 끝이라 갭 락 충돌이 적다. 반면 **UUID 같은 무작위 PK**는 삽입 위치가 흩어져 페이지 분할과 갭 락 충돌을 모두 늘린다(11-07 노트의 B+Tree와 이어지는 지점).
- **`innodb_lock_wait_timeout`(기본 50초)**: 락 대기 상한이다. API 응답 시간 목표(예: 3초)보다 훨씬 길어서, 락 대기가 쌓이면 **에러가 아니라 "느린 성공"** 으로 나타나 원인을 가린다. 짧은 트랜잭션 + 애플리케이션 레벨 타임아웃을 함께 건다.
- **`deleted_flag`와 유니크 인덱스**: delete-mark된 row도 유니크 인덱스에는 남아 있다. 소프트 삭제 후 같은 값을 다시 INSERT하면 중복키 오류가 날 수 있다. 이때는 유니크 인덱스에 삭제 여부를 포함한 복합 키를 쓰거나, 물리 삭제·이력 테이블 분리를 선택한다.
- **`SET SESSION TRANSACTION ISOLATION LEVEL`의 적용 범위**: **다음 트랜잭션부터** 적용된다. 이미 `START TRANSACTION`한 상태에서 바꾸면 현재 트랜잭션에는 영향이 없다. `SET TRANSACTION ISOLATION LEVEL`은 "다음 트랜잭션 한 번"에만 적용된다.
- **`READ UNCOMMITTED`는 실제로 거의 쓰지 않는다**: MVCC 경로를 타지 않아 일관성도 없고 성능 이득도 크지 않다. "가장 빠른 격리 수준"이라는 설명은 오해다.

---

## 3. 기술 면접 단골 꼬리 질문 5선 & 모범 답변

**Q1. MVCC란 무엇이고, 왜 필요한가요? 락 기반 동시성 제어와 비교해 설명해 보세요.**
- **A.** MVCC는 하나의 row에 대해 **여러 버전을 동시에 유지**하고, 각 트랜잭션이 "자기 시점에 유효한 버전"을 골라 읽게 하는 방식입니다. InnoDB는 row 헤더에 `DB_TRX_ID`(마지막 수정 트랜잭션)와 `DB_ROLL_PTR`(undo log 포인터)를 두고, UPDATE 시 이전 이미지를 undo에 남깁니다. 읽기 시점의 **Read View**(활성 트랜잭션 ID 집합)와 비교해 안 보이는 버전이면 undo 체인을 거슬러 올라가 보이는 버전을 복원합니다.
  락 기반 방식과의 차이는 **읽기가 쓰기를 막지 않고, 쓰기가 읽기를 막지 않는다**는 점입니다. 순수 락 방식에서는 조회에도 공유 락이 필요해 조회 트래픽이 많은 서비스에서 동시성이 급락합니다. MVCC는 그 대가로 **undo 보관 공간과 purge 비용**을 지불합니다. 즉 "공짜 최적화"가 아니라 읽기 동시성을 저장 공간·백그라운드 작업과 맞바꾼 설계이고, 그래서 긴 트랜잭션이 undo 적체를 일으키는 것이 MVCC의 대표적인 부작용입니다.

**Q2. READ COMMITTED와 REPEATABLE READ의 차이를 InnoDB 구현 기준으로 설명해 보세요.**
- **A.** 데이터 구조와 undo log, 숨은 컬럼은 동일하고, 차이는 **Read View를 언제 만드는가**입니다. READ COMMITTED는 **문장(statement)마다** Read View를 새로 만들어, 같은 트랜잭션 안에서도 그 사이 커밋된 변경이 보입니다(Non-Repeatable Read 발생). REPEATABLE READ는 **트랜잭션에서 첫 일관성 읽기를 하는 순간** Read View를 한 번 만들고 재사용합니다. 주의할 점은 `START TRANSACTION` 시점이 아니라 **첫 번째 `SELECT` 시점**이라는 것이고, `START TRANSACTION WITH CONSISTENT SNAPSHOT`을 쓰면 시작 시점에 고정됩니다.
  팬텀은 두 가지로 나눠 답해야 합니다. InnoDB의 RR은 **잠금 읽기에서 갭 락(next-key lock)** 으로 범위 내 INSERT를 막아 팬텀을 방지하고, RC는 갭 락을 비활성화해 팬텀을 허용합니다. 반면 **일관성 읽기(`SELECT`)에서는 RR도 RC도 스냅샷을 보므로 "이미 본 범위"의 팬텀은 나타나지 않지만, `UPDATE`/`DELETE`는 Read View를 쓰지 않는 현재 읽기라 새로 들어온 row까지 갱신 대상이 됩니다.** 그래서 "RR은 팬텀이 없다"고 단정하면 틀리고, "일관성 읽기 기준으로는 없다"가 정확합니다.

**Q3. REPEATABLE READ인데도 갱신 손실(lost update)이 발생할 수 있나요?**
- **A.** 발생합니다. MVCC는 **읽기 충돌**을 해결하는 장치이고, "읽은 값을 기반으로 계산해 절대값을 쓰는" 패턴은 막지 못합니다. 두 트랜잭션이 같은 스냅샷에서 `stock = 10`을 읽고 각자 9를 계산해 `UPDATE ... SET stock = 9`를 실행하면, 두 번째 UPDATE는 최신 row에 락을 걸고 실행되지만 쓰는 값은 이미 낡은 계산 결과입니다. 결과는 9가 되어 2건이 팔렸는데 1건만 차감됩니다. RR은 Non-Repeatable Read를 막아 줄 뿐, **"읽고-계산하고-쓰기" 사이의 경합**은 애플리케이션이 다뤄야 합니다.
  해결은 (1) **조건부 원자 UPDATE** — `UPDATE ... SET stock = stock - 1 WHERE id = ? AND stock >= 1` 후 영향 행 수로 판정, (2) `SELECT ... FOR UPDATE`로 현재 읽기 + row 락, (3) `@Version` 낙관적 락과 **트랜잭션 밖 재시도**, (4) Redis 원자 연산입니다. 실무 기본값은 조건부 원자 UPDATE이고, 도메인 검증 로직이 복잡할 때만 비관적 락을 씁니다.

**Q4. 일관성 읽기(Consistent Read)와 잠금 읽기(Locking Read)의 차이는 무엇이며, 같은 트랜잭션에서 값이 다르게 나오는 이유는 무엇인가요?**
- **A.** 일관성 읽기는 일반 `SELECT`로, Read View 시점의 버전을 **락 없이** 읽습니다. 잠금 읽기는 `SELECT ... FOR UPDATE`/`FOR SHARE`로, **최신 커밋 버전**을 읽으면서 row(및 갭) 락을 잡습니다. `UPDATE`/`DELETE`도 내부적으로 현재 읽기입니다.
  같은 트랜잭션에서 값이 다르게 나오는 것은 정상 동작입니다. 일반 `SELECT`는 내 스냅샷(undo로 복원한 과거 버전)을 보고, `FOR UPDATE`는 "지금 쓰려면 최신 값을 알아야 한다"는 목적이라 스냅샷을 무시하고 최신 커밋을 읽습니다. 이 비대칭을 모르면 "조회 결과와 락 조회 결과가 다르다"는 버그 리포트가 나옵니다. 실무에서는 **판단과 실행이 서로 다른 시점을 보지 않도록**, 판단이 필요하면 같은 트랜잭션 안에서 `FOR UPDATE`를 쓰거나 `UPDATE ... WHERE NOT EXISTS`처럼 한 문장으로 합칩니다.

**Q5. 긴 트랜잭션이 왜 위험한가요? 조회만 하는 트랜잭션이라도 문제가 되나요?**
- **A.** 문제가 됩니다. 일관성 읽기도 Read View를 고정시키고, 그 Read View가 "아직 볼 수 있어야 하는 가장 오래된 시점"이 되어 **purge 스레드가 그 이후의 undo를 지우지 못합니다.** 그 결과 `History list length`가 증가하고, undo 테이블스페이스가 팽창하며, 스냅샷 읽기가 거슬러 올라가야 할 버전 수가 늘어 조회가 느려지고 버퍼 풀이 오염됩니다. 즉 **한 세션의 오래된 Read View가 DB 전체 성능에 영향을 줍니다.**
  대응은 (1) 트랜잭션을 짧게 유지하고 외부 API 호출·파일 I/O를 트랜잭션 밖으로 빼기, (2) 대량 조회는 키셋 페이징으로 잘라 여러 트랜잭션으로 처리, (3) HikariCP `leakDetectionThreshold`와 `information_schema.INNODB_TRX`로 장기 트랜잭션 모니터링, (4) `open-in-view=false`로 OSIV가 커넥션을 붙잡는 시간 제거입니다. 읽기 전용 Replica로 보내도 그 Replica의 purge가 막히므로 근본 해결이 아닙니다.

**(보너스) Q6. RR에서 교착이 자주 발생하는 이유와 회피 방법은?**
- **A.** RR은 갭 락/넥스트키 락으로 범위 내 INSERT를 막기 때문에, **존재하지 않는 row를 잠그는 락**이 생깁니다. 대표적으로 "조회 후 없으면 INSERT" 패턴에서 두 트랜잭션이 같은 갭을 잡고 서로의 INSERT를 기다리다 교착이 됩니다. 회피는 (1) `UNIQUE` 제약 + `DuplicateKeyException` 처리로 조회-후-삽입을 없애기, (2) 락 획득 순서를 코드 전체에서 통일, (3) 갭 락이 필요 없는 `SKIP LOCKED`, (4) 교착이 잦은 구간에 한해 READ COMMITTED 검토입니다. 그리고 교착은 **정상적인 재시도 대상**으로 보고 애플리케이션에 2~3회 재시도를 넣어야 합니다. MySQL이 한쪽을 `ERROR 1213`으로 즉시 롤백하기 때문입니다.

---

## 4. 오늘의 체크리스트 (내일 이어서)

- [ ] 0-2/0-3을 실제 세션 2개로 재현하고, `SHOW ENGINE INNODB STATUS`의 `Trx read view will not see trx with id >= ...` 값이 두 격리 수준에서 언제 바뀌는지 캡처
- [ ] `START TRANSACTION` 직후와 첫 `SELECT` 직후의 Read View 생성 시점 차이를 `START TRANSACTION WITH CONSISTENT SNAPSHOT`과 비교
- [ ] 2-1의 naive 차감을 스레드 10개로 돌려 실제로 재고가 어긋나는지 수치 기록 → 조건부 UPDATE와 비교표 작성
- [ ] 0-5의 갭 락 교착을 RC로 바꿔 재실행해 교착이 사라지는지 확인
- [ ] 긴 트랜잭션(조회 8분)을 하나 열어 두고 `History list length` 증가와 undo 테이블스페이스 크기 변화 관찰
- [ ] `@Transactional(isolation = READ_COMMITTED)`가 기존 트랜잭션 참여 시 무시되는 것을 `validateExistingTransaction=true`로 바꿔 예외로 확인
- [ ] `DELETE`가 delete-mark만 남기는 것을 `performance_schema`/undo 관찰로 확인하고, 소프트 삭제 + 유니크 인덱스 충돌 케이스 재현
- [ ] `FOR UPDATE SKIP LOCKED`로 작업 큐 패턴(쿠폰 발급)을 구현하고 동시 워커 5개에서 중복 발급이 없는지 테스트로 고정
- [ ] 낙관적 락 재시도를 트랜잭션 밖으로 뺀 구조와 안에 둔 구조를 비교해, 안에 둔 경우 실패하는 이유를 로그로 남기기
- [ ] 아그작 랭킹 갱신 경로에서 RDB 커밋 → Redis 갱신 사이 실패 시나리오를 정리하고 보상 정책 초안 작성
