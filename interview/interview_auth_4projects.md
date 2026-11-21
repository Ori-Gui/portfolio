# 면접 답변 가이드 — 본인 프로젝트 코드 기반 (인증 도메인)

> 작성: 2026-05-07 / 검증 기준: 코드 1순위(grep + 직접 read), 포트폴리오·블로그 2순위
> 대상: 현대오토에버 등 엔터프라이즈 SSO·백엔드 직무 면접
> 근거 코드: `01_bookstore`·`02_ttomanito`·`03_toktokggubi`·`04_agzak` 인증 모듈

---

## 0. 4개 프로젝트의 실제 인증 구현 비교 (사실관계 검증)

| 항목 | 01_bookstore (MSA) | 02_ttomanito (모놀리식) | 03_toktokggubi (RN+모바일) | 04_agzak (PWA) |
|---|---|---|---|---|
| **아키텍처** | **Gateway + auth 모듈 분리 MSA** (Eureka 디스커버리) | 모놀리식, OAuth2 Resource Server | 모놀리식, 모바일 앱이 클라이언트 | 모놀리식, PWA + 모바일 |
| **토큰 방식** | JWT (gateway·auth 모듈 동일 secret) | JWT Access + Refresh | JWT Access + Refresh | JWT Access + Refresh |
| **토큰 저장(클라)** | (Spring MVC front 모듈이 처리) | **HttpOnly Cookie + Secure + SameSite=None/Lax** (`CookieUtils.java`) | RN SecureStorage (헤더 전송) | 헤더 전송(PWA) |
| **Refresh 저장(서버)** | DB(`InvalidRefreshToken` / `MemberLogoutSuccessHandler`) | **JPA Entity + Repository**(RDB) | **Redis** `auth:refresh:{userId}` + TTL | **Redis** `auth:refresh:{memberId}` + TTL |
| **RTR (Rotation)** | 미적용 | 미적용 (access만 재발급) | 토대 마련 (Redis save 덮어쓰기) | **완비**: `issueTokens()`에서 access·refresh 둘 다 재발급, Redis save로 이전 토큰 폐기 |
| **Access 강제 무효화** | 없음 | 없음 | 없음 | **`TokenRevocationService`**(Redis 블랙리스트 + access TTL) |
| **소셜 로그인** | 없음 | Kakao OAuth2 + `OAuth2AuthenticationSuccessHandler` | Kakao OAuth2 + 앱 복귀 딥링크 | **Kakao + 이메일 + 게스트 + 게스트→회원 전환** (4-way) |
| **WebSocket 인증** | — | **STOMP + `JwtHandshakeInterceptor` + `WebSocketAuthChannelInterceptor`** → `@MessageMapping` Principal 자동 주입 | STOMP 배틀 동기화 | — |
| **임시 상태 저장** | — | — | — | `SocialSignupTicketStore`, `GuestProfileStore` (모두 Redis) |
| **블로그 자료** | `bookstore_elasticsearch.md`(인증 X) | **`ttomanito_websocket_principal.md`** (STOMP Principal 주입 트러블슈팅) | `toktokggubi_gifticon_ocr_vision.md`(인증 X) | `agzak_scheduler_deadlock.md`, `agzak_ranking_db_split.md` (인증 X — 다만 ShedLock·Redis ZSET 분산락 경험) |

> **핵심 결론**: 본인은 같은 JWT 위에서 **저장소(JPA→Redis)**, **회전 정책(없음→RTR)**, **클라이언트 저장(쿠키→헤더)**, **무효화(없음→블랙리스트)** 까지 4단계로 점진 진화한 경험을 가짐. 04_agzak가 정점.

---

## 1. "세션 vs JWT" 질문에 — 본인 프로젝트로 답하기

> *"세 가지 인증 방식 중 가장 정교했던 게 04_agzak입니다. 게스트/이메일/카카오 3-way 진입을 단일 JWT 발급 체계로 묶으려면 서버가 상태를 들고 있는 세션은 답이 아니었습니다. 세션은 서버 메모리(또는 세션 DB)에 사용자 식별 정보를 보관하기 때문에 무상태 확장과 모바일/PWA 동시 지원이 어렵습니다."*

