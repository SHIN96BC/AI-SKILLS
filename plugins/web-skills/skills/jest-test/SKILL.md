---
name: jest-test
description: Jest 기반 테스트 코드 작성. 유닛/통합 테스트, React 컴포넌트 테스트, 훅 테스트, API 서비스 테스트. CBD/FSD 구조 호환.
argument-hint: [테스트 대상 파일 또는 기능]
allowed-tools: Read Grep Glob Write Edit Bash
effort: high
---

# Jest 테스트 스킬

Jest를 사용하여 테스트 코드를 작성한다.
React Testing Library와 함께 컴포넌트/훅 테스트를 지원하며, CBD/FSD 양쪽 프로젝트 구조에서 사용 가능하다.

## 기술 스택

- **Jest 30+** (테스트 러너)
- **jest-environment-jsdom** (DOM 환경)
- **@testing-library/react** (컴포넌트 테스트)
- **@testing-library/jest-dom** (DOM 매처 확장)
- **@testing-library/user-event** (사용자 인터랙션 시뮬레이션)
- **MSW** (API 모킹, 프로젝트에 이미 있으면 활용)

## 테스트 작성 프로세스

### 1단계: 기존 프로젝트 파악
- jest.config 또는 package.json의 jest 설정 확인
- 기존 테스트 파일 패턴 확인 (`*.test.ts`, `*.spec.ts`, `__tests__/`)
- 설치된 테스트 관련 패키지 확인
- **미설치 패키지가 있으면 설치 명령어 안내:**

```bash
# 필수 패키지 (프로젝트에 없으면 안내)
npm install -D @testing-library/react @testing-library/jest-dom @testing-library/user-event

# Jest 30 + jsdom (이미 있으면 스킵)
npm install -D jest jest-environment-jsdom

# TypeScript 지원 (필요 시)
npm install -D ts-jest @types/jest
```

- **jest.config.ts가 없으면 생성 안내:**

```ts
// jest.config.ts
import type { Config } from 'jest';
import nextJest from 'next/jest';

const createJestConfig = nextJest({ dir: './' });

const config: Config = {
  testEnvironment: 'jsdom',
  setupFilesAfterSetup: ['<rootDir>/jest.setup.ts'],
  moduleNameMapper: {
    // CBD 프로젝트
    '^@/(.*)$': '<rootDir>/$1',
    // FSD 프로젝트 (필요한 별칭만 추가)
    '^@FsdShared/(.*)$': '<rootDir>/src/fsd/shared/$1',
    '^@FsdEntities/(.*)$': '<rootDir>/src/fsd/entities/$1',
    '^@FsdFeatures/(.*)$': '<rootDir>/src/fsd/features/$1',
    '^@FsdWidgets/(.*)$': '<rootDir>/src/fsd/widgets/$1',
    '^@FsdApp/(.*)$': '<rootDir>/src/fsd/app/$1',
    '^@Libs/(.*)$': '<rootDir>/libs/$1',
  },
  testPathIgnorePatterns: ['<rootDir>/node_modules/', '<rootDir>/.next/'],
};

export default createJestConfig(config);
```

- **jest.setup.ts가 없으면 생성 안내:**

```ts
// jest.setup.ts
import '@testing-library/jest-dom';
```

### 2단계: 테스트 대상 코드 분석
- 테스트할 파일의 export 함수/컴포넌트 파악
- 의존성과 사이드이펙트 확인
- 모킹이 필요한 외부 의존성 식별

### 3단계: 테스트 코드 작성
- 테스트 파일은 대상 파일과 같은 디렉토리에 배치
- 파일명: `[대상파일명].test.ts(x)`
- describe/it 구조로 논리적 그룹핑

## 테스트 파일 배치 규칙

```
# 대상 파일 옆에 테스트 파일 배치
components/ui/Button.tsx
components/ui/Button.test.tsx    ← 같은 디렉토리

# FSD 구조도 동일
src/fsd/shared/button/ui/Button.tsx
src/fsd/shared/button/ui/Button.test.tsx

# 유틸리티
utils/format-date.ts
utils/format-date.test.ts

# 훅
hooks/useAuth.ts
hooks/useAuth.test.ts
```

## 테스트 실행 명령어

```bash
npm run test                    # 전체 테스트
npm run test:watch              # Watch 모드
npm run test:coverage           # 커버리지 리포트
npx jest [파일명]               # 단일 파일 실행
npx jest --testPathPattern="Button"  # 패턴 매칭 실행
```

## 테스트 작성 패턴

### 유닛 테스트 (순수 함수)
```tsx
// utils/format-date.test.ts
import { formatDate } from './format-date';

describe('formatDate', () => {
  it('날짜를 YYYY-MM-DD 형식으로 변환한다', () => {
    const date = new Date('2024-03-15');
    expect(formatDate(date)).toBe('2024-03-15');
  });

  it('잘못된 입력에 대해 빈 문자열을 반환한다', () => {
    expect(formatDate(null)).toBe('');
  });
});
```

### React 컴포넌트 테스트
```tsx
// components/ui/Button.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import Button from './Button';

describe('Button', () => {
  it('텍스트가 렌더링된다', () => {
    render(<Button>클릭</Button>);
    expect(screen.getByRole('button', { name: '클릭' })).toBeInTheDocument();
  });

  it('클릭 시 onClick이 호출된다', async () => {
    const user = userEvent.setup();
    const onClick = jest.fn();
    render(<Button onClick={onClick}>클릭</Button>);

    await user.click(screen.getByRole('button'));
    expect(onClick).toHaveBeenCalledTimes(1);
  });

  it('disabled 상태에서 클릭이 무시된다', async () => {
    const user = userEvent.setup();
    const onClick = jest.fn();
    render(<Button onClick={onClick} disabled>클릭</Button>);

    await user.click(screen.getByRole('button'));
    expect(onClick).not.toHaveBeenCalled();
  });
});
```

