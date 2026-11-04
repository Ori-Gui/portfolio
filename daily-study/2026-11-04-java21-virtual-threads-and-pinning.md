# [2026-11-04] Java 21 가상 스레드(Virtual Threads)와 Carrier Thread 피닝(Pinning)

> **한 줄 핵심 요약**: 가상 스레드는 OS 스레드(1:1 매핑) 비용을 획기적으로 줄여 높은 I/O 동시성을 제공하지만, `synchronized` 블록이나 JNI 호출 시 Carrier Thread가 고정(Pinning)되어 스레드 풀 고갈이 발생할 수 있으므로 `ReentrantLock` 대체 및 프로파일링이 필수적입니다.

---

## 0. 코드 예제 (전체 실행 가능 버전)

```java
// 파일 경로: src/main/java/study/vt/VirtualThreadPinningStudy.java
// 실행: java --source 21 -Djdk.tracePinnedThreads=short VirtualThreadPinningStudy.java
// (Gradle 사용 시: toolchain { languageVersion = JavaLanguageVersion.of(21) } 필요, 의존성 없음 — 순수 JDK)

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;
import java.util.concurrent.locks.ReentrantLock;

public class VirtualThreadPinningStudy {

    // 1) 안티패턴: synchronized 블록 안에서 블로킹 I/O 수행
    private static final Object BAD_LOCK = new Object();

    static void badBlockingIO() {
        synchronized (BAD_LOCK) {                 // 이 지점에서 Carrier Thread가 고정(Pinning)됨
            callRemoteApi();                      // 네트워크 I/O 동안 플랫폼 스레드가 통째로 점유됨
        }
    }

    // 2) 권장 패턴: ReentrantLock — I/O 대기 중 언마운트가 정상 동작
    private static final ReentrantLock LOCK = new ReentrantLock();

    static void goodBlockingIO() {
        LOCK.lock();
        try {
            callRemoteApi();                      // 대기 중 가상 스레드만 언마운트되고 Carrier는 해제됨
        } finally {
            LOCK.unlock();
        }
    }

    // 3) 풀링 대신 Semaphore로 자원(DB 커넥션 등) 동시성 제한
    private static final Semaphore DB_PERMITS = new Semaphore(10); // 커넥션 풀 크기만큼

    static void withDb(java.lang.Runnable dbWork) throws InterruptedException {
        DB_PERMITS.acquire();
        try {
            dbWork.run();
        } finally {
            DB_PERMITS.release();
        }
    }

    static void callRemoteApi() {
        // 실제로는 HttpClient 등의 블로킹 호출이라고 가정
        try { Thread.sleep(100); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }

    public static void main(String[] args) throws Exception {
        // 가상 스레드는 풀링하지 않고 태스크마다 생성
        try (ExecutorService vt = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 10_000; i++) {
                vt.submit(() -> { goodBlockingIO(); return null; });
            }
        } // close() 시 모든 태스크 종료 대기

        withDb(() -> System.out.println("db work on " + Thread.currentThread()));
    }
}
```

---

## 1. 개념 및 동작 원리

- **플랫폼 스레드(OS 스레드) vs 가상 스레드**
  - 플랫폼 스레드: JVM 스레드가 OS 스레드와 1:1 매핑. 생성 비용 수 MB 스택 메모리 + 커널 호출, 컨텍스트 스위치가 커널 영역에서 발생. 수천 개 이상 두기 어렵다.
  - 가상 스레드: JVM이 관리하는 경량 스레드. OS 스레드 위에서 **M:N 스케줄링**되며, 내부적으로 `ForkJoinPool`(FIFO 모드, `java.util.concurrent.ForkJoinPool`의 `virtual thread scheduler`)이 **캐리어(Carrier) 스레드** 역할을 한다. 기본 캐리어 수 = CPU 코어 수.
- **Continuation 기반 마운트/언마운트**
  - 가상 스레드의 실행 상태는 `Continuation` 객체에 담긴다. 블로킹 연산(I/O, `sleep`, 락 대기)에 진입하면 현재 스택을 힙으로 복사해 **언마운트(unmount)** 하고 캐리어를 반납 → 다른 가상 스레드를 그 캐리어에 **마운트(mount)**.
  - 즉, "블로킹"이 OS 스레드를 점유하는 블로킹이 아니라 JVM 레벨의 논블로킹 대기로 치환된다. 이것이 I/O Bound 동시성에서 처리량(Throughput)이 나오는 이유.
- **목적: I/O Bound 최적화**
  - CPU 연산을 빠르게 하는 기술이 아니다. 단순 스레드 전환 오버헤드 감소도 아니고, **I/O 대기 동안 OS 스레드 1개를 아무 일도 못 하게 붙잡고 있는 낭비를 없애는 것**이 핵심이다. CPU Bound 작업에는 플랫폼 스레드가 여전히 적합.

---

## 2. 실무 적용 및 주의사항 (Edge Cases & 동시성)

### 2-1. Carrier Thread Pinning 문제

