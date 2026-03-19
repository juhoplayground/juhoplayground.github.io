---
layout: post
title: "Attention Residuals: 기존 잔차 연결을 대체하는 새로운 Transformer 아키텍처"
author: 'Juho'
date: 2026-03-19 00:00:00 +0900
categories: [AI]
tags: [AI, LLM]
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
2. [방법론](#방법론)
   - [기존 잔차 연결의 한계](#기존-잔차-연결의-한계)
   - [Full AttnRes](#full-attnres)
   - [Block AttnRes](#block-attnres)
3. [주요 결과](#주요-결과)
   - [벤치마크 성능](#벤치마크-성능)
   - [학습 역학](#학습-역학)
4. [구현](#구현)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Moonshot AI에서 Attention Residuals(AttnRes)라는 새로운 Transformer 아키텍처 변형을 공개했다.
이 프로젝트는 기존 Transformer 모델에서 사용되는 표준 잔차 연결(residual connection)을 학습 가능한 어텐션 기반 집계 메커니즘으로 대체하는 구조적 변경을 제안한다.
Chen, Guangyu 등이 2026년에 발표한 이 연구는 [GitHub 저장소](https://github.com/MoonshotAI/Attention-Residuals/tree/master){:target="_blank"}에서 공식 코드를 확인할 수 있다.

## 방법론

### 기존 잔차 연결의 한계

표준 잔차 연결은 각 레이어의 출력을 고정된 균일 가중치로 누적한다.
모델이 깊어질수록 두 가지 핵심적인 문제가 발생한다.
첫째, 각 레이어의 기여도가 희석되어 개별 레이어의 영향력이 감소한다.
둘째, PreNorm 구조의 한계로 인해 은닉 상태(hidden state)의 크기가 무한정 증가할 수 있다.
이러한 문제는 모델의 학습 효율성과 성능에 부정적인 영향을 미친다.

### Full AttnRes

Full AttnRes는 기존 잔차 연결을 근본적으로 재설계한 방식이다.
각 레이어가 이전 모든 레이어의 출력에 대해 어텐션 가중치를 계산한다.
레이어마다 하나의 학습된 유사 쿼리(pseudo-query) 벡터를 사용하며, 이 벡터의 차원은 d이다.
이를 통해 입력에 의존적이고 콘텐츠를 인식하는 방식으로 이전 표현들을 선택할 수 있다.
최종 출력은 softmax로 계산된 어텐션 가중치를 사용하여 이전 출력들의 가중합으로 구성된다.

### Block AttnRes

Block AttnRes는 Full AttnRes의 실용적인 변형이다.
네트워크를 약 8개의 블록으로 분할하는 방식을 사용한다.
블록 내부(intra-block)에서는 기존의 표준 잔차 연결을 그대로 적용한다.
블록 수준의 표현에 대해서만 어텐션을 사용하여 계산량을 줄인다.
이 방식은 메모리 복잡도를 O(Ld)에서 O(Nd)로 감소시킨다.
기존 모델에 최소한의 오버헤드로 바로 적용할 수 있는 드롭인(drop-in) 교체 방식이다.

## 주요 결과

### 벤치마크 성능

Kimi Linear 48B 모델에서 1.4T 토큰으로 학습한 결과는 다음과 같다.

| 벤치마크 | 기존 모델 | AttnRes 적용 | 성능 향상 |
|----------|-----------|-------------|-----------|
| GPQA-Diamond | 36.9 | 44.4 | +7.5 |
| HumanEval | 59.1 | 62.2 | +3.1 |
| MMLU | 73.5 | 74.6 | +1.1 |
| C-Eval | 79.6 | 82.5 | +2.9 |

특히 GPQA-Diamond에서 +7.5포인트의 가장 큰 성능 향상을 보여주었다.
Block AttnRes는 1.25배 더 많은 연산량으로 학습한 기존 모델과 동등한 성능을 달성했다.
이는 동일한 연산 예산 내에서 더 높은 효율성을 확보할 수 있음을 의미한다.

### 학습 역학

AttnRes는 PreNorm 희석 문제를 효과적으로 완화한다.
출력 크기가 제한된 범위 내에서 유지되어 무한정 증가하지 않는다.
그래디언트 노름이 레이어 전체에 걸쳐 더 균일하게 분포된다.
이러한 학습 역학의 개선은 더 안정적이고 효율적인 학습을 가능하게 한다.

## 구현

AttnRes의 구현은 einsum 연산을 기반으로 한다.
어텐션과 MLP에 대해 별도의 프로젝션 레이어를 사용한다.
어텐션 계산 전에 RMSNorm을 적용하여 정규화를 수행한다.
Block AttnRes의 경우 기존 Transformer 모델에 최소한의 코드 변경으로 적용할 수 있어 실용성이 높다.

## 결론

Attention Residuals는 Transformer의 핵심 구성 요소인 잔차 연결을 재설계하여 모델 성능을 향상시키는 접근법이다.
특히 Block AttnRes는 실용적인 변형으로, 최소한의 오버헤드로 기존 모델에 적용할 수 있다.
Kimi Linear 48B에서의 실험 결과는 GPQA-Diamond +7.5포인트를 포함하여 여러 벤치마크에서 일관된 성능 향상을 보여준다.
깊은 모델에서 발생하는 레이어 기여도 희석과 은닉 상태 크기 증가 문제를 해결하는 의미 있는 연구이다.

## Reference

- [Attention Residuals - GitHub](https://github.com/MoonshotAI/Attention-Residuals/tree/master/)
