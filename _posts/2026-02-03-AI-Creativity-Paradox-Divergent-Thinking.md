---
layout: post
title: AI 창의성의 역설 - 평균은 넘었지만 천재는 못 따라간다
author: 'Juho'
date: 2026-02-03 00:00:00 +0900
categories: [Research]
tags: [AI, LLM, Benchmark, ChatGPT]
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
2. [연구 배경](#연구-배경)
3. [연구 방법론](#연구-방법론)
4. [주요 연구 결과](#주요-연구-결과)
5. [Temperature와 프롬프트 전략의 영향](#temperature와-프롬프트-전략의-영향)
6. [시사점](#시사점)
7. [Reference](#reference)

## 개요

몬트리올 대학교(Universite de Montreal) 연구팀이 Scientific Reports에 발표한 논문 "Divergent creativity in humans and large language models"는 AI와 인간의 창의성을 대규모로 비교 분석한 연구이다.
10만 명의 인간 참가자와 ChatGPT, Claude, Gemini 등 주요 LLM의 창의적 사고 능력을 체계적으로 비교했다.
연구 결과, LLM이 인간 평균 창의성 점수를 넘어섰지만, 상위권 인간의 창의성에는 여전히 미치지 못하는 역설적 결과가 나타났다.

## 연구 배경

AI의 창의성에 대한 논의가 활발해지고 있지만, LLM의 의미적 다양성(semantic divergence)을 인간의 발산적 사고(divergent thinking)와 체계적으로 비교한 연구는 부족했다.
이 연구는 계산적 창의성(computational creativity)의 최신 기법을 활용하여 이 격차를 해소하고자 했다.
연구팀에는 딥러닝 선구자이자 Mila(Quebec AI Institute) 창립자인 Yoshua Bengio도 포함되어 있다.

발산적 사고란 연상적 사고(associative thinking) 능력, 즉 의미 공간에서 먼 개념들을 접근하고 결합하는 능력을 의미한다.
이는 창의적 인지의 핵심 요소로 널리 인정받고 있다.

## 연구 방법론

### Divergent Association Task (DAT)

DAT는 참가자에게 의미적으로 가능한 한 멀리 떨어진 단어 10개를 생성하도록 요청하는 과제이다.
더 창의적인 사람일수록 더 넓은 의미적 범위를 포괄하는 단어를 생성하여, 단어 간 평균 의미 거리가 더 크게 나타난다.

### 창작 글쓰기 과제

DAT 외에도 장문 창작 글쓰기로 방법론을 확장했다.
하이쿠, 시놉시스, 플래시 픽션 등 다양한 장르의 창작물을 평가했다.
평가에는 Divergent Semantic Integration(DSI)과 Lempel-Ziv(LZ) 복잡도 점수를 활용했다.

### 평가 대상

- 인간 참가자 : 10만 명의 대규모 샘플
- LLM : GPT-3, GPT-4, Claude, Gemini Pro 등 2023년 1월부터 2025년 6월까지 출시된 주요 모델
- 동일한 객관적 채점 기준을 인간과 AI 모두에 적용

## 주요 연구 결과

### DAT 점수 비교

GPT-4는 전체 인간 샘플의 평균보다 높은 DAT 점수를 달성했다.
Gemini Pro는 인간과 통계적으로 유사한 수준의 성과를 보였다.
최적 조건에서 AI 모델은 평균 DAT 점수 85.6을 달성하여, 전체 인간 점수의 72%보다 높은 결과를 보였다.

### 인간 상위권과의 비교

그러나 상위 50% 인간 참가자는 테스트된 모든 AI 모델보다 높은 점수를 기록했다.
상위 10% 인간은 그 격차를 더욱 벌렸다.
이는 최고 수준의 인간 발산적 사고와 AI 사이에 지속적인 격차가 존재함을 보여준다.

### 창작 글쓰기 평가

일부 LLM은 다양한 단어 생성에서 인간을 앞섰지만, 스토리와 시 창작에서는 뒤처졌다.
생성 언어 모델의 크기가 창작 글쓰기 품질과 비례하지 않았다.
인간은 여전히 더 발산적인 하이쿠와 시놉시스를 작성했다.

## Temperature와 프롬프트 전략의 영향

### Temperature 조정 효과

GPT-4의 Temperature를 높이면 시놉시스와 플래시 픽션 과제에서 창의성 점수가 눈에 띄게 향상되었다.
이는 DAT에서의 하이퍼파라미터 튜닝 효과가 창작 글쓰기에도 재현됨을 확인시켜 준다.
다만 Temperature 상향이 무작위성을 늘리는 것과 의미 있는 창의성 향상은 구별되어야 한다.

### 프롬프트 전략 효과

LLM의 성과는 프롬프트 전략에 따라 크게 달라졌다.
다양한 어원의 단어를 사용하도록 명시적으로 프롬프트를 구성했을 때, GPT-3와 GPT-4 모두 기본 DAT 프롬프트보다 높은 성과를 보였다.
이는 단어의 뿌리를 참조함으로써 의미적 발산을 향상시킬 수 있는 가능성을 시사한다.

## 시사점

이 연구는 AI 창의성의 역설을 명확히 보여준다.
LLM은 훈련 데이터의 평균적 패턴을 재현하는 데 뛰어나지만, 천재적 수준의 창의성은 여전히 인간의 영역으로 남아 있다.

AI가 곧 창작 전문가를 대체할 것이라는 우려에 대해, 이 연구는 그러한 두려움이 아직 시기상조임을 시사한다.
최고 성능의 인간과 가장 진보한 LLM 사이의 지속적인 격차는 가장 높은 수준의 창의적 역할이 현재 AI 시스템으로 대체되기 어려움을 보여준다.

또한 이 연구의 방법론적 프레임워크는 창의성 지표가 향후 모델 성능 평가의 표준 척도가 될 수 있는 기반을 마련했다는 점에서 의의가 있다.

## Reference

- [Divergent creativity in humans and large language models - Scientific Reports](https://www.nature.com/articles/s41598-025-25157-3)
- [Divergent Creativity in Humans and Large Language Models - arXiv](https://arxiv.org/abs/2405.13012)
