---
layout: post
title: "Free Claude Code - Claude Code를 17개 AI 제공자로 라우팅하는 오픈소스 프록시"
author: 'Juho'
date: 2026-06-01 00:00:00 +0900
categories: [VibeCoding]
tags: [Agent, LLM, OpenAI, Documentation]
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
2. [저장소 통계](#저장소-통계)
3. [지원 제공자](#지원-제공자)
4. [설치 방법](#설치-방법)
   - [macOS/Linux](#macoslinux)
   - [Windows](#windows)
5. [사용 워크플로](#사용-워크플로)
6. [통합 기능](#통합-기능)
7. [아키텍처](#아키텍처)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

Free Claude Code는 Claude Code의 API 트래픽을 다양한 AI 모델 제공자로 라우팅하는 오픈소스 프록시다.
Anthropic의 유료 구독 없이도 Claude Code 네이티브 클라이언트와 호환되는 방식으로 다른 제공자의 모델을 사용할 수 있게 만든다.
MIT 라이선스로 공개되어 있으며, Python 기반 FastAPI 서버 위에 17개 제공자 백엔드를 묶었다.

## 저장소 통계

| 항목 | 값 |
|------|-----|
| Stars | 30.2k |
| Forks | 4.6k |
| Watchers | 162 |
| Open Issues | 115 |
| Pull Requests | 57 |
| 라이선스 | MIT |
| 주 언어 | Python (98%) |

3만 스타와 4.6k 포크라는 수치는 단순한 사이드 프로젝트를 넘어 커뮤니티가 실제로 활용하고 있다는 신호다.

## 지원 제공자

총 17개의 제공자 백엔드를 지원한다.

| 분류 | 제공자 |
|------|--------|
| 상용 클라우드 | NVIDIA NIM, OpenRouter, Google Gemini, DeepSeek, Mistral, Z.ai |
| 코딩 특화 | OpenCode (Zen 및 Go), Wafer, Kimi |
| 고속 추론 | Cerebras, Groq, Fireworks AI |
| 로컬 실행 | LM Studio, llama.cpp, Ollama |

각 제공자에 대해 per-model 라우팅, 스트리밍, 도구 사용, thinking 블록 처리, 로컬 요청 최적화가 지원된다.

## 설치 방법

uv 패키지 매니저와 Python 3.14.0이 필요하다.

### macOS/Linux

```bash
curl -fsSL "https://github.com/Alishahryar1/free-claude-code/blob/main/scripts/install.sh?raw=1" | sh
```

### Windows

```powershell
irm "https://github.com/Alishahryar1/free-claude-code/blob/main/scripts/install.ps1?raw=1" | iex
```

한 줄 인스톨러로 설치 후 바로 사용이 가능하다.

## 사용 워크플로

설치 이후의 흐름은 세 단계로 단순하다.

1. 프록시 서버 시작: `fcc-server`
2. 로컬 Admin UI(`/admin`)에서 제공자 설정
3. Claude Code 실행: `fcc-claude`

기존 Claude Code 사용 경험을 그대로 유지하면서 백엔드만 갈아끼우는 형태다.

## 통합 기능

단순 프록시를 넘어 여러 클라이언트 환경과의 통합이 준비되어 있다.

| 통합 | 설명 |
|------|------|
| VS Code Extension | 환경 변수 기반 설정 |
| JetBrains ACP | JetBrains IDE 연동 |
| Discord/Telegram Bot | 원격 세션 사용 |
| 음성 전사 | 로컬 Whisper 또는 NVIDIA NIM 활용 |

음성 입력을 Claude Code 세션에 연결하는 옵션은 hands-free 코딩을 시도하는 사용자에게 유용하다.

## 아키텍처

FastAPI 서버가 Anthropic 호환 API 경로(`/v1/messages`, `/v1/models`)를 제공자별 프로토콜로 번역하고, 응답을 Claude Code가 기대하는 형식으로 정규화한다.
즉, Claude Code 입장에서는 자기가 평소 호출하던 Anthropic 엔드포인트를 그대로 호출하는 것처럼 보이지만, 실제로는 사용자가 선택한 다른 제공자로 트래픽이 흘러간다.
도구 사용, 스트리밍, thinking 블록 같은 Claude Code 고유 기능까지 가능한 한 보존되도록 설계되어 있다.

## 결론

Free Claude Code는 Anthropic 구독료가 부담되는 사용자, 또는 사내 정책상 외부 API 사용이 제한되어 로컬 모델로 Claude Code를 돌려야 하는 환경에서 매력적인 선택지다.
17개 제공자를 단일 인터페이스로 묶어 모델 비교나 다중 백엔드 운영을 단순화하며, MIT 라이선스로 상업적 활용이 자유롭다.
다만 일부 Claude 고유 기능(예: 특정 thinking 동작이나 캐싱)이 제공자별로 어떻게 매핑되는지는 직접 검증할 필요가 있다.

## Reference

- [free-claude-code GitHub](https://github.com/Alishahryar1/free-claude-code)
