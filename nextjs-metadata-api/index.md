# 노트 — "Next.js App Router Metadata API"

> 작성일: 2026-05-08
> 태그: #개념정리 #nextjs #seo
> 출발점: 우문현답 초기 세팅 — layout.tsx에 한국어 메타데이터 + OG 태그 설정
> 원본 기록: 없음 (우문현답 초기 세팅)

## 한 줄 요약

Next.js App Router에서 `export const metadata`로 정적 메타데이터를 선언하면, `<head>` 안의 모든 SEO 태그를 서버 컴포넌트 레벨에서 타입 안전하게 관리할 수 있다. `title.template`은 하위 페이지 제목에 자동으로 사이트명을 붙여준다.

## 배경 지식

### Pages Router와의 차이

Pages Router에서는 `<Head>` 컴포넌트를 각 페이지에서 직접 import해서 메타 태그를 수동으로 넣었음:

```tsx
// Pages Router 방식 (구식)
import Head from 'next/head';

export default function Page() {
  return (
    <>
      <Head>
        <title>페이지 제목</title>
        <meta property="og:title" content="페이지 제목" />
      </Head>
      <main>...</main>
    </>
  );
}
```

App Router는 `Metadata` 타입 객체를 export하는 방식으로 통일됨. 타입 검사가 가능하고, 중복 선언을 자동으로 dedup해줌.

### 메타데이터 상속 규칙

App Router는 segment별로 메타데이터를 병합한다:
- `app/layout.tsx`에 선언 → 전체 앱에 적용 (기본값)
- `app/dashboard/page.tsx`에 선언 → dashboard 페이지에서 override

부모 메타데이터가 자식에게 **그대로 상속되지 않음**. 각 segment에서 명시적으로 선언해야 함. 단, `title.template`은 예외적으로 하위 segment에서 template 적용을 위해 동작.

## 동작 원리 / 메커니즘

### 정적 메타데이터

```tsx
// app/layout.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: {
    default: "우문현답 — AI 모의면접",  // 하위 페이지에서 title 미선언 시 사용
    template: "%s | 우문현답",          // 하위 페이지 title이 있으면 이 template 적용
  },
  description: "어떤 질문에도 답할 수 있게.",
  keywords: ["모의면접", "AI면접", "공채"],
  authors: [{ name: "우문현답" }],
  openGraph: {
    type: "website",
    locale: "ko_KR",
    siteName: "우문현답",
    title: "우문현답 — AI 모의면접",
    description: "어떤 질문에도 답할 수 있게.",
  },
};
```

### title.template 작동 방식

```
layout.tsx:  template: "%s | 우문현답"
page.tsx:    title: "면접 결과" (하위 페이지)
결과 HTML:   <title>면접 결과 | 우문현답</title>
```

`%s` 자리에 하위 page.tsx의 title이 들어감. 하위 페이지에서 title을 안 선언하면 layout.tsx의 `default` 값이 사용됨.

### lang="ko" 설정

Metadata API가 아니라 **layout.tsx의 `<html>` 태그에 직접** 설정:

```tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko">  {/* 여기서 lang 지정 */}
      <head>...</head>
      <body>{children}</body>
    </html>
  );
}
```

Metadata 객체에는 `lang` 필드가 없음. `openGraph.locale: "ko_KR"`은 OG 태그용이고, HTML 언어 속성과 별개.

### 동적 메타데이터

페이지별로 동적으로 생성해야 할 때 (예: `/companies/[slug]`):

```tsx
// app/companies/[slug]/page.tsx
import type { Metadata } from "next";

export async function generateMetadata({ params }: { params: { slug: string } }): Promise<Metadata> {
  const company = await fetchCompany(params.slug);

  return {
    title: company.name,  // template 적용돼서 "삼성전자 | 우문현답"이 됨
    description: `${company.name} 면접 후기와 AI 모의면접`,
    openGraph: {
      title: `${company.name} 면접 준비 | 우문현답`,
      images: [company.ogImageUrl],
    },
  };
}
```

### OG 이미지 자동 생성

`opengraph-image.tsx` 파일을 폴더에 두면 Next.js가 빌드타임 또는 요청시 OG 이미지를 자동 생성:

```tsx
// app/opengraph-image.tsx
import { ImageResponse } from "next/og";

export default function OGImage() {
  return new ImageResponse(
    <div style={{ background: "#1e3a8a", width: "100%", height: "100%", display: "flex", alignItems: "center", justifyContent: "center" }}>
      <span style={{ color: "white", fontSize: 60 }}>우문현답</span>
    </div>,
    { width: 1200, height: 630 }
  );
}
```

## 어떤 상황에서 마주쳤나

우문현답 초기 세팅. 랜딩 페이지부터 SEO 기본값을 잡고 싶었음. 특히:
- 카카오/라인/슬랙 링크 공유 시 OG 카드가 제대로 나와야 함 (취준생들이 링크 공유 많이 함)
- 하위 페이지가 늘어날 때마다 "| 우문현답" suffix를 수동으로 붙이는 게 번거로움 → `title.template`으로 해결

## 해당 상황을 반복하지 않으려면

- `metadata`는 `layout.tsx`에 기본값, 각 `page.tsx`에 세부값 선언하는 패턴 유지
- OG 태그에 `images` 없으면 카카오톡 미리보기에 이미지 안 나옴. 초기부터 `opengraph-image.tsx` 세팅 or `openGraph.images` 배열 채워두기
- `locale: "ko_KR"` (underscore)이고 `lang="ko"` (hyphen)이다. 헷갈리지 말 것

## 헷갈렸던 부분 / 함정

- 처음엔 `<html lang="ko">`를 Metadata 객체에서 설정할 수 있는 줄 알았는데, Metadata API에는 `lang` 필드가 없음. HTML의 `lang` 속성은 layout.tsx JSX에서 직접 설정해야 함. `openGraph.locale: "ko_KR"`은 소셜 미디어 크롤러용이고 `<html lang>` 과 다른 거임.
- `title.default`와 `title.template`을 같이 선언해야 함. template만 선언하면 하위 페이지에서 title 없을 때 빈 값 or 에러.
- `generateMetadata` 함수와 `metadata` 객체를 같은 파일에 동시 export 불가. 둘 중 하나만.
- OG title과 page title을 따로 선언 안 하면 같은 값을 쓰는데, 소셜 공유용 카피와 브라우저 탭 제목은 다를 수 있음 (OG는 더 마케팅 언어, title은 간결하게).
- Next.js 13.2에서 Metadata API 도입됨. 13.0, 13.1 App Router는 Metadata 없었음 → 그 시절 블로그 참고하면 구식 `<Head>` import 방식 나옴.

## 응용·확장

- `robots`, `canonical`, `alternates` 등도 Metadata 타입에 있음. 다국어 서비스 시 `alternates.languages`로 hreflang 자동 생성.
- `viewport`는 Next.js 14부터 Metadata에서 분리됨 (`export const viewport: Viewport = {...}`).
- `twitter` 객체로 Twitter Card 별도 설정 가능. OG 태그와 따로 선언 안 하면 OG 태그를 fallback으로 씀.

## 참고 자료

- [Next.js — generateMetadata 공식 API](https://nextjs.org/docs/app/api-reference/functions/generate-metadata) — `Metadata` 타입 전체 필드 목록
- [Next.js — Metadata and OG images 가이드](https://nextjs.org/docs/app/getting-started/metadata-and-og-images) — 시작하기 좋은 overview
- [Next.js 13.2 릴리즈 노트](https://nextjs.org/blog/next-13-2) — Metadata API 도입 배경