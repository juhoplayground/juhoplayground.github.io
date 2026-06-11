---
layout: post
title: "World Models Meet Language Models: 언제 시뮬레이션을 믿어야 하는가"
author: 'Juho'
date: 2026-06-10 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Benchmark]
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
2. [방법론: PF-OPSD](#방법론-pf-opsd)
3. [주요 결과](#주요-결과)
4. [한계와 주의사항](#한계와-주의사항)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

멀티모달 언어 모델(MLLM)이 정적 이미지 한 장으로부터 미래 결과를 예측할 때, world model을 어떻게 효과적으로 활용할 수 있는가를 다룬 연구다.
미래 지향 시각 추론은 추상적 언어 추론과 구체적 시각 시뮬레이션 사이의 조율을 요구한다.
이 연구의 핵심 통찰은 생성된 롤아웃이 정확한 오라클이 아니라 노이즈가 섞인 추론 흔적이라는 점이다.
생성된 롤아웃은 과제의 핵심 상호작용을 놓치거나 기하학적으로 드리프트할 수 있다.
따라서 world model을 MLLM에 단순히 부착하는 것만으로는 불충분하며, 시뮬레이션을 언제 선택하고 검증하며 신뢰할지를 학습해야 한다.

저자들은 두 가지 실패 모드를 식별한다.
첫째는 Simulation Inertia로, 사용 가능한 world model을 회피하는 경향이다.
둘째는 Forced-Simulation Paradox로, 오도하는 롤아웃을 그대로 수용하는 경향이다.
해법으로 Privileged-Future On-Policy Self-Distillation(PF-OPSD)을 제안한다.
이는 학습 시 ground-truth 미래를 특권적 교사 컨텍스트로 사용해 시각 시뮬레이션을 언제 호출하고 검증하며 신뢰할지를 보정하는 방법이다.

## 방법론: PF-OPSD

PF-OPSD는 world-model 보조 예측을 통제된 구체적 추론으로 본다.
정책은 다섯 가지 결정을 순차적으로 수행한다.
첫째, 시뮬레이션이 도움이 될지를 결정한다(d_sim).
둘째, 필요 시 시뮬레이션 프롬프트를 작성한다(p_sim).
셋째, 생성된 롤아웃을 검증한다(z_ver).
넷째, 롤아웃의 신뢰도를 결정한다(z_rel).
다섯째, 최종 답을 예측한다(y).

학습은 두 단계로 구성된다.
Stage 1은 Protocol SFT로, Gemini-3.1-Pro 워크플로가 ground-truth 미래를 교사 컨텍스트로 관찰하며 구조화된 trajectory를 생성한다.
규칙 검사와 답 일치를 모두 통과한 trajectory만 학습에 사용된다.
Stage 2는 PF-OPSD Self-Distillation으로, 각 예시에서 학생은 특권 접근 없이 trajectory를 생성한다.
특권 평가자(Qwen3.6-27B)가 ground-truth 미래와 답만 관찰해 advantage 가중 타깃을 부여한다.

보상 함수는 다음과 같다.

```
R+(τ_a) = 1[ŷ=y*] - λ_sim·N_sim - λ_FA·1[false accept] - λ_FR·1[false reject]
```

하이퍼파라미터는 λ_sim=0.05, λ_FA=0.50, λ_FR=0.25로 설정된다.
즉 false accept(잘못된 롤아웃 수용)에 가장 큰 페널티 0.50을, false reject에 0.25를, 시뮬레이션 호출 비용에 0.05를 부과한다.
학생은 이산 노드에 KL divergence를, 텍스트 노드에 가중 log-likelihood를 최소화한다.

추론 시 학생은 이미지와 질문, 선택지만 받으며 ground-truth 미래는 받지 못한다.
예시당 최대 3회까지 시뮬레이션을 시도하고, 학습된 검증 결정에 따라 롤아웃을 거부하고 재시도할 수 있다.

평가에는 두 벤치마크가 사용된다.
VRQABench는 학습 4,000개와 테스트 636개로 구성되며 미로, 불규칙 미로, 소코반 퍼즐을 포함한다.
범주는 회전 수 세기, 회전 방향, 밀기 횟수이며 결정론적 솔버가 타깃을 고정한다.
OpenWorldQA는 학습 3,904개와 테스트 500개로 구성되며 결과 직전 앵커 프레임이 있는 실세계 비디오를 다룬다.
순서, 세기, 접촉, 실패, 반사실 등 12개 물리 추론 유형을 5단계 파이프라인과 사람 검토로 구축했다.

## 주요 결과

Table 2는 zero-shot 베이스라인과 학습 방법들의 정확도를 비교한다.

VRQABench와 OpenWorldQA 정확도 비교

| 방법 | VRQABench | OpenWorldQA |
| --- | --- | --- |
| Gemini-3-Flash (zero-shot) | 45.9% | 48.2% |
| Qwen3.5-9B (zero-shot) | 33.2% | 39.8% |
| SFT (학습된 컨트롤러) | 61.8% | 59.6% |
| SFT + GRPO | 63.5% | 61.2% |
| PF-OPSD | 72.4% | 70.5% |

PF-OPSD는 VRQABench에서 72.4%, OpenWorldQA에서 70.5%를 기록했다.
SFT 대비 VRQABench에서 +10.6점, OpenWorldQA에서 +10.9점의 이득을 보였다.

Table 3의 Ablation은 각 구성 요소의 기여를 보여준다.

Ablation 결과

| 제거된 구성 요소 | 성능 변화 |
| --- | --- |
| 시뮬레이션 결정 제거 (VRQABench) | -3.9% |
| 롤아웃 검증 제거 | -7.2% |
| advantage 가중 제거 | -6.0% |
| answer-only distillation | -7.9% |

모델은 예시의 42.5%에서 world model을 호출했으며 예시당 평균 0.45회 호출했다.
롤아웃 검증 분석에서 유용한 롤아웃은 92.5% 수용했고, 손상되거나 오답인 롤아웃은 5.2%만 수용했다.
롤아웃 품질이 저하될 때 성능이 우아하게 저하되었다.
cross-world-model 전이 실험에서 Helios를 Wan 2.2로 교체하면 PF-OPSD와 always simulate가 모두 개선되었으나, 학습된 통제가 강한 생성기를 강제 사용하는 방식을 능가했다.

## 한계와 주의사항

첫째, world model이 최소한 부분적으로 관련된 미래를 생성한다고 가정한다.
정렬이 나쁘면 성능이 저하된다.
둘째, 두 벤치마크 설계와 하나의 world-model 인터페이스로 커버리지가 제한된다.
셋째, 학습 시 특권 미래 신호의 가용성에 의존한다.
넷째, 더 긴 시간 지평과 인터랙티브 환경은 미탐구 상태로 남아 있다.

## 결론

world model의 성공적 통합은 추상 추론과 구체 추론 사이의 학습된 중재를 요구한다.
단순한 도구 부착은 simulation inertia와 forced-simulation paradox로 실패한다.
특권 학습 컨텍스트는 효용 보정을 가능하게 하며, 환각 롤아웃을 거부하도록 학습하면 견고성이 향상된다.
성능 이득은 구조화된 도메인과 자연 도메인 전반에 일반화된다.

## Reference

- [World Models Meet Language Models: On the Complementarity of Concrete and Abstract Reasoning](https://arxiv.org/abs/2606.03603/){:target="_blank"}
