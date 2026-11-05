# [2026-11-05] JPA N+1 문제 완전 정복과 컬렉션 페이징 최적화 (Fetch Join, @EntityGraph, default_batch_fetch_size 비교)

> **한 줄 핵심 요약**: JPA N+1은 LAZY 로딩된 연관 엔티티를 루프에서 실제 사용하는 시점에 1차 캐시를 미확인하고 추가 SELECT를 N번 날리는 문제로, `fetch join`·`@EntityGraph`·`default_batch_fetch_size` 세 가지 해결책이 각각 페이징 호환성과 쿼리 형태가 다르므로, 특히 컬렉션(`@OneToMany`) + `Pageable` 조합에서는 페치 조인 페이징이 인메모리 페이징(OOM 위험)으로 빠지므로 배치 사이즈가 표준 해법이다.

---

## 0. 코드 예제 (전체 실행 가능 버전)

```java
// build.gradle
// dependencies {
//     implementation 'org.springframework.boot:spring-boot-starter-data-jpa:3.3.x'
//     implementation 'org.springframework.boot:spring-boot-starter-web:3.3.x'
//     runtimeOnly   'com.h2database:h2'
//     testImplementation 'org.springframework.boot:spring-boot-starter-test'
// }
//
// src/main/resources/application.yml
// spring:
//   jpa:
//     properties:
//       hibernate:
//         default_batch_fetch_size: 100   # 전역 배치 사이즈. 실무 표준값 100~1000
//     open-in-view: false                 # OSIV 비활성 권장 (트랜잭션 밖 LAZY 로딩 방지)

// 파일 경로: src/main/java/study/nplus1/domain/Team.java
package study.nplus1.domain;

import jakarta.persistence.*;
import java.util.ArrayList;
import java.util.List;

@Entity
public class Team {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // 기본 전략은 LAZY. (FetchType.EAGER는 실무에서 금지 — 이유는 2-2 참고)
    @OneToMany(mappedBy = "team", fetch = FetchType.LAZY)
    private List<Member> members = new ArrayList<>();

    protected Team() {}

    public Team(String name) { this.name = name; }

    public List<Member> getMembers() { return members; }
    public Long getId() { return id; }
}
```

```java
// 파일 경로: src/main/java/study/nplus1/domain/Member.java
package study.nplus1.domain;

import jakarta.persistence.*;

@Entity
public class Member {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String username;

    @ManyToOne(fetch = FetchType.LAZY)  // 다(N) 쪽도 반드시 LAZY 기본
    @JoinColumn(name = "team_id")
    private Team team;

    protected Member() {}

    public Member(String username, Team team) {
        this.username = username;
        this.team = team;
    }
}
```

```java
// 파일 경로: src/main/java/study/nplus1/repository/TeamRepository.java
package study.nplus1.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import study.nplus1.domain.Team;

import java.util.List;

public interface TeamRepository extends JpaRepository<Team, Long> {

    // ── 문제 코드: 일반 findAll ─────────────────────────────
    // 1) SELECT ... FROM team      (1번)
    // 2) 루프에서 team.getMembers() 호출 시마다
    //    SELECT ... FROM member WHERE team_id = ?  (N번, 팀마다 1번)
    List<Team> findAllBasic();  // = 상속받은 findAll()


    // ── 옵션 1: JPQL fetch join ─────────────────────────────
    // SELECT t FROM Team t JOIN FETCH t.members  → 조인 결과로 즉시 초기화
    // 단, 컬렉션 대상 페이징(Pageable)과 함께 쓰면 안 됨 (2-1 참고)
    @Query("select t from Team t join fetch t.members")
    List<Team> findAllWithMembersFetchJoin();


    // ── 옵션 2: @EntityGraph ────────────────────────────────
    // 페치 조인과 동일한 JOIN 쿼리가 나가지만, JPQL 없이 속성명만으로 선언.
    // querydsl/동적 쿼리와 섞일 때는 결국 fetch join으로 회귀한다.
    @EntityGraph(attributePaths = {"members"})
    List<Team> findAllWithEntityGraph();


    // ── 옵션 3: 배치 사이즈(IN 쿼리) — 컬렉션 페이징 표준 해법 ──
    // 별도 코드 없이 application.yml의 default_batch_fetch_size만으로 동작:
    // 1) SELECT ... FROM team LIMIT/OFFSET (페이징, 1번)
    // 2) SELECT ... FROM member WHERE team_id IN (?, ?, ..., ?)  (ceil(N/batch)번)
    // → 페이지에 로드된 팀 id만 모아서 IN으로 묶어 조회하므로 페이징과 안전하게 공존
    @Query("select t from Team t order by t.id desc")
    List<Team> findPage(@Param("pageable") org.springframework.data.domain.Pageable pageable);
}
```