> *"다만 JWT의 가장 큰 약점인 '강제 만료가 어렵다'는 문제는 04_agzak에서 두 단계로 막았습니다. 첫째는 Refresh Token Rotation — `AuthService.refresh()`에서 들어온 토큰을 Redis에 저장된 값과 비교하고, 일치하면 `issueTokens()`로 access·refresh를 함께 재발급하면서 Redis save로 이전 refresh를 덮어 씁니다. 둘째는 별도의 `TokenRevocationService`로 로그아웃된 access 토큰을 access TTL 동안 Redis 블랙리스트에 두는 것입니다."*

**비교용 문장**:

> *"02_ttomanito에서는 RefreshToken을 JPA Entity + RDBMS로 관리했는데, 인증 요청마다 디스크 I/O가 발생하는 한계가 있었고 RTR도 적용하지 않아 access만 재발급했습니다. 이 경험이 03·04에서 Redis 키 + TTL로 옮겨가게 된 직접적인 이유입니다."*

---

## 2. "HttpOnly 쿠키 / RTR / Redis" — 04_agzak + 02_ttomanito 두 사례로

### HttpOnly 쿠키 — 02_ttomanito

`CookieUtils.java`에서 `ResponseCookie.from(...).httpOnly(true).secure(secure).sameSite("None"|"Lax")` 형태로 발급. SameSite를 환경별로 분기한 이유는 마니또 모바일 PWA가 다른 도메인에서 호출할 때(`SameSite=None`)와 로컬 개발(`SameSite=Lax`)의 쿠키 전송 정책이 달랐기 때문이라고 답하면 깊이가 드러남.

> *"브라우저 LocalStorage에 토큰을 두면 XSS 한 줄로 토큰이 새어나갑니다. 02_ttomanito는 Refresh Token을 HttpOnly + Secure + SameSite=None Cookie로 내려, 자바스크립트 접근을 차단하면서 PWA 크로스도메인 인증을 가능하게 했습니다. 반면 04_agzak는 PWA지만 헤더 기반으로 단순화하고 그 대신 Redis 블랙리스트로 access 무효화를 책임졌습니다 — 어느 쪽이 정답이라기보다 클라이언트 환경(쿠키 자동전송이 가능한가, CORS preflight 비용이 부담인가)에 따른 트레이드오프였습니다."*

### RTR — 04_agzak 코드 직인용 가능

- `issueTokens(memberId)`: access·refresh 모두 새로 발급 → `refreshStore.save(memberId, refresh, Duration.ofDays(...))`가 같은 키에 덮어쓰기 = 이전 refresh 자동 폐기
- `refresh(refreshToken)`: 들어온 refresh의 memberId를 파싱 → Redis에서 가져온 값과 `equals` 비교 → 불일치 시 `BadCredentialsException("리프레시 토큰이 만료되었거나 회수되었습니다")`

> *"RTR은 '이미 폐기된 옛날 토큰을 누군가 다시 들이밀면 탈취 시그널로 본다'는 원리입니다. 04_agzak는 Redis가 단일 진실 원천이라 회전 검증이 O(1) GET 한 번으로 끝납니다."*

### Redis를 선택한 이유 — 04_agzak 코드의 사용처들

- `auth:refresh:{memberId}` (Refresh 보관)
- `TokenRevocationService` (Access 블랙리스트, access TTL 동기화)
- `GuestProfileStore` (게스트 임시 프로필)
- `SocialSignupTicketStore` (카카오 추가정보 입력 단계 티켓)

> *"이 4가지 모두 (1) 짧은 TTL이 본질이고 (2) 매 요청마다 검증되며 (3) 정합성보다 속도가 중요한 데이터입니다. RDBMS에 두면 인증 경로의 디스크 I/O가 그대로 병목이 됩니다. Redis의 TTL 자동 만료 덕분에 별도 청소 배치도 필요 없었고요."*

---

## 3. "RDBMS vs NoSQL" — 02_ttomanito가 가장 강한 사례

