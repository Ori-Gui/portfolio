# [2026-11-18] JVM 메모리 구조와 가비지 컬렉션 — G1·ZGC의 pause 특성, 컨테이너 메모리 산정, STW가 트랜잭션·분산락에 미치는 영향

> **한 줄 핵심 요약**: JVM이 쓰는 메모리는 힙 하나가 아니라 **힙 + Metaspace + CodeCache + 스레드 스택 + Direct Buffer의 합**이고, 컨테이너 limit은 이 합보다 커야 합니다. GC 선택은 "가장 좋은 GC"를 고르는 일이 아니라 **지연(pause)·처리량(throughput)·메모리 여유(footprint) 중 무엇을 우선할지 정하는 일**입니다. 그리고 STW pause는 애플리케이션 스레드만 멈출 뿐 **DB 트랜잭션을 롤백시키지 않으므로**, pause가 길어지면 증상은 GC 로그가 아니라 "클라이언트 타임아웃 → 재시도 → 중복 처리"로 나타납니다.

---

## 1. 개념 및 동작 원리

### 1.1 런타임 데이터 영역: 힙 밖까지 합쳐야 컨테이너가 산다

| 영역 | 무엇이 사는가 | 크기 결정 | 대표 사고 |
| --- | --- | --- | --- |
| 힙 (Young: Eden/Survivor, Old) | 객체 인스턴스, 배열 | `-Xms`/`-Xmx`, 기본 최대 = 물리(또는 cgroup) 메모리의 1/4 | `OutOfMemoryError: Java heap space` |
| Metaspace | 클래스 메타데이터, 상수 풀, 바이트코드 | native, 기본 상한 없음 | `Metaspace` OOM |
| Code Cache | JIT 컴파일된 네이티브 코드 | `-XX:ReservedCodeCacheSize` (기본 예약 240MB 계열) | `CodeCache is full` → 인터프리터 폴백, p99 상승 |
| 스레드 스택 | 프레임, 지역변수 | `-Xss` (Linux x64 기본 1MB) | `unable to create new native thread` |
| Direct Buffer | `ByteBuffer.allocateDirect`, Netty | `-XX:MaxDirectMemorySize` (기본 = 최대 힙) | `Direct buffer memory` OOM |
| GC 자료구조 | 카드 테이블, RSet, 마킹 비트맵 | 힙에 비례 | 힙의 5~20%를 GC가 점유 |
| JNI / native / mmap | 라이브러리, 파일 매핑 | 라이브러리 | RSS만 늘고 힙은 멀쩡 |

- `-Xmx2g`인 애플리케이션의 RSS는 보통 2.6~3.2GB가 됩니다. 컨테이너 limit을 2Gi로 잡으면 JVM은 힙을 다 쓰기 전에 **커널 OOM killer에게 SIGKILL** 당하고, 남는 단서는 GC 로그도 `OutOfMemoryError`도 아닌 `Exit Code: 137`(`Reason: OOMKilled`)뿐입니다.
- JDK 10+ 는 `UseContainerSupport`가 기본이라 cgroup limit을 인식합니다. 다만 자동으로 해주는 것은 **힙 기본값 계산**까지이고, non-heap 영역을 빼주지는 않습니다.

**Compressed Oops — 32GB 경계.** 참조를 4바이트로 압축(heap base + scaled offset)해 힙 사용량과 캐시 효율을 동시에 줄입니다. 32GB 미만 힙에서 기본 활성. 32GB를 넘기면 압축이 풀려 **같은 객체가 1.5배** 메모리를 차지합니다. "30GB에서 잘 돌던 앱을 40GB로 늘렸더니 더 빨리 OOM"이 전형적인 증상입니다. 실무 기준은 **31GB 이하로 맞추거나, 필요하면 48GB 이상으로 확실히 넘기는 것**입니다(어중간한 33~40GB가 가장 나쁩니다).

### 1.2 객체 할당: TLAB와 bump-the-pointer

- 스레드마다 Eden 안에 TLAB(Thread-Local Allocation Buffer)를 받고, 그 안에서는 포인터만 밀어 올려(bump-the-pointer) 할당합니다. 락도 CAS도 없는 경로라 새 객체 생성 자체는 매우 쌉니다.
- TLAB이 차면 새 TLAB을 받고, 그것도 안 되면 Eden에서 직접 CAS 할당, Eden이 차면 Young GC입니다.
- 탈출 분석(escape analysis)으로 다른 곳에서 참조되지 않는 객체는 스칼라로 치환되어 스택/레지스터에서 소멸할 수 있습니다(`-XX:+DoEscapeAnalysis` 기본).
- 실무 결론: **짧게 사는 객체는 거의 공짜, 문제는 오래 사는 객체와 큰 객체**입니다. 루프 안에서 `String` 연결이나 `String.format`으로 중간 문자열을 만들거나, 조회 결과 수십만 건을 `List`로 들고 있는 코드가 전형적인 원인입니다.

