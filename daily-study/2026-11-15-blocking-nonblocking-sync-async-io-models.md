# [2026-11-15] Blocking/Non-Blocking vs Sync/Async I/O 모델 — NIO Selector부터 Netty 이벤트 루프, WebFlux까지

> **한 줄 핵심 요약**: Sync/Async는 **"결과를 누가 완료 통지하느냐"**(호출한 스레드 vs 커널·프레임워크), Blocking/Non-Blocking은 **"호출 동안 스레드가 제어권을 잃느냐"** 라는 서로 다른 두 축이며, 실무 선택은 코드 스타일 취향이 아니라 **I/O 바운드 비율·동시 커넥션 수·팀의 디버깅 비용**으로 결정됩니다(스레드 1커넥션 모델 → Netty 이벤트 루프 → 가상 스레드).

---

## 1. 개념 및 동작 원리

### 1.1 두 축은 직교(orthogonal)한다
가장 많이 틀리는 지점입니다. "Non-Blocking = Async"가 아닙니다.

| | **Blocking** (호출 동안 스레드가 멈춤) | **Non-Blocking** (호출 즉시 제어권 반환) |
| --- | --- | --- |
| **Sync** (결과를 호출자가 직접 확인) | `InputStream.read()`, `Socket.connect()`, JDBC `executeQuery()` — 결과가 올 때까지 스레드 점유 | NIO `Selector.select()`, `SocketChannel.read()` → `0` 반환 후 **폴링**해 준비 상태 확인 |
| **Async** (결과를 나중에 통지받음) | `Future.get()`으로 대기, `CountDownLatch.await()` — 통지는 비동기지만 받는 쪽이 블록 | `AsynchronousSocketChannel` + `CompletionHandler`, `CompletableFuture` 체이닝, WebFlux `Mono` |

- **동기/비동기의 기준**: 결과를 **호출한 쪽이 스스로 가져오는가(Sync)** vs **완료 시점에 시스템이 알려주는가(Async)**.
- **블로킹/논블로킹의 기준**: 호출이 **제어 흐름을 즉시 돌려주는가**.
- 따라서 "비동기 + 블로킹"도 존재합니다(`Future.get()`). 실무에서 안 좋은 조합일 뿐, 개념상 불가능한 조합은 아닙니다.

### 1.2 왜 스레드 1개 = 커넥션 1개 모델이 무너지는가 (C10K)
전통적 Blocking I/O 서버는 요청마다 스레드를 하나 붙잡습니다. 문제는 **대기 시간이 CPU 작업이 아니라 커널 버퍼를 기다리는 시간**이라는 점입니다.

1. **메모리**: 플랫폼 스레드는 기본 스택 `-Xss` 1MB 수준. 1만 커넥션이면 스택만 수 GB.
2. **컨텍스트 스위칭**: 대부분의 스레드가 `RUNNABLE`이 아니라 `BLOCKED`/`WAITING`인데도 OS 스케줄러 큐를 오갑니다.
3. **처리량 상한**: Tomcat 기본값이 `maxThreads=200`, `acceptCount=100`인 이유가 여기 있습니다. 201번째 요청부터는 OS accept 큐에서 대기합니다.

