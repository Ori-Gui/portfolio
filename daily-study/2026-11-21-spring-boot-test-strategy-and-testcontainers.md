# [2026-11-21] Spring Boot 테스트 전략 — 슬라이스 테스트·Testcontainers·@Transactional 롤백 함정과 컨텍스트 캐시

> **한 줄 핵심 요약**: Spring Boot 테스트의 성패는 "무엇을 실제로 띄우고 무엇을 대체할 것인가"를 계층별로 정하는 데서 갈립니다. 단위 테스트는 프레임워크 없이 도메인을 검증하고, **슬라이스 테스트**(`@WebMvcTest`, `@DataJpaTest`)는 필요한 자동구성만 로드하며, **통합 테스트**는 `@SpringBootTest` + **Testcontainers의 실제 MySQL**로 검증합니다. 이때 (1) 슬라이스가 로드하지 않는 빈은 `@MockitoBean`으로 채워야 하고, (2) 테스트에 붙은 `@Transactional`은 **롤백 때문에 커밋 시점 로직(제약조건·이벤트·동시성)을 검증하지 못하게 만들며**, (3) 테스트가 느려지는 진짜 원인은 코드가 아니라 **컨텍스트 캐시가 깨져 매번 ApplicationContext를 새로 띄우는 것**입니다.

---

## 1. 개념 및 동작 원리

### 1.1 테스트 피라미드보다 먼저 정할 것 — "이 테스트가 깨지면 무엇을 알 수 있나"

테스트 개수 비율을 외우는 것보다 중요한 건 **각 계층이 잡아내는 결함의 종류**를 구분하는 것입니다.

| 계층 | 검증 대상 | Spring 컨텍스트 | 대략 실행 시간 | 잡는 결함 |
| --- | --- | --- | --- | --- |
| 단위(도메인·서비스 로직) | 값 객체 불변식, 상태 전이, 계산 | ❌ 없음 | ms | 로직 오류, 경계값 |
| 슬라이스 웹(`@WebMvcTest`) | 라우팅, 요청/응답 직렬화, 검증, 예외 → 상태코드 | ✅ (웹 계층만) | 0.5~2s | API 계약 위반, DTO 매핑 |
| 슬라이스 JPA(`@DataJpaTest`) | 쿼리, 연관관계 매핑, JPQL/QueryDSL | ✅ (JPA만) | 0.5~2s | 잘못된 매핑, N+1, 쿼리 오류 |
| 통합(`@SpringBootTest`) | 빈 배선, 트랜잭션 경계, 보안 필터, 외부 시스템 | ✅ 전체 | 2~10s (+컨테이너 5~20s) | 배선 오류, 실제 SQL/방언 차이 |

- **단위 테스트가 가능한 코드인지가 설계 품질의 지표**입니다. 서비스가 `EntityManager`와 `HttpServletRequest`에 직접 의존하면 단위 테스트를 쓸 수 없습니다. 도메인 규칙을 엔티티·값 객체로 밀어 넣으면 순수 Java 테스트로 검증되고, 서비스는 조합(오케스트레이션)만 남습니다.
- **슬라이스 테스트는 "빠른 통합 테스트"가 아니라 "좁힌 통합 테스트"** 입니다. 로드하는 자동구성이 줄어들 뿐, 실제 `DispatcherServlet`·실제 JPA 구현체를 씁니다. 그래서 MockMvc는 진짜 HTTP를 타지 않지만 컨트롤러 매핑·직렬화·검증은 실제로 수행합니다.
- **비율은 프로젝트 성격에 따라 달라집니다.** CRUD 중심 백엔드는 슬라이스+통합 비중이 커지고, 계산·정산 로직이 많은 서비스는 단위 테스트 비중이 커집니다. "피라미드를 지켰는가"보다 **핵심 도메인 규칙이 단위 테스트로 덮여 있는가**가 먼저입니다.

### 1.2 `@SpringBootTest`의 `webEnvironment` — 무엇이 진짜로 뜨는가

| 모드 | 기동되는 것 | MockMvc 사용 | 특징 |
| --- | --- | --- | --- |
| `MOCK`(기본) | 웹 `ApplicationContext` (서블릿 컨테이너 없음) | ✅ `@AutoConfigureMockMvc` | 가장 빠름, 포트 없음, 필터 체인은 등록됨 |
| `RANDOM_PORT` | 실제 내장 서버(랜덤 포트) | ❌ (`TestRestTemplate`, `WebTestClient`) | 진짜 HTTP·스레드·커넥션 풀, **테스트 트랜잭션이 서버에 적용되지 않음** |
| `DEFINED_PORT` | 실제 서버(고정 포트) | ❌ | 포트 충돌 위험, 병렬 실행 시 특히 |
| `NONE` | 웹 환경 없는 컨텍스트 | ❌ | 배치·스케줄러 등 비웹 검증 |

```java
// src/test/java/com/example/order/OrderApiIntegrationTest.java
package com.example.order;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderApiIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void getOrder_returnsReadyStatus() {
        ResponseEntity<String> response =
                restTemplate.getForEntity("/api/orders/{id}", String.class, 1L);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).contains("\"status\":\"READY\"");
    }
}
```

- **핵심 차이는 스레드**입니다. `RANDOM_PORT`에서는 HTTP 클라이언트와 서버가 **서로 다른 스레드 = 서로 다른 트랜잭션**에서 돕니다. 그래서 테스트 메서드에 `@Transactional`을 붙여도 **서버 쪽에서 커밋된 데이터는 롤백되지 않습니다**(1.5절). 반대로 `MOCK` + MockMvc는 같은 스레드에서 호출되므로 테스트 트랜잭션이 서비스 계층까지 이어집니다.
- **MockMvc는 "필터를 건너뛴다"는 오해**가 흔합니다. `@AutoConfigureMockMvc`는 시큐리티 필터 체인을 포함해 등록된 필터를 실제로 태웁니다. 그래서 인증 없이 호출하면 401이 나고, `@WithMockUser`나 토큰 준비가 필요합니다. 필터를 끄려면 `addFilters = false`를 명시합니다.
- **MockMvc를 쓰면 안 되는 경우**: 서블릿 비동기(`DeferredResult`, `StreamingResponseBody`), 실제 타임아웃, 멀티파트 대용량, HTTP/2, 실제 커넥션 풀 고갈. 이들은 `RANDOM_PORT`로 검증해야 합니다.

### 1.3 슬라이스 테스트는 "필터"로 동작한다 — 무엇이 스캔되지 않는지가 함정

`@WebMvcTest`, `@DataJpaTest` 같은 애노테이션은 내부적으로 다음 세 가지를 합니다.

