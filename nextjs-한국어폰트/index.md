# 노트 — "Next.js 한국어 폰트 최적화: next/font vs CDN dynamic subset"

> 작성일: 2026-05-08
> 태그: #설계결정 #nextjs #성능
> 출발점: 우문현답 초기 세팅 — Pretendard 폰트를 next/font/local 대신 jsDelivr CDN dynamic subset으로 적용
> 원본 기록: 없음 (우문현답 초기 세팅)

## 한 줄 요약

한국어 폰트는 CJK 글리프가 11,172자(현대 한글 전수)라 폰트 파일이 수십 MB에 달한다. `next/font/local`은 빌드타임에 폰트를 번들에 포함하므로 한국어 폰트에 부적합하고, CDN dynamic subset이 실제 페이지에 사용된 글자만 선택적으로 내려받아 더 빠르다.

![next/font vs CDN 로딩 흐름 비교](./font-loading-comparison.png)

## 배경 지식

### 한국어 폰트가 특수한 이유

영문 폰트: 알파벳 대소문자 + 숫자 + 특수문자 → 약 200~300 글리프. 폰트 파일 30~100KB.

한국어 폰트:
- 현대 한글 완성형: **11,172자** (유니코드 가나다-힣 전수)
- KS X 1001 완성형(사용 빈도 상위): **2,350자**
- Pretendard Variable 기준 전체 weight 포함 시: **수 MB**

Pretendard Variable (woff2) 단일 파일이 약 4~5MB. 9가지 굵기를 정적으로 다 넣으면 수십 MB.

### Dynamic Subset이란

Google Fonts가 한글 폰트에 쓰는 방식. 글리프를 여러 Unicode range 청크로 쪼개 두고, 페이지에서 실제로 등장하는 문자의 청크만 다운로드한다. 폰트 요청 수는 늘어나지만 **총 다운로드 용량이 극적으로 감소**함.

예: "우문현답"만 페이지에 있으면 'ㅇ', 'ㅜ' 포함 청크만 다운로드 → 전체 폰트의 1~3%만 내려받음.

Pretendard가 jsDelivr에 호스팅하는 dynamic subset CSS:
```
https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/variable/pretendardvariable-dynamic-subset.min.css
```

이 CSS 안에 `@font-face` 규칙이 `unicode-range`와 함께 청크별로 선언되어 있음.

### next/font의 한국어 문제

`next/font/google`은 Google Fonts API를 빌드타임에 호출해서 폰트를 self-hosting함. 한글 폰트(Noto Sans KR 등)는 CJK라서 **100개 이상의 폰트 파일로 분할됨**. 공식 문서에도 명시:

