# [2026-11-06] 분산 환경에서의 동시성 제어 전략 비교: 낙관적 락 vs 비관적 락 vs Redis 분산락(Redisson)

> **한 줄 핵심 요약**: 낙관적 락(`@Version`)은 충돌이 적을 때 유리하지만 재시도 코드를 직접 작성해야 하고, 비관적 락(`PESSIMISTIC_WRITE`)은 충돌이 많아도 정합성이 확실하지만 DB 커넥션을 점유해 데드락과 풀 고갈 위험이 있으며, Redis 분산락(Redisson)은 DB 부하 없이 프로세스 간 상호배제를 얻지만 **락 해제 시점이 트랜잭션 커밋보다 빨라지는 동시성 누수**를 반드시 AOP로 막아야 한다 — 세 전략은 "충돌 빈도 × 지연 시간 요구 × 인프라"의 트레이드오프다.

---

## 0. 코드 예제 (전체 실행 가능 버전)

### 0-1. 의존성 및 설정

```groovy
// build.gradle
// dependencies {
//     implementation 'org.springframework.boot:spring-boot-starter-data-jpa:3.3.x'
//     implementation 'org.springframework.boot:spring-boot-starter-web:3.3.x'
//     implementation 'org.springframework.boot:spring-boot-starter-aop'          // 락 해제 시점 제어용 AOP
//     implementation 'org.redisson:redisson-spring-boot-starter:3.30.0'          // Redis 분산락
//     implementation 'org.springframework.boot:spring-boot-starter-data-redis'   // Redisson starter가 redis를 끌어온다
//     runtimeOnly   'com.mysql:mysql-connector-j'
// }
```

```yaml
# src/main/resources/application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/study?characterEncoding=UTF-8
    hikari:
      maximum-pool-size: 10   # 비관적 락 사용 시 풀 크기가 곧 동시 처리 상한이 된다
  jpa:
    open-in-view: false
    properties:
      hibernate:
        # 비관적 락 타임아웃: innodb_lock_wait_timeout(기본 50초)보다 짧게
        jakarta.persistence.lock.timeout: 3000   # 3초 대기 후 예외 (ms 단위)
```

### 0-2. 공통 대상 엔티티와 리포지토리 (낙관적/비관적 락)

```java
// 파일 경로: src/main/java/study/lock/domain/Stock.java
package study.lock.domain;

import jakarta.persistence.*;
import study.lock.domain.exception.OutOfStockException;

@Entity
public class Stock {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long productId;

    private Long quantity;

    // [전략 1] 낙관적 락: 버전 컬럼. UPDATE 시 WHERE 절에 version이 붙는다.
    @Version
    private Long version;

    protected Stock() {}

    public Stock(Long productId, Long quantity) {
        this.productId = productId;
        this.quantity = quantity;
    }

    public void decrease(Long amount) {
        if (quantity - amount < 0) {
            throw new OutOfStockException("재고 부족: 현재 " + quantity);
        }
        quantity -= amount;
    }

    public Long getId() { return id; }
    public Long getVersion() { return version; }
}
```

```java
// 파일 경로: src/main/java/study/lock/repository/StockRepository.java
package study.lock.repository;

import jakarta.persistence.LockModeType;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Query;

public interface StockRepository extends JpaRepository<Stock, Long> {

    // [전략 1] 낙관적 락은 @Version만으로 동작하므로 별도 쿼리 불필요.
    // 재시도 제어가 필요할 때 LockModeType.OPTIMISTIC을 명시하기도 한다.

    // [전략 2] 비관적 락: SELECT ... FOR UPDATE. 트랜잭션이 끝날 때까지 Row Lock 유지.
    // NOWAIT / SKIP LOCKED 옵션도 있다 (하이버네이트 6.4+).
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select s from Stock s where s.productId = :productId")
    Stock findByProductIdWithPessimisticLock(Long productId);
}
```

