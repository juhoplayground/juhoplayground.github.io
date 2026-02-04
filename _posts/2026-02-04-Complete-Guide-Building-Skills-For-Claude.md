---
layout: post
title: Claude Skills 구축 완전 가이드 - 재사용 가능한 워크플로 자산 만들기
author: 'Juho'
date: 2026-02-04 00:00:00 +0900
categories: [Claude]
tags: [Claude Code, Anthropic, Agent]
pin: True
toc: True
---

<style>
  th{
    font-weight: bold;
    text-align: center;
    background-color: white;
  }
  td{
    background-color: white;
  }
</style>

## 목차
1. [개요](#개요)
2. [Skill이란 무엇인가](#skill이란-무엇인가)
3. [SKILL.md 구조](#skillmd-구조)
4. [Skill 저장 위치](#skill-저장-위치)
5. [Frontmatter 필드 참조](#frontmatter-필드-참조)
6. [호출 제어](#호출-제어)
7. [인자 전달](#인자-전달)
8. [서포팅 파일 구성](#서포팅-파일-구성)
9. [Progressive Disclosure 설계 원칙](#progressive-disclosure-설계-원칙)
10. [고급 패턴](#고급-패턴)
11. [Skill과 MCP의 관계](#skill과-mcp의-관계)
12. [배포 방법](#배포-방법)
13. [Reference](#reference)

## 개요

Anthropic이 Claude Skills 구축을 위한 공식 가이드를 발표했다.
Skills는 반복되는 작업 흐름을 한 번 정의하여 지속적으로 재사용할 수 있도록 설계된 워크플로 지식 패키지이다.
단순한 프롬프트를 넘어 스크립트, 템플릿, 참조 문서를 포함하는 재사용 가능한 자산으로 Claude의 기능을 확장할 수 있다.

이 가이드는 개발자, MCP 커넥터 빌더, 파워 유저, 팀 리더를 대상으로 한다.
상위 2~3개의 자동화할 워크플로가 정의되어 있다면, 첫 번째 Skill을 15~30분 내에 구축하고 테스트할 수 있다.

## Skill이란 무엇인가

Skill은 Claude가 특정 워크플로를 일관되게 따르도록 가르치는 메커니즘이다.
문서 생성, 연구 프로세스, 조직 표준화 등 반복 가능한 작업을 자동화할 수 있다.

Skill에는 두 가지 유형의 콘텐츠가 있다.
- Reference 콘텐츠 : Claude가 현재 작업에 적용하는 지식으로, 컨벤션, 패턴, 스타일 가이드, 도메인 지식 등이 해당한다
- Task 콘텐츠 : 배포, 커밋, 코드 생성 등 특정 작업을 위한 단계별 지침이다

주요 활용 카테고리는 세 가지로 나뉜다.
- 문서 및 자산 생성 : 스타일 가이드를 내장하여 일관된 산출물 생성
- 워크플로 자동화 : 다단계 프로세스를 하나의 명령으로 실행
- MCP 도구 강화 : 복합 서비스를 조합하여 활용

## SKILL.md 구조

모든 Skill은 SKILL.md 파일을 필수로 가진다.
SKILL.md는 두 부분으로 구성된다.

첫 번째는 YAML frontmatter로, `---` 마커 사이에 위치하며 Claude에게 Skill을 언제 사용해야 하는지 알려준다.
`name` 필드는 `/슬래시-명령어`가 되고, `description`은 Claude가 자동 로드 여부를 판단하는 데 사용된다.

두 번째는 Markdown 콘텐츠로, Skill이 호출될 때 Claude가 따라야 하는 지침을 담고 있다.

기본 예시는 다음과 같다.

```yaml
---
name: explain-code
description: Explains code with visual diagrams and analogies. Use when explaining how code works, teaching about a codebase, or when the user asks "how does this work?"
---

When explaining code, always include:

1. **Start with an analogy**: Compare the code to something from everyday life
2. **Draw a diagram**: Use ASCII art to show the flow, structure, or relationships
3. **Walk through the code**: Explain step-by-step what happens
4. **Highlight a gotcha**: What's a common mistake or misconception?
```

기존의 `.claude/commands/` 파일도 계속 동작하지만, Skill이 더 많은 기능을 제공한다.
같은 이름의 Skill과 Command가 존재하면 Skill이 우선한다.

## Skill 저장 위치

Skill을 저장하는 위치에 따라 사용 범위가 결정된다.

| 위치 | 경로 | 적용 범위 |
|------|------|----------|
| Enterprise | Managed Settings 참조 | 조직 전체 사용자 |
| Personal | ~/.claude/skills/skill-name/SKILL.md | 모든 프로젝트 |
| Project | .claude/skills/skill-name/SKILL.md | 해당 프로젝트만 |
| Plugin | plugin/skills/skill-name/SKILL.md | 플러그인 활성화 위치 |

같은 이름의 Skill이 여러 레벨에 존재하면 우선순위는 Enterprise > Personal > Project 순이다.
Plugin Skill은 `plugin-name:skill-name` 네임스페이스를 사용하므로 다른 레벨과 충돌하지 않는다.

하위 디렉토리에서 작업할 때 Claude Code는 중첩된 `.claude/skills/` 디렉토리를 자동으로 탐색한다.
예를 들어 `packages/frontend/` 내 파일을 편집하면 `packages/frontend/.claude/skills/`도 검색한다.
이는 모노레포 환경에서 패키지별로 독립적인 Skill을 운용할 수 있게 해준다.

## Frontmatter 필드 참조

SKILL.md 상단의 YAML frontmatter에서 설정할 수 있는 필드는 다음과 같다.

| 필드 | 필수 여부 | 설명 |
|------|----------|------|
| name | 아니오 | Skill의 표시 이름, 생략 시 디렉토리 이름 사용 |
| description | 권장 | Skill의 기능과 사용 시점 설명 (200자 이내) |
| argument-hint | 아니오 | 자동완성 시 표시되는 인자 힌트 |
| disable-model-invocation | 아니오 | true 설정 시 사용자만 호출 가능 |
| user-invocable | 아니오 | false 설정 시 / 메뉴에서 숨김 |
| allowed-tools | 아니오 | Skill 활성화 시 승인 없이 사용 가능한 도구 |
| model | 아니오 | Skill 활성화 시 사용할 모델 |
| context | 아니오 | fork 설정 시 서브에이전트 컨텍스트에서 실행 |
| agent | 아니오 | context: fork 설정 시 사용할 서브에이전트 유형 |
| hooks | 아니오 | Skill 라이프사이클에 범위가 제한된 Hooks |

문자열 치환 변수도 지원된다.

| 변수 | 설명 |
|------|------|
| $ARGUMENTS | 호출 시 전달된 모든 인자 |
| $ARGUMENTS[N] | 0 기반 인덱스로 특정 인자 접근 |
| $N | $ARGUMENTS[N]의 축약형 |
| ${CLAUDE_SESSION_ID} | 현재 세션 ID |

## 호출 제어

기본적으로 사용자와 Claude 모두 Skill을 호출할 수 있다.
두 가지 frontmatter 필드로 이를 제어할 수 있다.

`disable-model-invocation: true`를 설정하면 사용자만 Skill을 호출할 수 있다.
부수 효과가 있거나 타이밍을 직접 제어해야 하는 워크플로에 사용한다.
예를 들어 `/commit`, `/deploy`, `/send-slack-message` 등 Claude가 임의로 실행하면 안 되는 작업에 적합하다.

`user-invocable: false`를 설정하면 Claude만 Skill을 호출할 수 있다.
명령어로 실행할 의미가 없는 배경 지식에 사용한다.
예를 들어 레거시 시스템에 대한 컨텍스트 정보는 Claude가 관련 상황에서 참조하되, 사용자가 직접 호출할 필요는 없다.

| Frontmatter 설정 | 사용자 호출 | Claude 호출 | 컨텍스트 로딩 시점 |
|------------------|-----------|------------|------------------|
| 기본값 | 가능 | 가능 | Description 항상 로드, 호출 시 전체 로드 |
| disable-model-invocation: true | 가능 | 불가 | Description 미로드, 사용자 호출 시 전체 로드 |
| user-invocable: false | 불가 | 가능 | Description 항상 로드, 호출 시 전체 로드 |

## 인자 전달

사용자와 Claude 모두 Skill 호출 시 인자를 전달할 수 있다.
인자는 `$ARGUMENTS` 플레이스홀더를 통해 사용된다.

```yaml
---
name: fix-issue
description: Fix a GitHub issue
disable-model-invocation: true
---

Fix GitHub issue $ARGUMENTS following our coding standards.

1. Read the issue description
2. Understand the requirements
3. Implement the fix
4. Write tests
5. Create a commit
```

`/fix-issue 123`을 실행하면 Claude는 "Fix GitHub issue 123 following our coding standards..."를 받게 된다.
Skill에 `$ARGUMENTS`가 포함되지 않은 상태에서 인자를 전달하면, 콘텐츠 끝에 `ARGUMENTS: <입력값>`이 자동으로 추가된다.

위치 기반 인자 접근도 가능하다.

```yaml
---
name: migrate-component
description: Migrate a component from one framework to another
---

Migrate the $0 component from $1 to $2.
Preserve all existing behavior and tests.
```

`/migrate-component SearchBar React Vue`를 실행하면 `$0`은 SearchBar, `$1`은 React, `$2`는 Vue로 치환된다.

## 서포팅 파일 구성

Skill 디렉토리에 여러 파일을 포함하여 SKILL.md의 부담을 줄일 수 있다.
대규모 참조 문서, API 사양, 예시 컬렉션 등은 Skill이 실행될 때마다 컨텍스트에 로드될 필요가 없다.

표준 디렉토리 구조는 다음과 같다.

```
my-skill/
├── SKILL.md           # 메인 지침 (필수)
├── reference.md       # 상세 API 문서 (필요 시 로드)
├── examples.md        # 사용 예시 (필요 시 로드)
└── scripts/
    └── helper.py      # 유틸리티 스크립트 (실행용)
```

SKILL.md에서 서포팅 파일을 참조하여 Claude가 각 파일의 내용과 로드 시점을 알 수 있게 해야 한다.
SKILL.md는 500줄 이하로 유지하고, 상세한 참조 자료는 별도 파일로 분리하는 것이 권장된다.

## Progressive Disclosure 설계 원칙

Progressive Disclosure는 Skills의 핵심 설계 원칙이다.
정보를 3단계로 나누어 로드하는 방식으로, 기능 확장과 컨텍스트 비용 사이의 모순을 해결한다.

첫 번째 단계는 메타데이터 로딩이다.
시작 시 모든 설치된 Skill의 이름과 설명을 시스템 프롬프트에 사전 로드한다.
약 100토큰 수준으로, Claude가 각 Skill의 사용 시점을 판단할 수 있는 최소한의 정보만 제공한다.

두 번째 단계는 전체 지침 로딩이다.
Claude가 해당 Skill이 현재 작업에 관련이 있다고 판단하면 SKILL.md 전체를 컨텍스트에 로드한다.
5,000토큰 이하로 유지하는 것이 권장된다.

세 번째 단계는 번들 리소스 로딩이다.
필요한 경우에만 서포팅 파일(참조 문서, 스크립트 등)을 추가로 로드한다.

이 설계를 통해 여러 Skill이 컨텍스트 윈도우를 과도하게 소비하지 않으면서 동시에 사용 가능한 상태를 유지할 수 있다.

## 고급 패턴

### 동적 컨텍스트 주입

`` !`command` `` 구문을 사용하면 Skill 콘텐츠가 Claude에게 전달되기 전에 셸 명령을 실행할 수 있다.
명령 출력이 플레이스홀더를 대체하여 Claude는 실제 데이터를 받게 된다.

```yaml
---
name: pr-summary
description: Summarize changes in a pull request
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## Pull request context
- PR diff: !`gh pr diff`
- PR comments: !`gh pr view --comments`
- Changed files: !`gh pr diff --name-only`

## Your task
Summarize this pull request...
```

각 `` !`command` ``가 즉시 실행되고 출력으로 대체된 후, Claude는 완성된 프롬프트만 받는다.
이것은 전처리 과정이며 Claude가 실행하는 것이 아니다.

### 서브에이전트에서 Skill 실행

frontmatter에 `context: fork`를 추가하면 Skill이 격리된 환경에서 실행된다.
Skill 콘텐츠가 서브에이전트를 구동하는 프롬프트가 되며, 대화 기록에 접근하지 않는다.

```yaml
---
name: deep-research
description: Research a topic thoroughly
context: fork
agent: Explore
---

Research $ARGUMENTS thoroughly:

1. Find relevant files using Glob and Grep
2. Read and analyze the code
3. Summarize findings with specific file references
```

실행 흐름은 다음과 같다.
1. 새로운 격리 컨텍스트 생성
2. 서브에이전트가 Skill 콘텐츠를 프롬프트로 수신
3. `agent` 필드가 실행 환경 결정 (모델, 도구, 권한)
4. 결과가 요약되어 메인 대화로 반환

`agent` 필드에는 내장 에이전트(Explore, Plan, general-purpose)나 `.claude/agents/`의 커스텀 서브에이전트를 지정할 수 있다.

### 시각적 출력 생성

Skill에 스크립트를 번들하면 단일 프롬프트로는 불가능한 기능을 구현할 수 있다.
대표적인 패턴이 인터랙티브 HTML 파일 생성이다.
코드베이스 탐색기, 의존성 그래프, 테스트 커버리지 리포트, API 문서, 데이터베이스 스키마 시각화 등에 활용할 수 있다.

번들된 스크립트가 실제 작업을 수행하고 Claude는 오케스트레이션을 담당하는 구조이다.

## Skill과 MCP의 관계

Skill과 MCP는 서로 다른 문제를 해결하는 보완적 관계이다.

MCP는 GitHub, Slack, 데이터베이스, 외부 API 등에서 실시간 데이터를 가져올 때 사용한다.
즉 "무엇이 가능한지(도구 접근)"를 제공한다.

Skill은 워크플로의 일관된 실행, 브랜드 가이드라인, 포맷팅 표준, 전문 절차가 필요할 때 사용한다.
즉 "어떻게 사용해야 하는지(방법론)"를 정의한다.

정리하면 다음과 같다.
- Project : 알려진 정보 제공
- MCP : 최신 정보 제공
- Skills : 방법론 제공
- Subagents : 작업 분담 및 격리 제공

## 배포 방법

Skill은 대상에 따라 다양한 범위로 배포할 수 있다.

프로젝트 Skill은 `.claude/skills/`를 버전 관리에 커밋하면 된다.
플러그인 Skill은 플러그인 내에 `skills/` 디렉토리를 생성한다.
조직 전체 배포는 Managed Settings를 통해 수행한다.

외부 공개 시 GitHub 공개 저장소 운영이 권장된다.
사람 중심의 README를 포함하고, 내부용 소규모 Skill과 외부 공개 Skill에 동일한 테스트 기준을 적용할 필요는 없다.
조직 단위 배포 시에는 자동 업데이트가 지원된다.

## Reference

- [The Complete Guide to Building Skills for Claude - Anthropic](https://claude.com/blog/complete-guide-to-building-skills-for-claude)
- [Extend Claude with Skills - Claude Code Docs](https://code.claude.com/docs/en/skills)
- [Agent Skills Overview - Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
- [Equipping Agents for the Real World with Agent Skills - Anthropic Engineering](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
