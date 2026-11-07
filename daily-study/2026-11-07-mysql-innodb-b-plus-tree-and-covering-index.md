# [2026-11-07] MySQL InnoDB B+Tree 인덱스 동작 원리와 커버링 인덱스(Covering Index) 최적화

> **한 줄 핵심 요약**: InnoDB의 인덱스는 **모든 데이터가 리프 노드에 있고 리프끼리 양방향 연결 리스트로 이어진 B+Tree**이며, 세컨더리 인덱스로 조회하면 리프에서 PK를 얻은 뒤 **클러스터드 인덱스를 한 번 더 타는 랜덤 I/O(이중 탐색)** 가 발생한다 — 쿼리가 필요로 하는 컬럼을 인덱스가 전부 들고 있게 만들면(커버링 인덱스) 이 두 번째 탐색이 사라지고 실행 계획에 `Extra: Using index`가 찍힌다. 실무에서 성능 개선 여지가 가장 큰 지점은 "인덱스를 더 만드는 것"이 아니라 **"이미 있는 인덱스만으로 쿼리를 끝낼 수 있게 쿼리와 인덱스를 맞추는 것"** 이다.

---

## 0. 코드 예제 (전체 실행 가능 버전)

### 0-1. 실습 스키마 (MySQL 8.0, InnoDB, `innodb_page_size=16KB` 기본값)

```sql
-- 파일 경로: src/main/resources/db/migration/V1__orders.sql
-- 전제: orders 1,000만 건, 사용자 1명당 주문 수백 건, status는 소수 값(PAID/SHIPPED/CANCELED)
CREATE TABLE orders (
    id           BIGINT       NOT NULL AUTO_INCREMENT,
    user_id      BIGINT       NOT NULL,
    status       VARCHAR(20)  NOT NULL,
    total_amount INT          NOT NULL,
    ordered_at   DATETIME(6)  NOT NULL,
    memo         VARCHAR(500) NULL,          -- 실제로는 거의 안 읽는 넓은 컬럼
    PRIMARY KEY (id),                        -- 클러스터드 인덱스 = 테이블 자체
    KEY idx_user_status (user_id, status)    -- 세컨더리 인덱스 (내부적으로 user_id, status, id 로 저장됨)
) ENGINE = InnoDB;
```

### 0-2. 개선 전: 실행 계획에 `Using filesort` + 클러스터드 재탐색

```sql
-- 목표: 한 사용자의 결제 완료 주문을 최신순 20건, 화면에 필요한 컬럼은 id와 total_amount
EXPLAIN
SELECT id, total_amount
FROM orders
WHERE user_id = 42
  AND status = 'PAID'
ORDER BY ordered_at DESC
LIMIT 20;
```

```text
-- 실행 계획 예시 (MySQL 8.0 EXPLAIN 출력 형태)
+----+-------------+--------+------+-----------------+-----------------+---------+-------------+------+-----------------------------+
| id | select_type | table  | type | possible_keys   | key             | key_len | ref         | rows | Extra                       |
+----+-------------+--------+------+-----------------+-----------------+---------+-------------+------+-----------------------------+
|  1 | SIMPLE      | orders | ref  | idx_user_status | idx_user_status | 90      | const,const |  312 | Using filesort              |
+----+-------------+--------+------+-----------------+-----------------+---------+-------------+------+-----------------------------+
```

- `key = idx_user_status`: 인덱스는 탔다. 하지만 `ordered_at`이 인덱스에 없으므로 **정렬은 인덱스 순서를 못 쓰고**(`Using filesort`), `total_amount`를 읽으려고 **세컨더리 리프에서 얻은 PK로 클러스터드 인덱스를 312번 재탐색**한다.
- `Extra`에 `Using index`가 없으면, 그 쿼리는 테이블(클러스터드 인덱스) 접근이 남아 있다는 뜻이다.

### 0-3. 개선 후: 커버링 인덱스 (필요 컬럼을 인덱스 리프에 모두 넣는다)

```sql
-- ① 정렬 컬럼을 인덱스에 추가: user_id, status 는 동등 조건이므로 ordered_at 순서가 곧 결과 정렬 순서
-- ② 조회 컬럼(total_amount)을 인덱스 끝에 추가: 클러스터드 인덱스 접근 제거
ALTER TABLE orders
    ADD KEY idx_user_status_ordered_cover (user_id, status, ordered_at, total_amount);
```

```sql
EXPLAIN
SELECT id, total_amount
FROM orders
WHERE user_id = 42
  AND status = 'PAID'
ORDER BY ordered_at DESC
LIMIT 20;
```

```text
+----+-------------+--------+------+--------------------------------+--------------------------------+---------+-------------+------+--------------------------+
| id | select_type | table  | type | possible_keys                  | key                            | key_len | ref         | rows | Extra                    |
+----+-------------+--------+------+--------------------------------+--------------------------------+---------+-------------+------+--------------------------+
|  1 | SIMPLE      | orders | ref  | idx_user_status,idx_user_...   | idx_user_status_ordered_cover  | 90      | const,const |  312 | Using index              |
+----+-------------+--------+------+--------------------------------+--------------------------------+---------+-------------+------+--------------------------+
```

