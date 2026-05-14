# Supabase + Google OAuth — 두 단계 콜백 구조와 redirect URI 두 개의 의미

> 작성일: 2026-05-13
> 태그: #개념정리 #supabase #oauth #nextjs #google-auth
> 출발점: 우문현답 프로젝트에 소셜 로그인(Google) 붙이는 작업 중 `redirect_uri_mismatch`, `missing_code`, `Unable to exchange external code` 에러를 차례로 마주침
> 원본 기록: [woomun-hyundap CLAUDE.md](../../Git/woomun-hyundap/CLAUDE.md)

## 한 줄 요약

Supabase Auth로 Google 로그인을 붙일 땐 **콜백이 두 번 일어난다** — Google → Supabase, Supabase → 우리 앱. 그래서 redirect URI도 두 개 등록해야 하고, 그 둘은 서로 다른 역할을 한다.

## 배경 지식

### OAuth 2.0 Authorization Code Flow with PKCE (Supabase 기본값)

OAuth 표준 흐름은 단순히 "Google에 로그인하면 끝"이 아니다. 실제로는 다음 단계로 쪼개진다:

1. 사용자가 우리 앱에서 "Google 로그인" 클릭
2. 우리 앱이 사용자를 인증 페이지로 보냄 (`redirect_uri` 파라미터 포함). 이때 **code verifier**를 로컬에 저장하고, code challenge를 함께 보냄 (PKCE)
3. 사용자가 Google에 로그인 + 권한 동의
4. Google이 사용자를 `redirect_uri`로 돌려보냄. 이때 URL에 **`code`** 파라미터를 붙여 보냄
5. 우리 서버가 그 `code`와 **저장해뒀던 code verifier를 함께** 토큰 엔드포인트에 보내 토큰 교환 → access_token / refresh_token 발급
6. 토큰으로 사용자 정보 조회 → 우리 앱 세션 생성

핵심: **`code`는 그 자체로 로그인이 아니다.** code를 토큰으로 "교환"하는 별도 단계가 있다.

**PKCE가 추가하는 것**: code verifier — 흐름 ②에서 만들어 로컬 저장, ⑤에서 함께 보냄. code가 중간에 탈취돼도 verifier가 없으면 토큰 교환 못 함. 그래서 SPA·모바일에서도 안전.

**중요한 제약**:
- `code`는 **5분 만료**, **한 번만 사용 가능**
- 그래서 `exchangeCodeForSession`을 두 번 호출하거나 오래 묵혀두면 실패함
- (참고: code verifier는 `pkce_code_verifier` 쿠키나 localStorage에 들어감)

### Supabase가 끼면 콜백이 두 번이 된다

Supabase Auth를 쓰면 위 흐름이 이렇게 바뀐다:

```
사용자
  ↓ ① "Google 로그인" 클릭
우리 앱 (localhost:3000)
  ↓ ② supabase.auth.signInWithOAuth() 호출 → Supabase로 redirect
Supabase (tikycotngtobngokgknr.supabase.co)
  ↓ ③ Google로 redirect (redirect_uri = supabase URL)
Google
  ↓ ④ 사용자 로그인·동의 후 code 발급
  ↓ ⑤ Supabase로 redirect (with code)
Supabase
  ↓ ⑥ code를 Google에 보내 토큰 교환 (Client Secret 사용)
  ↓ ⑦ 우리 앱으로 redirect (with code — 이번엔 Supabase가 발급한 code)
우리 앱 /api/auth/callback
  ↓ ⑧ exchangeCodeForSession(code) — Supabase에 code 보내 세션 쿠키 받기
  ↓ ⑨ /dashboard로 redirect
```

→ **콜백이 두 번**이다. Google → Supabase, Supabase → 우리 앱.

그래서 redirect URI도 두 곳에 등록해야 한다:

| 등록 위치 | URL | 역할 |
|---|---|---|
| **Google Cloud Console** | `https://tikycotngtobngokgknr.supabase.co/auth/v1/callback` | Google이 Supabase로 사용자 돌려보낼 주소 |
| **Supabase → URL Configuration → Redirect URLs** | `http://localhost:3000/api/auth/callback` | Supabase가 우리 앱으로 돌려보낼 주소 |