### 1.3 GC의 기본 동작: 도달 가능성 판정 → 회수

GC는 "안 쓰는 객체"를 직접 알 수 없으므로 **GC Root에서 도달(reachable) 가능한지**로 판정합니다.

- GC Root: 활성 스레드의 스택 프레임 지역변수, `static` 필드, JNI 참조, 동기화 모니터, JVM 내부 참조.
- 회수 방식: mark → sweep(비움) / compact(압축) / copy(생존자 복사).
- 참조 강도(실무에서 구분이 필요합니다):
  - **Strong**: 일반 참조. 도달 가능하면 절대 회수 안 됨 → 누수의 주범.
  - **Soft**: 메모리가 부족할 때만 회수. 메모리 기반 캐시에 어울립니다.
  - **Weak**: 다음 GC에서 항상 회수. `WeakHashMap`, `ThreadLocal`의 키가 이 방식.
  - **Phantom + Cleaner**: 회수 직후 후처리. JDK 18(JEP 421)부터 `finalize()`는 제거 예정(deprecated for removal)이라 `Cleaner`로 대체합니다.

### 1.4 세대별 가설과 Young GC

약한 세대 가설: **대부분의 객체는 생성 직후 죽는다.** 그래서 Young과 Old를 나눠 Young만 자주, 싸게 수집합니다.

- Eden이 차면 minor GC: 살아있는 객체만 Survivor(S0→S1)로 복사하고 age를 1 올립니다. 복사 비용은 **살아남은 객체 수에 비례**하므로, 대부분이 죽는 Young에서는 매우 빠릅니다.
- age가 임계값(`-XX:MaxTenuringThreshold`, 기본 15)을 넘거나 Survivor가 넘치면 Old로 승격(promotion)됩니다. 임계값은 Survivor 점유율을 보고 동적으로 조정됩니다.
- Old → Young 참조를 찾기 위해 **카드 테이블 + write barrier**를 씁니다(전체 Old를 스캔하면 minor GC가 의미를 잃습니다).
- 트레이드오프: Eden을 키우면 minor GC 빈도는 줄지만 **1회 비용과 승격량**이 늘고, Old로 잘못 승격된 쓰레기가 늘어 Full GC 위험이 커집니다. "minor GC가 잦다"는 이유만으로 Young을 키우는 건 진단이 아니라 도박입니다.

### 1.5 G1 — JDK 9+의 기본 수집기

- 힙을 **1~32MB(2의 거듭제곱) region**으로 나눕니다. region 크기는 전체 region 수가 2048개를 넘지 않도록 자동 선택되고, `-XX:G1HeapRegionSize`로 고정할 수 있습니다.
- **Humongous 객체**: region 크기의 50%를 넘는 객체는 연속한 humongous region에 **바로 Old로** 할당됩니다. 큰 `byte[]`, 거대 JSON/문자열, 대용량 엑셀 셀이 원인이며, 조각화와 불필요한 concurrent cycle을 유발합니다. `-Xlog:gc+region`이나 `GC.heap_info`에서 `Humongous regions` 수를 봅니다.
- 사이클: **young-only**(young GC 반복) → **concurrent start**(initial mark, Old 점유가 임계치 도달) → **space-reclamation**(mixed GC로 Old region 회수) → 실패하면 **Full GC**(최후 수단, JEP 307 이후 병렬).
- `-XX:MaxGCPauseMillis=200`이 기본입니다. 이는 **목표이지 보장이 아니며**, G1은 과거 pause 통계로 수집할 region을 골라(가비지가 많은 region 우선 = "Garbage-First") 목표에 근접시킵니다.
- 마킹은 SATB(snapshot-at-the-beginning)로, 애플리케이션과 동시에 진행하고 RSet 갱신 비용을 애플리케이션 스레드가 나눠 부담합니다. 그래서 **처리량이 Parallel보다 조금 낮습니다**.
- old 점유 임계치(`-XX:InitiatingHeapOccupancyPercent`, 기본 45%)를 넘으면 concurrent cycle을 시작하며, 최근 JDK에서는 적응적으로 조정됩니다.