1. `@OverrideAutoConfiguration(enabled = false)` — 전체 자동구성 대신 **해당 슬라이스의 자동구성만** `@ImportAutoConfiguration`으로 가져옵니다.
2. `@TypeExcludeFilters(...TypeExcludeFilter.class)` — `@SpringBootApplication`의 컴포넌트 스캔에 **제외 필터**를 추가해, 해당 계층의 컴포넌트만 빈으로 등록합니다.
3. 필요하면 `@Transactional`, `@AutoConfigureTestDatabase`, `@AutoConfigureTestEntityManager` 등을 함께 겁니다.

| 슬라이스 | 스캔되는 컴포넌트 | 등록 안 되는 것 |
| --- | --- | --- |
| `@WebMvcTest` | `@Controller`, `@ControllerAdvice`, `@JsonComponent`, `Converter`, `Filter`, `WebMvcConfigurer` | `@Service`, `@Repository`, `@Component` 일반 빈 |
| `@DataJpaTest` | `@Entity`, Spring Data JPA 리포지토리, `@AutoConfigureTestEntityManager` | `@Service`, `@Controller`, 일반 `@Component` |
| `@JsonTest` | Jackson 모듈·직렬화 설정 | 그 외 전부 |

```java
// src/test/java/com/example/order/web/OrderControllerTest.java
package com.example.order.web;

import com.example.order.service.OrderService;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.BDDMockito.given;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean            // Boot 3.4+ 표준 (구 @MockBean은 4.0에서 제거 예정)
    private OrderService orderService;

    @Test
    @DisplayName("존재하는 주문을 조회하면 200과 주문 상태를 반환한다")
    void get_returnsOrderResponse() throws Exception {
        given(orderService.findById(1L)).willReturn(new OrderResponse(1L, "READY", 10_000));

        mockMvc.perform(get("/api/orders/{id}", 1L).accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status").value("READY"));
    }
}
```

- **`@DataJpaTest`는 기본값이 "임베디드 DB로 교체"** 입니다. `@AutoConfigureTestDatabase`가 기본으로 `replace = ANY`를 걸어, 설정에 있는 DataSource까지 H2 같은 인메모리 DB로 바꿔치기합니다(`show-sql`도 기본 `true`).
- **실제 MySQL로 돌리려면 명시적으로 꺼야** 합니다. 이때 주의할 점이 있습니다. **`replace = NONE`만으로는 "진짜 DB에 연결"이 보장되지 않습니다.** 교체(auto-config가 DataSource 빈을 바꿔치기하는 동작)만 꺼질 뿐, `spring.datasource.url`이 없으면 `DataSourceAutoConfiguration`이 클래스패스의 임베디드 드라이버로 H2를 새로 만들어 버립니다. 실제 DB로 붙으려면 **`@ServiceConnection`이나 실제 URL 설정으로 접속 정보가 먼저 존재해야** 합니다.
- `@ServiceConnection`은 슬라이스에서도 동작합니다. JDBC 슬라이스의 자동구성 목록에 `ServiceConnectionAutoConfiguration`이 **`optional:`로 등록**되어 있어, `spring-boot-testcontainers`가 클래스패스에 있으면 `@DataJpaTest` 안에서도 컨테이너 접속 정보가 반영됩니다.

```java
// src/test/java/com/example/member/MemberRepositoryTest.java
@RepositoryTest  // 아래에서 정의: @DataJpaTest + replace = NONE (+ 컨테이너 상속)
class MemberRepositoryTest extends AbstractContainerTest {

    @Autowired
    private MemberRepository memberRepository;

    @Test
    void findActiveMembers_returnsOnlyActive() {
        memberRepository.saveAll(List.of(
                Member.active("junha"),
                Member.withdrawn("old-user")));

        List<Member> found = memberRepository.findAllByStatus(Status.ACTIVE);

        assertThat(found).extracting(Member::getName).containsExactly("junha");
    }
}
```

```java
// src/test/java/com/example/support/RepositoryTest.java — 커스텀 합성 애노테이션
package com.example.support;

import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.context.annotation.Import;

import java.lang.annotation.*;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@DataJpaTest                                   // @Transactional 포함
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)  // 실제 DB 사용
@Import(QueryDslConfig.class)                  // QueryDSL JPAQueryFactory 등 슬라이스에 없는 빈
public @interface RepositoryTest {
}
```

- **QueryDSL 커스텀 구현체가 등록되지 않는 문제**는 실무 단골입니다. `@DataJpaTest`의 제외 필터는 "JPA 리포지토리·엔티티" 위주로 스캔하므로, `OrderRepositoryImpl` 같은 클래스가 빈으로 올라오지 않아 `NoSuchBeanDefinitionException`이 납니다. `@Import`로 명시하거나 커스텀 합성 애노테이션에 넣어 둡니다.
- **시큐리티 의존성이 있으면 `@WebMvcTest`에서 401이 나는 것**도 같은 원리입니다(필터 체인 자동구성은 로드, 인증 정보는 없음). `@WithMockUser`, 실제 토큰 헤더, 또는 `addFilters=false` 중 검증 목적에 맞는 것을 고릅니다. 인가 규칙 자체를 검증할 거라면 필터를 끄면 안 됩니다.

### 1.4 컨텍스트 캐시 — 테스트가 느린 진짜 원인

Spring TestContext 프레임워크는 **설정이 동일한 테스트끼리 `ApplicationContext`를 재사용**합니다. 캐시 키는 대략 다음 조합입니다.

- `locations`(설정 클래스), 활성 프로파일, `PropertySource`/`@TestPropertySource`
- 컨텍스트 커스터마이저(`@MockitoBean`/`@MockBean` 같은 **bean override도 여기에 반영됨**)
- `ContextInitializer`, 상속된 `@ContextConfiguration`

```text
같은 키  → 컨텍스트 재사용 (0.0s)
다른 키  → 새 ApplicationContext 기동 (2~10s, 통합 테스트면 그 이상)
@DirtiesContext → 현재 컨텍스트를 캐시에서 제거 (다음 테스트는 무조건 재기동)
```

- **테스트 100개가 30초 걸리면 코드가 느린 게 아니라 컨텍스트를 10번 띄우고 있는 경우가 대부분**입니다. `-Dlogging.level.org.springframework.test.context.cache=DEBUG`로 `Spring TestContext Cache Hit Ratio`를 확인해 원인을 수치로 확인합니다.
- **캐시를 깨는 요인**: 클래스마다 다른 `@MockitoBean` 조합, `@TestPropertySource`의 값 차이, `@ActiveProfiles` 불일치, `@DirtiesContext`, `@Transactional` 여부 차이(같은 클래스 안에서도 메서드별로 다르면 별도), `@DynamicPropertySource`의 값(컨테이너 포트가 매번 달라질 수 있음).
- **캐시 크기**는 기본 32개(`spring.test.context.cache.maxSize`)입니다. 캐시가 넘치면 LRU로 밀려나 재기동이 발생합니다. 프로파일·목 조합을 늘리기 전에 **공통 베이스 클래스로 설정을 통일**하는 편이 낫습니다.
- **`@DirtiesContext`는 최후의 수단**입니다. 전역 상태(스케줄러, 캐시, 정적 필드)를 바꾸는 테스트에서만 쓰고, 가능하면 해당 상태를 빈으로 분리해 테스트마다 리셋합니다.

