# 면접 답변 가이드 — JUnit5 + Mockito 테스트 (본인 코드 기준)

> 작성: 2026-05-09 / 검증 기준: `04_agzak/.../test/.../CouponServiceTest.java` 외 4개 파일
> 대상: 단위 테스트·통합 테스트·동시성 테스트 깊이 질문 대응
> 시리즈: [interview_auth_4projects.md](./interview_auth_4projects.md) · [interview_ranking_idempotency_agzak.md](./interview_ranking_idempotency_agzak.md)

---

## 0. 본인 코드에 들어 있는 패턴 한눈에

| 패턴 | 본인 코드 위치 | 라인 |
|---|---|---|
| `@ExtendWith(MockitoExtension.class)` | `CouponServiceTest.java` | 38 |
| `@Mock` + `@InjectMocks` | 위 파일 | 41-58 |
| `when().thenReturn()` 스터빙 | 위 파일 `createCoupon_정상_생성` | 67-70 |
| `ArgumentCaptor` 인자 캡처 | 위 파일 `createCoupon_정상_생성` | 76-82 |
| `verify()` 호출 검증 | 위 파일 전반 | 77, 96, 149 |
| `verify(..., never())` | 위 파일 `issueCoupon_수량초과시` | 191 |
| `verifyNoInteractions()` | 위 파일 `issueCoupon_존재하지_않는_쿠폰` | 210 |
| `assertThrows()` (JUnit5) | 위 파일 `expireCoupon_존재하지_않는_쿠폰` | 106 |
| `assertThat()` (AssertJ) | 위 파일 전반 | 156, 187 |
| `mock(Class.class)` 인라인 | 위 파일 `expireCoupon_정상_만료` | 89 |
| 동시성 테스트 (CountDownLatch + ExecutorService) | 위 파일 `issueCoupon_동시에...` | 216-263 |
| `@SpringBootTest + @ActiveProfiles("test")` | `CouponServiceAutowiredTest.java` | 30-31 (주석) |
| `@BeforeEach` 픽스처 셋업 | `CookieServiceTest.java` | 54-125 (주석) |
| `@DisplayName` 한글 라벨 | `CookieServiceTest.java` | 128, 140 (주석) |
| `lenient().when()` strict 우회 | `CookieServiceTest.java` | 64 (주석) |
| BDD `given().willReturn()` | `CookieServiceTest.java` | 169-174 (주석) |
| `assertThatThrownBy()` 예외 검증 | `CookieServiceTest.java` | 199 (주석) |

> **사실 인지**: `Autowired`·`CookieService` 테스트 2개는 주석 처리된 상태입니다. 면접에서 *"이 코드 어떻게 동작했나요?"* 물어보면 *"통합 테스트 + 동시성 검증을 짜뒀다가 H2 연동 충돌과 운영 일정상 단위 테스트만 활성화하고 통합 테스트는 주석으로 보관 중"* 정도로 정직하게 답하는 게 안전합니다.

---

## 1. JUnit5 + Mockito 통합의 진입점

```java
@ExtendWith(MockitoExtension.class)   // ★ JUnit5에 Mockito를 연결
class CouponServiceTest {
    @Mock private CouponRepository couponRepository;
    @Mock private CouponTypeRepository couponTypeRepository;
    @InjectMocks private CouponService couponService;
    // ...
}
```

### 핵심 개념 3가지

| 개념 | 의미 | 본인 코드 |
|---|---|---|
| `@ExtendWith` | JUnit5의 **확장 메커니즘**. 외부 라이브러리를 테스트 생명주기에 끼워 넣음 | `MockitoExtension.class` 부착 |
| `@Mock` | 해당 필드를 **mock 인스턴스**로 만듦 (실제 구현체 아님) | `CouponRepository couponRepository` |
| `@InjectMocks` | mock 인스턴스들을 **생성자/세터/필드 주입**으로 SUT에 자동 주입 | `CouponService couponService` |

> **SUT** = System Under Test = 지금 테스트하려는 진짜 객체. 그 외 의존성은 모두 mock으로 격리.

### JUnit4 vs JUnit5 차이 (면접 단골)

| JUnit4 | JUnit5 |
|---|---|
| `@RunWith(MockitoJUnitRunner.class)` | `@ExtendWith(MockitoExtension.class)` |
| `@Before` / `@After` | `@BeforeEach` / `@AfterEach` |
| `@BeforeClass` (static) | `@BeforeAll` (static or non-static with `@TestInstance(PER_CLASS)`) |
| `@Ignore` | `@Disabled` |
| 예외 검증: `@Test(expected = ...)` | `assertThrows(...)` 람다 |
| 단일 Runner만 가능 | **여러 Extension 동시 부착 가능** (예: Mockito + Spring) |