### 1.6 ZGC — pause를 힙 크기에서 분리

- **colored pointer + load barrier**(세대형은 store barrier 추가)로, 애플리케이션과 **동시에** mark / relocate / remap을 수행합니다. pause는 마킹 시작·종료 같은 짧은 지점뿐이며 **1ms 미만, 힙 크기와 무관**합니다(수백 MB ~ 수 TB).
- region(ZPage: small/medium/large) 기반이며, 큰 객체도 페이지 단위로 처리해 G1의 humongous 문제가 없습니다.
- JDK 21 기준 플래그: `-XX:+UseZGC -XX:+ZGenerational` (JEP 439). JDK 21에서 `-XX:+UseZGC`만 주면 **비세대형**이 선택되므로 실무에서는 `+ZGenerational`을 명시해야 합니다. 이후 세대형이 기본이 되고(JEP 474), 비세대형은 제거되었습니다(JEP 490).
- 대가:
  1. **처리량 손실** — 모든 참조 읽기/쓰기에 배리어가 붙습니다(대략 5~15%).
  2. **CPU 여유 필요** — 동시 작업이 코어를 먹습니다. 코어 2개짜리 컨테이너에서는 이득이 사라집니다.
  3. **힙 여유 필요** — 회수 속도가 할당 속도를 못 따라가면 **allocation stall**(사실상 STW)이 납니다. `-XX:SoftMaxHeapSize`로 상한을 두고 여유를 남깁니다.
  4. **메모리 반환 지연** — 커밋한 메모리를 OS에 즉시 돌려주지 않아 RSS가 천천히 줄어듭니다.
- Shenandoah는 또 다른 저지연 수집기로, 힙 크기와 무관한 pause를 노리며 일부 배포판에서 기본 후보입니다(`-XX:+UseShenandoahGC`).

### 1.7 수집기 비교와 선택 규칙

| 수집기 | 우선하는 것 | 전형 pause | 적합한 힙 | 쓰는 상황 |
| --- | --- | --- | --- | --- |
| Serial | footprint | 수십~수백 ms | < 100MB | CLI, 소형 배치, 1코어 |
| Parallel | throughput | 100ms ~ 수 초 | 수 GB | 야간 배치·ETL, 지연 무관 |
| G1 | 균형 | 20~200ms | 4~32GB | 일반 웹 API의 기본값 |
| ZGC | latency | < 1ms | 수백 MB ~ 수 TB | p99가 중요한 API, 큰 힙 |
| Shenandoah | latency | < 10ms | 중대형 | ZGC를 못 쓰는 배포판 |

선택 규칙:

1. **기본은 G1로 두고 측정합니다.** 대부분의 스프링 API는 튜닝 없이 G1으로 충분합니다.
2. p99 지연의 원인이 GC pause로 **확인된 경우에만** ZGC를 검토합니다(요청 지연 그래프와 GC 로그의 pause 시각이 겹치는지).
3. 처리량이 목표인 배치는 Parallel. 지연을 낮추려고 배치에 ZGC를 쓰면 전체 처리 시간이 늘어납니다.
4. 힙이 작고 코어가 적으면 ZGC의 배리어 비용이 이득보다 큽니다.

---

## 2. 실무 적용 및 주의사항 (Edge Cases)

### 2.1 컨테이너 메모리 산정 — 2Gi limit 실제 계산

```yaml
# k8s/deployment.yaml (env 부분)
env:
  - name: JAVA_TOOL_OPTIONS
    value: >-
      -XX:MaxRAMPercentage=55
      -XX:InitialRAMPercentage=55
      -XX:MaxMetaspaceSize=256m
      -XX:MaxDirectMemorySize=256m
      -XX:ReservedCodeCacheSize=128m
      -XX:+HeapDumpOnOutOfMemoryError
      -XX:HeapDumpPath=/var/log/app/heapdump.hprof
      -XX:+ExitOnOutOfMemoryError
      -Xlog:gc*,gc+heap=info:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=20M
resources:
  limits:
    memory: "2Gi"
```

계산: limit 2Gi에서 힙 55% = 1.1GB, Metaspace 256MB, Direct 256MB, CodeCache 128MB, Tomcat 스레드 200개 × 1MB = 200MB → 합계 약 1.94GB. 여기에 GC 자료구조와 JNI·mmap이 더해지면 **여유가 거의 없습니다**. 60%로 올리면 2.03GB를 넘겨 OOMKilled가 납니다. 그래서 실무 기본값은 **non-heap 합계를 먼저 정하고 남은 만큼을 힙 비율로** 잡는 순서입니다.

