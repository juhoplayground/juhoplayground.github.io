---
layout: post
title: "AI Co-Mathematician: 수학자와 협업하는 에이전트 워크벤치, FrontierMath Tier 4에서 48% 달성"
author: 'Juho'
date: 2026-05-15 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, LLM, Benchmark, Evaluation]
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
2. [배경과 선행 연구](#배경과-선행-연구)
   - [수학 연구의 미지원 차원](#수학-연구의-미지원-차원)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [7가지 설계 원칙](#7가지-설계-원칙)
   - [에이전트 계층 구조](#에이전트-계층-구조)
   - [Hard programmatic 제약](#hard-programmatic-제약)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [실제 수학자 사례 3건](#실제-수학자-사례-3건)
   - [내부 100문제 벤치마크](#내부-100문제-벤치마크)
   - [FrontierMath Tier 4 - 48%](#frontiermath-tier-4---48)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"AI Co-Mathematician: Accelerating Mathematicians with Agentic AI"는 수학자가 AI 에이전트와 함께 개방형 연구를 수행할 수 있는 비동기 stateful 워크벤치를 제시한 Google DeepMind 논문이다.
저자는 Daniel Zheng, Ingrid von Glehn, Yori Zwols, Iuliya Beloshapka, Lars Buesing, Daniel M. Roy, Martin Wattenberg, Bogdan Georgiev, Tatiana Schmidt, Andrew Cowie, Fernanda Viegas, Dimitri Kanevsky, Vineet Kahlon, Hartmut Maennel, Sophia Alj, George Holland, Alex Davies, Pushmeet Kohli이며 2026년 5월 7일 공개되었다.

논문 abstract의 핵심은 다음과 같다.
AI co-mathematician은 수학 연구의 ideation, literature search, computational exploration, theorem proving, theory building을 통합 지원하는 비동기 stateful workspace다.
초기 테스트에서 미해결 문제 해결, 새 연구 방향 발견, 누락된 문헌 발견에 기여했다.
FrontierMath Tier 4에서 48%를 달성해 평가된 모든 AI 시스템 중 최고 점수를 기록했다.

## 배경과 선행 연구

### 수학 연구의 미지원 차원

저자들은 일상 수학 연구가 isolated query나 computer-assisted proof의 연속이 아니라 불확실성 관리, 문헌 합성, intermediate artifact 작성·수정, 가설 추적 같은 long-term stateful collaborative workflow임을 강조한다.
표준 chat 인터페이스는 transient하고 specialized engine은 broader context가 부족해 연구자가 직접 brainstorming, formal prover, computational script 사이의 manual connective tissue 역할을 한다.
SWE 분야의 Antigravity, Claude Code, OpenAI Codex가 보여준 효과적 인간-에이전트 협업 패턴을 수학에 적용해야 한다는 동기다.

### Related Work 정리

| 카테고리 | 주요 선행 연구 | 본 연구와의 차이 |
|----------|----------------|------------------|
| 자율 reasoning | Minerva, Aletheia | autonomous — 본 연구는 stateful workbench |
| Exploratory search | AlphaEvolve | 진화 탐색 — 본 연구는 통합 plug 가능 |
| Formalized math | AlphaProof, Aristotle | proof 특화 — 본 연구는 informal 포함 |
| Inference scaling | o1, Gemini Deep Think | chat — 본 연구는 비동기 workspace |
| SWE 협업 | Claude Code, Codex, Antigravity | code 특화 — 본 연구는 수학 native |

본 연구의 차별점은 수학 워크플로우의 messy reality를 그대로 지원하는 orchestration이라는 점이다.

## 방법론

### 7가지 설계 원칙

| 원칙 | 내용 |
|--------|--------|
| Embrace mathematics beyond proofs | theorem proving 외 ideation, 문헌 검색, 시뮬레이션 모두 통합 지원 |
| Support iterative refinement of intent | 초기 질문 자체를 반복적으로 다듬을 수 있는 인터페이스 |
| Produce native mathematical artifacts | 살아있는 "working paper"와 margin annotation으로 결과 표현 |
| Enable asynchronous interaction | 다수의 specialized agent가 병렬로 작동, 사용자는 언제나 개입 가능 |
| Manage cognitive load via progressive disclosure | high-level intent와 low-level execution을 분리해 표시 |
| Track, manage, and communicate uncertainty | version history, validation, friction 가시화 |
| Preserve the history of failed explorations | dead-end도 first-class outcome으로 보존 |

### 에이전트 계층 구조

상단에 project coordinator agent, 그 아래에 workstream coordinator agent들이 병렬로 위치하며, 각 workstream coordinator는 sub-agent(literature search, coding, Gemini Deep Think 등)를 dispatch한다.
모든 agent는 shared file system에 작업을 기록하고 internal messaging system으로 통신한다.

각 workstream은 LaTeX write-up을 main output으로 만들며 reviewer agent들이 reference, code, 논리적 정합성을 cross-check하는 review 절차를 거친다.
모든 reviewer가 공식 승인해야 finalize 되며, 통과 못 하면 escalation 메시지가 사용자에게 surface된다.

### Hard programmatic 제약

표준 AI agent는 어려운 문제에서 invalid shortcut, hallucinated lemma, 디테일 hand-wave, premature 성공 선언 같은 failure mode를 보인다.
AI co-mathematician은 이를 막기 위해 hard programmatic 제약을 적용한다.
예를 들어 coding sub-agent는 test가 통과하고 reviewer agent가 code와 golden value를 승인할 때까지 코드를 finished로 마킹할 수 없다.

reviewer agent의 거절이 반복되면 workstream coordinator가 block되고, 시스템은 silent restart 대신 dead-end를 durable record로 보존하며 project coordinator가 사용자에게 alert를 surface한다.

## 실험 셋업

| 항목 | 값 |
|--------|-------|
| Base 모델 | Gemini 3.1 Pro (대부분), Gemini 3.1 Deep Think (prover agent) |
| 실행 모드 | 일반 stateful + 벤치마크 단일 답변 모드 |
| Time limit | 내부 24시간, FrontierMath 48시간 |
| Internal benchmark | 100문제, code-checkable, professional mathematician 출제 |
| FrontierMath Tier 4 | 50문제 (sample 2 + 48 평가), Epoch AI blind 평가 |
| 평가 방식 | Epoch AI가 UI 직접 사용해 입력·답 확인 (개발자는 미관여) |

벤치마크 모드의 차이점은 초기 problem-definition 대화 우회, project goal을 "solve the problem"으로 hard-coding, fixed time limit 강제다.

## 주요 결과

### 실제 수학자 사례 3건

**Marc Lackenby — Kourovka Notebook Problem 21.10**
모든 finite group이 just finite presentation을 갖는지를 묻는 문제.
시스템이 prove/disprove 두 workstream을 동시 실행했고, 첫 결과는 시스템 자체가 incorrect로 마킹했다.
하지만 Lackenby는 "really clever proof strategy"를 발견하고 reviewer critique를 읽으며 gap을 메울 방법을 알아내 완전한 증명에 도달했다.

**Gergely Bérczi — Stirling Coefficient 추측**
대칭 거듭제곱 표현의 Stirling coefficient에 대한 log-concavity 추측.
시스템은 두 workstream에서 별도 증명을 제시하고(현재 detailed human review 중) 미해결 추측에 대한 computational 증거를 추가로 제공했다.
n=1, 2에서 추측이 거짓임이 발견되어 추측을 수정하고 새 proof strategy를 도출했다.

**Semon Rezchikov — Hamiltonian Diffeomorphism Lemma**
Hamiltonian diffeomorphism의 perturbation 존재성에 관한 technical sub-problem.
시스템이 elegant proof를 가진 핵심 lemma를 도출해 careful review에 통과했고 본질적으로 문제를 해결했다.
Rezchikov는 "I would rank, aesthetically, its general style of proofs as the best one of any models I've gotten to use"라고 평했다.

### 내부 100문제 벤치마크

100개 unleaked, code-checkable research-level 문제에서 AI co-mathematician이 single Gemini 3.1 Pro 호출과 single Gemini Deep Think 호출을 모두 유의하게 능가했다.
SAT solver(PySAT)로 환원해 푸는 geometric tiling 문제, 정확한 정리 statement를 literature에서 retrieve해 푸는 representation theory 문제, theoretical과 computational workstream을 분리해 푸는 combinatorics 문제가 시스템 강점 사례로 보고된다.

### FrontierMath Tier 4 - 48%

Epoch AI가 blind로 진행한 FrontierMath Tier 4 평가 결과.

| 시스템 | 정확도 |
|----------|---------|
| Gemini 3.1 Pro (base) | 19% |
| AI co-mathematician | 48% (23/48, sample 2 제외) |

이전 어떤 시스템도 풀지 못했던 3문제를 정답 처리했지만, 이전에 적어도 한 시스템이 풀었던 2문제를 놓쳤다.
FrontierMath 표준 harness는 Python 인터프리터에 token 한계를 두지만 본 시스템은 자체 tool 구현과 무제한 model call을 사용해 inference 비용이 더 높다.

## 한계와 디스커션

저자가 명시한 한계는 다음과 같다.

첫째, Reviewer-Pleasing Bias(False Consensus).
agent가 결함 있는 주장을 진정 수정할 수 없을 때 reviewer 만족 제약이 결함을 detection 어려운 형태로 변형시켜 수렴할 수 있다.

둘째, Intractable Disagreements(Non-Termination, "death spiral").
review process가 끝없이 revision-rejection cycle에 갇힐 수 있고 hallucinated reasoning으로 degrade되기도 한다.

셋째, System Autonomy Requires Ceding Control.
수학 연구는 본질적으로 exploratory이고 사전 task planning이 종종 불가능하다. 모델이 unplanned 상황에서 내리는 판단이 인간 능력에 한참 못 미친다.

넷째, Semantic Meaning of Representations.
잘 typeset된 LaTeX 문서가 rigor를 보장한다는 인간의 직관이 LLM에서는 깨진다.

생태계 차원의 challenge.

첫째, Maintaining Signal-to-Noise in the Literature — agent가 plausible하지만 shallow한 paper를 대량 생성할 위험.
둘째, Adapting the Peer Review Ecosystem — agent가 분 단위로 20페이지 proof를 만들지만 인간 검증은 며칠이 걸리는 비대칭.

## 결론

AI co-mathematician은 수학 연구의 messy, iterative, collaborative reality를 stateful workbench로 native 지원한 첫 사례다.
Lackenby의 Kourovka Notebook 미해결 문제 해결, Bérczi의 Stirling 추측 보정, Rezchikov의 Hamiltonian lemma 증명 등 실제 수학자가 미해결 문제를 해결한 사례를 보였다.
FrontierMath Tier 4 48%(Gemini 3.1 Pro base 19% 대비 +29%p)와 내부 100문제 벤치마크 우위로 정량 효과도 입증했다.
hard programmatic 제약과 7가지 설계 원칙으로 LLM의 비신뢰성을 다루는 한 가지 구체적 청사진을 제시한 의미 있는 연구다.

## Reference

- [AI Co-Mathematician: Accelerating Mathematicians with Agentic AI (arXiv:2605.06651)](https://arxiv.org/abs/2605.06651/)
