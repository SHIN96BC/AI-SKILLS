# Jest 테스트 레퍼런스

## Jest 설정 (Next.js)

### jest.config.ts (next/jest 사용)
```ts
import type { Config } from 'jest';
import nextJest from 'next/jest';

const createJestConfig = nextJest({ dir: './' });

const config: Config = {
  testEnvironment: 'jsdom',
  setupFilesAfterSetup: ['<rootDir>/jest.setup.ts'],
  moduleNameMapper: {
    // 프로젝트 경로 별칭 매핑
    '^@/(.*)$': '<rootDir>/$1',
    // FSD 경로 별칭
    '^@FsdShared/(.*)$': '<rootDir>/src/fsd/shared/$1',
    '^@FsdEntities/(.*)$': '<rootDir>/src/fsd/entities/$1',
    '^@FsdFeatures/(.*)$': '<rootDir>/src/fsd/features/$1',
    '^@FsdWidgets/(.*)$': '<rootDir>/src/fsd/widgets/$1',
    '^@FsdApp/(.*)$': '<rootDir>/src/fsd/app/$1',
    '^@Libs/(.*)$': '<rootDir>/libs/$1',
  },
  testPathIgnorePatterns: ['<rootDir>/node_modules/', '<rootDir>/.next/'],
  coveragePathIgnorePatterns: ['node_modules', '.next', 'test-utils'],
};

export default createJestConfig(config);
```

### jest.setup.ts
```ts
import '@testing-library/jest-dom';

// 전역 모킹
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: jest.fn().mockImplementation((query: string) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: jest.fn(),
    removeListener: jest.fn(),
    addEventListener: jest.fn(),
    removeEventListener: jest.fn(),
    dispatchEvent: jest.fn(),
  })),
});

// IntersectionObserver 모킹
class MockIntersectionObserver {
  observe = jest.fn();
  unobserve = jest.fn();
  disconnect = jest.fn();
}
Object.defineProperty(window, 'IntersectionObserver', {
  writable: true,
  value: MockIntersectionObserver,
});
```

## Testing Library 쿼리 우선순위

접근성 기반 쿼리를 우선 사용한다 (사용자가 실제로 인식하는 방식):

| 우선순위 | 쿼리 | 용도 |
|---------|------|------|
| 1 | `getByRole` | 버튼, 링크, 헤딩 등 ARIA 역할 |
| 2 | `getByLabelText` | 폼 요소 (input, select) |
| 3 | `getByPlaceholderText` | label 없는 input |
| 4 | `getByText` | 비인터랙티브 텍스트 |
| 5 | `getByDisplayValue` | 현재 값이 있는 input |
| 6 | `getByAltText` | 이미지 |
| 7 | `getByTitle` | title 속성 |
| 8 | `getByTestId` | 최후의 수단 |

### 쿼리 변형

| 접두사 | 없을 때 | 여러 개 |
|--------|---------|---------|
| `getBy` | 에러 throw | `getAllBy` |
| `queryBy` | null 반환 | `queryAllBy` |
| `findBy` | Promise (비동기) | `findAllBy` |

```tsx
// 요소가 있는지 확인
expect(screen.getByRole('button', { name: '저장' })).toBeInTheDocument();

// 요소가 없는지 확인
expect(screen.queryByText('에러 메시지')).not.toBeInTheDocument();

// 비동기로 요소 등장 대기
const modal = await screen.findByRole('dialog');
```

## 자주 쓰는 jest-dom 매처

```tsx
// 존재 여부
expect(element).toBeInTheDocument();
expect(element).not.toBeInTheDocument();

// 가시성
expect(element).toBeVisible();
expect(element).not.toBeVisible();

// 비활성화
expect(button).toBeDisabled();
expect(button).toBeEnabled();

// 텍스트 내용
expect(element).toHaveTextContent('안녕');
expect(element).toHaveTextContent(/안녕/);

// CSS 클래스
expect(element).toHaveClass('active');

// 속성
expect(link).toHaveAttribute('href', '/about');

// 폼 값
expect(input).toHaveValue('test@email.com');
expect(checkbox).toBeChecked();

// 포커스
expect(input).toHaveFocus();
```

## userEvent 인터랙션 패턴