- `Extra: Using index` = **커버링 인덱스로 처리됨**(테이블 접근 0회).
- `Using filesort`가 사라졌다. `(user_id, status, ordered_at)` 순서에서 앞 두 컬럼이 `=` 로 고정되면, 남은 `ordered_at`이 인덱스 정렬 순서 그대로이므로 MySQL 8.0은 **인덱스를 역방향으로 읽어** `DESC` 정렬을 만족시킨다.

### 0-4. 실무 패턴: 커버링 인덱스 + 지연 조인(Deferred Join)으로 깊은 페이징 처리

`LIMIT 100000, 20` 같은 깊은 페이징에서 `SELECT *`로 커버링을 포기하면, 버려질 10만 건의 본문 row까지 클러스터드 인덱스에서 읽는다. 그래서 **"id만 커버링 인덱스로 뽑고 → 그 20건만 본문 조회"** 로 2단계를 나눈다.

```sql
-- 1단계: 인덱스만으로 id 20건 확보 (Extra: Using index)
SELECT id
FROM orders
WHERE user_id = 42 AND status = 'PAID'
ORDER BY ordered_at DESC
LIMIT 100000, 20;

-- 2단계: 확보한 id 20건만 PK로 본문 조회 (클러스터드 인덱스 20회)
SELECT id, user_id, status, total_amount, ordered_at, memo
FROM orders
WHERE id IN (/* 1단계 결과 20건 */);
```

```java
// 파일 경로: src/main/java/study/order/repository/OrderRepository.java
package study.order.repository;

import java.util.List;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import study.order.domain.Order;
import study.order.domain.OrderStatus;

public interface OrderRepository extends JpaRepository<Order, Long> {

    // 1단계: 엔티티가 아니라 id만 JPQL로 뽑는다.
    // 엔티티를 조회하면 하이버네이트가 모든 컬럼을 SELECT 하므로 커버링 인덱스가 성립할 수 없다.
    @Query("""
            select o.id
            from Order o
            where o.user.id = :userId
              and o.status = :status
            order by o.orderedAt desc
            """)
    List<Long> findPageIds(@Param("userId") Long userId,
                           @Param("status") OrderStatus status,
                           Pageable pageable);

    // 2단계: id 20건으로 본문만 조회. @BatchSize(또는 default_batch_fetch_size)로 연관 컬렉션은 따로 묶어 읽는다.
    @Query("""
            select o
            from Order o
            where o.id in :ids
            order by o.orderedAt desc
            """)
    List<Order> findAllByIdIn(@Param("ids") List<Long> ids);
}
```

```java
// 파일 경로: src/main/java/study/order/service/OrderQueryService.java
package study.order.service;

import java.util.List;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Slice;
import org.springframework.data.domain.SliceImpl;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import study.order.domain.Order;
import study.order.domain.OrderStatus;
import study.order.repository.OrderRepository;

@Service
public class OrderQueryService {

    private final OrderRepository orderRepository;

    public OrderQueryService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Transactional(readOnly = true)
    public Slice<Order> findOrders(Long userId, OrderStatus status, int page, int size) {
        // size + 1 로 다음 페이지 존재 여부를 판단 → count(*) 쿼리 자체를 없앤다.
        PageRequest request = PageRequest.of(page, size + 1);

        List<Long> ids = orderRepository.findPageIds(userId, status, request);
        boolean hasNext = ids.size() > size;
        List<Long> pageIds = hasNext ? ids.subList(0, size) : ids;

        if (pageIds.isEmpty()) {
            return new SliceImpl<>(List.of(), request, false);
        }
        // findAllByIdIn 은 IN 절 순서를 보장하지 않으므로 정렬은 쿼리의 order by 에 맡긴다.
        return new SliceImpl<>(orderRepository.findAllByIdIn(pageIds), request, hasNext);
    }
}
```

```java
// 파일 경로: src/main/java/study/order/repository/OrderQueryDslRepository.java (QueryDSL 버전)
package study.order.repository;

import static study.order.domain.QOrder.order;

import com.querydsl.jpa.impl.JPAQueryFactory;
import java.util.List;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Repository;
import study.order.domain.OrderStatus;

@Repository
public class OrderQueryDslRepository {

    private final JPAQueryFactory queryFactory;

    public OrderQueryDslRepository(JPAQueryFactory queryFactory) {
        this.queryFactory = queryFactory;
    }

    // select(order.id) 는 JPQL "select o.id" 로 변환된다 → 커버링 인덱스가 성립하는 유일한 형태
    public List<Long> findPageIds(Long userId, OrderStatus status, Pageable pageable) {
        return queryFactory
                .select(order.id)
                .from(order)
                .where(order.user.id.eq(userId), order.status.eq(status))
                .orderBy(order.orderedAt.desc())
                .offset(pageable.getOffset())
                .limit(pageable.getPageSize())
                .fetch();
    }
}
```

### 0-5. 검증 방법: `Handler_read%` 카운터와 `EXPLAIN ANALYZE`

`Extra: Using index`만 보고 끝내지 말고, **실제로 row를 몇 번 읽었는지**를 세션 카운터로 확인한다.

```sql
-- 쿼리 실행 직전/직후의 세션 카운터 차이를 본다 (FLUSH STATUS는 전역 변수를 초기화하며 RELOAD 권한이 필요)
SHOW SESSION STATUS LIKE 'Handler_read%';

SELECT id, total_amount FROM orders
WHERE user_id = 42 AND status = 'PAID'
ORDER BY ordered_at DESC LIMIT 20;

SHOW SESSION STATUS LIKE 'Handler_read%';   -- 직전 값과의 차이를 계산
```

