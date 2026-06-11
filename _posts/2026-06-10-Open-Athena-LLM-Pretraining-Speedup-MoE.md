---
layout: post
title: "Open Athena: LLM 사전학습을 3.6배 빠르게 만든 스택"
author: 'Juho'
date: 2026-06-10 00:00:00 +0900
categories: [LLM]
tags: [LLM, GPU, AI]
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
3. [주요 결과](#주요-결과)
4. [한계와 주의사항](#한계와-주의사항)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Open Athena는 LLM 사전학습 효율을 아키텍처 재설계와 표적 최적화 기법의 조합으로 크게 개선한 작업이다.
핵심은 dense 모델에서 mixture-of-experts(MoE)로의 전환과, 그 위에 쌓은 일련의 후속 개선이다.

가장 두드러진 결과는 dense 베이스라인을 자체 Marin MoE V1로 전환했을 때 1e23 FLOPs 규모에서 6.7배의 이론 속도향상을 보였고, 실제로 3.6배의 실현 속도향상을 달성했다는 점이다.
이 결과는 활성 파라미터 129B 규모 모델을 1조(1T) 토큰으로 학습해 검증했다.

또한 MoE V1 위에 네 가지 후속 개선을 누적해, 더 낮은 컴퓨트 규모(3e19 FLOPs)에서 추가로 2.1배의 이론 속도향상을 얻었다.
즉 아키텍처 전환과 표적 개선이 복리적으로 작동해 효율 이득이 누적되는 구조다.

## 방법론

실험 설계는 여섯 개 컴퓨트 예산(1e18 ~ 3e20 FLOPs)에서 isoFLOP 스윕을 수행해 컴퓨트 최적 구성을 식별하는 방식이다.
사전 등록(pre-registered)한 예측을 실제 결과와 대조했으며, 1e23 FLOPs 규모에서 0.8% 이내의 정확도를 보였다.
개선은 네 개의 컴퓨트 규모에서 테스트해, 대규모로 갈수록 이득이 사라지지 않음을 확인했다.

학습률은 다음 공식으로 모델링했다.

`lr = 1.680 · tokens^-0.282 · dim^-0.372 · bs^0.5`

이 공식은 19개 테스트 조건에서 R squared = 0.994의 적합도를 보였다.

검증 지표는 다음과 같다.

손실 예측 정확도는 1e23 FLOPs 사전 등록 예측이 2.252였고 실제 값은 2.234로, 0.8% 편차에 그쳤다.
손실 궤적은 전체 1T 토큰 학습에서 매끄러운 선형 진행을 보였다.
평가는 코드, wikitext, 일반 웹을 포함한 16개 범주의 Paloma 벤치마크로 수행했다.

## 주요 결과

핵심 개선과 각각의 정량 이득은 아래와 같다.

| 개선 | 이론 속도향상 | 실현 속도향상 | 핵심 기법 |
| --- | --- | --- | --- |
| Dense에서 MoE 전환 | 6.7배 | 3.6배 | 64개 라우팅 전문가의 quantile balancing, attention gate, gated norm, exclusive self-attention |
| Expert Sparsity 강화 | 1.4배 | 1.3배 | 라우팅 전문가 64에서 256으로 확대, 토큰당 연산 증가 없음 |
| Optimizer 교체 | 1.3배 | 1.25배 | AdamH에서 MuonH로 전환 |
| Partial Key Offset | 1.2배 | 1.2배 | 파라미터 0인 attention 전처리 |
| Routed Expert Normalization | 1.04배 | 1.04배 | 전문가 출력 재정규화 |

Dense에서 MoE로의 전환에서는 64개 라우팅 전문가(토큰당 4개 활성)의 quantile balancing, attention gate와 gated norm, exclusive self-attention을 도입했다.

Expert Sparsity 강화는 라우팅 전문가를 64에서 256으로 늘려 토큰당 연산을 증가시키지 않으면서 용량을 확대했다.
이로써 1.4배 이론 속도향상(1.3배 실현)을 얻었다.

Optimizer는 AdamH에서 MuonH로 교체해 1.3배 이론 속도향상(1.25배 실현)을 달성했다.
Muon은 행렬별로 그래디언트를 직교화해 출력 변화를 제한하며, 레이어 갱신 간 분포 시프트에 대응한다.

Partial Key Offset(PKO)은 파라미터가 0인 attention 전처리 기법으로, 네 개 컴퓨트 규모 전반에서 20%의 이득을 보였다(1.2배 이론, 1.2배 실현).

Routed Expert Normalization은 커뮤니티 기여로 추가된 개선으로, 스케일링 팩터 X = 2.5로 전문가 출력을 재정규화해 1.04배의 이득(1.04배 실현)을 얻었다.

## 한계와 주의사항

향후 개선 여지는 여러 방향으로 남아 있다.

residual bottleneck 개선이 한 축이다.
attention residual과 skip connection을 다루면 15% 이상의 추가 속도향상이 가능하다고 본다.

추론 효율도 과제다.
현재 4:1 Grouped Query Attention의 대안을 탐색 중이며, multihead latent attention, gated delta network, multi-token prediction 등이 후보로 제시된다.

인프라 측면에서는 Google TPU Research Cloud의 지원으로 대규모 검증을 수행했다.
이런 대규모 검증 환경에 대한 의존성은 결과 재현 시 고려해야 할 지점이다.

## 결론

Open Athena는 아키텍처 재설계(dense에서 MoE)와 표적 최적화 개선의 결합이 복리적 효율 이득을 만들어낸다는 점을 보였다.
1e23 FLOPs 규모에서 6.7배 이론 속도향상과 3.6배 실현 속도향상에 도달했고, 사전 등록 예측은 실제 결과와 0.8% 이내로 일치했다.
이 모든 과정에서 예측 가능하고 안정적인 학습 동역학을 유지했다는 점이 핵심이다.

## Reference

- [Open Athena: LLM Pretraining Speedup](https://openathena.ai/blog/pretraining-speedup/)