### 커스텀 훅 테스트
```tsx
// hooks/useCounter.test.ts
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('초기값으로 시작한다', () => {
    const { result } = renderHook(() => useCounter(10));
    expect(result.current.count).toBe(10);
  });

  it('increment로 1 증가한다', () => {
    const { result } = renderHook(() => useCounter(0));
    act(() => result.current.increment());
    expect(result.current.count).toBe(1);
  });
});
```

### API 서비스 테스트 (MSW 활용)
```tsx
// entities/site/api/queries.test.ts
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

const server = setupServer(
  http.get('/api/site/gnb', () => {
    return HttpResponse.json({
      success: true,
      result: [{ id: 1, name: '홈' }],
    });
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('site queries', () => {
  it('GNB 목록을 가져온다', async () => {
    // React Query + renderHook 조합으로 테스트
  });

  it('서버 에러 시 에러를 반환한다', async () => {
    server.use(
      http.get('/api/site/gnb', () => {
        return new HttpResponse(null, { status: 500 });
      })
    );
    // 에러 케이스 테스트
  });
});
```

### Redux Slice 테스트
```tsx
// shared/alert/model/common-alert-slice.test.ts
import reducer, { showCommonAlert, hideCommonAlert } from './common-alert-slice';

describe('commonAlertSlice', () => {
  it('초기 상태가 올바르다', () => {
    expect(reducer(undefined, { type: 'unknown' })).toEqual({
      isShow: false,
      messageHTML: '',
    });
  });

  it('showCommonAlert로 알림을 표시한다', () => {
    const state = reducer(undefined, showCommonAlert({
      messageHTML: '<p>테스트</p>',
    }));
    expect(state.isShow).toBe(true);
    expect(state.messageHTML).toBe('<p>테스트</p>');
  });

  it('hideCommonAlert로 초기 상태로 돌아간다', () => {
    const shown = reducer(undefined, showCommonAlert({ messageHTML: 'test' }));
    const hidden = reducer(shown, hideCommonAlert());
    expect(hidden.isShow).toBe(false);
  });
});
```

## 모킹 패턴

### 모듈 모킹
```tsx
// next/navigation 모킹
jest.mock('next/navigation', () => ({
  useRouter: () => ({ push: jest.fn(), back: jest.fn() }),
  usePathname: () => '/test',
  useSearchParams: () => new URLSearchParams(),
}));

// next/image 모킹
jest.mock('next/image', () => ({
  __esModule: true,
  default: (props: any) => <img {...props} />,
}));
```

### 서비스 컨테이너 모킹 (FSD)
```tsx
jest.mock('@FsdShared/config/service/service.setup', () => ({
  serviceContainer: {
    get: jest.fn().mockReturnValue({
      getGnb: jest.fn().mockResolvedValue({ success: true, result: [] }),
    }),
  },
}));
```

### React Query Provider 래퍼
```tsx
// test-utils/create-query-wrapper.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

export function createQueryWrapper() {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });

  return function Wrapper({ children }: { children: React.ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    );
  };
}

// 사용
const { result } = renderHook(() => useFindGnbQuery(), {
  wrapper: createQueryWrapper(),
});
```

### Redux Store 래퍼
```tsx
// test-utils/create-store-wrapper.tsx
import { configureStore } from '@reduxjs/toolkit';
import { Provider } from 'react-redux';

export function createStoreWrapper(preloadedState = {}) {
  const store = configureStore({
    reducer: { /* 필요한 리듀서 */ },
    preloadedState,
  });

  return function Wrapper({ children }: { children: React.ReactNode }) {
    return <Provider store={store}>{children}</Provider>;
  };
}
```

## 테스트 작성 원칙

### 무엇을 테스트하나
- **순수 함수**: 입력/출력 검증
- **컴포넌트**: 렌더링 결과, 사용자 인터랙션, 조건부 렌더링
- **훅**: 상태 변화, 반환값
- **Redux Slice**: 리듀서 로직, 액션
- **API 서비스**: 요청/응답, 에러 처리

### 무엇을 테스트하지 않나
- 서드파티 라이브러리 내부 동작
- 구현 세부사항 (내부 state, private 메서드)
- 스타일/CSS 클래스 (스냅샷으로 대체 가능)

### 네이밍 규칙
- `describe`: 테스트 대상 (함수명, 컴포넌트명)
- `it`: 한국어로 기대 동작 서술 (`it('버튼 클릭 시 모달이 열린다')`)
- 테스트 제목만으로 무엇을 검증하는지 알 수 있어야 함

### AAA 패턴
```tsx
it('장바구니에 상품을 추가한다', () => {
  // Arrange — 준비
  const cart = new Cart();
  const product = { id: 1, name: '상품' };

  // Act — 실행
  cart.add(product);

  // Assert — 검증
  expect(cart.items).toHaveLength(1);
  expect(cart.items[0]).toEqual(product);
});
```

## 상세 패턴은 [reference.md](reference.md) 참조

사용자 인자: `$ARGUMENTS`로 전달된 파일명이나 기능을 분석하여 테스트 대상을 파악하고 테스트 코드를 작성한다.