### 1.5 테스트용 `@Transactional` — 편리함의 대가

`@SpringBootTest`(또는 `@DataJpaTest`)에 `@Transactional`이 붙으면 각 테스트 메서드가 **하나의 트랜잭션 안에서 실행되고 끝나면 롤백**됩니다. 데이터 초기화 코드가 필요 없어 편하지만, 다음 다섯 가지를 가립니다.

| 가려지는 것 | 왜 문제인가 | 실무에서 드러나는 증상 |
| --- | --- | --- |
| **커밋 시점 검증** | 제약조건 위반·유니크 충돌·낙관적 락 충돌은 flush/commit에서 터짐 | 테스트는 통과, 운영에서 `DataIntegrityViolationException` |
| **`AFTER_COMMIT` 리스너** | 커밋이 없으면 `@TransactionalEventListener(phase = AFTER_COMMIT)`가 실행되지 않음 | 이벤트 발행 테스트가 조용히 아무것도 검증하지 못함 |
| **1차 캐시·지연 로딩** | 같은 트랜잭션이라 영속성 컨텍스트가 살아 있어 지연 로딩이 항상 성공 | 운영에서 `LazyInitializationException` |
| **별도 트랜잭션(`REQUIRES_NEW`)** | 바깥 트랜잭션에 묶여 실제 격리 동작이 재현되지 않음 | 동시성·데드락 재현 실패 |
| **실제 스레드 경계** | `RANDOM_PORT`면 서버는 다른 스레드·다른 트랜잭션 | 클라이언트 트랜잭션 롤백 ≠ 서버 데이터 롤백 |

```java
// flush 시점을 명시해야 "DB에 반영된 결과"를 검증할 수 있다
@Test
void updateMemberName_isPersistedAndReloaded() {
    Member saved = memberRepository.save(Member.active("before"));

    saved.rename("after");
    entityManager.flush();   // UPDATE 발행
    entityManager.clear();   // 1차 캐시 제거 → 다음 조회는 DB에서 읽음

    Member reloaded = memberRepository.findById(saved.getId()).orElseThrow();
    assertThat(reloaded.getName()).isEqualTo("after");
}
```

- **`flush()` 없이 `save()`만 하고 조회하면** 1차 캐시에서 같은 인스턴스가 반환되어, 실제 SQL 결과가 아닌 메모리 상태를 검증하게 됩니다. `saveAndFlush()`도 같은 목적입니다.
- **커밋이 필요한 로직은 `TestTransaction`으로 커밋을 강제**합니다.

```java
import org.springframework.test.context.transaction.TestTransaction;

@Test
void orderEvent_isPublishedAfterCommit() {
    orderService.place(command);

    TestTransaction.flagForCommit();
    TestTransaction.end();        // 여기서 실제 커밋 → AFTER_COMMIT 리스너 실행

    assertThat(eventCaptor.getAllValues()).hasSize(1);

    TestTransaction.start();      // 이후 정리를 위해 트랜잭션 재시작
}
```

- **`RANDOM_PORT` + `@Transactional` 조합은 대부분 의미가 없습니다.** 서버 쪽 트랜잭션은 테스트 트랜잭션과 무관하게 커밋되므로 데이터가 남고, 테스트 간 간섭이 생깁니다. 이 경우 `@Transactional`을 빼고 **테스트 데이터를 직접 정리**하거나(`@Sql` 실행 전 TRUNCATE, `@AfterEach` 삭제), 컨테이너·스키마를 테스트마다 새로 만드는 방식을 씁니다.
- 판단 기준은 단순합니다. **"운영에서 커밋된 뒤에 벌어지는 일"을 검증하는가?** 그렇다면 커밋을 실제로 일으켜야 하고(비트랜잭션 테스트 + 정리), **"쿼리·매핑이 의도대로인가"만 본다면** 롤백 트랜잭션이 효율적입니다.

### 1.6 Testcontainers — 임베디드 DB와 실제 MySQL은 다른 것을 검증한다

H2는 편하지만 **MySQL이 아닙니다.** 방언을 흉내 내는 것과 InnoDB가 동작하는 것은 다릅니다.

| 항목 | H2 (MySQL mode) | MySQL 8.0 (InnoDB) |
| --- | --- | --- |
| `ON DUPLICATE KEY UPDATE` | 미지원/부분 지원 | 지원 |
| 콜레이션·대소문자 | 기본 대소문자 구분 없음 | `utf8mb4_0900_ai_ci`(구분 안 함) vs `_bin`(구분) |
| 락 동작 | 테이블/행 단순 락 | **next-key lock, 갭 락**, `SELECT ... FOR UPDATE` 대기 |
| JSON 함수 | 일부만 | `JSON_EXTRACT`, `->>`, 다중값 인덱스 |
| 윈도우 함수 | 지원하지만 미묘한 차이 | 8.0 지원 |
| `EXPLAIN`·옵티마이저 | 없음(실행계획 검증 불가) | 커버링 인덱스·실행계획 검증 가능 |
| DDL 트랜잭션 | 자동 커밋 동작 차이 | DDL 즉시 커밋 |

```groovy
// build.gradle — 버전은 Spring Boot BOM이 관리한다
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.boot:spring-boot-testcontainers'
    testImplementation 'org.testcontainers:junit-jupiter'
    testImplementation 'org.testcontainers:mysql'
}
```

```java
// src/test/java/com/example/support/AbstractContainerTest.java
// 컨테이너만 제공하는 베이스 — 슬라이스와 통합 테스트가 함께 상속한다
package com.example.support;

import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.testcontainers.containers.MySQLContainer;

public abstract class AbstractContainerTest {

    // static 필드 + 정적 초기화: JVM 당 1회만 기동 (싱글턴 컨테이너 패턴)
    @ServiceConnection
    static final MySQLContainer<?> MYSQL = new MySQLContainer<>("mysql:8.0")
            .withDatabaseName("app")
            .withUsername("app")
            .withPassword("app")
            .withCommand("--character-set-server=utf8mb4",
                         "--collation-server=utf8mb4_0900_ai_ci");

    static {
        MYSQL.start();          // JUnit 확장 대신 직접 기동. 종료는 Ryuk이 담당
    }
}
```

```java
// 통합 테스트는 컨테이너 베이스 + @SpringBootTest 조합으로
@SpringBootTest
@ActiveProfiles("test")
class OrderPlacementIntegrationTest extends AbstractContainerTest {
    // ...
}
```