Google에는 `localhost`를 등록할 필요 없고, Supabase에는 Supabase URL을 등록할 필요 없다. **각각 자기 다음 단계의 주소만 알면 된다.**

> 단, 우문현답에서는 Google Cloud Console에 `localhost:3000/api/auth/callback`도 같이 등록했음. 이건 향후 Supabase를 거치지 않고 직접 PKCE flow를 쓸 가능성을 열어둔 형태인데, 현재 코드 흐름에선 사실 필요 없음. 있어도 무해.

## 동작 원리 / 메커니즘

### 우리가 만든 코드 (`/api/auth/callback/route.ts`)

```ts
export async function GET(request: NextRequest) {
  const { searchParams, origin } = new URL(request.url)
  const code = searchParams.get('code')

  if (!code) {
    return NextResponse.redirect(`${origin}/login?error=missing_code`)
  }

  const supabase = createServerClient(/* ... */)
  const { error } = await supabase.auth.exchangeCodeForSession(code)

  if (error) {
    return NextResponse.redirect(`${origin}/login?error=auth_failed`)
  }

  return NextResponse.redirect(`${origin}/dashboard`)
}
```

이게 위 흐름의 ⑦~⑨ 담당. `exchangeCodeForSession`이 하는 일:
- Supabase가 우리에게 넘긴 `code`를 다시 Supabase 서버에 보냄
- Supabase가 세션 토큰을 우리에게 돌려줌
- `@supabase/ssr`의 cookie handler가 그 세션을 쿠키에 저장 (`sb-*` 쿠키들)

### 클라이언트 측 호출

```ts
await supabase.auth.signInWithOAuth({
  provider: 'google',
  options: {
    redirectTo: `${location.origin}/api/auth/callback`,
  },
})
```

`redirectTo`는 흐름 ⑦의 주소 — Supabase가 토큰 교환 끝낸 후 우리 앱으로 돌려보낼 곳. 이 값이 **Supabase의 Redirect URLs 허용 목록에 있어야 함**. 없으면 Supabase가 거부하고 `error=missing_code` 같은 형태로 떨어진다.

### middleware 역할

```ts
const { data: { user } } = await supabase.auth.getUser()
```

middleware에서 매 요청마다 위 코드를 호출. `@supabase/ssr`이 쿠키를 읽어서 세션 유효성을 검증하고, 만료 임박이면 자동 갱신해서 새 쿠키를 set한다. 그래서 cookie handler에서 `getAll` / `setAll` 둘 다 구현해야 함.

## 어떤 상황에서 마주쳤나

우문현답 W0~W4 기반 셋업 단계에서 소셜 로그인 붙이는 작업.
기존 `SocialButtons.tsx`는 `router.push('/dashboard')` 가짜 흐름이었고, 이걸 실제 Supabase OAuth로 교체.

### 마주친 에러 시퀀스

1. **`redirect_uri_mismatch` (Google 측)**
   → Google Cloud Console에 `https://...supabase.co/auth/v1/callback` 안 넣어서 발생
   → 흐름 ③에서 Google이 "이 redirect_uri로는 못 보냄"이라고 거부

2. **`missing_code` + `Unable to exchange external code: 4/0A` (Supabase 측)**
   → Supabase가 Google로부터 code는 받았는데, 토큰 교환(⑥)에서 실패
   → **Client Secret이 잘못 입력되어 있었음** (복사 시 공백 섞임 추정)
   → Google에서 Secret 재발급 후 Supabase에 다시 붙여넣으니 해결

## 해당 상황을 반복하지 않으려면 어떤 조치를 취해야하나?

1. **redirect URI 두 개 등록 체크리스트 만들기**:
   - [ ] Google Cloud Console에 `<supabase-url>/auth/v1/callback`
   - [ ] Supabase URL Configuration에 `<app-url>/api/auth/callback`
   - [ ] Supabase Site URL도 `<app-url>` 로 설정

2. **Client Secret은 복사 버튼으로만**: 드래그 선택 시 앞뒤 공백·줄바꿈 섞임. 의심되면 재발급.

3. **에러 메시지 해석 표** (다음에 또 막힐 때):
   - `redirect_uri_mismatch` → Google Cloud Console 측 URI 누락
   - `Unable to exchange external code` → Supabase에 입력된 Google Client Secret 오류
   - `missing_code` (Supabase에서 우리 앱으로 올 때) → Supabase Redirect URLs에 우리 앱 콜백 미등록 OR 위 토큰 교환이 실패해서 code 못 받음

