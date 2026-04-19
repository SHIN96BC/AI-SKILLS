---
name: showcase-page
description: 화려한 CSS 애니메이션이 포함된 소개/쇼케이스/랜딩 페이지 제작. 스크롤 연동 애니메이션, 시차 효과, 등장 애니메이션 등. 외부 라이브러리 없이 CSS만 사용.
argument-hint: [페이지 주제 또는 요구사항]
allowed-tools: Read Grep Glob Write Edit Bash
effort: high
---

# 쇼케이스 페이지 스킬

소개 사이트, 포트폴리오, 랜딩 페이지 등 시각적으로 화려한 페이지를 구현한다.
**CSS 애니메이션만 사용** (외부 애니메이션 라이브러리 없음).
기술 스택과 코드 규칙은 프로젝트 구조에 따라 `/nextjs-cbd` 또는 `/nextjs-fsd` 스킬을 따르며, 애니메이션 레이어를 추가한다.
기존 프로젝트가 있으면 그 구조에 맞춰 파일을 배치한다.

## 기술 스택

- Next.js 14+ App Router + TypeScript
- Tailwind CSS v4 Mobile First (`@import "tailwindcss"`, `@theme inline`)
- **CSS @keyframes / transition / animation** (순수 CSS만)
- **Intersection Observer API** (스크롤 트리거)
- 외부 애니메이션 라이브러리 사용 금지

## 쇼케이스 페이지 구성 패턴

### 표준 섹션 구성
1. **Hero** — 전체화면, 강렬한 첫 인상, 핵심 메시지
2. **Features/Highlights** — 핵심 가치 3~4개, 아이콘+텍스트
3. **Detail/Story** — 스크롤 연동 스토리텔링, 이미지+텍스트 교차
4. **Social Proof** — 실적 숫자, 고객 후기, 파트너 로고
5. **CTA** — 명확한 행동 유도, 배경 강조

### Hero 섹션 패턴
```tsx
// CBD: components/showcase/HeroSection.tsx
// FSD: src/fsd/widgets/showcase-hero/ui/ShowcaseHero.tsx
'use client';

export default function HeroSection() {
  return (
    <section className="relative h-screen flex items-center justify-center overflow-hidden">
      {/* 배경 */}
      <div className="absolute inset-0 bg-gradient-to-br from-gray-900 via-gray-800 to-black" />

      {/* 콘텐츠 */}
      <div className="relative z-10 text-center text-white px-6">
        <h1 className="text-4xl md:text-5xl lg:text-7xl font-bold animate-fade-in-up">
          메인 헤드라인
        </h1>
        <p className="mt-6 text-lg md:text-xl text-gray-300 animate-fade-in-up animation-delay-200">
          서브 텍스트
        </p>
        <button className="mt-10 px-8 py-4 bg-white text-gray-900 rounded-full font-semibold hover:bg-gray-100 transition-colors animate-fade-in-up animation-delay-400">
          시작하기
        </button>
      </div>

      {/* 스크롤 유도 */}
      <div className="absolute bottom-8 left-1/2 -translate-x-1/2 animate-bounce">
        <div className="w-6 h-10 border-2 border-white/50 rounded-full flex justify-center pt-2">
          <div className="w-1 h-3 bg-white/50 rounded-full" />
        </div>
      </div>
    </section>
  );
}
```

## CSS 애니메이션 시스템

### globals.css에 애니메이션 정의 (Tailwind v4 프로젝트)

기존 `@import "tailwindcss"` 아래에 추가한다.
`@theme inline`에 커스텀 애니메이션 토큰을 등록하면 `animate-*` 유틸리티로 사용 가능:

```css
@theme inline {
  --animate-fade-in-up: fadeInUp 0.8s ease-out both;
  --animate-fade-in-down: fadeInDown 0.8s ease-out both;
  --animate-fade-in-left: fadeInLeft 0.8s ease-out both;
  --animate-fade-in-right: fadeInRight 0.8s ease-out both;
  --animate-fade-in-scale: fadeInScale 0.8s ease-out both;
  --animate-float: float 3s ease-in-out infinite;
  --animate-gradient: gradientShift 6s ease infinite;
  --animate-reveal: reveal 1s ease-out both;
}
```

```css
/* === 등장 애니메이션 === */
@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(30px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes fadeInDown {
  from {
    opacity: 0;
    transform: translateY(-30px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes fadeInLeft {
  from {
    opacity: 0;
    transform: translateX(-30px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

@keyframes fadeInRight {
  from {
    opacity: 0;
    transform: translateX(30px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

@keyframes fadeInScale {
  from {
    opacity: 0;
    transform: scale(0.9);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

/* === 지속 애니메이션 === */
@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-10px); }
}

@keyframes gradientShift {
  0% { background-position: 0% 50%; }
  50% { background-position: 100% 50%; }
  100% { background-position: 0% 50%; }
}

@keyframes reveal {
  from { clip-path: inset(0 100% 0 0); }
  to { clip-path: inset(0 0 0 0); }
}

/* v4에서 @theme inline에 등록하면 animate-fade-in-up 등으로 사용 가능 */
/* @theme inline에 등록하지 않는 경우 아래 유틸리티 클래스 사용 */

.animate-gradient {
  background-size: 200% 200%;
}

/* === 딜레이 유틸리티 === */
.animation-delay-100 { animation-delay: 100ms; }
.animation-delay-200 { animation-delay: 200ms; }
.animation-delay-300 { animation-delay: 300ms; }
.animation-delay-400 { animation-delay: 400ms; }
.animation-delay-500 { animation-delay: 500ms; }
.animation-delay-600 { animation-delay: 600ms; }
.animation-delay-700 { animation-delay: 700ms; }
.animation-delay-800 { animation-delay: 800ms; }

/* === 접근성 === */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

## 스크롤 트리거 애니메이션

Intersection Observer를 사용하여 요소가 뷰포트에 진입할 때 애니메이션을 트리거한다.

### useScrollAnimation 훅
```tsx
// CBD: hooks/useScrollAnimation.ts
// FSD: src/fsd/shared/scroll-animation/hooks/useScrollAnimation.ts
'use client';
import { useEffect, useRef, useState } from 'react';