JUnit5의 가장 큰 진전은 **Extension의 합성 가능성**입니다. 한 클래스에 Mockito + Spring + ParameterResolver를 동시에 걸 수 있어, JUnit4의 "Runner는 하나만" 제약이 사라졌습니다.

---

## 2. Mockito 5단 패턴 — 본인 코드로 따라가기

### 패턴 1: **스터빙(Stubbing)** — *"이 메서드 호출되면 이 값 반환해줘"*

```java
// CouponServiceTest.java:67-70
when(createCouponRequest.couponTypeId()).thenReturn(couponTypeId);
when(createCouponRequest.description()).thenReturn(description);
when(createCouponRequest.totalQuantity()).thenReturn(totalQuantity);
when(couponTypeRepository.getReferenceById(couponTypeId)).thenReturn(couponType);
```

문법: `when(mock.method(args)).thenReturn(value);`

핵심: **mock 객체는 기본적으로 모든 메서드가 null/0/false/empty를 반환**합니다. *"이 호출에는 이 값을"* 라고 명시적으로 알려주는 게 스터빙.

### 패턴 2: **검증(Verify)** — *"이 메서드가 (몇 번) 호출됐는지 확인"*

```java
// CouponServiceTest.java:77
verify(couponRepository).save(captor.capture());

// CouponServiceTest.java:191
verify(memberCouponRepository, never()).save(any());

// CouponServiceTest.java:210
verifyNoInteractions(fcsUtil);

// CouponServiceTest.java:261
verify(customCouponRepository, times(count)).updateCouponQuantity(couponId);
```

| 변형 | 의미 |
|---|---|
| `verify(mock).method(...)` | 정확히 1번 호출됐는지 |
| `verify(mock, times(N)).method(...)` | N번 호출됐는지 |
| `verify(mock, never()).method(...)` | **호출되지 않았는지** ★ 부정 케이스 검증의 핵심 |
| `verifyNoInteractions(mock)` | 그 mock의 어떤 메서드도 호출 안 됐는지 |
| `verify(mock, atLeast(N))` / `atMost(N)` | 범위 검증 |

> **`never()`가 왜 중요한가**: *"예외 발생 시 후속 작업이 실행되지 않았다"* 를 검증하는 게 견고한 테스트입니다. 본인 `issueCoupon_존재하지_않는_쿠폰` 테스트가 정확히 이 패턴 — 쿠폰이 없으면 fcsUtil, customCouponRepository, memberCouponRepository 모두 손대지 않았음을 확인.

### 패턴 3: **ArgumentCaptor** — *"메서드에 어떤 인자가 넘어갔는지 캡처해서 검증"*

본인 `createCoupon_정상_생성`의 명장면:

```java
// CouponServiceTest.java:76-82
ArgumentCaptor<Coupon> captor = ArgumentCaptor.forClass(Coupon.class);
verify(couponRepository).save(captor.capture());

Coupon saved = captor.getValue();
assertEquals(description, saved.getDescription());
assertEquals(totalQuantity, saved.getTotalQuantity());
assertEquals(couponType, saved.getCouponType());
```

**왜 필요한가**: `couponService.createCoupon(request)`가 내부에서 `new Coupon(...)`을 만들어 `couponRepository.save(coupon)`을 호출합니다. 이 *"내부에서 만들어진 Coupon"* 객체에 직접 접근할 방법이 없는데, ArgumentCaptor가 그걸 잡아냅니다.

> **테스트 깊이 차이**: `verify(couponRepository).save(any(Coupon.class))` 만으로는 *"뭔가 Coupon이 저장됐다"* 까지만 검증되고, **무엇이 저장됐는지**는 알 수 없습니다. ArgumentCaptor를 써야 "올바른 필드 값으로 저장됐는지"까지 검증할 수 있습니다.

### 패턴 4: **`thenAnswer()` — 동적 응답**

```java
// CouponServiceTest.java:238-239
when(memberCouponRepository.save(any(MemberCoupon.class)))
        .thenAnswer(inv -> inv.getArgument(0));
```