## 헷갈렸던 부분 / 함정

- **처음엔 "redirect URI 하나만 등록하면 되겠지"라고 생각함** → 실제로는 **두 곳에 각각 등록**. Google Cloud Console과 Supabase는 서로 다른 단계의 redirect를 관리한다.
- **`missing_code` 에러를 보고 Supabase Redirect URLs 문제라고 의심함** → 실제로는 그 전 단계인 **Google 측 토큰 교환 실패**. 에러 URL 끝에 붙은 `error_description=Unable+to+exchange+external+code`를 해석했어야 함.
- **Google이 "데스크탑 앱" vs "웹 애플리케이션"으로 OAuth Client를 만들 수 있게 함** → OAuth callback이 있는 서버라면 무조건 **웹 애플리케이션**. 데스크탑은 redirect URI 입력칸 자체가 없음.
- **Project URL과 Project ID는 동일 ref를 공유** → Project Settings → API에 URL이 안 보일 수도 있지만, Reference ID(`tikycotngtobngokgknr`)로 URL을 추론 가능: `https://<ref>.supabase.co`.

## 응용·확장

- 카카오·네이버 추가할 때도 같은 패턴 — provider별로 OAuth 앱 만들고, redirect URI 두 곳 등록, Supabase Provider 설정에 Client ID/Secret 입력. 콜백 코드는 그대로 재사용 (`/api/auth/callback`).
- 배포 시점에 redirect URI에 production 도메인 추가 필요. Google·Supabase 양쪽 모두.
- PKCE flow (Client Secret 없이 동작)는 모바일·SPA에서 더 안전한 방식. 우리는 Server Component 환경이라 Authorization Code Flow + Client Secret이 더 적합.
- `exchangeCodeForSession` 후 Prisma `User` 테이블에 row 생성하는 로직은 별도 필요 — Supabase Auth가 만드는 `auth.users`와 우리 도메인 테이블 분리되어 있음.

## 참고 자료

- [Supabase Docs — PKCE flow](https://supabase.com/docs/guides/auth/sessions/pkce-flow) — code verifier 흐름과 5분 만료 제약 명시
- [Supabase Docs — Login with Google](https://supabase.com/docs/guides/auth/social-login/auth-google) — Google Cloud Console에 Supabase callback URL 등록하는 절차
- [Supabase Docs — Redirect URLs](https://supabase.com/docs/guides/auth/redirect-urls) — Site URL과 Redirect URLs의 역할 구분
- [Supabase JS — exchangeCodeForSession](https://supabase.com/docs/reference/javascript/auth-exchangecodeforsession) — API 레퍼런스
- [Supabase Docs — Advanced server-side guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide) — `@supabase/ssr` cookie handler 상세
- RFC 6749 §4.1 — Authorization Code Grant 원본 스펙
- RFC 7636 — PKCE 확장 스펙

---

## 품질 피드백

```
🔴 필수
  1 민감정보 없음        ⚠️  Partial → anon key·project ref가 본문에 노출됨. anon key는 RLS 전제 하 공개 가능하지만, ref는 마스킹 권장.
  2 카테고리 태그        ✅ Pass  (#개념정리)
  3 배운 것이 명확함     ✅ Pass  ("콜백이 두 번이라 redirect URI도 두 개")
  4 원본 링크            ⚠️  Partial → CLAUDE.md만 링크. issues/backlog가 없는 프로젝트라 한계.

🟡 권장
  5 6개월 뒤 이해        ✅ Pass
  6 코드 블록 언어       ✅ Pass  (tsx, 텍스트 다이어그램은 의도적으로 무지정)
  7 헷갈렸던 부분        ✅ Pass  (4개 함정 기록)
  8 수치·근거            ⏭ Skip  (개념 정리라 수치 영역 적음)

🟢 선택
  9 응용·확장            ✅ Pass
 10 README 갱신          ❌ Fail  → /Users/seong-in/Desktop/Git/note/README.md 미갱신.

총평: 필수 1번(민감정보) 한 번 확인 필요. anon key는 본래 public이라 OK지만, project ref는 마스킹할지 결정.
```