- `-Xmx`와 `MaxRAMPercentage`를 같이 주면 **명시한 `-Xmx`가 이깁니다.** 혼용하면 의도가 조용히 무시되므로 한쪽만 씁니다.
- `-Xms`와 `-Xmx`는 같게 두는 편이 좋습니다. 힙 리사이즈와 그에 따른 Full GC를 막고, RSS 예측이 쉬워집니다.
- 스레드 수가 많은 앱(Tomcat `max-threads` 200 + Hikari 20 + Kafka 컨슈머 등)은 스택만 200~300MB입니다. **가상 스레드**를 쓰면 스택 청크가 힙에 올라가 네이티브 스택 부담이 크게 줄지만, JDK 21에서는 `synchronized` 블록과 native 프레임에서 **핀(pinning)** 되어 캐리어 스레드를 붙잡습니다(JDK 24 JEP 491에서 `synchronized` 핀이 해소). 핀이 많으면 캐리어 풀이 고갈되므로 JFR `jdk.VirtualThreadPinned` 이벤트로 확인합니다. (11-04 노트 참조)

### 2.2 OOM 유형별 진단 표

| 메시지 | 의미 | 첫 확인 |
| --- | --- | --- |
| `Java heap space` | 살아있는 객체가 힙을 다 씀 (누수 또는 순수 부족) | heap dump의 dominator tree |
| `GC overhead limit exceeded` | 힙의 거의 전부를 회수하는 데 시간의 대부분을 씀 | 사실상 누수. 힙 증설은 미봉책 |
| `Metaspace` | 클래스 로더 누수, 무한 프록시/람다 생성 | `jcmd <pid> VM.metaspace` |
| `Compressed class space` | 위와 같은 원인 | 동일 |
| `Direct buffer memory` | Netty/NIO 버퍼 미반납 | `-XX:NativeMemoryTracking=summary` + `jcmd <pid> VM.native_memory` |
| `unable to create new native thread` | 스레드 스택/프로세스 한도 초과 | `ulimit -u`, cgroup `pids.max`, 스레드풀 누수 |
| `Requested array size exceeds VM limit` | int 범위 배열 요청 | 쿼리 결과 전체 로딩 |
| 커널 `Out of memory: Killed process` | 컨테이너/호스트 OOM killer | `Exit Code: 137`, `dmesg -T \| tail` |

### 2.3 진단 절차 (재현 가능한 순서)

1. **증상 고정**: p50/p99, GC pause 분포, RSS, 힙 사용 곡선을 같은 시간축에 놓습니다.
2. `jcmd <pid> GC.heap_info` — 세대별 사용량과 region 상태.
3. `jcmd <pid> GC.class_histogram` — 인스턴스 수 상위 클래스. **한 번의 절대량보다 2회 간격의 증가율**이 중요합니다(이 명령은 full GC를 유발하므로 운영 중 주의).
4. `jcmd <pid> GC.heap_dump /var/log/app/heap.hprof` → Eclipse MAT에서 dominator tree, `Path to GC Roots`(weak/soft 제외)로 **누가 붙잡고 있는지** 확인.
5. 배포 전후 비교는 JFR(`jdk.GCPhasePause`, `jdk.ObjectAllocationSample`) 또는 Actuator metric(`jvm.memory.used`, `jvm.gc.pause`의 max)으로 남깁니다.

```bash
# 운영에서 바로 쓰는 진단 명령 모음
jcmd <pid> VM.flags                       # 실제 적용된 플래그(기본값 포함)
jcmd <pid> GC.heap_info                   # 세대/region 사용량
jcmd <pid> GC.class_histogram | head -30  # 점유 상위 클래스
jcmd <pid> JFR.start name=profile settings=profile duration=180s filename=/var/log/app/short.jfr
jcmd <pid> Thread.print                   # pause가 아니라 블로킹인지 구분
```

### 2.4 흔한 누수 패턴 (Java/Spring)