```java
// 파일 경로: src/main/java/study/lock/service/OptimisticStockService.java
package study.lock.service;

import org.springframework.orm.ObjectOptimisticLockingFailureException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OptimisticStockService {

    private final StockRepository stockRepository;

    public OptimisticStockService(StockRepository stockRepository) {
        this.stockRepository = stockRepository;
    }

    // [전략 1] 낙관적 락 + 수동 재시도.
    // 주의: 재시도에는 새로운 트랜잭션이 필요하다. 같은 트랜잭션 안에서 재시도하면
    // 이미 rollback-only로 마킹된 컨텍스트를 재사용해 UnexpectedRollbackException이 난다.
    // 그래서 REQUIRES_NEW로 재시도 전용 메서드를 분리한다.
    public void decreaseWithRetry(Long productId, Long amount) {
        int maxRetry = 50;          // 재시도 상한. 무한 재시도는 스레드를 재고 소진 시점까지 점유한다
        long backoffMs = 40;

        for (int i = 0; i < maxRetry; i++) {
            try {
                tryDecrease(productId, amount);
                return;
            } catch (ObjectOptimisticLockingFailureException e) {
                try {
                    Thread.sleep(backoffMs + (i % 5) * 10);  // 백오프 + 지터
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new IllegalStateException("재시도 인터럽트", ie);
                }
            }
        }
        throw new IllegalStateException("낙관적 락 재시도 " + maxRetry + "회 초과");
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    protected void tryDecrease(Long productId, Long amount) {
        Stock stock = stockRepository.findById(productId).orElseThrow();
        stock.decrease(amount);
        // 커밋 시점에 하이버네이트가
        // UPDATE stock SET quantity=?, version=? WHERE id=? AND version=?
        // 을 날리고, 갱신 row가 0건이면 OptimisticLockException(=StaleObjectStateException) 발생.
    }
}
```

```java
// 파일 경로: src/main/java/study/lock/service/PessimisticStockService.java
package study.lock.service;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class PessimisticStockService {

    private final StockRepository stockRepository;

    public PessimisticStockService(StockRepository stockRepository) {
        this.stockRepository = stockRepository;
    }

    // [전략 2] 비관적 락. SELECT 시점에 FOR UPDATE로 Row Lock을 걸고
    // 트랜잭션 커밋/롤백까지 유지하므로 재시도 코드가 필요 없다.
    // 대가: (1) 락 대기 중 DB 커넥션 점유, (2) 잠금 순서가 엇갈리면 데드락,
    // (3) 타임아웃 설정(lock.timeout / innodb_lock_wait_timeout)이 사실상 필수.
    @Transactional
    public void decrease(Long productId, Long amount) {
        Stock stock = stockRepository.findByProductIdWithPessimisticLock(productId);
        stock.decrease(amount);
    } // 커밋(=락 해제)이 메서드 끝에서 일어나므로 락-커밋 시점이 일치한다. 분산락과의 결정적 차이.
}
```

### 0-3. [전략 3] Redis 분산락 (Redisson) — 여기서 대부분의 실무 사고가 난다

```java
// 파일 경로: src/main/java/study/lock/service/RedissonStockService.java
package study.lock.service;

import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
public class RedissonStockService {

    private final RedissonClient redissonClient;
    private final StockTxService stockTxService;  // 트랜잭션은 별도 빈으로 분리!

    public RedissonStockService(RedissonClient redissonClient, StockTxService stockTxService) {
        this.redissonClient = redissonClient;
        this.stockTxService = stockTxService;
    }

    public void decrease(Long productId, Long amount) {
        String key = "lock:stock:" + productId;
        RLock lock = redissonClient.getLock(key);

        try {
            // tryLock(waitTime, leaseTime, unit)
            //  - waitTime: 락 획득 대기 시간. Pub/Sub 채널 구독으로 락 해제 알림을 받고 재시도하므로 스핀락이 아니다.
            //  - leaseTime 지정 시: 지정 시간 후 자동 해제(만료). Watchdog이 작동하지 않는다.
            //  - leaseTime 생략 시(-1): Watchdog이 30초(기본) TTL을 주기적으로 갱신해 작업 중 만료를 막는다.
            boolean acquired = lock.tryLock(3, TimeUnit.SECONDS);
            if (!acquired) {
                throw new IllegalStateException("락 획득 실패: " + key);
            }

            stockTxService.doDecrease(productId, amount);  // 반드시 다른 빈의 @Transactional 메서드 호출
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("락 대기 인터럽트", e);
        } finally {
            if (lock.isLocked() && lock.isHeldByCurrentThread()) {  // 내가 잡은 락인지 확인 필수
                lock.unlock();
            }
        }
    }
}
```