- `Handler_read_key`: 인덱스/ PK로 위치를 찾은 횟수
- `Handler_read_next`: 인덱스를 순서대로 넘긴 횟수(범위 스캔 = 스캔한 인덱스 엔트리 수)
- `Handler_read_rnd_next`: 인덱스 없이 다음 row를 읽은 횟수 = **테이블 풀 스캔의 지표**
- 커버링 인덱스가 걸리면 `Handler_read_next`는 늘고, 클러스터드 인덱스 재탐색에 해당하는 **`Handler_read_key`가 줄어든다.** 같은 데이터·같은 결과에서 이 숫자가 줄면 실제 이득이 있다는 증거다.

```sql
-- 8.0.18+ : 예상치(rows)가 아니라 실측 시간/실제 row 수로 확인
EXPLAIN ANALYZE
SELECT id, total_amount FROM orders
WHERE user_id = 42 AND status = 'PAID'
ORDER BY ordered_at DESC LIMIT 20;
```

### 0-6. 안티패턴 → 개선 SQL 대조

```sql
-- ① 인덱스 컬럼 가공: 인덱스가 있어도 못 쓴다 (함수 결과에는 인덱스가 없다)
SELECT id FROM orders WHERE SUBSTRING(status, 1, 3) = 'PAI';
SELECT id FROM orders WHERE DATE(ordered_at) = '2026-11-07';

-- 개선: 컬럼을 가공하지 않고 범위 조건으로 바꾼다
SELECT id FROM orders WHERE status LIKE 'PAI%';
SELECT id FROM orders
WHERE ordered_at >= '2026-11-07 00:00:00'
  AND ordered_at <  '2026-11-08 00:00:00';

-- ② 암시적 형변환: varchar 컬럼을 숫자와 비교하면 컬럼 쪽에 CAST가 붙어 인덱스 무효
--    phone_number VARCHAR(20) 인 경우
SELECT id FROM users WHERE phone_number = 1012345678;   -- 인덱스 미사용
SELECT id FROM users WHERE phone_number = '1012345678'; -- 인덱스 사용

-- ③ 선행 와일드카드: B+Tree는 앞에서부터 비교하므로 접두어가 없으면 범위를 좁힐 수 없다
SELECT id FROM orders WHERE memo LIKE '%환불%';

-- ④ 복합 인덱스 선행 컬럼 누락: (user_id, status) 인덱스에서 status만 조건이면 레인지 스캔 불가
SELECT id FROM orders WHERE status = 'PAID';
```

---

## 1. 개념 및 동작 원리

### 1-1. B-Tree vs B+Tree: 무엇이 다르고 왜 DB는 B+Tree를 쓰는가

| 구분 | B-Tree | B+Tree (InnoDB) |
|---|---|---|
| 데이터 저장 위치 | 내부 노드·리프 노드 모두에 데이터(또는 데이터 포인터) | **리프 노드에만** 저장, 내부 노드는 키와 자식 포인터만 |
| 노드당 키 수 | 데이터까지 담아야 해서 적음 | 키+포인터만 담아 **한 블록에 훨씬 많은 키** 수용 |
| 트리 높이 | 같은 row 수에서 더 높음 | 더 낮음 → **디스크 I/O 감소** |
| 범위 검색 | 중위 순회로 여러 노드를 오르내려야 함 | **리프끼리 양방향 연결 리스트** → 시작점만 찾으면 순차 이동 |
| 정렬/전체 스캔 | 상대적으로 불리 | 리프 리스트를 따라 **순차 I/O** 로 전체 스캔 |
| 탐색 시간 | 키마다 성공/실패 위치가 달라질 수 있음 | **모든 탐색이 리프까지 동일 깊이** → 예측 가능한 비용 |

핵심은 "B+Tree는 **탐색은 B-Tree처럼, 스캔은 연결 리스트처럼** 한다"는 점이다. DB의 쿼리는 `WHERE a = 1 AND b BETWEEN 10 AND 20 ORDER BY b` 처럼 **동등 + 범위 + 정렬이 섞여** 있고, 이 조합을 가장 싸게 처리하는 구조가 B+Tree다.

### 1-2. InnoDB 페이지와 트리 높이: 왜 실무에서 3~4단계면 충분한가

- InnoDB는 16KB 페이지(= B+Tree의 노드) 단위로 디스크와 메모리를 오간다. 버퍼 풀에 없는 페이지를 읽으면 **디스크 I/O 1회**가 발생한다.
- 내부 노드 엔트리 = 키(8바이트 `BIGINT`) + 자식 페이지 포인터(6바이트) ≈ 14바이트 → 16KB 페이지에 **약 1,170개**의 자식 포인터.
- 리프 엔트리 = 실제 row(또는 세컨더리 인덱스의 키+PK). row가 1KB라면 리프 페이지당 16건.

```text
row 1KB, 내부 노드 1,170개 포인터 기준
높이 1: 16건
높이 2: 1,170 × 16         ≈ 18,700건
높이 3: 1,170² × 16        ≈ 2,190만건   ← 1,000만 건 테이블이 여기에 들어간다
```

