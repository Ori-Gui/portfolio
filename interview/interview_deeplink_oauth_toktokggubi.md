# 면접 답변 가이드 — 똑똑꺼비 모바일 OAuth2 + 커스텀 스킴 딥링크

> 작성: 2026-05-10 / 검증 기준: 03_toktokggubi 실제 코드 (백엔드 `KakaoAuthController`·`KakaoOAuthClient`·`KakaoAuthService` + 프론트 `kakaoAuthService.ts` + `app.config.js`)
> 대상: 외부 IdP 연동·모바일 딥링크·OAuth2 흐름 깊이 질문 대응
> 시리즈: [interview_auth_4projects.md](./interview_auth_4projects.md) · [interview_junit5_mockito_agzak.md](./interview_junit5_mockito_agzak.md)

---

## 0. 코드 사실관계 한눈에

| 주장 | 파일 | 핵심 |
|---|---|---|
| 커스텀 스킴 정의 (Expo) | `frontend/app.config.js:8` | `scheme: "frontend"` (slug 기반 자동 스킴이 `toktokggubi://` 추가 작동) |
| 실제 사용 스킴 | `kakaoAuthService.ts:9` | `APP_RETURN_URL = "toktokggubi://oauth/kakao"` |
| Android 패키지 | `app.config.js:33` | `com.technologia.toktokggubi` |
| 카카오 authorize URL 빌드 | `KakaoAuthController.authorizeUrl` | `kauth.kakao.com/oauth/authorize` + PKCE optional |
| 카카오 → 백엔드 콜백 | `KakaoAuthController.callback` | code → token 교환 → 자체 JWT 발급 → 앱으로 딥링크 |
| 토큰 교환 | `KakaoOAuthClient.exchangeCodeForToken` | `kauth.kakao.com/oauth/token`, `code_verifier` 옵션 |
| 앱 복귀 트리거 (HTML) | `KakaoAuthController.deepLinkHtml` | **meta refresh + JS intent URL + 수동 링크 3중 fallback** |
| 앱 측 딥링크 수신 (이중 대기) | `kakaoAuthService.ts:46-65` | `Linking.addEventListener` + `WebBrowser.openAuthSessionAsync` 둘 중 먼저 도착한 것 사용 |
| 사용자 상태 분기 | `KakaoAuthService.login` | ACTIVE → access/refresh / INACTIVE → signupToken (추가정보 단계) |

---

## 1. 왜 모바일 앱 OAuth2가 웹보다 어려운가

### 웹 OAuth2 흐름 (간단함)

```
브라우저 ──▶ /oauth/authorize ──▶ IdP 로그인 ──▶ /callback?code=xxx ──▶ 같은 브라우저 안에서 끝
```

브라우저 한 컨텍스트 안에서 모든 단계가 끝납니다. 쿠키도 그대로 따라옵니다.

### 모바일 앱 OAuth2 흐름 (복잡함)

```
앱 ──▶ 외부 브라우저(Chrome/Safari) ──▶ IdP 로그인 ──▶ 콜백 도착... 그런데 결과를 앱으로 어떻게 돌려주지?
```

**핵심 문제 3가지**:

1. **컨텍스트 전환**: 앱 → 외부 브라우저 → 다시 앱. 브라우저는 앱과 다른 프로세스라 메모리·쿠키 공유 안 됨
2. **카카오 IdP가 모르는 것**: 카카오는 우리 앱의 존재를 모름. 카카오가 리다이렉트할 수 있는 곳은 **HTTPS URL**뿐 — 우리 앱으로 직접 못 보냄
3. **앱을 어떻게 깨우지**: 브라우저에서 우리 앱을 다시 띄울 수단이 필요 → **커스텀 스킴 딥링크**

### 그래서 일반적인 모바일 OAuth2는 이렇게 풀린다

```
앱 ──▶ Chrome Custom Tab ──▶ 카카오 로그인
                              │
                              ▼
                          우리 백엔드 /callback (HTTPS, 카카오가 도달 가능)
                              │
                              │ ① code → token 교환
                              │ ② 자체 JWT 발급
                              │ ③ 커스텀 스킴 URL로 응답 ───▶ 브라우저 → 앱 깨움
                              ▼
                     toktokggubi://oauth/kakao?token=...
```

