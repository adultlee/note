# 제목 — "Prisma generate 누락 시 빌드 에러"

> 작성일: 2026-05-07  
> 태그: #prisma #codegen #typescript #빌드에러  
> 출발점: `ProbabilitySnapshot` 모델 추가 후 빌드 실패  
> 원본 기록: [../backlog.md](../backlog.md)

## 한 줄 요약

Prisma 스키마에 모델 추가 → `npx prisma generate` 필수. 안 하면 TS가 해당 모델을 모른다.

## 배경 지식

Prisma는 `schema.prisma`를 소스로, `node_modules/@prisma/client` 안에 TypeScript 타입을 **코드 생성**해서 집어넣는다.

즉, `prisma.probabilitySnapshot` 같은 접근자는 실제 JS 코드 + TS 타입이 `@prisma/client` 안에 존재해야만 동작한다.

```
schema.prisma  →  prisma generate  →  @prisma/client (타입 + 런타임 코드)
```

`schema.prisma`는 사람이 읽는 선언이고, `@prisma/client`는 그걸 컴파일한 결과물. 둘 사이의 동기화는 자동이 아니라 **명시적 generate** 커맨드로 이루어짐.

## 동작 원리 / 메커니즘

1. `prisma/schema.prisma`에 `model ProbabilitySnapshot { ... }` 추가
2. 이 시점에서 `@prisma/client`는 아직 이전 버전 → `PrismaClient`에 `probabilitySnapshot` 프로퍼티 없음
3. TypeScript가 `prisma.probabilitySnapshot.findMany(...)` 를 보고 타입 에러 발생

```
Type error: Property 'probabilitySnapshot' does not exist on type 'PrismaClient<...>'
```

4. `npx prisma generate` 실행 → `@prisma/client` 재생성 → 타입 추가됨 → 에러 해소

```bash
npx prisma generate
# → Generated Prisma Client to ./node_modules/@prisma/client
```

## 어떤 상황에서 마주쳤나

`impact/route.ts`에서 경기 결과 확정 전후 LCK 확률 비교를 위해 `ProbabilitySnapshot` 스냅샷을 조회하는 로직을 추가했음.

`schema.prisma`에 모델 정의는 들어가 있었는데, `prisma generate`를 실행하지 않은 상태로 Vercel 빌드가 돌아서 타입 에러로 빌드 실패.

## 헷갈렸던 부분 / 함정

- "스키마에 있으면 당연히 타입도 있겠지" → **아님**. generate 해야 반영됨
- `prisma db push` / `prisma migrate dev`는 DB에 테이블을 만드는 것이고, `prisma generate`는 클라이언트 코드를 만드는 것. 목적이 다름
- `migrate dev`를 실행하면 내부적으로 `generate`도 같이 돌아줌 → 마이그레이션할 때는 자동. 스키마만 손으로 수정했을 때는 수동으로 해야 함

## 응용·확장

- CI/CD 파이프라인에서는 빌드 스텝 전에 `prisma generate`를 명시적으로 넣어야 함
- Vercel 배포 시 `package.json`의 `postinstall`에 `prisma generate`를 등록하면 자동화 가능:
  ```json
  "scripts": {
    "postinstall": "prisma generate"
  }
  ```
- 이 프로젝트는 `"build": "prisma generate && next build"`로 빌드 시 자동 실행됨. 그런데도 에러가 났다면 로컬 `node_modules/@prisma/client`가 오래된 상태에서 tsc를 별도로 돌렸거나, Vercel 빌드 캐시 문제일 가능성 있음

## 참고 자료

- [Prisma Generate 공식 문서](https://www.prisma.io/docs/orm/reference/prisma-cli-reference#generate) — generate 옵션 전체 정리
