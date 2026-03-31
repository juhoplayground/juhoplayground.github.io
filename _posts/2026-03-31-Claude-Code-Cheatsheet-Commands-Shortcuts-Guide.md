---
layout: post
title: "Claude Code 치트시트 - 단축키, 명령어, 설정 완벽 가이드"
author: 'Juho'
date: 2026-03-31 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, MCP]
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
2. [키보드 단축키](#키보드-단축키)
   - [일반 제어](#일반-제어)
   - [모드 전환](#모드-전환)
   - [입력 및 접두사](#입력-및-접두사)
   - [세션 피커](#세션-피커)
3. [슬래시 명령어](#슬래시-명령어)
   - [세션 관리](#세션-관리)
   - [설정 관련](#설정-관련)
   - [도구 및 통합](#도구-및-통합)
   - [특수 명령어](#특수-명령어)
4. [메모리 시스템](#메모리-시스템)
5. [MCP 서버 관리](#mcp-서버-관리)
6. [CLI 사용법](#cli-사용법)
   - [핵심 명령어](#핵심-명령어)
   - [주요 플래그](#주요-플래그)
   - [권한 모드](#권한-모드)
7. [고급 워크플로우](#고급-워크플로우)
   - [Plan 모드](#plan-모드)
   - [Thinking과 Effort](#thinking과-effort)
   - [Git Worktree](#git-worktree)
   - [Voice 모드](#voice-모드)
   - [컨텍스트 관리](#컨텍스트-관리)
8. [스킬과 에이전트](#스킬과-에이전트)
9. [설정 파일과 환경 변수](#설정-파일과-환경-변수)
10. [커뮤니티 피드백](#커뮤니티-피드백)
11. [결론](#결론)
12. [Reference](#reference)

## 개요

Claude Code v2.1.83 버전 기준의 인터랙티브 레퍼런스 가이드인 [Claude Code Cheat Sheet](https://cc.storyfox.cz/){:target="_blank"}가 공개되었다.
이 치트시트는 A4 가로 형식으로 인쇄 가능하며, 색상으로 구분된 섹션으로 구성되어 있다.
크론 작업을 통해 매일 자동 업데이트되며, 새로운 기능에는 NEW 배지가 표시된다.
Mac/Windows 단축키를 자동으로 감지하여 플랫폼에 맞는 정보를 보여준다.

이 글에서는 치트시트에 포함된 키보드 단축키, 슬래시 명령어, MCP 설정, 메모리 시스템, CLI 사용법, 고급 워크플로우 등을 정리한다.

## 키보드 단축키

### 일반 제어

| 단축키 | 기능 |
|--------|------|
| Ctrl+C | 입력/생성 취소 |
| Ctrl+D | 세션 종료 |
| Ctrl+L | 화면 초기화 |
| Ctrl+O | 상세 출력 토글 |
| Ctrl+R | 히스토리 역방향 검색 |
| Ctrl+G | 프롬프트 에디터 열기 |
| Ctrl+X Ctrl+E | 에디터 열기 (Ctrl+G 별칭) |
| Ctrl+B | 실행 중인 작업 백그라운드로 전환 |
| Ctrl+T | 작업 목록 토글 |
| Ctrl+V | 이미지 붙여넣기 |
| Ctrl+X Ctrl+K | 백그라운드 에이전트 종료 |
| Esc Esc | 되돌리기(undo) |

### 모드 전환

| 단축키 | 기능 |
|--------|------|
| Shift+Tab | 권한 모드 순환 (Normal - Auto-Accept - Plan) |
| Alt+P | 모델 전환 |
| Alt+T | Thinking 토글 |

### 입력 및 접두사

| 입력 | 기능 |
|------|------|
| \ Enter | 줄바꿈 (빠른 방법) |
| Ctrl+J | 줄바꿈 (제어 시퀀스) |
| / | 슬래시 명령어 |
| ! | 직접 bash 실행 |
| @ | 파일 멘션 + 자동완성 |

### 세션 피커

| 키 | 기능 |
|----|------|
| 위/아래 화살표 | 탐색 |
| 좌/우 화살표 | 펼치기/접기 |
| P | 미리보기 |
| R | 이름 변경 |
| / | 검색 |
| A | 모든 프로젝트 |
| B | 현재 브랜치 |

## 슬래시 명령어

Claude Code는 40개 이상의 슬래시 명령어를 제공한다.
용도별로 분류하면 다음과 같다.

### 세션 관리

| 명령어 | 기능 |
|--------|------|
| /clear | 대화 초기화 |
| /compact [focus] | 컨텍스트 압축 (포커스 지정 가능) |
| /resume | 세션 재개/전환 |
| /rename [name] | 현재 세션 이름 지정 |
| /branch [name] | 대화 분기 (/fork 별칭) |
| /cost | 토큰 사용량 통계 |
| /context | 컨텍스트 시각화 (그리드 뷰) |
| /diff | 인터랙티브 diff 뷰어 |
| /copy [N] | 마지막 (또는 N번째) 응답 복사 |
| /rewind | 대화/코드 체크포인트로 되돌리기 |
| /export | 대화 내보내기 |

### 설정 관련

| 명령어 | 기능 |
|--------|------|
| /config | 설정 열기 |
| /model [model] | 모델 전환 (effort 조정 포함) |
| /fast [on/off] | 빠른 모드 토글 |
| /vim | vim 모드 토글 |
| /theme | 색상 테마 변경 |
| /permissions | 권한 보기/수정 |
| /effort [level] | effort 설정 (low/med/high/max/auto) |
| /color [color] | 프롬프트 바 색상 설정 |
| /keybindings | 키보드 단축키 커스터마이즈 |
| /terminal-setup | 터미널 키바인딩 설정 |

### 도구 및 통합

| 명령어 | 기능 |
|--------|------|
| /init | CLAUDE.md 생성 |
| /memory | CLAUDE.md 파일 편집 |
| /mcp | MCP 서버 관리 |
| /hooks | 훅 관리 |
| /skills | 사용 가능한 스킬 목록 |
| /agents | 에이전트 관리 |
| /chrome | Chrome 통합 |
| /reload-plugins | 플러그인 핫 리로드 |
| /add-dir [path] | 작업 디렉토리 추가 |

### 특수 명령어

| 명령어 | 기능 |
|--------|------|
| /btw [question] | 사이드 질문 (컨텍스트 비용 없음) |
| /plan [desc] | Plan 모드 (자동 시작) |
| /loop [interval] | 반복 예약 작업 |
| /voice | 푸시투톡 음성 (20개 언어) |
| /doctor | 설치 진단 |
| /pr-comments [PR] | GitHub PR 코멘트 가져오기 |
| /stats | 사용 통계 및 선호도 |
| /insights | 세션 분석 리포트 |
| /desktop | Desktop 앱에서 계속 |
| /remote-control (/rc) | claude.ai/code 브리지 |
| /usage | 플랜 제한 및 요금 상태 |
| /schedule | 클라우드 예약 작업 |
| /security-review | 변경사항 보안 분석 |
| /batch | 대규모 병렬 변경 (5-30 worktree) |

## 메모리 시스템

Claude Code의 메모리 시스템은 CLAUDE.md 파일을 기반으로 동작한다.
컨텍스트가 압축(compaction)될 때도 CLAUDE.md 내용은 유지된다.

### CLAUDE.md 저장 위치

| 위치 | 경로 | 용도 |
|------|------|------|
| 프로젝트 | ./CLAUDE.md | 팀 공유용 |
| 개인 | ~/.claude/CLAUDE.md | 모든 프로젝트에 적용 |
| 조직 | /etc/claude-code/ | 조직 전체 관리 |

### 규칙 및 임포트

프로젝트별 규칙은 `.claude/rules/*.md` 경로에, 사용자별 규칙은 `~/.claude/rules/*.md` 경로에 저장한다.
frontmatter의 `paths:` 필드로 특정 경로에만 적용되는 규칙을 설정할 수 있다.
CLAUDE.md 내에서 `@path/to/file` 구문으로 외부 파일을 임포트할 수 있다.

### 자동 메모리

자동 메모리는 `~/.claude/projects/<proj>/memory/` 디렉토리에 저장된다.
MEMORY.md와 주제별 파일이 포함되며 자동으로 로드된다.

## MCP 서버 관리

MCP(Model Context Protocol)를 통해 로컬 및 원격 서버에 연결할 수 있다.

### 전송 방식

| 방식 | 설명 |
|------|------|
| http | 원격 HTTP (권장) |
| stdio | 로컬 프로세스 |
| sse | 원격 SSE |

### 설정 범위

| 범위 | 파일 | 설명 |
|------|------|------|
| 로컬 | settings.local.json | 사용자 전용 |
| 프로젝트 | .mcp.json | 공유/VCS |
| 사용자 | ~/.claude.json | 전역 |

### 관리 명령어

`/mcp` 명령어로 인터랙티브 UI를 사용할 수 있다.
`claude mcp list`로 모든 서버를 조회하고, `claude mcp serve`로 Claude Code 자체를 MCP 서버로 사용할 수 있다.
MCP 서버는 작업 중간에 사용자 입력을 요청하는 Elicitation 기능도 지원한다.

## CLI 사용법

### 핵심 명령어

| 명령어 | 기능 |
|--------|------|
| claude | 인터랙티브 모드 |
| claude "질문" | 프롬프트와 함께 실행 |
| claude -p "쿼리" | 헤드리스(비대화형) 모드 |
| claude -c | 마지막 대화 이어서 |
| claude -r "이름" | 이름으로 세션 재개 |
| claude update | CLI 업데이트 |

### 주요 플래그

| 플래그 | 기능 |
|--------|------|
| --model | 모델 설정 |
| -w | Git worktree |
| -n / --name | 세션 이름 |
| --add-dir | 디렉토리 추가 |
| --agent | 에이전트 사용 |
| --allowedTools | 도구 사전 승인 |
| --output-format json/stream | 출력 형식 |
| --json-schema | 구조화된 출력 |
| --max-turns | 턴 제한 |
| --max-budget-usd | 비용 상한 |
| --console | Anthropic Console 인증 |
| --verbose | 상세 로깅 |
| --bare | 최소 헤드리스 (훅/LSP 없음) |
| --remote | 웹 세션 |
| --effort low/med/high/max | effort 수준 |
| --permission-mode | 권한 모드 |
| --dangerously-skip-permissions | 모든 프롬프트 건너뛰기 (주의) |

### 권한 모드

| 모드 | 설명 |
|------|------|
| default | 작업마다 프롬프트 표시 |
| acceptEdits | 편집 자동 수락 |
| plan | 읽기 전용 |
| dontAsk | 허용된 것 외 거부 |
| bypassPermissions | 모두 건너뛰기 |

## 고급 워크플로우

### Plan 모드

Shift+Tab으로 Normal, Auto-Accept, Plan 모드를 순환할 수 있다.
`--permission-mode plan` 플래그로 시작 시 바로 Plan 모드에 진입한다.
Plan 모드에서는 읽기 전용으로 동작하여 코드 변경 없이 계획만 수립할 수 있다.

### Thinking과 Effort

Alt+T로 Thinking을 켜고 끌 수 있다.
프롬프트에 "ultrathink"을 입력하면 해당 턴에 최대 effort가 적용된다.
Ctrl+O로 상세 출력 모드에서 Thinking 과정을 확인할 수 있다.

`/effort` 명령어로 effort 수준을 설정한다.

| 수준 | 표시 |
|------|------|
| low | 빈 원 |
| medium | 반 원 |
| high | 가득 찬 원 |

### Git Worktree

`--worktree name` 또는 `-w` 플래그로 기능별 격리된 브랜치에서 작업할 수 있다.
에이전트 frontmatter에서 `isolation: worktree`를 설정하면 에이전트가 자체 worktree에서 실행된다.
`sparsePaths`로 필요한 디렉토리만 체크아웃하여 효율적으로 작업할 수 있다.
`/batch` 명령어는 5-30개의 worktree를 자동 생성하여 대규모 병렬 변경을 수행한다.

### Voice 모드

`/voice` 명령어로 푸시투톡 음성 입력을 활성화한다.
Space 키를 누른 상태에서 녹음하고, 놓으면 전송된다.
영어, 스페인어, 프랑스어, 독일어, 체코어, 폴란드어 등 20개 언어를 지원한다.

### 컨텍스트 관리

`/context` 명령어로 현재 컨텍스트 사용량과 최적화 팁을 확인할 수 있다.
`/compact [focus]`로 포커스를 지정하여 컨텍스트를 압축할 수 있다.
약 95% 용량에 도달하면 자동으로 컨텍스트가 압축된다.
Opus 4.6 모델은 Max/Team/Enterprise 플랜에서 최대 1M 토큰 컨텍스트를 사용할 수 있다.

## 스킬과 에이전트

### 내장 스킬

| 스킬 | 기능 |
|------|------|
| /simplify | 코드 리뷰 (3개 병렬 에이전트) |
| /batch | 대규모 병렬 변경 (5-30 worktree) |
| /debug [desc] | 디버그 로그에서 문제 해결 |
| /loop [interval] | 반복 예약 작업 |
| /claude-api | API + SDK 레퍼런스 로드 |

### 커스텀 스킬

프로젝트 스킬은 `.claude/skills/<name>/` 경로에, 개인 스킬은 `~/.claude/skills/<name>/` 경로에 저장한다.

스킬 frontmatter에서 설정할 수 있는 주요 필드는 다음과 같다.

| 필드 | 기능 |
|------|------|
| description | 자동 호출 트리거 |
| allowed-tools | 권한 프롬프트 건너뛰기 |
| model | 스킬용 모델 오버라이드 |
| effort | effort 수준 오버라이드 |
| context: fork | 서브에이전트에서 실행 |
| {% raw %}$ARGUMENTS{% endraw %} | 사용자 입력 플레이스홀더 |
| {% raw %}${CLAUDE_SKILL_DIR}{% endraw %} | 스킬 자체 디렉토리 |

### 내장 에이전트

| 에이전트 | 기능 |
|----------|------|
| Explore | 빠른 읽기 전용 (Haiku 모델) |
| Plan | Plan 모드용 리서치 |
| General | 전체 도구, 복잡한 작업 |
| Bash | 별도 컨텍스트의 터미널 |

에이전트 frontmatter에서 `permissionMode`, `isolation`, `memory`, `background`, `maxTurns` 등을 설정할 수 있다.

## 설정 파일과 환경 변수

### 설정 파일 위치

| 파일 | 용도 |
|------|------|
| ~/.claude/settings.json | 사용자 설정 |
| .claude/settings.json | 프로젝트 설정 (공유) |
| .claude/settings.local.json | 로컬 전용 |
| ~/.claude.json | OAuth, MCP, 상태 |
| .mcp.json | 프로젝트 MCP 서버 |

### 주요 환경 변수

| 변수 | 기능 |
|------|------|
| ANTHROPIC_API_KEY | API 키 |
| ANTHROPIC_MODEL | 모델 지정 |
| CLAUDE_CODE_EFFORT_LEVEL | effort 수준 (low/med/high) |
| MAX_THINKING_TOKENS | Thinking 토큰 (0으로 비활성화) |
| ANTHROPIC_CUSTOM_MODEL_OPTION | 커스텀 모델 항목 |
| CLAUDE_CODE_PLUGIN_SEED_DIR | 다중 플러그인 시드 디렉토리 |
| CLAUDECODE | 셸 감지 (=1) |
| IS_DEMO | 데모 모드 (이메일/조직 숨김) |
| CLAUDE_CODE_MAX_OUTPUT_TOKENS | 최대 출력 토큰 (기본 32K) |
| CLAUDE_CODE_DISABLE_CRON | 크론 비활성화 |

### v2.1.83 최신 변경사항

최신 버전에서 추가된 주요 변경사항은 다음과 같다.

- `managed-settings.d/`를 통한 드롭인 정책 프래그먼트 지원
- `CwdChanged` / `FileChanged` 훅 이벤트 추가
- 에이전트 종료 키바인딩이 Ctrl+X Ctrl+K로 변경 (이전 Ctrl+F)
- 상세 모드에서 `/` 키로 트랜스크립트 검색, `n`/`N`으로 단계 탐색
- `initialPrompt` 에이전트 frontmatter 지원

## 커뮤니티 피드백

커뮤니티에서는 이 치트시트의 포괄성을 높이 평가했다.
로그인 없이 접근할 수 있다는 점도 호평을 받았다.
다만 Mac과 Windows 간 키보드 단축키 불일치 문제가 지적되었다.
특히 이미지 붙여넣기와 편집 관련 단축키에서 Cmd+V와 Ctrl+V 간 차이가 언급되었다.
일부 사용자는 자신의 버전에서 /cost 같은 특정 명령어가 존재하지 않는다고 보고했다.
환경 변수 목록이 불완전하다는 의견도 있었다.

## 결론

Claude Code 치트시트는 CLI 기반 AI 코딩 도구의 방대한 기능을 체계적으로 정리한 레퍼런스다.
40개 이상의 슬래시 명령어, 다양한 키보드 단축키, MCP 서버 관리, 메모리 시스템, Git Worktree 등 고급 워크플로우까지 한눈에 파악할 수 있다.
매일 자동 업데이트되므로 최신 기능을 놓치지 않고 확인할 수 있다.
Claude Code를 본격적으로 활용하려는 개발자라면 이 치트시트를 북마크해 두는 것을 권장한다.

## Reference

- [Claude Code Cheat Sheet](https://cc.storyfox.cz/)