> *"또마니또에서는 같은 도메인에 3개 저장소를 책임 분리했습니다. 회원·매칭·결제 메타데이터는 MySQL(JPA), 마니또 1:1 채팅 메시지는 MongoDB, 인증 토큰·세션은 Redis. 채팅을 MongoDB로 분리한 이유는 (1) 메시지 폭주에 append-only 특성이 적합했고 (2) 조인이 필요 없고 (3) STOMP 1:1 라우팅에서 사용자별 룸 ID로 시간순 조회가 90% 이상이었기 때문입니다. 이걸 MySQL에 넣었다면 chat_message 테이블 인덱스 비용과 행 잠금이 실시간 채팅 지연으로 직결됐을 겁니다."*

> *"04_agzak에서도 같은 원리로, 운세 강화게임 랭킹은 Redis ZSET(O(log N) 정렬)에서 실시간 계산하고, 자정 ShedLock 분산락으로 MySQL 스냅샷에 영속화하는 이원화 구조를 썼습니다. ZSET은 정합성보다 속도, MySQL 스냅샷은 감사·재현용 — 데이터 성격에 맞춰 저장소를 나누는 사고방식입니다."*

---

## 4. "Spring Boot 3계층 / Spring Security" — 01_bookstore MSA가 답변 자산

> *"3계층은 모든 프로젝트에 공통이지만, 01_bookstore에서 한 단계 더 나아가 6개 Maven 모듈로 책임을 분리했습니다 — Eureka(디스커버리)·Gateway(라우팅·인증 진입점)·Auth(JWT 발급·로그아웃)·Book(상품)·Batch(스케줄)·Front(MVC). Spring Security 필터 체인은 각 모듈마다 자체 보안 정책을 두되, Gateway가 ReactiveSecurity(`SecurityWebFilterChain`)로 1차 진입점이 됩니다."*

> *"이 구조가 면접에서 SSO 환경과 직결되는 이유는, 현대오토에버처럼 여러 내부 서비스가 하나의 인증을 공유해야 할 때 Gateway가 'SSO의 단일 진입점' 역할을, Auth 모듈이 'IDP(identity provider)' 역할을 분리해서 맡는 것이 자연스럽기 때문입니다. 비즈니스 로직(Book·Batch)은 Resource Server로서 토큰 검증만 하면 되고, 인증 정책 변경이 도메인 코드에 새지 않습니다."*

> *"02_ttomanito에서는 STOMP WebSocket까지 같은 보안 정책을 확장하기 위해 `JwtHandshakeInterceptor`(핸드셰이크 단계 토큰 추출) + `WebSocketAuthChannelInterceptor`(STOMP CONNECT 프레임에서 Authentication을 simpUser로 주입) 2단 필터를 직접 구성했고, 그 결과 `@MessageMapping` 메서드에서 Principal을 자동 주입받아 1:1 채팅 라우팅을 검증할 수 있었습니다 — 이 경험을 블로그(`ttomanito_websocket_principal.md`)에 정리해 뒀습니다."*

---

## 5. 마지막 시뮬레이션 질문에 대한 답변

> **질문**: *"수만 명의 임직원이 동시에 접속하는 SSO 로그인 시스템에 일시적 트래픽 폭주가 발생한다면, JWT와 Redis의 특성을 활용해 어떻게 부하를 줄이면서 안전하게 인증을 처리하시겠습니까?"*

### 답변 골격 (본인 04_agzak 경험으로)

**1. Stateless JWT로 검증 부하 분산**

> *"먼저 access token은 RSA 또는 HMAC 검증만으로 끝나는 stateless 토큰으로 두면, 모든 backend pod가 인증 검증을 메모리 내에서 처리합니다. SSO IDP는 발급만 책임지고, 각 서비스는 검증을 분산 처리하므로 로그인 폭주가 IDP 한 곳에 몰리지 않습니다."*

**2. 로그인 폭주 자체는 Redis로 흡수**

> *"실제 로그인 요청은 (1) 비밀번호 해시 검증 — 04_agzak에서 BCrypt 사용 — 과 (2) Refresh 발급으로 나뉩니다. (1)은 PasswordEncoder가 의도적으로 느린 함수라 CPU에 묶이고, (2)는 Redis SET 한 번이면 충분합니다. 04_agzak에서는 `refreshStore.save(memberId, token, ttl)` 한 줄이 RTR과 만료까지 책임집니다 — RDBMS에 INSERT/UPDATE 했다면 바로 병목이 됐을 부분입니다."*

