---
layout: post
title: "Nous Research Hermes Agent 자기 진화하는 AI 에이전트 프레임워크"
author: 'Juho'
date: 2026-04-18 00:00:00 +0900
categories: [AI]
tags: [Agent, AI, Skill]
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
2. [Hermes Agent 핵심 기능](#hermes-agent-핵심-기능)
   - [학습 루프와 메모리](#학습-루프와-메모리)
   - [멀티 플랫폼과 도구 통합](#멀티-플랫폼과-도구-통합)
   - [모델 유연성과 배포](#모델-유연성과-배포)
3. [Self-Evolution 시스템](#self-evolution-시스템)
   - [방법론](#방법론)
   - [최적화 대상과 단계](#최적화-대상과-단계)
   - [안전 가드레일](#안전-가드레일)
4. [설치와 사용](#설치와-사용)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Nous Research가 공개한 Hermes Agent는 경험으로부터 학습하고, 스킬을 자율적으로 만들어 다듬으며, 세션을 가로질러 영구 메모리를 유지하는 자기 개선형 AI 시스템이다.
함께 공개된 Hermes Agent Self-Evolution은 진화 알고리즘으로 에이전트 자체를 자동 개선하는 별도 프로젝트로, GPU 학습 없이 API 호출만으로 작동한다.
두 프로젝트 모두 MIT 라이선스로 공개되어 있다.

## Hermes Agent 핵심 기능

### 학습 루프와 메모리

에이전트는 복잡한 작업을 완료한 후 자동으로 스킬을 생성하고 사용 중에 다듬는다.
주기적인 강화 넛지를 통해 에이전트가 자체 큐레이션한 메모리를 유지한다.
과거 대화는 FTS5 전문 검색과 LLM 요약을 결합해 세션 간 회상이 가능하다.

### 멀티 플랫폼과 도구 통합

단일 게이트웨이 프로세스가 CLI, Telegram, Discord, Slack, WhatsApp, Signal에 동시 연결된다.
음성 메모 전사를 지원하며 플랫폼 간에 대화 연속성이 유지된다.
40개 이상의 내장 도구와 MCP(Model Context Protocol) 확장을 지원한다.
터미널 백엔드는 local, Docker, SSH, Daytona, Singularity, Modal 등 6가지를 제공한다.

### 모델 유연성과 배포

OpenRouter, Nous Portal, Hugging Face, OpenAI, 커스텀 엔드포인트를 통해 200개 이상의 모델 중에서 선택할 수 있다.
모델 전환은 코드 변경 없이 `hermes model` 명령어 하나로 처리된다.
Daytona와 Modal 백엔드는 유휴 시 하이버네이션을 지원해 세션 사이 비용을 줄인다.
5달러짜리 VPS부터 GPU 클러스터까지 다양한 인프라에서 동작한다.

## Self-Evolution 시스템

### 방법론

두 가지 핵심 기술을 결합한다.

| 기술 | 역할 |
|------|------|
| DSPy + GEPA | 실행 트레이스를 분석해 실패 원인을 파악하고 개선안 제안 |
| Darwinian Evolver | Git 기반 유기체로 코드 진화 (향후 단계 예정) |

워크플로는 현재 스킬과 프롬프트를 읽어 평가 데이터셋을 생성하고, GEPA로 후보를 최적화한 뒤 제약 게이트를 통과시켜 PR로 제안한다.
한 번의 최적화 사이클 비용은 약 2~10달러로 추정된다.

### 최적화 대상과 단계

프로젝트는 단계별로 진행된다.

| 단계 | 대상 | 상태 |
|------|------|------|
| Phase 1 | 스킬 파일 | 완료 |
| Phase 2~3 | 도구 설명, 시스템 프롬프트 | 계획 |
| Phase 4~5 | 코드 구현, 연속 루프 | 계획 |

### 안전 가드레일

진화된 변형이 적용되려면 전체 테스트 100% 통과, 스킬 크기 15KB 이하, 의미 보존 검사를 통과해야 한다.
모든 변경은 사람의 리뷰를 거친 후에만 반영된다.

## 설치와 사용

설치 명령은 한 줄로 끝난다.

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

Linux, macOS, WSL2, Termux 기반 Android를 지원한다.
주요 명령은 `hermes`(대화형 CLI), `hermes setup`(설정 마법사), `hermes gateway`(메시징 통합), `/skills`(스킬 뱅크 탐색) 등이다.
Self-Evolution 사용 예시는 다음과 같다.

```bash
python -m evolution.skills.evolve_skill \
  --skill github-code-review \
  --iterations 10 \
  --eval-source synthetic
```

OpenClaw에서 이주하는 사용자는 `hermes claw migrate` 명령으로 페르소나, 메모리, 스킬, API 키를 자동 가져올 수 있다.

## 결론

Hermes Agent는 단일 코드베이스에서 멀티 플랫폼, 멀티 모델, 자기 학습 루프를 한 번에 제공하는 야심 찬 통합형 에이전트 프레임워크다.
별도의 Self-Evolution 프로젝트는 GPU 없이 진화 알고리즘과 LLM 리플렉션만으로 에이전트를 개선하는 실험적 접근을 보여준다.
두 프로젝트 모두 자기 개선형 에이전트가 학술 영역을 넘어 실용 도구로 진입하고 있음을 시사한다.

## Reference

- [Hermes Agent](https://github.com/nousresearch/hermes-agent/)
- [Hermes Agent Self-Evolution](https://github.com/NousResearch/hermes-agent-self-evolution/)
