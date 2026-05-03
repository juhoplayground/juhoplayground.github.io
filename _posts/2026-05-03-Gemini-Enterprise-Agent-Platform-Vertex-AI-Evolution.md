---
layout: post
title: "Gemini Enterprise Agent Platform - Vertex AI를 잇는 4 필러 에이전트 플랫폼"
author: 'Juho'
date: 2026-05-03 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, Management, Security]
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
   - [Pillar 1: BUILD](#pillar-1-build)
   - [Pillar 2: SCALE](#pillar-2-scale)
   - [Pillar 3: GOVERN](#pillar-3-govern)
   - [Pillar 4: OPTIMIZE](#pillar-4-optimize)
   - [Model Garden과 통합](#model-garden과-통합)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Google Cloud가 2026년 4월 23일 Gemini Enterprise Agent Platform을 공개했다.
이는 기존 Vertex AI를 흡수하여 진화시킨 엔터프라이즈 에이전트 인프라 플랫폼이다.
공식 블로그는 "앞으로 모든 Vertex AI 서비스와 로드맵은 더 이상 독립 서비스가 아니라 Agent Platform을 통해서만 제공된다"고 명시했다.

플랫폼은 BUILD, SCALE, GOVERN, OPTIMIZE 네 개의 필러로 구성되며, Model Garden을 통해 200개 이상의 모델에 접근할 수 있다.
Burns & McDonnell, Color Health, Comcast, L'Oréal, PayPal 등이 초기 도입 사례로 이름을 올렸다.

## 배경

엔터프라이즈가 자율 에이전트를 프로덕션에 올리려 할 때 마주치는 문제는 단일 모델 호출이 아니다.
멀티모달 입출력, 다일(multi-day) 워크플로, 에이전트 간 위임, 정체성 관리, 도구 거버넌스, 위협 탐지, 지속 평가가 한 번에 필요해진다.
지금까지 Vertex AI는 모델 학습/배포 중심이었고, 에이전트 운영은 사용자가 직접 조립해야 했다.

Google은 이 조립 부담을 플랫폼으로 끌어올렸다.
Agent Platform은 개발(BUILD), 운영(SCALE), 통제(GOVERN), 개선(OPTIMIZE)이라는 에이전트 라이프사이클 전 구간을 단일 제품으로 묶는다.
Gemini Enterprise 앱과 직접 통합되어 사내 직원에게도 그대로 배포된다.

## 핵심 내용

### Pillar 1: BUILD

개발 단계는 코드 우선과 비주얼 우선을 모두 지원한다.
Agent Studio는 로우코드 비주얼 인터페이스로 에이전트를 조립한다.
Agent Development Kit(ADK)은 코드 우선 환경으로 월 6조 토큰 이상을 처리하는 규모로 운영되고 있다.

Agent Garden은 코드 현대화, 재무 분석, 송장 처리 같은 사전 빌드 템플릿을 제공한다.
Workspace는 모델이 생성한 bash 명령을 안전하게 실행하는 샌드박스다.
멀티모달 스트리밍은 실시간 오디오/비디오를 지원한다.

Native Ecosystem Integrations는 엔터프라이즈 시스템에 플러그앤플레이로 붙는 커넥터를 제공한다.
Batch & Event-driven Agents는 BigQuery 테이블과 Pub/Sub 스트림에 직접 연결되어 백그라운드에서 동작한다.

### Pillar 2: SCALE

운영 단계는 재설계된 Agent Runtime이 핵심이다.
서브초(sub-second) 콜드 스타트와 초 단위 에이전트 프로비저닝을 제공한다.
Multi-day Workflows는 며칠 단위로 지속되는 장기 실행 에이전트를 위한 영속성을 제공한다.

Agent Sandbox는 모델이 생성한 코드 실행과 브라우저 자동화를 위한 강화된 환경이다.
Agent-to-Agent Orchestration은 생성형 패턴과 결정론적 패턴을 모두 지원하는 위임 메커니즘이다.
Memory Bank는 Memory Profile 기반의 동적 장기 컨텍스트를 생성한다.

Agent Sessions는 커스텀 Session ID를 사용해 CRM이나 데이터베이스에 매핑된다.
Bidirectional Streaming은 WebSocket 프로토콜로 실시간 양방향 상호작용을 지원한다.

### Pillar 3: GOVERN

거버넌스 계층은 이전에 별도로 발표되었던 5계층 거버넌스 스택을 그대로 흡수한다.
Agent Identity는 에이전트별 고유 암호 ID를 부여하고 모든 행동을 감사 가능한 트레일로 남긴다.
Agent Registry는 승인된 도구, 스킬, 자산의 중앙 라이브러리다.

Agent Gateway는 에이전트 생태계의 항공 관제탑 역할을 하며 통합 연결성을 제공한다.
Model Armor는 프롬프트 인젝션과 데이터 유출을 차단한다.
Agent Anomaly Detection은 통계 모델과 LLM-as-a-judge를 결합해 의심스러운 행동을 표시한다.

Agent Threat Detection은 리버스 셸이나 악성 IP 같은 실시간 악성 활동을 탐지한다.
Agent Security Dashboard는 Security Command Center 위에서 통합 위협 탐지 뷰를 제공한다.

### Pillar 4: OPTIMIZE

지속 개선 계층은 시뮬레이션, 평가, 관찰, 자동 튜닝의 네 가지로 구성된다.
Agent Simulation은 합성된 사람과 유사한 상호작용을 통제된 환경에서 재현해 회귀 테스트를 자동화한다.
Agent Evaluation은 멀티턴 자동평가자(autorater)를 사용해 라이브 트래픽을 지속적으로 채점한다.

Agent Observability는 복잡한 추론 과정을 시각적으로 트레이싱하고 디버깅 대시보드를 제공한다.
Agent Optimizer는 실패를 자동으로 클러스터링하고 시스템 인스트럭션 개선안을 제안한다.

### Model Garden과 통합

Model Garden을 통해 200개 이상의 모델을 단일 인터페이스로 사용할 수 있다.
1st-party 모델로는 Gemini 3.1 Pro, Gemini 3.1 Flash Image, Lyria 3, 그리고 오픈 모델 Gemma 4가 제공된다.
3rd-party로는 Anthropic Claude(Opus, Sonnet, Haiku)가 정식 지원된다.

| Pillar | 핵심 컴포넌트 | 목적 |
|--------|--------------|------|
| BUILD | Agent Studio, ADK, Agent Garden | 개발과 템플릿 |
| SCALE | Agent Runtime, Memory Bank, Sessions | 프로덕션 운영 |
| GOVERN | Identity, Registry, Gateway, Model Armor | 보안과 통제 |
| OPTIMIZE | Simulation, Evaluation, Observability, Optimizer | 지속 개선 |

## 의미와 시사점

가장 큰 변화는 Vertex AI라는 이름의 사실상의 종료다.
공식 발표는 "모든 Vertex AI 서비스와 로드맵 진화는 Agent Platform을 통해 독점적으로 제공된다"고 명시했다.
모델 서빙 플랫폼이 에이전트 플랫폼으로 완전히 재정의되었다는 신호다.

두 번째는 4 필러가 라이프사이클 단절을 줄이려는 시도라는 점이다.
지금까지 빌드, 배포, 거버넌스, 평가는 서로 다른 도구로 운영되었고 그 사이의 데이터 손실이 운영 부담의 원인이었다.
하나의 플랫폼이 ID, 트레이스, 메모리, 평가 결과를 일관되게 보유하면 디버깅과 보안 모두에서 유리하다.

세 번째는 자체 모델만으로 가두지 않는 모델 중립 정책이다.
Gemini 3.1 라인업과 함께 Claude Opus/Sonnet/Haiku가 1급 시민으로 명시된 것은 엔터프라이즈가 멀티 모델 전략을 유지하면서도 단일 운영 평면을 갖길 원한다는 시장 요구를 반영한다.
PayPal의 Agent Payment Protocol(AP2) 코멘트는 결제 같은 규제 워크로드까지 이 플랫폼이 후보가 될 수 있음을 시사한다.

다만 가격은 발표에 명시되지 않았다.
Try now로 즉시 사용 가능하지만, 4 필러 전부를 켰을 때의 운영 비용은 도입 검토 시 별도로 측정해야 할 영역이다.

## 결론

Gemini Enterprise Agent Platform은 Google Cloud의 AI 제품 라인업을 에이전트 중심으로 재배치하는 발표다.
Vertex AI는 이름 뒤로 물러나고, BUILD/SCALE/GOVERN/OPTIMIZE 4 필러가 새 진입점이 된다.
도입을 검토하는 조직이라면 모델 선택보다 라이프사이클 통합과 거버넌스 흡수 비용을 먼저 평가하는 것이 합리적이다.

## Reference

- [Introducing Gemini Enterprise Agent Platform](https://cloud.google.com/blog/products/ai-machine-learning/introducing-gemini-enterprise-agent-platform?hl=en)