```java
// 파일 경로: src/main/java/study/nplus1/service/TeamService.java
package study.nplus1.service;

import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import study.nplus1.domain.Team;
import study.nplus1.repository.TeamRepository;

import java.util.List;

@Service
public class TeamService {

    private final TeamRepository teamRepository;

    public TeamService(TeamRepository teamRepository) { this.teamRepository = teamRepository; }

    // N+1 재현용 (테스트/로깅 검증용)
    @Transactional(readOnly = true)
    public void reproduceNPlusOne() {
        List<Team> teams = teamRepository.findAll();
        for (Team team : teams) {
            team.getMembers().size();  // 이 시점마다 member SELECT 1번 발생
        }
    }

    // 컬렉션 페이징 표준 패턴: 페이징은 부모만, 자식은 IN 쿼리 배치 로딩
    @Transactional(readOnly = true)
    public List<Team> getTeamPageWithMembers(int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        List<Team> teams = teamRepository.findPage(pageable);
        for (Team team : teams) {
            team.getMembers().size();  // team_id IN (...) 쿼리로 묶여서 나감
        }
        return teams;
    }
}
```

```java
// 파일 경로: src/test/java/study/nplus1/NPlusOneCountTest.java
package study.nplus1;

import jakarta.persistence.EntityManager;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.annotation.Transactional;
import study.nplus1.domain.Member;
import study.nplus1.domain.Team;
import study.nplus1.repository.TeamRepository;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Transactional
class NPlusOneCountTest {

    @Autowired TeamRepository teamRepository;
    @Autowired EntityManager em;

    @Test
    void 배치사이즈_적용시_쿼리수_검증() {
        // given: 팀 3개, 팀당 멤버 10명
        for (int t = 0; t < 3; t++) {
            Team team = new Team("team-" + t);
            for (int m = 0; m < 10; m++) {
                team.getMembers().add(new Member("member-" + t + "-" + m, team));
            }
            teamRepository.save(team);
        }
        em.flush();
        em.clear();  // 1차 캐시 비우기 필수: 영속 컨텍스트에 남아 있으면 쿼리 카운트가 왜곡됨

        // when: 페이징 조회 + 자식 접근
        var page = teamRepository.findPage(org.springframework.data.domain.PageRequest.of(0, 3));
        page.forEach(team -> team.getMembers().size());

        // then: 기대 쿼리 = team SELECT 1 + member IN 배치 SELECT 1 = 총 2
        // (default_batch_fetch_size=100 기준, N=3이므로 1번의 IN 쿼리로 커버)
        org.hibernate.stat.Statistics stats = em.getEntityManagerFactory()
                .unwrap(org.hibernate.stat.Statistics.class);
        assertThat(stats.getPrepareStatementCount()).isEqualTo(2L);
    }
}
```

---

## 1. 개념 및 동작 원리

- **N+1이 발생하는 구조**
  - JPQL(`findAll` 등)은 엔티티 조회 시 연관 객체를 같이 가져오지 않는다. 글로벌 페치 전략이 `LAZY`면 연관 필드는 **프록시**로만 채워진다.
  - 이후 `team.getMembers().size()`처럼 실제로 필드를 **사용하는 시점**에야 로딩이 시작된다. 이때 하이버네이트는 영속 컨텍스트(1차 캐시)에서 엔티티 id를 키로 찾지만, JPQL로 조회한 부모 목록에는 **자식 컬렉션이 비어 있는 상태**로 로딩되어 있으므로 캐시를 미확인(미적중)하고 DB로 나간다.
  - 결과: 부모 1번 SELECT + 자식 N번 SELECT = **1 + N**. 루프 안에서 발생하기 때문에 부모 건수가 늘수록 쿼리 수가 정직하게 곱해지고, 커넥션 점유 시간과 응답 지연이 선형적으로 증가한다.