즉 1,000만 건 테이블도 **루트 → 중간 → 리프 3번의 페이지 접근**으로 원하는 row에 도달한다. 게다가 루트와 상위 레벨은 버퍼 풀에 상주하므로 실제 디스크 I/O는 1회 수준이다. **인덱스가 빠른 이유는 "탐색을 잘해서"가 아니라 "읽는 페이지 수가 적어서"** 라는 관점을 기억해 두면 좋다.

### 1-3. 클러스터드 인덱스(PK) vs 세컨더리 인덱스

```text
[클러스터드 인덱스 = 테이블 그 자체]  PK(id) 순서로 리프에 "row 전체" 저장
            ┌──────────────┐
   root →   │  10 | 500 |  │
            └──┬───┬───┬───┘
        ┌──────┘   │   └──────┐
   leaf(id 1~9) leaf(10~499) leaf(500~)
   (id, user_id, status, total_amount, ordered_at, memo) ...

[세컨더리 인덱스]  (user_id, status) 순서로 리프에 "인덱스 컬럼 + PK" 저장
   leaf: (user_id=42, status=PAID, id=812) (…, id=901) (…, id=1057)
                                              └── PK만 있다. total_amount는 없다.
                                                  → 클러스터드 인덱스로 한 번 더 가야 한다
```

- **클러스터드 인덱스는 테이블당 1개**다. PK 순서대로 row를 물리적으로 정렬해 저장하므로 "테이블 = 클러스터드 인덱스"다. PK가 없으면 InnoDB가 내부적으로 6바이트 row id로 만든 클러스터드 인덱스를 쓴다(사용자가 인지할 수 없는 숨은 PK).
- **세컨더리 인덱스의 리프에는 항상 PK가 붙는다.** 그래서 `KEY idx_user_status (user_id, status)`는 실제로 `(user_id, status, id)` 순서로 저장된다. 이 사실이 커버링 인덱스 실무에서 중요하다: **PK 컬럼은 인덱스에 명시하지 않아도 이미 인덱스에 있다.**
- 따라서 **PK가 길면(예: UUID 문자열) 모든 세컨더리 인덱스의 리프가 함께 커진다.** 세컨더리 인덱스 10개에 PK 36바이트가 붙으면 인덱스 전체 크기가 수백 MB 단위로 늘어나고, 버퍼 풀에 담을 수 있는 인덱스 페이지 수가 줄어든다. 신입 포트폴리오에서도 "PK는 `BIGINT AUTO_INCREMENT`, 외부 노출용 식별자는 별도 UUID 컬럼" 같은 선택의 근거가 된다.

### 1-4. 이중 탐색(Double Lookup): 세컨더리 인덱스 조회의 진짜 비용

```text
WHERE user_id = 42 AND status = 'PAID'  (인덱스: (user_id, status))

1) 세컨더리 B+Tree 탐색        : root → 중간 → 리프 (버퍼 풀 hit 가능성 높음)
2) 리프 리스트를 순차로 따라가며 PK 수집  : (42,PAID,812), (42,PAID,901), ...  ← 순차 I/O
3) 각 PK로 클러스터드 인덱스 재탐색       : id=812 → 다른 위치의 페이지, id=901 → 또 다른 페이지
                                            ← 무작위 위치의 16KB 페이지 읽기 = 랜덤 I/O
```

- 3단계가 **랜덤 I/O**다. 세컨더리 인덱스에서 얻은 PK 순서는 `user_id, status` 기준이지 `id` 기준이 아니므로 클러스터드 인덱스에서 읽을 페이지가 흩어진다. 결과가 1,000건이면 이론상 최대 1,000번의 랜덤 페이지 접근이 필요하다.
- 커버링 인덱스는 **3단계를 통째로 제거**한다. 필요한 컬럼이 인덱스 리프에 이미 있으므로 2단계에서 끝난다. 1,000건 조회에서 "랜덤 1,000회"가 "순차 1,000엔트리"로 바뀐다.
- 여기서 성능 차이의 본질이 드러난다. 순차 읽기는 같은 페이지를 버퍼 풀에서 재사용하고 OS read-ahead도 받지만, 랜덤 읽기는 페이지 하나하나가 **버퍼 풀 miss 시 디스크 seek**이 된다. 데이터가 버퍼 풀에 다 올라와 있으면 차이가 작고, **콜드 캐시이거나 테이블이 버퍼 풀보다 클 때 차이가 커진다.** 그래서 커버링 인덱스는 "항상 2배 빠르다"가 아니라 **"랜덤 I/O 의존을 없애 캐시 미스에 강해진다"** 로 이해해야 한다.

### 1-5. 커버링 인덱스의 성립 조건 (판정 체크리스트)

`Extra: Using index`가 나오려면 쿼리가 필요로 하는 모든 컬럼이 **하나의 인덱스 안에** 있어야 한다.

- [ ] `SELECT` 절의 컬럼이 모두 인덱스에 있다 (PK는 자동 포함)
- [ ] `WHERE` 절의 컬럼이 모두 인덱스에 있다
- [ ] `ORDER BY` / `GROUP BY` 컬럼까지 인덱스에 있다 — 없으면 인덱스로 필터링 후 `filesort`/`temporary`가 붙는다
- [ ] 인덱스가 **하나**로 성립한다 (서로 다른 인덱스의 컬럼을 섞으면 `Using index` 불가)
- [ ] 인덱스 컬럼을 가공하지 않았다 (가공하면 그 컬럼은 인덱스로 필터링되지 않는다)
- [ ] `SELECT *`를 쓰지 않았다 — `SELECT *`는 커버링 인덱스와 **원리적으로 양립 불가**

