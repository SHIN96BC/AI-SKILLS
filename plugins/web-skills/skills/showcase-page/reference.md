# CSS 애니메이션 패턴 레퍼런스

## 1. 텍스트 애니메이션

### 글자 단위 순차 등장
```tsx
// components/ui/TextReveal.tsx
'use client';
import { useScrollAnimation } from '@/hooks/useScrollAnimation';

interface Props {
  text: string;
  className?: string;
  charDelay?: number; // 글자 간 딜레이 (ms)
}

export default function TextReveal({ text, className = '', charDelay = 40 }: Props) {
  const { ref, isVisible } = useScrollAnimation();

  return (
    <span ref={ref} className={className} aria-label={text}>
      {text.split('').map((char, i) => (
        <span
          key={i}
          className={`inline-block transition-all duration-500 ${
            isVisible ? 'opacity-100 translate-y-0' : 'opacity-0 translate-y-4'
          }`}
          style={{ transitionDelay: `${i * charDelay}ms` }}
          aria-hidden="true"
        >
          {char === ' ' ? '\u00A0' : char}
        </span>
      ))}
    </span>
  );
}
```

### 단어 단위 등장
```tsx
// components/ui/WordReveal.tsx
'use client';
import { useScrollAnimation } from '@/hooks/useScrollAnimation';

interface Props {
  text: string;
  className?: string;
  wordDelay?: number;
}

export default function WordReveal({ text, className = '', wordDelay = 100 }: Props) {
  const { ref, isVisible } = useScrollAnimation();

  return (
    <span ref={ref} className={className}>
      {text.split(' ').map((word, i) => (
        <span
          key={i}
          className={`inline-block mr-[0.3em] transition-all duration-600 ${
            isVisible ? 'opacity-100 translate-y-0' : 'opacity-0 translate-y-6'
          }`}
          style={{ transitionDelay: `${i * wordDelay}ms` }}
        >
          {word}
        </span>
      ))}
    </span>
  );
}
```

### 타이핑 효과
```css
@keyframes typing {
  from { width: 0; }
  to { width: 100%; }
}

@keyframes blinkCaret {
  50% { border-color: transparent; }
}

.animate-typing {
  overflow: hidden;
  white-space: nowrap;
  border-right: 3px solid;
  width: 0;
  animation:
    typing 2s steps(30) forwards,
    blinkCaret 0.75s step-end infinite;
}
```

## 2. 배경 효과

### 그라데이션 애니메이션 배경
```tsx
<div className="bg-gradient-to-br from-purple-600 via-blue-500 to-cyan-400 bg-[length:200%_200%] animate-gradient">
  {/* 콘텐츠 */}
</div>
```

### 메쉬 그라데이션 (정적 but 화려)
```css
.mesh-gradient {
  background:
    radial-gradient(at 40% 20%, rgba(99, 102, 241, 0.3) 0px, transparent 50%),
    radial-gradient(at 80% 0%, rgba(236, 72, 153, 0.3) 0px, transparent 50%),
    radial-gradient(at 0% 50%, rgba(34, 211, 238, 0.3) 0px, transparent 50%),
    radial-gradient(at 80% 50%, rgba(168, 85, 247, 0.2) 0px, transparent 50%),
    radial-gradient(at 0% 100%, rgba(59, 130, 246, 0.3) 0px, transparent 50%);
}
```

### 파티클/도트 패턴 배경
```css
.dot-pattern {
  background-image: radial-gradient(circle, rgba(0, 0, 0, 0.1) 1px, transparent 1px);
  background-size: 24px 24px;
}
```

## 3. 섹션 전환 효과

### 이미지-텍스트 교차 레이아웃 (지그재그)
```tsx
{sections.map((section, i) => (
  <div
    key={i}
    className={`flex flex-col md:flex-row items-center gap-8 md:gap-16 ${
      i % 2 === 1 ? 'md:flex-row-reverse' : ''
    }`}
  >
    <AnimateOnScroll animation={i % 2 === 0 ? 'fade-in-left' : 'fade-in-right'} className="flex-1">
      <Image src={section.image} alt={section.title} width={600} height={400} className="rounded-2xl" />
    </AnimateOnScroll>
    <AnimateOnScroll animation={i % 2 === 0 ? 'fade-in-right' : 'fade-in-left'} className="flex-1">
      <h3 className="text-3xl max-md:text-2xl font-bold mb-4">{section.title}</h3>
      <p className="text-gray-600 leading-relaxed">{section.description}</p>
    </AnimateOnScroll>
  </div>
))}
```

