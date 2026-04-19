---
name: web-design
description: 웹사이트 UX/UI 설계. 페이지 구조, 와이어프레임, 컴포넌트 계층, 반응형 전략을 설계한다. 코드 작성 전 단계에서 사용.
argument-hint: [사이트 유형 또는 요구사항]
allowed-tools: Read Grep Glob Agent
effort: high
---

# 웹 디자인 설계 스킬

사용자의 요구사항을 분석하여 웹사이트의 구조, 레이아웃, UX 흐름, 컴포넌트 구성을 설계한다.
설계 결과물은 `/nextjs-publish` 또는 `/showcase-page` 스킬로 구현할 수 있다.

## 설계 프로세스

### 1단계: 요구사항 분석
- 사이트 목적과 타겟 사용자 파악
- 핵심 기능과 콘텐츠 정리
- 경쟁 사이트나 레퍼런스가 있다면 분석

### 2단계: 정보 구조(IA) 설계
- 페이지 목록과 계층 구조 정의
- 네비게이션 흐름도 작성
- 사용자 여정(User Flow) 주요 경로 정의

### 3단계: 와이어프레임
- PC(1440px) 기준 레이아웃을 ASCII로 설계
- 모바일(< 768px) 레이아웃을 ASCII로 설계
- 각 섹션의 목적과 우선순위 명시

**ASCII 와이어프레임 예시:**
```
PC (1440px)
┌─────────────────────────────────────────┐
│  Logo          Nav1  Nav2  Nav3    CTA  │
├─────────────────────────────────────────┤
│                                         │
│            Hero Section                 │
│         Headline + SubText              │
│            [CTA Button]                 │
│                                         │
├────────────┬────────────┬───────────────┤
│  Feature 1 │  Feature 2 │  Feature 3    │
│  Icon+Text │  Icon+Text │  Icon+Text    │
├────────────┴────────────┴───────────────┤
│              Footer                     │
└─────────────────────────────────────────┘

Mobile (< 768px)
┌──────────────────┐
│ Logo      ☰ Menu │
├──────────────────┤
│                  │
│   Hero Section   │
│  Headline+Sub    │
│   [CTA Button]   │
│                  │
├──────────────────┤
│   Feature 1      │
│   Icon + Text    │
├──────────────────┤
│   Feature 2      │
│   Icon + Text    │
├──────────────────┤
│   Feature 3      │
│   Icon + Text    │
├──────────────────┤
│     Footer       │
└──────────────────┘
```

### 4단계: 컴포넌트 계층 구조
- 재사용 가능한 공통 컴포넌트 식별
- 컴포넌트 트리 구조도 작성
- 각 컴포넌트의 역할과 props 개요
- **구현 스킬에 따라 적합한 구조를 선택하여 산출물 제공**

#### CBD 구조 (`/nextjs-cbd` 용)
```
app/
├── layout.tsx          ← RootLayout (Header + Footer 포함)
├── page.tsx            ← HomePage
components/
├── ui/                 ← 공통 UI 컴포넌트
│   ├── Button.tsx
│   ├── Card.tsx
│   └── Input.tsx
├── layout/             ← 레이아웃 컴포넌트
│   ├── Header.tsx
│   ├── Footer.tsx
│   └── Container.tsx
└── home/               ← 페이지별 컴포넌트
    ├── HeroSection.tsx
    └── FeatureSection.tsx
```

#### FSD 구조 (`/nextjs-fsd` 용)
```
src/app/                     ← Next.js App Router (라우팅만 담당)
├── layout.tsx
├── page.tsx
src/fsd/
├── app/                     ← 최상위: providers, layouts, 설정
│   └── layouts/
├── widgets/                 ← 대형 UI 조합 (features/entities 조합)
│   ├── header/ui/
│   └── home-hero/ui/
├── features/                ← 기능 단위 비즈니스 로직
│   └── [기능명]/ui/
├── entities/                ← 도메인 엔티티
│   └── [엔티티명]/ui/
└── shared/                  ← 재사용 UI, 설정, 유틸
    ├── button/ui/
    ├── card/ui/
    └── config/
```
- FSD 레이어 간 임포트 규칙: shared → entities → features → widgets → pages → app (상위만 하위를 import)

### 5단계: 반응형 전략
- Mobile First 접근 (Tailwind 기본): 모바일 기본 → 큰 화면으로 확장
- 브레이크포인트: 모바일(기본) → `sm:` (640px) → `md:` (768px) → `lg:` (1024px) → `xl:` (1280px)
- 브레이크포인트별 변화 명세:

| 요소 | 모바일 (기본) | 태블릿 (md: 768px~) | PC (lg: 1024px~) |
|------|-------------|---------------------|------------------|
| 네비게이션 | 햄버거 메뉴 | 가로 메뉴 (축소) | 가로 메뉴 |
| 그리드 | 1열 | 2열 | 3~4열 |
| 폰트 크기 | 85% | 90% | 기본 |
| 여백 | 최소 | 중간 | 넉넉 |

### 6단계: UX 가이드라인
- 인터랙션 패턴 (호버, 클릭 피드백, 로딩 상태)
- 접근성 고려사항 (키보드 네비게이션, 스크린리더, 색상 대비)
- 에러 상태와 빈 상태 처리 방안

## 핵심 디자인 원칙

상세 원칙은 [reference.md](reference.md) 참조.

- **3초 룰**: 사용자가 3초 내에 페이지 목적을 파악할 수 있어야 한다
- **최소 클릭**: 핵심 기능까지 최대 3클릭 이내
- **일관성**: 동일 패턴의 인터랙션은 동일하게 동작
- **여백 우선**: 콘텐츠 밀도보다 가독성 우선
- **터치 타겟**: 모바일 최소 44x44px

## 산출물 형태

설계가 완료되면 다음을 제공한다:
1. **페이지별 ASCII 와이어프레임** (PC/MO 각각)
2. **컴포넌트 트리 구조도** (디렉토리 + 컴포넌트 목록)
3. **반응형 동작 명세** (브레이크포인트별 변화 테이블)
4. **UX 인터랙션 명세** (주요 인터랙션 동작 정의)

설계 완료 후 구현 스킬로 넘길 수 있다:
- `/nextjs-cbd` — CBD 패턴 일반 페이지
- `/nextjs-fsd` — FSD 아키텍처 프로젝트
- `/showcase-page` — 화려한 애니메이션 랜딩 페이지

## 사용자 인자 처리

`$ARGUMENTS`로 전달된 사이트 유형이나 요구사항을 분석의 출발점으로 사용한다.
예: `/web-design SaaS 랜딩 페이지` → SaaS 제품 소개에 적합한 구조로 설계 시작
