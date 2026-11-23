# 백엔드 포트폴리오

Java · Spring Boot 백엔드 개발자 **양준하**의 포트폴리오입니다. SSAFY 13기와 NHN 아카데미 7기에서 진행한 프로젝트 4건을 "문제 → 해결 → 결과" 순서로 페이지마다 정리했고, 같은 내용을 PDF로도 내려받을 수 있습니다. 이 저장소는 [GitHub Pages](https://ori-gui.github.io/portfolio/)로 배포되는 사이트 소스이며, 빌드 과정 없이 정적 HTML만으로 동작합니다.

- 🌐 **배포 사이트** — https://ori-gui.github.io/portfolio/
- 🪪 **이력서(프로필 · 자격증)** — [00_profile_summary_assets/html/profile_summary.html](00_profile_summary_assets/html/profile_summary.html) (2페이지)
- 📄 **통합 PDF** — [full_portfolio.pdf](pdf/full_portfolio.pdf) (18페이지) / [all_projects_portfolio.pdf](pdf/all_projects_portfolio.pdf) (백엔드 4건 13페이지 · 12.6MB)
- 🐙 **GitHub** — https://github.com/Ori-Gui
- ✍️ **Velog** — https://velog.io/@ori_gui/posts

---

## 🛠 기술 스택

**Backend**
<img src="https://img.shields.io/badge/Java-007396?style=flat-square&logo=openjdk&logoColor=white"> <img src="https://img.shields.io/badge/Spring%20Boot-6DB33F?style=flat-square&logo=springboot&logoColor=white"> <img src="https://img.shields.io/badge/Spring%20Security-6DB33F?style=flat-square&logo=springsecurity&logoColor=white"> <img src="https://img.shields.io/badge/Spring%20Data%20JPA-6DB33F?style=flat-square&logo=spring&logoColor=white"> <img src="https://img.shields.io/badge/QueryDSL-0769AD?style=flat-square&logo=java&logoColor=white">

**Auth / Realtime**
<img src="https://img.shields.io/badge/JWT-000000?style=flat-square&logo=jsonwebtokens&logoColor=white"> <img src="https://img.shields.io/badge/OAuth2-3C3C3D?style=flat-square&logo=oauth&logoColor=white"> <img src="https://img.shields.io/badge/WebSocket%20%2F%20STOMP-010101?style=flat-square&logo=socketdotio&logoColor=white">

**Database / Cache / Search**
<img src="https://img.shields.io/badge/MySQL-4479A1?style=flat-square&logo=mysql&logoColor=white"> <img src="https://img.shields.io/badge/Redis-DC382D?style=flat-square&logo=redis&logoColor=white"> <img src="https://img.shields.io/badge/MongoDB-47A248?style=flat-square&logo=mongodb&logoColor=white"> <img src="https://img.shields.io/badge/Elasticsearch-005571?style=flat-square&logo=elasticsearch&logoColor=white">

**Test**
<img src="https://img.shields.io/badge/JUnit5-25A162?style=flat-square&logo=junit5&logoColor=white"> <img src="https://img.shields.io/badge/Mockito-25A162?style=flat-square&logo=mockito&logoColor=white">

**Infra / DevOps**
<img src="https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white"> <img src="https://img.shields.io/badge/Jenkins-D24939?style=flat-square&logo=jenkins&logoColor=white"> <img src="https://img.shields.io/badge/Nginx-009639?style=flat-square&logo=nginx&logoColor=white"> <img src="https://img.shields.io/badge/AWS%20EC2-FF9900?style=flat-square&logo=amazonec2&logoColor=white"> <img src="https://img.shields.io/badge/GitHub%20Actions-2088FF?style=flat-square&logo=githubactions&logoColor=white">

**Frontend**
<img src="https://img.shields.io/badge/React-20232A?style=flat-square&logo=react&logoColor=61DAFB"> <img src="https://img.shields.io/badge/Thymeleaf-005F0F?style=flat-square&logo=thymeleaf&logoColor=white">

**Collaboration / Docs**
<img src="https://img.shields.io/badge/Swagger%20%2F%20OpenAPI-85EA2D?style=flat-square&logo=swagger&logoColor=black"> <img src="https://img.shields.io/badge/Git-F05032?style=flat-square&logo=git&logoColor=white"> <img src="https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white"> <img src="https://img.shields.io/badge/Jira-0052CC?style=flat-square&logo=jira&logoColor=white"> <img src="https://img.shields.io/badge/Notion-000000?style=flat-square&logo=notion&logoColor=white">

---

## 🚀 프로젝트

전체 18페이지 구성입니다 — **이력서(프로필·자격증) 2p → 백엔드 메인 4건 13p → 서브/AI 3p**. 지원 직무에 따라 13페이지(백엔드 전용)와 18페이지(이력서·서브 포함) 중 골라 제출할 수 있습니다.

| # | 프로젝트 | 기간 | 팀 · 역할 | 핵심 주제 |
|---|---|---|---|---|
| 1 | [아그작](#1-아그작--데일리-운세-플랫폼) | 2025.10 – 11 (6주) | 6명 · Backend | Redis ZSET + MySQL 랭킹 이원화, 토큰 회전(RTR), 일일 Cap 멱등성 |
| 2 | [똑똑꺼비](#2-똑똑꺼비--ai-기반-예산-관리-앱) | 2025.08 – 09 (5주) | 6명 · Backend | 모바일 OAuth2 딥링크, GCP Vision OCR 파이프라인 |
| 3 | [또마니또](#3-또마니또--실시간-익명-채팅-마니또) | 2025.07 – 08 (5주) | 6명 · Backend | WebSocket Principal 통일, MySQL·MongoDB·Redis 3DB 분리 |
| 4 | [이조핑 북스토어](#4-이조핑-북스토어--온라인-도서-쇼핑몰) | 2024.10 – 12 (9주) | 8명 · FullStack | MySQL → Elasticsearch 검색 마이그레이션 (1.2초 → 280ms) |
| S1 | [트래블 퀘스트](#s1-트래블-퀘스트--여행-rpg-파티-매칭) | 2025.05 | 2명 · FullStack | MySQL 프로시저 던전 생성, Spring AI + Scheduler 퀘스트 자동 생성 |
| S2 | [YOLO 노면표지 탐지](#s2-yolo-노면표지-탐지--횡단보도과속방지턱-훼손-판별) | 2024.03 – 12 | 3명 · AI Model & Data | AI Hub 주행영상 전처리, YOLOv8s mAP50 0.80 (KSC2024 장려상) |
| S3 | [FastAPI 운영 실습](#s3-fastapi-운영-실습--28일-커리큘럼) | 2026 | 개인 | Keyset 인덱스, 인증 테스트, Redis/Celery, RDS 풀, Blue/Green 배포 |

### 1. 아그작 — 데일리 운세 플랫폼

운세 강화 게임이 결합된 데일리 운세 웹 서비스. 14일 운영 기간 방문자 1,100명·이벤트 11만 건을 받으며 랭킹·인증·스케줄러 구조를 실 트래픽으로 검증했습니다. 🏆 SSAFY 13기 자율 프로젝트 우수상 2등
`Spring Boot` `JPA` `MySQL` `Redis` `FastAPI` `OpenAI` `Docker` `Jenkins` `Nginx` `AWS EC2`

- **랭킹 이원화** — 점수 변동이 잦은 당일 랭킹은 Redis ZSET(`ZADD` O(log N), `ZREVRANGE`·`ZREVRANK`)으로, 확정된 과거 랭킹은 MySQL 스냅샷 테이블로 분리. 14일 운영 기간 Slow Query 0건.
- **인증** — 이메일·카카오·게스트 3-way 진입을 단일 JWT 발급 체계로 묶고, Refresh Token Rotation과 Redis 블랙리스트로 액세스 토큰 강제 만료까지 처리.
- **게임 정합성** — 일일 시도 제한(Cap)에 멱등성 키와 Lua 스크립트 원자 연산(`SETNX` + `INCR`)을 적용해 더블클릭·재시도에도 정확한 한도 유지, 재고 차감은 비관적 락으로 제어.
- **운영** — Blue/Green 두 슬롯과 `@SchedulerLock`으로 자정 cron 중복 실행 데드락(SQLState 40001) 재발 0건, Jenkins 4 파이프라인과 Prometheus·Grafana·Loki 모니터링 운영. GA4 기준 참여율 91%.

[개요](04_agzak_portfolio_assets/html/agzak_page1_overview.html) · [아키텍처](04_agzak_portfolio_assets/html/agzak_page2_architecture.html) · [랭킹 이원화](04_agzak_portfolio_assets/html/agzak_page3_ranking.html) · [운영 성과](04_agzak_portfolio_assets/html/agzak_page4_operations.html) · [PDF](pdf/agzak_portfolio.pdf)

### 2. 똑똑꺼비 — AI 기반 예산 관리 앱

이상 소비 탐지·개인화 예산 추천·기프티콘 통합 관리를 제공하는 모바일 앱(React Native/Expo). 🏆 SSAFY 13기 특화 프로젝트 우수상 1등
`Spring Boot` `MySQL` `Redis` `MongoDB` `WebSocket(STOMP)` `GCP Vision` `FCM` `Docker` `Jenkins` `AWS EC2`

- **모바일 OAuth2 SSO** — 앱 → 외부 브라우저 → 카카오 로그인 → 백엔드 콜백 → `toktokggubi://oauth/kakao` 커스텀 스킴 복귀까지 직접 설계. HTML 3중 fallback(meta refresh·intent URL·수동 링크)과 앱 측 이중 대기로 복귀 누락 방지.
- **OCR 파이프라인** — GCP Vision으로 기프티콘 이미지에서 텍스트를 추출하고, `GifticonTextParser`에서 브랜드 캐노니컬 매핑·만료일 정규식 파싱으로 정형 엔티티화. "스타벅스 / Starbucks / 스벅"이 같은 가맹점 ID로 저장됩니다.
- **외부 API 통합** — SSAFY 금융 OpenAPI, 카카오 OAuth, OCR처럼 지연 시간이 다른 외부 호출을 타임아웃·재시도·예외 격리 정책으로 묶어 단일 API 장애가 전체로 번지지 않게 처리.

[개요](03_toktokggubi_portfolio_assets/html/toktokggubi_page1_overview.html) · [OCR 파이프라인](03_toktokggubi_portfolio_assets/html/toktokggubi_page2_ocr_pipeline.html) · [OAuth2 + 딥링크](03_toktokggubi_portfolio_assets/html/toktokggubi_page3_oauth2_deeplink.html) · [PDF](pdf/toktokggubi_portfolio.pdf)

### 3. 또마니또 — 실시간 익명 채팅 마니또

익명 1:1 채팅·미션·푸시 알림·AI 성향 분석을 결합한 마니또 게임 모바일 웹(PWA).
`Spring Boot` `JPA` `MySQL` `MongoDB` `Redis` `WebSocket(STOMP)` `FCM` `GPT-4.1 mini` `Docker` `Jenkins` `Nginx` `AWS EC2`

- **WebSocket Principal 통일** — HTTP의 `SecurityContext`(ThreadLocal)가 비동기 STOMP 채널로 전파되지 않아 `Principal.getName()`이 비어 1:1 라우팅이 실패하던 문제를, `UserDetails`와 `Principal`을 동시에 구현한 `JwtUserDetails`와 Handshake·Channel 2단 인터셉터로 해결.
- **3DB 분리** — 외래 키와 트랜잭션이 필요한 메타 데이터는 MySQL, append-only 채팅 메시지는 MongoDB, 세션·Refresh 토큰·FCM 캐시는 Redis(TTL)로 책임 분리.
- **AI·푸시** — 채팅 30건을 비식별화해 GPT-4.1 mini로 성향 Top 3를 분석(호출 일일 1회·캐시)하고, FCM으로 PWA 백그라운드 푸시를 수신. 운영 Nginx·SSL·CORS 환경에서 토큰 유실 이슈 해결.

[개요](02_ttomanito_portfolio_assets/html/ttomanito_page1_overview.html) · [WebSocket Principal](02_ttomanito_portfolio_assets/html/ttomanito_page2_websocket_principal.html) · [3DB + AI](02_ttomanito_portfolio_assets/html/ttomanito_page3_3db_ai.html) · [PDF](pdf/ttomanito_portfolio.pdf)

### 4. 이조핑 북스토어 — 온라인 도서 쇼핑몰

Spring Boot + Thymeleaf 기반 도서 커머스. NHN 아카데미 7기 8인 팀에서 FullStack으로 검색 엔진 도입과 도메인 API를 담당했습니다.
`Spring Boot` `Thymeleaf` `MySQL` `Elasticsearch` `Redis` `Nginx` `GitHub Actions` `SonarQube` `NHN Cloud`

- **검색 마이그레이션** — `LIKE '%키워드%'` 앞 와일드카드로 B-Tree 인덱스를 쓰지 못하고 4개 테이블 LEFT JOIN·N+1까지 겹쳐 평균 1.2초가 걸리던 검색을 Elasticsearch로 이전. nori(형태소)·ngram(부분 일치)·jaso(자모·초성) 3개 분석기와 BM25 가중치(`title^10`/`desc^3`/`publisher^2`)로 약 280ms(76% 단축).
- **인덱스 운영** — 운영 DB와 검색 인덱스를 분리하고 Logstash JDBC 10초 polling upsert로 동기화. 정렬 6종(popularity·newest·price·rating·reviews 등)을 지원.
- **팀 협업** — 8인 팀의 WBS 기반 일정·분담, Swagger API 문서화, PR 코드 리뷰 정책과 스프린트 회고(KPT)를 주도.

[개요](01_bookstore_portfolio_assets/html/bookstore_page1_overview.html) · [검색 엔진 도입](01_bookstore_portfolio_assets/html/bookstore_page2_search_engine.html) · [한글 분석기](01_bookstore_portfolio_assets/html/bookstore_page3_korean_analyzer.html) · [PDF](pdf/bookstore_portfolio.pdf)

---

## 🧩 서브 / AI · 실습 프로젝트

메인 4건과 성격·무게가 다른 프로젝트는 별도 섹션으로 분리해, 백엔드 지원 시 기존 13페이지의 밀도가 흐려지지 않게 했습니다.

### S1. 트래블 퀘스트 — 여행 RPG 파티 매칭

여행을 RPG로 치환한 게이미피케이션 여행 플랫폼(SSAFY 13기 관통 프로젝트 · 2인). 직업 = MBTI 8종, 던전 = 여행지+기간, 퀘스트 = 여행지 미션, 파티 = 동행 모집으로 구조화했습니다.
`Spring Boot 3.4.5` `MyBatis` `MySQL 8` `Redis` `MongoDB` `Spring AI` `Vue 3` `Kakao Map`

- **던전 자동 생성** — 매일 00시 `@Scheduled` 배치가 던전 10개를 생성하고, MySQL 저장 프로시저 `create_dungeon_with_attractions`가 기간(0~8일)에서 난이도·필요 명소 수를 역산해 지역을 배정합니다.
- **퀘스트 자동 생성** — 배정된 관광지 정보를 LLM(gpt-4o-mini)에 JSON으로 넘겨 관광지당 퀘스트 1개를 생성하고, 코드펜스 제거·개수 불일치 검증(`QuestGenerationException`) 후 `@Transactional`로 저장합니다.
- **지도·인증** — Kakao Map 마커로 던전 표시, 인증 사진 업로드(UUID 저장·10MB 제한) → 관리자 승인/반려 워크플로.
- 팀 공동 구현으로 Kakao/Naver OIDC + JWT, STOMP 채팅(MongoDB), 직업군별 정원 검증이 함께 들어갔습니다(2인 분담).

[페이지 보기](05_sub_projects_assets/html/sub_page1_travelquest.html) · [저장소](https://github.com/Ori-Gui/travelquest)

### S2. YOLO 노면표지 탐지 — 횡단보도·과속방지턱 훼손 판별

학부 졸업작품(3인, AI · 데이터). AI Hub 「도로 로드마크 인식을 위한 주행 영상」 온전·비온전 약 59,000장 중 주간 28,000장으로 훼손 여부까지 4클래스 탐지를 학습했습니다.
`Python` `YOLOv8s` `PyTorch` `OpenCV` `AI Hub`

- **전처리** — 라벨 bbox와 이미지 크기가 어긋난 사례가 잦아 segmentation 폴리곤을 [x,y] 쌍으로 풀어 0~1 정규화 후 YOLO 포맷으로 변환.
- **학습** — YOLOv8s, 100 epoch, imgsz 640, batch 16 (8×Tesla P100). 1차 학습(42 epoch 중단) 후 설정을 정리해 2차에서 **mAP50 0.8057 / Precision 0.780 / Recall 0.753** 달성.
- **성과** — 「딥러닝 기반 횡단보도 및 과속방지턱 인식 및 훼손여부 탐지」로 2024 한국소프트웨어종합학술대회(KSC2024) 게재, 학부생/주니어 논문경진대회 **장려상**.

[페이지 보기](05_sub_projects_assets/html/sub_page2_yolo.html) · [저장소](https://github.com/Ori-Gui/yolodetect)

### S3. FastAPI 운영 실습 — 28일 커리큘럼

하나의 채용 서비스 API를 Day 단위 브랜치(27개)로 점진 진화시킨 개인 실습. 강의 골격 위에 5개 브랜치를 직접 구현·검증했습니다(직접 추가한 테스트 72건 전부 통과).
`Python 3.12` `FastAPI` `SQLModel` `PostgreSQL` `Redis` `Celery` `Docker` `AWS EC2/RDS` `GitHub Actions`

| 실습 | 구현 | 검증 |
|---|---|---|
| 인덱스 최적화 | `GET /job-posts/feed` Keyset 커서 페이지네이션, 복합·정렬·부분 인덱스 | `EXPLAIN QUERY PLAN` 인덱스 사용 확인 · 7 passed |
| 인증 · CRUD | 비밀번호 변경(Refresh 전체 폐기), 이메일 변경(400/409 구분) | 상태 코드별 13 케이스 · 27 passed |
| Redis · Celery | `CacheManager`(TTL·적중률·장애 폴백), 이메일 비동기 202 | TTL 경계·폴백·취소 시나리오 · 31 passed |
| RDS PostgreSQL | `build_engine()` 풀 튜닝(`pool_pre_ping`), `/health/db` | 풀 설정 주입·503 확인 · 28 passed |
| CI/CD · Blue/Green | `/health/ready`·`/health/version`, `scripts/bluegreen.py` CLI | 전환·롤백·구성 일치 · 35 passed |

> 강의 기반 비공개 저장소이므로 전체 소스는 공개하지 않고, 아키텍처·설계 판단과 직접 추가 구현한 기능·테스트 결과만 정리했습니다.

[페이지 보기](05_sub_projects_assets/html/sub_page3_fastapi_practice.html)

---

## 📄 포트폴리오 PDF · 에셋

각 HTML 페이지는 Figma A4(1190×1684) 기준으로 제작된 정적 문서이고, 같은 내용을 PDF로도 제공합니다.

| 파일 | 내용 |
|---|---|
| [pdf/full_portfolio.pdf](pdf/full_portfolio.pdf) | **전체 18페이지** — 이력서 2 + 백엔드 4건 13 + 서브 3 |
| [pdf/all_projects_portfolio.pdf](pdf/all_projects_portfolio.pdf) | 백엔드 4개 프로젝트 13페이지 통합본 (12.6MB) |
| [pdf/profile_summary.pdf](pdf/profile_summary.pdf) | 이력서 2페이지 — 소개·주요 기술·학력·교육 / 수상·자격증·링크 |
| [pdf/sub_projects_portfolio.pdf](pdf/sub_projects_portfolio.pdf) | 서브/AI 3페이지 — Travel Quest, YOLO Detect, FastAPI 실습 |
| [pdf/agzak_portfolio.pdf](pdf/agzak_portfolio.pdf) | 아그작 — 개요·아키텍처·랭킹 이원화·운영 성과 4페이지 |
| [pdf/toktokggubi_portfolio.pdf](pdf/toktokggubi_portfolio.pdf) | 똑똑꺼비 — 개요·OCR 파이프라인·OAuth2 딥링크 3페이지 |
| [pdf/ttomanito_portfolio.pdf](pdf/ttomanito_portfolio.pdf) | 또마니또 — 개요·WebSocket Principal·3DB + AI 3페이지 |
| [pdf/bookstore_portfolio.pdf](pdf/bookstore_portfolio.pdf) | 이조핑 북스토어 — 개요·검색 엔진 도입·한글 분석기 3페이지 |

페이지에서 쓰는 스택 아이콘은 `skill_icon/`(png·svg), 프로젝트 스크린샷과 아키텍처·모니터링 캡처는 각 프로젝트 폴더의 `img/`에 있습니다.

---

## 🪪 자격증 · 어학

| 구분 | 자격증 / 시험 | 발급 기관 | 취득일 |
|---|---|---|---|
| 자격증 | 정보처리기사 | 한국산업인력공단 | 2025.06.13 |
| 자격증 | SQLD | 한국데이터산업진흥원 | 2023.07.07 |
| 자격증 | ADsP | 한국데이터산업진흥원 | 2026.06.05 |
| 자격증 | 리눅스마스터 2급 | 한국정보통신진흥협회 | 2026.07.03 |
| 자격증 | TOPCIT — 3수준 | 정보통신기획평가원 | 2024.10.12 |
| 자격증 | 한국사능력검정 2급 | 국사편찬위원회 | 2026.10.30 |
| 어학 | TOEIC Speaking — IM3 | YBM한국TOEIC위원회 | 2025.10.11 |

## 🏆 수상

| 수상일 | 대회 | 수상 | 수여 기관 |
|---|---|---|---|
| 2025.12.18 | 삼성 청년 SW·AI 아카데미 — 성적 부문 | 우수상 | 고용노동부 |
| 2025.11.20 | 삼성 청년 SW·AI 아카데미 — 자율 프로젝트 | 우수상 (2등) | 삼성전자 |
| 2025.09.29 | 삼성 청년 SW·AI 아카데미 — 특화 프로젝트 | 우수상 (1등) | 삼성전자 |
| 2025.02.03 | KSC2024 학부생/주니어논문경진대회 | 장려상 | 한국정보과학회 |
| 2024.11.22 | CNU 창의SW/AI 축전 DevDay 코딩경진대회 | 장려상 | 충남대학교 컴퓨터융합학부 |
| 2023.11.10 | CNU SW 아카데미 프로젝트 경진대회 | 우수상 | 충남대학교 정보화본부 |

## 🎓 교육

| 기간 | 기관 | 과정 |
|---|---|---|
| 2025.01 – 2025.12 | 삼성 청년 SW·AI 아카데미 (SSAFY) 13기 | Java 전공자반 |
| 2024.07 – 2024.12 | NHN 아카데미 7기 | Backend (Java·Spring Boot·MySQL) |
| 2023.07 – 2023.12 | CNU SW 아카데미 3기 | Frontend (HTML·CSS·JS·React) |
| 2019.03 – 2025.02 | 충남대학교 | 컴퓨터융합학부 (컴퓨터공학) |

---

## 📁 레포지토리 구조

```
portfolio/
├── index.html                          # 포트폴리오 메인 (GitHub Pages 진입점)
├── pdf/
│   ├── full_portfolio.pdf              # 전체 18페이지 (이력서 2 + 백엔드 13 + 서브 3)
│   ├── all_projects_portfolio.pdf      # 백엔드 4개 프로젝트 13페이지 통합본
│   ├── profile_summary.pdf             # 이력서 2페이지
│   ├── sub_projects_portfolio.pdf      # 서브/AI 3페이지
│   ├── agzak_portfolio.pdf
│   ├── ttomanito_portfolio.pdf
│   ├── toktokggubi_portfolio.pdf
│   └── bookstore_portfolio.pdf
├── 00_profile_summary_assets/
│   ├── html/                           # 이력서 (2페이지)
│   └── img/profile_photo.jpg            # 증명사진
├── 01_bookstore_portfolio_assets/
│   ├── html/                           # 개요 · 검색 엔진 · 한글 분석기 (3페이지)
│   └── img/                            # 검색 화면 스크린샷
├── 02_ttomanito_portfolio_assets/
│   ├── html/                           # 개요 · WebSocket Principal · 3DB + AI (3페이지)
│   └── img/                            # 채팅·미션 화면 스크린샷
├── 03_toktokggubi_portfolio_assets/
│   ├── html/                           # 개요 · OCR 파이프라인 · OAuth2 딥링크 (3페이지)
│   └── img/                            # 예산·기프티콘 화면 스크린샷
├── 04_agzak_portfolio_assets/
│   ├── html/                           # 개요 · 아키텍처 · 랭킹 · 운영 (4페이지)
│   └── img/                            # 아키텍처 다이어그램, GA4·Grafana·Jenkins 캡처
├── 05_sub_projects_assets/
│   ├── html/                           # Travel Quest (1p) / YOLO Detect (1p) / FastAPI 실습 (1p)
│   └── img/                            # 서비스 화면 캡처, YOLO 예측·학습 곡선
└── skill_icon/                         # 페이지용 기술 스택 아이콘 (png, svg)
```

## 🌿 브랜치

| 브랜치 | 내용 |
|---|---|
| `main` | 포트폴리오 사이트(index.html·페이지·PDF). 이 문서의 내용은 모두 `main` 기준입니다. |
| [`interview`](https://github.com/Ori-Gui/portfolio/tree/interview/interview) | 프로젝트 4건의 실제 코드를 근거로 정리한 면접 답변 가이드 4건 — 인증 도메인 비교, 랭킹·멱등, JUnit5 + Mockito, 모바일 OAuth2 딥링크 |

---

## 📬 Contact

<a href="https://ori-gui.github.io/portfolio/" target="_blank">
  <img src="https://img.shields.io/badge/Portfolio-2563EB?style=flat-square&logo=googlechrome&logoColor=white" alt="Portfolio">
</a>
<a href="https://velog.io/@ori_gui/posts" target="_blank">
  <img src="https://img.shields.io/badge/Velog-20C997?style=flat-square&logo=velog&logoColor=white" alt="Velog">
</a>
<a href="mailto:dign552@naver.com" target="_blank">
  <img src="https://img.shields.io/badge/Email-EA4335?style=flat-square&logo=gmail&logoColor=white" alt="Email">
</a>

© 2026 Ori-Gui · yangjunha · dign552@naver.com
