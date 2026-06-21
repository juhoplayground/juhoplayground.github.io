---
layout: post
title: "Agent Deck: 여러 AI 코딩 에이전트를 통합 관리하는 터미널 커맨드 센터"
author: 'Juho'
date: 2026-06-17 00:00:00 +0900
categories: [AI]
tags: [Agent, MCP, Management]
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
2. [아키텍처와 기술 스택](#아키텍처와-기술-스택)
   - [핵심 컴포넌트](#핵심-컴포넌트)
   - [멀티 에이전트 지원 범위](#멀티-에이전트-지원-범위)
3. [설치와 기본 사용법](#설치와-기본-사용법)
   - [설치 방법](#설치-방법)
   - [기본 명령과 TUI 단축키](#기본-명령과-tui-단축키)
4. [주요 기능](#주요-기능)
   - [Conductor 시스템](#conductor-시스템)
   - [Git Worktree와 Docker 샌드박스](#git-worktree와-docker-샌드박스)
   - [MCP 소켓 풀링과 비용 추적](#mcp-소켓-풀링과-비용-추적)
   - [Watcher 프레임워크와 원격 관리](#watcher-프레임워크와-원격-관리)
5. [설정 시스템](#설정-시스템)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

Agent Deck는 여러 AI 코딩 에이전트를 동시에 관리하는 터미널 기반 커맨드 센터다.
프로젝트는 스스로를 "AI 코딩 에이전트를 위한 미션 컨트롤"이라고 소개한다.
Claude Code, Gemini CLI, OpenCode, Codex 등 다양한 터미널 AI 도구를 하나의 통합 인터페이스에서 오케스트레이션할 수 있다.

이 도구가 해결하려는 문제는 명확하다.
여러 프로젝트에 걸쳐 다수의 에이전트 세션을 실행하는 개발자는 중앙화된 가시성과 제어가 필요하다.
여러 터미널 창을 오가는 대신, Agent Deck는 세션 관리, 상태 추적, 컨텍스트 전환, 리소스 모니터링을 하나의 터미널 UI로 통합한다.

[Agent Deck 저장소](https://github.com/asheshgoplani/agent-deck){:target="_blank"}는 MIT 라이선스로 공개되어 있으며, 현재 v1.9.70(2026년 6월) 버전을 추적하고 있다.

## 아키텍처와 기술 스택

Agent Deck는 Go로 작성되었으며, 코드베이스의 86.7%를 Go가 차지한다.
TUI는 charmbracelet의 Bubble Tea 프레임워크 위에 구축되었다.
세션 멀티플렉서로는 tmux를 기반으로 사용한다.
이 외에 TypeScript, JavaScript, Python, Shell 컴포넌트가 보조 도구로 함께 쓰인다.

### 핵심 컴포넌트

시스템은 다섯 개의 핵심 컴포넌트로 구성된다.

| 컴포넌트 | 역할 |
|------|------|
| Session Manager | AI 에이전트 세션의 레지스트리와 라이프사이클 오케스트레이션 |
| Conductor System | 자식 워커를 모니터링하는 영속적 오케스트레이터 세션 |
| MCP Manager | Model Context Protocol 서버 통합 레이어 |
| Skills Manager | Claude 전용 역량 attach와 detach |
| Watcher Framework | GitHub, Slack, webhook 등 외부 소스의 이벤트 수집 |

### 멀티 에이전트 지원 범위

Agent Deck는 도구별로 서로 다른 통합 수준을 제공한다.

| 도구 | 통합 수준 |
|------|------|
| Claude Code | 전체 지원 (status, MCP, fork, resume) |
| Gemini CLI | 전체 지원 (status, MCP, resume) |
| OpenCode | 상태 감지, 조직화, fork |
| Codex | 상태 감지, 조직화, conductor, fork |
| Copilot | 조직화, 실행 |
| 커스텀 도구 | config.toml로 설정 가능 |

세션 상태는 스마트 폴링을 통해 지능적으로 식별된다.
Running은 작업이 진행 중인 활성 상태, Waiting은 사용자 입력 대기 상태다.
Idle은 명령을 받을 준비가 된 상태, Error는 결함이 발생한 상태를 나타낸다.

## 설치와 기본 사용법

### 설치 방법

프로젝트는 여러 설치 경로를 제공한다.

| 방식 | 명령 |
|------|------|
| Curl | install.sh 스크립트 직접 실행 |
| Homebrew | brew install asheshgoplani/tap/agent-deck |
| Go | go install github.com/asheshgoplani/agent-deck/cmd/agent-deck@latest |
| Source | 저장소 클론 후 make install |

플랫폼은 macOS, Linux, Windows(WSL 경유)를 지원한다.

### 기본 명령과 TUI 단축키

빠른 세션 작업은 다음과 같은 명령으로 수행한다.

```bash
agent-deck                    # TUI 실행
agent-deck add . -c claude    # Claude로 세션 등록
agent-deck session fork my-proj   # 세션 fork 생성
agent-deck mcp attach my-proj exa # MCP 서버 attach
agent-deck web                # 127.0.0.1:8420 에서 웹 인터페이스 시작
```

TUI 내 키보드 내비게이션은 다음과 같다.

| 키 | 동작 |
|------|------|
| Enter | 선택한 세션에 attach |
| n | 새 세션 생성 (Enter로 필드 이동, Ctrl+S로 제출) |
| f / F | 빠른 fork / 커스터마이즈 fork 다이얼로그 |
| / | 세션 퍼지 검색 |
| G | 전역 대화 검색 |
| $ | 비용 대시보드 |
| m / s | MCP Manager / Skills Manager |
| ? | 전체 도움말 레퍼런스 |

## 주요 기능

### Conductor 시스템

Conductor는 자식 워커를 모니터링하는 장시간 실행 오케스트레이터 세션이다.
설정 워크플로는 영속적인 슈퍼바이저를 구축한다.

```bash
agent-deck conductor setup work --description "Work fleet"
agent-deck session start conductor-work
```

Conductor의 주요 역량은 다음과 같다.
워커의 일상적인 질의에 자동으로 응답한다.
복잡한 이슈는 사람 운영자에게 에스컬레이션한다.
Telegram과 Slack과 통합되어 원격 제어를 지원한다.
세션 전환을 추적하는 상태 파일을 유지하며, 기본 15분 간격의 heartbeat 기반 모니터링을 지원한다.

### Git Worktree와 Docker 샌드박스

세션은 격리된 Git worktree 안에서 동작할 수 있다.
새 worktree에 세션을 생성하려면 add 명령에 worktree와 new-branch 옵션을 붙인다.
worktree finish 명령은 브랜치를 머지하고 정리한다.
gitignore된 산출물을 선택적으로 복사하기 위해 .worktreeinclude 파일을 지원하며, 선택적으로 .agent-deck/worktree-setup.sh와 worktree-destruction.sh 스크립트를 실행한다.

```toml
[worktree]
default_location = "subdirectory"  # "sibling" (기본값) 또는 커스텀 경로
```

세션은 격리된 컨테이너 안에서도 실행할 수 있다.
프로젝트 디렉토리는 읽기-쓰기로 마운트된다.
도구 인증은 샌드박스 디렉토리를 통해 자동으로 공유되며, macOS에서는 keychain 자격 증명이 추출된다.
docker 섹션에서 default_enabled = true 옵션으로 활성화한다.

### MCP 소켓 풀링과 비용 추적

여러 세션은 Unix 소켓을 통해 MCP 프로세스를 공유한다.
이 방식은 메모리를 85-90% 절감한다.
설정에서 pool_all = true로 활성화할 수 있다.

비용 추적 대시보드는 세션 전반의 사용량을 실시간으로 모니터링한다.
Claude Code 트랜스크립트에서 자동으로 데이터를 수집한다.
Claude Opus/Sonnet/Haiku, Gemini, GPT-4o 변형 등 14개 모델의 가격이 책정되어 있으며, 가격은 매일 갱신할 수 있다.
TUI 대시보드는 $ 키로, 웹 인터페이스는 /costs 엔드포인트로 접근한다.
예산 한도는 일간, 주간, 월간, 그룹별, 세션별 수준으로 설정할 수 있다.

### Watcher 프레임워크와 원격 관리

Watcher 프레임워크는 네 가지 어댑터 유형을 통해 이벤트 기반 오케스트레이션을 제공한다.

| 어댑터 | 목적 | 설정 |
|------|------|------|
| webhook | 범용 HTTP POST 리스너 | --port |
| github | HMAC 검증이 포함된 저장소 이벤트 | --secret |
| ntfy | ntfy.sh의 푸시 알림 | --topic |
| slack | ntfy로 연결되는 Cloudflare Worker 브리지 | --topic |

설정 예시는 다음과 같다.

```bash
agent-deck watcher create github --name gh-alerts --secret $GITHUB_WEBHOOK_SECRET
agent-deck watcher start gh-alerts
```

Agent Deck는 SSH로 접근 가능한 원격 머신으로도 확장된다.

```bash
agent-deck remote add dev user@dev-box
agent-deck remote sessions dev
agent-deck remote attach dev my-session
agent-deck remote update dev
```

원격 관리는 known_hosts에 대한 OpenSSH 호스트-키 검증, 엄격한 인증을 강제하는 BatchMode, 바이너리 배포 시 SHA-256 체크섬 검증을 보안 장치로 갖춘다.

## 설정 시스템

기본 설정 위치는 XDG_CONFIG_HOME/agent-deck/config.toml이며, 기본적으로 ~/.config/agent-deck/config.toml을 사용한다.

그룹별 Claude 설정 예시는 다음과 같다.

```toml
[groups."conductor".claude]
config_dir = "~/.claude-team"
env_file = "~/git/work/.envrc"
```

설정 우선순위는 가장 구체적인 것에서 덜 구체적인 순서로 적용된다.
CLAUDE_CONFIG_DIR 환경 변수가 가장 우선한다.
다음으로 conductors 블록, groups 블록, profiles 블록, claude 전역 섹션, 마지막으로 ~/.claude 기본값 순서다.

tmux 소켓을 격리하면 개인 설정과의 충돌을 방지할 수 있다.

```toml
[tmux]
socket_name = "agent-deck"
```

## 결론

Agent Deck는 여러 터미널 AI 에이전트를 하나의 통합 인터페이스에서 관리하려는 개발자를 위한 도구다.
세션 관리, Conductor 기반 오케스트레이션, MCP 통합, 비용 추적, Git worktree 격리, 원격 제어를 단일 터미널 UI로 묶어낸다.
Go와 Bubble Tea, tmux를 기반으로 한 가벼운 구조 위에서 멀티 에이전트 워크플로를 중앙에서 통제할 수 있다는 점이 핵심 가치다.
멀티 에이전트 환경의 운영 복잡도가 커지는 상황에서, 통합 커맨드 센터라는 접근은 실용적인 해법을 제시한다.

## Reference

- [Agent Deck GitHub Repository](https://github.com/asheshgoplani/agent-deck/)
