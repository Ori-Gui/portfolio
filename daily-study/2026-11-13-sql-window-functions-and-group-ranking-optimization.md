# [2026-11-13] SQL 윈도우 함수(Window Functions)와 그룹별 집계·순위 최적화

> **한 줄 핵심 요약**: 윈도우 함수는 **기존 행의 개수와 세부 정보를 보존하면서(Non-collapsing)** 특정 파티션 범위(Window) 내의 연산(순위, 누적합, 이전/이후 행 참조)을 수행하는 ANSI SQL 표준 기능이다. `GROUP BY`가 행을 하나의 요약 행으로 집약하는 반면, 윈도우 함수는 각 행에 집계 결과를 부착할 수 있어 서브쿼리·자기 조인(Self-Join)의 복잡성과 성능 비용을 대폭 줄인다. 실무 및 코딩테스트(프로그래머스/공기업 전산 실기)에서는 **그룹별 Top-N 추출(`ROW_NUMBER()`), 누적 합계(`SUM() OVER`), 시계열 변동 추적(`LAG`/`LEAD`)**이 핵심이며, 대용량 환경에서는 `PARTITION BY + ORDER BY`가 인덱스를 타지 못해 발생하는 임시 테이블(TempTable/Filesort) 부하를 막기 위해 **복합 인덱스 설계**와 **비결정적 순위를 방지하는 고유 키 정렬(`Tie-breaker`)**을 반드시 고려해야 한다.

---

## 1. 윈도우 함수의 핵심 개념과 실행 구조

### 1-1. GROUP BY vs WINDOW FUNCTION: 행 집약(Collapsing) 여부

관계형 데이터베이스에서 데이터를 그룹화하여 계산할 때 가장 큰 구조적 차이는 **"기존 행이 보존되는가"**이다.

| 비교 항목 | `GROUP BY` 집계 | 윈도우 함수 (`OVER()`) |
|---|---|---|
| **행 수 변화** | 그룹당 1개의 요약 행으로 압축(**Collapsing**) | 원본 테이블의 행 수가 그대로 유지됨(**Non-collapsing**) |
| **개별 행 접근** | 집계에 포함되지 않은 원본 컬럼 직접 조회 불가 (MySQL `ONLY_FULL_GROUP_BY` 위반) | 원본 행의 세부 컬럼과 집계 결과를 같은 행에서 동시에 조회 가능 |
| **적용 시점** | `WHERE` 이후, `HAVING` 이전 | `HAVING` 이후, `SELECT` 절 평가 시점 |
| **주요 용도** | 총합, 평균 등 요약 통계 리포트 | 그룹별 순위, 누적합(Running Total), 이동평균, 직전/직후 행 비교 |

```text
[GROUP BY]                        [WINDOW FUNCTION (OVER)]
원본 행 (5개)       집계 결과 (2개)    원본 행 (5개)       집계 결과 (5개 유지)
[Dept A, 100]                      [Dept A, 100] ───> [Dept A, 100, Sum: 250, Rank: 2]
[Dept A, 150] ───> [Dept A, 250]   [Dept A, 150] ───> [Dept A, 150, Sum: 250, Rank: 1]
[Dept B,  80]                      [Dept B,  80] ───> [Dept B,  80, Sum: 280, Rank: 3]
[Dept B,  90] ───> [Dept B, 280]   [Dept B,  90] ───> [Dept B,  90, Sum: 280, Rank: 2]
[Dept B, 110]                      [Dept B, 110] ───> [Dept B, 110, Sum: 280, Rank: 1]
```

과거 MySQL 5.7 이전이나 윈도우 함수를 쓰지 않던 시절에는 그룹별 1위 행을 뽑기 위해 다음과 같이 복잡한 서브쿼리와 자기 조인(Self-Join)을 수행해야 했다.

```sql
-- 과거 방식 (Self-Join + GROUP BY 서브쿼리)
-- 부서별 최고 연봉자의 세부 정보 조회
SELECT e.dept_id, e.emp_name, e.salary
FROM employee e
INNER JOIN (
    SELECT dept_id, MAX(salary) AS max_salary
    FROM employee
    GROUP BY dept_id
) m ON e.dept_id = m.dept_id AND e.salary = m.max_salary;
```
위 쿼리는 `employee` 테이블을 두 번 풀 스캔하거나 인덱스를 두 번 탐색해야 하므로 데이터가 수백만 건에 달하면 심각한 I/O 병목을 유발한다. 반면 윈도우 함수를 사용하면 테이블 단일 스캔(또는 적절한 인덱스 단일 스캔)으로 해결할 수 있다.

