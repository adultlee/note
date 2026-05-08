# BetResultModal 빌드 실패 분석

> **발생일:** 2026-04-30  
> **해결 시간:** ~5분  
> **심각도:** 🔴 빌드 블로킹 (배포 불가)

---

## 증상

```
Failed to compile.
./src/components/BetResultModal.tsx:49:45
Type error: Property 'fanBreakdown' does not exist on type
'{ totalPredictions: number; teamAPercent: number; ... } | undefined'.
```

Next.js 빌드(`npm run build`)가 실패했고, 로컬 개발 서버(`npm run dev`)는 정상 동작했습니다.

---

## 원인 1 — 잘못된 타입 인덱싱

### 문제 코드

```ts
// ❌ BetResultModal.tsx (수정 전)
function FanCrossLine({ fanBreakdown, predictedTeam, myTeamName }: {
  fanBreakdown: MatchData["predictionStats"]["fanBreakdown"];  // 49번 줄
  ...
})
```

### 타입 구조

`src/types/index.ts`의 `MatchData`는 다음과 같이 정의돼 있습니다.

```ts
interface MatchData {
  predictionStats?: {          // ← optional (undefined 가능)
    totalPredictions: number
    teamAPercent: number
    fanBreakdown?: Array<{     // ← optional (undefined 가능)
      team: TeamData
      count: number
      ...
    }>
  }
}
```

### 왜 에러가 나는가

```
MatchData["predictionStats"]
  = { totalPredictions: number; ... fanBreakdown?: ... } | undefined
                                                          ^^^^^^^^^^
                                                          undefined도 포함!
```

`predictionStats` 자체가 `optional` (`?`) 이기 때문에 타입은 `객체 | undefined` 입니다.  
`undefined["fanBreakdown"]` 는 존재하지 않으므로 TypeScript가 에러를 냅니다.

```
┌─────────────────────────────────────┐
│  MatchData                          │
│                                     │
│  predictionStats ──┬── { ... }      │
│      (optional)    │                │
│                    └── undefined    │
│                         │           │
│                         ▼           │
│                    ["fanBreakdown"] │
│                    = ❌ 접근 불가   │
└─────────────────────────────────────┘
```

---

## 원인 2 — .next 캐시 문제

로컬에서 `tsc --noEmit`은 통과했지만 **빌드는 실패**했습니다.

이는 `.next/` 디렉토리에 **이전 컴파일 결과물이 캐시**되어, 파일을 수정했어도 변경 전 버전을 참조했기 때문입니다.

```
수정 전 파일 ──→ .next 캐시에 저장
       ↓
파일 수정 (타입 선언 추가)
       ↓
npm run build
       ↓
.next 캐시 참조 → 수정 전 코드로 타입 검사 → ❌ 실패
```

---

## 해결 방법

### Step 1 — 타입 안전하게 선언

`NonNullable`을 중첩 사용해 `undefined`를 제거합니다.

```ts
// ✅ BetResultModal.tsx (수정 후)

// predictionStats가 undefined일 수 있으므로 NonNullable로 감싸서 추출
type FanBreakdownItem = NonNullable<
  NonNullable<MatchData["predictionStats"]>["fanBreakdown"]
>[number];

function FanCrossLine({ fanBreakdown, predictedTeam, myTeamName }: {
  fanBreakdown: FanBreakdownItem[] | undefined;  // 명시적으로 undefined 허용
  ...
})
```

```
NonNullable<MatchData["predictionStats"]>
  = { totalPredictions: number; ... fanBreakdown?: ... }  (undefined 제거)
        ↓
["fanBreakdown"]
  = Array<{ team: TeamData; count: number; ... }> | undefined
        ↓
NonNullable<...>
  = Array<{ team: TeamData; count: number; ... }>
        ↓
[number]
  = { team: TeamData; count: number; ... }  (배열 원소 타입)
```

### Step 2 — .next 캐시 삭제

```bash
rm -rf .next
npm run build
```

---

## 왜 dev에서는 안 잡혔는가?

| 환경 | 동작 |
|------|------|
| `npm run dev` | 증분 컴파일, 타입 에러를 경고로 표시하지만 실행은 허용 |
| `npm run build` | 전체 타입 검사 후 에러 시 빌드 중단 (strict mode) |
| `tsc --noEmit` | 타입 검사만 수행 (캐시 영향 없음) |

`dev` 모드에서 타입 에러가 터미널에 보여도 화면은 정상으로 보이기 때문에 놓치기 쉽습니다.

---

## 재발 방지

### 1. optional 체이닝이 있는 타입 인덱싱 시 항상 NonNullable 사용

```ts
// ❌ optional 프로퍼티를 직접 인덱싱
type Bad = SomeInterface["optionalProp"]["nestedProp"]

// ✅ NonNullable로 undefined 제거 후 인덱싱
type Good = NonNullable<SomeInterface["optionalProp"]>["nestedProp"]
```

### 2. 배포 전 빌드 확인 습관

```bash
# 캐시 없이 클린 빌드
rm -rf .next && npm run build
```

### 3. .next는 소스가 아님

`.next/` 는 빌드 산출물이므로 `.gitignore`에 포함돼 있어야 합니다.  
캐시 문제가 의심될 때 항상 삭제 후 재빌드합니다.