### 수치/카운터 애니메이션
```tsx
// components/ui/CountUp.tsx
'use client';
import { useEffect, useState } from 'react';
import { useScrollAnimation } from '@/hooks/useScrollAnimation';

interface Props {
  end: number;
  duration?: number;
  suffix?: string;
  prefix?: string;
}

export default function CountUp({ end, duration = 2000, suffix = '', prefix = '' }: Props) {
  const { ref, isVisible } = useScrollAnimation();
  const [count, setCount] = useState(0);

  useEffect(() => {
    if (!isVisible) return;
    let start = 0;
    const increment = end / (duration / 16);
    const timer = setInterval(() => {
      start += increment;
      if (start >= end) {
        setCount(end);
        clearInterval(timer);
      } else {
        setCount(Math.floor(start));
      }
    }, 16);
    return () => clearInterval(timer);
  }, [isVisible, end, duration]);

  return (
    <span ref={ref}>
      {prefix}{count.toLocaleString()}{suffix}
    </span>
  );
}
```

## 4. 카드/요소 효과

### 3D 틸트 (CSS perspective)
```css
.card-tilt {
  transition: transform 0.3s ease;
  transform-style: preserve-3d;
  perspective: 1000px;
}
.card-tilt:hover {
  transform: rotateX(-5deg) rotateY(5deg) translateZ(10px);
}
```

### 글래스모피즘 카드
```tsx
<div className="bg-white/10 backdrop-blur-lg border border-white/20 rounded-2xl p-8 shadow-xl">
  {/* 콘텐츠 */}
</div>
```

### 보더 그라데이션
```css
.gradient-border {
  position: relative;
  border-radius: 16px;
  padding: 1px;
  background: linear-gradient(135deg, #667eea, #764ba2);
}
.gradient-border-inner {
  background: white;
  border-radius: 15px;
  padding: 24px;
}
```

## 5. 스크롤 연동 효과

### CSS scroll-driven animations (모던 브라우저)
```css
/* 스크롤에 따라 요소 크기 변환 — 지원하는 브라우저에서만 동작 */
@supports (animation-timeline: scroll()) {
  .scroll-scale {
    animation: scaleUp linear;
    animation-timeline: scroll();
    animation-range: 0% 50%;
  }

  @keyframes scaleUp {
    from { transform: scale(0.8); opacity: 0.5; }
    to { transform: scale(1); opacity: 1; }
  }
}
```

### 패럴랙스 (JS 기반, 모바일 비활성화)
```tsx
// hooks/useParallax.ts
'use client';
import { useEffect, useState } from 'react';

export function useParallax(speed: number = 0.5) {
  const [offset, setOffset] = useState(0);
  const [isMobile, setIsMobile] = useState(false);

  useEffect(() => {
    const checkMobile = () => setIsMobile(window.innerWidth < 768);
    checkMobile();
    window.addEventListener('resize', checkMobile);

    const handleScroll = () => {
      if (!isMobile) setOffset(window.scrollY * speed);
    };
    window.addEventListener('scroll', handleScroll, { passive: true });

    return () => {
      window.removeEventListener('resize', checkMobile);
      window.removeEventListener('scroll', handleScroll);
    };
  }, [speed, isMobile]);

  return isMobile ? 0 : offset;
}
```

## 6. 로딩/진입 시퀀스

### 페이지 초기 로딩 애니메이션
```css
@keyframes pageEnter {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.page-enter {
  animation: pageEnter 0.6s ease-out;
}
```

### 로고/브랜드 진입
```css
@keyframes logoReveal {
  0% {
    opacity: 0;
    transform: scale(0.8) rotate(-5deg);
  }
  60% {
    opacity: 1;
    transform: scale(1.05) rotate(0deg);
  }
  100% {
    transform: scale(1) rotate(0deg);
  }
}
```

## 7. 모바일 최적화 가이드

### 모바일에서 비활성화할 효과 (Mobile First)

Mobile First에서는 기본이 모바일이므로, 복잡한 효과를 `md:` 이상에서만 적용:

```tsx
{/* 패럴랙스: PC에서만 적용 */}
<div style={{ transform: `translateY(${offset}px)` }} className="md:will-change-transform">

{/* 3D 틸트: PC에서만 */}
<div className="md:hover:-rotate-x-[5deg] md:hover:rotate-y-[5deg]">

{/* 좌우 슬라이드: 모바일은 단순 페이드, PC에서만 좌우 */}
{/* AnimateOnScroll에서 모바일은 fade-in-up, PC는 fade-in-left/right */}
```

```css
/* CSS로 모바일 간소화 */
@media (max-width: 767px) {
  .parallax { transform: none !important; }
  .card-tilt:hover { transform: none; }
}
```

### 모바일 성능 체크리스트
- [ ] 동시 애니메이션 요소 5개 이하
- [ ] 패럴랙스 효과 모바일 비활성화
- [ ] 3D transform 모바일 비활성화
- [ ] 그라데이션 애니메이션 bg-size 축소
- [ ] will-change 사용 최소화
- [ ] passive 이벤트 리스너 사용
