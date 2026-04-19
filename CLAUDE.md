# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

Claude Code 특화 커스텀 스킬(Skills)을 플러그인으로 배포하는 프로젝트.

## 설치 방법

```bash
# 1. 마켓플레이스 등록
/plugin marketplace add SHIN96BC/AI-SKILLS

# 2. 플러그인 설치
/plugin install web-skills@shin96bc-ai-skills

# 3. 스킬 사용 (플러그인명:스킬명)
/web-skills:web-design [요구사항]
/web-skills:nextjs-fsd [페이지명]
```

## 저장소 구조

```
AI-SKILLS/
├── .claude-plugin/
│   └── marketplace.json        # 마켓플레이스 카탈로그
├── plugins/
│   └── web-skills/             # 플러그인
│       ├── .claude-plugin/
│       │   └── plugin.json     # 플러그인 매니페스트
│       └── skills/             # 스킬 모음
│           ├── web-design/
│           ├── nextjs-cbd/
│           ├── nextjs-fsd/
│           ├── showcase-page/
│           └── jest-test/
└── .claude/skills/             # 로컬 개발용 (이 저장소에서 직접 테스트)
```

## 스킬 파일 구조

```
skills/<skill-name>/
├── SKILL.md           # 스킬 본체 (필수)
├── reference.md       # 상세 레퍼런스 (선택)
├── examples.md        # 사용 예시 (선택)
└── scripts/           # 보조 스크립트 (선택)
```

## SKILL.md 형식

```yaml
---
name: skill-name              # 소문자+하이픈, 최대 64자
description: 스킬 설명         # 자동 트리거 판단에 사용, ~250자 이내
argument-hint: [args]          # 자동완성 힌트 (선택)
disable-model-invocation: false # true면 사용자만 호출 가능
user-invocable: true           # false면 Claude만 사용 가능
allowed-tools: Read Grep Bash  # 허용 도구 목록
model: claude-opus-4-6       # 모델 오버라이드 (선택)
effort: high                   # low/medium/high/max (선택)
context: fork                  # fork면 서브에이전트로 실행 (선택)
agent: general-purpose         # 서브에이전트 타입 (선택)
---

스킬 본문 (마크다운)
```

## 스킬 작성 규칙

- SKILL.md는 500줄 이내로 유지, 상세 내용은 별도 파일로 분리
- description은 핵심 용도를 앞에 배치 (잘림 방지)
- `$ARGUMENTS`로 사용자 인자 참조, `$ARGUMENTS[N]` 또는 `$N`으로 개별 인자 참조
- `${CLAUDE_SKILL_DIR}`로 스킬 디렉토리 경로 참조
- `` !`command` `` 구문으로 전처리 쉘 명령 삽입 가능

## 스킬 스코프 우선순위

Enterprise(`관리자 설정`) > Personal(`~/.claude/skills/`) > Project(`.claude/skills/`)

## 현재 스킬 목록

| 스킬 | 명령어 | 설명 |
|------|--------|------|
| **web-design** | `/web-skills:web-design` | 웹사이트 UX/UI 설계 (와이어프레임, 컴포넌트 계층, 반응형 전략). CBD/FSD 양쪽 산출물 지원 |
| **nextjs-cbd** | `/web-skills:nextjs-cbd` | Component-Based Design 퍼블리싱. 컴포넌트 역할별 분류(ui/layout/page) |
| **nextjs-fsd** | `/web-skills:nextjs-fsd` | Feature-Sliced Design 퍼블리싱. FSD 레이어 규칙, DI 컨테이너, React Query, Redux |
| **showcase-page** | `/web-skills:showcase-page` | CSS 애니메이션 기반 화려한 소개/쇼케이스/랜딩 페이지. CBD/FSD 양쪽 호환 |
| **jest-test** | `/web-skills:jest-test` | Jest + React Testing Library 기반 테스트. 유닛/통합/컴포넌트/훅/API 테스트 |

### 스킬 간 워크플로우

```
/web-skills:web-design → 설계 → /web-skills:nextjs-cbd   (CBD 패턴)
                               → /web-skills:nextjs-fsd   (FSD 아키텍처)
                               → /web-skills:showcase-page (화려한 랜딩)

구현 완료 후 → /web-skills:jest-test (테스트 코드 작성)
```

### 공통 기술 스택

- Next.js 14+ App Router, TypeScript
- Tailwind CSS v4 Mobile First (`@import "tailwindcss"`, `@theme inline`, `sm:`, `md:`, `lg:`)
- CSS 애니메이션 only (외부 라이브러리 없음)
- 반응형: 모바일(기본) → md:(768px) → lg:(1024px) → xl:(1280px)
