---
layout: post
title: "ELF: 임베딩 공간 Flow Matching 으로 디스크리트 확산 모델을 추월한 연속 언어 확산"
author: 'Juho'
date: 2026-05-28 00:00:00 +0900
categories: [LLM]
tags: [LLM, Benchmark, Embedding]
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
   - [임베딩 공간에서의 Flow Matching](#임베딩-공간에서의-flow-matching)
   - [공유 가중치 디코딩 메커니즘](#공유-가중치-디코딩-메커니즘)
   - [Classifier-Free Guidance 통합](#classifier-free-guidance-통합)
3. [실험 셋업](#실험-셋업)
4. [주요 결과](#주요-결과)
   - [Unconditional Generation](#unconditional-generation)
   - [기계 번역과 요약](#기계-번역과-요약)
   - [Ablation 결과](#ablation-결과)
5. [한계와 주의사항](#한계와-주의사항)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

ELF(Embedded Language Flows)는 Keya Hu, Linlu Qiu, Tianhong Li, Yoon Kim, Jacob Andreas, Kaiming He 등이 제안한 연속(continuous) 확산 언어 모델이다.
이미지·비디오 도메인에서 확산 모델이 표준이 된 것과 달리, 언어 생성에서는 최근 MDLM, Duo 같은 디스크리트(discrete) 확산 모델이 주도해왔다.
ELF는 “언어가 본질적으로 이산적이라서가 아니라, 연속 확산 모델의 설계가 충분히 탐구되지 않았을 뿐”이라는 가설로 출발한다.
핵심 아이디어는 모든 디노이징 단계를 임베딩 공간에서 연속적으로 수행하고, 마지막 한 단계에서만 토큰으로 이산화하는 미니멀 설계다.
그 결과 ELF-B(105M)는 학습 토큰을 기존 방식 대비 10배 적게 쓰고도 generative perplexity 24를 32 스텝 만에 달성했고, WMT14 De-En 번역에서는 26.4 BLEU로 같은 크기의 autoregressive 모델(25.2)과 디스크리트 확산 baseline 을 모두 뛰어넘었다.

## 방법론

### 임베딩 공간에서의 Flow Matching

ELF는 토큰을 사전학습된 T5-small 인코더로 임베딩한 뒤, 512차원을 128차원 보틀넥으로 투영해 작업한다.
인코더는 학습 시에만 사용되고 추론에서는 빠진다.
노이즈가 섞인 잠재 변수는 다음과 같은 선형 보간으로 정의한다.

$$\mathbf{z}_t = t\mathbf{x} + (1-t)\boldsymbol{\epsilon}, \quad t \in [0,1], \quad \boldsymbol{\epsilon} \sim \mathcal{N}(0, \mathbf{I})$$

표준 Flow Matching 은 속도 $\mathbf{v} = \mathbf{x} - \boldsymbol{\epsilon}$ 를 직접 예측하지만, ELF는 깨끗한 임베딩 $\mathbf{x}$ 자체를 예측하는 $\mathbf{x}$-prediction parameterization 을 채택한다.
고차원 임베딩에서 velocity 보다 안정적이며, 학습 목적함수는 다음과 같다.

$$\mathcal{L}_{\text{MSE}} = \mathbb{E}_{t,\mathbf{x},\boldsymbol{\epsilon}} \frac{1}{(1-t)^2} \|\mathbf{x}_\theta(\mathbf{z}_t, t) - \mathbf{x}\|^2$$

기존 임베딩 기반 확산 LM들(Diffusion-LM, CDCD, DiffuSeq 등)은 매 스텝마다 cross-entropy 손실을 끼워넣어 trajectory 를 토큰 공간에 고정시킨다.
ELF는 매 스텝 이산화 손실을 가하지 않고 전 과정에서 연속 표현을 유지한다.

### 공유 가중치 디코딩 메커니즘

ELF는 별도 디코더를 두지 않고, 동일한 네트워크에 “모드 토큰”을 주입해 디노이징과 디코딩을 모두 수행한다.
$t=1$ 에서 토큰 수준 corruption $\tilde{\mathbf{z}}$ 가 주어지면, 네트워크는 임베딩을 출력하고 학습 가능한 unembedding 행렬 $W$ 로 logits 로 사영한다.

$$\mathcal{L}_{\text{CE}} = \mathbb{E}_{\tilde{\mathbf{z}}} [\text{CrossEnt}(W\mathbf{x}_\theta(\tilde{\mathbf{z}}), \mathbf{s})]$$

학습 시에는 배치의 80% 가 디노이징, 20% 가 디코딩 분기에 할당되며 단일 forward pass 로 두 목적함수를 동시에 최적화한다.
추론에서는 Gaussian noise $\mathbf{z}_0$ 부터 ODE 솔버로 반복 디노이징을 수행한 뒤, 마지막 스텝에서 unembedding + argmax 로 토큰을 산출한다.

### Classifier-Free Guidance 통합

이미지 확산에서 검증된 CFG 를 그대로 도입하기 위해, 학습 시 50% 확률로 직전 예측 $\mathbf{x}'_\theta$ 를 self-condition 으로 concatenate 한다.

$$\mathbf{v}_{\text{cfg}}(\mathbf{z}_t|\mathbf{c}) = \omega\mathbf{v}(\mathbf{z}_t|\mathbf{c}) + (1-\omega)\mathbf{v}(\mathbf{z}_t|\varnothing)$$

ELF의 차별점은 training-time CFG 다.
추론 시 두 번의 forward pass 없이 네트워크 자체가 가이드된 속도장을 직접 학습한다.
조건부 생성(번역, 요약)에서는 입력 시퀀스의 깨끗한 임베딩을 prepend 한 뒤 corruption 없이 보존하여, self-attention 만으로 조건화한다.

## 실험 셋업

| 항목 | Unconditional | 기계 번역 | 요약 |
|------|---------------|-----------|------|
| 데이터셋 | OpenWebText 9B tokens | WMT14 De-En 144M | XSum 6M |
| 시퀀스 길이 | 1024 | 표준 | 표준 |
| 평가 지표 | Generative PPL, Unigram Entropy | BLEU | ROUGE-1/2/L |
| 학습 길이 | 5 epoch, 약 95K steps | 100 epoch | 100 epoch |

옵티마이저는 Muon, learning rate 0.002, batch size 512 를 사용했다.
인코더는 T5-small(35M, 512차원) 고정, 보틀넥 차원은 128 이다.
모델 스케일은 ELF-B(105M), ELF-M(342M), ELF-L(652M) 세 가지로 비교했다.

## 주요 결과

### Unconditional Generation

OpenWebText 에서 ELF-B(105M)는 32 스텝 만에 generative perplexity 24를 달성했다.
같은 영역의 baseline 들과 비교하면 다음과 같다.

| Method | Size | Gen. PPL | Steps |
|--------|------|----------|-------|
| ELF-B | 105M | 24 | 32 |
| MDLM | 170M | 약 26 | 64+ |
| Duo | 170M | 약 25 | 100+ |
| FLM | 170M | 약 24.5 | 64 |
| LangFlow | 170M | 약 23 | 64 |

ELF는 45B 토큰으로 학습됐고, 비교 대상 baseline 들은 500B 이상의 학습 토큰을 사용했다.
즉 약 10배 적은 데이터로 동등 이상 품질을 낸다.
추가로 MDLM+SDTT, Duo+DCD, FMLM 같은 distillation 기반 변형까지도 ELF가 distillation 없이 추월했다.

### 기계 번역과 요약

WMT14 De-En 번역 결과는 다음과 같다.

| Model | Size | BLEU |
|-------|------|------|
| Autoregressive | 99M | 25.2 |
| MDLM | 99M | 18.4 |
| Duo | 170M | 21.3 |
| E2D2 | 99M | 24.8 |
| SeqDiffuSeq | 미공개 | 21.3 |
| CDCD | 미공개 | 24.9 |
| ELF-B | 105M | 26.4 |

ELF-B 는 105M 규모로 autoregressive baseline 을 1.2 BLEU 상회하고, 디스크리트 확산 baseline 들을 모두 추월했다.
XSum 요약 결과는 다음과 같다.

| Model | ROUGE-1 | ROUGE-2 | ROUGE-L |
|-------|---------|---------|---------|
| Autoregressive | 30.5 | 10.2 | 24.4 |
| MDLM | 33.4 | 11.6 | 25.8 |
| Duo | 31.4 | 10.1 | 25.0 |
| E2D2 | 28.4 | 8.3 | 22.0 |
| SeqDiffuSeq | 19.3 | 1.7 | 14.1 |
| ELF-B | 36.0 | 12.2 | 27.8 |

ROUGE-1 기준으로 baseline 대비 최소 2.6 포인트 상회하는 최고 성능이다.

### Ablation 결과

CFG 스케일을 키우면 perplexity 가 내려가지만 entropy 도 함께 줄어든다.
즉 품질-다양성 trade-off 가 명확하게 관찰된다.
임베딩 종류별 성능은 contextual pretrained(T5)가 가장 좋고, scratch-trained contextual, non-contextual, frozen Gaussian, learnable 순으로 떨어진다.
샘플러 측면에서는 SDE-inspired sampling 이 ODE 대비 같은 perplexity 를 약 30% 적은 스텝으로 달성했다.
모델 스케일은 ELF-B(105M) -> ELF-M(342M) -> ELF-L(652M) 로 키울 때마다 perplexity-entropy frontier 가 일관되게 개선됐다.
공유 가중치 디코딩 전략이 두 단계 학습(별도 디코더) 대비 저-perplexity 영역까지 더 안정적으로 확장된다는 점도 확인됐다.

## 한계와 주의사항

논문에는 별도의 Limitations 섹션이 있지 않지만 본문 곳곳에서 다음 제약이 드러난다.
첫째, 임베딩 품질이 T5-small 인코더에 의존하므로 사전학습 인코더 선택이 성능을 좌우한다.
둘째, 128차원 보틀넥은 튜닝이 필요하다.
셋째, training-time CFG 와 두 분기(80/20) 계산이 약간의 학습 오버헤드를 더한다.
넷째, generative perplexity 를 GPT-2 Large 로 측정하기 때문에, 평가자 모델의 편향이 결과 해석에 영향을 줄 수 있다.

## 결론

ELF는 “연속 확산 LM 이 디스크리트보다 본질적으로 열등하다”는 최근의 통설을 정면으로 반박한다.
$\mathbf{x}$-prediction, 공유 가중치 디코딩, training-time CFG, SDE-inspired 샘플러라는 4가지 설계 선택만으로, 105M 모델이 500B 토큰 학습 baseline 을 45B 토큰으로 추월했다.
저자들은 이 결과를 “언어가 이산적이라서 연속 확산이 어려운 것이 아니라, 적절한 미니멀 설계가 그동안 부재했을 뿐”이라는 메시지로 정리한다.
이미지 생성에서 표준이 된 기법을 그대로 언어에 옮겨올 수 있는 통로가 열렸다는 점에서, 향후 연속 DLM 연구의 출발점이 될 가능성이 크다.

## Reference

- [ELF: Embedded Language Flows (arXiv:2605.10938)](https://arxiv.org/abs/2605.10938)
