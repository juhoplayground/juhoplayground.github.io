---
layout: post
title: "AutoResearch - 과학 연구 자동화의 5단계 자율성 스펙트럼(L0-L4) 서베이"
author: 'Juho'
date: 2026-06-02 00:00:00 +0900
categories: [AI]
tags: [Agent, AI, LLM, Evaluation]
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
2. [AutoResearch 정의](#autoresearch-정의)
3. [L0-L4 자율성 스펙트럼](#l0-l4-자율성-스펙트럼)
   - [L0 Human Only](#l0-human-only)
   - [L1 Human-Led AI-Assisted](#l1-human-led-ai-assisted)
   - [L2 Human-Verified AI-Executed](#l2-human-verified-ai-executed)
   - [L3 AI-Led Human-Assisted](#l3-ai-led-human-assisted)
   - [L4 AI-Autonomous](#l4-ai-autonomous)
4. [Vibe Research vs AutoResearch](#vibe-research-vs-autoresearch)
5. [기술적 기반: 5단계 워크플로](#기술적-기반-5단계-워크플로)
6. [평가 차원](#평가-차원)
7. [도메인별 자율성 천장](#도메인별-자율성-천장)
8. [한계와 시사점](#한계와-시사점)
9. [결론](#결론)
10. [Reference](#reference)

## 개요

Guiyao Tie 외 23명의 저자가 공동 작성한 "AutoResearch AI: Towards AI-Powered Research Automation for Scientific Discovery"는 2026년 5월 22일 arXiv에 공개된 49페이지 서베이다.
이 서베이는 "AI가 더 강해졌다"보다 "AI가 과학 연구 워크플로를 어디까지 자동화할 수 있는가"라는 워크플로 중심 관점에서 현재 시스템을 정리한다.
저자들은 자율성을 L0에서 L4까지 5단계로 정의하고, 그 중 일부 구간을 Vibe Research로 부른다.

## AutoResearch 정의

서베이는 AutoResearch를 다음과 같이 정의한다.

> AutoResearch는 인간과 AI의 기여가 통제, 실행, 검증, 과학적 책임이라는 축을 따라 워크플로 전반에 분산되는 과학적 탐구의 워크플로 수준 패러다임이다.

핵심은 "AI가 무엇을 자동화하는가"가 아니라 "워크플로의 어느 단계에서 누가 책임을 지는가"라는 분배 문제로 재정의한 점이다.
모델 패밀리, 에이전트 아키텍처, 벤치마크 점수 같은 기존 분류 축 대신 워크플로의 통제·증거·실행·검증·책임 분배를 분석 단위로 삼는다.

## L0-L4 자율성 스펙트럼

서베이가 정의한 다섯 단계는 다음과 같다.

### L0 Human Only

인간이 문제 식별, 가설 수립, 실험 설계와 실행, 평가, 수용 판단까지 모든 과정을 직접 수행한다.
디지털 도구는 보조에 그치고, 과학적 판단과 워크플로 종결 권한은 모두 인간에게 남는다.

### L1 Human-Led AI-Assisted

워크플로는 여전히 인간 주도이지만, AI가 문헌 검색, 요약, 설명, 브레인스토밍, 초안 작성, 가벼운 분석 같은 인지 보조를 일상적으로 수행한다.
ChatGPT, Claude, Gemini 같은 일반 LLM 인터페이스가 대표적이다.

### L2 Human-Verified AI-Executed

AI가 실제 연구 노동(파일 수정, 코드 생성, 도구 호출, 분석 수행)을 수행하지만, 검증과 수용은 인간이 한다.
OpenHands, Aider, SWE-agent 같은 코딩 에이전트, 또는 The AI Scientist, AI Scientist-v2, Agent Laboratory 같은 통합 파이프라인이 여기 속한다.
파이프라인이 길어도 단계마다 인간 검증이 필요하면 L2에 머문다.

### L3 AI-Led Human-Assisted

AI가 grounding, planning, 실행, 검증, 수정, 보고를 포함한 더 큰 부분을 조직한다.
인간은 단계별 검증에서 벗어나 더 높은 수준의 감독과 예외 처리에 집중한다.
서베이는 L3을 단순한 end-to-end 파이프라인 구현이 아니라, "통상적인 분기 선택과 수용을 인간 검증 없이 진행할 수 있는지"로 판정한다.

### L4 AI-Autonomous

AI가 가설 수립, 실행, 검증, 약한 방향의 기각, provenance 보존, 보고까지 인간 개입 없이 처리한다.
서베이는 L4를 "분석적 상한선"으로 두며, 현재 시스템은 신뢰성·검증·도메인 타당성 측면에서 여기에 도달하지 못했다고 본다.

## Vibe Research vs AutoResearch

L1-L2 구간은 Vibe Research로, L3-L4 구간은 본격적인 AutoResearch로 부른다.

| 영역 | 단계 | 특징 |
|------|------|------|
| Vibe Research | L1-L2 | 프롬프트 기반 보조, 인간 검증된 AI 실행 |
| AutoResearch | L3-L4 | AI 주도 워크플로, 검증·수용까지 일부 또는 전부 AI |

서베이는 "현재의 통합 파이프라인들은 L2를 빠르게 채우고 있고, L3 방향으로 압력을 만들고 있지만, 아직 L3의 성숙한 사례는 드물다"고 평가한다.

## 기술적 기반: 5단계 워크플로

서베이는 과학 워크플로를 다섯 단계로 나누어 각 단계의 기술적 기반을 분석한다.

| 단계 | 내용 |
|------|------|
| Stage I | Literature and Research Grounding |
| Stage II | Hypothesis Formation and Planning |
| Stage III | Experimentation and Tool Use |
| Stage IV | Feedback, Validation, and Review |
| Stage V | Reporting and Knowledge Communication |

각 단계에서 자율성 천장과 인간 의존도가 다르며, 같은 시스템이라도 단계마다 다른 L 레벨에 위치할 수 있다.

## 평가 차원

서베이는 task 완료율만으로 AutoResearch를 평가하는 관행을 비판하고, 다섯 가지 평가 차원을 제시한다.

| 차원 | 설명 |
|------|------|
| Novelty | 새로움 |
| Validity | 타당성 |
| Impact | 영향력 |
| Reliability | 신뢰성 |
| Provenance | 출처와 재현성 추적 |

이 다섯 차원은 "출력이 만들어졌다" 수준을 넘어 "그 출력이 과학적으로 신뢰 가능한가"를 묻는다.

## 도메인별 자율성 천장

서베이의 중요한 결론은 자율성 천장이 도메인 의존적이라는 점이다.
연구 산출물이 구조화되고 실행 가능하며 빠르게 검증 가능한 영역(예: 계산·형식 과학, 소프트웨어 공학)에서는 더 높은 자율성이 신뢰 가능하다.
반면 신체적 실험, 지연된 검증, 이질적 증거, 윤리적 제약, 제도적 책임이 필요한 분야는 자율성 천장이 낮다.
즉 같은 L3 시스템이라도 컴퓨터 과학과 임상의학에서 신뢰도와 적용 가능성이 크게 다르다.

| 분야 | 자율성 천장이 높을 가능성 |
|------|--------------------------|
| Computational and Formal Sciences | 높음 |
| Physical Sciences and Engineering | 중간 |
| Chemistry and Materials | 중간 |
| Biology and Biomedicine | 낮음~중간 |
| Medicine and Clinical Research | 낮음 |
| Economics and Social Sciences | 낮음~중간 |
| Earth and Environmental Sciences | 중간 |
| Embodied Intelligence | 중간 |

## 한계와 시사점

서베이는 현재 시스템이 검색, 초안 작성, 코딩, 일부 도구 사용에는 강하지만, 약한 방향의 기각, 예외 처리, 재현성, provenance 보존, 책임 있는 종결에는 약하다고 진단한다.
이는 단순히 "더 큰 모델"로 해결되지 않으며, 평가 체계와 책임 구조의 재설계가 필요하다는 함의를 가진다.
저자들은 향후 연구가 task completion이 아닌 scientific credibility 중심으로 측정되어야 한다고 강조한다.

## 결론

AutoResearch 서베이는 "AI가 과학 연구에 얼마나 깊이 들어와 있는가"를 모델 성능이 아닌 워크플로 분배와 책임 구조로 재정의한다.
L0-L4 스펙트럼과 다섯 가지 평가 차원, 그리고 도메인별 자율성 천장이라는 세 축은 향후 AI for Science 논의의 공통 언어가 될 가능성이 크다.
The AI Scientist 같은 통합 파이프라인이 L3에 도달하지 못한 이유를 명확히 설명하고, AutoResearch 시스템의 책임성을 평가하기 위한 프레임을 제공한다는 점이 이 서베이의 핵심 기여다.

## Reference

- [AutoResearch AI: Towards AI-Powered Research Automation for Scientific Discovery - arXiv 2605.23204](https://arxiv.org/abs/2605.23204)
