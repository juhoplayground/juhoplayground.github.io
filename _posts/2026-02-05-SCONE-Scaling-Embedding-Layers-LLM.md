---
layout: post
title: SCONE - 언어 모델의 임베딩 레이어 확장 기법
author: 'Juho'
date: 2026-02-05 02:00:00 +0900
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
2. [배경과 선행 연구](#배경과-선행-연구)
   - [어휘 확장의 두 가지 한계](#어휘-확장의-두-가지-한계)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [F-gram 식별 알고리즘](#f-gram-식별-알고리즘)
   - [학습과 추론 분리 설계](#학습과-추론-분리-설계)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [F-gram 길이와 수의 영향](#f-gram-길이와-수의-영향)
   - [A_f-gram 모델 크기 스케일링](#a_f-gram-모델-크기-스케일링)
   - [대규모 사전학습](#대규모-사전학습)
   - [추론 효율과 저장 비용](#추론-효율과-저장-비용)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"Scaling Embedding Layers in Language Models"는 입력 임베딩 레이어를 확장해 모델 성능을 끌어올리되 추론 시 FLOPs는 그대로 유지하는 SCONE(Scalable, Contextualized, Offloaded, N-gram Embedding) 기법을 제안한 Google 연구팀의 논문이다.
저자는 Da Yu, Edith Cohen, Badih Ghazi, Yangsibo Huang, Pritish Kamath, Ravi Kumar, Daogao Liu, Chiyuan Zhang(Google)이다.

논문 abstract의 핵심은 다음과 같다.
SCONE은 원본 어휘를 유지한 채 자주 등장하는 n-gram(f-gram)에 대한 추가 임베딩을 도입한다.
이 임베딩은 학습 시점에는 별도 트랜스포머 모델로 contextualize되어 계산되고, 추론 시점에는 사전 계산되어 accelerator 외부 메모리에 캐싱된다.
1B 파라미터 모델에 SCONE을 적용하면 추론 시 FLOPs를 약 두 배 사용하는 baseline을 능가한다.

## 배경과 선행 연구

### 어휘 확장의 두 가지 한계

저자들은 단순한 어휘 확장이 직면하는 두 가지 핵심 문제를 지적한다.
첫째, 입력과 출력 임베딩이 가중치를 공유하는 일반적인 설정에서는 어휘를 키워도 logits 계산이 accelerator 메모리를 동일하게 차지하므로 확장 이득이 사라진다.
둘째, 어휘를 키우면 tail token이 폭증해 학습이 충분히 이루어지지 않는다.

GPT-2에 대한 분석에서 어휘 32K에서는 토큰의 97.6%가 100M 학습 토큰 동안 100회 이상 업데이트되지만, 어휘 2M에서는 그 비율이 7.3%로 떨어진다.
SCONE은 어휘 자체를 키우는 대신 토큰을 n-gram contextualized 변형으로 보강하는 방식으로 이 두 문제를 회피한다.

### Related Work 정리

| 카테고리 | 주요 흐름 | 본 연구와의 차이 |
|----------|-----------|------------------|
| Contextualized embedding | 전체 시퀀스 또는 짧은 n-gram을 downstream에 통합 | 본 연구는 사전 계산 + offload로 추론 비용 분리 |
| Vocabulary scaling | Tao 2024 — 70B 모델 최적 어휘 216K | 어휘 키우기 한계를 우회 |
| Concurrent work | Huang 2025 — 학습 중 추가 임베딩 instantiate | accelerator 메모리 부담 — 본 연구는 사전 계산으로 회피 |

## 방법론

### F-gram 식별 알고리즘

f-gram 후보는 학습 코퍼스를 (n−1)회 linear scan하는 BPE 스타일 알고리즘으로 식별한다.
모든 k-gram 빈도를 세고 최소 빈도 5 임계로 필터링한 뒤 빈도 상위 S개를 선택한다.
저자들은 최대 길이 n=5를 채택했다(2~8 실험 결과 n=4 부근에서 perplexity가 정체).

각 f-gram은 학습 시점에 추가 트랜스포머 모델 A_f-gram을 통과해 contextualized embedding으로 변환된다.
추론 시점에는 lookup table F: V_f-gram → R^d로 사전 계산되어 hash 또는 tree 구조로 저장된다.

### 학습과 추론 분리 설계

학습 단계에서 직접 큰 임베딩 테이블에 backprop하지 않고 트랜스포머 A_f-gram을 통과시키는 이유는 sparse update 문제 회피다.
n-gram 간 dependency가 있어 트랜스포머가 공유된 표현을 학습하므로, 큰 어휘 임베딩 테이블에서 발생하는 업데이트 희소성이 사라진다.

추론 단계에서는 트랜스포머가 사라지고 사전 계산된 lookup만 남기 때문에 accelerator FLOPs가 늘어나지 않는다.
캐시는 CPU 메모리 또는 NVMe SSD 같은 secondary storage에서 제공된다.

## 실험 셋업

| 항목 | 값 |
|------|-----|
| Main 모델 변형 | 0.7B, 1.0B, 1.3B, 1.9B 파라미터 |
| Context length | 2048 |
| Embedding 차원 | 2048 |
| 학습 토큰 | 200B |
| 토크나이저 | OLMo |
| 평가 corpus | OLMo 11-corpus mixture (c4-en, books, common-crawl 등) |
| F-gram 최대 길이 n | 5 |
| F-gram 수 | 512K ~ 1B |
| A_f-gram 크기 | 0.5x ~ 3x non-embedding 파라미터 |
| 추론 환경 | vLLM, A100, batch 1 / 16 |

## 주요 결과

### F-gram 길이와 수의 영향

n=2~8 스윕에서 perplexity 향상은 n=4 부근에서 정체되며 평균 매칭 길이도 안정화된다.
저자는 n=5를 본 실험의 기본값으로 선택했다.

f-gram 수를 512K에서 100M으로 늘리면 perplexity가 단조 감소한다.
100M f-gram을 추가한 419M baseline은 759M, 1099M baseline에 도달하거나 능가한다.

### A_f-gram 모델 크기 스케일링

419M main 모델에 170M A_f-gram을 결합하면 perplexity가 26.1에서 23.4로 감소해 589M baseline(24.7)을 능가한다.
A_f-gram 크기를 0.5x에서 3x까지 늘릴수록 perplexity가 단조 개선된다.

### 대규모 사전학습

| 모델 | 파라미터 | 평균 perplexity | 해석 |
|------|-----------|------------------|------|
| Baseline 1B | 1.0B | 16.082 | 기준 |
| 1B + 10M f-gram + 0.6B A | 1.0B | 15.459 | f-gram 추가 |
| 1B + 10M f-gram + 1.8B A | 1.0B | 15.125 | 1.3B baseline(15.284)보다 우수 |
| 1B + 1B f-gram + 1.8B A | 1.0B | 14.581 | 1.9B baseline(14.598)을 절반 FLOPs로 능가 |
| Baseline 1.3B | 1.3B | 15.284 | 비교군 |
| Baseline 1.9B | 1.9B | 14.598 | 비교군 |

가장 인상적인 결과는 1B 모델에 1B f-gram + 1.8B A_f-gram을 결합해 1.9B baseline을 추론 FLOPs 절반으로 능가한 사례다.

### 추론 효율과 저장 비용

| F-gram 수 | 시스템 메모리 | SSD |
|------------|---------------|-----|
| 10^7 | 41.4 GB | 77.3 GB |
| 10^8 | 413.6 GB | 766.8 GB |
| 10^9 | CPU 메모리 초과 | 7665.4 GB |

vLLM A100 batch 1 환경에서 10M f-gram은 무시 가능한 추가 지연만 유발한다.
NVMe에 1B f-gram을 저장한 경우에도 토큰당 2.3ms로, 상용 API의 약 10ms 임계 아래다.
batch 16에서 in-memory 접근 시 토큰당 0.017ms로 amortized된다.

저장 비용은 시스템 메모리 약 $2/GB와 SSD 약 $0.1/GB의 차이로 SSD를 활용한 대규모 저장이 경제적으로 가능하다.

## 한계와 디스커션

저자가 명시적·암시적으로 다룬 한계는 다음과 같다.
첫째, A_f-gram이 추가 학습 비용을 요구한다(추론 비용은 0이지만 학습 시 트랜스포머 한 번 더).
둘째, f-gram 식별이 BPE 스타일이라 도메인 변화나 다국어 환경에서는 별도 갱신이 필요하다.
셋째, 저장 인프라(시스템 메모리, NVMe) 가정에 의존한다.
넷째, contextualized embedding이라는 표현 형태가 모든 downstream 태스크에 동일한 이득을 주는지는 추가 검증이 필요하다.

디스커션에서 저자는 두 가지 시사점을 제시한다.
첫째, 임베딩 레이어 확장은 추론 FLOPs와 분리해서 설계할 때 가장 큰 가치를 갖는다.
둘째, 학습 시점의 contextualization과 추론 시점의 lookup 분리는 conditional computation(MoE)이 아닌 별도 sparsity 축이며, 두 축은 결합 가능하다.

## 결론

SCONE은 어휘 확장이 직면하는 input/output 결합 한계와 tail token 한계를 토큰 단위가 아닌 f-gram 단위로 우회했다.
1B 모델이 1B f-gram + 1.8B A_f-gram 결합으로 1.9B baseline을 추론 FLOPs 절반으로 능가한다는 결과는 임베딩 확장의 실질적 가치를 입증한다.
NVMe 저장에서도 토큰당 2.3ms 수준의 지연으로 상용 환경에 통합 가능하다는 실용성과 단순한 lookup 구조는 도입 장벽이 낮다는 점에서 주목할 만한 연구다.

## Reference

- [Scaling Embedding Layers in Language Models (arXiv:2502.01637)](https://arxiv.org/abs/2502.01637/)
