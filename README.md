# 학습 노트 인덱스

작업하면서 새로 알게 된 개념·메커니즘·원리를 정리하는 공간.

---

- **2026-05-07** [#개념정리 #elo #lck] [ELO 알고리즘 기초와 K팩터](./ELO-알고리즘-기초와-K팩터/index.md)
  - 한 줄 요약: ELO는 예상 승률 대비 실제 결과의 차이만큼 레이팅을 업데이트하고, K팩터가 그 폭을 조절한다. 국제전 K×3은 리그 간 첫 교차 정보의 정보량을 반영.
  - 출발점: FanClash ELO 도입 및 국내전/국제전 K팩터 분리 설계

- **2026-05-07** [#원인분석 #elo #typescript #lck] [anchor 소급 보정과 선형 분산](./anchor-소급보정과-선형분산/index.md)
  - 한 줄 요약: anchor 즉시 수렴은 차트에 점프를 만든다. 구간 이벤트에 선형 보간 소급 적용으로 ±183pt → ±8pt
  - 출발점: GPL anchor 도입 후 ELO 차트에 경기 없는 날 T1 -83, GEN -183pt 급락

- **2026-05-07** [#prisma #codegen #typescript] [Prisma generate 누락 시 빌드 에러](./prisma-generate-codegen/index.md)
  - 한 줄 요약: 스키마 수정 후 `npx prisma generate` 안 하면 TS가 새 모델 타입을 모른다
  - 출발점: `ProbabilitySnapshot` 모델 추가 후 Vercel 빌드 실패

- **2026-05-07** [#설계결정 #elo #lck] [팀 개인 ELO와 리그 레이팅 분리 설계](./리그-레이팅-분리-설계/index.md)
  - 한 줄 요약: 표시 ELO = 팀 개인 ELO + (리그 레이팅 변동 × 0.4), 국제전 결과를 리그 전체에 전파
  - 출발점: LPL/LEC/LCS 확장 후 리그 간 ELO 비교 불가 문제

- **2026-05-07** [#설계결정 #elo #lck] [GPL 앵커로 ELO 초기값 역산하기](./GPL-앵커-역산으로-초기값-설정하기/index.md)
  - 한 줄 요약: 1500은 "아무것도 모를 때 쓰는 값", GPL 앵커는 과거 국제전 성적으로 시작점을 역산한 값 (실제 범위: 1044~1663)
  - 출발점: LPL/LEC/LCS 확장 시 1500 고정값으로는 리그 간 실력 격차가 반영되지 않는 문제

- **2026-05-07** [#설계결정 #게임설계 #배당률] [봇 유저를 처음부터 넣은 이유](./봇-유저를-처음부터-넣은-이유/index.md)
  - 한 줄 요약: 파리뮤추얼에서 배팅 풀이 비어있으면 배당률이 수학적으로 의미없어지고, 봇이 그 풀을 채우는 동시에 소셜 토스트 소재도 된다
  - 출발점: Phase 1 뼈대 단계에서 봇을 동시 투입한 설계 이유

- **2026-05-07** [#설계결정 #typescript #lck #시뮬레이션] [몬테카를로 시뮬레이션 횟수 결정 — 왜 50,000인가](./몬테카를로-시뮬레이션-횟수-결정/index.md)
  - 한 줄 요약: 10,000→50,000은 오차 55% 감소, 50,000→100,000은 29%만 감소. √N 법칙으로 50,000이 효용 꺾임점
  - 출발점: `SIMULATIONS = 50_000` 상수 결정 시 10,000 / 100,000과 비교

- **2026-05-07** [#개념정리 #게임설계 #배당률 #예측] [파리뮤추얼 배당률 계산 — 유저 예측 분포가 배당률을 결정하는 원리](./파리뮤추얼-배당률-계산/index.md)
  - 한 줄 요약: 파리뮤추얼은 다른 유저들의 쏠림이 배당률을 결정한다 — 동조하면 수익 감소, 역행하면 수익 증가
  - 출발점: 고정 배당 대신 파리뮤추얼을 선택한 이유와 `calcOdds` 수식 구조 이해

- **2026-05-07** [#성능튜닝 #nextjs] [CSS `backgroundImage`의 한계, `next/image`로 교체하면 얻는 것](./CSS-background-to-next-image/index.md)
  - 한 줄 요약: CSS background-image는 CSSOM 파싱 후에야 요청 → next/image는 HTML 파싱 즉시 요청 + WebP 자동변환 + CDN 캐싱
  - 출발점: MatchBackground.tsx, teams/[slug]/page.tsx 배경 이미지를 next/image fill로 교체

- **2026-05-07** [#설계결정 #tanstack-query #typescript] [컴포넌트에서 직접 fetch 금지, 훅 중앙화로 얻는 것](./TanStack-Query-훅-중앙화/index.md)
  - 한 줄 요약: `useQuery` 훅을 한 파일에 모아두면 캐시 키 충돌을 막고, 동일 API를 여러 컴포넌트가 호출해도 네트워크 요청은 1번만 나간다
  - 출발점: Phase 4 리팩토링 — fetch 로직이 여러 컴포넌트에 분산된 상태를 `src/lib/queries/index.ts`로 중앙화

- **2026-05-07** [#원인분석 #css #nextjs] [fixed position 툴팁: scrollY 함정과 overflow-hidden 탈출](./fixed-position-tooltip-scrollY-함정/index.md)
  - 한 줄 요약: getBoundingClientRect는 viewport 기준이라 fixed에 그대로 쓰면 되고, scrollY를 더하면 오히려 틀어진다. z-index 9999도 부모의 overflow:hidden은 못 뚫는다
  - 출발점: EloTooltip이 스크롤하면 위치 틀어지고 overflow-hidden 컨테이너에 잘리는 두 버그

- **2026-05-07** [#원인분석 #nextjs #vercel] [fs.readFileSync vs import — Vercel serverless 파일 접근 문제](./Vercel-serverless-fs-접근-문제/index.md)
  - 한 줄 요약: Vercel serverless에서 `fs.readFileSync`는 `process.cwd()` 경로가 달라 실패, `import`는 빌드 타임 번들에 포함되어 항상 성공
  - 출발점: 로컬 정상 → Vercel 배포 후 `/api/elo` 빈 응답 반환

- **2026-05-07** [#원인분석 #nextjs #캐시] [.next 폴더 오래된 캐시 문제와 해결 패턴](./next-캐시-문제와-해결패턴/index.md)
  - 한 줄 요약: .next/cache는 소스 변경을 못 잡는 경우가 있고, "Vercel만 터짐 vs 로컬만 터짐" 패턴으로 캐시 문제를 빠르게 판별할 수 있다
  - 출발점: Vercel 배포 후 ELO API 빈 응답 + 로컬에서 재현 안 되는 버그들

- **2026-05-07** [#설계결정 #nextjs #react] [iframe vs React 컴포넌트 — 리포트 카드 렌더링 방식 선택](./iframe-vs-react-component-트레이드오프/index.md)
  - 한 줄 요약: API가 HTML을 반환하고 iframe으로 띄우면 스타일 격리·반응형·html2canvas 세 문제가 동시에 발목을 잡는다. JSON + React 컴포넌트가 처음에 더 많이 만들지만 결국 코드가 더 적어진다
  - 출발점: 리포트 카드를 iframe으로 만들었다가 18분 만에 React 컴포넌트로 전환

- **2026-05-07** [#설계결정 #nextjs] [Next.js 캐시 전략 선택 기준: force-dynamic vs revalidate](./force-dynamic-vs-revalidate/index.md)
  - 한 줄 요약: `force-dynamic`은 요청마다 항상 실행 (쓰기·인증·실시간), `revalidate = N`은 N초 캐시 후 재검증 (읽기 전용·집계). 선택 기준은 요청 컨텍스트 의존 여부
  - 출발점: API 라우트 50개 중 34개 force-dynamic, 8개 revalidate — 선택 기준 정리 필요

- **2026-05-07** [#개념정리 #typescript #nextjs] [tsconfig.tsbuildinfo: TypeScript 증분 빌드 캐시 파일](./tsconfig-tsbuildinfo-증분빌드-캐시/index.md)
  - 한 줄 요약: `"incremental": true`가 켜지면 생성되는 컴파일 캐시. 변경 파일만 재컴파일해서 빌드 최대 5~10x 빠름. 런타임 무관, git에 올리면 안 됨
  - 출발점: git status에서 tsbuildinfo가 계속 수정 상태로 잔존 → .gitignore 누락 확인

- **2026-05-08** [#보안 #nextjs #typescript] [인메모리 Rate Limiter로 회원가입 API 남용 차단하기](./인메모리-rate-limit-signup-보호/index.md)
  - 한 줄 요약: Map 하나로 IP당 요청 횟수를 카운팅, 단일 IP 봇 기준 어뷰징 속도 97% 감소 (무제한 → 1시간 3회)
  - 출발점: `/api/auth/signup`에 제한 없어 계정 대량 생성 → 포인트 어뷰징 경로 발견

- **2026-05-09** [#개념정리 #css #성능튜닝 #web-vitals] [CSS 렌더링 파이프라인 — Reflow, Repaint, Composite & CLS](./css-렌더링파이프라인-reflow-cls/index.md)
  - 한 줄 요약: `max-height` 트랜지션은 매 프레임 Reflow를 유발하고, `translateY`는 Composite 단계만 타서 GPU가 처리한다. CLS = 영향 분율 × 이동 거리 분율, 0.1 이하가 Good
  - 출발점: max-height transition이 왜 layout shift를 유발하는지, translateY가 왜 부드러운지 브라우저 렌더링 파이프라인 관점에서 탐구

- **2026-05-09** [#개념정리 #성능튜닝 #nextjs #web-vitals] [WebP 이미지 최적화 — 압축률, cwebp -q, Next.js 자동 변환](./webp-이미지-최적화/index.md)
  - 한 줄 요약: WebP는 JPEG 대비 동일 화질 기준 25~34% 작다 (Google 공식). cwebp -q 80은 0~100 중 웹 권장 범위(75~85). Next.js Image는 설정 없이 WebP 자동 변환
  - 출발점: 이미지 최적화 시 WebP 실제 효과와 Next.js 자동 지원 여부 확인

- **2026-05-08** [#개념정리 #tanstack-query #nextjs #성능튜닝] [SWR 패턴과 TanStack Query staleTime](./SWR-패턴과-TanStack-Query-staleTime/index.md)
  - 한 줄 요약: SWR은 캐시된 것 먼저 보여주고 백그라운드 갱신. HTTP 헤더(CDN)와 TanStack Query(클라이언트) 두 레이어에 각각 존재하며 독립적으로 작동
  - 출발점: revalidate 튜닝이 초기 로딩에 효과 없다는 걸 확인하면서 SWR 개념 정리

- **2026-05-08** [#패턴발견 #nextjs #tanstack-query #성능튜닝] [서버 컴포넌트 + initialData로 첫 화면 즉시 렌더링](./서버컴포넌트-initialData-첫화면-즉시렌더링/index.md)
  - 한 줄 요약: page.tsx를 서버 컴포넌트로 바꾸고 initialDataUpdatedAt: 0을 설정하면, 팀 정보는 즉시 렌더링되고 예측 통계는 백그라운드에서 채워진다
  - 출발점: 매치 상세 페이지 전체가 "use client"라 첫 방문마다 스켈레톤 노출

- **2026-05-08** [#도구습득 #nextjs #react #claude-code] [공유카드 컴팩트 디자인 & Claude 헤드리스 캡처](./공유카드-컴팩트-디자인-및-Claude-헤드리스-캡처/index.md)
  - 한 줄 요약: Claude는 Chrome `--headless=new`로 정적 HTML을 렌더링해 PNG를 찍고 멀티모달로 확인. `/tmp`는 임시라 영구 보존은 `docs/notes-assets/`에 복사해야 함
  - 출발점: 커뮤니티 공유용 경기 리포트 카드 컴팩트 디자인 작업 + 캡처 방식 궁금증
