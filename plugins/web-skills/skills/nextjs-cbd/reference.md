# Next.js + Tailwind CSS v4 퍼블리싱 레퍼런스

## Tailwind CSS v4 설정

### postcss.config.mjs
```js
const config = {
  plugins: {
    '@tailwindcss/postcss': {},
  },
};
export default config;
```

### globals.css (v4 방식)
```css
@import "tailwindcss";

@custom-variant dark (&:where(.dark, .dark *));

@theme inline {
  --color-background: var(--bg);
  --color-foreground: var(--fg);
  --color-primary: var(--primary);
  --color-primary-hover: var(--primary-hover);
  /* 필요한 커스텀 컬러 토큰 추가 */
}

:root {
  --bg: #ffffff;
  --fg: #111827;
  --primary: #2563eb;
  --primary-hover: #1d4ed8;
}

:root.dark {
  --bg: #0f172a;
  --fg: #f1f5f9;
  --primary: #3b82f6;
  --primary-hover: #60a5fa;
}
```

**v3 → v4 마이그레이션 핵심:**
- `tailwind.config.ts` → CSS 내 `@theme inline`
- `@tailwind base/components/utilities` → `@import "tailwindcss"`
- `darkMode: 'class'` → `@custom-variant dark (...)`
- `theme.extend.colors` → `@theme inline { --color-*: ... }`
- PostCSS 플러그인: `tailwindcss` → `@tailwindcss/postcss`

## 자주 쓰는 반응형 패턴 (Mobile First)

### 그리드 레이아웃
```tsx
{/* 1열 → 2열 → 4열 */}
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4 md:gap-8">
  {items.map(item => <Card key={item.id} {...item} />)}
</div>
```

### Flex 방향 전환
```tsx
{/* 세로 → 가로 */}
<div className="flex flex-col md:flex-row gap-4 md:gap-8">
  <div className="flex-1">텍스트</div>
  <div className="flex-1">이미지</div>
</div>
```

### 조건부 표시/숨기기
```tsx
{/* PC에서만 표시 */}
<div className="hidden md:block">PC 전용 콘텐츠</div>

{/* 모바일에서만 표시 */}
<div className="block md:hidden">모바일 전용 콘텐츠</div>
```

### 반응형 타이포그래피
```tsx
<h1 className="text-2xl sm:text-3xl md:text-4xl lg:text-5xl xl:text-6xl font-bold leading-tight">
  제목
</h1>
<p className="text-base md:text-lg text-gray-600 leading-relaxed">
  본문 텍스트
</p>
```

### 반응형 여백/간격
```tsx
{/* 섹션 간격 */}
<section className="py-12 md:py-16 lg:py-24">
  <Container>
    {/* 콘텐츠 */}
  </Container>
</section>
```

### 반응형 네비게이션
```tsx
// Header.tsx (Server Component)
export default function Header() {
  return (
    <header className="fixed top-0 w-full bg-white/80 backdrop-blur-sm border-b z-50">
      <Container>
        <div className="flex items-center justify-between h-16">
          <Logo />
          {/* PC 메뉴 */}
          <nav className="hidden md:flex items-center gap-8">
            <NavLink href="/about">소개</NavLink>
            <NavLink href="/services">서비스</NavLink>
            <NavLink href="/contact">문의</NavLink>
            <Button>시작하기</Button>
          </nav>
          {/* 모바일 메뉴 토글 */}
          <MobileMenuToggle className="block md:hidden" />
        </div>
      </Container>
    </header>
  );
}
```

## 공통 UI 컴포넌트 패턴

### Button
```tsx
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'outline';
  size?: 'sm' | 'md' | 'lg';
}

export default function Button({
  variant = 'primary',
  size = 'md',
  className,
  children,
  ...props
}: ButtonProps) {
  const base = 'inline-flex items-center justify-center rounded-lg font-medium transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2';

  const variants = {
    primary: 'bg-primary text-white hover:bg-primary-hover focus:ring-primary',
    secondary: 'bg-secondary text-white hover:bg-secondary-hover focus:ring-secondary',
    outline: 'border-2 border-primary text-primary hover:bg-primary hover:text-white focus:ring-primary',
  };

  const sizes = {
    sm: 'px-3 py-1.5 text-sm',
    md: 'px-5 py-2.5 text-base',
    lg: 'px-8 py-3.5 text-lg',
  };

  return (
    <button className={`${base} ${variants[variant]} ${sizes[size]} ${className ?? ''}`} {...props}>
      {children}
    </button>
  );
}
```

### Card
```tsx
interface CardProps {
  title: string;
  description: string;
  icon?: React.ReactNode;
}

export default function Card({ title, description, icon }: CardProps) {
  return (
    <div className="p-4 md:p-6 rounded-2xl border border-border hover:border-muted-foreground transition-colors">
      {icon && <div className="mb-4 text-foreground">{icon}</div>}
      <h3 className="text-lg md:text-xl font-semibold mb-2">{title}</h3>
      <p className="text-muted-foreground leading-relaxed">{description}</p>
    </div>
  );
}
```

## 8px 그리드 기반 간격

Tailwind 기본 spacing을 8px 배수로 활용:
| Tailwind | px | 용도 |
|----------|----|------|
| `gap-1`, `p-1` | 4px | 아이콘-텍스트 간격 |
| `gap-2`, `p-2` | 8px | 요소 내부 최소 간격 |
| `gap-3`, `p-3` | 12px | 소형 카드 내부 |
| `gap-4`, `p-4` | 16px | 일반 내부 여백 |
| `gap-6`, `p-6` | 24px | 카드, 섹션 내부 |
| `gap-8` | 32px | 요소 간 간격 |
| `gap-12` | 48px | 섹션 내부 블록 간 |
| `gap-16` | 64px | 섹션 간 간격 (모바일) |
| `gap-20` | 80px | 섹션 간 간격 (태블릿) |
| `gap-24` | 96px | 섹션 간 간격 (PC) |

## Next.js App Router 패턴

### 루트 레이아웃
```tsx
// app/layout.tsx
import type { Metadata } from 'next';
import { Inter } from 'next/font/google';
import './globals.css';
import Header from '@/components/layout/Header';
import Footer from '@/components/layout/Footer';

const inter = Inter({ subsets: ['latin'] });

export const metadata: Metadata = {
  title: { default: '사이트명', template: '%s | 사이트명' },
  description: '사이트 설명',
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ko" className={inter.className}>
      <body className="min-h-screen flex flex-col">
        <Header />
        <main className="flex-1">{children}</main>
        <Footer />
      </body>
    </html>
  );
}
```

### 페이지 컴포넌트
```tsx
// app/page.tsx
import type { Metadata } from 'next';
import HeroSection from '@/components/home/HeroSection';
import FeatureSection from '@/components/home/FeatureSection';

export const metadata: Metadata = {
  title: '홈',
  description: '홈 페이지 설명',
};

export default function HomePage() {
  return (
    <>
      <HeroSection />
      <FeatureSection />
    </>
  );
}
```

### 경로 별칭
`tsconfig.json`에서 `@/` 경로 별칭 사용:
```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./*"]
    }
  }
}
```

## 다크모드 패턴 (v4)

```tsx
{/* 라이트/다크 대응 */}
<div className="bg-background text-foreground">
  <div className="bg-card text-card-foreground border border-border">
    <h2 className="text-foreground">제목</h2>
    <p className="text-muted-foreground">설명</p>
  </div>
</div>

{/* dark: 접두사로 직접 지정 */}
<div className="bg-white dark:bg-gray-900 text-gray-900 dark:text-gray-100">
```