주의: `Extra`에 보이는 `Using index condition`은 **인덱스 컨디션 푸시다운(ICP)** 으로, 커버링과 다른 것이다. ICP는 "인덱스에 있는 컬럼 조건을 스토리지 엔진 단에서 먼저 걸러" 클러스터드 인덱스 접근 횟수를 줄이는 최적화일 뿐, **여전히 테이블 접근이 남아 있다.** 커버링은 `Using index`만 해당한다.

### 1-6. 복합 인덱스 컬럼 순서 결정 원칙

면접에서 "카디널리티 높은 컬럼을 앞에"라고만 답하면 절반만 맞다. 실무 판단 순서는 다음과 같다.

1. **동등 조건(`=`, `IN`) 컬럼을 앞에, 범위 조건(`>`, `<`, `BETWEEN`, `LIKE 'x%'`) 컬럼을 뒤에.**
   - 이유: B+Tree는 앞 컬럼부터 비교한다. 앞이 범위 조건이면 그 뒤 컬럼은 "정렬된 상태"가 아니게 되어 범위를 좁히는 데 쓰이지 못한다.
   - `(a, b)` 인덱스에서 `WHERE a BETWEEN 1 AND 100 AND b = 5` → `a`로 넓게 스캔한 뒤 `b`는 **인덱스 필터(ICP)** 로만 쓰인다. `(b, a)` 인덱스라면 `b=5`로 먼저 좁힌 뒤 `a` 범위를 스캔한다.
2. **동등 조건이 여러 개면 선택도(카디널리티)가 높은 컬럼을 앞에.**
   - `status`(5종) vs `user_id`(100만 종) → `(user_id, status)`가 낫다. 앞 컬럼 하나로 결과를 크게 줄일수록 리프 스캔량이 줄어든다.
   - 단, **동등 조건끼리는 순서를 바꿔도 결과는 같다**(어느 쪽이든 정확히 같은 범위로 좁혀진다). 차이는 스캔 엔트리 수뿐이므로, 카디널리티가 비슷하면 **다른 쿼리도 재사용할 수 있는 순서**를 택한다(왼쪽 접두어 규칙).
3. **`ORDER BY` / `GROUP BY` 컬럼을 그 뒤에 붙여 정렬 비용을 제거한다.** (0-3의 예: `user_id, status, ordered_at`)
4. **커버링까지 노린다면 `SELECT` 컬럼을 인덱스 끝에 추가한다.** (`total_amount`) 단, 3·4번으로 인덱스가 넓어지면 쓰기 비용과 인덱스 크기가 늘어나므로 **읽기 빈도 × 절감량**으로 판단한다.
5. **실제 순서는 `EXPLAIN`과 `Handler_read%`로 검증한다.** 옵티마이저는 통계(카디널리티)로 판단하므로, 사람이 계산한 순서와 다르게 계획이 잡히면 그 이유(통계 오래됨, 조건절 형태)를 먼저 확인한다.

---

## 2. 실무 적용 및 주의사항 (Edge Cases)

### 2-1. 인덱스가 무력화되는 안티패턴 정리

| 안티패턴 | 예시 | 실행 계획에서 벌어지는 일 | 개선 |
|---|---|---|---|
| 컬럼 가공 | `WHERE SUBSTRING(name,1,3)='김철'`, `WHERE age+10=30` | 인덱스 키가 아니라 **함수 결과값**을 비교하므로 인덱스 사용 불가(`type: ALL`) | 조건을 컬럼 단독 형태로 변형 (`age = 20`), `LIKE '김철%'` |
| 날짜 함수 | `WHERE DATE(created_at)='2026-11-07'` | 위와 동일 | `>= '2026-11-07 00:00' AND < '2026-11-08 00:00'` |
| 암시적 형변환 | varchar 컬럼 vs 숫자 리터럴 | 컬럼 쪽에 `CAST`가 붙어 인덱스 무효. **타입이 다르면 항상 리터럴을 컬럼 타입에 맞춘다** | `'1012345678'`처럼 문자열 리터럴로 비교 |
| 선행 와일드카드 | `LIKE '%환불%'` | 접두어가 없어 B+Tree 범위 축소 불가 → 인덱스 풀 스캔 또는 테이블 풀 스캔 | `LIKE '환불%'`, 전문 검색이 필요하면 FULLTEXT/검색엔진 |
| 복합 인덱스 선행 컬럼 누락 | `(user_id, status)`인데 `WHERE status=?` | 왼쪽 접두어가 없어 **레인지 스캔 불가** → 인덱스 풀 스캔(`type: index`) 또는 풀 스캔 | 선행 컬럼을 조건에 추가하거나, `status` 단독 인덱스/다른 복합 인덱스 검토 |
| 부정형 조건 | `WHERE status != 'CANCELED'`, `NOT IN` | 대부분의 값을 읽어야 하므로 옵티마이저가 인덱스를 포기 | 긍정 조건으로 재작성, 또는 조건을 애플리케이션에서 처리 |
| `OR` 남용 | `WHERE user_id=? OR memo LIKE ?` | 인덱스 머지(union) 또는 풀 스캔으로 떨어짐 | `UNION ALL`로 분리해 각각 인덱스 태우기 |
| `NULL` 비교 | `WHERE deleted_at IS NULL` | `IS NULL`은 인덱스 사용 가능하지만 **선택도가 낮으면** 옵티마이저가 포기 | NULL 비율이 낮은 컬럼이면 유효, 높으면 부분 인덱스/별도 플래그 검토 |
| 함수 기반 정렬 | `ORDER BY UPPER(name)` | 인덱스 정렬 순서 사용 불가 → `filesort` | 정렬용 컬럼을 별도 저장(비정규화) 또는 정렬 기준 변경 |
| 서로 다른 인덱스 혼용 | `WHERE a=1 ORDER BY b` (a 인덱스, b 인덱스 따로) | 한 인덱스로 끝나지 않아 `filesort` | `(a, b)` 복합 인덱스로 통합 |