```tsx
import userEvent from '@testing-library/user-event';

// 항상 setup() 호출
const user = userEvent.setup();

// 클릭
await user.click(screen.getByRole('button'));

// 더블 클릭
await user.dblClick(element);

// 텍스트 입력
await user.type(screen.getByRole('textbox'), 'hello');

// 기존 값 지우고 입력
await user.clear(screen.getByRole('textbox'));
await user.type(screen.getByRole('textbox'), 'new value');

// 키보드
await user.keyboard('{Enter}');
await user.keyboard('{Escape}');

// 탭 이동
await user.tab();

// 셀렉트
await user.selectOptions(screen.getByRole('combobox'), 'option1');

// 호버
await user.hover(element);
await user.unhover(element);
```

## 비동기 테스트 패턴

### waitFor — 조건이 만족될 때까지 대기
```tsx
import { waitFor } from '@testing-library/react';

await waitFor(() => {
  expect(screen.getByText('로딩 완료')).toBeInTheDocument();
});
```

### findBy — 요소 등장 대기 (waitFor + getBy 축약)
```tsx
const element = await screen.findByText('데이터 로드됨');
expect(element).toBeInTheDocument();
```

### act — 상태 업데이트 래핑
```tsx
import { act } from '@testing-library/react';

await act(async () => {
  fireEvent.click(button);
});
```

## 스냅샷 테스트

```tsx
it('컴포넌트가 올바르게 렌더링된다', () => {
  const { container } = render(<Card title="테스트" description="설명" />);
  expect(container.firstChild).toMatchSnapshot();
});

// 인라인 스냅샷
it('포맷 결과를 확인한다', () => {
  expect(formatPrice(1000)).toMatchInlineSnapshot(`"₩1,000"`);
});
```

## React Query 테스트

### Query 훅 테스트
```tsx
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

function createWrapper() {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: Infinity },
    },
  });
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
}

it('데이터를 성공적으로 가져온다', async () => {
  const { result } = renderHook(() => useFindGnbQuery(), {
    wrapper: createWrapper(),
  });

  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data?.result).toHaveLength(3);
});
```

### Mutation 훅 테스트
```tsx
it('등록 mutation이 성공한다', async () => {
  const { result } = renderHook(() => useRegisterTestMutation(), {
    wrapper: createWrapper(),
  });

  await act(async () => {
    result.current.mutate({ name: '테스트' });
  });

  await waitFor(() => expect(result.current.isSuccess).toBe(true));
});
```

## Redux 통합 테스트

### 컴포넌트 + Redux Store
```tsx
import { configureStore } from '@reduxjs/toolkit';
import { Provider } from 'react-redux';
import { render, screen } from '@testing-library/react';

function renderWithStore(ui: React.ReactElement, preloadedState = {}) {
  const store = configureStore({
    reducer: {
      commonAlert: commonAlertReducer,
      // 필요한 리듀서 추가
    },
    preloadedState,
  });

  return render(<Provider store={store}>{ui}</Provider>);
}

it('알림이 표시된다', () => {
  renderWithStore(<CommonAlert />, {
    commonAlert: { isShow: true, messageHTML: '<p>테스트</p>' },
  });

  expect(screen.getByText('테스트')).toBeInTheDocument();
});
```

## MSW 서버 설정 (Node 환경)

```tsx
// test-utils/msw-server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);

// jest.setup.ts에 추가
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### 테스트별 핸들러 오버라이드
```tsx
it('에러 응답을 처리한다', async () => {
  server.use(
    http.get('/api/data', () => {
      return new HttpResponse(null, { status: 500 });
    })
  );

  // 에러 케이스 테스트
});
```

## 테스트 커버리지 기준

| 타입 | 목표 | 우선순위 |
|------|------|---------|
| 유틸 함수 | 90%+ | 높음 |
| 커스텀 훅 | 80%+ | 높음 |
| Redux Slice | 90%+ | 중간 |
| 컴포넌트 (로직) | 70%+ | 중간 |
| 컴포넌트 (표시만) | 50%+ | 낮음 |
| API 서비스 | 70%+ | 중간 |

## 디버깅 팁

```tsx
// 현재 DOM 출력
screen.debug();

// 특정 요소만 출력
screen.debug(screen.getByRole('button'));

// 접근 가능한 역할 확인
import { logRoles } from '@testing-library/react';
const { container } = render(<Component />);
logRoles(container);

// 쿼리 제안 받기
screen.logTestingPlaygroundURL(); // Testing Playground URL 생성
```