---

### 1-2. 윈도우 함수 문법과 프레임(Frame) 지정

윈도우 함수의 기본 구조는 다음과 같다.

```sql
FUNCTION_NAME([arguments]) OVER (
    [PARTITION BY partition_column_list]
    [ORDER BY sort_column_list [ASC | DESC]]
    [frame_clause]
)
```

1. **`PARTITION BY`**: 전체 행을 계산할 소그룹 단위로 나눈다. 지정하지 않으면 전체 테이블이 하나의 파티션이 된다.
2. **`ORDER BY`**: 파티션 내부에서 행의 정렬 순서를 지정한다. 순위 함수나 누적 계산 시 필수적이다.
3. **`frame_clause` (윈도우 프레임)**: 파티션 내에서 현재 행(Current Row)을 기준으로 실제 계산에 포함할 행의 범위를 세밀하게 제어한다.

#### ⚠️ Frame Clause 기본값의 치명적 함정: RANGE vs ROWS

`ORDER BY`를 지정하고 프레임을 생략했을 때 적용되는 ANSI SQL 표준 기본값은 다음과 같다.

```sql
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

여기서 **`ROWS`**와 **`RANGE`**의 차이를 정확히 알아야 성능 버그와 논리적 오류를 방지할 수 있다.

- **`ROWS`**: 물리적인 **행(Row)의 개수** 기준이다. `CURRENT ROW`는 오직 현재 처리 중인 단 1개의 물리 행을 의미한다.
- **`RANGE`**: 논리적인 **값(Value)의 범위** 기준이다. `CURRENT ROW`는 현재 행의 `ORDER BY` 컬럼 값과 **동일한 값을 가진 모든 행(Peer rows)**을 포함한다.

```sql
-- 동점자가 있을 때의 누적합 차이 예시
SELECT emp_name, salary,
       -- RANGE (기본값): salary가 같은 사람이 있으면 그 동점자들의 연봉까지 한 번에 더해버림
       SUM(salary) OVER (ORDER BY salary) AS sum_range_default,
       -- ROWS: 동점자 여부와 무관하게 현재 물리적 행까지만 정직하게 누적합 계산
       SUM(salary) OVER (ORDER BY salary ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS sum_rows
FROM employee;
```

| emp_name | salary | sum_range_default | sum_rows | 설명 |
|---|---|---|---|---|
| 철수 | 3000 | 3000 | 3000 | 첫 번째 행 |
| 영희 | 4000 | **11000** | 7000 | 영희와 민수의 salary가 4000으로 동일하여 |
| 민수 | 4000 | **11000** | 11000 | RANGE는 4000인 두 사람을 묶어 3000+4000+4000=11000을 출력 |
| 지훈 | 5000 | 16000 | 16000 | 정상 누적 |

실무 및 코딩테스트에서 단순 누적합(Running Total)을 구할 때는 의도치 않은 중복 합산을 막고 임시 정렬 연산 비용을 줄이기 위해 가급적 **`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`**를 명시하거나, `ORDER BY`에 고유 키(PK)를 함께 지정해 동점 행을 원천 차단해야 한다.

---

### 1-3. 쿼리 실행 순서와 윈도우 함수의 위치

SQL 질의의 논리적 실행 순서는 다음과 같다.

```text
① FROM (JOIN)
    │
    ▼
② WHERE
    │
    ▼
③ GROUP BY
    │
    ▼
④ HAVING
    │
    ▼
⑤ SELECT (산술 연산, Window Functions 계산)  <── 윈도우 함수는 여기서 실행!
    │
    ▼
⑥ DISTINCT
    │
    ▼
⑦ ORDER BY
    │
    ▼
⑧ LIMIT / OFFSET
```

#### 💡 왜 WHERE 절에 윈도우 함수를 직접 쓸 수 없는가?
- `WHERE` 절은 `FROM` 절에서 가져온 행을 필터링하는 단계(②)다.
- 반면 윈도우 함수는 `WHERE`, `GROUP BY`, `HAVING`을 모두 거친 뒤 완성된 결과 셋을 바탕으로 `SELECT` 시점(⑤)에 비로소 윈도우를 나누어 계산된다.
- 따라서 **`WHERE ROW_NUMBER() OVER (...) <= 3`과 같이 작성하면 `Unknown column` 또는 `Window function ... not allowed in WHERE` 문법 에러가 발생**한다.
- **해결책**: 반드시 **서브쿼리(인라인 뷰)**나 **CTE(`WITH`)**로 감싼 뒤 바깥쪽 `WHERE` 절에서 필터링해야 한다.

```sql
-- 올바른 패턴: CTE(Common Table Expression)를 활용한 필터링
WITH ranked_emp AS (
    SELECT dept_id, emp_name, salary,
           ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
    FROM employee
)
SELECT dept_id, emp_name, salary
FROM ranked_emp
WHERE rnk <= 3;
```

---

## 2. 핵심 윈도우 함수 분류 및 실전 패턴

### 2-1. 순위 함수 (Ranking Functions)

가장 빈번하게 출제되고 실무에서 쓰이는 세 가지 순위 함수의 차이는 **동점자(Tie) 처리 방식**이다.

| 함수 | 동점자 순위 처리 | 다음 순위 번호 | 예시 (점수: 100, 90, 90, 80) |
|---|---|---|---|
| **`ROW_NUMBER()`** | 고유한 연속 번호 부여 (동점 무시) | 1, 2, 3, 4 | **1등, 2등, 3등, 4등** |
| **`RANK()`** | 동점자는 같은 순위, 다음 순위 건너뜀 | 1, 2, 2, 4 | **1등, 2등, 2등, 4등** (3등 누락) |
| **`DENSE_RANK()`** | 동점자는 같은 순위, 다음 순위 건너뛰지 않음 | 1, 2, 2, 3 | **1등, 2등, 2등, 3등** (연속성 유지) |

#### ⚠️ ROW_NUMBER()의 비결정적 정렬(Non-deterministic) 버그
`ROW_NUMBER()`를 쓸 때 `ORDER BY`에 지정한 컬럼에 동일한 값이 존재하면, 데이터베이스 엔진의 스캔 순서나 인덱스 상태에 따라 매 실행 시마다 행의 순위가 달라질 수 있다.
- 예를 들어, 영희와 민수가 둘 다 90점인데 `ORDER BY score DESC`만 주면 어떤 날은 영희가 2등, 어떤 날은 민수가 2등이 된다.
- **실무 해결 원칙 (Tie-breaker)**: 순위가 고정되어야 하는 페이지네이션이나 배치 작업에서는 반드시 고유한 컬럼(PK 등)을 2차 정렬 키로 추가해야 한다.
  ```sql
  -- 나쁜 예: 비결정적 결과 발생
  ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY score DESC)
  
  -- 좋은 예: 결정적(Deterministic) 결과 보장
  ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY score DESC, emp_id ASC)
  ```

---

### 2-2. 행 탐색 함수 (Value Navigation Functions)

시계열 데이터 분석이나 사용자 행동 추적에서 이전/이후 행의 데이터를 가져올 때 사용한다.

#### ① `LAG()` & `LEAD()`
- `LAG(col, offset, default)`: 현재 행 기준으로 `offset`만큼 **이전** 행의 값을 가져옴 (기본 offset=1).
- `LEAD(col, offset, default)`: 현재 행 기준으로 `offset`만큼 **이후** 행의 값을 가져옴 (기본 offset=1).

```sql
-- 실전 예제: 사용자의 직전 로그인 시각과의 차이(간격) 계산
SELECT user_id, login_time,
       LAG(login_time, 1) OVER (PARTITION BY user_id ORDER BY login_time) AS prev_login_time,
       TIMESTAMPDIFF(
           HOUR,
           LAG(login_time, 1) OVER (PARTITION BY user_id ORDER BY login_time),
           login_time
       ) AS hours_since_last_login