> CJK users will have to disable preloading due to this constraint.
> (https://github.com/vercel/next.js/discussions/47309)

즉 `preload: false`로 설정해야 하고, 그러면 `next/font`의 핵심 이점인 **LCP 최적화 preload 태그 자동 삽입**이 사라짐.

`next/font/local`로 woff2 파일을 직접 넣으면:
- 파일을 public/fonts/ 에 커밋해야 함 → 레포 용량 수 MB 증가
- 전체 폰트를 항상 다운로드 (dynamic subset 불가)
- 빌드 결과물에 폰트 파일 포함 → 초기 번들 크기 증가

## 동작 원리 / 메커니즘

### CDN Dynamic Subset 방식 (우문현답 선택)

```tsx
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">
      <head>
        {/* Pretendard — jsDelivr CDN dynamic subset */}
        <link
          rel="stylesheet"
          href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/variable/pretendardvariable-dynamic-subset.min.css"
        />
      </head>
      <body>{children}</body>
    </html>
  );
}
```

```css
/* globals.css */
:root {
  --font-sans: "Pretendard Variable", Pretendard, system-ui, sans-serif;
}

body {
  font-family: var(--font-sans);
}
```

브라우저 로딩 순서:
1. HTML 파싱 → `<link>` 태그 발견
2. jsDelivr CDN에서 CSS 파일 다운로드 (캐시 HIT 시 즉시)
3. CSS 파싱 → `@font-face` + `unicode-range` 청크 목록 확인
4. 현재 페이지 렌더링에 필요한 Unicode range 청크만 선택적 다운로드
5. 폰트 적용 (FOUT: Flash of Unstyled Text 발생 가능, `font-display: swap` 기본)

### next/font/local 방식 (비교)

```tsx
// app/layout.tsx (next/font 방식)
import localFont from "next/font/local";

const pretendard = localFont({
  src: [
    { path: "../fonts/Pretendard-Regular.woff2", weight: "400" },
    { path: "../fonts/Pretendard-Bold.woff2", weight: "700" },
    // ... 9개 weight 다 추가해야 함
  ],
  variable: "--font-sans",
});

export default function RootLayout({ children }) {
  return (
    <html lang="ko" className={pretendard.variable}>
      <body>{children}</body>
    </html>
  );
}
```

이 경우 woff2 파일들을 `src/fonts/`나 `public/fonts/`에 직접 보관해야 함. 9개 weight × ~500KB ≈ 4.5MB 레포 추가.

### 선택 근거 (트레이드오프 표)

| 항목 | next/font/local | CDN dynamic subset |
|---|---|---|
| 초기 로딩 속도 | 느림 (전체 파일) | 빠름 (필요 청크만) |
| 외부 네트워크 의존성 | 없음 (self-hosted) | jsDelivr 의존 |
| 레포 용량 | +4~5MB | 0 |
| FOUT | 없음 (preload) | 발생 가능 |
| CDN 캐시 효과 | 없음 | 있음 (재방문 즉시) |
| 버전 고정 | 자동 (빌드시) | URL에 명시 (`@v1.3.9`) |
| 오프라인/Private | 작동 | 불가 |

jsDelivr는 전 세계 CDN 노드가 200개 이상. 한국 사용자 기준으로도 캐시 HIT 시 5ms 이내. 한국 서비스라 CDN 의존성은 실질적 리스크 낮음.

## 어떤 상황에서 마주쳤나

우문현답 초기 세팅. Tailwind를 안 쓰기로 해서 `next/font` 연동(`className={font.variable}`)이 더 복잡해짐. CDN 방식이 CSS Custom Properties로 바로 연결되어 더 단순했음.

결정 포인트:
1. 한국어 서비스 → 사용자 거의 전원 브라우저 캐시에 jsDelivr 리소스 이미 있을 확률 높음
2. 9가지 weight 폰트 파일을 레포에 커밋하고 싶지 않음
3. next/font의 CJK preloading 제한이 장점을 상쇄

## 해당 상황을 반복하지 않으려면

- 한국어 폰트 선택 시 무조건 dynamic subset 지원 여부 확인. Pretendard는 지원, 일부 커스텀 폰트는 미지원.
- CDN URL에 버전 고정 (`@v1.3.9`). `@latest`로 쓰면 폰트 업데이트 시 디자인이 자동으로 바뀌어 버림.
- 영문 폰트는 `next/font/google` 그대로 써도 됨. 한국어만 특수 케이스.
- `<link rel="preconnect" href="https://cdn.jsdelivr.net" />` 추가하면 CDN 연결 지연 줄일 수 있음 (현재 미적용).

## 헷갈렸던 부분 / 함정

- 처음엔 `next/font`가 항상 더 좋다고 생각했음. "Next.js 공식 방식이니까". 근데 공식 문서 내 이슈 트래커 보니 CJK는 `preload: false` 필수라고 명시되어 있었음. preload 없으면 next/font 쓰는 의미가 절반 이상 사라짐.
- "dynamic subset"이 서버에서 글자를 잘라서 보내는 건 줄 알았는데, 실제로는 클라이언트가 `unicode-range`를 보고 **스스로** 필요한 청크만 요청하는 방식. 서버 로직 없음.
- Font Variable이랑 Pretendard 둘 다 `font-family`에 선언하는 이유: 가변 폰트를 지원하는 브라우저는 "Pretendard Variable"을, 안 되는 구형 브라우저는 "Pretendard" 정적 파일을 fallback으로 사용. 2026 기준 가변 폰트 지원율은 97%+ 이므로 사실상 Pretendard fallback은 거의 필요 없지만 명시해두는 게 안전.
- CSS `font-display: swap` 때문에 FOUT(폰트 로딩 전 시스템 폰트 잠깐 보임)이 발생함. 체감상 한국 사용자는 재방문 시 CDN 캐시로 즉시 로드되어 FOUT 거의 없음. 신규 방문자는 100~200ms 정도 시스템 폰트 → Pretendard 전환이 보임.

## 응용·확장

- **다른 한국어 폰트**: Noto Sans KR, Spoqa Han Sans도 동일한 문제. 각각 Google Fonts CDN dynamic subset 사용 권장.
- **Variable Font 지원 확인**: `font-variation-settings: "wght" 600` 식으로 weight 축 조절 가능. 9개 정적 weight 대신 1개 파일로 모든 굵기 커버.
- **`<link rel="preload">`**: 특정 청크를 미리 로드하고 싶다면 unicode-range 청크 URL을 직접 preload. 단 어떤 청크가 필요할지 미리 알아야 하므로 실용성은 낮음.
- **Fallback 폰트 최적화**: `@font-face` + `size-adjust`, `ascent-override` 속성으로 시스템 폰트와 Pretendard의 크기 차이를 줄여 FOUT CLS(Layout Shift) 최소화 가능.

## 참고 자료

- [Pretendard GitHub — dynamic subset 적용법](https://github.com/orioncactus/pretendard) — 공식 CDN URL 확인
- [Pretendard dynamic subset 사용방법 Discussion](https://github.com/orioncactus/pretendard/discussions/148) — 실제 적용 예제
- [Next.js — next/font CJK 이슈 Discussion](https://github.com/vercel/next.js/discussions/47309) — `unicode-range` 미지원 공식 언급
- [Next.js Font Optimization 공식 문서](https://nextjs.org/docs/app/getting-started/fonts) — next/font 기본 사용법