- **`@SpringBootTest`와 `@DataJpaTest`를 한 클래스에 같이 붙이면 안 됩니다.** 둘 다 컨텍스트 부트스트래퍼를 지정하는 애노테이션이라 충돌합니다. 그래서 **컨테이너만 담은 베이스 클래스**를 두고, 슬라이스 테스트는 `@RepositoryTest`, 통합 테스트는 `@SpringBootTest`를 각자 붙여 상속합니다.

- **`@ServiceConnection`**(Boot 3.1+)은 컨테이너의 접속 정보로 `JdbcConnectionDetails` 빈을 만들어, `spring.datasource.*` 설정을 **덮어씁니다**. `@DynamicPropertySource`로 URL·계정을 손으로 등록하던 코드가 사라지고, MySQL·Redis·Kafka·PostgreSQL 등은 타입별 컨테이너만 있으면 자동 인식됩니다.
- **싱글턴 패턴이 필요한 이유**: `@Container static`을 쓰면 컨테이너는 **테스트 클래스마다** 시작/정지됩니다. 그런데 Spring 컨텍스트는 캐시되어 재사용되므로, 두 번째 클래스에서 컨테이너가 새 포트로 뜨면 **캐시된 컨텍스트는 죽은 포트를 보고 있습니다**. 정적 필드 + `static { start(); }`로 JVM 수명 동안 한 번만 띄우면 이 불일치가 사라집니다.
- **`@ServiceConnection`이 매칭되지 않으면**(커스텀 이미지, `GenericContainer`) `name` 속성으로 힌트를 주거나 `@DynamicPropertySource`로 되돌아갑니다.

```java
// @ServiceConnection이 안 맞을 때의 대체 수단
@DynamicPropertySource
static void registerProperties(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", MYSQL::getJdbcUrl);
    registry.add("spring.datasource.username", MYSQL::getUsername);
    registry.add("spring.datasource.password", MYSQL::getPassword);
}
```

- **로컬에서 재사용**은 `~/.testcontainers.properties`에 `testcontainers.reuse.enable=true`를 두고 컨테이너에 `.withReuse(true)`를 주면 됩니다. CI에서는 재사용이 오히려 상태 오염을 만들므로 켜지 않습니다.
- **개발 시점에도 같은 컨테이너를 쓸 수 있습니다.** `spring-boot-docker-compose`나 Testcontainers의 개발 모드(`@TestConfiguration`에서 컨테이너 빈 선언 + `SpringApplication.from(...).with(...)`)로, 개발자가 MySQL을 직접 설치하지 않아도 됩니다. 이때 스키마는 Flyway/Liquibase로 프로덕션과 동일하게 만듭니다.
- **Flyway를 테스트에서도 돌리는 것이 신뢰도의 핵심**입니다. `ddl-auto=create`로 만든 스키마는 마이그레이션 스크립트의 오류를 잡지 못합니다. `@DataJpaTest`(replace=NONE) + Testcontainers + Flyway 조합이면 "실제 MySQL + 실제 스키마"에서 리포지토리를 검증합니다.

### 1.7 목 대체 — `@MockBean`에서 `@MockitoBean`으로

Spring Boot 3.4에서 `@MockBean`/`@SpyBean`은 **deprecated(제거 예정: 4.0)** 되었고, Spring Framework의 `@MockitoBean`/`@MockitoSpyBean`으로 대체되었습니다.

| 구분 | `@MockBean`(Boot) | `@MockitoBean`(Framework) |
| --- | --- | --- |
| 위치 | 테스트 클래스, `@Configuration` 클래스 | 테스트 클래스의 필드 (클래스 레벨 `types`/`name`으로도 가능) |
| 상태 | 3.4 deprecated, 4.0 제거 예정 | 현재 표준 |
| `@Configuration`에서 공유 | 가능 | 불가 → 테스트 클래스나 합성 애노테이션으로 이동 |

```java
// 여러 테스트에서 같은 목 조합을 공유하고 싶을 때 — 합성 애노테이션
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@MockitoBean(types = {PaymentClient.class, MailClient.class})
public @interface SharedExternalMocks {
}
```

- **목은 경계에만 둡니다.** 외부 PG사·메일·SMS·타 시스템 API는 목이 맞지만, **내부 서비스끼리 목을 씌우면 통합 검증이 사라집니다.** "주문 서비스가 재고 서비스를 호출했는가"를 목으로 검증하면, 정작 재고 차감 로직이 깨져도 통과합니다.
- **목 개수가 늘어나면 설계 신호**입니다. 컨트롤러 테스트 하나에 목 5개가 필요하면 서비스가 비대하다는 뜻이고, 도메인 로직을 꺼내 단위 테스트로 옮기는 편이 낫습니다.
- **`MockitoBean`은 필드 단위라 `@Nested`·상속과 조합할 때 주의**합니다. 상위 클래스 필드의 목을 하위에서 재정의하면 컨텍스트 커스터마이저가 달라져 캐시가 갈립니다.

### 1.8 테스트 데이터 준비 — "고정 픽스처"와 "테스트 간 격리"

| 방식 | 장점 | 주의 |
| --- | --- | --- |
| `@BeforeEach`에서 저장 | 명시적, IDE에서 추적 쉬움 | 테스트가 늘면 중복, 대량 데이터 느림 |
| `@Sql("/sql/members.sql")` | SQL 그대로, 스키마·데이터 분리 | 스크립트와 엔티티 매핑이 어긋날 수 있음 |
| Fixture 객체(빌더) | 가독성, 기본값 관리 | 기본값이 바뀌면 전체 테스트 영향 |
| Flyway + TRUNCATE 정리 | 프로덕션 스키마와 동일 | TRUNCATE 순서(FK) 관리 필요 |

```java
@Test
@Sql(scripts = "/sql/members.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
@Sql(scripts = "/sql/truncate.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
void countByStatus() {
    // members.sql이 넣은 3건 기준
    assertThat(memberRepository.countByStatus(Status.ACTIVE)).isEqualTo(2);
}
```

- **테스트가 서로의 데이터에 의존하면 실행 순서가 바뀌는 순간 깨집니다.** "앞 테스트가 만든 id=1을 뒤 테스트가 쓴다"는 구조를 만들지 않습니다. AUTO_INCREMENT는 롤백해도 되돌아가지 않으므로 **id를 하드코딩하지 않는 편**이 안전합니다.
- **`TRUNCATE`는 FK 제약 때문에 순서를 지켜야** 하고, MySQL에서 `TRUNCATE`는 암묵적 커밋을 일으켜 테스트 트랜잭션 롤백과 충돌합니다. 롤백 트랜잭션을 쓰는 테스트에서는 `@Sql` 대신 엔티티 저장 방식을 권장합니다.

### 1.9 결정성(determinism) — flaky 테스트를 만드는 세 가지

