# 면접 답변 가이드 (Interview Notes)

본인 프로젝트 4개(`01_bookstore` · `02_ttomanito` · `03_toktokggubi` · `04_agzak`)의 **실제 코드를 근거로** 정리한 기술 면접 답변 가이드입니다.
각 문서는 "코드 사실관계 → 질문별 답변 → 한 줄 무기 → 면접 시뮬레이션" 순서로 구성되어 있고, 모든 주장에 파일·메서드 위치를 함께 적어 두었습니다.

| # | 문서 | 다루는 주제 | 작성일 |
|---|---|---|---|
| 1 | [interview_auth_4projects.md](./interview_auth_4projects.md) | 인증 도메인 4개 프로젝트 비교 — 세션 vs JWT, HttpOnly 쿠키·RTR·Redis 저장소, RDBMS vs NoSQL, Spring Security 3계층 | 2026-05-07 |
| 2 | [interview_ranking_idempotency_agzak.md](./interview_ranking_idempotency_agzak.md) | 04_agzak 랭킹·게임 멱등 — Redis ZSET 랭킹, reseed·스냅샷·fallback, ShedLock 분산락, Lua 원자 연산, 비관적 락 | 2026-05-08 |
| 3 | [interview_junit5_mockito_agzak.md](./interview_junit5_mockito_agzak.md) | 테스트 — JUnit5 + Mockito 5단 패턴, AAA 구조, 예외 검증, 단위 vs 통합 테스트, CountDownLatch 동시성 테스트 | 2026-05-09 |
| 4 | [interview_deeplink_oauth_toktokggubi.md](./interview_deeplink_oauth_toktokggubi.md) | 03_toktokggubi 모바일 OAuth2 — 커스텀 스킴 딥링크, HTML 3중 fallback, 이중 복귀 대기, PKCE 트레이드오프 | 2026-05-10 |

## 읽는 순서

1. **인증 전반**은 1번 문서의 §0 비교표에서 시작합니다. 4개 프로젝트의 진화 축(저장소 JPA→Redis, 회전 정책 없음→RTR, 블랙리스트)이 한 표에 정리되어 있습니다.
2. **깊이 질문**(동시성·분산락·캐시·멱등)은 2번, **테스트 역량**은 3번, **외부 IdP·모바일 딥링크**는 4번이 대응합니다.
3. 각 문서 말미의 "외울 핵심 4문장"만 따로 읽어도 답변 출발점으로 쓸 수 있습니다.

## 참고

- 문서끼리 상대 링크(`./interview_*.md`)로 연결되어 있어 이 폴더 안에서 함께 읽어야 링크가 동작합니다.
- 코드 위치 표기는 로컬 저장소 기준 경로(예: `04_agzak/domain/ranking/...`)이며, 공개 저장소의 최신 코드와 경로가 다를 수 있습니다.
