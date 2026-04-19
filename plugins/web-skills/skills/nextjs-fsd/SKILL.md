---
name: nextjs-fsd
description: Next.js App Router + Tailwind CSS v4 Feature-Sliced Design 퍼블리싱. FSD 레이어 규칙 엄수, Mobile First 풀 반응형. DI 컨테이너, React Query, Redux 패턴 포함.
argument-hint: [페이지명 또는 요구사항]
allowed-tools: Read Grep Glob Write Edit Bash
effort: high
---

# Next.js FSD (Feature-Sliced Design) 퍼블리싱 스킬

Next.js App Router + Tailwind CSS로 FSD 아키텍처 기반 반응형 웹사이트를 구현한다.
**FSD 레이어 간 임포트 규칙을 반드시 준수**하며, 기존 프로젝트의 패턴을 따른다.
CBD 패턴이 필요한 경우 `/nextjs-cbd` 스킬을 사용할 것.

## 기술 스택

- **Next.js 14+** App Router (`src/app/` 디렉토리)
- **Tailwind CSS v4** (`@import "tailwindcss"`, `@theme inline`, Mobile First)
- **TypeScript** 기본
- **Redux Toolkit** + redux-persist (상태관리)
- **React Query** (@tanstack/react-query, 서버 상태)
- **DI 컨테이너** (ServiceContainer 기반 서비스 주입)
- **Biome** (린터/포매터)

## Tailwind CSS v4 설정

프로젝트는 Tailwind v4를 사용한다. v3과 다른 핵심 차이:

- `tailwind.config.ts` 대신 CSS 파일 내 `@theme inline`으로 커스텀 토큰 정의
- `@import "tailwindcss"` 단일 지시어 사용
- `@custom-variant dark (...)` 으로 다크모드 정의
- PostCSS 플러그인: `@tailwindcss/postcss`

```css
/* globals.css */
@import "tailwindcss";
@custom-variant dark (&:where(.dark, .dark *));

@theme inline {
  --color-primary: var(--primary);
  --color-background: var(--bg);
  --color-foreground: var(--fg);
}
```

## FSD 레이어 구조

```
src/
├── app/                          ← Next.js App Router (라우팅만)
│   ├── layout.tsx
│   ├── [lang]/                   ← i18n 동적 세그먼트
│   │   ├── page.tsx
│   │   └── [도메인]/(withLayouts)/layout.tsx
│   └── api/                      ← API Routes
│
└── fsd/                          ← FSD 레이어 루트
    ├── app/                      ← 최상위 레이어: 프로바이더, 레이아웃, 설정
    │   ├── layouts/              ← 도메인별 레이아웃 컴포넌트
    │   ├── store/                ← Redux store 설정
    │   ├── auth/                 ← 인증 프로바이더/가드
    │   ├── i18n/                 ← 국제화 프로바이더
    │   └── react-query/          ← React Query 프로바이더
    │
    ├── pages/                    ← 페이지 레벨 컴포넌트
    │
    ├── widgets/                  ← 대형 UI 조합 (features + entities 결합)
    │   └── [위젯명]/ui/
    │
    ├── features/                 ← 기능 단위 비즈니스 로직
    │   └── [기능명]/
    │       ├── ui/               ← 컴포넌트
    │       ├── model/            ← 타입, 스키마, 상태
    │       └── api/              ← 서비스 훅 (선택)
    │
    ├── entities/                 ← 도메인 엔티티
    │   └── [엔티티명]/
    │       ├── ui/               ← 엔티티 관련 UI
    │       ├── model/
    │       │   ├── server/       ← 서버 응답 타입
    │       │   ├── client/       ← 클라이언트 모델
    │       │   └── mapper/       ← 서버→클라이언트 변환
    │       └── api/
    │           ├── index.ts      ← 서비스 클래스 export
    │           ├── queries.ts    ← React Query 옵션
    │           ├── mutations.ts  ← Mutation 옵션
    │           └── use[Entity]Service.ts  ← 커스텀 훅
    │
    └── shared/                   ← 재사용 UI, 설정, 유틸
        ├── [컴포넌트명]/ui/      ← button, input, modal, alert 등
        ├── config/               ← 공통 설정
        │   ├── store/            ← Redux 타입
        │   ├── i18n/             ← 번역 파일, 타입 생성
        │   ├── mock/             ← MSW 핸들러
        │   ├── proxy/            ← 프록시 설정
        │   ├── theme/            ← 테마 색상/타입
        │   └── service/          ← DI 컨테이너 설정
        ├── validations/          ← Zod 스키마
        └── utils/                ← 공통 유틸리티
```
아까
## FSD 임포트 규칙 (위반 시 lint 에러)