- **무제한 `static Map` 캐시** — 가장 흔합니다. `Caffeine`으로 교체하고 `maximumSize`·`expireAfterWrite`를 반드시 겁니다.
- **`ThreadLocal` 미해제** — 스레드풀에서 스레드가 재사용되므로 값이 계속 살아남습니다. `try/finally`로 `remove()`.
- **리스너/구독 미해제** — `ApplicationListener`, Redis 구독, SSE `SseEmitter` 타임아웃 미설정.
- **커넥션·스트림 미반납** — `try-with-resources` 누락.
- **대량 조회를 메모리로** — `findAll()` 후 애플리케이션 집계, 엑셀/CSV 전량 로딩. 페이징(keyset)이나 스트리밍으로 바꿉니다.
- **무제한 내부 큐** — `LinkedBlockingQueue` 용량 미지정, 재시도 큐 적재.
- **클래스 로더 누수** — 재배포 시 이전 웹앱 로더가 살아남아 Metaspace가 계단식으로 증가(개발/운영 모두에서 발생).

```java
// src/main/java/com/example/support/InMemoryCacheConfig.java
package com.example.support;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import java.time.Duration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * 로컬 캐시는 힙에 산다. 상한과 만료가 없으면 그 자체가 메모리 누수다.
 * - maximumSize: 평균 객체 크기 x 개수로 힙 예산을 역산해 정한다.
 * - expireAfterWrite: 정합성 허용 시간. 인스턴스마다 값이 다르다는 점을 전제한다.
 */
@Configuration
public class InMemoryCacheConfig {

    @Bean
    public Cache<Long, ProductSummary> productSummaryCache() {
        return Caffeine.newBuilder()
                .maximumSize(10_000)
                .expireAfterWrite(Duration.ofMinutes(10))
                .recordStats()          // hitRate를 Actuator로 노출해 캐시가 실제로 일하는지 확인
                .build();
    }
}
```

```groovy
// build.gradle
dependencies {
    implementation 'com.github.ben-manes.caffeine:caffeine:3.1.8'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```

정합성이 필요한 데이터는 로컬 캐시가 아니라 Redis로 보냅니다. 로컬 캐시는 인스턴스 N대의 값이 서로 다를 수 있고, 그 차이가 "가끔 옛 가격이 보인다"는 형태로 나타납니다. (11-10 노트 참조)

### 2.5 STW가 트랜잭션·동시성에 미치는 영향

여기가 GC 공부가 실무 지식으로 바뀌는 지점입니다. 정확히 말하면:

- **STW는 애플리케이션 스레드만 멈춥니다.** DB 커넥션도, 이미 전송된 SQL도, DB 트랜잭션도 살아 있습니다. 그래서 **GC 때문에 트랜잭션이 롤백되지는 않습니다.** pause가 끝나면 커밋은 예정대로 됩니다.
- 문제는 그 사이 **응답이 사라지는 쪽**입니다. 이것이 중복 처리로 이어집니다.

| 구간 | pause가 초과하면 | 결과 |
| --- | --- | --- |
| 클라이언트/게이트웨이 read timeout | 504 응답, 사용자 재시도 | 서버는 커밋 성공 → **중복 주문·중복 결제** |
| HikariCP `connection-timeout` (기본 30s) | `SQLTransientConnectionException` | 커넥션을 쥔 채 멈추면 풀 고갈 → 정상 요청까지 실패 |
| `@Transactional(timeout)` / JDBC `queryTimeout` | `QueryTimeoutException` | 이건 **실제로 롤백**됩니다. 단 중단이 확정되지 않는 경우가 있어 최종 상태는 DB에서 확인 |
| Kafka `max.poll.interval.ms` (기본 5분) | 컨슈머 그룹 리밸런스 | 같은 레코드를 다른 컨슈머가 재처리 → **멱등 키 필수** |
| Kafka `session.timeout.ms` (Kafka 3.0+ 기본 45s) | 그룹 이탈 | 반복 리밸런스로 처리 지연 |
| ShedLock `lockAtMostFor` | 락 만료 후 다른 노드가 실행 | 배치 **중복 실행** |
| Redisson leaseTime | watchdog 갱신도 STW에 멈춤 | 임계구역 동시 진입 → 재고 초과 판매 |

대응은 네 가지입니다.

1. **pause를 줄인다** — humongous 객체 제거, 대량 조회 페이징, GC/힙 조정.
2. **시간 예산을 pause보다 크게 잡는다** — `connectionTimeout 30s`, `queryTimeout 3~5s`, 락 lease = 예상 실행시간 × 3 + 최대 pause × 2.
3. **멱등성** — 유니크 제약 + 요청 키. 재시도가 와도 한 번만 반영.
4. **fencing token** — 락이 만료된 뒤 이전 소유자의 쓰기를 DB가 거부.