*"save에 들어온 인자를 그대로 반환해줘"* — DB가 ID를 채워서 반환하는 상황을 mock으로 흉내내야 할 때 자주 씁니다. `thenReturn`은 고정값, `thenAnswer`는 **들어온 인자에 따라 다른 응답**.

### 패턴 5: **예외 스터빙**

본인 코드엔 없지만 자주 쓰는 패턴:

```java
when(repo.findById(anyLong())).thenThrow(new EntityNotFoundException("not found"));
// 또는
doThrow(new RuntimeException()).when(mock).voidMethod();
```

`void` 메서드는 `when().thenThrow()`가 안 되고 `doThrow().when()` 패턴을 써야 합니다 — 문법적 함정.

---

## 3. AAA 패턴 (Given-When-Then) — 본인이 일관되게 쓴 구조

본인 모든 테스트가 `// given / // when / // then` 주석으로 구간을 나누고 있습니다.

```java
// CouponServiceTest.java:166-192
@Test
void issueCoupon_수량초과시_실패응답을_반환한다() {
    // given  ← 사전 조건 셋업
    Long couponId = 1L;
    Coupon coupon = mock(Coupon.class);
    when(couponRepository.findById(couponId)).thenReturn(Optional.of(coupon));
    when(coupon.getExpiresAt()).thenReturn(null);
    when(fcsUtil.nextOrder(couponId)).thenReturn(101L);
    when(customCouponRepository.updateCouponQuantity(couponId)).thenReturn(0L);

    // when  ← SUT 실행 (한 줄이 이상적)
    CouponIssueResultResponse res = couponService.issueCoupon(couponId, memberId);

    // then  ← 결과·부수효과 검증
    assertThat(res.isSuccess()).isFalse();
    assertThat(res.order()).isEqualTo(101L);
    verify(memberCouponRepository, never()).save(any());
}
```

| 구간 | 책임 | 본인 코드 |
|---|---|---|
| **Given** | 입력값, mock 스터빙, 픽스처 준비 | `when(...).thenReturn(...)` |
| **When** | **SUT의 단 하나의 행위만 실행** | `couponService.issueCoupon(...)` |
| **Then** | 반환값 + mock 호출 + 상태 변화 검증 | `assertThat(...)`, `verify(...)` |

> **이상적인 테스트**: When이 한 줄이고, Then이 그 한 줄의 결과를 다양한 각도에서 검증합니다. When이 길어지면 *"여러 동작을 한 테스트에 묶고 있다"* 는 신호.

---

## 4. 예외 검증 — JUnit5 vs AssertJ

본인 코드에 두 스타일이 모두 등장합니다.

### JUnit5 `assertThrows` (본인이 주로 쓰는 스타일)

```java
// CouponServiceTest.java:106-110
IllegalArgumentException e = assertThrows(
        IllegalArgumentException.class,
        () -> couponService.expireCoupon(couponId)
);
assertEquals("존재하지 않는 쿠폰입니다.", e.getMessage());
```

- 예외 타입 + 람다로 트리거
- 반환된 예외 객체를 받아서 메시지·필드 추가 검증 가능

### AssertJ `assertThatThrownBy` (Cookie 테스트 주석에 등장)

```java
// CookieServiceTest.java:213-215 (주석)
assertThatThrownBy(() -> cookieService.useCookie(1L, 404L))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("유효하지 않은 쿠키 아이디");
```

- 체이닝 가독성이 좋음
- `hasMessageContaining`, `hasCauseInstanceOf` 등 풍부한 검증 메서드

> **권장**: 한 프로젝트 안에서는 하나로 통일하세요. 본인은 `assertThrows`가 우세하므로 그 스타일을 일관 유지하면서, 메시지가 길거나 cause 체인이 있을 때만 `assertThatThrownBy` 쓰는 게 자연스럽습니다.

---

## 5. 단위 테스트 vs 통합 테스트 — 본인 코드의 두 분기

### 단위 테스트 (`CouponServiceTest.java`) — Mockito 단독

```java
@ExtendWith(MockitoExtension.class)
class CouponServiceTest {
    @Mock private CouponRepository couponRepository;
    @InjectMocks private CouponService couponService;
}
```

- **Spring 컨텍스트 안 띄움** → 빠름 (수십 ms)
- DB·Redis·외부 시스템 모두 mock으로 대체
- 비즈니스 로직 분기만 정밀하게 검증

### 통합 테스트 (`CouponServiceAutowiredTest.java`, 주석 상태) — Spring Boot 전체 띄움