- **왜 `LAZY`인데도 발생하는가 (EAGER가 더 위험한 이유)**
  - `EAGER`로 바꾸면 조회 시점에 무조건 조인/추가 SELECT가 붙어 N+1이 "사라진 것처럼" 보이지만, JPA 표준은 EAGER 연관의 로딩 방식(조인 vs 즉시 별도 SELECT)을 보장하지 않아 하이버네이트는 기본적으로 **별도 SELECT**로 푼다. 즉 N+1을 몇 번이나 반복할지 예측하기 어려워질 뿐이다. 그래서 **전부 LAZY 기본 + 필요한 화면에서만 페치 전략을 명시**하는 것이 실무 표준이다.
- **세 가지 해결책의 쿼리 형태 비교**
  - `fetch join`: 부모와 자식을 **하나의 JOIN SELECT**로 즉시 로딩. 쿼리 1번. 단, 컬렉션 조인이므로 결과 행이 자식 수만큼 뻥튬(카테시안 곱)된다.
  - `@EntityGraph`: 페치 조인과 **동일한 LEFT JOIN 쿼리**를 JPQL 없이 선언만으로 생성. 다만 중첩/동적 제어에 한계가 있어 복잡한 조건이 붙으면 fetch join으로 회귀한다.
  - `default_batch_fetch_size`: 부모 조회(필요시 페이징) 후 자식을 `WHERE team_id IN (?, ..., ?)` **배치 IN 쿼리**로 묶어 조회. 쿼리 1 + ceil(N/batch)번. 카테시안 곱이 없으므로 페이징과 안전하게 공존한다.

---

## 2. 실무 적용 및 주의사항 (Edge Cases & 동시성)

### 2-1. 치명적 함정: 컬렉션 페치 조인 + Pageable = 인메모리 페이징 (HHH000104, OOM 위험)

- **현상**: `@OneToMany` 컬렉션에 `join fetch`를 걸고 `Pageable`로 페이징하면, 하이버네이트는 SQL 레벨 LIMIT/OFFSET 대신 **애플리케이션 메모리에 전체 조인 결과를 다 읽어서 직접 자르는 인메모리 페이징**을 수행한다. 경고 로그:
  ```
  HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory!
  ```
  (하이버네이트 6.x에서 `HHH000104` 관련 처리가 `HHH90003004`로 표기된다. 의미는 동일.)
- **왜 위험한가**: 페이지당 20건을 요청해도 실제로는 **조건에 맞는 전체 데이터를 메모리에 적재**한다. 데이터 수십만 건이면 그대로 OOM. 또한 SQL LIMIT가 아니므로 인덱스로 튜닝한 페이징 효과가 완전히 사라진다.
- **왜 이런가**: 페치 조인 결과는 (부모 × 자식) 행이 늘어난 카테시안 곱이라, SQL에서 `LIMIT 20`을 걸면 부모 20건이 아니라 "조인 행 20개"만 잘려 부모 일부가 누락되는 데이터 손상이 생긴다. 하이버네이트가 이를 막기 위해 메모리에서 잘라서라도 정합성을 지키는 설계다.
- **표준 해법 (AGZAK에서도 그대로 적용할 패턴)**
  1. 페이징은 **부모 단독 쿼리**(페치 조인 없이, `order by` 필수)로 수행
  2. 자식 컬렉션은 `default_batch_fetch_size: 100`으로 **IN 쿼리 배치 로딩**
  ```yaml
  # src/main/resources/application.yml
  spring:
    jpa:
      properties:
        hibernate:
          default_batch_fetch_size: 100   # 100~1000 권장. 너무 크면 IN 파라미터 과다(오라클 1000 제한 등)
  ```
  3. 부모가 한 페이지에 최대 size건이므로 IN 쿼리는 **1번**만 추가로 나간다. 총 쿼리 2번으로 페이징 + N+1 해결 동시 달성.
