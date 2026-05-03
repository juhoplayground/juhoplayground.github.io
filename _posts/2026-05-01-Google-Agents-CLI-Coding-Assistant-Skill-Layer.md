---
layout: post
title: "Google agents-cli - 코딩 어시스턴트를 Google Cloud 에이전트 전문가로 만드는 스킬 레이어"
author: 'Juho'
date: 2026-05-01 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, Skill, Python]
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
3. [핵심 내용](#핵심-내용)
   - [설치 방법](#설치-방법)
   - [핵심 스킬 모듈](#핵심-스킬-모듈)
   - [주요 CLI 명령](#주요-cli-명령)
4. [호환 에이전트와 아키텍처](#호환-에이전트와-아키텍처)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Google이 `agents-cli`를 공개했다.
이 도구는 그 자체가 에이전트가 아니라 "코딩 어시스턴트를 Google Cloud 에이전트 전문가로 만드는 스킬 레이어"로 포지셔닝된다.
Google Cloud에서 AI 에이전트를 생성, 평가, 배포하는 역량을 임의의 코딩 어시스턴트에 주입하는 CLI와 스킬 모음이다.

## 배경

에이전트 개발은 설계, 스캐폴딩, 평가, 배포, 관측까지 여러 단계를 거친다.
각 단계는 Google Cloud의 Agent Development Kit(ADK), Agent Runtime, Cloud Run, GKE, Gemini Enterprise 등 제각기 다른 기술과 맞닿는다.
`agents-cli`는 이 흐름을 하나의 명령줄 도구와 스킬 집합으로 묶어 Gemini CLI, Claude Code, OpenAI Codex 같은 다양한 코딩 에이전트가 공통 작업 절차로 접근할 수 있게 한다.

## 핵심 내용

### 설치 방법

전제 조건은 Python 3.11 이상, `uv` 패키지 매니저, 그리고 Node.js다.
설치 방법은 두 가지다.

```bash
uvx google-agents-cli setup
```

스킬만 설치하려면 다음 명령을 사용한다.

```bash
npx skills add google/agents-cli
```

### 핵심 스킬 모듈

`agents-cli`는 여섯 개의 스킬 모듈을 제공한다.

| 스킬 | 역할 |
|------|------|
| Workflow | 개발 생애주기와 모델 선택 가이드 |
| ADK Code | 에이전트, 도구, 오케스트레이션의 Python API 접근 |
| Scaffold | 프로젝트 생성과 확장 도구 |
| Eval | 메트릭과 스코어링을 포함한 평가 방법론 |
| Deploy | Agent Runtime, Cloud Run, GKE 배포 |
| Observability | Cloud Trace와 로깅 통합 |

개발 단계에 따라 해당 스킬이 코딩 어시스턴트의 작업 지식으로 주입되는 구조다.

### 주요 CLI 명령

핵심 명령은 다음과 같다.

```bash
agents-cli scaffold <name>
agents-cli eval run
agents-cli deploy
agents-cli publish gemini-enterprise
agents-cli login
```

`scaffold`로 프로젝트를 생성하고, `eval run`으로 평가를 실행한 뒤, `deploy`로 Google Cloud에 배포하는 순서가 일반적이다.
`publish gemini-enterprise` 명령은 생성한 에이전트를 Gemini Enterprise Agent Platform에 등록한다.

## 호환 에이전트와 아키텍처

도구는 Google Cloud의 에이전트 스택 위에 구축됐다.
ADK 프레임워크, Gemini Enterprise Agent Platform과 통합된다.

호환 가능한 코딩 에이전트는 다음과 같다.

| 에이전트 | 지원 여부 |
|---------|----------|
| Gemini CLI | 지원 |
| Claude Code | 지원 |
| OpenAI Codex | 지원 |
| 기타 코딩 에이전트 | 지원 |

라이선스는 Apache-2.0이다.
리포지토리는 공개 시점에서 스타 817, 포크 82, 오픈 이슈 8건을 기록했다.

## 결론

`agents-cli`는 Google Cloud의 에이전트 개발 절차를 표준화된 CLI와 스킬 집합으로 제공해, 어떤 코딩 어시스턴트든 동일한 워크플로에 맞춰 에이전트를 만들고 배포할 수 있게 한다.
자체 에이전트 프레임워크 경쟁 대신, 기존 코딩 에이전트 생태계에 "Google Cloud 에이전트 전문성"을 삽입하는 접근이 특징이다.
스킬 분리, Apache-2.0 라이선스, 다중 런타임 배포 지원은 엔터프라이즈 환경에서의 채택을 염두에 둔 설계로 읽힌다.

## Reference

- [google/agents-cli on GitHub](https://github.com/google/agents-cli)
