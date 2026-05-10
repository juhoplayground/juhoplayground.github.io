---
layout: post
title: "Deep Think with Confidence - LLM 추론의 신뢰도 평가 연구"
author: 'Juho'
date: 2026-03-14 00:00:00 +0900
categories: [LLM]
tags: [LLM, AI, Benchmark]
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
   - [Test-time scaling과 majority voting의 한계](#test-time-scaling과-majority-voting의-한계)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [Confidence 측정 기법 5종](#confidence-측정-기법-5종)
   - [Offline DeepConf](#offline-deepconf)
   - [Online DeepConf](#online-deepconf)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [Offline 정확도](#offline-정확도)
   - [GPT-OSS-120B AIME25 99.9%](#gpt-oss-120b-aime25-999)
   - [Online 토큰 절감과 정확도](#online-토큰-절감과-정확도)
   - [Confidence 측정 기법 비교](#confidence-측정-기법-비교)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"Deep Think with Confidence(DeepConf)"는 Meta AI와 UCSD 공동 연구로, LLM 추론 trace의 내부 confidence 신호를 활용해 저품질 trace를 필터링하고 동시에 토큰 사용량을 크게 줄이는 test-time 기법을 제안한다.
저자는 Yichao Fu(UCSD 인턴), Xuewei Wang, Yuandong Tian, Jiawei Zhao이며, 2025년 8월 21일 arXiv에 공개되었다.

논문 abstract의 핵심은 다음과 같다.
LLM은 self-consistency와 majority voting 같은 test-time scaling 기법으로 추론 정확도를 높일 수 있지만, 정확도 수익이 점점 줄어들고 연산 비용이 선형적으로 증가하는 한계가 있다.
DeepConf는 추가 학습이나 하이퍼파라미터 조정 없이 모델 내부 confidence 신호를 사용해 저품질 추론 trace를 생성 도중 또는 생성 후에 동적으로 필터링한다.
GPT-OSS-120B로 AIME 2025에서 DeepConf@512는 최대 99.9% 정확도에 도달했고, 동일 정확도 기준 생성 토큰 수를 최대 84.7%까지 줄였다.

## 배경과 선행 연구

### Test-time scaling과 majority voting의 한계

LLM 추론 능력 향상은 두 축에서 진행되어 왔다.
첫째, CoT(Chain-of-Thought)의 깊이를 늘리는 단일 trace 길이 확장.
둘째, self-consistency처럼 다수의 trace를 병렬 생성하고 majority voting으로 답을 집계하는 병렬 확장.
저자들은 두 번째 축의 비용 문제를 정량화한다.
Qwen3-8B로 AIME25 pass@1을 68%에서 82%로 끌어올리려면 trace 511개를 추가로 생성해야 하며 토큰 1억 개가 추가로 소비된다.

또한 majority voting은 모든 trace의 표를 동등하게 다루므로 저품질 trace가 다수일 때 결과가 흔들린다.
trace 품질을 측정하는 신호로 token entropy나 average confidence가 제안되었지만, 두 가지 한계가 있다.
첫째, 전역 평균은 trace 중간의 결정적 추론 붕괴(예: "wait", "however", "think again" 같은 토큰에서 발생하는 confidence 급락)를 가린다.
둘째, 전역 측정은 trace가 완전히 끝난 뒤에야 계산되어 조기 종료가 불가능하다.

### Related Work 정리

| 카테고리 | 주요 흐름 | 본 연구와의 차이 |
|----------|-----------|------------------|
| CoT 깊이 확장 | o1 (Jaech 2024), DeepSeek-R1 (Guo 2025), Kimi K1.5, Qwen3, Grok-4 | 단일 trace 길이 — DeepConf는 병렬 trace 품질 필터링 |
| 병렬 확장 | Self-Consistency (Wang 2023), Best-of-N (Brown 2024), REBASE | 균등 voting — DeepConf는 confidence 가중 |
| 효율 voting | ESC, RASC, Adaptive-Consistency, Dynamic Voting, Dynasor | 표본 수만 줄임 — DeepConf는 trace 내부 토큰까지 절감 |
| Confidence 추정 | Kang 2025 (self-certainty), Fadeeva 2024 (token entropy), Chuang 2024 (confidence tokens) | 전역 평균 — DeepConf는 local segment 기반 |

DeepConf의 차별점은 local confidence 신호(group, bottom-10%, lowest, tail)를 활용해 trace 중간의 추론 붕괴를 정확히 잡아낸다는 점, 그리고 그 신호를 online 생성 단계에서 조기 종료 트리거로 사용한다는 점이다.

## 방법론

### Confidence 측정 기법 5종

토큰 confidence는 위치 i에서 top-k 로그확률의 음의 평균으로 정의된다.

```
C_i = -(1/k) Σ_{j=1..k} log P_i(j)
```

여기서 k는 상위 토큰 수, P_i(j)는 j-번째 어휘 토큰 확률이다.
높은 C_i는 분포가 뾰족함(높은 확신)을, 낮은 C_i는 불확실함을 의미한다.

| 측정 기법 | 정의 | 특성 |
|-----------|------|------|
| Average Trace Confidence | 전체 trace 토큰 평균 | 전역 신호, 중간 붕괴 가림 |
| Group Confidence | 슬라이딩 윈도우(n=1024 또는 2048) 내 평균 | local 신호 |
| Bottom-10% Group Confidence | 가장 낮은 10% 그룹들의 평균 | 결정적 붕괴 강조 |
| Lowest Group Confidence | 모든 그룹 중 최솟값 | 극단적 local 신호, online 종료 트리거 |
| Tail Confidence | 마지막 2048 토큰 평균 | 결론부 품질 강조 |

저자는 HMMT25 30문제 × 4096 trace 데이터로 정답/오답 trace의 confidence 분포를 비교한 결과, bottom-10%와 tail 측정이 평균 confidence보다 정답·오답을 더 명확히 분리함을 보였다.

### Offline DeepConf

offline 모드는 모든 trace 생성이 완료된 뒤 confidence 가중 voting과 필터링을 적용한다.

```
V(a) = Σ_{t∈T} C_t · I(answer(t) = a)
```

여기서 C_t는 위 5종 중 하나의 trace-level confidence다.
필터링은 retention ratio η로 표현되며 η=10%는 상위 10% trace만, η=90%는 상위 90% trace를 voting에 포함한다.

### Online DeepConf

online 모드는 두 알고리즘을 제공한다.
DeepConf-low(η=10%)와 DeepConf-high(η=90%)이며 둘 다 두 단계로 구성된다.

첫째, offline warmup으로 N_init=16개의 완전한 trace를 생성해 stopping threshold s를 설정한다.

```
s = Percentile_{100−η}({C_t : t ∈ T_warmup})
```

DeepConf-low는 90 percentile, DeepConf-high는 10 percentile에 해당하며, threshold는 lowest group confidence 기준으로 잡힌다.

둘째, adaptive sampling으로 trace 생성 도중 group confidence가 s 아래로 떨어지면 즉시 종료하고, 합의 비율 β = V(â) / Σ V(a)가 consensus threshold τ=0.95를 넘으면 추가 sampling을 멈춘다.

## 실험 셋업

| 항목 | 값 |
|------|-----|
| 평가 모델 | DeepSeek-8B, Qwen3-8B, Qwen3-32B, GPT-OSS-20B, GPT-OSS-120B |
| 벤치마크 | AIME24, AIME25, BRUMO25, HMMT25, GPQA-Diamond |
| 사전 생성 trace pool | 문제당 4096개 |
| 기본 voting 크기 | K=512 |
| Group window | 2048 토큰 |
| Online warmup | N_init=16 trace |
| Consensus threshold | τ=0.95 |
| 반복 실험 | 설정당 64회 평균 |
| Baseline | self-consistency + majority voting (cons@K) |

DeepSeek-8B는 DeepSeek-R1 (0528) 모델에서 distillation된 Qwen3-8B 기반 모델이다.
GPT-OSS-120B는 OpenAI가 공개한 오픈 가중치 reasoning 모델이며, 본 논문은 no-tools 설정으로 평가한다.

## 주요 결과

### Offline 정확도

K=512 voting에서 confidence 가중과 필터링은 cons@512 대비 일관된 향상을 보였다.

| 모델 | 데이터셋 | Pass@1 | Cons@512 | Mean@512 | Bottom-10@10% | Tail@10% |
|------|----------|--------|----------|----------|---------------|----------|
| DeepSeek-8B | AIME24 | 83.0 | 86.7 | 86.7 | 93.3 | 93.3 |
| DeepSeek-8B | AIME25 | 76.9 | 82.3 | 82.3 | 87.5 | 87.4 |
| DeepSeek-8B | HMMT25 | 58.1 | 69.6 | 69.9 | 79.5 | 83.9 |
| Qwen3-32B | AIME24 | 80.6 | 85.3 | 85.7 | 90.8 | 89.4 |
| GPT-OSS-120B | AIME24 | 91.9 | 96.7 | 96.7 | 96.5 | 97.4 |
| GPT-OSS-120B | AIME25 | 91.8 | 97.0 | 97.1 | 98.1 | 99.9 |

η=10% 공격적 필터링은 평균 +1.22%p의 정확도 향상을 가져오고, 일부 설정에서는 +9.38%p까지 상승했다.
반면 η=90% 보수적 필터링은 cons@512 대비 −0.21%에서 +0.73% 사이로 안정적으로 동작한다.

### GPT-OSS-120B AIME25 99.9%

가장 인상적인 수치는 GPT-OSS-120B에서 AIME25 정확도 99.9%를 달성한 점이다.
Tail Confidence(마지막 2048 토큰)를 측정하고 상위 10% trace만 보존한 Tail(2k)@10% 설정에서 도달한 수치로, cons@512(97.0%)와 pass@1(91.8%)을 큰 폭으로 능가한다.
이는 vehicle 벤치마크 한 곳에서 사실상 saturate에 도달한 사례다.

### Online 토큰 절감과 정확도

K=512 budget의 online DeepConf는 cons@512 대비 토큰 사용을 큰 폭으로 줄이면서 정확도를 유지한다.

| 모델 | 데이터셋 | Cons@512 토큰 / 정확도 | DeepConf-high 토큰 변화 / 정확도 | DeepConf-low 토큰 변화 / 정확도 |
|------|----------|------------------------|------------------------------------|----------------------------------|
| DeepSeek-8B | AIME24 | 3.55×10^8 / 86.7% | −59.0% / 86.7% | −77.9% / 92.5% |
| DeepSeek-8B | AIME25 | 4.01×10^8 / 82.3% | −40.9% / 81.4% | −69.0% / 86.4% |
| DeepSeek-8B | HMMT25 | 4.49×10^8 / 69.8% | −23.5% / 70.0% | −64.4% / 77.6% |
| Qwen3-32B | AIME24 | 2.00×10^8 / 84.8% | −56.0% / 86.4% | −66.8% / 89.5% |
| GPT-OSS-120B | AIME25 | 3.23×10^8 / 97.1% | −56.0% / 97.0% | −84.7% / 97.9% |

DeepSeek-8B 평균 토큰 절감은 DeepConf-low 62.88%, DeepConf-high 47.67%로 동일 정확도 대비 확실한 비용 우위를 보인다.
GPT-OSS-120B AIME25에서 −84.7% 토큰 절감과 동시에 정확도 +0.8%p 향상을 달성한 사례가 가장 극단적이다.

### Confidence 측정 기법 비교

23개 모델·데이터셋 조합 평균을 비교한 ablation 결과는 다음과 같다.

| 측정 기법 | 평균 정확도 |
|-----------|-------------|
| Majority Voting (baseline) | 83.0 |
| Mean Confidence@10% | 83.9 |
| Bottom-10%@10% | 84.0 |
| Lowest Group Confidence (2k)@10% | 84.4 |
| Tail (2k)@10% | 84.5 |
| Tail (10%)@10% | 84.6 |

Tail과 lowest group 같은 local 신호가 mean confidence보다 일관되게 더 효과적이며, head(첫 10% 토큰) 기반 confidence는 majority voting과 거의 차이가 없거나 오히려 떨어진다.
이는 추론의 초반은 setup과 paraphrase가 지배해 변별력이 낮다는 점을 보여준다.

## 한계와 디스커션

저자가 명시한 한계는 두 가지로 요약된다.
첫째, 일부 설정에서 모델이 잘못된 답에 자신감을 집중시키는 confidently wrong 현상이 발생한다.
GPT-OSS-120B HMMT25에서 η=10% 필터링이 정확도를 떨어뜨린 사례가 대표적이다.
이런 경우 보수적 η=90% 설정이 더 안전하다.

둘째, 적정 retention ratio η가 데이터셋과 모델에 따라 다르다.
top-25% 또는 top-50%가 가장 좋은 모델·데이터셋 조합도 존재하므로, η는 사전 검증이 필요한 하이퍼파라미터로 봐야 한다.

future work 섹션에서 저자는 두 방향을 제시한다.
첫째, RL 학습에 confidence 기반 조기 종료를 통합하여 정책 탐험과 sample 효율성을 개선하는 방향.
둘째, 더 견고한 confidence calibration과 uncertainty quantification 기법으로 confidently wrong 사례를 줄이는 방향이다.

ablation은 consensus threshold τ에서 τ=0.95가 최적임을 보였다(정확도 손실 없이 토큰 절반 이상 절감), warmup 크기 N_init=16이 정확도 안정성과 비용의 균형점이며, retention η는 데이터셋·모델별로 10~50% 사이에서 최적값이 갈린다.

## 결론

DeepConf는 self-consistency의 두 가지 약점, 즉 모든 trace의 균등 가중과 전역 confidence의 중간 붕괴 마스킹을 동시에 해결한다.
local confidence(group, bottom-10%, lowest, tail) 다섯 가지를 비교하고 가장 효과적인 조합을 찾아낸 ablation을 통해, GPT-OSS-120B AIME25 99.9%, DeepSeek-8B AIME25 정확도 +5.1%p, 토큰 절감 최대 84.7%라는 정량 결과를 보였다.
추가 학습 없이 추론 시점에 적용 가능하고 5개 모델 × 5개 벤치마크에 걸쳐 일관된 효과를 입증해, test-time compression의 실용 도구로 자리잡을 가능성이 높다.

## Reference

- [Deep Think with Confidence (arXiv:2508.15260)](https://arxiv.org/abs/2508.15260/)
- [Project Page - DeepConf](https://jiaweizzhao.github.io/deepconf/)