```
shared  → 다른 FSD 레이어 import 금지
entities → features, widgets, pages, app import 금지
features → widgets, pages, app import 금지
widgets  → pages, app import 금지
pages    → app import 금지
```

**상위 레이어만 하위 레이어를 import할 수 있다.**
같은 레이어 내 슬라이스 간 import도 주의 (순환 의존성 방지).

## 경로 별칭

| 별칭 | 경로 |
|------|------|
| `@NextApp/*` | `./src/app/*` |
| `@Src/*` | `./src/*` |
| `@FsdApp/*` | `./src/fsd/app/*` |
| `@FsdPages/*` | `./src/fsd/pages/*` |
| `@FsdWidgets/*` | `./src/fsd/widgets/*` |
| `@FsdFeatures/*` | `./src/fsd/features/*` |
| `@FsdEntities/*` | `./src/fsd/entities/*` |
| `@FsdShared/*` | `./src/fsd/shared/*` |
| `@Libs/*` | `./libs/*` |

## 슬라이스 내부 세그먼트 구조

각 슬라이스(기능/엔티티)는 다음 세그먼트로 구성:

| 세그먼트 | 역할 | 예시 |
|---------|------|------|
| `ui/` | React 컴포넌트 | `LoginForm.tsx` |
| `model/` | 타입, 상태, 스키마 | `types.ts`, `schema.ts`, `slice.ts` |
| `api/` | 서비스 호출, React Query 훅 | `queries.ts`, `mutations.ts` |
| `hooks/` | 커스텀 훅 | `useAuth.ts` |
| `utils/` | 유틸리티 함수 | `format-date.ts` |

**각 세그먼트는 `index.ts`로 public API를 내보낸다:**
```tsx
// features/login/ui/index.ts
export { default as LoginForm } from './LoginForm';
```

## REST 서비스 패턴

### DIP 원칙: 인터페이스에 의존, 구현체에 의존하지 않음

```tsx
// entities/[entity]/api/queries.ts
import { SiteService } from '@FsdEntities/site/api';
import { serviceContainer } from '@FsdShared/config/service/service.setup';
import { SERVICE_NAME } from '@Libs/service-container';

const queryKeys = {
  findGnb: ['findGnb'] as const,
};

const queryOptions = {
  findGnb: () => ({
    queryKey: queryKeys.findGnb,
    queryFn: async () => {
      const service = serviceContainer.get<SiteService>(SERVICE_NAME.SITE);
      const response = await service.getGnb();
      return {
        ...response,
        result: response.result ? mapServerToClient(response.result) : undefined,
      };
    },
  }),
};
```

### 네이밍 규칙

| 종류 | 패턴 | 예시 |
|------|------|------|
| 서비스 메서드 | `{httpMethod}{lastPath}` | `getGnb`, `postUser` |
| Request 타입 | `{LastPath}{Method}Req` | `SearchGNBGetReq` |
| Response 타입 | `{LastPath}{Method}Res` | `SearchGNBGetRes` |
| Query 훅 | `use{Action}{LastPath}Query` | `useFindGnbQuery` |
| Mutation 훅 | `use{Action}{LastPath}Mutation` | `useRegisterTestMutation` |
| queries.ts 함수 | `find*` | `findGnb` |
| mutations.ts 함수 | `register*`, `edit*`, `remove*` | `registerUser` |