interface Options {
  threshold?: number;
  rootMargin?: string;
  triggerOnce?: boolean;
}

export function useScrollAnimation({
  threshold = 0.1,
  rootMargin = '0px 0px -50px 0px',
  triggerOnce = true,
}: Options = {}) {
  const ref = useRef<HTMLDivElement>(null);
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true);
          if (triggerOnce) observer.unobserve(element);
        } else if (!triggerOnce) {
          setIsVisible(false);
        }
      },
      { threshold, rootMargin }
    );

    observer.observe(element);
    return () => observer.disconnect();
  }, [threshold, rootMargin, triggerOnce]);

  return { ref, isVisible };
}
```

### AnimateOnScroll 컴포넌트
```tsx
// CBD: components/ui/AnimateOnScroll.tsx
// FSD: src/fsd/shared/scroll-animation/ui/AnimateOnScroll.tsx
'use client';
import { useScrollAnimation } from '@/hooks/useScrollAnimation';

interface Props {
  children: React.ReactNode;
  animation?: 'fade-in-up' | 'fade-in-down' | 'fade-in-left' | 'fade-in-right' | 'fade-in-scale';
  delay?: number;
  className?: string;
}

export default function AnimateOnScroll({
  children,
  animation = 'fade-in-up',
  delay = 0,
  className = '',
}: Props) {
  const { ref, isVisible } = useScrollAnimation();

  return (
    <div
      ref={ref}
      className={`${className} ${isVisible ? `animate-${animation}` : 'opacity-0'}`}
      style={delay ? { animationDelay: `${delay}ms` } : undefined}
    >
      {children}
    </div>
  );
}
```

### Stagger 패턴 (순차 등장)
```tsx
// 리스트 아이템 순차 등장
<div className="grid grid-cols-1 md:grid-cols-3 gap-8">
  {features.map((feature, i) => (
    <AnimateOnScroll key={i} animation="fade-in-up" delay={i * 150}>
      <Card {...feature} />
    </AnimateOnScroll>
  ))}
</div>
```

## 성능 규칙

1. **GPU 가속 속성만 애니메이션**: `transform`, `opacity`만 사용
   - `width`, `height`, `margin`, `padding`, `top/left` 등은 애니메이션 금지
2. **`will-change` 최소화**: 실제 성능 이슈가 있는 요소에만 적용
3. **모바일 성능 고려**:
   - 복잡한 애니메이션은 모바일에서 간소화 또는 제거 (`md:` 이상에서만 적용)
   - 동시에 애니메이션되는 요소 수 제한
   - 패럴랙스 효과는 모바일에서 비활성화
4. **`prefers-reduced-motion`**: 반드시 존중 (globals.css에 기본 포함)

## 인터랙션 효과 패턴

### 호버 효과
```tsx
{/* 카드 호버 리프트 */}
<div className="transition-all duration-300 hover:-translate-y-2 hover:shadow-xl">

{/* 이미지 줌 */}
<div className="overflow-hidden rounded-xl">
  <img className="transition-transform duration-500 hover:scale-110" />
</div>

{/* 버튼 글로우 */}
<button className="relative overflow-hidden transition-all duration-300 hover:shadow-[0_0_30px_rgba(0,0,0,0.3)]">
```

### 스크롤 진행률 바
```tsx
// CBD: components/ui/ScrollProgress.tsx
// FSD: src/fsd/shared/scroll-progress/ui/ScrollProgress.tsx
'use client';
import { useEffect, useState } from 'react';

export default function ScrollProgress() {
  const [progress, setProgress] = useState(0);

  useEffect(() => {
    const handleScroll = () => {
      const total = document.documentElement.scrollHeight - window.innerHeight;
      setProgress((window.scrollY / total) * 100);
    };
    window.addEventListener('scroll', handleScroll, { passive: true });
    return () => window.removeEventListener('scroll', handleScroll);
  }, []);

  return (
    <div className="fixed top-0 left-0 w-full h-1 z-[100]">
      <div
        className="h-full bg-gradient-to-r from-blue-500 to-purple-500 transition-[width] duration-150"
        style={{ width: `${progress}%` }}
      />
    </div>
  );
}
```

## 상세 CSS 애니메이션 패턴은 [reference.md](reference.md) 참조

사용자 인자: `$ARGUMENTS`로 전달된 주제나 요구사항을 기반으로 쇼케이스 페이지를 구성한다.