- **대안**: DTO 프로젝션 + 컬렉션은 별도 IN 쿼리로 조립하거나(쿼리 2번, 명시적 제어), Spring Data 표준 밖이지만 하이버네이트 `@BatchSize`로 엔티티 단위 제어도 가능.

### 2-2. FetchType.EAGER를 실무에서 금지하는 이유

- **예측 불가능성**: 위에서 본 대로, EAGER 연관 조회 시 하이버네이트가 조인을 쓸지 별도 SELECT를 쓸지 보장이 없어 **같은 코드에서도 엔티티 로딩 경로에 따라 쿼리 수가 달라진다**.
- **전파**: EAGER는 자신을 로딩하는 **모든 경로**에서 강제로 조인/추가 SELECT를 유발한다. "이 화면에선 필요 없는데도" 무조건 로딩되므로 불필요한 쿼리가 상수비용으로 붙는다.
- **JPQL에서의 함정**: JPQL은 엔티티 단위로 파싱되므로, `select m from Member m` 실행 후 영속 컨텍스트에 올라온 연관 엔티티의 EAGER 로딩이 **추가 N+1을 재발**시킨다 (em.find와 달리 JPQL은 글로벌 페치 전략을 무시하지 못함).
- **결론**: 전부 `LAZY`로 두고, 화면/유즈케이스 단위로 fetch join·EntityGraph·배치사이즈 중 하나를 **명시적으로 선택**한다.

### 2-3. 기타 엣지 케이스

- **`MultipleBagFetchException`**: `Bag`(리스트, 중복 허용, 순서 없음) 컬렉션 2개 이상을 동시에 fetch join하면 결과가 (부모 × 자식1 × 자식2) 카테시안 곱이 되어 하이버네이트가 바로 예외를 던진다. 해결: (a) 컬렉션을 `List` → `Set`으로 바꿔 순서 보장을 포기하거나, (b) **fetch join을 쿼리 2번으로 분리**(두 번째 쿼리는 1차 캐시에 이미 로딩된 부모와 자식을 매핑하므로 N+1 없음), (c) 배치 사이즈로 처리.
- **1차 캐시 미확인과 `em.clear()`**: 테스트에서 N+1을 재현/검증할 때는 `em.flush()` + `em.clear()`를 먼저 해야 한다. 영속 컨텍스트에 이미 로딩된 엔티티가 남아 있으면 쿼리가 아예 나가지 않아 카운트 검증이 무의미해진다.
- **OSIV와의 조합**: `spring.jpa.open-in-view: true`(기본값)면 컨트롤러/뷰 렌더링까지 LAZY 로딩이 살아 있어, 서비스에서 해결했다고 생각한 N+1이 **뷰 템플릿에서 몰래 재발**한다. OSIV를 끄고 서비스/리포지토리 경계 안에서 페치 전략을 명시하는 것이 안전하다.
- **PagingPerformance**: 페치 조인을 쓰더라도 `count` 쿼리는 별도로 나가므로, `Page<T>` 반환 시 count 쿼리 비용도 함께 고려해 `Slice<T>`나 별도 count로 최적화한다.

### 2-4. 아그작(AGZAK) 관점 메모

- 아그작은 RAG 검색 결과 목록과 랭킹(ZSET → 엔티티 역조회)에서 **부모 목록 + 자식 컬렉션** 구조가 반복될 가능성이 높다. Redis에서 id 목록을 받아 JPA로 역조회하는 경로라면, id IN 쿼리 후 자식 컬렉션 접근 시 N+1이 나기 쉬우므로 `default_batch_fetch_size`를 전역 100으로 걸어두고 리뷰 시 `HHH90003004` 로그를 체크리스트에 넣어두면 좋다.
- QueryDSL을 쓰는 순간 `@EntityGraph`보다 `fetchJoin()` 명시 쪽이 직관적이므로, "EntityGraph는 단순 리포지토리, fetch join은 동적 쿼리"로 사용 기준을 고정해두면 코드 일관성이 유지된다.