### Query 훅 패턴
```tsx
// entities/[entity]/api/use[Entity]Service.ts
import queryOptions from '@FsdEntities/site/api/queries';
import { useQuery } from '@tanstack/react-query';

export const useFindGnbQuery = (lazy?: boolean) =>
  useQuery({
    ...queryOptions.findGnb(),
    enabled: !lazy,
  });
```

## 서버/클라이언트 모델 분리 패턴

```
entities/site/model/
├── server/gnb.ts           ← API 응답 그대로의 타입
├── client/gnb.ts           ← 프론트에서 사용하는 가공된 타입
└── mapper/map-server-gnb-to-client.ts  ← 변환 함수
```

서버 모델을 직접 UI에서 쓰지 않고, mapper를 통해 클라이언트 모델로 변환 후 사용한다.

## Mobile First 반응형 규칙

Tailwind 기본 브레이크포인트 (min-width 기반):

```
기본값     = 모바일 (~639px)    → 클래스에 접두사 없음
sm:       = ≥ 640px           → 큰 모바일 / 소형 태블릿
md:       = ≥ 768px           → 태블릿
lg:       = ≥ 1024px          → 데스크톱
xl:       = ≥ 1280px          → 큰 데스크톱
2xl:      = ≥ 1536px          → 와이드 모니터
```

**예시:**
```tsx
{/* 1열 → 2열 → 3열 */}
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 md:gap-8">

{/* 세로 → 가로 */}
<div className="flex flex-col md:flex-row items-center gap-6">

{/* 모바일 작게 → PC 크게 */}
<h1 className="text-3xl md:text-4xl lg:text-5xl font-bold">
```

## 컴포넌트 작성 규칙

- 한 파일 200줄 이내, 초과 시 분리
- Props는 `interface`로 타입 정의
- Server Components 기본, 인터랙션 필요시만 `'use client'`
- 클라이언트 컴포넌트는 최소 범위로 분리
- 각 슬라이스의 `ui/index.ts`를 통해 export

## 파일 네이밍 규칙

| 타입 | 컨벤션 | 예시 |
|------|--------|------|
| React 컴포넌트 | PascalCase | `LoginForm.tsx` |
| Next.js 페이지 | lowercase | `page.tsx`, `layout.tsx` |
| 유틸리티 | kebab-case | `format-date.ts` |
| 커스텀 훅 | camelCase | `useAuth.ts` |
| 타입/모델 | kebab-case | `user-model.ts` |
| 스키마(Zod) | kebab-case | `user-schema.ts` |

## Biome 포매팅 규칙

- 들여쓰기: 2칸
- 줄 너비: 120자
- 따옴표: JS/TS는 작은따옴표, JSX는 큰따옴표
- 트레일링 콤마: ES5
- 세미콜론: 항상
- `console.log` 금지 (`console.error`, `console.info`만 허용)

## 구현 프로세스

### 1단계: 기존 프로젝트 파악
- FSD 구조와 기존 패턴 확인
- 경로 별칭, 서비스 컨테이너 설정 확인
- 기존 shared 컴포넌트 재사용 가능 여부 확인

### 2단계: 슬라이스 배치 결정
- 새 기능이 어느 레이어에 속하는지 결정
- 기존 엔티티/피처와의 관계 파악
- 임포트 규칙 위반 여부 사전 확인

### 3단계: 코드 작성
- 레이어 규칙에 맞게 파일 배치
- `index.ts`로 public API export
- 서버 모델 → mapper → 클라이언트 모델 패턴 준수

### 4단계: 검증
- Biome 린트 통과 확인 (`npm run lint`)
- FSD 임포트 규칙 위반 없는지 확인
- TypeScript 타입 체크 통과 (`npm run typecheck`)

## 상세 패턴은 [reference.md](reference.md) 참조

사용자 인자: `$ARGUMENTS`로 전달된 페이지명이나 요구사항을 기반으로 구현을 시작한다.