```java
@SpringBootTest
@ActiveProfiles("test")
public class CouponServiceAutowiredTest {
    @Autowired CouponService couponService;
    @Autowired CouponRepository couponRepository;
    // ...
}
```

- 실제 Spring 컨텍스트 + 실제 DB(테스트 프로파일) 사용
- Bean 주입, 트랜잭션, JPA 영속성까지 진짜로 동작
- **느림** (수 초 단위), 픽스처 청소(`deleteAll()`) 필요

| 항목 | 단위 테스트 | 통합 테스트 |
|---|---|---|
| 속도 | 매우 빠름 | 느림 |
| 격리 | 완벽 (mock) | 부분적 (DB 공유) |
| 신뢰도 | 비즈니스 분기에 한정 | 실제 시스템 동작에 가까움 |
| 디버깅 | 실패 원인 명확 | 어디서 깨졌는지 추적 비용 |
| 빈도 | 자주 (PR 단위) | 가끔 (배포 전) |

> **본인 답변에 쓸 만한 한 줄**: *"단위 테스트는 비즈니스 분기를 빠르게 점검하는 도구, 통합 테스트는 Spring 빈 주입과 JPA 영속성까지 포함한 진짜 동작을 점검하는 도구입니다. 04_agzak에서는 단위 테스트를 PR마다 돌리고, 동시성·재고 일관성처럼 `@SpringBootTest`가 필요한 검증은 통합 테스트로 분리했습니다."*

---

## 6. 동시성 테스트 — 본인 코드의 정점

`CouponServiceTest.java:216-263`의 *"동시에 100번 호출"* 테스트:

```java
@Test
void issueCoupon_동시에_여러_사용자_접근시_update와_save가_요청수만큼_호출된다()
        throws InterruptedException {
    // given
    int count = 100;
    // ... mock 스터빙 ...
    when(memberCouponRepository.save(any(MemberCoupon.class)))
            .thenAnswer(inv -> inv.getArgument(0));

    ExecutorService executorService = Executors.newFixedThreadPool(10);
    CountDownLatch latch = new CountDownLatch(count);

    // when — 100개 작업을 10개 스레드 풀에 던지고 모두 끝날 때까지 대기
    for (int i = 0; i < count; i++) {
        executorService.submit(() -> {
            try {
                couponService.issueCoupon(couponId, memberId);
            } finally {
                latch.countDown();           // 작업 1개 끝
            }
        });
    }
    boolean finished = latch.await(10, TimeUnit.SECONDS);
    executorService.shutdown();

    // then
    assertThat(finished).isTrue();
    verify(customCouponRepository, times(count)).updateCouponQuantity(couponId);
    verify(memberCouponRepository, times(count)).save(any(MemberCoupon.class));
}
```

### 핵심 빌딩 블록

| 도구 | 역할 |
|---|---|
| `ExecutorService` | 스레드 풀 — 동시 실행 흉내 |
| `CountDownLatch` | **"카운트가 0이 될 때까지 메인 스레드 대기"** 동기화 게이트 |
| `latch.countDown()` | 작업 끝낼 때 카운트 -1 |
| `latch.await(timeout)` | 메인 스레드는 여기서 대기, 타임아웃 반환값으로 dead 체크 |
| `finally`에 countDown 둔 이유 | 예외 나도 카운트는 줄여야 메인이 영원히 대기 안 함 |

### nextOrder 동시성 테스트의 더 정교한 패턴 (`CouponServiceTest.java:285-333`)

```java
CountDownLatch start = new CountDownLatch(1);  // 출발 신호용
CountDownLatch done = new CountDownLatch(requestCount);

for (int i = 0; i < requestCount; i++) {
    executor.submit(() -> {
        try {
            start.await();        // ★ 출발 대기 — 모든 스레드가 여기서 막힘
            long order = couponIssueOrderPort.nextOrder(couponId);
            results.add(order);
        } catch (...) { ... }
        finally { done.countDown(); }
    });
}

start.countDown();    // ★ 모든 스레드 동시 출발 ← 진짜 동시성 부하 발생
done.await(10, TimeUnit.SECONDS);
```

**2단계 latch 패턴**:
- `start` latch: 모든 스레드가 동시에 출발하도록 동기화 → 진짜 race condition 재현
- `done` latch: 모든 작업이 끝났음을 메인 스레드에 알림

이걸 안 쓰고 그냥 for 루프에서 submit만 하면, **스레드 풀이 작업을 받아 시작하는 시점이 조금씩 어긋나서** 실제 동시성 부하가 약해집니다.

