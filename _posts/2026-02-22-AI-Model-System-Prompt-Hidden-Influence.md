---
layout: post
title: "같은 AI 모델이 다르게 작동하는 이유 - 시스템 프롬프트의 숨은 영향력"
author: 'Juho'
date: 2026-02-22 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Prompt]
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
2. [연구 배경 및 방법](#연구-배경-및-방법)
3. [모델의 숨겨진 편향](#모델의-숨겨진-편향)
4. [실험 결과](#실험-결과)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

같은 AI 모델도 어떤 시스템 프롬프트를 사용하느냐에 따라 완전히 다른 방식으로 작동한다.
"같은 뇌를 쓰는데 다른 성격을 가진 셈"이라는 표현처럼, 모델 자체보다 프롬프트가 실제 동작을 결정하는 핵심 요소일 수 있다.
Drew Breunig와 Srihari Sriraman이 6개 코딩 에이전트의 시스템 프롬프트를 분석한 연구를 살펴본다.

## 연구 배경 및 방법

연구팀은 Claude Code, Cursor, Gemini CLI, Codex CLI, OpenHands, Kimi CLI 등 6개의 코딩 에이전트의 시스템 프롬프트를 수집했다.
context-viewer 도구를 사용해 각 프롬프트의 의미론적 구성을 비교 분석했다.
Claude Code 환경에서 동일 모델(Claude Opus 4.5)에 서로 다른 프롬프트를 적용하는 실험도 수행했다.

## 모델의 숨겨진 편향

### 병렬 도구 호출 회피

모델은 기본적으로 여러 도구를 순차적으로 호출하려는 경향이 있다.
이는 학습 데이터 대부분이 순차적 작업 예시였기 때문이다.
이를 교정하기 위해 각 에이전트는 다음과 같은 방식으로 프롬프트를 작성했다.

| 에이전트 | 교정 방식 |
|---------|---------|
| Claude Code | 여러 도구를 한 번에 호출하라는 내용을 7번 반복 강조 |
| Cursor | CRITICAL INSTRUCTION, MANDATORY 같은 대문자로 강조 |

### 과도한 코드 주석 작성

모델은 기본적으로 코드에 주석을 과도하게 달려는 경향이 있다.
튜토리얼과 노트북 같은 설명이 많은 코드가 학습 데이터에 과도하게 포함되었기 때문이다.
이를 교정하기 위해 프롬프트에 명시적인 지시를 추가했다.

| 에이전트 | 교정 방식 |
|---------|---------|
| Gemini | 절대 주석으로 사용자와 대화하지 말 것(NEVER) |
| Cursor | 당연한 코드에 주석 달지 말 것 |

## 실험 결과

같은 모델(Claude Opus 4.5)에 Claude 프롬프트와 Codex 프롬프트를 번갈아 적용한 실험에서 놀라운 차이가 나타났다.

Codex 프롬프트를 적용했을 때는 문서를 철저히 읽고 전체 계획을 세운 후 한 번에 구현했다.
Claude 프롬프트를 적용했을 때는 시도 → 에러 확인 → 수정하는 반복적인 접근 방식을 사용했다.

두 에이전트 모두 문제를 정확히 해결했지만, 해결 과정이 완전히 달랐다.
이는 에이전트의 성능이 모델만큼이나 시스템 프롬프트에 의해 결정된다는 것을 보여준다.

## 결론

AI 에이전트를 도입하거나 구축할 때 모델 선택만큼 시스템 프롬프트 설계가 중요하다.
모델이 가진 기본 편향(병렬 호출 회피, 과도한 주석)을 파악하고 프롬프트로 교정해야 한다.
동일 모델을 사용하더라도 시스템 프롬프트의 차이만으로 에이전트의 행동 방식이 크게 달라질 수 있다.
더 좋은 모델을 찾기 전에, 현재 프롬프트를 먼저 점검해보는 것이 효율적인 접근법이다.

## Reference

- [How System Prompts Reveal Model Biases - nilenso blog](https://blog.nilenso.com/blog/2026/02/12/how-system-prompts-reveal-model-biases/)
- [How System Prompts Define Agent Behavior - Drew Breunig](https://www.dbreunig.com/2026/02/10/system-prompts-define-the-agent-as-much-as-the-model.html)
