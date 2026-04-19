# Next.js FSD 퍼블리싱 레퍼런스

## FSD 슬라이스 생성 가이드

### 새 feature 추가 시
```
src/fsd/features/[feature-name]/
├── ui/
│   ├── [FeatureName]Form.tsx   ← 또는 적절한 컴포넌트명
│   └── index.ts                ← public export
├── model/
│   ├── types.ts                ← 타입 정의
│   ├── schema.ts               ← Zod 스키마 (폼 검증 시)
│   ├── index.ts                ← public export
│   └── [feature-name]-slice.ts ← Redux slice (상태 필요 시)
└── api/                        ← API 호출 필요 시
    ├── queries.ts
    ├── mutations.ts
    └── index.ts
```

### 새 entity 추가 시
```
src/fsd/entities/[entity-name]/
├── ui/
│   ├── [EntityName]Card.tsx
│   └── index.ts
├── model/
│   ├── server/                 ← API 응답 타입
│   │   └── [entity].ts
│   ├── client/                 ← 프론트 사용 타입
│   │   └── [entity].ts
│   └── mapper/                 ← 서버→클라이언트 변환
│       └── map-server-[entity]-to-client.ts
└── api/
    ├── index.ts                ← 서비스 클래스 re-export
    ├── queries.ts              ← queryKeys + queryOptions
    ├── mutations.ts            ← mutationOptions
    └── use[Entity]Service.ts   ← 커스텀 query/mutation 훅
```

### 새 widget 추가 시
```
src/fsd/widgets/[widget-name]/
├── ui/
│   ├── [WidgetName].tsx
│   └── index.ts
└── model/                      ← 위젯 내부 상태 필요 시
    └── types.ts
```

### shared UI 추가 시
```
src/fsd/shared/[component-name]/
├── ui/
│   ├── [ComponentName].tsx
│   └── index.ts
├── model/                      ← 타입/상태 필요 시
│   └── types.ts
└── hooks/                      ← 관련 훅 필요 시
    └── use[Something].ts
```

## Provider 계층 구조

```
StoreProvider (Redux + redux-persist)
  → ReactQueryProvider (@tanstack/react-query)
    → NextAuthProvider (next-auth)
      → AuthProvider (세션 → 서비스 컨테이너 동기화)
        → MockServerInit (MSW)
          → I18nProvider (locale from [lang] route param)
```

새 프로바이더 추가 시 이 계층에 적절히 삽입한다.

## i18n 패턴

### 라우트 구조
```
src/app/[lang]/           ← en, ko, ja 등
├── page.tsx
└── [도메인]/
    └── page.tsx
```

### Server Component에서 번역
```tsx
import { getI18nTranslator } from '@FsdShared/config/i18n/utils/get-i18n-translator';

export default async function Page({ params }: { params: { lang: string } }) {
  const t = await getI18nTranslator(params.lang, 'common');
  return <h1>{t('title')}</h1>;
}
```

### Client Component에서 번역
```tsx
'use client';
import { useI18n } from '@FsdShared/config/i18n/hooks/useI18n';

export default function Component() {
  const { t } = useI18n('common');
  return <p>{t('description')}</p>;
}
```

### 번역 파일 추가 후
```bash
npm run gen:i18n           # 타입 + 네임스페이스 전체 생성
npm run gen:i18n-types     # 타입만 재생성
```

## Hydration 전략

| 시나리오 | 방법 |
|---------|------|
| 순수 Server Component | 페이지에서 직접 fetch → props 전달 |
| Client + SSR 필요 | React Query hydration 사용 |
| Client + CSR만 | hooks 직접 사용 |

### React Query Hydration 패턴
```tsx
// app/[lang]/page.tsx (Server Component)
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query';
import queryOptions from '@FsdEntities/site/api/queries';

export default async function Page() {
  const queryClient = new QueryClient();
  await queryClient.prefetchQuery(queryOptions.findGnb());

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ClientComponent />
    </HydrationBoundary>
  );
}
```

## Redux Slice 패턴

```tsx
// shared/alert/model/common-alert-slice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface CommonAlertState {
  isShow: boolean;
  messageHTML: string;
  okBtnName?: string;
  cancelBtnName?: string;
}

const initialState: CommonAlertState = {
  isShow: false,
  messageHTML: '',
};

const commonAlertSlice = createSlice({
  name: 'commonAlert',
  initialState,
  reducers: {
    showCommonAlert: (state, action: PayloadAction<Partial<CommonAlertState>>) => {
      return { ...state, ...action.payload, isShow: true };
    },
    hideCommonAlert: () => initialState,
  },
});

export const { showCommonAlert, hideCommonAlert } = commonAlertSlice.actions;
export default commonAlertSlice.reducer;
```

## 서비스 컨테이너 (DI) 패턴

```tsx
// shared/config/service/service.setup.ts
import { ServiceContainer } from '@Libs/service-container';

export const serviceContainer = new ServiceContainer();

// 서비스 등록
serviceContainer.bind(SERVICE_NAME.SITE, SiteService);

// 서비스 사용 (entities/[entity]/api/ 내에서)
const service = serviceContainer.get<SiteService>(SERVICE_NAME.SITE);
```

## 인증 가드 패턴

```tsx
// app/auth/guards/AuthGuard.tsx — 로그인 필수 페이지
// app/auth/guards/GuestGuard.tsx — 비로그인 전용 (로그인 페이지 등)
```

레이아웃에서 가드로 감싸서 사용:
```tsx
<AuthGuard>
  <CommerceLayout>{children}</CommerceLayout>
</AuthGuard>
```

## Tailwind v4 Mobile First 패턴

```tsx
{/* 1열 → 2열 → 4열 */}
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4 md:gap-8">

{/* PC에서만 표시 */}
<div className="hidden md:block">PC 전용</div>

{/* 모바일에서만 표시 */}
<div className="block md:hidden">모바일 전용</div>

{/* 반응형 텍스트 */}
<h1 className="text-3xl md:text-4xl lg:text-5xl font-bold">

{/* 반응형 간격 */}
<section className="py-12 md:py-16 lg:py-24">
```

### 다크모드 (v4 CSS 변수 활용)
```tsx
{/* @theme inline 토큰 사용 */}
<div className="bg-background text-foreground">
<div className="bg-card text-card-foreground border border-border">
<button className="bg-primary hover:bg-primary-hover text-white">

{/* dark: 접두사 직접 사용 */}
<div className="bg-white dark:bg-gray-900">
```

## 새 페이지 추가 체크리스트

1. [ ] `src/app/[lang]/[도메인]/page.tsx` 생성
2. [ ] 필요 시 `(withLayouts)/layout.tsx` 추가
3. [ ] FSD 레이어에 슬라이스 배치
   - shared: 재사용 UI
   - entities: 도메인 모델/API
   - features: 기능 로직/UI
   - widgets: 대형 조합 UI
4. [ ] 각 슬라이스 `index.ts` export 작성
5. [ ] 임포트 규칙 위반 없는지 확인
6. [ ] i18n 번역 키 추가 후 `npm run gen:i18n`
7. [ ] `npm run lint` 통과 확인

## MSW 목 핸들러 패턴

```tsx
// shared/config/mock/handlers/[domain]/[domain]-mock-handler.ts
import { http, HttpResponse } from 'msw';

export const siteMockHandlers = [
  http.get('/api/site/gnb', () => {
    return HttpResponse.json({
      success: true,
      result: [/* mock data */],
    });
  }),
];
```

핸들러를 `shared/config/mock/handlers/index.ts`에 등록.