FROM user_login_log;
```

#### ② `FIRST_VALUE()` & `LAST_VALUE()`
- `FIRST_VALUE(col)`: 윈도우 프레임의 첫 번째 값을 반환.
- `LAST_VALUE(col)`: 윈도우 프레임의 마지막 값을 반환.

#### ⚠️ `LAST_VALUE()` 사용 시의 흔한 실수
많은 개발자들이 `LAST_VALUE(salary) OVER (PARTITION BY dept_id ORDER BY salary)`를 실행하고 "왜 그룹 전체의 최고/최저 연봉이 안 나오고 현재 행의 연봉이 나오는가?"라고 당황한다.
- 원인은 앞서 다룬 **기본 프레임(`RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`)** 때문이다.
- 기본 프레임의 끝이 `CURRENT ROW`로 닫혀 있으므로, `LAST_VALUE`는 항상 '현재 행'을 가리키게 된다.
- **해결책**: 파티션 전체의 끝까지 프레임을 확장해야 한다.
  ```sql
  -- 파티션 전체의 마지막 행을 참조하려면 프레임 명시 필수
  LAST_VALUE(salary) OVER (
      PARTITION BY dept_id 
      ORDER BY salary 
      ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  )
  ```

---

### 2-3. 집계 윈도우 함수 (Aggregate Window Functions)

`SUM()`, `AVG()`, `COUNT()`, `MAX()`, `MIN()`을 `OVER()`와 결합하면 요약과 디테일을 한 번에 다룰 수 있다.

#### ① 전체 대비 비율 구하기
```sql
-- 개별 사원의 연봉이 해당 부서 전체 연봉 총합에서 차지하는 백분율
SELECT dept_id, emp_name, salary,
       SUM(salary) OVER (PARTITION BY dept_id) AS dept_total_salary,
       ROUND(salary / SUM(salary) OVER (PARTITION BY dept_id) * 100, 2) AS pct_of_dept