**3. Refresh Token Rotation으로 탈취 위험 차단**

> *"임직원 SSO는 한 번 탈취되면 사내 다수 시스템이 위험합니다. 04_agzak처럼 RTR을 적용해 매 갱신마다 refresh를 회전시키면, 같은 refresh가 두 번 들어오는 순간을 탈취 시그널로 잡고 즉시 무효화할 수 있습니다."*

**4. Access 무효화는 짧은 TTL + 블랙리스트**

> *"강제 로그아웃·세션 해제 요구가 들어오면, access TTL이 끝나기 전이라도 04_agzak의 `TokenRevocationService`처럼 Redis에 access 토큰 jti를 access 잔여 TTL만큼 보관하면 됩니다. TTL이 자동 만료라 메모리도 알아서 회수됩니다."*

**5. 그래도 Redis 자체가 SPOF가 되지 않도록**

> *"마지막으로 SSO Redis는 Sentinel 또는 Cluster로 HA 구성하고, 캐시 미스 시 RDB로 폴백하는 안전장치가 필요합니다. 04_agzak 운영 14일 동안 InnoDB Deadlock 0건을 유지한 비결도 ShedLock 분산락 + Redis 임시 상태 분리였는데, 같은 원리를 SSO에서는 'Redis HA + RDB 영속화 분리'로 적용할 수 있습니다."*

---

## 6. 답변 시 활용할 한 줄 무기들 (외워두면 좋음)

| 상황 | 한 줄 |
|---|---|
| RTR을 왜 했나 | *"세션의 즉시 만료 능력을 JWT 위에서 흉내 내는 가장 가벼운 방법이 RTR이라고 판단했습니다."* |
| Redis vs RDBMS | *"인증 경로 데이터는 정합성보다 속도와 자동만료가 본질이라 RDBMS의 트랜잭션 보장이 오히려 오버헤드입니다."* |
| MSA에서 SSO | *"Gateway는 진입점, Auth는 IDP, Resource Server는 토큰 검증만 — 인증 책임을 도메인 코드에서 분리하는 것이 SSO의 출발입니다."* |
| HttpOnly 선택 분기 | *"쿠키냐 헤더냐는 클라이언트 환경의 문제입니다. 02_ttomanito PWA는 HttpOnly Cookie, 04_agzak PWA는 헤더 + Redis 블랙리스트 — 같은 위협을 다른 위치에서 막은 사례입니다."* |
| 게스트→회원 전환 | *"04_agzak는 게스트 사용자도 게임 결과를 잃지 않게 GuestProfileStore + SocialSignupTicketStore 2단 Redis로 임시상태를 보관하고, 카카오 추가정보 완료 시점에 영속화했습니다."* |

---

## 7. 면접관 검증 대비 — 코드 위치 인덱스

| 주장 | 파일 경로 |
|---|---|
| RTR 구현 | `04_agzak/agzak/backend/src/main/java/com/ssafy/agzak/domain/auth/service/AuthService.java` (`issueTokens`, `refresh` 메서드) |
| Redis Refresh Store | `04_agzak/.../global/security/RefreshTokenStore.java` (`save`/`get`/`delete`) |
| Access 블랙리스트 | `04_agzak/.../global/security/TokenRevocationService.java` |
| HttpOnly 쿠키 | `02_ttomanito/.../global/security/utils/CookieUtils.java` |
| STOMP Principal 주입 | `02_ttomanito/.../global/security/filter/JwtHandshakeInterceptor.java` + `WebSocketAuthChannelInterceptor.java` |
| 게이트웨이 보안 | `01_bookstore/2joping-gateway/.../config/SecurityConfig.java` (`SecurityWebFilterChain`, ReactiveSecurity) |
| Refresh JPA Entity | `02_ttomanito/.../domain/auth/entity/RefreshToken.java` + `RefreshTokenRepository.java` |
| Redis Refresh (toktokggubi) | `03_toktokggubi/.../global/security/token/RefreshTokenStore.java` |

> 면접관이 "실제로 그렇게 구현했나요?" 라고 물으면 위 파일·메서드명까지 댈 수 있게 인덱싱.