### 2-2. 커버링 인덱스를 깨뜨리는 흔한 실수

- **`SELECT *` / 엔티티 조회**: JPA에서 `findAll()`, `findById()`는 엔티티의 모든 컬럼을 SELECT 한다. 커버링 인덱스는 **DTO/id 프로젝션 쿼리에서만** 성립한다. "JPA를 쓰면 커버링 인덱스를 못 쓴다"가 아니라, **"엔티티를 통째로 조회하는 한 못 쓴다"** 가 정확하다.
- **프리픽스 인덱스는 커버링 불가**: `KEY idx_title (title(20))`은 20바이트까지만 저장하므로, 실제 값을 확인하려면 테이블 접근이 필요하다. 긴 문자열 전문을 인덱스에 넣어야 커버링이 된다.
- **인덱스에 없는 컬럼을 하나만 추가해도 끝**: `SELECT id, total_amount, memo`로 바꾸는 순간 커버링은 깨지고 재탐색이 부활한다. 화면 스펙이 바뀌면 실행 계획도 바뀐다는 것을 전제로 봐야 한다.
- **`SELECT COUNT(*)`도 커버링의 대상**: 인덱스 리프만 세면 되므로 가장 얇은 인덱스를 고르게 된다. 반대로 `COUNT(컬럼)`은 NULL 여부를 봐야 해 컬럼이 인덱스에 있어야 한다.
- **옵티마이저가 커버링을 포기하는 경우**: 결과 건수가 전체의 큰 비율이면 "인덱스 풀 스캔 + 커버링"보다 "테이블 풀 스캔"이 싸다고 판단한다. 이때 `Using index`가 안 나오는 것은 버그가 아니라 **비용 판단**이다. 인덱스 힌트(`USE INDEX`)로 강제하기 전에 정말 그게 빠른지 `EXPLAIN ANALYZE`로 확인한다.

### 2-3. 커버링 인덱스 도입 판단: 비용은 쓰기 쪽에서 발생한다

- 인덱스는 **읽기를 위해 쓰기를 희생**한다. 인덱스 컬럼을 하나 추가하면 `INSERT`/`UPDATE`/`DELETE`마다 해당 인덱스의 B+Tree도 갱신되고, 리프 페이지가 가득 차면 **페이지 분할(split)** 이 일어나 조각화와 쓰기 증폭이 생긴다.
- 무작위 PK(예: UUID)는 삽입 위치가 매번 달라 페이지 분할이 잦고, `AUTO_INCREMENT` PK는 항상 오른쪽 끝에 append 되어 분할이 드물다. **인덱스 설계는 PK 선택에서 이미 시작된다.**
- 커버링 인덱스는 "컬럼을 더 넣는" 최적화이므로 인덱스 크기가 커진다. 예를 들어 `INT`(4B) 하나를 1,000만 건 인덱스에 추가하면 리프에 약 40MB가 더 붙고(페이지 오버헤드 제외), 그만큼 버퍼 풀이 다른 페이지를 밀어낸다. **"조회 쿼리 1개를 위해 모든 쓰기와 캐시를 희생하는가"** 를 기준으로 판단한다.
- 판단 기준: (1) 그 쿼리가 **핫 패스**(주문 목록, 타임라인, 랭킹 조회)인가, (2) 초당 호출 수가 높은가, (3) 테이블이 버퍼 풀보다 큰가(랜덤 I/O가 실제 디스크로 내려가는가). 셋 다 '예'일 때 도입 가치가 크다. 하루 몇 번 도는 배치 조회라면 인덱스를 늘리기보다 쿼리를 고치는 편이 낫다.

### 2-4. 그 밖의 엣지 케이스