```java
// src/main/java/com/example/order/service/OrderService.java
package com.example.order.service;

import com.example.order.dto.OrderResult;
import com.example.order.dto.PlaceOrderCommand;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.stereotype.Service;

/**
 * 멱등 주문 진입점. 트랜잭션 경계를 여기 두지 않는다.
 * 이유: 제약 위반(DataIntegrityViolationException)이 난 트랜잭션은 이미 rollback-only라
 * 같은 트랜잭션 안에서 잡아 복구할 수 없다. 프록시 경계 밖에서 잡아야 새 트랜잭션으로 다시 읽을 수 있다.
 */
@Service
public class OrderService {

    private final OrderTxService orderTxService;

    public OrderService(OrderTxService orderTxService) {
        this.orderTxService = orderTxService;
    }

    public OrderResult place(PlaceOrderCommand command) {
        try {
            return orderTxService.insert(command);              // REQUIRES_NEW, 짧은 트랜잭션
        } catch (DataIntegrityViolationException e) {            // (order_request_key 유니크 제약)
            return orderTxService.findByRequestKey(command.requestKey())
                    .orElseThrow(() -> e);                       // 중복 요청: 최초 결과를 그대로 반환
        }
    }
}
```

```java
// src/main/java/com/example/order/service/OrderTxService.java
package com.example.order.service;

import com.example.order.domain.Order;
import com.example.order.dto.OrderResult;
import com.example.order.dto.PlaceOrderCommand;
import com.example.order.repository.OrderRepository;
import java.util.Optional;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderTxService {

    private final OrderRepository orderRepository;

    public OrderTxService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    /**
     * timeout은 커밋까지의 예산이다. 외부 PG 호출 같은 블로킹 I/O를 이 안에 넣으면
     * 커넥션을 쥔 채 기다리게 되어 풀이 고갈된다. 외부 호출은 커밋 이후 이벤트로 분리한다.
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW, timeout = 3)
    public OrderResult insert(PlaceOrderCommand command) {
        Order order = orderRepository.saveAndFlush(Order.create(command)); // 유니크 제약을 즉시 확인
        return OrderResult.of(order);
    }

    @Transactional(readOnly = true, propagation = Propagation.REQUIRES_NEW)
    public Optional<OrderResult> findByRequestKey(String requestKey) {
        return orderRepository.findByRequestKey(requestKey).map(OrderResult::of);
    }
}
```

```sql
-- src/main/resources/db/migration/V3__order_request_key.sql
-- 멱등성의 최종 방어선은 애플리케이션 코드가 아니라 DB 제약이다.
ALTER TABLE orders
    ADD COLUMN request_key VARCHAR(64) NOT NULL,
    ADD CONSTRAINT uk_orders_request_key UNIQUE (request_key);

-- fencing token: 락이 만료된 뒤 이전 소유자의 쓰기를 거부한다.
UPDATE job_state
   SET payload = :payload,
       fencing_token = :myToken
 WHERE id = :id
   AND fencing_token < :myToken;   -- 영향 행 0이면 내 락은 이미 빼앗긴 것
```

- 참고: `@Transactional(timeout)`은 JPA 프로바이더에 쿼리 타임아웃 힌트로 전달되며, 실제 적용 범위는 구현에 따라 다릅니다. 확실히 하려면 JDBC `Statement.setQueryTimeout`이나 HikariCP `connection-timeout`·`validation-timeout`을 함께 봅니다. (11-08 노트의 전파·자기호출 주의와 함께 읽기)

### 2.6 Full GC를 부르는 습관

- `System.gc()` — 기본적으로 Full GC를 유발합니다. 일부 라이브러리(RMI, direct buffer 정리)가 호출하며 `-XX:+ExplicitGCInvokesConcurrent`로 완화합니다. `-XX:+DisableExplicitGC`는 Direct 메모리 정리까지 막혀 더 위험할 수 있습니다.
- `finalize()`/과도한 `Cleaner` — 회수 지연과 GC 부담.
- 임시 대형 배열·문자열 — 이미지/리포트/엑셀 생성. 스트리밍(Jackson `JsonGenerator`, StAX)으로 전환.
- humongous 크기 객체 — region 50% 초과 `byte[]`.
- 상한 없는 캐시·컬렉션.

### 2.7 배포 전 체크리스트

