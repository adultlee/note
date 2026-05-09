# 노트 — "CSS Custom Properties로 디자인 토큰 시스템 만들기"

> 작성일: 2026-05-08
> 태그: #개념정리 #css #디자인시스템
> 출발점: 우문현답 초기 세팅 — Tailwind 없이 globals.css를 토큰 시스템으로 구성
> 원본 기록: 없음 (우문현답 초기 세팅)

## 한 줄 요약

CSS Custom Properties를 `:root`에 선언하면 Tailwind 없이도 일관된 디자인 시스템을 만들 수 있다. 핵심은 **Primitive → Semantic → Component** 3계층 분리다.

![CSS 디자인 토큰 계층 구조](./token-hierarchy.png)

## 배경 지식

### 디자인 토큰이란

디자인 토큰(Design Token)은 색상·타이포·간격·애니메이션 등을 **코드로 표현한 변수 집합**이다. "버튼 색은 #1e3a8a" 같은 하드코딩 대신, "버튼 색은 브랜드 컬러"로 의미를 부여한다.

### CSS Custom Properties vs Tailwind

Tailwind는 유틸리티 클래스를 HTML에 뿌리는 방식. 빠르지만:
- 클래스명이 마크업에 노출 → `className="bg-blue-800 text-beige-100 hover:..."` 식으로 길어짐
- 커스텀 색상·비율이 복잡해지면 `tailwind.config.ts` 가 거대해짐
- 디자인 시스템을 Figma 토큰과 1:1 매핑하려면 결국 CSS 변수도 써야 함

CSS Custom Properties 단독은:
- `var(--brand)` 한 곳에서 바꾸면 전체 반영
- 브라우저 지원 IE11 이하 제외하면 100% (2026 기준 신경 안 써도 됨)
- 런타임 재정의 가능 (다크모드, 테마 스위칭)

### 3계층 구조

```
Primitive (원시값)
  → --color-blue-900: #1e3a8a

Semantic (의미)
  → --brand: var(--color-blue-900)   // "브랜드 컬러"

Component (컴포넌트)
  → --btn-bg: var(--brand)           // "버튼 배경"
```

실제론 작은 프로젝트에서 3계층 다 쓰면 오버엔지니어링. 우문현답은 **Primitive + Semantic** 2계층으로 단순화했음.

## 동작 원리 / 메커니즘

`:root` 선언 → 전역 스코프에 커스텀 프로퍼티 등록 → `var()` 함수로 참조.

```css
/* globals.css */
:root {
  /* Primitive (팔레트) */
  --brand:        #1e3a8a;
  --brand-light:  #e0e7ff;
  --brand-soft:   #eef2ff;
  --warm:         #c19560;
  --warm-light:   #faf6f0;

  /* Grayscale */
  --gray-900: #191f28;
  --gray-700: #4e5968;
  --gray-500: #6b7684;
  --gray-300: #b0b8c1;
  --gray-100: #f2f4f6;
  --gray-50:  #f9fafb;

  /* Semantic */
  --success: #00a881;
  --warning: #f08c00;

  /* Dark mode 전용 (면접 화면) */
  --dark-bg:      #0a0e14;
  --dark-surface: #161b25;
  --dark-border:  rgba(255,255,255,0.08);
  --dark-text:    #f2f4f6;
  --dark-accent:  #6b8aff;

  /* Typography */
  --font-sans: "Pretendard Variable", Pretendard, system-ui, sans-serif;
  --font-mono: "SF Mono", Monaco, monospace;

  /* Transition */
  --transition-fast:   150ms ease;
  --transition-normal: 250ms ease;
}
```

컴포넌트에서 쓸 때:

```css
/* button.module.css */
.btn-primary {
  background: var(--brand);
  color: white;
  transition: background var(--transition-fast);
}

.btn-primary:hover {
  background: var(--brand-light);
  color: var(--brand);
}
```

다크 모드는 `.dark` 클래스나 `@media (prefers-color-scheme: dark)`로 재정의:

```css
@media (prefers-color-scheme: dark) {
  :root {
    --brand: #6b8aff;  /* 어두운 배경에 맞게 재정의 */
  }
}
```

## 어떤 상황에서 마주쳤나

우문현답 초기 세팅. Tailwind를 쓰지 않기로 결정한 이유:
- 면접 화면(어두운 배경)과 마케팅 랜딩(밝은 배경)을 같은 컴포넌트로 커버해야 하는데, Tailwind `dark:` prefix가 번거로움
- 디자인 토큰을 Figma에서 직접 추출해서 CSS 변수로 1:1 매핑하고 싶었음
- Next.js CSS Modules + CSS Custom Properties 조합이 빌드 설정 없이 바로 됨

## 해당 상황을 반복하지 않으려면

- **토큰 이름은 의미 중심**으로 짓기. `--color-blue-800` 보다 `--brand`가 낫고, 나중에 파란색이 초록색으로 바뀌어도 의미가 깨지지 않음
- **다크 모드 토큰은 처음부터 분리**해서 선언. `--dark-bg`, `--dark-surface` 식으로 prefix 붙이면 나중에 찾기 쉬움
- **Tailwind와 혼용은 최대한 피하기.** `var(--brand)`랑 `bg-blue-800`이 같은 파일에 공존하면 유지보수가 지옥

## 헷갈렸던 부분 / 함정

- 처음엔 `--color-brand-primary` 식으로 네이밍하려 했는데, 실제 써보니 `var(--color-brand-primary)` 치다가 손목 아파서 그냥 `--brand`로 단순화했음. 컴포넌트 토큰 레이어는 팀이 커지거나 디자인 시스템을 라이브러리로 배포할 때만 필요.
- "Tailwind를 안 쓰면 반응형이 불편하다"고 생각했는데, `@media` 쿼리에서 CSS Custom Properties 재정의하면 오히려 더 깔끔. Tailwind 반응형은 `sm:`, `md:` prefix를 모든 클래스에 붙여야 하는데, 변수 하나만 바꾸면 되니까 코드량이 줄어듦.
- `var()` 안에 fallback 값을 항상 넣어야 한다고 생각했는데, `:root`에 선언한 건 fallback 없어도 됨. Fallback은 CSS Custom Properties를 외부에서 주입받는 컴포넌트 라이브러리 만들 때나 필요.

## 응용·확장

- **CSS-in-JS와 연동**: `styled-components`나 `emotion`에서도 `var(--brand)` 그대로 쓸 수 있음. 런타임 토큰 주입 없이 그냥 됨.
- **Radix UI와 조합**: Radix UI는 자체 CSS Custom Properties 시스템 제공. 우리 토큰과 매핑해서 쓰면 Radix 컴포넌트도 브랜드 색상 적용 가능.
- **Style Dictionary**: 토큰을 JSON으로 관리하고 CSS / iOS / Android 등 여러 포맷으로 빌드해주는 도구. 팀이 커지면 도입 고려.
- **CSS `@layer`**: 토큰을 별도 레이어로 선언하면 specificity 충돌 방지 가능.

## 참고 자료

- [Contentful — Design token system explained](https://www.contentful.com/blog/design-token-system/) — 3계층 구조 설명 깔끔
- [CSS Custom Properties: The Modern Way to Manage Design Tokens](https://dev.to/snappy_tools/css-custom-properties-the-modern-way-to-manage-design-tokens-1ohg) — 실전 예제 포함
- [Panda CSS Tokens](https://panda-css.com/docs/theming/tokens) — 토큰 시스템 레퍼런스 구현체