- **통계가 낡으면 계획이 틀어진다**: InnoDB는 인덱스 카디널리티 통계로 계획을 세운다. 대량 INSERT/DELETE 후 `ANALYZE TABLE orders`로 갱신하지 않으면 잘못된 인덱스를 고른다. `SHOW INDEX FROM orders`의 `Cardinality`가 실제 값과 크게 다르면 의심한다.
- **`IN` 리스트가 길면 인덱스 다이브를 건너뛴다**: `eq_range_index_dive_limit`(기본 200)를 넘는 `IN` 목록은 정확한 통계 대신 추정치를 써서 계획이 달라질 수 있다. 대량 `IN` 쿼리에서 갑자기 느려지면 이 파라미터를 확인한다.
- **`ORDER BY`에 컬럼이 섞이면 인덱스 정렬이 깨진다**: `ORDER BY status, ordered_at`처럼 인덱스 순서와 다른 조합, 또는 `ORDER BY a ASC, b DESC`처럼 방향이 섞인 경우(8.0은 내림차순 인덱스로 대응 가능) `filesort`가 붙는다.
- **`filesort`가 항상 느린 것은 아니다**: 정렬 대상이 `sort_buffer_size`에 들어가면 메모리 정렬로 끝난다. 느린 것은 **디스크 임시 파일로 넘어갈 때**다. `Using temporary; Using filesort`가 함께 보이면 특히 주의한다.
- **인덱스 머지(`index_merge`)는 만능이 아니다**: `OR` 조건에서 두 인덱스를 각각 스캔해 합치는 방식인데, 결과 집합이 크면 오히려 느리다. `type: index_merge`가 보이면 `UNION ALL` 분리를 검토한다.
- **`Using index for group-by`**: `GROUP BY` 컬럼이 인덱스 선두와 일치하면 인덱스만으로 그룹핑한다(loose index scan). `GROUP BY` 쿼리 최적화의 핵심도 결국 인덱스 순서다.
- **테스트 환경의 함정**: 소량 데이터에서는 커버링 인덱스의 이득이 거의 보이지 않고, 옵티마이저도 다른 계획을 고른다. **성능 검증은 실제 규모(수십만~수백만 건)와 콜드 캐시 조건**에서 해야 한다.

### 2-5. 아그작(AGZAK) 관점 메모

- 아그작의 Redis ZSET 랭킹 조회는 이미 인덱스가 필요 없는 경로지만, **랭킹 스냅샷을 RDB에 적재해 이력/통계를 조회하는 경로**라면 `(game_id, recorded_at)` 형태의 복합 인덱스에 점수·순위 컬럼을 붙여 커버링으로 끝낼 수 있다.
- JWT 인증 후 사용자별 데이터를 조회하는 API는 대부분 `WHERE user_id = ?`로 시작한다. 이때 `(user_id, ...)` 인덱스에 조회 컬럼을 붙여 커버링을 만들면, 세컨더리 인덱스에 PK가 이미 포함된다는 점 덕분에 **`SELECT id` 형태의 목록 API는 추가 인덱스 없이도 커버링이 된다.**
- ShedLock(`shedlock` 테이블)처럼 작은 테이블은 커버링 인덱스의 이득이 거의 없다. **최적화 대상은 데이터가 큰 테이블의 핫 쿼리**라는 우선순위를 지키는 것이 중요하다.

---

## 3. 기술 면접 단골 꼬리 질문 & 모범 답변 팁

**Q1. Hash Index가 동등 비교에 O(1)로 더 빠른데, 왜 MySQL InnoDB의 기본 인덱스는 B+Tree인가요?**
- **A.** Hash 인덱스는 해시값으로 버킷을 찾으므로 동등 비교(`=`)에서는 평균 O(1)로 B+Tree(O(log N))보다 빠릅니다. 하지만 (1) **범위 검색이 불가능**하고(`BETWEEN`, `>`, `<`는 전체 스캔), (2) **정렬 순서를 유지하지 못해 `ORDER BY`에 쓸 수 없고**, (3) 앞부분 일치(`LIKE 'abc%'`)도 안 되며, (4) 해시 충돌·리해싱 비용이 있고, (5) 데이터가 커지면 해시 테이블 전체를 메모리에 두기 어렵습니다. 실제 쿼리는 동등·범위·정렬이 섞여 있고, 디스크 기반 저장소에서는 "읽는 페이지 수"가 성능을 좌우하므로 B+Tree가 범용 기본값입니다. 참고로 InnoDB도 내부적으로 자주 쓰이는 페이지 접근을 해시로 캐시하는 **Adaptive Hash Index**를 두고 있고, `MEMORY` 엔진은 해시 인덱스를 지원합니다. "동등 비교만 하는 작은 테이블이면 해시가 유리"가 정확한 답의 마무리입니다.

**Q2. Clustered Index와 Non-Clustered(Secondary) Index의 차이는 무엇이고, 왜 클러스터드 인덱스는 테이블당 하나인가요?**
- **A.** 클러스터드 인덱스는 **row 데이터 자체를 PK 순서로 정렬해 저장한 구조**이고, 리프 노드에 row 전체가 있습니다. 그래서 "테이블 = 클러스터드 인덱스"이고 PK를 통한 조회는 B+Tree 탐색 한 번으로 끝납니다. 세컨더리 인덱스는 별도의 B+Tree로, 리프에 **인덱스 컬럼 + PK**만 저장합니다. 따라서 세컨더리 인덱스 조회는 리프에서 PK를 얻고 클러스터드 인덱스를 다시 타는 **이중 탐색(랜덤 I/O)** 이 필요합니다. 클러스터드 인덱스가 하나인 이유는 **row를 물리적으로 한 가지 순서로만 정렬해 저장할 수 있기 때문**입니다. 세컨더리 인덱스는 여러 개 만들 수 있지만(테이블당 최대 64개, 인덱스당 16개 컬럼) 그만큼 쓰기 비용과 저장 공간이 늘어납니다.