```java
// 파일 경로: src/main/java/study/lock/service/StockTxService.java
package study.lock.service;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class StockTxService {

    private final StockRepository stockRepository;

    public StockTxService(StockRepository stockRepository) {
        this.stockRepository = stockRepository;
    }

    @Transactional
    public void doDecrease(Long productId, Long amount) {
        Stock stock = stockRepository.findById(productId).orElseThrow();
        stock.decrease(amount);
    } // ← 여기서 커밋. 이 커밋이 끝나야 Redis 락을 풀 수 있다.
}
```

### 0-4. [실무 엣지 케이스 핵심] 락 해제가 커밋보다 빠른 동시성 누수와 AOP 해결

```java
// 파일 경로: src/main/java/study/lock/lock/LockAfterCommitHelper.java
//
// 문제 상황: 아래와 같이 @Transactional 메서드 안에서 락을 잡고 finally에서 unlock하면
//
//   @Transactional
//   public void decrease(...) {
//       lock.tryLock(...);
//       try { /* 재고 감소 */ } finally { lock.unlock(); }
//   }  ← 사실은 여기서 커밋!
//
//  시간선이 이렇게 된다:
//   1. TX 시작 → 재고 감소 (아직 커밋 전)
//   2. unlock() 호출 → 다른 스레드가 락 획득
//   3. (2)의 스레드가 재고를 읽음 → 아직 커밋 전이라 "갱신 전" 값을 본다
//   4. (1)의 TX 커밋 → (2)가 계산한 값과 무관하게 커밋되어 **갱신 유실(lost update)**
//
// 즉 분산락을 걸었는데도 동시성 제어가 뚫린다. 락의 보호 범위가 "커밋"까지여야 하기 때문.
//
// 해결: 락 획득/해제를 트랜잭션 바깥에서 수행한다. 두 가지 방법.
//   (a) 위 0-3처럼 락 담당 메서드와 @Transactional 메서드를 **빈 경계로 분리** (가장 단순, 표준)
//   (b) 트랜잭션 동기화로 커밋 완료 이후에 unlock — 트랜잭션 컨트롤러가 분리되지 않은 레거시 코드에 적용할 때

package study.lock.lock;

import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.stereotype.Component;
import org.springframework.transaction.support.TransactionSynchronization;
import org.springframework.transaction.support.TransactionSynchronizationManager;

@Component
public class LockAfterCommitHelper {

    private final RedissonClient redissonClient;

    public LockAfterCommitHelper(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }

    /**
     * 트랜잭션 안에서 호출하면 afterCommit까지 락 해제를 미루고,
     * 트랜잭션 밖이면 즉시 해제한다.
     */
    public void unlockAfterCommit(String key, RLock lock) {
        if (TransactionSynchronizationManager.isSynchronizationActive()) {
            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
                @Override
                public void afterCommit() {
                    if (lock.isLocked() && lock.isHeldByCurrentThread()) {
                        lock.unlock();
                    }
                }

                @Override
                public void afterCompletion(int status) {
                    // 롤백인 경우에도 락은 반드시 풀어야 한다 (afterCommit은 롤백에서 호출 안 됨)
                    if (status == STATUS_ROLLED_BACK && lock.isLocked() && lock.isHeldByCurrentThread()) {
                        lock.unlock();
                    }
                }
            });
        } else {
            if (lock.isLocked() && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

```java
// 파일 경로: src/main/java/study/lock/lock/DistributedLockAspect.java
// (a)를 범용화한 어노테이션 기반 분산락 AOP.
// [AOP: 락 획득] → [빈 경계: @Transactional] → [커밋] → [AOP: unlock] 순서가 되도록
// @Order를 트랜잭션 AOP(기본 Ordered.LOWEST_PRECEDENCE)보다 바깥(작은 값)으로 잡는다.
package study.lock.lock;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;
import java.util.concurrent.TimeUnit;

