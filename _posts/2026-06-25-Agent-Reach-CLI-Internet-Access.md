---
layout: post
title: "Agent-Reach: AI 에이전트에게 API 비용 없이 인터넷 접근을 주는 CLI"
author: 'Juho'
date: 2026-06-25 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, Python]
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
2. [배경](#배경)
3. [작동 방식](#작동-방식)
   - [멀티 백엔드 라우팅](#멀티-백엔드-라우팅)
   - [지원 플랫폼](#지원-플랫폼)
4. [설치와 사용법](#설치와-사용법)
   - [설치](#설치)
   - [CLI 명령어](#cli-명령어)
   - [사용 예시](#사용-예시)
5. [인증과 한계](#인증과-한계)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

Agent-Reach는 AI 에이전트에게 인터넷 접근 능력을 부여하는 통합 CLI 도구다.
Claude Code, OpenClaw, Cursor, Windsurf 같은 에이전트가 대상이다.
자체 스크래퍼를 새로 만드는 대신, 이미 존재하는 오픈소스 도구들을 번들로 묶고 지능형 백엔드 라우팅으로 오케스트레이션한다.
핵심 차별점은 유료 API 없이 Twitter, Reddit, YouTube, GitHub, Bilibili, XiaoHongShu 등의 콘텐츠를 읽게 해준다는 점이다.

## 배경

AI 에이전트는 인터넷 작업에서 자주 막힌다.
Twitter 검색은 유료 API를 요구한다.
Reddit 접근은 차단당한다.
YouTube는 자막 추출이 필요하다.
XiaoHongShu(샤오훙수)는 로그인을 요구하고, Bilibili는 봇 차단 장치가 있다.
플랫폼마다 별도의 설정, 인증, 우회책이 필요해지는 것이다.
Agent-Reach는 이 마찰을 제거하는 것을 목표로 한다.

## 작동 방식

Agent-Reach는 능력 계층(ability layer)으로 동작한다.
각 플랫폼마다 기본 도구 하나와 여러 폴백(fallback) 옵션을 둔다.
하나의 백엔드가 실패하면 시스템이 자동으로 다음 백엔드를 순서대로 시도한다.
모든 백엔드 도구는 오픈소스이며 유료 API 요구사항이 없다.

### 멀티 백엔드 라우팅

라우팅은 채널별로 우선순위가 정해진 도구 체인으로 구성된다.
백엔드가 차단되면 다음 도구로 자동 전환되므로 사용자는 중단을 거의 겪지 않는다.

아래는 라우팅 체인의 예시다.

| 채널 | 라우팅 순서 |
|------|-------------|
| Twitter | twitter-cli → OpenCLI → bird |
| Bilibili | bili-cli → OpenCLI → search API |
| Reddit | OpenCLI(데스크톱) → rdt-cli |

이 프로젝트는 단순 스냅샷이 아니라 플랫폼 변화를 지속적으로 모니터링한다.
예를 들어 2026년 6월 yt-dlp가 Bilibili에서 차단되었을 때, 메인테이너가 다음 백엔드로 전환하는 식이다.

### 지원 플랫폼

플랫폼별로 무설정(Zero-Config) 가능 여부와 로그인 필요 여부, 사용 도구가 다르다.

| 플랫폼 | 무설정 | 로그인 필요 | 도구 |
|--------|--------|-------------|------|
| 웹 페이지 | 가능 | 불필요 | Jina Reader |
| YouTube | 가능 (자막) | 불필요 | yt-dlp |
| RSS/Atom | 가능 | 불필요 | feedparser |
| GitHub | 가능 (공개) | 비공개 레포 | gh CLI |
| Twitter/X | 가능 (읽기) | 검색/타임라인 | twitter-cli |
| Bilibili | 가능 (검색) | 자막 | bili-cli |
| Reddit | 불가 | 필요 | OpenCLI/rdt-cli |
| XiaoHongShu | 불가 | 필요 | OpenCLI |
| 글로벌 검색 | 가능 | 불필요 | Exa (MCP) |
| LinkedIn | 가능 (공개) | 프로필 | linkedin-mcp |
| V2EX | 가능 | 불필요 | Native |
| Xueqiu | 가능 | 불필요 | Native |

## 설치와 사용법

### 설치

설치는 에이전트에게 한 줄로 지시하는 방식이다.
에이전트가 pip 설치, Node.js 감지, 시스템 패키지 설정, 환경 구성, 채널 등록을 모두 처리한다.

설치 시 안전 옵션을 사용할 수 있다.

| 옵션 | 동작 |
|------|------|
| install --safe | 미리보기만 수행, 자동 설치 안 함 |
| install --dry-run | 어떤 작업이 일어날지 표시 |

환경 자동 감지로 설치하려면 다음과 같이 실행한다.

```bash
agent-reach install --env=auto
```

OpenClaw 사용자는 설치 전에 exec 권한을 먼저 활성화해야 한다.

```bash
openclaw config set tools.profile "coding"
```

### CLI 명령어

진단과 제거를 포함한 주요 명령어는 다음과 같다.

```bash
# 설치
agent-reach install --env=auto

# 진단: 채널별로 현재 활성화된 백엔드를 확인
agent-reach doctor

# 제거
agent-reach uninstall
agent-reach uninstall --keep-config  # 토큰 등 설정 보존
```

agent-reach doctor 명령은 각 채널에서 현재 어떤 백엔드가 동작 중인지 진단해준다.

### 사용 예시

번들된 도구들은 다음과 같이 직접 호출된다.

```bash
# 웹페이지 읽기
curl https://r.jina.ai/URL

# GitHub 레포 정보
gh repo view owner/repo

# YouTube 자막
yt-dlp --write-sub --skip-download "URL"

# Bilibili 검색
bili search "AI教程"

# Twitter 검색
twitter search "keyword"

# RSS 파싱
feedparser parse "feed-url"

# 글로벌 시맨틱 검색
exa search "LLM frameworks"
```

## 인증과 한계

쿠키 기반 플랫폼(Twitter, XiaoHongShu, Reddit)은 인증 절차가 필요하다.
데스크톱에서는 Chrome 확장 "Cookie-Editor"로 쿠키를 내보내 에이전트에 전달한다.
서버 환경에서는 OpenCLI(데스크톱) 또는 플랫폼별 MCP 서버의 QR 코드 로그인을 사용한다.
자격 증명은 로컬의 {% raw %}~/.agent-reach/config.yaml{% endraw %}에 권한 600으로 저장된다.
GitHub 비공개 레포는 gh auth login으로 인증한다.
Exa 검색은 MCP를 통해 키 없이 무료로 동작하므로 API 키가 필요 없다.

다만 명확한 한계도 있다.

| 항목 | 내용 |
|------|------|
| 읽기 전용 중심 | 자동 폼 제출, 멀티 계정 격리, 복잡한 브라우저 자동화는 미지원 (BrowserAct 사용 권장) |
| 계정 정지 위험 | 스크립트 접근 시 Twitter/XiaoHongShu는 일회용 계정 사용 권장 |
| 지역 접근 | 중국 본토 사용자는 Reddit/Twitter용으로 월 1달러가량 프록시 필요 |

비용 측면에서 Agent-Reach 자체는 완전 무료다.
모든 백엔드 도구가 오픈소스라 유료 API 요구사항이 없다.
지역 차단 우회용 거주지 프록시만 월 1달러 정도 선택적으로 든다.

## 결론

Agent-Reach는 스크래퍼를 새로 만들지 않고 검증된 오픈소스 도구들을 라우팅으로 묶어 AI 에이전트의 인터넷 접근 문제를 해결한다.
백엔드가 실패하면 자동으로 다음 도구로 전환되는 멀티 백엔드 구조가 핵심이다.
유료 API 없이 다양한 플랫폼을 읽을 수 있고, 플랫폼 변화에 따라 지속적으로 유지보수된다는 점에서 에이전트 운영에 실용적인 선택지다.

## Reference

- [Agent-Reach (GitHub)](https://github.com/Panniantong/Agent-Reach/)