본인 코드가 정확히 이 패턴입니다.

---

## 2. 커스텀 스킴이란?

### 정의

URL의 가장 앞 `https://`, `http://` 자리에 들어가는 프로토콜 이름을 **스킴(Scheme)** 이라고 합니다. OS는 *"이 스킴이 들어오면 어느 앱을 띄울지"* 를 매핑 테이블로 관리합니다.

| 스킴 | 매핑 |
|---|---|
| `https://` | 시스템 기본 브라우저 |
| `mailto:` | 메일 앱 |
| `tel:` | 전화 앱 |
| `toktokggubi://` | **앱이 OS에 등록한 커스텀 스킴 → 우리 똑똑꺼비 앱** |

브라우저(또는 다른 앱)에서 `toktokggubi://oauth/kakao?token=xxx`를 열면 → Android/iOS가 *"등록된 앱"* 을 찾아서 그 앱에 URL을 전달하며 깨웁니다.

### 본인 코드의 스킴 등록 위치

```javascript
// frontend/app.config.js:8
expo: {
    name: "toktokggubi",
    slug: "toktokggubi",          // ★ Expo가 slug 기반으로 자동 스킴 등록
    scheme: "frontend",            // ← 추가 스킴 (개발용)
    // ...
    android: {
        package: "com.technologia.toktokggubi",   // ★ Android 패키지명
    }
}
```

> **약간의 사실 인지**: `scheme: "frontend"` 한 줄만으로는 `toktokggubi://`가 작동하지 않습니다. Expo SDK는 slug(`toktokggubi`)를 기본 deep link 스킴으로 자동 추가하는 동작이 있고, 본인 코드가 그걸 의도해 사용하는 것으로 추정됩니다. 면접에서 *"app.config.js에서 scheme이 'frontend'인데 어떻게 toktokggubi://를 쓰나요?"* 같은 질문이 들어오면 *"Expo slug 기반 자동 스킴 매핑을 활용했고, 'frontend'는 개발 빌드용으로 별도 추가한 것"* 정도로 답하는 게 안전합니다.

---

## 3. 전체 시퀀스 — 정상 로그인 한 사이클

```
[앱: KakaoAuthService.login()]
    │
    │ ① fetch GET /api/auth/kakao/authorize-url
    │ ───────────────────────────────────────────────▶ [백엔드: KakaoAuthController.authorizeUrl]
    │                                                   - kauth.kakao.com/oauth/authorize 빌드
    │ { url: "https://kauth.kakao.com/oauth/authorize?  - client_id, redirect_uri,
    │        client_id=...&redirect_uri=...&             response_type=code, scope, (PKCE)
    │        response_type=code&scope=profile_nickname"}
    │ ◀───────────────────────────────────────────────
    │
    │ ② const sub = Linking.addEventListener("url", ...);
    │    WebBrowser.openAuthSessionAsync(url, "toktokggubi://oauth/kakao", {...});
    │                            │
    │                            ▼
    │                   ┌────────────────────────────┐
    │                   │ Chrome Custom Tab / Safari │
    │                   │ (외부 브라우저)             │
    │                   │  카카오 로그인 페이지        │
    │                   │  사용자: ID/PW 입력 + 동의   │
    │                   └────────────┬───────────────┘
    │                                │
    │                                │ ③ POST /oauth/authorize 동의 완료
    │                                │   카카오가 redirect_uri로 302 보냄
    │                                ▼
    │                       [카카오 → 백엔드 콜백]
    │                       GET /api/auth/kakao/callback?code=xxx&state=...
    │                                │
    │                                │ ④ 백엔드 처리:
    │                                │   - exchangeCodeForToken(code) → 카카오 access_token
    │                                │   - kakaoApi.me(accessToken)  → 카카오 사용자 정보
    │                                │   - service.login(me)          → 자체 JWT 발급
    │                                │   - 딥링크 URL 빌드:
    │                                │       toktokggubi://oauth/kakao?token=...&refreshToken=...&userId=...
    │                                ▼
    │                                ⑤ HTML 응답 (meta refresh + JS intent + 수동 링크 3중 fallback)
    │                                │
    │                                ▼
    │                       브라우저가 toktokggubi:// 를 열려고 시도
    │                                │
    │                                │ ⑥ Android: package=com.technologia.toktokggubi 매칭
    │                                │            → 우리 앱 깨움 + URL 전달
    │                                │    iOS: scheme=toktokggubi 매칭
    │                                ▼
    │ ⑦ Linking 이벤트 발화 OR WebBrowser.openAuthSessionAsync result.url
    │    중 먼저 도착한 것 사용
    │
    │ ⑧ parseDeepLinkParams(finalUrl) → { token, refreshToken, userId, ... }
    │ ⑨ 로컬 저장 + 인증 상태 전환
    │
    └─▶ login() 함수 종료, 결과 반환
```

