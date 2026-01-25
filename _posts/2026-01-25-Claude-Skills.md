---
layout: post
title: Claude Skills - AI 에이전트를 위한 확장 가능한 스킬 시스템
author: 'Juho'
date: 2026-01-25 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Claude, MCP]
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
2. [Agent Skills란](#agent-skills란)
3. [Claude Code Skills](#claude-code-skills)
4. [스킬 생성하기](#스킬-생성하기)
5. [스킬 저장 위치](#스킬-저장-위치)
6. [스킬 설정](#스킬-설정)
7. [고급 패턴](#고급-패턴)
8. [커뮤니티 스킬](#커뮤니티-스킬)
9. [스킬 설치 방법](#스킬-설치-방법)

## 개요

Claude Skills는 Claude의 기능을 확장하는 재사용 가능한 지시사항, 스크립트, 리소스 모음이다.
특정 작업을 반복 가능한 방식으로 수행하도록 Claude를 가르치는 시스템이다.
필요할 때만 로드되어 성능에 영향을 주지 않으면서 수백 개의 스킬을 유지할 수 있다.

## Agent Skills란

Agent Skills는 Anthropic이 발표한 AI 에이전트 확장 시스템이다.
Claude 앱, Claude Code, API 전반에서 사용할 수 있는 표준화된 형식을 따른다.

### 핵심 특징

Agent Skills는 네 가지 핵심 특성을 가진다.

### 조합 가능성 (Composable)

여러 스킬이 함께 작동할 수 있다.
Claude가 작업에 필요한 스킬을 자동으로 식별하여 조합한다.

### 이식성 (Portable)

한 번 만들면 Claude 앱, Claude Code, API 어디서든 동일하게 작동한다.
플랫폼 간 호환성이 보장된다.

### 효율성 (Efficient)

필요할 때만 리소스를 로드한다.
최소한의 정보만 가져와 Claude를 빠르게 유지한다.

### 강력함 (Powerful)

실행 가능한 코드를 포함할 수 있다.
토큰 생성보다 프로그래밍이 더 효과적인 작업에 유용하다.

### 기본 제공 스킬

Claude에서 기본으로 제공되는 스킬은 다음과 같다.

- Excel 스프레드시트 생성 (수식 포함)
- PowerPoint 프레젠테이션 제작
- Word 문서 작성
- 작성 가능한 PDF 생성

## Claude Code Skills

Claude Code에서 스킬은 SKILL.md 파일로 정의된다.
스킬을 만들면 Claude의 도구 목록에 추가되어 관련 작업 시 자동으로 활성화되거나 /skill-name으로 직접 호출할 수 있다.

### 기존 명령어와의 통합

기존의 .claude/commands/ 디렉토리에 있던 커스텀 슬래시 명령어는 스킬로 통합되었다.
.claude/commands/review.md와 .claude/skills/review/SKILL.md 모두 /review로 동일하게 작동한다.
기존 commands 파일도 계속 작동하지만 스킬 형식이 더 많은 기능을 제공한다.

### Agent Skills 표준

Claude Code Skills는 Agent Skills 오픈 표준을 따른다.
이 표준은 여러 AI 도구에서 공통으로 사용된다.
Claude Code는 호출 제어, 서브에이전트 실행, 동적 컨텍스트 주입 등 추가 기능을 제공한다.

## 스킬 생성하기

간단한 스킬을 만들어보자.
코드를 시각적 다이어그램과 비유로 설명하는 스킬이다.

### 스킬 디렉토리 생성

개인 스킬 폴더에 디렉토리를 생성한다.

```bash
mkdir -p ~/.claude/skills/explain-code
```

### SKILL.md 작성

~/.claude/skills/explain-code/SKILL.md 파일을 생성한다.

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

Keep explanations conversational. For complex concepts, use multiple analogies.
```

### 스킬 테스트

두 가지 방법으로 테스트할 수 있다.

Claude가 자동으로 호출하도록 설명과 일치하는 질문을 한다.

```
How does this code work?
```

또는 스킬 이름으로 직접 호출한다.

```
/explain-code src/auth/login.ts
```

## 스킬 저장 위치

스킬을 저장하는 위치에 따라 사용 범위가 달라진다.

| 위치 | 경로 | 적용 범위 |
|------|------|-----------|
| Enterprise | 관리 설정 참조 | 조직 전체 사용자 |
| Personal | ~/.claude/skills/skill-name/SKILL.md | 모든 프로젝트 |
| Project | .claude/skills/skill-name/SKILL.md | 해당 프로젝트만 |
| Plugin | plugin/skills/skill-name/SKILL.md | 플러그인 활성화된 곳 |

동일한 이름의 스킬이 여러 위치에 있으면 우선순위가 적용된다.
enterprise > personal > project 순서로 우선된다.
플러그인 스킬은 plugin-name:skill-name 네임스페이스를 사용하여 충돌하지 않는다.

### 중첩 디렉토리 자동 탐색

하위 디렉토리에서 작업할 때 Claude Code는 중첩된 .claude/skills/ 디렉토리도 자동으로 탐색한다.
예를 들어 packages/frontend/에서 파일을 편집하면 packages/frontend/.claude/skills/도 확인한다.
모노레포 환경에서 패키지별 스킬을 관리하는 데 유용하다.

### 스킬 디렉토리 구조

각 스킬은 SKILL.md를 진입점으로 하는 디렉토리다.

```
my-skill/
├── SKILL.md           # 주요 지시사항 (필수)
├── template.md        # Claude가 채울 템플릿
├── examples/
│   └── sample.md      # 예상 형식을 보여주는 예시
└── scripts/
    └── validate.sh    # Claude가 실행할 스크립트
```

## 스킬 설정

스킬은 SKILL.md 상단의 YAML 프론트매터와 그 아래 마크다운 콘텐츠로 구성된다.

### 스킬 콘텐츠 유형

스킬 파일은 두 가지 유형으로 나눌 수 있다.

### 참조 콘텐츠

현재 작업에 적용할 지식을 추가한다.
컨벤션, 패턴, 스타일 가이드, 도메인 지식 등이 해당된다.

```yaml
---
name: api-conventions
description: API design patterns for this codebase
---

When writing API endpoints:
- Use RESTful naming conventions
- Return consistent error formats
- Include request validation
```

### 작업 콘텐츠

특정 액션을 위한 단계별 지시사항이다.
배포, 커밋, 코드 생성 같은 작업에 적합하다.
disable-model-invocation: true를 추가하면 Claude가 자동으로 트리거하지 않는다.

```yaml
---
name: deploy
description: Deploy the application to production
context: fork
disable-model-invocation: true
---

Deploy the application:
1. Run the test suite
2. Build the application
3. Push to the deployment target
```

### 프론트매터 필드

| 필드 | 필수 | 설명 |
|------|------|------|
| name | 아니오 | 스킬 표시 이름, 생략 시 디렉토리 이름 사용 |
| description | 권장 | 스킬이 하는 일과 사용 시점, Claude가 자동 활성화 판단에 사용 |
| argument-hint | 아니오 | 자동완성 시 표시되는 인수 힌트 |
| disable-model-invocation | 아니오 | true면 수동 호출만 가능 |
| user-invocable | 아니오 | false면 / 메뉴에서 숨김 |
| allowed-tools | 아니오 | 스킬 활성화 시 허용할 도구 목록 |
| model | 아니오 | 스킬 실행 시 사용할 모델 |
| context | 아니오 | fork로 설정하면 분기된 서브에이전트에서 실행 |
| agent | 아니오 | context: fork일 때 사용할 서브에이전트 유형 |

### 문자열 치환

스킬에서 동적 값을 위한 문자열 치환을 지원한다.

| 변수 | 설명 |
|------|------|
| $ARGUMENTS | 스킬 호출 시 전달된 모든 인수 |
| ${CLAUDE_SESSION_ID} | 현재 세션 ID |

예시는 다음과 같다.

```yaml
---
name: session-logger
description: Log activity for this session
---

Log the following to logs/${CLAUDE_SESSION_ID}.log:

$ARGUMENTS
```

### 지원 파일 추가

스킬 디렉토리에 여러 파일을 포함할 수 있다.
SKILL.md는 핵심에 집중하고 상세 참조 자료는 필요할 때만 로드된다.

```
my-skill/
├── SKILL.md (필수 - 개요 및 탐색)
├── reference.md (상세 API 문서 - 필요 시 로드)
├── examples.md (사용 예시 - 필요 시 로드)
└── scripts/
    └── helper.py (유틸리티 스크립트 - 실행됨)
```

SKILL.md에서 지원 파일을 참조하여 Claude가 각 파일의 내용과 로드 시점을 알 수 있게 한다.

### 호출 제어

기본적으로 사용자와 Claude 모두 스킬을 호출할 수 있다.
두 가지 프론트매터 필드로 제한할 수 있다.

**disable-model-invocation: true**

사용자만 호출 가능하다.
/commit, /deploy, /send-slack-message처럼 부작용이 있거나 타이밍을 제어해야 하는 워크플로우에 적합하다.

**user-invocable: false**

Claude만 호출 가능하다.
사용자가 직접 호출할 필요 없는 배경 지식에 적합하다.
레거시 시스템 컨텍스트처럼 Claude가 관련 상황에서 알아야 할 정보다.

| 프론트매터 | 사용자 호출 | Claude 호출 | 컨텍스트 로드 시점 |
|------------|-------------|-------------|-------------------|
| 기본값 | 가능 | 가능 | 설명 항상 로드, 호출 시 전체 로드 |
| disable-model-invocation: true | 가능 | 불가 | 설명 미로드, 사용자 호출 시 전체 로드 |
| user-invocable: false | 불가 | 가능 | 설명 항상 로드, 호출 시 전체 로드 |

### 도구 접근 제한

allowed-tools 필드로 스킬 활성화 시 사용 가능한 도구를 제한할 수 있다.

```yaml
---
name: safe-reader
description: Read files without making changes
allowed-tools: Read, Grep, Glob
---
```

이 스킬은 읽기 전용 모드를 생성하여 파일을 탐색하되 수정하지 않는다.

### 인수 전달

사용자와 Claude 모두 스킬 호출 시 인수를 전달할 수 있다.
$ARGUMENTS 플레이스홀더로 인수에 접근한다.

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

/fix-issue 123을 실행하면 Claude는 "Fix GitHub issue 123 following our coding standards..."를 받는다.

## 고급 패턴

### 동적 컨텍스트 주입

!`command` 구문으로 스킬 콘텐츠가 Claude에 전송되기 전에 셸 명령을 실행한다.
명령 출력이 플레이스홀더를 대체하여 Claude는 명령이 아닌 실제 데이터를 받는다.

```yaml
---
name: pr-summary
description: Summarize changes in a pull request
context: fork
agent: Explore
allowed-tools: Bash(gh:*)
---

## Pull request context
- PR diff: !`gh pr diff`
- PR comments: !`gh pr view --comments`
- Changed files: !`gh pr diff --name-only`

## Your task
Summarize this pull request...
```

이 스킬이 실행되면 각 명령이 즉시 실행되고 출력이 스킬 콘텐츠에 삽입된다.
Claude는 최종 렌더링된 프롬프트만 받는다.

### 서브에이전트에서 실행

context: fork를 추가하면 스킬이 격리된 환경에서 실행된다.
스킬 콘텐츠가 서브에이전트를 구동하는 프롬프트가 된다.
대화 기록에 접근할 수 없다.

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

agent 필드는 사용할 서브에이전트 구성을 지정한다.
Explore, Plan, general-purpose 같은 내장 에이전트나 .claude/agents/의 커스텀 서브에이전트를 사용할 수 있다.

### 시각적 출력 생성

스킬은 어떤 언어로든 스크립트를 번들하고 실행할 수 있다.
대화형 HTML 파일을 생성하여 브라우저에서 데이터를 탐색하거나 보고서를 만들 수 있다.

코드베이스 시각화 스킬 예시는 다음과 같다.

```yaml
---
name: codebase-visualizer
description: Generate an interactive collapsible tree visualization of your codebase. Use when exploring a new repo, understanding project structure, or identifying large files.
allowed-tools: Bash(python:*)
---

# Codebase Visualizer

Generate an interactive HTML tree view that shows your project's file structure with collapsible directories.

## Usage

Run the visualization script from your project root:

python ~/.claude/skills/codebase-visualizer/scripts/visualize.py .

This creates codebase-map.html in the current directory and opens it in your default browser.
```

## 커뮤니티 스킬

### Anthropic 공식 스킬 저장소

Anthropic은 공식 스킬 저장소를 GitHub에서 운영한다.
51,000개 이상의 스타를 받았으며 다양한 스킬을 제공한다.

### 문서 스킬

- skills/docx - Word 문서 생성 및 편집
- skills/pdf - PDF 조작
- skills/pptx - PowerPoint 프레젠테이션
- skills/xlsx - Excel 스프레드시트

### 예제 스킬

- Creative & Design - 생성 아트, 캔버스 디자인, GIF 생성
- Development - React/Tailwind 웹 아티팩트, MCP 서버, 웹앱 테스트
- Branding & Communication - 브랜드 가이드라인, 내부 커뮤니케이션

### Awesome Claude Skills

VoltAgent에서 관리하는 awesome-claude-skills 저장소는 커뮤니티 스킬의 종합 컬렉션이다.
3,900개 이상의 스타를 받았으며 다양한 도메인의 스킬을 제공한다.

### 보안 도구

Trail of Bits에서 제공하는 정적 분석, 스마트 컨트랙트, 퍼징 도구가 포함되어 있다.

### 클라우드 플랫폼

Cloudflare, AWS, Neon, Supabase 관련 스킬이 있다.

### 개발 워크플로우

Git, GitHub, 테스트, 디버깅 관련 스킬을 제공한다.

### 전문 도메인

과학 연구, 마케팅, UI/UX 패턴 관련 스킬도 포함되어 있다.

### 다양한 AI 도구 지원

이 스킬들은 여러 AI 코딩 어시스턴트에서 사용할 수 있다.

| AI 도구 | 스킬 경로 |
|---------|-----------|
| Claude Code | ~/.claude/skills/ |
| GitHub Copilot | ~/.copilot/skills/ |
| Cursor | ~/.cursor/skills/ |
| Gemini CLI | ~/.gemini/skills/ |
| Windsurf | ~/.codeium/windsurf/skills/ |

## 스킬 설치 방법

### Claude Code에서 설치

플러그인 마켓플레이스로 등록한다.

```bash
/plugin marketplace add anthropics/skills
```

개별 스킬을 설치한다.

```bash
/plugin install document-skills@anthropic-agent-skills
/plugin install example-skills@anthropic-agent-skills
```

### Claude.ai에서 사용

유료 플랜 사용자는 예제 스킬을 사용할 수 있다.
설정 메뉴에서 커스텀 스킬을 업로드할 수 있다.

### Claude API에서 사용

Messages API와 /v1/skills 엔드포인트를 통해 사용한다.
API 구현에는 Code Execution Tool 베타가 필요하다.

## Reference

- [Extend Claude with skills - Claude Code Docs](https://code.claude.com/docs/en/skills)
- [Introducing Agent Skills - Claude](https://claude.com/blog/skills)
- [GitHub - anthropics/skills](https://github.com/anthropics/skills)
- [GitHub - VoltAgent/awesome-claude-skills](https://github.com/VoltAgent/awesome-claude-skills)
- [Agent Skills Standard](https://agentskills.io)