---

## 3. 기술 면접 단골 꼬리 질문 & 모범 답변 팁

**Q. 페치 조인(fetch join)과 일반 조인(JOIN)의 차이는 무엇인가요?**
- **A.** 일반 조인은 WHERE 절 필터링용으로 연관 엔티티를 **영속화하지 않고** 버리고, SELECT 절에는 부모만 담아 반환합니다. 페치 조인은 조인으로 가져온 연관 엔티티(혹은 컬렉션)를 **즉시 로딩해 영속 컨텍스트에 함께 담아** 반환합니다. 즉, 페치 조인만이 연관 객체를 프록시가 아닌 실제 객체로 초기화합니다.

**Q. 둘 이상의 컬렉션을 동시에 fetch join하면 왜 안 되나요?**
- **A.** 하이버네이트는 `List`(Bag) 타입 컬렉션 여러 개를 동시에 페치 조인하면 (부모 × 자식1 × 자식2) 카테시안 곱으로 결과 정합성과 메모리가 모두 위험해지므로 `MultipleBagFetchException`을 던집니다. 해법은 (1) 컬렉션 타입을 Set으로 변경, (2) 페치 조인을 쿼리 2번으로 분리해 두 번째 쿼리는 1차 캐시에 매핑, (3) 배치 사이즈로 각각 IN 쿼리 조회하는 방식입니다. 실무에서는 2번(쿼리 분리) 또는 3번이 데이터 정합성 관점에서 안전합니다.

**Q. FetchType.EAGER를 실무에서 금지하는 이유는 무엇인가요?**
- **A.** 세 가지입니다. (1) 연관 로딩 방식(조인 vs 별도 SELECT)이 JPA 표준에 보장되지 않아 쿼리 수 예측이 어렵고, (2) 사용하지 않는 화면에서도 항상 강제 로딩되어 상수비용이 붙으며, (3) JPQL 실행 시 영속 컨텍스트에 올라온 EAGER 연관이 추가 SELECT를 재발시켜 오히려 N+1을 숨기고 악화시킵니다. 그래서 전부 LAZY로 두고 필요한 곳에서만 페치 전략을 명시합니다.

**Q. 컬렉션 페치 조인과 페이징을 같이 쓰면 어떻게 되나요?**
- **A.** 하이버네이트가 SQL LIMIT/OFFSET 대신 **인메모리 페이징**(`HHH90003004`/`HHH000104` 경고)으로 전환해, 요청 페이지 크기와 무관하게 조건 전체를 메모리에 적재합니다. 대용량 테이블에서 OOM 직전 상황이 만들어집니다. 표준 해법은 부모만 페이징 조회하고, 자식은 `default_batch_fetch_size`의 IN 쿼리로 배치 로딩하는 것입니다.

**Q. `default_batch_fetch_size`는 어떤 기준으로 정하나요?**
- **A.** 한 화면(한 페이지)에서 기대하는 부모 건수와 자식 건수를 고려해 정하고, 실무에서는 보통 100~1000으로 설정합니다. 너무 크면 IN 파라미터 수가 DB 한계(예: Oracle 1000)를 넘거나 실행 계획이 나빠질 수 있고, 너무 작으면 ceil(N/batch)번 쿼리가 추가로 나갑니다. 페이지 크기 × 2~3 수준에서 시작해 슬로우 쿼리 로그로 조정합니다.

---

## 4. 오늘의 체크리스트 (내일 이어서)

- [ ] 아그작 코드베이스에서 `@OneToMany` + `Pageable` 조합 grep으로 점검, `HHH90003004` 경고 발생 여부 확인
- [ ] QueryDSL로 컬렉션 페이징 표준 패턴(부모 페이징 + 자식 IN 배치) 실습 리포지토리 작성
- [ ] `MultipleBagFetchException` 재현 → Set 변경 / 쿼리 2번 분리 / 배치 사이즈 3가지 해법 비교 데모
- [ ] OSIV를 끈 상태에서 뷰 렌더링 중 LAZY 초기화 예외(`LazyInitializationException`)가 나는 케이스 정리