1. **시간**: `LocalDateTime.now()`를 직접 쓰면 자정·월말·서머타임 근처에서 깨집니다. `Clock`을 주입하고 테스트에서 `Clock.fixed(...)`를 씁니다.
2. **난수·UUID**: 검증 대상이 아니면 고정값으로 대체하거나, 검증에서 제외합니다.
3. **비동기**: `@Async`, 이벤트 리스너, 스케줄러, 외부 콜백은 `Thread.sleep()`이 아니라 **Awaitility**로 조건을 기다립니다.

```java
// Clock 주입 — 테스트에서 시간을 고정
@Service
public class CouponService {
    private final Clock clock;

    public CouponService(Clock clock) {
        this.clock = clock;
    }

    public boolean isExpired(Coupon coupon) {
        return coupon.getExpiresAt().isBefore(Instant.now(clock));
    }
}

// 테스트
Clock fixed = Clock.fixed(Instant.parse("2026-11-21T04:00:00Z"), ZoneId.of("Asia/Seoul"));
CouponService service = new CouponService(fixed);
assertThat(service.isExpired(couponExpiringAt("2026-11-21T03:59:00Z"))).isTrue();
```

```java
// Awaitility — 비동기 결과를 조건으로 대기 (고정 sleep 금지)
Awaitility.await()
        .atMost(Duration.ofSeconds(3))
        .pollInterval(Duration.ofMillis(100))
        .untilAsserted(() -> assertThat(orderRepository.findById(orderId))
                .get()
                .extracting(Order::getStatus)
                .isEqualTo(Status.PAID));
```

- **`Thread.sleep(1000)`은 두 방향으로 틀립니다.** 느린 CI에서는 부족해 실패하고, 빠른 로컬에서는 불필요하게 느려집니다. 조건 기반 대기가 유일하게 안정적입니다.
- **`@Transactional` 테스트에서 `@Async` 결과를 기다리면** 별도 스레드가 커밋되지 않은 데이터를 보지 못해 영원히 대기할 수 있습니다. 비동기 검증은 커밋이 실제로 일어나는(비트랜잭션) 테스트에서 합니다.
- **JUnit 5 병렬 실행**(`junit.jupiter.execution.parallel.enabled=true`)은 테스트 시간을 크게 줄이지만, **공유 DB·공유 컨테이너에서 데이터 간섭**을 만듭니다. 병렬화하려면 ① 테스트별 스키마 분리, ② 컨테이너 다중화, ③ 컨텍스트 캐시와 `@DirtiesContext` 조합 점검이 선행되어야 합니다. 슬라이스 단위 테스트부터 병렬화하는 편이 안전합니다.

### 1.10 어디까지 테스트할 것인가 — 실무 기준

- **커버리지 100%는 목표가 아닙니다.** 라인 커버리지보다 중요한 건 "핵심 도메인 규칙이 실패 케이스까지 덮였는가"입니다. `if`의 true 경로만 테스트한 80%는 0%에 가깝습니다.
- **버그가 난 곳에는 반드시 회귀 테스트를 남깁니다.** 이슈 번호를 테스트 이름이나 `@DisplayName`에 남기면("이슈 #142: 재고 0일 때 주문 실패") 6개월 뒤에도 의도가 보입니다.
- **테스트는 리팩터링을 방해하지 않아야 합니다.** 구현 세부(내부 메서드 호출 순서, private 메서드)를 검증하면 좋은 리팩터링이 테스트를 깹니다. 공개 행동과 결과 상태를 검증합니다.

---

## 2. 실무 적용 및 주의사항 (Edge Cases)

### 2.1 계층별 테스트 예시 (전체 흐름)

```java
// 1) 도메인 단위 테스트 — Spring 없음, 가장 빠르고 가장 많이
// src/test/java/com/example/order/domain/OrderTest.java
class OrderTest {

    @Test
    @DisplayName("재고보다 많은 수량을 주문하면 IllegalStateException이 발생한다")
    void create_fails_whenQuantityExceedsStock() {
        assertThatThrownBy(() -> Order.create(1L, 11, 10))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("재고");
    }

    @Test
    @DisplayName("결제가 완료되면 상태가 PAID로 전이한다")
    void markPaid_changesStatusToPaid() {
        Order order = Order.create(1L, 1, 10);

        order.markPaid();

        assertThat(order.getStatus()).isEqualTo(OrderStatus.PAID);
    }
}
```

```java
// 2) 리포지토리 슬라이스 — 실제 MySQL 컨테이너 + Flyway 스키마
// src/test/java/com/example/order/OrderRepositoryTest.java
@RepositoryTest
class OrderRepositoryTest extends AbstractContainerTest {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private TestEntityManager testEntityManager;

    @Test
    @DisplayName("상태별 주문 수를 집계한다")
    void countByStatus_groupsByStatus() {
        testEntityManager.persist(Order.create(1L, 1, 10));

        Order paid = Order.create(2L, 1, 10);
        paid.markPaid();
        testEntityManager.persist(paid);
        testEntityManager.flush();

        assertThat(orderRepository.countByStatus(OrderStatus.PAID)).isEqualTo(1);
    }
}
```

```java
// 3) 컨트롤러 슬라이스 — 검증 실패 → 400, 없는 주문 → 404
@WebMvcTest(OrderController.class)
class OrderControllerValidationTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private OrderService orderService;

    @Test
    @DisplayName("수량이 0이면 400과 필드 오류를 반환한다")
    void create_returns400_whenQuantityIsZero() throws Exception {
        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("""
                                {"productId": 1, "quantity": 0}
                                """))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.errors[0].field").value("quantity"));
    }

    @Test
    @DisplayName("없는 주문을 조회하면 404를 반환한다")
    void get_returns404_whenNotFound() throws Exception {
        given(orderService.findById(99L)).willThrow(new OrderNotFoundException(99L));

        mockMvc.perform(get("/api/orders/{id}", 99L))
                .andExpect(status().isNotFound());
    }
}
```

```java
// 4) 통합 — 실제 커밋까지 검증 (RANDOM_PORT + 실제 DB)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class OrderPlacementIntegrationTest extends AbstractContainerTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private OrderRepository orderRepository;

    @Test
    @DisplayName("주문 생성 후 DB에 실제로 저장되어 있다")
    void place_persistsOrder() {
        // given: 상품·재고는 @BeforeEach 또는 @Sql로 준비 (여기서는 생략)

        ResponseEntity<OrderResponse> response = restTemplate.postForEntity(
                "/api/orders", new OrderRequest(1L, 2), OrderResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);

        Long orderId = response.getBody().id();
        assertThat(orderRepository.findById(orderId)).isPresent();   // 커밋된 데이터
    }
}
```

- **통합 테스트에는 `@Transactional`을 붙이지 않았습니다.** 서버 스레드가 커밋하는데 테스트 트랜잭션으로 롤백을 기대하면 앞뒤가 맞지 않습니다. 대신 `@Sql`/`@BeforeEach`로 데이터를 만들고 `@AfterEach`나 컨테이너 재사용 정책으로 정리합니다.
- **정리 비용이 크면 트랜잭션 롤백 + `RANDOM_PORT`를 포기**하고 MockMvc로 내리는 판단도 맞습니다. 검증 목적(HTTP 계약 vs 데이터 영속)에 따라 고릅니다.