- [ ] 컨테이너 limit ≥ 힙 + Metaspace + Direct + CodeCache + 스택 + 10% 여유 (배포 후 RSS 실측으로 검증)
- [ ] `-Xms == -Xmx` (또는 `InitialRAMPercentage == MaxRAMPercentage`)
- [ ] `-Xmx` 또는 `MaxRAMPercentage` 중 하나만 사용
- [ ] 힙 ≤ 31GB 또는 ≥ 48GB (Compressed Oops 경계 회피)
- [ ] GC 로그 파일 로테이션 + `HeapDumpOnOutOfMemoryError` + 덤프 경로 볼륨 확보
- [ ] 커넥션풀 타임아웃 · 쿼리 타임아웃 · 락 lease의 예산 관계를 문서로 남김
- [ ] 배포 후 24시간 p99와 GC pause 상관 확인 (Actuator/JFR)

---

## 3. 기술 면접 단골 꼬리 질문 & 답변 팁

- **Q. GC는 왜 필요한가요? STW는 무엇인가요?**
  - **A.** "개발자가 free를 직접 부르지 않게 하려고 도입됐습니다. GC는 GC Root에서 도달 가능한 객체를 표시(mark)하고 나머지를 회수합니다. STW는 정확한 스냅샷을 얻기 위해 애플리케이션 스레드를 멈추는 구간이고, 회수 방식이 복사·압축이면 객체가 이동하므로 그 순간 이동을 막아야 합니다. 그래서 pause를 줄이는 것이 지연 수집기의 목표입니다."

- **Q. G1과 ZGC의 가장 근본적인 차이는?**
  - **A.** "일을 **언제** 하느냐입니다. G1은 young GC를 STW로 하고, Old 정리(마킹·mixed GC)만 애플리케이션과 동시에 합니다. 그래서 pause가 ms~수백 ms이고, 힙이 커질수록 길어집니다. ZGC는 마킹과 **객체 이동(relocate)까지** 동시에 하며, 참조에 메타데이터를 심은 colored pointer와 load barrier로 이를 가능하게 합니다. pause가 1ms 미만이고 힙 크기와 무관합니다. 대신 모든 참조 접근에 배리어 비용이 붙어 처리량이 조금 떨어집니다."

- **Q. G1은 pause 목표를 어떻게 지키나요?**
  - **A.** "힙을 region으로 나누고, 과거 pause 통계로 region당 회수 비용과 얻는 공간을 추정해 **가비지가 많은 region부터** 목표 시간 안에 들어오는 만큼만 고릅니다. `MaxGCPauseMillis`는 목표이고 확률적으로 접근합니다. 목표를 너무 낮게 잡으면 한 번에 회수하는 region이 줄어 mixed GC가 길어지고 결국 Full GC로 갑니다."

- **Q. 힙을 32GB 넘기면 왜 손해인가요?**
  - **A.** "Compressed Oops가 꺼집니다. 참조가 4바이트에서 8바이트가 되어 같은 객체가 1.5배 커지고 캐시 효율도 떨어집니다. 그래서 33~40GB 구간은 이득이 없고, 48GB 이상으로 넘기거나 31GB 아래로 맞추는 게 실무 기준입니다."

- **Q. minor GC가 자주 발생할 때와 Full GC가 반복될 때, 각각 어떻게 접근하나요?**
  - **A.** "minor가 잦으면 할당률이 높다는 뜻이라 Eden 크기나 `TLAB` 문제를 먼저 봅니다. 다만 Young을 키우면 1회 비용과 승격량이 늘어 Old를 압박합니다. Full GC가 반복되면 거의 항상 **승격된 쓰레기 또는 실제 누수**입니다. class histogram으로 증가하는 클래스를 찾고 heap dump로 붙잡는 경로를 확인합니다. 힙 증설은 원인 제거가 아니라 시간 벌기입니다."

- **Q. OOM이 나면 어떤 순서로 접근하나요?**
  - **A.** "먼저 **어느 영역인지** 메시지로 구분합니다(heap / Metaspace / Direct / native thread). heap이면 `HeapDumpOnOutOfMemoryError`로 덤프를 떠서 dominator tree와 GC Root 경로를 봅니다. `GC overhead limit exceeded`면 거의 누수로 단정합니다. Direct면 NMT로 native 사용을, Metaspace면 클래스 로더 증가를 봅니다. 마지막으로 재발 방지로 타임아웃·상한·캐시 정책을 코드로 고정합니다."

