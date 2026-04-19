---
name: nextjs-cbd
description: Next.js App Router + Tailwind CSS v4 Component-Based Design 퍼블리싱. 컴포넌트 역할별 분류(ui/layout/page). Mobile First 풀 반응형.
argument-hint: [페이지명 또는 요구사항]
allowed-tools: Read Grep Glob Write Edit Bash
effort: high
---

# Next.js CBD (Component-Based Design) 퍼블리싱 스킬

Next.js App Router + Tailwind CSS v4로 반응형 웹사이트를 구현한다.
컴포넌트를 역할별(ui/layout/page)로 분류하는 CBD 패턴을 사용한다.
`/web-design`의 설계 산출물이 있으면 그에 따라, 없으면 직접 구조를 잡고 구현한다.
FSD 아키텍처가 필요한 경우 `/nextjs-fsd` 스킬을 사용할 것.

## 기술 스택

- **Next.js 14+** App Router (`app/` 디렉토리)
- **Tailwind CSS v4** (`@import "tailwindcss"`, `@theme inline`)
- **TypeScript** 기본
- **Server Components** 기본, 인터랙션 필요시만 `'use client'`

## Tailwind CSS v4 설정

### globals.css (v4 방식)
```css
@import "tailwindcss";

/* 다크모드 (class 기반) */
@custom-variant dark (&:where(.dark, .dark *));

/* 커스텀 테마 토큰 */
@theme inline {
  --color-primary: var(--primary);
  --color-secondary: var(--secondary);
  /* 필요한 토큰 추가 */
}

:root {
  --primary: #2563eb;
  --secondary: #6b7280;
}
```

**v4 핵심 차이점:**
- `tailwind.config.ts` 대신 CSS 파일 내 `@theme inline`으로 토큰 정의
- `@tailwind base/components/utilities` 대신 `@import "tailwindcss"` 단일 지시어
- `@custom-variant`로 다크모드 등 커스텀 변형 정의
- PostCSS 플러그인: `@tailwindcss/postcss`

## 구현 프로세스

### 1단계: 프로젝트 구조 확인
- 기존 Next.js 프로젝트가 있는지 확인
- 없으면 필요한 파일만 생성 (전체 프로젝트 초기화는 사용자에게 안내)
- 기존 프로젝트가 있으면 구조와 패턴을 파악하고 따름

### 2단계: 디렉토리 구조
```
app/
├── layout.tsx              ← 루트 레이아웃 (HTML, 폰트, 공통 구조)
├── page.tsx                ← 홈페이지
├── globals.css             ← @import "tailwindcss" + @theme inline
├── [페이지명]/
│   └── page.tsx
components/
├── ui/                     ← 범용 UI 컴포넌트
│   ├── Button.tsx
│   ├── Card.tsx
│   ├── Input.tsx
│   └── Modal.tsx
├── layout/                 ← 레이아웃 컴포넌트
│   ├── Header.tsx
│   ├── Footer.tsx
│   ├── Container.tsx
│   └── MobileMenu.tsx
└── [페이지명]/             ← 페이지 전용 컴포넌트
    ├── HeroSection.tsx
    └── FeatureSection.tsx
```

### 3단계: 코드 작성

**레이아웃 구현 순서:**
1. `app/layout.tsx` — 루트 레이아웃, 폰트, 메타데이터
2. `components/layout/Header.tsx` — 네비게이션
3. `components/layout/Footer.tsx` — 푸터
4. `components/layout/Container.tsx` — 콘텐츠 래퍼

**페이지 구현 순서:**
1. 페이지 컴포넌트 (`app/[페이지]/page.tsx`)
2. 섹션 컴포넌트 (`components/[페이지]/`)
3. 필요한 공통 UI 컴포넌트 (`components/ui/`)

## Mobile First 반응형 규칙

Tailwind CSS 기본 브레이크포인트는 Mobile First (min-width 기반):

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
{/* 모바일: 1열 → 태블릿: 2열 → PC: 3열 */}
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 md:gap-8">

{/* 모바일: 세로 배치 → PC: 가로 배치 */}
<div className="flex flex-col md:flex-row items-center gap-6">

{/* 모바일: 작은 텍스트 → PC: 큰 텍스트 */}
<h1 className="text-3xl md:text-4xl lg:text-5xl font-bold">
```

## 컴포넌트 작성 규칙

### 파일 크기
- 한 파일 200줄 이내, 초과 시 하위 컴포넌트로 분리

### 타입 정의
```tsx
interface SectionProps {
  title: string;
  description: string;
  children?: React.ReactNode;
}

export default function Section({ title, description, children }: SectionProps) {
  // ...
}
```

### Server vs Client Component
- **기본은 Server Component** (별도 지시어 없음)
- `'use client'`는 다음 경우에만 사용:
  - `useState`, `useEffect` 등 React 훅 사용
  - 클릭/입력 등 이벤트 핸들러 필요
  - 브라우저 API 접근 필요
- 클라이언트 컴포넌트는 최소 범위로 분리

### 네비게이션 (Header)
- PC: 가로 메뉴 + CTA 버튼
- 모바일 (< 768px): 햄버거 아이콘 → 모바일 메뉴 (오버레이 또는 드로어)
- 모바일 메뉴는 별도 `MobileMenu.tsx` 클라이언트 컴포넌트로 분리

### Container 패턴
```tsx
export default function Container({ children }: { children: React.ReactNode }) {
  return (
    <div className="w-full max-w-[1200px] mx-auto px-4 md:px-6">
      {children}
    </div>
  );
}
```

## SEO & 메타데이터

Next.js metadata API 사용:
```tsx
// app/layout.tsx 또는 app/[page]/page.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: '사이트 제목',
  description: '사이트 설명',
  openGraph: {
    title: '사이트 제목',
    description: '사이트 설명',
    type: 'website',
  },
};
```

## 이미지 & 폰트

```tsx
// 이미지: next/image 사용
import Image from 'next/image';
<Image src="/hero.jpg" alt="설명" width={1200} height={600} priority />

// 폰트: next/font 사용
import { Inter } from 'next/font/google';
const inter = Inter({ subsets: ['latin'] });
```

## 시맨틱 HTML & 접근성

- `<header>`, `<nav>`, `<main>`, `<section>`, `<article>`, `<footer>` 적절히 사용
- 모든 `<img>`에 `alt` 속성
- 버튼과 링크에 명확한 텍스트 또는 `aria-label`
- 폼 요소에 `<label>` 연결
- 키보드 포커스 순서가 논리적으로 흐르도록 구성

## 상세 패턴은 [reference.md](reference.md) 참조

사용자 인자: `$ARGUMENTS`로 전달된 페이지명이나 요구사항을 기반으로 구현을 시작한다.