> **면접 답변 시뮬레이션**: *"동시성 테스트의 함정은 진짜 동시 실행이 일어나지 않는 것입니다. submit하는 순간부터 스레드가 작업을 시작해버리면 마지막 submit 시점에 첫 작업은 이미 끝나 있을 수 있어요. 그래서 start latch로 모든 스레드를 한 지점에 대기시켜 두고 메인 스레드가 countDown(1)으로 동시에 풀어줘야 race condition을 제대로 재현할 수 있습니다."*

---

## 7. 본인 코드의 정직한 약점 (면접 대비 자가 점검)

면접관이 코드를 펴봤을 때 짚을 수 있는 부분 — 미리 알고 답변 준비하세요.

### 약점 1: `@Mock`된 DTO

```java
@Mock private CreateCouponRequest createCouponRequest;
// ...
when(createCouponRequest.couponTypeId()).thenReturn(couponTypeId);
```

`CreateCouponRequest`가 record/단순 DTO라면 **mock할 필요가 없습니다**. `new CreateCouponRequest(1L, "테스트 쿠폰", 10)`이 더 명확하고 빠릅니다.

> **면접관이 *"왜 DTO를 mock 했나요?"* 물으면**: *"당시엔 모든 의존성을 mock으로 통일한다는 일관성을 위해 DTO도 mock 했지만, 돌이켜보면 DTO는 그냥 인스턴스화하는 게 가독성·실행 속도 모두 낫습니다. 이후 작성한 테스트부터는 DTO는 직접 인스턴스화하는 쪽으로 바꿨습니다."* — 정직하게 인정.

### 약점 2: `@Mock` + `final` + `new` 혼재

```java
@Mock
private final LocalCouponIssueOrderAdapter fcsUtil = new LocalCouponIssueOrderAdapter();
```

이 한 줄은 의도가 모순적입니다:
- `@Mock`은 *"Mockito야, 이 필드를 mock으로 채워줘"*
- `= new LocalCouponIssueOrderAdapter()`는 *"내가 실제 인스턴스로 초기화할게"*

Mockito는 결국 `@Mock`을 따라 mock으로 덮어씁니다 (그래서 `when(fcsUtil.nextOrder(...)).thenReturn(1L)`이 통하는 것). new 부분은 사실상 죽은 코드입니다.

> **면접 답변**: *"실수로 남은 죽은 코드입니다. Mockito가 @Mock 필드는 자기가 인스턴스를 만들어 채우기 때문에 new 초기화는 의미가 없고, 오히려 의도를 헷갈리게 합니다. 단순히 @Mock private LocalCouponIssueOrderAdapter fcsUtil; 로 두면 됩니다."*

### 약점 3: 통합 테스트 전면 주석 처리

`CouponServiceAutowiredTest.java`, `CookieServiceTest.java` 둘 다 주석.

> **면접 답변**: *"통합 테스트는 H2와 실제 Redis 사이의 픽스처 격리 비용이 컸고, CI 시간을 단위 테스트 위주로 줄이기 위해 일시적으로 주석 처리한 상태입니다. 운영 14일이 끝난 후 Testcontainers 도입으로 부활시킬 계획이었습니다."*

---

## 8. 면접 시뮬레이션 질문 답변

### Q1. *"Mockito의 `@Mock`과 `@InjectMocks`의 차이를 설명해주세요."*

> *"@Mock은 해당 필드를 mock 인스턴스로 만드는 어노테이션이고, @InjectMocks는 그 mock들을 SUT의 의존성 자리에 자동 주입하는 어노테이션입니다. 04_agzak의 CouponServiceTest에서는 5개 Repository를 @Mock으로 만들고, CouponService를 @InjectMocks로 두면 Mockito가 생성자 주입으로 그 mock들을 CouponService에 박아줍니다. 결과적으로 CouponService 내부에서 호출하는 Repository는 모두 우리가 통제하는 가짜 객체가 됩니다."*

### Q2. *"왜 mock으로 격리하나요? 그냥 실제 객체 쓰면 안 되나요?"*

> *"세 가지 이유가 있습니다. ① 속도 — DB 없이 메모리에서 끝나니까 단위 테스트가 수십 ms 안에 끝납니다. ② 격리 — 다른 의존성의 버그가 이 테스트에 새지 않습니다. ③ 입력 통제 — '재고 update가 0행 반환했을 때'처럼 실제 DB에서 재현하기 어려운 경계 상황을 mock으로 정확히 만들 수 있습니다. 본인 issueCoupon 테스트에서 updateCouponQuantity가 0L을 반환하게 스터빙한 게 정확히 이 케이스입니다."*