- **Q. Metaspace와 PermGen의 차이는?**
  - **A.** "PermGen은 JDK 8 이전의 힙 내부 고정 크기 영역이라 `-XX:MaxPermSize`로 맞춰야 했고 OOM이 잦았습니다. Metaspace는 JDK 8부터 **native 메모리**로 옮겨져 기본 상한이 없고, 클래스가 언로드되면 반환됩니다. 상한이 없다는 건 컨테이너에서는 **위험**이라는 뜻이라 `-XX:MaxMetaspaceSize`를 두는 편이 안전합니다."

- **Q. GC를 튜닝한다면 무엇부터 하나요?**
  - **A.** "튜닝 전에 측정입니다. ① 실제 적용된 플래그를 `VM.flags`로 확인하고, ② GC 로그로 pause 분포와 Full GC 발생을 보고, ③ p99 지연과 pause 시각이 겹치는지 확인합니다. 겹치지 않으면 GC는 원인이 아니므로 커넥션풀·외부 API·스레드 블로킹을 봐야 합니다. GC가 원인일 때만 힙·GC 선택·객체 생성 패턴 순으로 손댑니다."

- **Q. GC pause가 길어지면 트랜잭션은 어떻게 됩니까?**
  - **A.** "GC는 애플리케이션 스레드만 멈추므로 트랜잭션 자체가 롤백되지는 않습니다. 커밋은 pause 뒤에 정상적으로 됩니다. 문제는 그 사이 클라이언트가 먼저 포기한다는 점입니다. read timeout으로 재시도가 오면 **중복 처리**가 되고, 커넥션을 쥔 채 멈추면 풀이 고갈되며, 쿼리 타임아웃이 걸리면 실제로 롤백됩니다. 분산락은 lease가 만료되어 다른 노드가 같은 작업을 시작합니다. 그래서 pause를 줄이는 것과 함께 **멱등 키·fencing token·타임아웃 예산**을 같이 설계해야 합니다."

- **Q. `System.gc()`를 호출하면 어떻게 되나요?**
  - **A.** "기본 설정에서는 Full GC를 유발해 전체 STW가 걸립니다. 라이브러리가 호출하는 경우가 있어 `ExplicitGCInvokesConcurrent`로 concurrent cycle로 돌리는 편이 낫습니다. `DisableExplicitGC`는 Direct 메모리 정리까지 막을 수 있어 무조건 좋은 선택이 아닙니다."

- **Q. GC가 있는데도 메모리 누수가 생기나요?**
  - **A.** "생깁니다. GC는 **도달 불가능한** 객체만 회수합니다. `static` 컬렉션, `ThreadLocal`, 해제하지 않은 리스너, 상한 없는 캐시처럼 참조가 남아 있으면 영원히 살아남습니다. 즉 GC 언어의 누수는 '안 지워진 것'이 아니라 **'계속 참조되는 것'**입니다."

- **Q. 가상 스레드와 GC는 무관한가요?**
  - **A.** "무관하지 않습니다. 가상 스레드의 스택은 힙에 청크로 저장되므로 스레드 수가 많아지면 **힙 사용량과 GC 부담이 늘어납니다**. 네이티브 스택이 사라지는 대신 힙 예산을 다시 계산해야 하고, 핀(pinning)이 많은 코드는 캐리어 스레드를 붙잡아 지연을 만듭니다."

- **Q. Soft/Weak 참조는 어디에 쓰나요?**
  - **A.** "Soft는 메모리가 부족할 때만 회수되므로 메모리 기반 캐시에, Weak은 다음 GC에서 회수되므로 `WeakHashMap`·`ThreadLocal` 키처럼 **소유자가 사라지면 같이 사라져야 하는** 매핑에 씁니다. 둘 다 '정확히 언제 사라지는지'를 보장하지 않으므로 정합성이 필요한 데이터에는 쓰면 안 됩니다."

---

## 참고

- Oracle JDK 21 GC 튜닝 가이드: https://docs.oracle.com/en/java/javase/21/gctuning/
- G1 상세: https://docs.oracle.com/en/java/javase/21/gctuning/garbage-first-g1-garbage-collector1.html
- JEP 439 (Generational ZGC, JDK 21): https://openjdk.org/jeps/439
- JEP 474 (ZGC 세대형 기본): https://openjdk.org/jeps/474
- JEP 490 (ZGC 비세대형 제거): https://openjdk.org/jeps/490
- JEP 307 (G1 병렬 Full GC): https://openjdk.org/jeps/307
- 통합 로깅(`-Xlog`): https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html
- `jcmd` 레퍼런스: https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html
