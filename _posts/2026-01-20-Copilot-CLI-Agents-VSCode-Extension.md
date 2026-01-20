---
layout: post
title: Copilot CLI Agents - VS Code에서 Claude와 Gemini 통합하기
author: 'Juho'
date: 2026-01-20 00:00:00 +0900
categories: [AI]
tags: [Claude, Gemini, AI, LLM, Coding, VS Code]
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
2. [프로젝트 소개](#프로젝트-소개)
3. [주요 기능](#주요-기능)
4. [설치 및 사용 방법](#설치-및-사용-방법)
5. [슬래시 명령어](#슬래시-명령어)
6. [제한사항](#제한사항)
7. [결론](#결론)
8. [참고 자료](#참고-자료)

## 개요

GitHub Copilot을 사용하면서 Claude Code나 Gemini CLI의 강력한 기능을 함께 활용하고 싶었던 적이 있으신가요?
이제 VS Code를 떠나지 않고도 여러 AI 모델을 동시에 활용할 수 있는 확장 프로그램이 등장했습니다.
이 글에서는 Copilot CLI Agents 확장 프로그램의 기능과 사용 방법에 대해 살펴보겠습니다.

## 프로젝트 소개

### Copilot CLI Agents란?

Copilot CLI Agents는 GitHub Copilot Chat 내에서 Claude Code와 Gemini CLI를 직접 호출할 수 있게 해주는 VS Code 확장 프로그램입니다.
별도의 터미널이나 애플리케이션 간 컨텍스트 전환 없이 통합된 환경에서 작업할 수 있습니다.

### 개발 배경

개발자가 Copilot Pro+ 사용자이면서 동시에 Claude Pro와 Google AI Pro를 구독하는 상황에서 탄생했습니다.
Claude Code와 Gemini CLI의 강력한 기능을 활용하되, 채팅에서 터미널로 컨텍스트를 옮기는 불편함을 해소하려는 목적으로 만들어졌습니다.

### 핵심 가치

- 워크스페이스 기반 작업 디렉토리 설정
- Copilot과 다른 AI 도구 간 교차 검증 가능
- CLI의 대용량 컨텍스트 윈도우 활용 (Gemini의 경우 100만 토큰)
- 의도적으로 단순하게 설계 (암묵적 시스템 지침 최소화)

## 주요 기능

### 채팅 참여자

Copilot Chat에서 다음 명령어로 각 AI를 호출할 수 있습니다.

| 명령어 | 설명 |
|--------|------|
| @gemini | Google Gemini CLI 통합 |
| @claude | Anthropic Claude Code 통합 |

### 모델 선택

각 에이전트에서 사용할 모델을 직접 지정할 수 있습니다.
Claude의 경우 Sonnet, Opus 등 원하는 모델을 선택하여 사용할 수 있습니다.

### 도구 접근 권한 설정

보안을 위해 도구 접근 권한을 세밀하게 설정할 수 있습니다.
기본적으로 쓰기 작업 도구는 비활성화되어 있어 안전하게 사용할 수 있습니다.

### 프로젝트 구조 자동 생성

명령어를 통해 프로젝트 구조를 자동으로 생성하는 기능을 제공합니다.

## 설치 및 사용 방법

### 사전 요구사항

1. Gemini CLI 또는 Claude Code가 설치되어 있어야 합니다.
2. 각 CLI 도구에 로그인되어 있어야 합니다.
3. VS Code에 GitHub Copilot 확장이 설치되어 있어야 합니다.

### 설치 방법

1. VS Code Marketplace에서 "Copilot CLI Agents" 확장을 검색합니다.
2. 확장을 설치합니다.
3. VS Code를 재시작합니다.

### 기본 사용법

1. VS Code에서 Copilot Chat을 엽니다.
2. `@gemini` 또는 `@claude`를 입력합니다.
3. 질문이나 요청 사항을 입력합니다.

```
@claude 이 코드의 버그를 찾아줘
@gemini 이 함수를 리팩토링하는 방법을 알려줘
```

## 슬래시 명령어

### /doctor

CLI 도구의 설치 상태를 확인합니다.
Claude Code나 Gemini CLI가 제대로 설치되어 있는지, 로그인이 되어 있는지 등을 점검합니다.

### /session

현재 세션 ID를 표시합니다.
세션 관리나 디버깅 시 유용하게 사용할 수 있습니다.

### /handoff

대화형 CLI 터미널을 실행합니다.
전체 CLI 기능에 접근해야 할 때 사용합니다.
쓰기 작업을 포함한 모든 기능을 사용할 수 있습니다.

### /passAgent

커스텀 에이전트 지침을 전달합니다.
특정 작업에 맞춘 에이전트 동작을 설정할 수 있습니다.

## 제한사항

### 쓰기 작업 제한

보안상의 이유로 쓰기 작업 도구는 기본적으로 비활성화되어 있습니다.
전체 기능이 필요한 경우 `/handoff` 명령어를 통해 대화형 터미널로 전환해야 합니다.

### 컨텍스트 연결 제한

Copilot의 컨텍스트 기능(예: 현재 열린 파일 정보)은 CLI 쿼리에 자동으로 연결되지 않습니다.
필요한 컨텍스트는 명시적으로 전달해야 합니다.

### CLI 설치 필수

확장 프로그램 자체만으로는 작동하지 않습니다.
반드시 Gemini CLI 또는 Claude Code가 사전에 설치되어 있어야 합니다.

## 결론

Copilot CLI Agents는 여러 AI 도구를 구독하고 있는 개발자에게 유용한 확장 프로그램입니다.
VS Code 환경 내에서 컨텍스트 전환 없이 Claude와 Gemini의 강력한 기능을 활용할 수 있습니다.
특히 Gemini의 100만 토큰 컨텍스트 윈도우를 활용한 대용량 코드 분석이나, Claude와 Copilot 간의 교차 검증에 유용합니다.
다만 쓰기 작업이 기본적으로 비활성화되어 있으므로, 코드 수정이 필요한 경우에는 `/handoff` 명령어를 활용해야 합니다.

## 참고 자료

- [Copilot CLI Agents GitHub Repository](https://github.com/sbluemin/vsc-copilot-cli-agents)
