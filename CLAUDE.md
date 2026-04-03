# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

Claude Code 특화 커스텀 스킬(Skills)을 개발하는 프로젝트. 각 스킬은 `.claude/skills/<skill-name>/SKILL.md` 형태로 구성된다.

## 스킬 파일 구조

```
.claude/skills/<skill-name>/
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
