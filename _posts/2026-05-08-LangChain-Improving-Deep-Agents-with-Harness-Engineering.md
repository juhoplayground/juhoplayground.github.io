---
layout: post
title: "하네스 엔지니어링으로 Deep Agents 점수 13.7점 끌어올리기"
author: 'Juho'
date: 2026-05-08 00:00:00 +0900
categories: [LangChain]
tags: [LangChain, Agent, Evaluation]
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
2. [실험 설계와 결과](#실험-설계와-결과)
3. [세 가지 최적화 레버](#세-가지-최적화-레버)
4. [핵심 개선 사항](#핵심-개선-사항)
   - [Build & Self-Verify 루프](#build--self-verify-루프)
   - [Context Engineering](#context-engineering)
   - [Loop Detection](#loop-detection)
   - [Reasoning Sandwich](#reasoning-sandwich)
   - [Trace Analyzer Skill](#trace-analyzer-skill)
5. [실패 모드와 대응](#실패-모드와-대응)
6. [실용적 교훈 다섯 가지](#실용적-교훈-다섯-가지)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

LangChain은 모델을 고정한 채 하네스만 바꿔서 Terminal Bench 2.0 점수를 52.8%에서 66.5%로 13.7점 끌어올렸다.
이는 Top 30 밖에서 Top 5로의 점프를 의미한다.
"We only changed the harness"라는 표현은 모델 업그레이드 없이 시스템 아키텍처 재설계만으로 이러한 개선이 가능함을 보여준다.

저자는 하네스 엔지니어링을 "모델 주변에 시스템과 도구를 구축해 작업 성능, 토큰 효율성, 지연 시간 같은 목표를 최적화하는 작업"으로 정의한다.

## 실험 설계와 결과

| 항목 | 값 |
|------|------|
| 시작 점수 | 52.8% (Top 30+) |
| 최종 점수 | 66.5% (Top 5) |
| 개선폭 | +13.7 포인트 |
| 벤치마크 | Terminal Bench 2.0 (89개 작업) |
| 사용 모델 | gpt-5.2-codex (불변) |
| Reasoning-only baseline | xhigh 모드 53.9%, high 모드 63.6% |

벤치마크는 89개의 다양한 도메인 작업으로 구성되었고, 모델은 일관되게 gpt-5.2-codex로 유지되었다.

## 세 가지 최적화 레버

연구팀은 의도적으로 하네스 구성 요소를 세 가지로 한정했다.

1. 시스템 프롬프트
2. 도구
3. 미들웨어 (모델과 도구 호출 주변의 훅)

이 제약은 변경 사항의 영향을 분석하기 쉽게 만들었다.

## 핵심 개선 사항

### Build & Self-Verify 루프

가장 중요한 개선은 의도적인 검증 사이클의 도입이었다.
저자들은 에이전트가 "솔루션을 작성하고, 자기 코드를 다시 읽고, 괜찮아 보인다고 확인한 뒤 멈추는" 패턴을 발견했다.

4단계 접근:

- **Planning & Discovery**: 작업 읽기, 코드베이스 스캔, 검증 전략을 포함한 초기 계획 수립
- **Build**: 테스트와 함께 구현, happy path와 엣지 케이스 모두 테스트
- **Verify**: 테스트 실행, 출력을 작업 명세와 비교
- **Fix**: 오류 분석, 명세 재방문, 수정 구현

구현 메커니즘은 두 가지다.
시스템 프롬프트를 이 사이클을 중심으로 재구성했고, `PreCompletionChecklistMiddleware`가 에이전트의 종료 직전에 검증 통과를 강제한다.
이는 훅이 종료 시도를 가로채는 "Ralph Wiggum Loop" 패턴의 적용이다.

### Context Engineering

`LocalContextMiddleware`는 환경 온보딩을 자동화한다.

- 시작 시 현재 작업 디렉토리와 부모/자식 디렉토리 매핑
- bash 명령으로 도구 발견(Python 설치 위치 등)
- 발견된 컨텍스트 주입으로 오류 표면 축소
- 테스트 가능한 코드 요구사항과 엄격한 타임아웃 안내
- 무한 반복 대신 검증을 유도하는 시간 예산 경고

낯선 환경에서 에이전트가 컨텍스트 조립에 실패하는 문제를 사전 차단하는 접근이다.

### Loop Detection

`LoopDetectionMiddleware`는 에이전트가 같은 파일에 10회 이상 반복 변경을 시도하는 "doom loop"를 다룬다.

- 도구 호출 훅으로 파일별 편집 횟수 추적
- N회 편집 후 "접근 방식을 재고하라"는 컨텍스트 주입
- 근시안적 계획 고착에서 벗어나도록 유도

### Reasoning Sandwich

gpt-5.2-codex는 low, medium, high, xhigh의 네 가지 추론 모드를 제공한다.
연구팀은 단계별로 다른 모드를 적용했다.

| 단계 | 추론 모드 | 의도 |
|------|------|------|
| Planning | xhigh | 복잡한 문제 이해 |
| Implementation | high | 속도와 품질의 균형 |
| Verification | xhigh | 실수 포착 |

결과 비교:

- xhigh-only: 53.9% (에이전트 타임아웃)
- high-only: 63.6%
- Reasoning sandwich: 66.5%

추론 예산을 단계에 맞게 배분하는 것이 단일 모드 사용보다 우월하다는 명확한 증거다.

### Trace Analyzer Skill

하네스 개선을 자동화하는 분석 파이프라인이다.

- LangSmith에서 실험 트레이스 수집
- 병렬 오류 분석 에이전트 스폰
- 메인 에이전트가 발견 사항을 종합하고 변경을 제안
- 선택적 인간 검증으로 과적합 방지

이를 통해 부스팅 알고리즘과 유사한 빠른 실험 사이클이 가능해졌다.

## 실패 모드와 대응

연구팀이 식별한 주요 실패 모드는 다음과 같다.

- 추론 오류
- 작업 지시 미준수
- 테스트와 검증 누락
- 시간 초과
- 근시안적 계획 고착(doom loop)
- 낯선 환경에서의 부족한 컨텍스트 발견
- 부적절한 엣지 케이스 처리

각각에 대해 시스템 프롬프트, 미들웨어, 또는 도구 변경으로 대응했다.

모델별 하네스 요구사항도 다르게 나타났다.
Claude Opus 4.6은 이전 하네스로 59.6%를 기록해 Codex보다 낮았다.
모델마다 맞춤 프롬프팅이 필요하지만, 컨텍스트 준비와 검증 중심이라는 일반 원칙은 일반화된다.

## 실용적 교훈 다섯 가지

1. **컨텍스트 엔지니어링**: 낯선 환경에서 에이전트는 컨텍스트 조립에 어려움을 겪는다. 사전 온보딩으로 오류 표면을 줄인다.
2. **자기 검증 강조**: 모델은 첫 번째 그럴듯한 솔루션에 편향된다. 테스트와 정제를 적극적으로 프롬프팅해야 한다.
3. **트레이싱을 피드백으로**: 트레이스는 에이전트 자기 평가와 도구·추론 상호작용 문제를 드러낸다.
4. **패턴 완화**: 모델의 현재 단점(blind retry, 검증 부재)을 우회하도록 설계하되, 모델이 개선되면 이러한 가드레일이 불필요해질 것을 예상한다.
5. **모델별 튜닝**: 다른 모델은 다른 프롬프팅 전략을 요구한다. 모델별 개선 루프 실행이 성능을 극대화한다.

## 결론

하네스 엔지니어링은 모델 업그레이드와 동등하거나 그 이상의 성능 향상 레버다.
13.7 포인트의 개선은 모델 변경 없이 가능했고, 이는 시스템 설계가 모델 능력을 얼마나 증폭하거나 제약할 수 있는지를 보여준다.

연구팀이 사용한 인프라는 Harbor(오케스트레이션), Daytona(샌드박스), LangSmith(관측)이다.
오픈소스 Deep Agents 저장소(Python과 JavaScript)와 트레이스 데이터셋이 공개되었다.
다중 모델 시스템, 메모리 프리미티브, RLM(Reinforcement Learning from Model traces)이 다음 연구 방향으로 제시되었다.

## Reference

- [Improving Deep Agents with Harness Engineering](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering/)