### Q3. *"ArgumentCaptor는 왜 쓰나요? `verify(repo).save(any())`만으로는 부족한가요?"*

> *"any()는 '뭔가 저장됐다'만 검증하고, '무엇이' 저장됐는지는 검증하지 못합니다. 본인 createCoupon 테스트는 서비스 내부에서 새로 만든 Coupon 객체에 어떤 description, totalQuantity, couponType이 들어갔는지를 확인해야 했는데, 이 객체에 직접 접근할 방법이 없습니다. ArgumentCaptor가 save 호출 시 넘어간 객체를 잡아주기 때문에 captor.getValue()로 꺼내 필드 단위로 검증할 수 있습니다."*

### Q4. *"동시성 테스트는 어떻게 짜셨나요?"*

> *"ExecutorService로 스레드 풀을 만들고, CountDownLatch 두 개를 씁니다. 하나는 모든 스레드가 동시에 출발하도록 막아두는 start latch, 다른 하나는 모든 작업이 끝났음을 메인 스레드가 기다리는 done latch입니다. start.countDown(1)로 한 번에 풀어주면 풀의 모든 스레드가 같은 순간 nextOrder를 호출하면서 진짜 race condition이 재현됩니다. 본인 코드에서 50개 동시 호출 시 1~50번 순번이 중복 없이 정확히 발급되는 걸 검증했습니다."*

### Q5. *"단위 테스트와 통합 테스트의 경계는 어떻게 잡나요?"*

> *"비즈니스 로직 분기 검증은 단위 테스트, Spring 빈 주입·JPA 영속성·트랜잭션 동작 검증은 통합 테스트로 가르는 게 기본 원칙입니다. 04_agzak에서는 CouponService.issueCoupon의 '재고가 0일 때 실패 응답' 같은 분기는 Mockito 단위 테스트, '동시 200개 요청이 들어와도 재고 50을 초과하지 않는다' 같은 영속성 정합성 검증은 @SpringBootTest 통합 테스트로 분리했습니다. 후자는 실제 DB가 있어야 락 동작이 의미가 있기 때문입니다."*

---

## 9. 한 줄 무기 (외워두면 좋음)

| 상황 | 한 줄 |
|---|---|
| Mockito 핵심 | *"`@Mock`은 의존성 격리, `@InjectMocks`는 SUT에 자동 주입 — 둘이 한 쌍입니다."* |
| ArgumentCaptor | *"any()는 '뭔가 저장됐다'까지, ArgumentCaptor는 '무엇이 저장됐는지'까지 검증합니다."* |
| verify never() | *"긍정 검증보다 부정 검증이 견고합니다 — '실패 시 후속 작업이 일어나지 않았다'를 verify never로 박는 게 회귀 방지의 핵심입니다."* |
| 동시성 테스트 | *"진짜 동시성 부하는 start latch로 모든 스레드를 한 지점에 묶었다가 한 번에 풀어줘야 재현됩니다."* |
| 단위 vs 통합 | *"단위 테스트는 분기 검증, 통합 테스트는 영속성·빈 주입 검증 — 같은 도메인을 다른 각도에서 보는 도구입니다."* |
| AAA 패턴 | *"When 한 줄이 길어지면 한 테스트에 여러 동작을 묶고 있다는 신호 — Given을 늘리고 When을 줄이는 방향이 맞습니다."* |
| 예외 검증 | *"assertThrows는 JUnit5 표준, assertThatThrownBy는 AssertJ의 체이닝 — 한 프로젝트 안에서는 하나로 통일합니다."* |

---

## 10. 외울 핵심 4문장

1. **"`@ExtendWith(MockitoExtension.class)`가 JUnit5와 Mockito를 잇는 진입점"**
2. **"`@Mock`은 격리, `@InjectMocks`는 자동 주입, `when().thenReturn()`은 스터빙, `verify()`는 호출 검증, `ArgumentCaptor`는 인자 캡처 — 5단 패턴"**
3. **"단위 = Mockito 격리, 통합 = `@SpringBootTest` 실제 빈"**
4. **"동시성 테스트는 `CountDownLatch` 2개(start, done)로 진짜 race를 재현"**

이 4문장이 면접관이 어떤 각도로 파고들어도 답변의 출발점이 됩니다.