### 2.2 Testcontainers 운영 시 자주 만나는 문제

| 증상 | 원인 | 대응 |
| --- | --- | --- |
| `Could not find a valid Docker environment` | Docker 미실행/소켓 권한 없음 | 데몬 확인, `DOCKER_HOST`, CI 에이전트 권한 |
| 첫 클래스만 성공, 다음 클래스 실패 | 컨테이너 재기동 + 캐시된 컨텍스트의 죽은 포트 | **싱글턴 컨테이너 패턴**(1.6절) |
| 전체 스위트가 5분 이상 | 클래스마다 컨테이너 기동 | 싱글턴 + 이미지 pull 캐시, `@DirtiesContext` 제거 |
| `Access denied` / 스키마 없음 | `withDatabaseName`/계정 불일치, Flyway 미실행 | 접속 정보 단일화, 마이그레이션 실행 확인 |
| 한글 데이터 깨짐 | 컨테이너 기본 콜레이션이 latin1 | `--character-set-server=utf8mb4 --collation-server=utf8mb4_0900_ai_ci` |
| 시간대 때문에 테스트 실패 | 컨테이너 UTC vs 앱 KST | `TZ`, `spring.jackson.time-zone`, DB `time_zone` 일치 |
| CI에서만 느림 | 이미지 캐시 없음 | 레지스트리 미러, 사전 pull, 필요 시 Testcontainers Cloud |

```groovy
// build.gradle — 테스트 태스크 설정
tasks.named('test') {
    useJUnitPlatform()
    maxParallelForks = 1          // 공유 컨테이너에서 병렬 포크는 데이터 간섭 위험
    systemProperty 'spring.profiles.active', 'test'
    testLogging {
        events 'failed', 'skipped'
        exceptionFormat 'full'
    }
    // 테스트 JVM에 컨테이너 재사용을 강제하지 않는다 (CI 오염 방지)
}
```

```groovy
// Jenkins 파이프라인 — Docker가 있는 에이전트에서 실행
stage('Test') {
    steps {
        sh './gradlew test --no-daemon'
    }
    post {
        always {
            junit '**/build/test-results/test/*.xml'
        }
    }
}
```

- **DinD 환경에서 Testcontainers**는 `DOCKER_HOST`가 가리키는 데몬에 접근할 수 있어야 합니다. Jenkins Kubernetes 플러그인이라면 도커 사이드카 또는 호스트 소켓 마운트가 필요하고, 공유 에이전트에서 소켓 마운트는 권한 문제가 될 수 있습니다(11-20 노트의 DinD vs 소켓 마운트 논의와 같은 지점).
- **테스트 리포트를 아티팩트로 남깁니다.** 실패한 테스트의 스택트레이스와 stdout(컨테이너 로그 포함)이 없으면 CI에서만 나는 실패를 재현할 수 없습니다.

### 2.3 `@Transactional` 테스트 판단표

| 검증하려는 것 | `@Transactional` 테스트 | 비트랜잭션 테스트 |
| --- | --- | --- |
| 엔티티 매핑·JPQL/QueryDSL 결과 | ✅ 적합 (빠름, 정리 불필요) | 가능하지만 느림 |
| 유니크 제약·FK 위반 | ⚠️ `flush()`·`TestTransaction` 필요 | ✅ 적합 |
| `@TransactionalEventListener(AFTER_COMMIT)` | ❌ 커밋이 없어 실행 안 됨 | ✅ 필수 |
| 낙관적 락 충돌, 데드락, 락 대기 | ❌ 재현 어려움(별도 트랜잭션 필요) | ✅ 별도 스레드 + 실제 커밋 |
| 스케줄러·배치 1회 실행 | ❌ | ✅ |
| 캐시(Redis) 갱신, 외부 API 호출 | ⚠️ 커밋 후 로직이면 검증 불가 | ✅ |

- **"테스트는 통과하는데 운영에서 터진다"의 단골 원인**이 이 표의 ⚠️/❌ 칸입니다. 특히 `AFTER_COMMIT` 리스너는 테스트가 **검증을 못 한 채 초록불**이 되기 때문에 위험합니다.
- **`@Transactional`을 메서드 단위로 켜고 끄면** 같은 클래스에서도 컨텍스트 캐시 키가 달라질 수 있습니다. 클래스 단위로 통일하고, 커밋 검증이 필요한 메서드는 **별도 테스트 클래스**로 분리합니다.

### 2.4 느린 테스트 스위트 줄이기 — 효과 순서

| 순서 | 조치 | 기대 효과 |
| --- | --- | --- |
| 1 | `@DirtiesContext` 제거·최소화 | 수십 초~수 분 |
| 2 | `@MockitoBean`/프로파일 조합 통일 → 컨텍스트 수 감소 | 수십 초 |
| 3 | Testcontainers 싱글턴화 | 수십 초~수 분 |
| 4 | 통합 테스트를 슬라이스로 내리기(가능한 것만) | 수십 초 |
| 5 | `@SpringBootTest` 대신 도메인 단위 테스트로 이동 | 지속적 |
| 6 | 테스트 병렬화(안전성 확보 후) | CPU 코어 수에 비례 |

```bash
# 컨텍스트 캐시 히트율 확인 — "왜 느린가"를 추정하지 않고 측정
./gradlew test -Dlogging.level.org.springframework.test.context.cache=DEBUG
```

```text
Spring TestContext Cache Hit Ratio: 12 hits / 15 misses
→ 컨텍스트를 15번 띄웠다는 뜻. 설정 통일이 우선.
```

### 2.5 테스트 안티패턴 체크리스트

- [ ] 테스트 이름이 **행동과 기대 결과**를 말하는가 (`test1`, `testSave` 금지)
- [ ] 한 테스트에 **assert가 여러 개념** 섞여 있지 않은가(실패 시 원인 추적 불가)
- [ ] `Thread.sleep()`으로 비동기를 기다리고 있지 않은가 → Awaitility
- [ ] `LocalDateTime.now()`를 직접 호출하지 않는가 → `Clock` 주입
- [ ] 테스트 간 **실행 순서에 의존**하지 않는가
- [ ] `@DirtiesContext`로 문제를 덮고 있지 않은가
- [ ] 내부 서비스끼리 목을 씌워 **통합 검증을 비우지** 않았는가
- [ ] `AFTER_COMMIT`·제약조건·락을 롤백 트랜잭션으로 검증하려 하지 않았는가
- [ ] H2로 통과한 쿼리가 **실제 MySQL에서도** 통과하는가(방언·콜레이션·락)
- [ ] 테스트가 **DB 스키마를 프로덕션과 같은 경로**(Flyway)로 만들고 있는가
- [ ] 실패 메시지에 **기대값과 실제값**이 남는가(AssertJ의 `as(...)` 활용)
- [ ] CI에서 실패한 테스트를 **로컬에서 재현**할 수 있는가(시드·시간·환경 고정)
- [ ] 커버리지를 위해 **의미 없는 게터 테스트**를 만들지 않았는가

