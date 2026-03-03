---
layout: post
title: "Claude Code 메모리 관리 - Auto Memory와 CLAUDE.md 완벽 가이드"
author: 'Juho'
date: 2026-03-03 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, Context, Management]
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
2. [메모리 유형 개요](#메모리-유형-개요)
3. [Auto Memory](#auto-memory)
   - [저장되는 내용](#저장되는-내용)
   - [저장 위치와 구조](#저장-위치와-구조)
   - [작동 방식](#작동-방식)
   - [관리 방법](#관리-방법)
4. [CLAUDE.md 파일 활용](#claudemd-파일-활용)
   - [Import 구문](#import-구문)
   - [모듈형 규칙](#모듈형-규칙)
   - [경로별 규칙](#경로별-규칙)
5. [조직 수준 메모리 관리](#조직-수준-메모리-관리)
6. [모범 사례](#모범-사례)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

Claude Code에 세션 간 학습 내용을 유지하는 Auto Memory 기능이 추가되었다.
이 기능을 통해 Claude는 프로젝트 컨텍스트, 디버깅 경험, 선호하는 접근 방식 등을 자동으로 저장하고 다음 세션에서 재활용할 수 있다.
Claude Code는 크게 두 가지 종류의 지속 메모리를 제공한다.
Auto Memory는 Claude가 작업하면서 자동으로 유용한 컨텍스트를 저장하는 기능이고, CLAUDE.md 파일은 사용자가 직접 작성하고 관리하는 지침 파일이다.
두 메모리 모두 매 세션 시작 시 Claude의 컨텍스트에 로드된다.

## 메모리 유형 개요

Claude Code는 계층 구조로 여러 메모리 위치를 제공하며, 각각 다른 용도로 사용된다.

| 메모리 유형 | 위치 | 용도 |
|------------|------|------|
| Managed Policy | OS별 시스템 경로 | 조직 전체 지침 (IT/DevOps 관리) |
| Project Memory | ./CLAUDE.md 또는 ./.claude/CLAUDE.md | 팀 공유 프로젝트 지침 |
| Project Rules | ./.claude/rules/*.md | 모듈형 주제별 프로젝트 지침 |
| User Memory | ~/.claude/CLAUDE.md | 모든 프로젝트에 적용되는 개인 설정 |
| Project Memory (Local) | ./CLAUDE.local.md | 개인용 프로젝트별 설정 |
| Auto Memory | ~/.claude/projects/프로젝트/memory/ | Claude의 자동 메모와 학습 내용 |

| 메모리 유형 | 공유 범위 | 사용 예시 |
|------------|----------|----------|
| Managed Policy | 조직 내 모든 사용자 | 회사 코딩 표준, 보안 정책 |
| Project Memory | 소스 컨트롤을 통한 팀원 | 프로젝트 아키텍처, 코딩 표준 |
| Project Rules | 소스 컨트롤을 통한 팀원 | 언어별 가이드라인, 테스트 규칙 |
| User Memory | 본인만 (모든 프로젝트) | 코드 스타일 선호, 개인 도구 단축키 |
| Project Memory (Local) | 본인만 (현재 프로젝트) | 샌드박스 URL, 선호하는 테스트 데이터 |
| Auto Memory | 본인만 (프로젝트별) | 프로젝트 패턴, 디버깅 인사이트 |

작업 디렉토리 상위의 CLAUDE.md 파일은 시작 시 전체가 로드되고, 하위 디렉토리의 CLAUDE.md 파일은 해당 디렉토리의 파일을 읽을 때 온디맨드로 로드된다.
더 구체적인 지침이 더 넓은 범위의 지침보다 우선한다.
CLAUDE.local.md 파일은 자동으로 .gitignore에 추가되므로 버전 관리에 포함되지 않아야 하는 개인 설정에 적합하다.

## Auto Memory

Auto Memory는 Claude가 작업하면서 발견한 내용을 기반으로 자동으로 메모를 기록하는 지속 디렉토리이다.
사용자가 작성하는 CLAUDE.md와 달리, Auto Memory는 Claude가 스스로 작성하는 노트이다.
이 기능은 기본적으로 활성화되어 있으며, `/memory` 명령으로 토글할 수 있다.

### 저장되는 내용

Claude는 작업하면서 다음과 같은 정보를 자동으로 저장할 수 있다.
- 프로젝트 패턴: 빌드 명령어, 테스트 규칙, 코드 스타일 선호
- 디버깅 인사이트: 까다로운 문제의 해결책, 일반적인 오류 원인
- 아키텍처 노트: 주요 파일, 모듈 관계, 중요한 추상화
- 사용자 선호: 커뮤니케이션 스타일, 워크플로 습관, 도구 선택

### 저장 위치와 구조

각 프로젝트는 `~/.claude/projects/<project>/memory/` 경로에 고유한 메모리 디렉토리를 가진다.
프로젝트 경로는 git 저장소 루트에서 파생되므로, 같은 저장소 내 모든 하위 디렉토리는 하나의 Auto Memory 디렉토리를 공유한다.
Git worktree는 별도의 메모리 디렉토리를 가진다.

디렉토리 구조는 다음과 같다.

```text
~/.claude/projects/<project>/memory/
├── MEMORY.md          # 간결한 인덱스, 매 세션에 로드
├── debugging.md       # 디버깅 패턴에 대한 상세 노트
├── api-conventions.md # API 설계 결정사항
└── ...                # Claude가 생성하는 기타 주제 파일
```

MEMORY.md는 메모리 디렉토리의 인덱스 역할을 한다.
Claude는 세션 동안 이 디렉토리의 파일을 읽고 쓰며, MEMORY.md를 통해 어디에 무엇이 저장되어 있는지 추적한다.

### 작동 방식

MEMORY.md의 처음 200줄이 매 세션 시작 시 Claude의 시스템 프롬프트에 로드된다.
200줄을 초과하는 내용은 자동으로 로드되지 않으며, Claude는 상세 노트를 별도의 주제 파일로 이동시켜 간결하게 유지하도록 지시받는다.
debugging.md나 patterns.md 같은 주제 파일은 시작 시 로드되지 않고, Claude가 해당 정보가 필요할 때 표준 파일 도구를 사용해 온디맨드로 읽는다.
Claude는 세션 동안 메모리 파일을 읽고 쓰므로, 작업 중에 메모리 업데이트가 발생하는 것을 볼 수 있다.

### 관리 방법

Auto Memory 파일은 일반 마크다운 파일이므로 언제든지 편집할 수 있다.
`/memory` 명령을 사용하면 CLAUDE.md 파일과 함께 Auto Memory 엔트리포인트를 포함한 파일 선택기를 열 수 있다.
Claude에게 특정 내용을 저장하도록 직접 요청할 수도 있다.

```
"pnpm을 사용하고 npm은 사용하지 않는다는 것을 기억해줘"
"API 테스트에 로컬 Redis 인스턴스가 필요하다는 것을 메모리에 저장해줘"
```

설정이나 환경 변수를 통해서도 제어할 수 있다.

모든 프로젝트에서 비활성화하려면 사용자 설정에 추가한다.

```json
// ~/.claude/settings.json
{ "autoMemoryEnabled": false }
```

단일 프로젝트에서 비활성화하려면 프로젝트 설정에 추가한다.

```json
// .claude/settings.json
{ "autoMemoryEnabled": false }
```

환경 변수로 모든 설정을 재정의할 수도 있다.
이 방법은 CI나 관리 환경에서 유용하다.

```bash
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=1  # 강제 비활성화
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=0  # 강제 활성화
```

## CLAUDE.md 파일 활용

### Import 구문

CLAUDE.md 파일은 `@path/to/import` 구문을 사용하여 추가 파일을 임포트할 수 있다.

```text
프로젝트 개요는 @README 를, 사용 가능한 npm 명령어는 @package.json 을 참고하세요.

# 추가 지침
- git 워크플로 @docs/git-instructions.md
```

상대 경로와 절대 경로 모두 사용할 수 있다.
상대 경로는 작업 디렉토리가 아닌, 임포트를 포함하는 파일을 기준으로 해석된다.
여러 git worktree에서 작업하는 경우, CLAUDE.local.md는 하나에만 존재하므로 홈 디렉토리 임포트를 사용하면 모든 worktree에서 동일한 개인 지침을 공유할 수 있다.

```text
# 개인 설정
- @~/.claude/my-project-instructions.md
```

외부 임포트를 처음 만나면 Claude Code는 해당 파일 목록을 보여주는 승인 대화 상자를 표시한다.
이는 프로젝트당 한 번의 결정이며, 거부하면 대화 상자가 다시 나타나지 않고 임포트는 비활성화된 상태로 유지된다.
임포트된 파일은 최대 5단계 깊이까지 재귀적으로 추가 파일을 임포트할 수 있다.

### 모듈형 규칙

더 큰 프로젝트의 경우, `.claude/rules/` 디렉토리를 사용하여 지침을 여러 파일로 구성할 수 있다.

```text
your-project/
├── .claude/
│   ├── CLAUDE.md           # 주요 프로젝트 지침
│   └── rules/
│       ├── code-style.md   # 코드 스타일 가이드라인
│       ├── testing.md      # 테스트 규칙
│       └── security.md     # 보안 요구사항
```

`.claude/rules/`의 모든 .md 파일은 `.claude/CLAUDE.md`와 동일한 우선순위로 프로젝트 메모리로 자동 로드된다.
하위 디렉토리로 구성할 수도 있으며, 모든 .md 파일이 재귀적으로 검색된다.
심볼릭 링크도 지원되어 여러 프로젝트에서 공통 규칙을 공유할 수 있다.

사용자 수준의 규칙은 `~/.claude/rules/`에 생성할 수 있으며, 프로젝트 규칙보다 낮은 우선순위로 로드된다.

### 경로별 규칙

규칙은 YAML 프론트매터의 paths 필드를 사용하여 특정 파일에만 적용되도록 범위를 지정할 수 있다.

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API 개발 규칙

- 모든 API 엔드포인트에 입력 검증을 포함해야 한다
- 표준 오류 응답 형식을 사용해야 한다
- OpenAPI 문서 주석을 포함해야 한다
```

paths 필드가 없는 규칙은 무조건적으로 로드되어 모든 파일에 적용된다.
paths 필드는 표준 glob 패턴을 지원한다.

| 패턴 | 매칭 대상 |
|------|----------|
| **/*.ts | 모든 디렉토리의 TypeScript 파일 |
| src/**/* | src/ 디렉토리 하위 모든 파일 |
| *.md | 프로젝트 루트의 Markdown 파일 |
| src/components/*.tsx | 특정 디렉토리의 React 컴포넌트 |

중괄호 확장도 지원되어 여러 확장자나 디렉토리를 한 번에 매칭할 수 있다.

```markdown
---
paths:
  - "src/**/*.{ts,tsx}"
  - "{src,lib}/**/*.ts"
---
```

## 조직 수준 메모리 관리

조직은 모든 사용자에게 적용되는 중앙 관리 CLAUDE.md 파일을 배포할 수 있다.
Managed Policy 위치에 파일을 생성하고 구성 관리 시스템(MDM, Group Policy, Ansible 등)을 통해 배포하면 된다.

| OS | Managed Policy 경로 |
|----|---------------------|
| macOS | /Library/Application Support/ClaudeCode/CLAUDE.md |
| Linux | /etc/claude-code/CLAUDE.md |
| Windows | C:\Program Files\ClaudeCode\CLAUDE.md |

## 모범 사례

Claude Code 메모리를 효과적으로 활용하기 위한 모범 사례는 다음과 같다.
- 구체적으로 작성한다: "코드를 적절히 포맷팅해라"보다 "2칸 들여쓰기를 사용해라"가 낫다
- 구조를 활용하여 정리한다: 각 개별 메모리를 불릿 포인트로 작성하고, 관련 메모리를 설명적인 마크다운 헤딩으로 그룹화한다
- 주기적으로 검토한다: 프로젝트가 발전함에 따라 메모리를 업데이트하여 Claude가 항상 최신 정보와 컨텍스트를 사용하도록 한다

추가 디렉토리에서 메모리를 로드하려면 `--add-dir` 플래그를 사용할 수 있다.
기본적으로 추가 디렉토리의 CLAUDE.md 파일은 로드되지 않지만, CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD 환경 변수를 설정하면 로드할 수 있다.

```bash
CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1 claude --add-dir ../shared-config
```

`/init` 명령으로 코드베이스에 맞는 CLAUDE.md를 빠르게 부트스트랩할 수도 있다.

## 결론

Claude Code의 메모리 시스템은 Auto Memory와 CLAUDE.md 파일이라는 두 축으로 구성되어 세션 간 지속적인 컨텍스트를 제공한다.
Auto Memory는 Claude가 자동으로 학습 내용을 저장하고 재활용하며, CLAUDE.md는 사용자가 명시적인 규칙과 선호를 정의한다.
계층적 메모리 구조를 통해 조직, 프로젝트, 개인 수준에서 유연하게 지침을 관리할 수 있다.
모듈형 규칙과 경로별 규칙 기능을 활용하면 대규모 프로젝트에서도 체계적인 지침 관리가 가능하다.

## Reference

- [Manage Claude's memory - Claude Code Documentation](https://code.claude.com/docs/en/memory)