FROM employee;
```

#### ② 이동 평균 (Moving Average)
최근 N일간의 주가나 매출 추세를 평활화(Smoothing)할 때 사용한다.
```sql
-- 현재 일자를 포함하여 직전 2일부터 오늘까지(3일간)의 이동평균
SELECT sale_date, daily_revenue,
       AVG(daily_revenue) OVER (
           ORDER BY sale_date
           ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ) AS moving_avg_3days
FROM daily_sales;
```

---

## 3. 실무 최적화 및 주의사항 (Performance & Edge Cases)

### 3-1. 인덱스 최적화와 실행 계획 (Explain Plan)

윈도우 함수는 대용량 테이블에서 잘못 사용하면 수 기가바이트(GB)의 임시 테이블(TempTable)과 디스크 기반 파일소트(Filesort)를 일으킨다.

MySQL 8.0 이상에서 윈도우 함수 쿼리의 `EXPLAIN FORMAT=TREE`를 확인하면 다음과 같은 연산자가 나타난다.

```text
-> Window aggregate: row_number() OVER (PARTITION BY dept_id ORDER BY salary DESC)
    -> Sort: employee.dept_id, employee.salary DESC
        -> Table scan on employee
```

- 만약 인덱스가 없다면: 전체 테이블을 읽어 메모리/디스크 임시 테이블에 적재한 뒤 `dept_id, salary DESC`로 전체 정렬(**Filesort**)을 수행한다.
- **인덱스 최적화 공식**:
  윈도우 함수의 `PARTITION BY` 컬럼과 `ORDER BY` 컬럼 순서대로 **복합 인덱스(Composite Index)**를 생성해야 정렬 작업을 생략(Streaming evaluation)할 수 있다.
  ```sql
  -- 최적 인덱스 설계
  CREATE INDEX idx_emp_dept_salary ON employee (dept_id, salary DESC);
  ```
  이 인덱스가 있으면 옵티마이저는 별도의 정렬 단계 없이 B+Tree 인덱스를 순서대로 읽으면서 즉시 윈도우 함수 연산을 수행하므로 속도가 수십~수백 배 향상된다.

---

### 3-2. MySQL 8.0 윈도우 함수 메모리 버퍼

MySQL 8.0은 윈도우 함수를 평가할 때 `windowing_buffer_size` (기본값 256KB) 시스템 변수를 사용한다.
- 윈도우 연산 중 필요한 행 데이터가 이 버퍼 크기를 초과하면 임시 디스크 파일(Disk-based temporary table)을 생성하여 쓰기/읽기 I/O가 폭증한다.
- **실무 팁**: 대용량 배치 처리나 분석 쿼리 실행 전 세션 단위로 버퍼 크기를 상향 조정하거나, `SELECT` 목록에서 불필요한 대형 컬럼(TEXT, BLOB 등)을 배제하고 인덱스 컬럼 위주로 커버링 윈도우 연산을 유도하는 것이 유리하다.

---

### 3-3. 프로그래머스 및 실기시험 빈출 엣지 케이스 체크리스트

1. **NULL 값의 정렬 위치**:
   - MySQL에서는 `ORDER BY col ASC` 시 **NULL이 가장 먼저(가장 작은 값)** 나오고, `DESC` 시 가장 나중에 나온다.
   - Oracle은 반대로 기본이 `NULLS LAST`이거나 문법적으로 `NULLS FIRST / NULLS LAST`를 지원하지만, MySQL 8.0은 이를 지원하지 않는다.
   - MySQL에서 NULL을 강제로 뒤로 보내려면 `ORDER BY (col IS NULL), col ASC` 패턴을 사용한다.
2. **카테고리별 최댓값 행만 추출 (Top 1)**:
   - `ROW_NUMBER() = 1`을 쓰면 동점자 중 임의의 1명만 나온다.
   - 문제 요구사항이 "동점자가 있으면 모두 출력할 것"인 경우 반드시 **`DENSE_RANK() = 1`** 또는 **`RANK() = 1`**을 써야 한다.
3. **WINDOW 절의 재사용 (Named Window)**:
   - 동일한 `OVER` 절을 여러 집계 함수에서 반복 사용할 경우 MySQL 8.0의 `WINDOW` 절로 묶어 가독성을 높이고 실행 계획을 최적화할 수 있다.
   ```sql
   SELECT emp_name,
          SUM(salary) OVER w AS dept_sum,
          AVG(salary) OVER w AS dept_avg,
          COUNT(*) OVER w AS dept_count
   FROM employee
   WINDOW w AS (PARTITION BY dept_id ORDER BY hire_date);
   ```

---

## 4. 기술 면접 단골 꼬리 질문 & 모범 답변

### Q1. `GROUP BY`와 윈도우 함수(`OVER`)의 근본적인 차이는 무엇인가요?
- **답변 요약**: `GROUP BY`는 집계 기준 컬럼에 따라 결과 행을 단일 행으로 압축(**Collapsing**)하므로 집계되지 않은 개별 원본 행의 세부 데이터를 조회할 수 없습니다. 반면 윈도우 함수는 원본 행의 구조와 개수를 1:1로 유지(**Non-collapsing**)하면서 그룹별 통계나 순위, 이전/이후 행 데이터를 부가적인 컬럼으로 부착할 수 있다는 점이 가장 큰 차이입니다.

---

### Q2. `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`의 동점자 처리 차이를 설명해 주세요.
- **답변 요약**: 점수가 [100, 90, 90, 80]일 때:
  - `ROW_NUMBER()`는 동점 여부와 상관없이 고유 번호를 매겨 **[1, 2, 3, 4]**가 됩니다.
  - `RANK()`는 동점자에게 동일 순위를 부여하되 다음 순위를 건너뛰어 **[1, 2, 2, 4]**가 됩니다.
  - `DENSE_RANK()`는 동점자에게 동일 순위를 주되 순위를 건너뛰지 않아 **[1, 2, 2, 3]**이 됩니다.
  - 실무에서 `ROW_NUMBER()`를 쓸 때 정렬 키에 고유성이 없으면 결과가 비결정적(Non-deterministic)이 되므로 PK를 2차 정렬 키로 반드시 추가해야 합니다.

---

### Q3. `WHERE` 절에서 윈도우 함수를 직접 필터링할 수 없는 이유와 해결 방법은 무엇인가요?
- **답변 요약**: SQL 쿼리 실행 라이프사이클 상 `WHERE` 절(행 필터링)이 `SELECT` 절(윈도우 함수 계산)보다 먼저 실행되기 때문입니다. 윈도우 함수 연산 시점에는 아직 윈도우가 만들어지지 않아 문법 에러가 발생합니다. 따라서 서브쿼리(인라인 뷰)나 CTE(`WITH`)로 윈도우 함수 결과를 먼저 도출한 뒤 바깥쪽 `WHERE` 절에서 해당 컬럼을 필터링해야 합니다.

---

### Q4. 윈도우 함수를 사용할 때 성능 병목이 발생하는 원인과 최적화 방안은 무엇인가요?
- **답변 요약**: `PARTITION BY`와 `ORDER BY`를 수행하기 위해 데이터베이스가 전체 데이터를 임시 테이블(TempTable)에 복사하고 파일소트(Filesort)를 거치면서 디스크 I/O와 메모리 부하가 발생하기 때문입니다. 이를 해결하려면 `PARTITION BY` 컬럼과 `ORDER BY` 컬럼의 순서에 맞춰 **복합 B+Tree 인덱스를 생성**하여, 정렬 과정 없이 인덱스 스캔만으로 윈도우 함수를 즉시 평가(Streaming Evaluation)하도록 유도해야 합니다.

---

### Q5. 누적합을 구할 때 `ROWS`와 `RANGE`의 차이는 무엇인가요?
- **답변 요약**: `ORDER BY` 지정 시 기본 프레임은 `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`입니다. `RANGE`는 정렬 컬럼 값이 동일한 모든 행(Peer rows)을 한 번에 묶어 처리하므로 동점자가 있으면 누적합이 중복 합산되는 왜곡이 생깁니다. 반면 `ROWS`는 물리적인 행 개수 단위로 엄격히 누적하므로 동점자가 있어도 정직하게 행 단위 누적합이 계산됩니다. 따라서 의도치 않은 버그 방지를 위해 명시적으로 `ROWS` 프레임을 선언하는 것이 안전합니다.