### 2.6 검증 시나리오 — 설정이 실제로 의도대로 동작하는지 확인

| 확인하려는 것 | 방법 | 기대 결과 |
| --- | --- | --- |
| `@WebMvcTest`가 서비스 빈을 로드하지 않는가 | `@MockitoBean` 제거 후 실행 | `NoSuchBeanDefinitionException` (의도된 슬라이스) |
| `@DataJpaTest`가 실제 MySQL을 쓰는가 | 테스트 로그의 JDBC URL / `SELECT @@version` | `mysql:8.0` 컨테이너 |
| Testcontainers 1회 기동 | `docker ps` 관찰 + 전체 스위트 시간 | 클래스 수와 무관하게 1개 |
| 컨텍스트 재사용 | `logging.level...cache=DEBUG` | Hit Ratio 상승 |
| `AFTER_COMMIT` 리스너 | 비트랜잭션 테스트 + `TestTransaction` | 이벤트 발행 확인 |
| 유니크 제약 검증 | 같은 값 2건 저장 + `flush()` | `DataIntegrityViolationException` |
| flaky 여부 | 같은 테스트 5회 반복 (`--rerun-tasks`) | 항상 동일 결과 |

---

## 3. 기술 면접 단골 꼬리 질문 & 답변 팁

- **Q. Spring Boot에서 테스트 계층을 어떻게 나누고, 각각 무엇을 검증하나요?**
  - **A.** "세 층으로 나눕니다. 도메인 로직은 Spring 없이 순수 단위 테스트로 검증하고, 웹 계약은 `@WebMvcTest`로 라우팅·직렬화·검증·예외 응답을, 데이터 접근은 `@DataJpaTest`로 매핑과 쿼리를 봅니다. 마지막으로 빈 배선과 트랜잭션 경계, 실제 SQL 방언까지 필요한 경로만 `@SpringBootTest` + Testcontainers로 통합 검증합니다. 위로 갈수록 느리고 실제에 가깝기 때문에, 실패했을 때 원인을 좁힐 수 있도록 아래층을 두껍게 둡니다."

- **Q. `@SpringBootTest`의 `webEnvironment` 차이를 설명해 보세요.**
  - **A.** "기본값 `MOCK`은 서블릿 컨테이너를 띄우지 않고 MockMvc로 `DispatcherServlet`을 직접 호출합니다. 가장 빠르고 필터 체인도 실제로 타지만 진짜 네트워크는 없습니다. `RANDOM_PORT`는 내장 서버를 랜덤 포트로 띄워 실제 HTTP를 태우므로 커넥션 풀·타임아웃·비동기 응답까지 검증되지만 느립니다. 결정적인 차이는 스레드 경계로, `RANDOM_PORT`에서는 서버가 별도 스레드·별도 트랜잭션이라 테스트의 `@Transactional` 롤백이 서버 데이터에 적용되지 않습니다."

- **Q. 테스트에 `@Transactional`을 붙이면 어떤 문제가 있나요?**
  - **A.** "편하지만 검증을 가립니다. 첫째, 제약조건·유니크 충돌·낙관적 락 충돌은 커밋 시점에 발생하는데 롤백하면 드러나지 않습니다. 둘째, `@TransactionalEventListener(AFTER_COMMIT)`는 커밋이 없어 실행되지 않아 이벤트 테스트가 조용히 무의미해집니다. 셋째, 영속성 컨텍스트가 살아 있어 지연 로딩이 항상 성공하므로 운영의 `LazyInitializationException`을 못 잡습니다. 그래서 매핑·쿼리 검증에는 롤백을 쓰고, 커밋 이후 동작은 비트랜잭션 테스트로 분리합니다."

- **Q. 그래서 `@Transactional` 테스트는 언제 쓰고 언제 피하나요?**
  - **A.** "리포지토리 슬라이스처럼 '쿼리 결과와 매핑이 의도대로인가'만 볼 때는 씁니다. 정리 비용이 사라지고 빠릅니다. 반대로 커밋 이후 로직, 락 경합, 데드락, 스케줄러·배치 실행, 외부 시스템 호출은 커밋이 실제로 일어나야 하므로 피합니다. `RANDOM_PORT`로 실제 서버를 띄우는 테스트에도 붙이지 않습니다."

- **Q. 테스트가 갑자기 느려졌습니다. 어떻게 접근하나요?**
  - **A.** "코드보다 컨텍스트 기동 횟수를 먼저 봅니다. `logging.level.org.springframework.test.context.cache=DEBUG`로 히트율을 확인하면 캐시 미스가 몇 번인지 수치로 나옵니다. 미스가 많으면 `@DirtiesContext`, 프로파일, `@MockitoBean` 조합, `@TestPropertySource` 값 차이로 캐시 키가 갈린 것입니다. 설정을 공통 베이스 클래스로 통일하고, Testcontainers는 JVM당 한 번만 뜨도록 싱글턴 패턴으로 바꾸고, 통합 테스트 중 슬라이스로 내릴 수 있는 것을 내립니다."

- **Q. H2로 테스트하는 것과 Testcontainers로 실제 DB를 쓰는 것의 차이는?**
  - **A.** "H2는 MySQL 방언을 흉내만 냅니다. `ON DUPLICATE KEY UPDATE`, 콜레이션, JSON 함수, `SELECT ... FOR UPDATE`의 갭 락, 실행계획 같은 InnoDB 특유의 동작이 다르거나 없습니다. 그래서 인덱스가 실제로 먹는지, 커버링 인덱스가 되는지, 데드락이 나는지는 H2로 검증할 수 없습니다. 통합 테스트에서 Testcontainers로 MySQL 8.0을 띄우고 Flyway까지 돌리면 스키마와 방언을 프로덕션과 같게 맞출 수 있습니다. 대가는 Docker 의존성과 기동 시간입니다."

- **Q. Testcontainers를 쓰면 컨테이너가 클래스마다 뜨는 문제가 있는데 어떻게 해결하나요?**
  - **A.** "`@Container static`은 JUnit 확장이 클래스 단위로 컨테이너를 시작·정지시킵니다. 그런데 Spring 컨텍스트는 캐시되어 다음 클래스에서도 재사용되므로, 재기동된 컨테이너의 새 포트와 캐시된 컨텍스트의 옛 포트가 어긋납니다. 이때는 정적 필드에 컨테이너를 두고 `static` 초기화 블록에서 한 번만 `start()` 하는 싱글턴 패턴을 씁니다. 접속 정보는 `@ServiceConnection`(Boot 3.1+)이나 `@DynamicPropertySource`로 등록합니다."