**Q3. 복합 인덱스에서 컬럼 순서가 왜 중요한가요? Leftmost Prefix Rule을 설명해 보세요.**
- **A.** B+Tree는 앞 컬럼부터 비교해 범위를 좁히므로, **인덱스는 왼쪽 컬럼부터 연속으로 조건이 있어야** 레인지 스캔이 됩니다. `(a, b, c)` 인덱스에서 `WHERE a=?`, `WHERE a=? AND b=?`, `WHERE a=? AND b=? AND c=?`는 인덱스를 타지만, `WHERE b=?`나 `WHERE b=? AND c=?`는 선행 컬럼이 빠져 레인지 스캔이 불가합니다. 또 `WHERE a=? AND c=?`처럼 중간이 비면 `a`까지만 범위로 쓰고 `c`는 인덱스 필터로만 쓰입니다. 순서를 정할 때는 (1) 동등 조건 컬럼을 앞에, 범위 조건을 뒤에, (2) 동등 조건끼리는 선택도가 높은 것(단, 순서를 바꿔도 결과는 같고 스캔량만 달라짐), (3) `ORDER BY`/`GROUP BY` 컬럼을 그 뒤에 배치해 정렬을 인덱스로 해결하는 순서로 정합니다. 최종 확인은 `EXPLAIN`의 `key`, `rows`, `Extra`로 합니다.

**Q4. 커버링 인덱스란 무엇이고, `LIMIT offset, size` 페이징 성능 개선에 어떻게 활용하나요?**
- **A.** 커버링 인덱스는 쿼리가 필요한 모든 컬럼을 하나의 인덱스가 포함해, **클러스터드 인덱스(테이블) 접근 없이 인덱스만으로 결과를 반환**하는 것입니다. 실행 계획에 `Extra: Using index`로 나타납니다. 이득은 세컨더리 리프에서 PK로 테이블을 재탐색하는 랜덤 I/O를 제거하는 것입니다.
  깊은 페이징에서 `SELECT * FROM t WHERE ... ORDER BY ... LIMIT 100000, 20`은 **버려질 10만 건의 본문 row까지 전부 읽습니다**. 여기서 두 가지를 겁니다. (1) 정렬·필터 조건과 `id`를 담은 인덱스로 **`SELECT id`만 먼저 뽑아** 오프셋을 커버링 인덱스 안에서 소화하고(이때 인덱스는 순차 읽기), (2) 그 20건의 `id`로 본문을 PK 조회(`IN`)합니다. 이것이 지연 조인(deferred join)입니다. 추가로 `Slice`(size+1 조회)로 `count(*)` 쿼리를 없애면 카운트 풀 스캔도 사라집니다. JPA에서는 엔티티가 아니라 `select o.id` 프로젝션 쿼리여야 커버링이 성립하고, 두 번째 단계의 컬렉션 로딩은 `default_batch_fetch_size`로 묶는 것이 안전합니다.

**Q5. 인덱스를 많이 만들면 안 되는 이유와, 인덱스 설계 시 판단 기준을 설명해 보세요.**
- **A.** 인덱스는 **읽기 성능을 쓰기 비용으로 사는 구조**입니다. (1) `INSERT`/`UPDATE`/`DELETE`마다 모든 인덱스의 B+Tree를 갱신해야 하므로 쓰기 증폭이 생기고, (2) 리프 페이지가 가득 차면 **페이지 분할**이 일어나 조각화와 추가 I/O가 발생하며, (3) 인덱스가 버퍼 풀과 디스크를 차지는 만큼 실제 데이터 페이지가 밀려나 캐시 효율이 떨어지고, (4) 옵티마이저가 고를 수 있는 계획이 늘어나 **잘못된 인덱스를 고를 확률**도 함께 올라갑니다. 판단 기준은 "핫 쿼리인가 × 호출 빈도 × 데이터 규모(버퍼 풀 대비)"이고, 그 조건을 만족하면 (a) `WHERE`-`ORDER BY`-`SELECT`를 한 인덱스로 묶은 복합/커버링 인덱스를 만들고, (b) 중복되거나 선행 컬럼이 겹치는 인덱스는 정리하며, (c) 도입 전후를 `EXPLAIN ANALYZE`와 `Handler_read%`로 실측합니다. **"인덱스 추가"보다 "쿼리·스키마 수정"이 먼저**라는 순서도 함께 말하면 좋습니다.

---

## 4. 오늘의 체크리스트 (내일 이어서)

- [ ] 1,000만 건 규모의 `orders` 더미 데이터를 만들고, 커버링 인덱스 유무에 따른 응답 시간·`Handler_read%` 차이를 수치로 기록
- [ ] 콜드 캐시(`innodb_buffer_pool_size`를 작게, 또는 `FLUSH TABLES` 후) 조건에서 동일 비교 → 랜덤 I/O 영향 정량화
- [ ] 0-4 지연 조인 페이징을 `LIMIT 100000, 20` 기준으로 단순 페이징과 비교 (JPA `Slice` vs `Page`의 count 쿼리 비용 포함)
- [ ] `(a, b)` vs `(b, a)` 인덱스에서 `WHERE a BETWEEN ... AND b = ...` 실행 계획이 어떻게 달라지는지 확인
- [ ] `Using index` / `Using index condition` / `Using where` / `Using filesort` / `Using temporary` / `Using index for group-by`가 각각 어떤 상황에 찍히는지 재현해 표로 정리
- [ ] `eq_range_index_dive_limit`를 넘는 대량 `IN` 쿼리에서 계획이 바뀌는지 실험
- [ ] UUID PK와 `BIGINT AUTO_INCREMENT` PK의 세컨더리 인덱스 크기·삽입 성능 비교 (페이지 분할 관찰)