이 한 흐름에 본인 코드의 모든 영리한 부분이 들어있습니다.

---

## 4. 백엔드의 핵심 — HTML 3중 Fallback

[KakaoAuthController.java:40-50](03_toktokggubi/toktokggubi/backend/src/main/java/com/ssafy/toktokggubi/domain/auth/controller/KakaoAuthController.java#L40-L50)

```java
private static String deepLinkHtml(URI deep) {
    String intentUrl = "intent://oauth/kakao#Intent;scheme=toktokggubi;package=com.technologia.toktokggubi;end";
    return """
            <!doctype html><html><head><meta charset="utf-8">
            <meta http-equiv="refresh" content="0; url=%s">
            <script>setTimeout(function(){ location.href = '%s'; }, 150);</script>
            </head><body>
            <p>앱으로 이동 중… 이동하지 않으면 <a href="%s">여기</a>를 누르세요.</p>
            </body></html>
            """.formatted(deep.toString(), intentUrl, deep.toString());
}
```

### 왜 그냥 `302 Redirect Location: toktokggubi://...` 안 쓰나?

이게 정말 자주 묻는 면접 질문입니다.

**문제**: Chrome Custom Tab(Android)이나 Safari(iOS)가 **HTTP 응답의 Location 헤더로 커스텀 스킴을 받으면 보안상 차단**하는 경우가 있습니다. 특히:
- Chrome은 HTTP/HTTPS가 아닌 스킴으로의 302 리다이렉트를 종종 거부
- HTTPS → 커스텀 스킴은 *"안전한 컨텍스트에서 안전하지 않은 컨텍스트로 이동"* 으로 간주

그래서 *"HTML 페이지를 일단 보여주고, 그 페이지가 자체적으로 커스텀 스킴을 트리거하게 만드는"* 우회가 필요합니다.

### 3중 Fallback 분석

| Fallback | 대상 OS/브라우저 | 동작 |
|---|---|---|
| ① `<meta http-equiv="refresh" content="0; url=toktokggubi://...">` | iOS Safari, 대부분 Android Chrome | 페이지 로드 즉시 커스텀 스킴 트리거 (0초) |
| ② `<script>setTimeout(..., 150ms);</script>` + **Intent URL** | Android Chrome 일부 버전 (meta refresh 차단 시) | 150ms 후 `intent://...#Intent;scheme=...;package=...;end` 형태로 재시도. Intent URL은 Android Chrome이 공식 지원하는 *"앱 열기"* 표준 |
| ③ `<a href="toktokggubi://...">여기</a>` | 자동 트리거 실패 시 | 사용자가 직접 링크 클릭 (최후 안전망) |

**Intent URL의 위력**: Android Chrome은 `intent://` 형태 URL을 만나면 — 매니페스트의 `package` 필드를 확인해 정확히 그 앱을 깨우거나, 없으면 Play 스토어로 안내합니다. 보안과 UX 모두 챙기는 표준입니다.

---

## 5. 프론트엔드의 핵심 — 이중 복귀 경로 대기

[kakaoAuthService.ts:43-67](03_toktokggubi/toktokggubi/frontend/src/shared/services/kakaoAuthService.ts#L43-L67)

```typescript
// A) Linking 이벤트로도 동시에 대기
let deepLinkFromEvent: string | null = null;
const sub = Linking.addEventListener("url", ({ url }) => {
    if (!resolved && url.startsWith(APP_RETURN_URL)) {
        deepLinkFromEvent = url;
        resolved = true;
    }
});

try {
    // B) 브라우저 세션 시작
    const result = await WebBrowser.openAuthSessionAsync(url, APP_RETURN_URL, {
        showInRecents: false,
        preferEphemeralSession: false,
    });

    // C) 어떤 경로든 먼저 도착한 URL 사용
    const finalUrl =
        deepLinkFromEvent ??
        (result.type === "success" && result.url?.startsWith(APP_RETURN_URL) ? result.url : null);

    if (!finalUrl) {
        throw new Error("앱으로 돌아오지 못했습니다. (딥링크 미수신)");
    }
    // ...
}
```

### 왜 두 경로를 동시에 대기하나?

**`WebBrowser.openAuthSessionAsync` 단독 사용 시의 문제**:
- iOS의 `ASWebAuthenticationSession`은 result.url을 잘 반환
- Android의 `Custom Tab + Trusted Web Activity` 조합은 OS·버전에 따라 result.url을 **반환하지 못하고 그냥 `type: "cancel"`로 끝남**
- 그런데 사실은 딥링크가 이미 앱을 깨웠고 → `Linking` 이벤트가 발화한 상태

**`Linking.addEventListener` 단독 사용 시의 문제**:
- 사용자가 브라우저를 그냥 닫아버리면(취소) Linking 이벤트도 안 옴
- 무한히 대기하는 함수가 됨

**해결**: 두 경로를 동시에 listen하고 **둘 중 먼저 도착한 것**을 사용. 어느 한쪽이 실패해도 다른 쪽이 받아줌.

```typescript
const finalUrl = deepLinkFromEvent ??     // ① Linking 이벤트가 먼저 도착했으면 그것
    (result.type === "success" && ...)    // ② WebBrowser result.url이 도착했으면 그것
    ?? null;                              // ③ 둘 다 실패면 null → 에러
```

이게 *"운영 환경의 변수를 제어한다"* 의 모바일 버전입니다 — 플랫폼·OS 버전마다 동작이 미묘하게 달라서 한 가지 경로만 믿으면 깨집니다.

---

## 6. 자체 JWT 발급 분기 — ACTIVE vs INACTIVE

[KakaoAuthService.java:43-87](03_toktokggubi/toktokggubi/backend/src/main/java/com/ssafy/toktokggubi/domain/auth/service/KakaoAuthService.java#L43-L87)

```java
@Transactional
public KakaoLoginResponse login(KakaoMe me) {
    User user = userRepository.findByKakaoSub(String.valueOf(kakaoId)).orElse(null);

    if (user == null) {
        // ① 신규 — INACTIVE 상태로 우선 저장
        user = userRepository.save(User.builder()
                .kakaoSub(String.valueOf(kakaoId))
                .status(UserStatus.INACTIVE)
                ...
                .build());
    } else if (user.getStatus() == UserStatus.WITHDRAWN) {
        // ② 탈퇴 회원 재가입 — INACTIVE로 복귀
        user = userRepository.save(user.toBuilder()
                .status(UserStatus.INACTIVE)...
                .build());
    }

    financeLinkService.ensureFinanceUserKey(user);

    if (user.getStatus() == UserStatus.ACTIVE) {
        // ③ 정식 회원 — 액세스/리프레시 발급
        TokenEnvelope token = issueTokens(user.getUserId());
        return KakaoLoginResponse.builder()...token(token).build();
    } else {
        // ④ 추가정보 미입력 — signupToken만 발급
        String signupToken = jwtProvider.createSignupToken(user.getUserId());
        return KakaoLoginResponse.builder()...signupToken(signupToken).build();
    }
}
```

### 3-tier 사용자 상태 모델

| 상태 | 의미 | 발급 토큰 |
|---|---|---|
| `INACTIVE` | 카카오 인증은 됐지만 닉네임·성별 등 **앱 자체 추가정보 미입력** | `signupToken`만 |
| `ACTIVE` | 추가정보까지 완료된 정식 회원 | `accessToken + refreshToken` |
| `WITHDRAWN` | 탈퇴 회원 (재가입 시 INACTIVE로 복귀) | — |

### signupToken의 의미

- access/refresh와 분리된 **단계 전용 토큰**
- 추가정보 입력 화면에서만 유효
- 추가정보 제출 완료 시점에 ACTIVE로 전환 + 정식 JWT 발급

이 분기를 딥링크 쿼리 파라미터로 어떻게 표현하는지 KakaoAuthController.callback:94-108에서 보입니다:

```java
deepB.queryParam("token", res.token().accessToken())          // ACTIVE 분기
     .queryParam("refreshToken", res.token().refreshToken())
// 또는
deepB.queryParam("signupToken", res.signupToken());             // INACTIVE 분기
```

프론트 `kakaoAuthService.ts:75-76`에서 그 분기를 받아냅니다:

```typescript
if (token) return { status: "active", userId, accessToken: token, refreshToken };
return { status: "needs-profile", userId, signupToken: signupToken! };
```

---

## 7. 보안 고려 — PKCE / state / 스킴 위·변조 위험

### PKCE 지원 (Optional)

[KakaoAuthController.java:66-69](03_toktokggubi/toktokggubi/backend/src/main/java/com/ssafy/toktokggubi/domain/auth/controller/KakaoAuthController.java#L66-L69)

```java
if (codeChallenge != null && !codeChallenge.isBlank()) {
    b.queryParam("code_challenge", codeChallenge)
     .queryParam("code_challenge_method", "S256");
}
```

[KakaoOAuthClient.java:34](03_toktokggubi/toktokggubi/backend/src/main/java/com/ssafy/toktokggubi/domain/auth/client/KakaoOAuthClient.java#L34)

```java
if (codeVerifier != null) form.add("code_verifier", codeVerifier);
```

**PKCE(Proof Key for Code Exchange)란**:
- 클라이언트가 `code_verifier`(랜덤 문자열) 만들고 → SHA256으로 해시한 `code_challenge`를 authorize 요청에 첨부
- token 교환 시 원본 `code_verifier`를 같이 보내 IdP가 일치 검증
- **목적**: authorization code가 중간에 탈취돼도, code_verifier 없이는 토큰 교환 불가

### 본인 구현의 한계 — 정직하게

코드를 보면 `@RequestParam(required = false, name = "code_verifier") String codeVerifier`로 콜백에서 받지만, **카카오는 콜백 시 code_verifier를 다시 보내주지 않습니다** (그건 클라이언트만 보관하는 비밀). 즉 백엔드가 토큰 교환을 대신 해주는 본인 구조에서는 **PKCE가 실질적으로 동작하지 않습니다**.

PKCE가 의미가 있으려면 클라이언트(앱)가 직접 카카오와 토큰 교환을 해야 하는데, 그러려면 카카오 client secret이 앱에 박혀야 해서 또 다른 보안 문제가 생깁니다. 이 트레이드오프는 모바일 OAuth2의 본질적 어려움입니다.

> **면접 답변**: *"PKCE 코드를 지원 가능한 형태로 짜뒀지만, 백엔드가 토큰 교환을 대행하는 본인 구조에서는 code_verifier가 백엔드까지 도달하지 못해 PKCE 효과가 제한적입니다. 정통 PKCE는 클라이언트가 직접 토큰 교환을 해야 의미가 있는데, 그러려면 client secret이 앱에 박혀야 하는 다른 위험이 생깁니다. 본인 프로젝트는 BFF(Backend For Frontend) 패턴으로 가닥을 잡되 PKCE 자리는 코드에 남겨둬 향후 확장 가능성을 열어둔 상태입니다."*

### state 파라미터

```java
if (state != null && !state.isBlank()) {
    b.queryParam("state", state);
}
```

CSRF 방지용으로 state를 흘려주는 자리는 있지만, **콜백에서 state를 검증하는 코드는 없습니다**. 이것도 정직하게 인정 포인트.

### 커스텀 스킴 위·변조 위험

**가장 큰 약점**: 다른 악성 앱이 똑같은 `toktokggubi://` 스킴을 자기 매니페스트에 등록할 수 있습니다. Android는 스킴 충돌 시 사용자 선택 다이얼로그를 띄우지만, **실수로 악성 앱을 선택하면 토큰이 그쪽으로 전달**됩니다.

**더 안전한 대안**: **Android App Links / iOS Universal Links** — HTTPS URL을 딥링크로 등록하면서 도메인 소유 증명(`.well-known/assetlinks.json`)을 통해 OS가 *"이 도메인은 이 앱만 처리"* 를 강제. 본인 프로젝트에서는 미적용.

> **면접 답변**: *"커스텀 스킴은 다른 앱이 같은 스킴을 등록할 수 있다는 위험이 있어 운영에서는 Android App Links / iOS Universal Links로 옮기는 게 표준입니다. 본인은 자율 프로젝트 일정상 커스텀 스킴까지만 적용했고, 사내 SSO처럼 보안이 더 중요한 환경이라면 도메인 소유 증명 기반 딥링크로 즉시 마이그레이션할 것입니다."*

---

## 8. 본인 구현의 정직한 약점 (자가 점검)

| 약점 | 위험 | 면접용 답변 톤 |
|---|---|---|
| PKCE 실질 미작동 | 토큰 교환 path 보안 약화 | *"BFF 패턴 채택의 트레이드오프, 향후 확장 자리만 남겨둠"* |
| state 검증 누락 | CSRF 공격 가능성 | *"발급은 했지만 검증 미구현, 후속 개선 대상"* |
| 커스텀 스킴 vs App Links | 다른 앱이 스킴 가로채기 가능 | *"자율 프로젝트 일정상 1차 적용, 운영용은 App Links 권장"* |
| signupToken을 URL 쿼리로 전달 | 브라우저 히스토리·로그·HTML 캐시에 토큰 남을 위험 | *"단기 만료라 위험 제한적이지만, POST body로 옮기는 게 표준"* |
| `app.config.js` scheme 불명확 | scheme: "frontend"인데 toktokggubi:// 사용 | *"Expo slug 기반 자동 스킴이 함께 작동했고, 정리되지 않은 부분"* |

이 약점들을 *"몰랐다"* 가 아니라 *"의식한 트레이드오프"* 로 답하면 면접관에게 *"운영 보안 의식이 있다"* 는 시그널이 됩니다.

---

## 9. 면접 시뮬레이션

### Q1. *"모바일 OAuth2가 웹과 어떻게 다른가요?"*

> *"세 가지 핵심 차이가 있습니다. ① 컨텍스트 전환 — 앱에서 외부 브라우저로 갔다가 다시 앱으로 돌아와야 하는데 둘은 다른 프로세스라 메모리·쿠키 공유가 안 됩니다. ② 카카오 같은 IdP는 우리 앱을 모르고 HTTPS URL로만 리다이렉트할 수 있어서, 우리 백엔드가 중개를 해야 합니다. ③ 앱을 다시 깨우려면 커스텀 스킴 딥링크 또는 App Links가 필요합니다. 03_toktokggubi에서는 토큰 교환은 백엔드가 BFF로 처리하고, 결과를 toktokggubi://oauth/kakao 딥링크로 앱에 전달하는 구조를 채택했습니다."*

### Q2. *"왜 백엔드에서 302 Redirect로 커스텀 스킴 안 쓰고 HTML 페이지를 응답했나요?"*

> *"Chrome Custom Tab과 Safari가 HTTPS 응답의 Location 헤더로 비-HTTP 스킴을 받으면 보안상 차단하는 경우가 있어서입니다. 그래서 일단 HTML 페이지를 보여주고, 그 페이지가 자체적으로 커스텀 스킴을 트리거하게 만들었습니다. 3중 fallback을 뒀는데 meta refresh 즉시 트리거, 150ms 후 Android intent URL fallback, 그리고 사용자 수동 클릭용 anchor 링크 순서입니다. Android Chrome의 intent URL은 매니페스트의 패키지명을 명시할 수 있어 정확히 우리 앱을 깨우는 표준 방식입니다."*

### Q3. *"앱 쪽에서는 Linking 이벤트와 WebBrowser.openAuthSessionAsync 둘을 동시에 쓰던데 왜죠?"*

> *"플랫폼·OS 버전마다 어느 한쪽이 실패할 수 있어서입니다. iOS는 ASWebAuthenticationSession이 result.url을 잘 반환하지만, Android Chrome Custom Tab은 OS 버전에 따라 result.url 없이 type:'cancel'로 끝나면서 실제로는 딥링크가 이미 발화한 상태가 됩니다. 그래서 두 경로를 동시에 listen하고 둘 중 먼저 도착한 URL을 사용하는 패턴으로 짰습니다. 운영 환경의 플랫폼 변수를 코드로 흡수하는 사례입니다."*

### Q4. *"PKCE를 적용했다고 했는데 실제로 동작하나요?"*

> *"솔직히 말씀드리면 코드 자리는 마련해뒀지만 실질적으로는 동작하지 않습니다. 백엔드가 토큰 교환을 대행하는 BFF 패턴이라 code_verifier가 백엔드까지 도달하지 못합니다. 정통 PKCE는 클라이언트가 직접 토큰 교환을 해야 의미가 있는데, 그러려면 client secret이 앱에 박혀야 해서 다른 위험이 생깁니다. 트레이드오프 끝에 BFF로 가닥을 잡았고, PKCE 자리는 향후 client_secret-less 흐름으로 가게 됐을 때를 위한 확장점으로 남겨뒀습니다."*

### Q5. *"커스텀 스킴이면 다른 앱이 가로챌 수 있지 않나요?"*

> *"맞습니다. Android는 같은 스킴을 등록한 앱이 여러 개면 사용자 선택 다이얼로그를 띄우는데, 사용자가 잘못 선택하면 토큰이 악성 앱으로 흘러갑니다. 운영에서 안전한 방식은 Android App Links / iOS Universal Links로 — HTTPS URL을 딥링크로 쓰면서 .well-known/assetlinks.json으로 도메인 소유를 증명하는 방식입니다. OS가 그 도메인은 우리 앱만 처리하도록 강제합니다. 03_toktokggubi는 5주 자율 프로젝트 일정상 커스텀 스킴까지만 적용했고, 엔터프라이즈 SSO 환경이라면 즉시 App Links로 옮길 만한 항목으로 인지하고 있습니다."*

### Q6. *"signupToken과 accessToken을 굳이 분리한 이유는?"*

> *"권한이 다르기 때문입니다. signupToken은 '카카오 인증은 완료됐지만 추가정보 미입력' 상태에서만 사용 가능한 단계 전용 토큰입니다. 추가정보 입력 API에만 통과하고, 다른 비즈니스 API는 거부됩니다. 추가정보 제출이 끝나는 시점에 사용자 상태를 ACTIVE로 바꾸고 정식 access·refresh를 발급하는 흐름입니다. 토큰의 권한 스코프를 사용자 생명주기 단계와 맞춘 설계입니다."*

---

## 10. 한 줄 무기

| 상황 | 한 줄 |
|---|---|
| 모바일 OAuth2 본질 | *"앱과 외부 브라우저는 다른 프로세스라 카카오의 응답을 어떻게 다시 앱으로 돌려보낼지가 본질적 문제 — 그 다리가 커스텀 스킴 딥링크입니다."* |
| BFF 패턴 | *"백엔드가 토큰 교환을 대행하면 client secret이 앱에 박히지 않아 안전하지만, PKCE 효과는 제한적 — 트레이드오프입니다."* |
| HTML 3중 fallback | *"302로 커스텀 스킴 리다이렉트는 브라우저가 차단할 수 있어 HTML + meta refresh + intent URL + 수동 링크 4중 안전망이 표준입니다."* |
| 이중 복귀 대기 | *"Linking 이벤트와 WebBrowser result.url 둘 중 먼저 도착한 것을 쓰는 게 플랫폼 변수를 흡수하는 방법입니다."* |
| 스킴 보안 | *"커스텀 스킴은 다른 앱이 가로챌 수 있어, 운영급은 App Links / Universal Links로 도메인 소유 증명 기반 딥링크가 표준입니다."* |
| 단계 전용 토큰 | *"signupToken은 추가정보 입력 단계에만 유효한 권한 제한 토큰 — 토큰 스코프와 사용자 생명주기를 맞춘 설계입니다."* |

---

## 11. 외울 핵심 4문장

1. **"모바일 OAuth2는 앱 → 외부 브라우저 → IdP → 백엔드 → 딥링크 → 앱의 5단계, 그 다리가 커스텀 스킴"**
2. **"커스텀 스킴 트리거는 HTML + meta refresh + intent URL + 수동 링크 4중 fallback"**
3. **"앱 복귀 대기는 Linking 이벤트 + WebBrowser result.url 두 경로 동시 listen"**
4. **"운영급 보안은 App Links / Universal Links로 도메인 소유 증명 기반 딥링크"**

이 4문장이면 모바일 OAuth2 질문 95%는 받아칠 수 있고, 본인 코드의 약점도 *"트레이드오프 인지"* 톤으로 답할 수 있습니다.