- **Q. `@ServiceConnection`은 무엇을 해주나요?**
  - **A.** "Testcontainers 컨테이너에서 `ConnectionDetails` 빈을 자동으로 만들어 주는 애노테이션입니다. 기존에는 `@DynamicPropertySource`로 JDBC URL·계정을 손으로 넣었는데, 이제 컨테이너 타입만 보고 MySQL·PostgreSQL·Redis·Kafka 등의 접속 정보를 만들어 `spring.datasource.*` 같은 설정을 덮어씁니다. `spring-boot-testcontainers` 모듈이 필요하고, `GenericContainer`처럼 타입이 없으면 `name` 속성으로 힌트를 줘야 합니다."

- **Q. MockMvc는 실제 서블릿 컨테이너를 띄우나요? 그럼 무엇을 검증하지 못하나요?**
  - **A.** "띄우지 않습니다. `DispatcherServlet`을 직접 호출하고 등록된 필터 체인은 실제로 태웁니다. 그래서 매핑·검증·직렬화·예외 핸들링·시큐리티 인가는 검증됩니다. 반면 실제 커넥션 풀, 네트워크 타임아웃, 서블릿 비동기 처리, 대용량 멀티파트, HTTP/2 같은 것은 검증하지 못하므로 `RANDOM_PORT` 테스트로 내려야 합니다."

- **Q. `@MockBean`과 `@MockitoBean`의 차이는?**
  - **A.** "`@MockBean`은 Spring Boot가 제공하던 애노테이션으로 3.4에서 deprecated 되고 4.0에서 제거 예정이며, Spring Framework의 `@MockitoBean`으로 대체됐습니다. 가장 큰 실무 차이는 `@Configuration` 클래스에서 쓸 수 없다는 점입니다. 여러 테스트가 공유하는 목은 `@MockitoBean(types = ...)`를 붙인 합성 애노테이션이나 테스트 클래스 필드로 옮겨야 합니다."

- **Q. 어디까지 목을 써야 하나요?**
  - **A.** "경계에만 씁니다. PG사·메일·SMS·타 시스템 API처럼 우리가 제어할 수 없거나 호출 비용이 큰 것은 목이 맞습니다. 내부 서비스 사이에 목을 씌우면 '호출했는가'만 검증되어 실제 로직 결함이 통과합니다. 목이 늘어나면 서비스가 비대하다는 신호로 보고, 도메인 로직을 꺼내 단위 테스트로 옮깁니다."

- **Q. flaky 테스트의 원인과 대응은?**
  - **A.** "대표적으로 세 가지입니다. 시간 의존(`now()` 직접 호출)은 `Clock` 주입과 고정 시각으로, 비동기 대기는 고정 `sleep` 대신 Awaitility로, 테스트 간 데이터·순서 의존은 픽스처 격리와 id 하드코딩 제거로 해결합니다. 여기에 병렬 실행과 공유 컨테이너 조합, 컨테이너 시간대(UTC vs KST)도 원인이 됩니다. flaky 테스트는 격리하거나 고치고, 재시도로 덮지 않습니다. 재시도는 실제 버그를 숨깁니다."

- **Q. 테스트 커버리지는 몇 %가 적당한가요?**
  - **A.** "숫자 목표는 부적절합니다. 라인 커버리지는 게터만 호출해도 올라갑니다. 핵심 도메인 규칙과 실패 경로가 덮였는지가 기준이고, 버그가 발생한 곳에는 반드시 회귀 테스트를 남깁니다. 커버리지는 '테스트되지 않은 위험한 코드를 찾는 지표'로만 씁니다."

- **Q. 테스트에서 트랜잭션이나 커밋 시점 이벤트를 어떻게 검증하나요?**
  - **A.** "트랜잭션 롤백 테스트에서는 `TestTransaction.flagForCommit()` + `TestTransaction.end()`로 실제 커밋을 일으켜 `AFTER_COMMIT` 리스너를 실행시키고, 이후 `start()`로 트랜잭션을 다시 엽니다. 그보다 권장하는 방식은 해당 테스트를 비트랜잭션으로 두고 `@Sql`이나 저장 코드로 데이터를 준비한 뒤 실제 커밋 경로를 그대로 통과시키는 것입니다. 검증 후 정리는 TRUNCATE나 컨테이너 재사용 정책으로 처리합니다."

- **Q. 통합 테스트가 10분 걸립니다. 면접관이 '그래도 필요하다'고 하면 어떻게 답하나요?**
  - **A.** "필요한 통합 테스트는 남기되, 그것이 유일한 안전망이 되지 않게 층을 나눕니다. 핵심 시나리오 5~10개만 `@SpringBootTest`로 유지하고, 나머지 경로는 슬라이스와 단위 테스트로 검증합니다. 동시에 컨텍스트 캐시 히트율과 컨테이너 기동 횟수를 측정해 불필요한 재기동을 제거하면 같은 커버리지에서 시간을 크게 줄일 수 있습니다. 테스트 시간도 관리 대상 지표로 두고 CI에서 추적합니다."

---

## 참고

- Spring Boot — Testing Spring Boot Applications: https://docs.spring.io/spring-boot/reference/testing/spring-boot-applications.html
- Spring Boot — Testcontainers / Service Connections: https://docs.spring.io/spring-boot/reference/testing/testcontainers.html
- Spring Boot — Development-time services (Docker Compose, Testcontainers): https://docs.spring.io/spring-boot/reference/features/dev-services.html
- Spring Boot API — `@DataJpaTest`: https://docs.spring.io/spring-boot/api/java/org/springframework/boot/test/autoconfigure/orm/jpa/DataJpaTest.html
- Spring Boot API — `@AutoConfigureTestDatabase`: https://docs.spring.io/spring-boot/api/java/org/springframework/boot/test/autoconfigure/jdbc/AutoConfigureTestDatabase.html
- Spring Boot 3.4 Release Notes — `@MockBean`/`@SpyBean` deprecation: https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.4-Release-Notes
- Spring Framework — `@MockitoBean` / `@MockitoSpyBean`: https://docs.spring.io/spring-framework/reference/testing/annotations/integration-spring/annotation-mockitobean.html
- Spring Framework — Transaction Management in Tests (`TestTransaction`): https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/tx.html
- Spring Framework — Context Caching: https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/ctx-management/caching.html
- Testcontainers for Java — Reuse: https://java.testcontainers.org/features/reuse/
- Testcontainers for Java — JUnit 5 integration: https://java.testcontainers.org/test_framework_integration/junit_5/
- JUnit 5 User Guide — Parallel Execution: https://junit.org/junit5/docs/current/user-guide/#writing-tests-parallel-execution
- Awaitility: https://github.com/awaitility/awaitility
- AssertJ — Soft assertions / description: https://assertj.github.io/doc/