동시 처리량은 **리틀의 법칙(Little's Law)** 으로 바로 계산됩니다.

```text
동시 요청 수 = 도착률(RPS) × 평균 응답시간(초)
예) 500 RPS × 0.2s = 100 동시 요청 → 스레드 100개 필요(문제 없음)
예) 5000 RPS × 0.2s = 1000 동시 요청 → 스레드 1000개 → 메모리·스위칭 비용 폭증
```

즉 **동시 커넥션 수 × 대기 시간**이 커질수록 논블로킹 멀티플렉싱의 이득이 커지고, 초당 요청이 적고 응답이 짧으면 Blocking 모델이 오히려 단순하고 빠릅니다.

### 1.3 OS 레벨: 커널 버퍼와 readiness 통지
Non-Blocking I/O의 실체는 **"대기"를 애플리케이션 스레드에서 커널로 옮기는 것**입니다.

```text
[Blocking read]  앱 스레드 ──read()──▶ 커널: 데이터 도착까지 재움 ──▶ 복사 후 반환
[Non-Blocking]   앱 스레드 ──read()──▶ 커널: 없으면 즉시 EAGAIN(Java는 0 반환)
                 (다른 채널 처리)  ...  ──select/epoll_wait──▶ 준비된 fd 목록만 통지
```

| 방식 | 특징 | 비고 |
| --- | --- | --- |
| `select`/`poll` | fd 배열 전체를 매번 순회 → O(N) | fd 1024 제한(select) |
| `epoll`(Linux) | 관심 fd를 커널에 등록, 준비된 것만 이벤트 큐로 → O(1) 수준 | Java NIO Selector가 내부적으로 사용, **level-triggered** 기본 |
| `kqueue`(BSD/macOS), `IOCP`(Windows) | 동등 역할 | 플랫폼별 구현 분기 |
| `io_uring`(Linux 5.1+) | 진짜 비동기(제출/완료 큐) | JDK 표준 NIO는 아직 미사용 |

**중요한 함정**: `Selector`는 **소켓 계열 채널만** 등록할 수 있습니다(`SelectableChannel`). `FileChannel`은 셀렉터에 등록할 수 없어 **Java의 파일 I/O는 항상 블로킹**입니다. `AsynchronousFileChannel`도 리눅스에서는 내부 스레드 풀에 위임하는 방식이라 "커널이 완료를 알려주는" 진짜 비동기는 아닙니다. → "WebFlux로 다 바꿨으니 파일 읽기도 논블로킹"은 면접에서 바로 지적당하는 오답입니다.

### 1.4 Java·Spring 생태계의 계보
```text
JDK 1.0  java.io (Blocking, Stream)
JDK 1.4  java.nio (Buffer/Channel/Selector, Non-Blocking 멀티플렉싱)
JDK 7    NIO.2 (AsynchronousSocketChannel, Path/Files)
Servlet 3.0 (2010)  AsyncContext — 서블릿 스레드는 반환, 응답은 나중에
Servlet 3.1 (2013)  Non-blocking I/O (ReadListener/WriteListener)
Spring 5 (2017)     WebFlux + Reactor (Netty 위에서 전 구간 Non-Blocking)
Spring Boot 3.2 (2023)  spring.threads.virtual.enabled=true — Blocking 코드 그대로 두고 스레드만 경량화
```

---

## 2. 실무 구현: 같은 "외부 API 3개 호출" 을 세 모델로

시나리오: 요청 1건당 외부 API 3개(각 200ms)를 호출해 합쳐 응답한다. 순차 Blocking이면 600ms, 병렬화하면 200ms가 됩니다.

### 2.1 모델 A — Blocking MVC + `CompletableFuture` 병렬화(Tomcat)

```java
// 파일 경로: src/main/java/com/example/io/AggregateController.java
// 의존성: spring-boot-starter-web (Spring Boot 3.3.x, Java 21)

@RestController
@RequiredArgsConstructor
public class AggregateController {

    private final RestClient restClient;                 // Boot 3.2+ 동기 HTTP 클라이언트
    private final ExecutorService ioPool =            // I/O 대기 전용 풀: 코어 수보다 크게
            Executors.newFixedThreadPool(64);

    @GetMapping("/aggregate")
    public AggregateResponse aggregate(@RequestParam String keyword) {
        // 각 호출을 별도 스레드에 위임하고, 마지막에 join(동기 + 블로킹 지점 1회)
        CompletableFuture<Profile>    p = CompletableFuture.supplyAsync(() -> get("/profile", keyword), ioPool);
        CompletableFuture<List<Post>> b = CompletableFuture.supplyAsync(() -> getList("/posts", keyword), ioPool);
        CompletableFuture<Stats>      s = CompletableFuture.supplyAsync(() -> get("/stats", keyword), ioPool);

        // 타임아웃을 반드시 준다: allOf() 자체에는 타임아웃 개념이 없다
        CompletableFuture.allOf(p, b, s).orTimeout(1_500, TimeUnit.MILLISECONDS).join();
        return new AggregateResponse(p.join(), b.join(), s.join());
    }
}
```

- **언제 유리**: 팀이 명령형 코드에 익숙하고, 동시 커넥션이 수백~수천 규모이며, JDBC(블로킹) 의존이 큰 경우.
- **주의**: `ioPool`을 무제한(`newCachedThreadPool`)으로 두면 외부 API가 느려질 때 스레드가 무한 증식합니다. **풀 크기 = 하류가 감당 가능한 동시 호출 수**로 고정해야 합니다.

### 2.2 모델 B — WebFlux + Reactor(Netty, 전 구간 Non-Blocking)

```java
// 파일 경로: src/main/java/com/example/io/ReactiveAggregateHandler.java
// 의존성: spring-boot-starter-webflux (reactor-netty), spring-boot-starter-data-r2dbc
// import static org.springframework.web.reactive.function.server.RequestPredicates.GET;
// import static org.springframework.web.reactive.function.server.RouterFunctions.route;

@Configuration
public class ReactiveAggregateHandler {

    @Bean
    public RouterFunction<ServerResponse> routes(WebClient client, PostR2dbcRepository repo) {
        return route(GET("/aggregate"),
                req -> {
                    String keyword = req.queryParam("keyword").orElse("");
                    // zip: 세 개의 Mono를 동시에 구독하고 모두 완료되면 튜플로 결합
                    Mono<AggregateResponse> body = Mono.zip(
                            client.get().uri("/profile?k={k}", keyword).retrieve().bodyToMono(Profile.class),
                            client.get().uri("/posts?k={k}", keyword).retrieve().bodyToFlux(Post.class).collectList(),
                            repo.findByKeyword(keyword).collectList()
                    ).map(t -> new AggregateResponse(t.getT1(), t.getT2(), t.getT3()))
                     .timeout(Duration.ofMillis(1500));   // Reactive에는 타임아웃 연산자가 1급 시민

                    return ServerResponse.ok().body(body, AggregateResponse.class);
                });
    }
}
```

- **언제 유리**: 커넥션이 수만 개로 늘어나는 스트리밍·SSE·게이트웨이·BFF, 하류가 모두 논블로킹(R2DBC, Lettuce)일 때.
- **주의**: 체인 중간에 JDBC 한 줄만 들어가도 이벤트 루프 스레드가 통째로 막힙니다(3장 참조).

### 2.3 모델 C — Virtual Threads(코드는 Blocking, 런타임은 경량)

```java
// 파일 경로: src/main/resources/application.yml
spring:
  threads:
    virtual:
      enabled: true      # Boot 3.2+: Tomcat 요청 처리·@Async가 가상 스레드로 실행
```

```java
// 파일 경로: src/main/java/com/example/io/VirtualThreadAggregate.java
// 선행 학습: daily-study/2026-11-04-java21-virtual-threads-and-pinning.md (피닝 주의)

@Configuration
public class VirtualThreadAggregate {

    @Bean
    public ExecutorService virtualIo() {
        // 가상 스레드는 "풀링하지 않는다". 매 요청 새로 만들어도 생성 비용이 힙 객체 수준
        return Executors.newVirtualThreadPerTaskExecutor();
    }
}
```

```java
// 서비스 계층: 모델 A와 코드가 사실상 동일하다(블로킹 API를 그대로 쓴다)
public AggregateResponse aggregate(String keyword) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {   // Java 21: preview
        var p = scope.fork(() -> restClient.get().uri("/profile?k={k}", keyword).retrieve().body(Profile.class));
        var b = scope.fork(() -> getPosts(keyword));
        var s = scope.fork(() -> getStats(keyword));
        scope.joinUntil(Instant.now().plusMillis(1_500));
        scope.throwIfFailed();
        return new AggregateResponse(p.get(), b.get(), s.get());
    }
}
```

### 2.4 세 모델 비교 — 선택 기준

| 항목 | Blocking MVC(+pool) | WebFlux(Netty) | Virtual Threads |
| --- | --- | --- | --- |
| **코드 복잡도** | 낮음 | 높음(연산자 체이닝, 컨텍스트 전파) | 낮음(기존 코드 그대로) |
| **스택 트레이스/디버깅** | 친숙 | 어려움(구독 시점 기준) | 친숙 |
| **커넥션 수 확장성** | 스레드 수에 비례(수백~수천) | 매우 높음(수만) | 높음(스레드 비용은 해결, 커넥션·커널은 별개) |
| **블로킹 라이브러리 사용** | 자유 | **금지**(JDBC 포함) | 자유 |
| **전환 비용** | — | R2DBC·리액티브 스택 전면 재작성 | 설정 1줄 + 피닝 점검 |
| **적합 사례** | CRUD 백오피스, 사내 시스템 | API 게이트웨이, 스트리밍, 다수 외부 호출 팬아웃 | 신규 Spring Boot 3.2+ 서비스 기본값 후보 |

> 정리: **"논블로킹이 더 좋은 기술"이 아니라 "대기 시간을 스레드에 묶어두지 않는 설계"가 목적**입니다. 가상 스레드는 같은 목적을 코드 변경 없이 달성하는 쪽이고, WebFlux는 스트림·백프레셔까지 필요할 때 선택합니다.

---

## 3. 실무 적용 및 주의사항 (Edge Cases)

1. **Non-Blocking이 항상 빠르지 않다**
   - 단일 요청의 **지연(latency)** 은 오히려 늘 수 있습니다. 이벤트 루프 큐잉·콜백 스케줄링·직렬화 오버헤드가 붙기 때문입니다. 논블로킹의 이득은 **처리량(throughput)** 과 **메모리 효율**에서 나옵니다. "우리 API가 느린데 WebFlux로 바꾸면 빨라지나요?"의 답은 대개 **아니오**입니다(병목이 DB 쿼리라면).
2. **CPU 바운드 작업을 논블로킹으로 바꾸는 건 무의미**
   - 암호화·압축·대량 직렬화는 스레드가 실제로 일하는 구간이라 이득이 없습니다. 오히려 이벤트 루프를 막아 전체 지연이 커집니다.
3. **이벤트 루프 스레드에서 절대 블로킹 금지**(WebFlux 최다 사고)
   - 금지 대상: JDBC, `Thread.sleep`, `InputStream`/`Files` I/O, `bcrypt` 같은 CPU 무거운 해시, 동기 HTTP 클라이언트 호출.
   - 불가피하면 반드시 별도 스케줄러로 격리합니다.
   ```java
   // 잘못된 예: 이벤트 루프(reactor-http-nio-N)에서 JDBC 실행 → 커넥션 수만큼 서버 정지
   Mono.fromCallable(() -> jdbcTemplate.queryForList(SQL))   // ← 여기까지는 안전
       .subscribeOn(Schedulers.boundedElastic())             // ← 반드시 격리(최대 10×CPU 스레드)
       .doOnNext(list -> { /* 이 콜백은 다시 이벤트 루프에서 실행된다 */ });
   ```
   - 참고: WebFlux에서 `block()`을 이벤트 루프 스레드에서 호출하면 Reactor가 `IllegalStateException`으로 막아줍니다. 이건 안전망이지 설계 허가가 아닙니다.
4. **리액티브 트랜잭션은 ThreadLocal이 아니라 구독 컨텍스트에 묶인다**
   - 명령형 `@Transactional`은 ThreadLocal 기반이라 리액티브 체인에서 그대로 쓰면 트랜잭션이 유실될 수 있습니다. R2DBC를 쓴다면 `R2dbcTransactionManager`(또는 `TransactionalOperator`)로 **리액티브 반환 타입**에 맞춰 경계를 잡고, 체인 중간에 `publishOn`/`subscribeOn`으로 스케줄러를 바꿔도 컨텍스트가 유지되는지 확인해야 합니다.
   - 같은 이유로 **MDC 로그 추적 ID**도 ThreadLocal에 있으면 사라집니다. Reactor는 `contextCapture()`(Micrometer Context Propagation)로 컨텍스트를 전파합니다.
5. **가상 스레드의 피닝(Pinning)** — Java 21 기준
   - `synchronized` 블록 안에서 블로킹 I/O를 하면 캐리어 스레드가 고정되어 처리량이 급락합니다. `ReentrantLock`으로 대체하고 `-Djdk.tracePinnedThreads=short`로 탐지합니다(JDK 24의 JEP 491에서 `synchronized` 피닝은 해소됨).
   - 더 흔한 함정: **`ThreadLocal` 다량 사용 시 메모리 폭증**(수만 개 가상 스레드 × 각자의 ThreadLocal 맵). 스레드 풀 재사용을 전제로 만든 캐시성 ThreadLocal은 재검토 대상입니다.
6. **커넥션 풀·백프레셔가 새로운 병목**
   - 논블로킹으로 스레드 병목을 없애면 다음 병목은 **DB 커넥션 풀(HikariCP 기본 10)** 과 하류 API의 Rate Limit입니다. 스레드만 늘리면 큐 대기가 커넥션 풀 앞으로 이동할 뿐입니다.
   - 리액티브 스트림에서는 소비 속도를 제어해야 합니다. `flatMap(fn, concurrency)`로 동시 구독을 제한하고, 무한 소스에는 `onBackpressureBuffer/Drop` 정책을 명시합니다. 백프레셔 없는 `Flux`는 메모리 폭발로 이어집니다.
7. **숨은 블로킹 지점 체크리스트**
   - DNS 조회(`InetAddress.getByName`, 캐시 미스 시 시스템 리졸버 호출), JWT 공개키 로테이션 시 원격 JWKS 조회, 로컬 파일 로그 appender, `@Transactional` 첫 쿼리 전 커넥션 획득 대기.
   - 외부 호출에는 **connect timeout과 read timeout을 각각** 지정합니다(리액티브는 `.timeout()`, 동기는 `RestClient` request factory 타임아웃).
8. **트랜잭션 안에서 외부 I/O 하지 않기**(모델과 무관한 원칙)
   - 커넥션을 잡은 채 외부 API 200ms를 기다리면, 스레드 모델이 무엇이든 **커넥션 풀 고갈**로 이어집니다. 외부 호출은 트랜잭션 밖으로 빼고, 결과 반영만 짧은 트랜잭션으로 처리합니다.

---

## 4. 기술 면접 단골 꼬리 질문 & 답변 팁

- **Q. Blocking/Non-Blocking과 Sync/Async의 차이를 한 문장으로 설명해 보세요.**
  - **A.** "Sync/Async는 **결과 통지 방식**(호출자가 직접 받는가 vs 나중에 통보받는가), Blocking/Non-Blocking은 **호출 중 제어권 소유**(스레드가 멈추는가 vs 즉시 돌아오는가)입니다. 두 축은 독립이라 `Future.get()`처럼 '비동기지만 블로킹'인 조합도 존재합니다."

- **Q. Non-Blocking I/O가 항상 더 빠른가요?**
  - **A.** "아닙니다. 개별 요청의 지연은 큐잉·콜백 오버헤드 때문에 오히려 늘 수 있습니다. 논블로킹은 **대기 시간을 스레드에서 분리**해 적은 스레드로 많은 동시 커넥션을 처리하는 **처리량·메모리** 최적화이고, 병목이 DB 쿼리나 CPU라면 효과가 없습니다."

- **Q. 그럼 Java 21 가상 스레드가 있으면 WebFlux는 필요 없나요?**
  - **A.** "상당 부분 대체됩니다. 가상 스레드는 블로킹 코드를 그대로 두고 스레드 비용만 없애므로 전환 비용이 낮습니다. 다만 ① 스트리밍·SSE처럼 **백프레셔와 무한 스트림**을 다루는 경우, ② 하류까지 전 구간 논블로킹(R2DBC·Lettuce)으로 커넥션 수를 극한까지 늘려야 하는 게이트웨이, ③ 이미 리액티브 스택을 쓰는 팀에서는 WebFlux가 여전히 유효합니다."

- **Q. Netty는 왜 이벤트 루프 스레드를 `CPU 코어 수 × 2`로 두나요?**
  - **A.** "이벤트 루프 스레드는 I/O 대기 없이 CPU만 쓰는 구조라 코어 수와 1:1이 이론적 최적입니다. ×2는 시스템 콜·GC·기타 스레드와의 경합을 고려한 여유값입니다. 코어 수보다 훨씬 크게 잡으면 컨텍스트 스위칭만 늘고 이득이 없습니다."

- **Q. WebFlux에서 `@Transactional`을 그대로 쓰면 어떤 문제가 있나요?**
  - **A.** "명령형 트랜잭션은 ThreadLocal 기반인데 리액티브 체인은 스케줄러를 넘나들며 실행될 수 있어 트랜잭션 컨텍스트가 유실될 수 있습니다. R2DBC를 쓴다면 `R2dbcTransactionManager`/`TransactionalOperator`로 **구독 컨텍스트 기반** 경계를 설정하고, 체인 중간 블로킹(JPA)이 섞이지 않도록 분리해야 합니다."

- **Q. select와 epoll의 차이, 그리고 왜 epoll이 확장성이 좋은가요?**
  - **A.** "`select`는 매 호출마다 감시할 fd 집합 전체를 커널에 복사하고 순회해 O(N)이고 fd 1024개 제한도 있습니다. `epoll`은 감시 대상 fd를 커널에 **한 번 등록**해두고 준비된 fd만 이벤트 큐로 반환하므로, 감시 대상이 늘어도 비용이 선형으로 늘지 않습니다. Java NIO `Selector`가 리눅스에서 이 epoll을 사용합니다."

- **Q. 이 서비스에 어떤 I/O 모델을 고르시겠어요? (지원 회사 서비스 기준)**
  - **A.** "동시 커넥션 규모와 병목 위치를 먼저 봅니다. 커넥션이 수백 수준이고 JDBC 중심 CRUD라면 Blocking MVC가 유지보수 비용이 가장 낮습니다. 수만 커넥션의 스트리밍·팬아웃이면 논블로킹(Netty/WebFlux)을, 신규 서비스에서 코드 단순성과 확장성을 함께 원하면 **Spring Boot 3.2+ 가상 스레드**를 먼저 검토하겠습니다."