- **현상**: `synchronized` 블록/메서드 내부에서 블로킹 I/O를 수행하면, JVM이 스택을 언마운트하지 못하고 **캐리어(플랫폼) 스레드를 그 자리에 고정**한다. 고정된 캐리어는 다른 가상 스레드를 받아들일 수 없어, 동시 피닝 수가 캐리어 수(≈CPU 코어 수)를 넘는 순간 **가상 스레드 전체가 정체(starvation)** 되고, 결국 스레드 풀 고갈처럼 보이는 장애가 발생한다.
- **JDK 21 vs 24 차이 주의**: JDK 21(JEP 444)은 여전히 `synchronized`에서 피닝이 발생하며, JDK 24(JEP 491)부터 피닝이 해소되었다. 실무에서 JDK 21을 쓰는 동안에는 여전히 대응이 필요하다.
- **JNI / `native` 호출**: 네이티브 프레임 안에서 블로킹하면 언마운트가 불가능하므로 피닝된다. 이 쪽은 언어 레벨 대체재가 없어 라이브러리 선택 시 확인 대상이다.
- **대응책**: `synchronized` 대신 `java.util.concurrent.locks.ReentrantLock` 사용. 잠금 경계가 라이브러리 내부에 있으면 해당 라이브러리 이슈/릴리스노트를 확인한다.
- **감지**: `-Djdk.tracePinnedThreads=short`(스택 프레임 요약) 또는 `full` JVM 옵션으로 피닝 발생 지점을 스택 트레이스에 출력한다. 로그가 무한정 늘어날 수 있으므로 운영에서는 샘플링/로그 레벨 관리가 필요하다. (JDK 25에서 이 시스템 프로퍼티는 제거되고 JFR `jdk.VirtualThreadPinned` 이벤트로 대체되었으므로, 버전 업그레이드 시점에 함께 점검한다.)

### 2-2. 스레드 풀(Thread Pool) 안티패턴

- 가상 스레드는 생성 비용이 극히 낮으므로 **풀링하지 않고 필요할 때마다 생성**한다. 표준 방식은 `Executors.newVirtualThreadPerTaskExecutor()` (태스크당 1개 생성).
- 기존 코드에서 `newFixedThreadPool(200)`으로 "스레드 부족을 막는" 습관을 그대로 옮기면 가상 스레드의 의미가 없어진다. **동시성 제한이 필요한 자원**(DB 커넥션, 외부 API Rate Limit 등)은 풀 크기 대신 `Semaphore`로 제어한다.
- **DB 커넥션 풀과의 관계**: 가상 스레드 수천 개가 열려도 JDBC 커넥션 풀(HikariCP 등)은 여전히 유한하다. 커넥션 대기가 병목이면 가상 스레드 이점이 사라지므로, 커넥션 풀 크기·대기 타임아웃 튜닝이 선행되어야 한다.
- **ThreadLocal 주의**: 수많은 가상 스레드가 각자 큰 객체를 `ThreadLocal`에 담으면 메모리가 수천 배로 복제되는 효과가 생긴다. JDK 21의 `ScopedValue`(Preview)가 대안이다.

### 2-3. 아그작(AGZAK) 관점 메모

- Redis ZSET 랭킹, Redis 분산락(ShedLock 포함)처럼 I/O가 잦은 경로에 가상 스레드를 도입하면 처리량 이점이 크다. 단, Lettuce/Jedis 클라이언트의 커넥션 관리가 피닝 없이 동작하는지 버전을 확인해야 하고, 락 획득 대기(`setIfAbsent` 폴링)를 `synchronized`로 감싸면 그대로 피닝 지점이 되므로 코드 리뷰 때 락 경계를 별도 체크 리스트로 둘 것.

---

## 3. 기술 면접 단골 꼬리 질문 & 모범 답변 팁

**Q. 가상 스레드를 도입하면 연산 속도(Latency) 자체가 빨라지나요?**
- **A.** 아닙니다. 단일 작업의 실행 시간(Latency)은 단축되지 않으며, I/O 대기 시간 동안 OS 스레드를 점유하지 않아 시스템 전체의 동시 처리량(Throughput)을 극대화하는 기술입니다. CPU 집약적인 연산에는 기존 플랫폼 스레드가 더 적합합니다.

**Q. synchronized를 ReentrantLock으로 무조건 다 바꿔야 하나요?**
- **A.** synchronized 블록 내에서 긴 I/O 작업(네트워크 호출, DB 쿼리 등)을 수행하는 경우가 아니라, 메모리 상의 짧은 변수 조작 수준이라면 피닝 영향이 미미합니다. 다만 라이브러리 레벨의 I/O 경계에서는 ReentrantLock으로 전환하는 것이 안전합니다. (JDK 24부터는 JEP 491로 피닝이 해소되어 JDK 21에서만 임시 대응이라고 정리하면 점수가 붙습니다.)

**Q. 가상 스레드 수천 개를 띄워도 DB 커넥션은 어떻게 되나요?**
- **A.** 커넥션 풀은 여전히 유한하므로 병목이 그대로 이동합니다. `Semaphore`로 애플리케이션 레벨 동시성을 커넥션 풀 크기에 맞추거나, 풀 크기·대기 타임아웃을 함께 튜닝해야 가상 스레드 이점이 실측으로 나옵니다.

**Q. 피닝이 실제로 일어나는지 어떻게 확인하나요?**
- **A.** `-Djdk.tracePinnedThreads=short|full`로 피닝 발생 시점의 스택을 출력해 확인하고, 운영 환경에서는 JFR의 `jdk.VirtualThreadPinned` 이벤트로 지속 시간 기준 임계치를 걸어 모니터링합니다.

---

## 4. 오늘의 체크리스트 (내일 이어서)

- [ ] AGZAK 코드베이스에서 `synchronized` + I/O 조합이 있는지 grep으로 점검
- [ ] Lettuce 클라이언트 버전별 가상 스레드 호환 이슈 조사
- [ ] `-Djdk.tracePinnedThreads=short` 로컬 데모 실행 후 로그 캡처
