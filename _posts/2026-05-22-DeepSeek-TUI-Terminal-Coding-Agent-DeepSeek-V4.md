---
layout: post
title: "DeepSeek-TUI - 터미널에서 실행되는 DeepSeek V4 코딩 에이전트"
author: 'Juho'
date: 2026-05-22 00:00:00 +0900
categories: [AI]
tags: [Agent, LLM, Dev, MCP]
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
2. [핵심 기능](#핵심-기능)
   - [자동 모델 선택](#자동-모델-선택)
   - [세 가지 운영 모드](#세-가지-운영-모드)
   - [도구 통합과 MCP 확장](#도구-통합과-mcp-확장)
3. [기술 스택과 아키텍처](#기술-스택과-아키텍처)
4. [설치와 사용](#설치와-사용)
   - [설치 경로](#설치-경로)
   - [기본 사용 예시](#기본-사용-예시)
   - [주요 단축키](#주요-단축키)
5. [지원 모델과 배포 옵션](#지원-모델과-배포-옵션)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

DeepSeek-TUI는 DeepSeek V4 기반의 터미널 코딩 에이전트다.
"Terminal coding agent for DeepSeek V4"라는 한 줄 설명대로, 터미널에서 파일 편집, 셸 실행, Git 관리, 웹 검색, 서브 에이전트 조율을 수행한다.
GitHub에서 30.2k 별과 2.5k 포크를 기록하며 활발한 커뮤니티를 형성하고 있다.

DeepSeek Inc.와 공식적인 관계는 없는 커뮤니티 프로젝트라는 점은 미리 짚어둘 필요가 있다.

## 핵심 기능

### 자동 모델 선택

`--model auto` 옵션이 가장 흥미로운 차별점이다.
매 턴마다 작업 성격에 따라 모델(deepseek-v4-pro 또는 deepseek-v4-flash)과 사고 수준(thinking level)을 자동으로 선택한다.
복잡한 추론에는 pro, 단순 응답에는 flash를 쓰는 식으로 비용과 성능을 균형 맞춘다.

스트리밍 사고 블록도 함께 제공된다.
DeepSeek의 추론 과정을 실시간으로 화면에 표시하므로, 에이전트가 어떤 근거로 다음 행동을 결정하는지 사용자가 추적할 수 있다.

### 세 가지 운영 모드

운영 모드는 사용자가 통제 수준을 선택할 수 있게 한다.

| 모드 | 설명 | 적합한 상황 |
|------|------|------------|
| Plan | 읽기 전용 | 코드베이스 탐색, 계획 수립 |
| Agent | 도구 사용 시 승인 필요 | 일반 개발 작업 |
| YOLO | 자동 승인 | 자동화된 배치 작업 |

YOLO 모드는 신뢰할 수 있는 작업이나 격리된 환경에서 사용할 것이 권장된다.
Plan 모드는 새 코드베이스에 처음 들어갈 때 안전하게 둘러보는 용도다.

### 도구 통합과 MCP 확장

기본 도구로 파일 작업, 셸 실행, Git, 웹 검색 및 브라우징이 내장돼 있다.
MCP(Model Context Protocol) 서버 연결을 지원하여 외부 도구를 손쉽게 추가할 수 있다.
1M 토큰 컨텍스트 윈도우와 프리픽스 캐시를 지원해 긴 대화의 비용 효율도 챙겼다.

세션 저장 및 재개 기능 덕분에 장시간 작업이 중단되어도 이어서 진행할 수 있다.
UI는 영어, 일본어, 중국어 간체, 포르투갈어를 지원한다.

## 기술 스택과 아키텍처

코드베이스의 94.9%가 Rust로 작성됐다.
TypeScript 2.8%, JavaScript 1.4%가 보조적으로 쓰인다.

| 컴포넌트 | 기술 |
|---------|------|
| 언어 | Rust |
| UI 프레임워크 | ratatui |
| API 클라이언트 | OpenAI 호환 스트리밍 |
| 아키텍처 | 디스패처 CLI + TUI 런타임 + 비동기 엔진 |

아키텍처는 `deepseek` 디스패처 CLI가 `deepseek-tui` 컴패니언 바이너리를 호출하고, 그 위에서 ratatui가 화면을 그리는 구조다.
도구 호출은 타입화된 레지스트리(typed registry)를 통해 라우팅된다.
새로운 도구 추가가 안전한 타입 시스템 위에서 이뤄진다.

## 설치와 사용

### 설치 경로

여러 패키지 매니저를 통해 설치할 수 있다.

```bash
npm install -g deepseek-tui
cargo install deepseek-tui-cli --locked
brew tap Hmbown/deepseek-tui && brew install deepseek-tui
```

직접 다운로드, Docker, Nix 패키지 관리자도 지원된다.
첫 실행 시 DeepSeek API 키 입력이 요구되며, `~/.deepseek/config.toml`에 저장된다.

### 기본 사용 예시

```bash
deepseek                                  # 대화형 TUI 실행
deepseek "질문"                            # 한 번의 프롬프트 실행
deepseek --model auto "작업"               # 자동 모델 선택
deepseek --yolo                           # 자동 도구 승인
deepseek serve --http                     # HTTP/SSE API 서버
```

`serve --http` 모드는 다른 도구에서 DeepSeek-TUI를 백엔드로 사용하고 싶을 때 유용하다.
Zed 에디터와의 통합도 지원된다.

### 주요 단축키

| 키 | 동작 |
|-----|------|
| Tab | 자동완성 또는 모드 전환 |
| Shift+Tab | 사고 난이도 순환 |
| F1 | 도움말 오버레이 |
| Ctrl+K | 명령 팔레트 |
| Ctrl+R | 이전 세션 재개 |

## 지원 모델과 배포 옵션

| 모델 | 특징 |
|------|------|
| deepseek-v4-pro | 1M 토큰 컨텍스트 |
| deepseek-v4-flash | 경량 빠른 응답 |

NVIDIA NIM, OpenRouter, 자체 호스팅(SGLang, vLLM, Ollama)을 통한 배포가 가능하다.
실시간 비용 추적과 LSP(Language Server Protocol) 기반 코드 진단도 내장돼 있다.
라이선스는 MIT다.

## 결론

DeepSeek-TUI는 DeepSeek 생태계에 특화된 터미널 코딩 에이전트의 잘 만들어진 사례다.
Rust 기반의 안정적 구조 위에 자동 모델 선택, 세 단계 운영 모드, MCP 확장, 1M 토큰 컨텍스트 같은 현대적 에이전트 기능을 모두 담았다.
DeepSeek V4를 진지하게 코딩 워크플로우에 통합하려는 사용자에게 우선 검토할 만한 옵션이다.

## Reference

- [DeepSeek-TUI GitHub](https://github.com/Hmbown/DeepSeek-TUI/)