@Aspect
@Component
@Order(0)  // 숫자가 작을수록 바깥. 트랜잭션 AOP보다 바깥에서 락을 잡아야
public class DistributedLockAspect {  // "락 획득 → TX 시작 → 커밋 → 락 해제" 순서가 성립한다.

    private final RedissonClient redissonClient;

    public DistributedLockAspect(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }

    @Around("@annotation(study.lock.lock.DistributedLock)")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        Method method = ((MethodSignature) pjp.getSignature()).getMethod();
        DistributedLock anno = method.getAnnotation(DistributedLock.class);

        // SpEL 대신 단순화: 파라미터를 키로 사용 (실무에서는 KeyParser/SpEL 평가가 붙는다)
        String key = "lock:" + anno.keyPrefix() + ":" + pjp.getArgs()[0];
        RLock lock = redissonClient.getLock(key);

        boolean acquired = false;
        try {
            acquired = lock.tryLock(anno.waitTime(), anno.leaseTime(), TimeUnit.SECONDS);
            if (!acquired) {
                throw new IllegalStateException("락 획득 실패: " + key);
            }
            return pjp.proceed();
        } finally {
            // proceed()가 반환된 시점 = 이 프록시가 감싼 트랜잭션 커밋 완료 시점이므로
            // finally의 unlock이 안전하다.
            // (0-4의 문제는 "@Transactional 메서드 내부에서 unlock"하는 경우에만 발생한다)
            if (acquired && lock.isLocked() && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

```java
// 파일 경로: src/main/java/study/lock/lock/DistributedLock.java
package study.lock.lock;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributedLock {
    String keyPrefix();
    long waitTime() default 3;   // 락 획득 최대 대기 (초)
    long leaseTime() default -1; // -1이면 Watchdog 활성
}
```

---

## 1. 개념 및 동작 원리

### 1-1. 세 전략 비교표

| 구분 | 낙관적 락 (`@Version`) | 비관적 락 (`PESSIMISTIC_WRITE`) | Redis 분산락 (Redisson RLock) |
|---|---|---|---|
| 잠금 단위 | 버전 컬럼 (충돌 감지) | DB Row Lock (`SELECT ... FOR UPDATE`) | Redis 키 (`SET NX PX` + Pub/Sub) |
| 충돌 처리 | 커밋 시 실패 → **재시도 코드 직접 작성** | 대기(blocking) or 타임아웃 예외 | 대기(waitTime) 후 실패 반환 |
| 재시도 비용 | 스레드가 폴링 → 충돌 많으면 CPU 낭비 | 불필요 | 불필요 (Pub/Sub 알림 기반) |
| DB 커넥션 | 짧게 점유 | **대기 내내 점유** (풀 고갈 위험) | 점유 안 함 |
| 데드락 | 없음 | **있음** (다중 락, 순서 불일치) | 없음 (단일 키, TTL로 만료 보장) |
| 락-커밋 시점 | 커밋과 동일 (일치) | 커밋과 동일 (일치) | **불일치 가능** (unlock이 커밋보다 빠르면 유실) |
| 적합한 상황 | 충돌 적음, 읽기 위주 | 충돌 많음, 정합성 최우선, 단일 DB | 다중 인스턴스, DB 부하 분산, 대기 시간 짧게 |

### 1-2. 낙관적 락: 충돌을 막는 게 아니라 감지한다

- `@Version` 컬럼이 있으면 하이버네이트의 UPDATE 쿼리는 `WHERE id = ? AND version = ?` 형태가 된다. 다른 트랜잭션이 먼저 커밋해 version이 바뀌었으면 갱신 row가 0건 → `OptimisticLockException`(`ObjectOptimisticLockingFailureException`)이 트랜잭션 커밋 시점에 발생한다.
- 핵심 트레이드오프: **충돌이 없으면 락 비용이 0**이지만, 충돌이 잦으면(예: 인기 상품 재고, 좌석 예매) 재시도 폭풍이 난다. 재시도 N회 × 동시 스레드 수만큼 DB SELECT가 늘어나고, 실패 임계값을 넘은 요청은 결국 사용자에게 에러로 돌아간다.
- 재시도 구현 시 함정: 재시도에 `REQUIRES_NEW` 없이 같은 트랜잭션을 재사용하면, 실패한 시점에 이미 `rollback-only`로 마킹된 영속 컨텍스트를 그대로 쓰게 되어 `UnexpectedRollbackException`이 난다. 재시도는 반드시 **새 트랜잭션**에서.

### 1-3. 비관적 락: 정합성은 확실하지만 DB에 빚을 진다

- `PESSIMISTIC_WRITE`는 `SELECT ... FOR UPDATE`로 변환된다. Row Lock을 트랜잭션 종료까지 유지하므로 다른 트랜잭션의 일반 읽기는 통과시키지만, 같은 Row에 대한 쓰기/잠금 읽기는 전부 대기한다.
- 비용 ① **커넥션 점유**: 락 대기 시간 동안 커넥션 풀에서 커넥션을 빌려 둔 채 놓지 않는다. 히카리 기본 10커넥션 중 절반이 인기 상품 락 대기에 묶이면, 나머지 API 전체가 커넥션 대기로 함께 막힌다. 비관적 락의 실패는 "쿼리 실패"가 아니라 **서비스 전체 장애**로 번질 수 있다.
- 비용 ② **데드락**: 하나의 트랜잭션이 여러 Row를 FOR UPDATE로 잠글 때 획득 순서가 엇갈리면 상호 대기가 발생한다. InnoDB가 데드락을 감지해 한쪽을 롤백시키지만 애플리케이션 입장에서는 예외 처리 대상이다. **항상 같은 순서(예: id 오름차순)로 잠그는 것이 원칙.**
- 타임아웃은 필수다. `jakarta.persistence.lock.timeout` + MySQL `innodb_lock_wait_timeout`을 애플리케이션 요구 대기 시간(수백 ms ~ 수 초)에 맞춰 짧게 잡아야 풀 고갈을 막는다.

### 1-4. Redis 분산락: 왜 Redisson인가, Watchdog은 뭘 하는가

- 단순 구현(`SET key value NX EX 10`)은 세 가지 문제가 있다: (1) 락이 만료될 때까지 **스핀하며** 폴링하거나, (2) 작업이 오래 걸리면 락이 **먼저 만료되어** 다른 스레드가 진입하고, (3) 만료된 락을 남이 풀어버리는 문제(value 검증으로 완화).
- **Redisson**은 세 문제를 다룬다:
  - **Pub/Sub 기반 대기**: 락을 잡지 못한 스레드는 Redis 채널(`redisson_lock__channel:{key}`)을 구독하고, 락이 해제되면 그때 재시도한다. 스핀락이 아니므로 Redis 부하가 낮고 대기 반응이 빠르다.
  - **Watchdog**: `tryLock`에서 `leaseTime`을 **생략**하면 락 TTL 30초(기본)를 내부 스케줄러가 10초 간격으로 계속 갱신해, 작업이 오래 걸려도 락이 먼저 만료되지 않게 한다. **`leaseTime`을 명시하면 Watchdog이 꺼진다** — "작업이 몇 초 걸릴지 예측 가능한가"가 leaseTime 명시 여부의 기준이다.
  - **`isHeldByCurrentThread()`**: unlock 전 반드시 확인한다. Watchdog 갱신 실패로 락이 이미 해제되었거나, 다른 스레드가 같은 키를 잡은 상황에서 unlock하면 `IllegalMonitorStateException` 또는 남의 락을 푸는 사고가 난다.
- 한계: 단일 Redis에 의존하면 **Redis 장애 = 락 기능 상실**이고, 마스터-레플리카 구조에서는 **락이 레플리카에 복제되기 전 마스터 장애 → 페일오버 → 락 유실** 가능성이 있다(이를 다루는 것이 RedLock 논쟁). 실무에서는 "분산락은 절대적 보장이 아니라 확률적 상호배제"로 보고, 재고 감소 같은 정합성에는 DB 단의 낙관적 락/유니크 제약/조건부 UPDATE를 **병행**하는 것이 정답에 가깝다.

---

## 2. 실무 적용 및 주의사항 (Edge Cases & 동시성)

### 2-1. 치명적 함정: `unlock()`이 `commit()`보다 빠르면 분산락이 뚫린다

- 0-4에서 본 그대로, `@Transactional` 메서드 **내부**에서 finally로 unlock하면 시간선이 "TX 시작 → 감소 → unlock → (다른 스레드 진입·조회) → 커밋"이 되어 **갱신 유실(lost update)**이 발생한다. 분산락을 걸었는데 동시성 제어가 안 되는, 잡기 어려운 버그의 대표 양상이다.
- 올바른 순서는 `락 획득 → 트랜잭션 시작 → 커밋 → 락 해제`이며, 이를 위한 세 가지 방법:
  1. **빈 경계 분리** (0-3): 락을 잡는 메서드와 `@Transactional` 메서드를 서로 다른 빈으로 둔다. 가장 단순하고 가장 많이 쓴다. (같은 빈 안에서 `this.메서드()` 호출하면 프록시를 우회해 `@Transactional`이 안 붙는 것도 같은 계열의 함정.)
  2. **AOP @Order 분리** (0-4): 커스텀 `@DistributedLock` 어노테이션 + AOP를 트랜잭션 AOP보다 바깥에 둔다. `proceed()`가 끝난 시점 = 커밋 완료 시점이므로 finally의 unlock이 안전하다.
  3. **커밋 후 콜백**: `TransactionSynchronization.afterCommit`에서 unlock (위 `LockAfterCommitHelper`). 단 롤백 경로에서는 `afterCommit`이 호출되지 않으므로 **반드시 `afterCompletion`에서도 롤백 상태의 unlock을 처리**해야 락이 영구 점유되는 사고를 막는다.
- 진단 방법: 로그에 `TX begin → unlock → commit` 순서가 찍히는지 확인하거나, unlock 시점에 `TransactionSynchronizationManager.isActualTransactionActive()`가 true인지 확인하면 바로 눈에 들어온다.

### 2-2. 전략 선택의 실무 기준

- **재고 감소/포인트 차감 등 정합성이 절대적인 단일 DB 트랜잭션**: 비관적 락 또는 **조건부 원자적 UPDATE**(`UPDATE stock SET quantity = quantity - :n WHERE quantity >= :n` — 영향 row 수로 성공 판정). 단순하고 잡을 게 없어 오히려 가장 강하다.
- **읽기 비율이 압도적이고 충돌이 드문 데이터**(회원 프로필, 설정값): 낙관적 락. 충돌 비용이 0에 가깝고 재시도도 거의 안 뜬다.
- **여러 서버 인스턴스가 스케줄러/배치를 돌려 중복 실행을 막고 싶다**: Redisson 분산락 + Watchdog. (아그작에서 ShedLock을 쓰는 이유와 정확히 같은 문제 — ShedLock은 "스케줄러 중복 방지" 특화, Redisson RLock은 "임의 코드 블록 상호배제" 범용.)
- **좌석 예매처럼 대기열이 필요한 도메인**: 락만으로는 부족하고 Redis 대기열(ZSET) + 스케줄러 단계 처리가 별도로 필요하다.

### 2-3. 기타 엣지 케이스

- **분산락 단독으로 정합성을 보장하지 않는다**: Redis 장애·페일오버로 락이 유실될 수 있으므로, 최종 방어선은 DB 제약(유니크 키, 조건부 UPDATE)이어야 한다. "분산락 걸었으니 안전"은 금물.
- **키 설계**: 락 키에 반드시 **비즈니스 식별자**(`lock:stock:{productId}`)를 넣는다. 전역 키 하나(`lock:stock`)로 잠그면 서로 무관한 상품의 감소 요청까지 직렬화되어 처리량이 붕괴한다.
- **waitTime=0 (즉시 포기) vs 대기**: 재고 감소처럼 사용자 응답이 걸리는 경로는 waitTime을 짧게(수백 ms~수 초) 잡고 실패 시 재시도 유도가 맞고, 스케줄러는 waitTime=0으로 잡고 다음 주기에 재시도하는 게 자연스럽다.
- **벌크 연산은 @Version을 갱신하지 않는다**: JPQL `UPDATE ... SET quantity = quantity - 1`은 `@Version` 컬럼을 자동 증가시키지 않는다(`OPTIMISTIC_FORCE_INCREMENT` 사용 또는 벌크 쿼리에서 직접 증가). 벌크 연산과 낙관적 락을 섞어 쓰면 version 정합성이 깨진다.
- **테스트 환경**: 분산락 로직은 Testcontainers로 실제 Redis를 띄워 통합 테스트해야 한다. Mock으로는 Watchdog·Pub/Sub 타이밍 문제가 전혀 검증되지 않는다.

### 2-4. 아그작(AGZAK) 관점 메모

- 아그작은 Redis ZSET 랭킹 + ShedLock 조합을 이미 쓰고 있다. ZSET 갱신 자체는 `INCR`/`ZADD`가 원자적이므로 분산락이 필요 없지만, **랭킹 산정 배치 집계**가 여러 인스턴스에서 동시에 돌면 중복 집계가 되므로 이 지점에만 Redisson 또는 ShedLock이 맞다.
- 다중 인스턴스 환경에서 Refresh Token 재발급/로그아웃 블랙리스트 처리에도 Redisson의 `isHeldByCurrentThread()` 가드 + Watchdog 패턴을 그대로 적용할 수 있다.

---

## 3. 기술 면접 단골 꼬리 질문 & 모범 답변 팁

**Q. 낙관적 락과 비관적 락을 언제 선택하나요?**
- **A.** 충돌 빈도가 기준입니다. 낙관적 락은 충돌이 드문 데이터에 비용이 거의 없지만, 충돌이 잦으면 재시도 폭풍으로 DB 부하와 응답 지연이 커집니다. 비관적 락은 재시도 없이 확실히 직렬화되지만 대기 동안 DB 커넥션을 점유하므로 풀 고갈과 데드락 위험이 있습니다. 실무 기준은 "동시 수정이 예상되는 인기 자원(재고·좌석)은 비관적 락 또는 조건부 원자적 UPDATE, 대부분 읽기 전용인 데이터는 낙관적 락"입니다.

**Q. `ObjectOptimisticLockingFailureException`이 발생했을 때 재시도는 어떻게 구현하나요? 그냥 catch해서 다시 호출하면 왜 안 되나요?**
- **A.** 같은 트랜잭션 안에서 재시도하면 안 됩니다. 낙관적 락 실패 시 해당 트랜잭션은 이미 rollback-only로 마킹되므로 재시도 로직을 실행해도 커밋 시점에 `UnexpectedRollbackException`이 납니다. 따라서 재시도 대상 메서드를 `Propagation.REQUIRES_NEW`로 분리해 새 트랜잭션에서 실행하거나, Spring Retry의 `@Retryable`에 백오프를 걸어 사용합니다. 무한 재시도는 충돌이 지속될 때 스레드를 계속 점유하므로 상한과 지수 백오프+지터가 필수입니다.

**Q. 비관적 락에서 데드락이 발생하는 시나리오와 예방책을 설명해 보세요.**
- **A.** 트랜잭션 A가 상품 1을 FOR UPDATE로 잠근 뒤 상품 2를 잠그려 하고, 트랜잭션 B는 반대 순서(2 → 1)로 접근하면 상호 대기가 생깁니다. InnoDB가 데드락을 감지해 한쪽을 롤백시키지만 애플리케이션은 예외를 받습니다. 예방책은 (1) 여러 Row를 잠글 때 **항상 같은 정렬 기준**(예: id 오름차순)으로 획득, (2) 트랜잭션을 짧게 유지해 락 보유 시간 최소화, (3) `innodb_lock_wait_timeout`과 JPA `lock.timeout`을 짧게 설정해 무기한 대기를 막는 것입니다.

**Q. Redisson의 Watchdog이 무엇이고, `leaseTime`을 명시하면 어떻게 되나요?**
- **A.** Watchdog은 락을 잡은 클라이언트가 살아 있는 동안 락이 만료되지 않게 TTL을 주기적으로 갱신하는 Redisson의 백그라운드 스레드입니다. `tryLock`에서 `leaseTime`을 생략하면 기본 30초 TTL이 10초 간격으로 갱신되어 작업이 길어져도 락이 유지됩니다. 반면 `leaseTime`을 명시하면 Watchdog이 작동하지 않아 지정 시간 뒤 락이 무조건 만료됩니다. 작업 소요 시간을 예측할 수 없으면 leaseTime 생략(Watchdog 사용), 예측 가능하고 짧으면 명시하는 것이 기준입니다.

**Q. Redis 분산락을 걸었는데도 재고가 음수가 되는 버그가 발생했습니다. 원인과 해결책을 말해 보세요.**
- **A.** 가장 유력한 원인은 락 해제가 트랜잭션 커밋보다 빠른 경우입니다. `@Transactional` 메서드 내부에서 finally로 unlock하면 커밋 전에 다른 스레드가 락을 얻고, 아직 커밋되지 않은 재고 값을 기준으로 감소를 계산해 갱신이 유실됩니다. 해결은 락 획득 → 트랜잭션 시작 → 커밋 → 락 해제 순서를 보장하는 것으로, (1) 락 담당 메서드와 트랜잭션 메서드를 빈 경계로 분리하거나, (2) 커스텀 어노테이션 + AOP(`@Order`를 트랜잭션 AOP보다 바깥으로)에서 unlock하거나, (3) `TransactionSynchronization.afterCommit`(+ 롤백 대비 `afterCompletion`) 콜백에서 unlock하면 됩니다. 근본 방어선으로 DB의 조건부 UPDATE(`WHERE quantity >= :n`)나 유니크 제약을 병행해 Redis 장애 시에도 정합성이 깨지지 않게 합니다.

---

## 4. 오늘의 체크리스트 (내일 이어서)

- [ ] 조건부 원자적 UPDATE(`UPDATE ... WHERE quantity >= :n`)를 낙관적/비관적/분산락과 벤치마크 비교하는 데모 작성
- [ ] `unlock → commit` 시간선으로 재고 유실이 재현되는 테스트 코드 작성 → 빈 경계 분리로 해결되는 것까지 확인
- [ ] Redisson Watchdog 동작 확인: leaseTime 생략/명시 각각에서 TTL 갱신 로그 비교
- [ ] AOP `@Order`를 트랜잭션보다 안쪽으로 잘못 잡았을 때 락-TX 순서가 어떻게 꼬이는지 재현
- [ ] RedLock 논쟁(Martin Kleppmann vs antirez) 원문 읽고 "분산락의 한계" 요약 정리
