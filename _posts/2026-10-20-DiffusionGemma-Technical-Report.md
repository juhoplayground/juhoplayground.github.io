---
layout: post
title: "DiffusionGemma Technical Report 정리 - Gemma 4를 256 토큰 블록 텍스트 디퓨전 모델로 전환하기"
author: 'Juho'
date: 2026-10-20 00:00:00 +0900
categories: [LLM]
tags: [LLM, AI, Benchmark, Evaluation, GPU, vLLM]
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
2. [배경: 자기회귀 디코딩의 병목과 이산 디퓨전](#배경-자기회귀-디코딩의-병목과-이산-디퓨전)
3. [방법론](#방법론)
   - [이산 디퓨전 정식화](#이산-디퓨전-정식화)
   - [아키텍처와 블록 자기회귀 생성](#아키텍처와-블록-자기회귀-생성)
   - [DiffusionGemma 샘플러](#diffusiongemma-샘플러)
   - [추론 효율 지표](#추론-효율-지표)
   - [1단계: Supervised Finetuning](#1단계-supervised-finetuning)
   - [2단계: Sampler Distillation과 강화학습(SD²RL)](#2단계-sampler-distillation과-강화학습sdrl)
   - [추론 최적화](#추론-최적화)
4. [주요 결과](#주요-결과)
   - [실험 셋업](#실험-셋업)
   - [품질 벤치마크](#품질-벤치마크)
   - [속도와 디노이징 효율](#속도와-디노이징-효율)
   - [다운스트림 파인튜닝](#다운스트림-파인튜닝)
   - [텍스트 디퓨전의 실용적 이점](#텍스트-디퓨전의-실용적-이점)
5. [프롬프트 포맷](#프롬프트-포맷)
6. [한계와 주의사항](#한계와-주의사항)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

DiffusionGemma는 Google DeepMind가 공개한 실험적 오픈 웨이트 언어 모델로, 이산 디퓨전(discrete diffusion)을 사용해 텍스트를 매우 빠르게 생성합니다.
토큰을 하나씩 디코딩하는 대신 256개 토큰 블록을 병렬로 반복 정제하여 기존 자기회귀(AR) LLM의 순차 디코딩 병목을 피합니다.
처음부터 학습하지 않고, 활성 파라미터 3.8B, 전체 파라미터 25.2B인 Mixture-of-Experts 모델 Gemma 4 26B A4B를 파인튜닝해 얻었습니다.
2단계 학습 파이프라인은 출발점이 된 AR 모델 전체 학습 토큰 예산의 10% 미만만 사용합니다.

리포트가 제시하는 핵심 수치는 다음과 같습니다.

| 항목 | 내용 |
|------|------|
| 기반 모델 | Gemma 4 26B A4B (MoE) |
| 캔버스(블록) 크기 | 256 토큰 |
| 평균 Tokens Per Forward | 약 20 (평가 7개 벤치마크 평균 19.74) |
| 출력 속도 | 단일 H100에서 약 1,500 tokens/s |
| 캔버스당 평균 디노이징 스텝 | 약 12 (최대 48) |
| 학습 비용 | AR 모델 전체 학습 토큰의 10% 미만 |
| 라이선스 | Apache 2.0 |

DiffusionGemma는 원본 모델의 thinking 모드, 멀티모달 입력, 긴 컨텍스트 지원을 그대로 유지합니다.
디퓨전 파인튜닝 이후에도 AR 방식 생성이 가능하며 성능 저하는 크지 않아, 디퓨전과 AR을 혼합한 하이브리드 디코딩의 가능성도 제시합니다.
리포트는 이 모델이 생성 속도와 모델 성능 사이의 트레이드오프에서 새로운 Pareto frontier를 형성한다고 주장합니다.

## 배경: 자기회귀 디코딩의 병목과 이산 디퓨전

### AR 디코딩의 메모리 병목

AR 모델의 왼쪽에서 오른쪽으로 토큰 하나씩 생성하는 방식은 메모리 병목을 만듭니다.
많은 요청을 동시에 처리할 때는 배칭으로 처리량을 확보할 수 있지만, 단일 또는 저동시성 요청은 본질적으로 memory-bound입니다.
모델 가중치와 KV cache를 메모리에서 가속기로 옮기는 시간이 실제 연산 시간보다 훨씬 길어서, 연산 유닛이 놀고 사용자당 생성 속도가 제한됩니다.

Speculative decoding은 작은 drafter 모델이 보통 8개 토큰 정도의 후보 시퀀스를 만들고 LLM이 이를 검증하는 방식으로 활용도를 높입니다.
draft 길이 8 기준으로 forward pass당 3~6개 토큰(TPF)을 얻을 수 있습니다.
하지만 AR drafter는 여전히 순차 생성에 묶이고, 병렬 drafter는 뒤쪽 draft 위치로 갈수록 acceptance rate가 떨어지는 한계가 있습니다.

텍스트 디퓨전은 토큰 블록 전체를 동시에 예측하여 실행을 memory-bound 영역에서 compute-bound 영역으로 옮깁니다.
리포트는 Gemini Diffusion, Mercury처럼 독점 API에 묶인 모델과, 추론 능력·멀티모달 이해가 부족하거나 디퓨전의 지연 시간 이점을 제대로 살리지 못하는 기존 오픈 웨이트 모델 사이의 공백을 지적합니다.
DiffusionGemma는 높은 지능, 매우 빠른 속도, 공개 접근성을 동시에 갖춘 텍스트 디퓨전 모델을 목표로 합니다.

### 연속 디퓨전 대신 이산 디퓨전을 택한 이유

초기 텍스트 디퓨전은 토큰을 연속 임베딩 공간에 매핑하고 Gaussian 디퓨전을 적용했습니다.
Diffusion-LM, SED처럼 연속 벡터를 토큰으로 강제 반올림(rounding)하는 방식이나, CDCD처럼 연속 과정을 유지하면서 categorical logit을 직접 예측하는 방식이 있었습니다.
리포트는 이러한 방식이 하드 반올림으로 디퓨전 likelihood bound의 정확성을 깨뜨리고, 유효한 어휘 토큰이 고차원 공간에서 차지하는 비율이 극히 작아 역과정이 의미 없는 영역으로 표류하기 쉽다고 설명합니다.

이산 디퓨전은 BERT의 마스킹·랜덤 토큰 교체, ELECTRA의 대체 토큰 아이디어를 다단계 Markov 과정으로 일반화한 계보입니다.
연속 벡터 대신 이산 상태에서 직접 동작하므로 임베딩 투영 불일치가 없고 categorical 전이에 대한 이론적 근거가 더 탄탄합니다.
DiffusionGemma는 이 이산 디퓨전 계보 위에 구축되었습니다.

부록의 관련 연구에서는 Embedded Language Flows, discrete flow map, Cosmos·TEncDM 같은 잠재 공간 방식, hyperspherical flow, CANDI 등 연속·하이브리드 접근의 최근 성과도 정리합니다.
리포트는 연속 텍스트 디퓨전의 초기 실패가 모달리티 자체의 한계라기보다 최적이 아닌 공간 기하와 학습 목표 때문이었을 수 있다고 평가합니다.
또한 TiDAR, DFlash, DSpark 같은 디퓨전 기반 drafter와 달리 DiffusionGemma는 독립형 디퓨전 모델로 동작해 검증 병목이 없고 더 긴 캔버스로 확장된다고 구분합니다.

## 방법론

### 이산 디퓨전 정식화

리포트는 continuous-time Markov chain(CTMC) 기반의 discrete flow matching 프레임워크를 채택합니다.
반복 정제 대상이 되는 토큰 시퀀스를 캔버스(canvas)라 부르며, 시간 t는 0(깨끗한 데이터)에서 1(완전히 오염된 상태) 사이 값을 가집니다.
t = 1에서의 소스 분포는 캔버스의 각 토큰이 어휘 V 위에서 독립적으로 균등 분포를 따르는 uniform prior입니다.

#### Forward Process

깨끗한 캔버스 x0가 주어졌을 때 forward 전이 확률은 토큰 위치별로 독립적으로 분해됩니다.

```text
p_t(x_t | x_0) = Π_i [ κ_t · δ(x_t^i, x_0^i) + (1 - κ_t) · 1/|V| ]
```

κ_t는 κ_0 = 1에서 κ_1 = 0으로 단조 감소하는 noise schedule이고 δ는 Kronecker delta입니다.
시간이 1에 가까워질수록 각 토큰이 어휘에서 균등하게 뽑은 임의 토큰으로 교체될 확률이 커집니다.
이 닫힌 형태의 forward process로 깨끗한 텍스트에서 학습 데이터를 생성합니다.

#### Backward Denoising Process

역과정은 작은 시간 간격 h만큼 거꾸로 이동하는 전이 함수 Step으로 정의되며, 이를 계산하려면 노이즈 상태가 주어졌을 때 원본 토큰의 사후분포가 필요합니다.
이 사후분포를 신경망 p_θ로 근사하고, 생성 시에는 x_(t-h) ~ Step(x_t, p_θ(· | x_t)) 형태로 다음 상태를 샘플링합니다.
Step의 대표적인 예로 Gat et al. (2024)의 discrete Euler update가 언급됩니다.

역과정이 위치별로 분해되어 있으므로, 한 스텝 안에서 특정 위치의 토큰은 현재 노이즈 캔버스 전체를 보지만 다른 위치에서 동시에 내려진 샘플링 결과는 보지 못합니다.
이 조건부 독립 가정은 국소적인 불일치나 문법 충돌을 일으킬 수 있으며, DiffusionGemma는 self-conditioning과 entropy-bounded 샘플링으로 이를 보완합니다.

### 아키텍처와 블록 자기회귀 생성

DiffusionGemma는 가중치를 공유하는 encoder-decoder transformer로 동작합니다.
디퓨전 모델을 처음부터 사전학습하지 않고 공개된 Gemma 4 26B A4B MoE 체크포인트로 초기화합니다.
리포트는 이것이 절대적인 생성 품질을 일부 양보하는 대신 학습 비용을 크게 줄이는 실용적 선택이며, 확장된 컨텍스트 윈도우와 네이티브 멀티모달 이해 같은 기반 모델의 기능을 물려받는 효과가 있다고 설명합니다.

| 구성 요소 | 파라미터 수 |
|------|------|
| 전체 | 25.2B |
| 활성(비전 인코더 제외, self-conditioning MLP 포함) | 3.85B |
| 비전 인코더 | 550M |
| 임베더 | 740M |
| Self-conditioning 블록 | 7.8M |
| 활성 / 전체 Expert | 8 / 128 + 공유 1개 |

어휘 크기는 262k 토큰입니다.

#### 블록 자기회귀(Block-AR) 생성

discrete flow matching은 고정 길이 시퀀스에서 동작하므로, 긴 텍스트를 생성하기 위해 block-AR 전략을 사용합니다.
생성 과정은 세 단계로 구성됩니다.

1. 컨텍스트 인코딩: 시스템 지시와 사용자 프롬프트를 causal attention 마스크를 쓰는 Encoder로 처리해 KV cache를 만듭니다.
2. 디노이징 루프: 균등 랜덤 토큰으로 채운 256 토큰 캔버스를 Decoder가 반복 정제합니다. Decoder는 캔버스 내부에 bidirectional attention을, KV cache에 cross-attention을 사용합니다.
3. 인코딩 후 추가: 완성된 캔버스를 다시 causal Encoder에 통과시켜 KV cache에 append하고 다음 캔버스로 넘어갑니다.

모델 턴 종료를 나타내는 특수 토큰이 생성될 때까지 이 과정을 반복하며, 모든 캔버스를 이어 붙여 최종 응답을 만듭니다.
이는 BART, T5 같은 기존 encoder-decoder 구조를 뒤집은 형태입니다.
기존 모델이 컨텍스트에 bidirectional encoder, 생성에 causal decoder를 쓰는 것과 달리, DiffusionGemma는 시퀀스 이력에 causal encoder, 캔버스 생성에 bidirectional decoder를 씁니다.
causal 인코딩 때문에 앞쪽 컨텍스트가 최신 캔버스를 참조할 수는 없지만, 늘어나는 컨텍스트를 매번 처음부터 다시 인코딩할 필요가 없어 긴 추론 생성으로 확장할 수 있습니다.
리포트에 따르면 이 block-AR 전략은 저자 그룹이 2023년 6월 미공개 원고에서 개발했으며, 유사한 블록 단위 모델링과 KV caching 방식이 Fast-dLLM v2, Block diffusion, BlockGen에서 독립적으로 개발되었습니다.

#### 디노이징 반복과 self-conditioning

각 캔버스는 최대 T 스텝, 스텝 크기 h = 1/T로 정제됩니다.
Decoder는 현재 노이즈 캔버스 x_t, 컨텍스트 KV cache, 그리고 이전 예측을 다시 입력으로 넣는 연속 self-conditioning 신호 s_t를 받아 logit을 출력합니다.

```text
logits    = Decoder(x_t, KV, s_t)
p̂_0       = Softmax(logits / τ_t)
s_(t-h)   = FFW(p̂_0 · E)        # E: 토큰 임베딩 행렬
x_(t-h)   ~ Step(x_t, p̂_0)
```

시간에 따라 변하는 temperature τ_t로 예측을 날카롭게 만든 뒤 전이 확률을 계산합니다.

DiffusionGemma는 masked diffusion이 아닌 multinomial(uniform) diffusion을 사용합니다.
모든 토큰이 서로 전이될 수 있으므로, 현재 캔버스 안에서 앞선 스텝에 확정된 토큰도 이후 스텝에서 수정될 수 있습니다.
다만 이전에 생성이 끝난 캔버스의 토큰은 영구적으로 고정됩니다.

self-conditioning 신호는 연속 잠재 공간을 시퀀스에 주입하지만, 최근 해석 가능성 연구에 따르면 이 중간 벡터들은 해석 가능한 토큰 병목에 안정적으로 매핑되어 알고리즘 투명성이 유지된다고 리포트는 덧붙입니다.

#### AR 생성 능력 유지

DiffusionGemma는 Gemma 4와 완전히 같은 transformer 구조를 공유하므로, 최종 가중치를 원래 구조에 다시 로드해 causal attention 기반의 일반 AR 생성을 수행할 수 있습니다.
AR 모드 성능은 DiffusionGemma의 텍스트 디퓨전 모드와 기반 Gemma 4 체크포인트 사이에 위치합니다.

### DiffusionGemma 샘플러

AR 생성이 temperature, top-k, top-p 같은 휴리스틱에 의존하는 것과 달리, 디퓨전은 predictor-corrector, multi-step solver 등 훨씬 넓은 샘플링 설계 공간을 가집니다.
DiffusionGemma의 기본 권장 샘플러는 temperature annealing과 adaptive stopping을 결합한 entropy-bounded sampler입니다.
모델은 모듈식이라 이 설정에 엄격히 묶이지 않습니다.

| 하이퍼파라미터 | 값 |
|------|------|
| 캔버스 길이 | 256 |
| 최대 디노이징 스텝 T | 48 |
| Adaptive stopping 엔트로피 임계값 | 0.005 |
| 토큰 선택 엔트로피 예산 γ | 0.1 |
| Temperature 스케줄(선형) | 0.8 에서 0.4로 감소 |

adaptive stopping을 적용하면 작업에 따라 실제 평균 디노이징 스텝은 약 12입니다.

#### Entropy-bounded 토큰 정제

각 위치의 엔트로피를 계산한 뒤 엔트로피가 낮은 토큰부터 순서대로 수락합니다(MaskGIT 디코딩 방식과 유사).
수락된 토큰들의 엔트로피 누적합, 즉 상호정보 상한이 오차 허용치 γ = 0.1 이하로 유지되는 최대 개수까지만 수락합니다.
임계값에 도달하면 나머지 토큰은 모두 균등 랜덤으로 다시 노이즈를 입혀, 확정되지 않은 위치가 다음 forward pass에서 탐색되도록 합니다.

#### Temperature annealing

temperature는 디노이징 시간축에 따라 0.8에서 0.4로 선형 감소합니다.
노이즈가 많은 초기 상태에서는 다양한 토큰을 탐색하고, 의미 구조가 잡히는 후반에는 높은 확신의 시퀀스로 적극 수렴하도록 하는 장치입니다.

#### Adaptive stopping

다음 두 조건이 동시에 만족되면 디노이징을 조기 종료합니다.

- 확신 조건: 캔버스 전체의 평균 예측 엔트로피가 0.005 미만으로 떨어짐
- 안정 조건: 연속된 두 스텝의 결정적 예측(가장 가능성 높은 토큰 시퀀스)이 동일함

최대 48 스텝을 모두 쓰는 대신 평가 작업 전반에서 평균 약 12 effective denoising step만 사용하며, 리포트는 이를 품질 손실 없이 지연 시간을 4배 줄인 것으로 설명합니다.
도메인별로는 코드처럼 구조화된 작업에서 스텝이 적고 자연어에서 더 많습니다.
난이도도 영향을 미쳐, 어려운 코드 문제(LiveCodeBench)가 쉬운 문제(HumanEval)보다 더 많은 스텝을 사용합니다.

### 추론 효율 지표

adaptive stopping 때문에 디노이징 스텝 수가 확률 변수가 되므로 리포트는 다음 지표를 정의합니다.
K는 생성된 캔버스 수, s_k는 k번째 캔버스의 디노이징 스텝 수, n_k는 k번째 캔버스의 유효 토큰 수(첫 EOS까지)입니다.

```text
Total Tokens            = Σ_k n_k
Total Denoising Steps   = Σ_k s_k
Effective Denoising Steps = (1 / Total Tokens) · Σ_k n_k · s_k
TPF = Total Tokens / (Total Denoising Steps + K - 1)
TPS = TPF / t_fwd
```

Effective Denoising Steps는 토큰 수로 가중 평균하여, 일부만 채워져 스텝이 적게 드는 마지막 캔버스 때문에 값이 낮게 왜곡되는 것을 막습니다.
TPF의 K - 1 항은 캔버스 사이마다 새로 생성된 토큰을 인코딩해 KV cache에 추가하는 추가 forward pass를 반영합니다.
t_fwd는 디노이징 한 스텝에 걸리는 시간으로, attention 때문에 컨텍스트 길이에 따라 달라집니다.

### 1단계: Supervised Finetuning

공개된 Gemma 4 26B A4B 체크포인트에서 시작해, 노이즈가 섞인 입력으로부터 256 토큰 블록을 예측하도록 확장 파인튜닝을 수행합니다.
block-diagonal attention mask로 각 블록 내부에서는 bidirectional attention을 허용하되 다른 디노이징 블록은 참조하지 못하게 합니다.
캔버스는 encoder KV cache를 통해 프롬프트와 이전의 깨끗한 토큰을 조건으로 받습니다.

오염 과정은 discrete multinomial diffusion이며, 캔버스마다 노이즈 레벨 t를 Unif[0, 1]에서 뽑고 각 토큰을 확률 t로 어휘에서 균등 샘플링한 토큰으로 교체합니다.
학습 목표는 정답 캔버스에 대한 cross-entropy 최소화입니다.

```text
L(θ) = - Σ_i log p_θ(x_0^i | x_t, c, s)
```

c는 KV cache로 인코딩된 프롬프트와 이전 캔버스, s는 self-conditioning 신호, i는 캔버스 내 위치 인덱스입니다.

디노이징 성능은 학습 초기에 빠르게 오른 뒤 학습 진행에 따라 log-linear하게 개선됩니다.
non-thinking 모드는 적당한 양의 SFT로 좋은 성능에 도달하지만, thinking 모드는 확장된 SFT가 필수적입니다.
학습 초기에는 일관된 추론 과정을 유지하지 못하고 말더듬(stuttering)이나 순환에 빠지는 경우가 많았습니다.
thinking 성능의 log-linear 추세는 더 낮은 지점에서 시작하지만 기울기가 non-thinking보다 가파릅니다.
SFT 단계 평가는 adaptive stopping을 적용한 entropy-bounded 샘플러와 최대 192 디노이징 스텝을 사용했습니다.

### 2단계: Sampler Distillation과 강화학습(SD²RL)

SFT 후 모델은 많은 디노이징 스텝을 쓸 때 좋은 품질을 보이지만, 고급 추론과 코딩에서는 기반 AR 모델보다 다소 떨어집니다.
더 큰 문제는 초저지연 추론에 필요한 few-step 영역에서 생성 품질이 붕괴한다는 점입니다.

일반적으로 보상 기반 정렬과 sampler distillation은 분리된 다단계 파이프라인으로 처리하지만, DiffusionGemma는 이를 하나의 온라인 학습 단계인 SD²RL로 통합했습니다.
단일 gradient update가 두 가지를 동시에 수행합니다.

1. 보상 최대화: AR·디퓨전 모델의 일반적인 RL처럼 절대 생성 품질과 정렬 수준을 높임
2. Sampler distillation: 고품질 생성을 few-step 영역으로 옮기는, 반복형 디퓨전에 고유한 압축 문제 해결

#### 학습 셋업

Gemma 4 RL 레시피의 데이터 분포를 DiffusionGemma 용도에 맞게 조정했으며, thinking과 non-thinking 모드를 모두 포함해 helpfulness, 수학 추론, 코딩, instruction-following 능력을 대상으로 합니다.
SFT 가중치로 초기화된 모델이 online teacher 역할을 하며, 최대 디노이징 스텝을 크게 잡고 완만한 temperature annealing을 적용한 샘플러로 고품질 디노이징 궤적을 생성합니다.
SD²RL의 결합 목표는 이 궤적을 활용해 보상을 최대화하는 동시에 모델의 최고 품질 출력을 few-step 영역으로 압축합니다.

#### 학습 동역학과 암묵적 커리큘럼

학습이 진행되면서 online teacher의 평균 보상은 꾸준히 증가하고, adaptive stopping 덕분에 높은 보상에 필요한 effective denoising step은 점점 줄어듭니다.
이는 SD²RL 목표가 모델의 예측 엔트로피를 체계적으로 낮추기 때문입니다.
학습 초기에는 엔트로피가 높아 adaptive stopping이 늦게 발동하고, 확신이 높아지며 엔트로피가 떨어지면 더 일찍 발동합니다.
그 결과 학습 분포가 점점 더 짧은 디노이징 궤적으로 이동하며, 알고리즘이 스스로 sampler distillation 속도를 조절하는 커리큘럼 효과가 생깁니다.
AR 모델의 RL과 달리 보상이 정체된 이후에도 SD²RL을 계속하는 것이 유익하며, 엔트로피 감소가 추가 추론 속도 향상으로 이어집니다.

#### Pareto frontier 개선

SD²RL은 SFT 체크포인트의 Pareto frontier를 크게 확장합니다.

| 지표 | 변화 |
|------|------|
| GPQA-Diamond와 LiveCodeBench-v6 결합 점수 | 10점 향상 |
| TPF | 5에서 약 20으로 4배 증가 |
| 생성 길이 | SFT 체크포인트 대비 약 2배 짧아짐 |
| 전체 forward pass | Gemma 4 AR 대비 5% 미만 |

이 비교에서 SD²RL 설정은 최대 48 스텝 DiffusionGemma 샘플러를 쓰고, SFT frontier는 T를 48에서 192까지 바꿔가며 구했으며, 두 지표 모두 thinking 모드에서 GPQA-Diamond와 LiveCodeBench-v6 평균(3 seed)입니다.

SFT 체크포인트를 기본 샘플러(최대 48 스텝)로 평가하면 정확도가 낮고 effective denoising step도 비정상적으로 낮게 나옵니다.
제한된 스텝에서 SFT 모델이 반복 토큰 루프에 자주 빠지고, 루프에 빠지면 예측 엔트로피가 붕괴해 adaptive stopping이 조기에 발동하기 때문입니다.
부록의 GPQA-Diamond 예시에서 SFT 모델은 유전자 G2를 전사 인자로 올바르게 식별하는 등 정상적인 추론으로 시작하지만 숫자 1이나 닫는 괄호를 반복하는 루프에 빠져 빈 응답을 냅니다.
SD²RL 이후 모델은 같은 문제에서 추론을 끝까지 마치고 정답 (B)를 출력합니다.

#### Few-step 영역 특화와 간결성

SFT 체크포인트는 최대 디노이징 스텝 T를 192까지 늘릴수록 성능이 꾸준히 올라가며, 192 스텝에서는 디노이징이 대략 any-order autoregressive에 가까워집니다.
SD²RL 이후에는 T가 커질 때 성능이 더 빠르게 오르고 더 일찍 정체되어, T = 48까지는 꾸준히 개선되지만 그 이후로는 수익이 체감합니다.
SD²RL 목표가 예측 엔트로피를 적극적으로 줄여 모델을 few-step 영역에 특화시키기 때문입니다.

SD²RL의 창발적 특성으로 출력이 간결해집니다.
긴 추론 과정을 유도해 보상을 높이는 경향이 있는 AR 모델의 RL과 대조적입니다.
긴 추론에 따르는 일부 성능 이득은 포기하지만, 더 적은 총 토큰과 캔버스당 더 적은 디노이징 스텝이 곱해져 종단 지연 시간을 크게 낮춥니다.

### 추론 최적화

디퓨전의 AR 대비 처리량 이점은 두 요소의 경쟁으로 결정됩니다.
디노이징 스텝 하나가 TPF개 토큰을 디코딩하므로 forward pass는 TPF배 적게 필요하지만, 각 스텝이 256 토큰 캔버스를 처리하므로 스텝당 시간은 r배 길어집니다.
따라서 AR 대비 처리량은 TPF / r이 되며, forward pass 오버헤드는 하드웨어 최적화로 크게 줄일 수 있습니다.

텍스트 디퓨전은 생성 토큰당 FLOPs가 더 많지만 forward pass가 훨씬 적습니다.
LLM 서빙은 대개 KV cache 용량과 메모리 대역폭에 좌우되는 memory-bound이므로, 메모리 전송 감소에 따른 지연 시간 이점이 추가 연산 비용을 상쇄합니다.
그래서 텍스트 디퓨전은 배치 크기가 작은 상황에서 특히 효과적이며, 리포트의 분석은 배치 크기 1을 기준으로 Gemma 4 26B A4B와 비교합니다.
참조 구현은 HuggingFace Transformers와 vLLM에 공개되어 있습니다.

#### GPU 시간 분석

H100, FP8, 입력 4,096 토큰, 출력 1,024 토큰, 배치 크기 1 조건에서 DiffusionGemma의 스텝은 256배 많은 토큰을 처리하지만 단일 토큰 AR 스텝보다 3.2배만 느립니다.
두 모델 모두 종단 지연 시간은 detokenization 등 CPU 측 오버헤드로 GPU 커널 시간보다 약 1ms 더 깁니다.
차이는 주로 다음 세 연산에서 발생하며, shared expert나 attention output projection 같은 나머지 연산은 최대 2배 느린 수준입니다.

| 연산 | 내용 |
|------|------|
| MoE | AR은 토큰당 MoE 레이어마다 8개 expert만 활성화되지만, DiffusionGemma는 256 토큰 캔버스당 평균 약 84개 고유 expert가 활성화됨(PG-19 측정). MoE 커널이 4.3배 느려짐. dense 구조였다면 FFN 감속이 2배 미만으로 줄어듦 |
| 샘플링 | 262k 어휘 차원에서 self-conditioning 임베딩 matmul과 캔버스 전체 softmax를 full precision으로 수행. 디퓨전 샘플링 3.06ms, AR 샘플링 0.56ms. 커뮤니티 확장성을 위해 수작업 커널 대신 PyTorch 기본 연산과 torch.compile 사용 |
| Attention | 캔버스 256 토큰에 bidirectional attention을 쓰므로 단일 토큰 디코딩용 attention을 쓸 수 없고 FlashAttention-4 커널 사용. AR 대비 4배 느림 |

#### CPU-GPU 동기화 제거

DiffusionGemma 서빙은 adaptive stopping에 따라 디노이징 스텝 수가 달라지고, 서로 다른 attention mask를 쓰는 두 종류의 forward pass(디노이징 스텝과 KV cache 업데이트)가 같은 배치 안에 섞일 수 있습니다.
요청 지연을 최소화하기 위해 이 복잡도를 추가 CPU-GPU 동기화 없이 GPU 연산만으로 처리하며, asynchronous scheduling을 텍스트 디퓨전으로 확장하고 시퀀스별 causal attention 플래그를 도입했습니다.

#### 배치 크기 1 처리량

H100(FP8)에서 입력 4,096 토큰, 출력 1,024 토큰 단일 요청 기준 디노이징 한 스텝은 평균 t_fwd = 13.56ms(종단 기준, GPU 시간은 12.63ms)입니다.
7개 벤치마크 평균 TPF 19.74를 적용하면 평균 디코딩 처리량은 1,456 TPS입니다.

| 모델 | TPS | DiffusionGemma 대비 |
|------|------|------|
| DiffusionGemma | 1,456 | - |
| Gemma 4 AR | 204 | 7.1배 느림 |
| Gemma 4 AR + MTP | 303 | 4.8배 느림 |

#### 다중 사용자 처리량

배치 크기를 늘린 비교(H100, FP8, PG-19, 입력 4,096·출력 1,024, MTP draft 길이 4)에서 DiffusionGemma는 저배치 영역에서 사용자당 TPS와 전체 처리량 모두 Gemma 4 AR(MTP)보다 높습니다.
AR 모델은 동시 요청이 약 32개 정도인 중간 배치 크기부터 처리량 우위를 가지기 시작합니다.
이 결과는 배치 크기 1 초과에 대한 별도 최적화 없이 얻은 것으로, 커널 선택이 최적이 아니고 샘플링이 배치 크기에 맞게 튜닝되지 않았습니다.
예를 들어 샘플링 단계에 top-k truncation을 적용하면 출력 품질 영향은 거의 없이 높은 배치 크기에서 처리량이 크게 오를 것으로 예상됩니다.

리포트는 DiffusionGemma가 attention 레이어에서 KV cache 전송을 TPF배 줄이고 MoE 레이어에서 FLOPs를 늘린다는 점에 주목합니다.
이는 메모리 병목인 attention이 FFN/MoE 레이어의 연산 활용도를 제한하는 문제, 특히 긴 컨텍스트 에이전트 워크플로에서의 문제를 완화할 잠재력이 있습니다.
연산 대 대역폭 비율이 계속 커지는 최신 GPU에서 데이터 이동을 연산으로 바꾸는 방식이 유리하지만, 실제 트래픽 조건에서의 분석은 이번 범위 밖입니다.

## 주요 결과

### 실험 셋업

평가는 네 가지 동작 모드로 이루어집니다.
텍스트 디퓨전(TD)과 AR 생성 각각에 대해 thinking 활성화 여부를 나누며, 별도 언급이 없으면 TD + thinking이 기본입니다.
비교 대상은 초기화에 쓰인 Gemma 4 26B A4B, 오픈 웨이트 텍스트 디퓨전 모델 LLaDA 2.1 Flash 100B와 Nemotron Diffusion 14B, 그리고 독점 API인 Mercury 2입니다.

| 영역 | 벤치마크 |
|------|------|
| 수학 추론 | AIME 2026, GSM8K, MGSM, Putnam, HiddenMath(내부) |
| 코딩 | LiveCodeBench-v6, Codeforces, HumanEval, BigCodeBench, LBPP(v2), Natural2Code(내부) |
| 일반 지식 | GPQA-Diamond, BIG-Bench Extra Hard, MMMLU, MMLU-Pro |
| 멀티모달 | MMMU-Pro |
| 지시 수행 | IFEval |
| 에이전트 | Tau2-bench Retail, Airline, Telecom |

Natural2Code와 HiddenMath는 유출되지 않은 독점 평가입니다.
속도 측정 환경은 DiffusionGemma와 Gemma 4가 H100 1장(FP8, 배치 1), Nemotron 14B가 H100 1장(bfloat16, 배치 1), LLaDA 2.1 Flash 100B가 B200 8장(bfloat16, 배치 1)이며, Mercury 2는 공개 API로 측정했습니다.
TPS는 prefill 시간을 제외하며, TPS·TPF·총 토큰은 모든 모델의 측정값이 있는 7개 벤치마크(AIME 2026, GPQA Diamond, LiveCodeBench-v6, MGSM, HumanEval, LBPP, Natural2Code) 평균입니다.
Gemma 4(MTP)의 TPS와 TPF는 SPEED-Bench로 측정했습니다.

### 품질 벤치마크

아래는 DiffusionGemma의 네 가지 모드 중 TD, TD(No-think), AR과 기반 모델 Gemma 4 26B A4B(thinking)의 점수입니다.

| 벤치마크 | DiffusionGemma TD | DiffusionGemma TD (No-think) | DiffusionGemma AR | Gemma 4 26B A4B |
|------|------|------|------|------|
| AIME 2026 | 69.1 | 50.8 | 84.2 | 88.3 |
| GPQA Diamond | 73.2 | 64.6 | 79.8 | 82.3 |
| LiveCodeBench-V6 | 69.1 | 60.6 | 71.4 | 77.1 |
| Codeforces ELO | 1429 | 959 | 1569 | 1718 |
| BigBench EH | 47.6 | 40.0 | 59.1 | 64.8 |
| GSM8K | 96.3 | 95.8 | 96.6 | 96.7 |
| MGSM | 84.8 | 80.7 | 87.9 | 92.9 |
| MMMLU | 81.5 | 76.3 | 82.2 | 86.3 |
| MMMU Pro | 54.3 | 66.0 | 63.3 | 73.8 |
| Putnam | 67.4 | 57.1 | 74.7 | 81.0 |
| HumanEval | 94.5 | 92.7 | 98.2 | 98.8 |
| BigCodeBench | 46.0 | 41.9 | 47.7 | 50.2 |
| LBPP | 81.0 | 68.9 | 86.3 | 89.5 |
| IFEval | 97.4 | 94.5 | 97.2 | 98.7 |
| Tau2 Retail | 71.5 | 57.5 | 75.4 | 85.5 |
| Tau2 Airline | 69.0 | 49.0 | 72.0 | 76.0 |
| Tau2 Telecom | 28.1 | 32.0 | 33.8 | 43.0 |
| MMLU-Pro | 77.6 | 77.9 | 78.8 | 82.6 |
| Natural2Code | 94.0 | 90.1 | 96.2 | 96.3 |
| HiddenMath | 80.6 | 74.3 | 85.4 | 87.2 |

같은 표의 속도 관련 행은 다음과 같습니다.

| 모델 / 모드 | 출력 속도(TPS) | TPF | 평균 총 토큰 |
|------|------|------|------|
| DiffusionGemma TD | 1,479 | 19.74 | 4,001 |
| DiffusionGemma TD (No-think) | 1,512 | 18.76 | 829 |
| DiffusionGemma AR | 204 | 1.00 | 5,184 |
| Gemma 4 AR (MTP) | 303 | 1.40 | 7,207 |

리포트의 해석은 다음과 같습니다.

- DiffusionGemma는 기존 오픈 웨이트 디퓨전 모델을 크게 앞서면서 TPF를 약 한 자릿수(10배) 높여 텍스트 디퓨전 모델의 새 성능 frontier를 제시합니다.
- 품질 면에서 클로즈드 웨이트 디퓨전 모델인 Mercury 2와 경쟁력이 있으며, 이전 세대 H100 1장에서 약 1,500 TPS로 Mercury 2 대비 약 2.5배 빠릅니다.
- TD 모드는 원본 AR 모델보다 세 역량 영역(추론·지식, 코딩, 지시 수행·에이전트) 모두에서 점수가 낮지만, 고도로 최적화된 MTP 서빙의 Gemma 4 대비 출력 처리량이 약 5배(1,479 vs 303 TPS)입니다.
- AR 모드로 실행하면 TD 모드에서 생긴 성능 격차의 일부를 회복하지만 처리량은 낮아지며, 이 이중 모드는 지연 요구와 작업 난이도에 따라 요청을 동적으로 라우팅하는 용도로 쓸 수 있습니다.

Mercury 2 속도는 OpenRouter API에 질의해 응답 메타데이터(입력 토큰, 캐시 토큰, 출력 토큰, 총 소요 시간)로부터 추정했습니다.
총 시간 = α · 실질 prefill 토큰 + β · 출력 토큰 + c 모델을 비음수 최소제곱(NNLS)으로 벤치마크별로 맞춘 뒤, 1/β를 생성 TPS로 보고 산술 평균했습니다.
GPQA Diamond 452.6 TPS, LiveCodeBench-v6 525.5 TPS로 평균 489 TPS를 얻었고, 7개 벤치마크 기준으로는 high reasoning effort 600 TPS, medium 547 TPS입니다.
HumanEval과 Natural2Code에서 thinking 루프에 빠져 50,000 토큰을 매우 빠르게 생성하는 이상치가 있어 추정치가 다소 유리하게 편향되었으며, high effort가 medium보다 TPS가 높은 이유도 이것으로 설명됩니다.
Inception은 Blackwell GPU에서 1,000 TPS, Artificial Analysis는 비공개 하드웨어에서 중앙값 987 TPS를 보고했으며, 리포트는 하드웨어·서빙 최적화·측정 데이터셋 차이를 원인으로 들고 자신들의 추정치는 OpenRouter API 사용자가 경험하는 평균 속도를 반영한다고 밝힙니다.

### 속도와 디노이징 효율

TD thinking 모드의 벤치마크별 효율 지표입니다(E2E 시간은 샘플당, prefill 제외).

| 벤치마크 | 점수 | TPF | TPS | Effective DNS | E2E 시간(s) |
|------|------|------|------|------|------|
| AIME 2026 | 69.1 | 19.3 | 1365.4 | 12.6 | 4.72 |
| GPQA Diamond | 73.2 | 16.7 | 1207.8 | 15.1 | 4.68 |
| LiveCodeBench-V6 | 69.1 | 18.5 | 1278.3 | 13.8 | 5.89 |
| Codeforces ELO | 1429 | 15.1 | 950.5 | 17.1 | 12.23 |
| BigBench EH | 47.6 | 20.8 | 1390.2 | 11.9 | 6.52 |
| GSM8K | 96.3 | 23.2 | 1866.2 | 9.1 | 0.47 |
| MGSM | 84.8 | 19.0 | 1526.7 | 11.5 | 0.71 |
| MMMU Pro | 54.3 | 17.7 | 1351.3 | 13.8 | 2.35 |
| Putnam | 67.4 | 18.1 | 1330.8 | 13.4 | 3.55 |
| HumanEval | 94.5 | 23.0 | 1838.2 | 9.4 | 0.64 |
| BigCodeBench | 46.0 | 19.6 | 1560.1 | 11.3 | 0.90 |
| LBPP | 81.0 | 20.5 | 1509.8 | 11.6 | 3.13 |
| IFEval | 97.4 | 17.2 | 1368.3 | 13.0 | 1.07 |
| Natural2Code | 94.0 | 21.1 | 1682.7 | 10.5 | 0.83 |
| HiddenMath | 80.6 | 21.0 | 1591.2 | 11.3 | 2.20 |

GSM8K, HumanEval처럼 쉬운 작업은 Effective DNS가 9 수준으로 낮고 TPF가 23 이상으로 높습니다.
Codeforces, GPQA Diamond처럼 어려운 작업은 Effective DNS가 15~17로 늘어 TPS가 낮아지며, adaptive stopping이 작업 난이도에 따라 연산량을 조절한다는 앞선 설명과 일치합니다.

### 다운스트림 파인튜닝

DiffusionGemma와 함께 도메인 특화 데이터셋에 모델을 맞출 수 있는 오픈소스 파인튜닝 툴킷이 공개되었습니다.
JAX 기반 모듈식 생성 모델링 연구 도구인 Hackable Diffusion 위에 구축되었으며, 소비자급 하드웨어에서도 파인튜닝할 수 있도록 LoRA 레시피를 제공합니다.

학습 시퀀스 길이는 프롬프트 토큰 수 P에 캔버스 수 K와 캔버스 크기 L의 곱을 더한 값입니다.
툴킷은 causal encoder 손실과 diffusion decoder 손실을 모두 사용하며 최종 손실은 두 손실의 합입니다.

- Encoder 손실: 전체 시퀀스를 처리해 KV cache를 채우고 다음 토큰 예측에 대한 표준 cross-entropy를 계산
- Decoder 손실: K개 캔버스 중 하나를 균등 샘플링해, 노이즈 상태와 프롬프트 및 이전 캔버스의 KV cache를 조건으로 디노이징 cross-entropy를 계산
- Self-conditioning: 배치의 50%는 이전 forward pass에서 계산한 self-conditioning 상태를 조건으로 받고, 나머지 50%는 0으로 둠

LoRA는 attention projection, MLP gate, MoE router, self-conditioning feedforward 블록 등 모든 linear 연산에 적용되며 A100 80GB 2장으로 학습합니다.

#### Sudoku 사례

Sudoku는 비자기회귀적 성격 때문에 이산 디퓨전의 좋은 시험대입니다.
오픈소스 Sudoku 데이터셋으로 파인튜닝한 결과는 다음과 같습니다.

| 모델 | Effective Denoising Steps | 정확도(%) |
|------|------|------|
| DiffusionGemma(파인튜닝 전) | 40.65 | 0.00 |
| DiffusionGemma + LoRA rank 8 | 10.72 | 84.40 |

파인튜닝 전 모델은 올바른 Sudoku 격자를 하나도 만들지 못했고, "해답 격자만 출력하라"는 제약을 지키지 못한 채 thought 채널에서 잘못된 격자를 만들다 응답을 내지 못하기도 했습니다.
rank 8 LoRA는 모든 linear 레이어에 적용해도 약 8M 파라미터만 학습합니다.
full finetuning(A100 80GB 8장, Adafactor)으로는 4,096개 held-out 퍼즐에서 85% 이상의 퍼즐 단위 정확도를 기록했으며, LoRA rank를 낮추면 정확도와 연산량이 트레이드오프됩니다.
파인튜닝 후 예측 엔트로피가 낮아져 디노이징 스텝도 크게 줄었습니다.

생물의학 질의응답 벤치마크 PubMedQA에도 rank 4 LoRA로 파인튜닝했으며, 리포트는 기본 성능이 좋은 모델에서 정확도가 소폭 상승했다고 설명합니다.
의학 연구 초록을 읽고 yes/no/maybe 답과 설명 문단을 생성하는 과제로, 설명 문단은 BLEU로 평가합니다.

### 텍스트 디퓨전의 실용적 이점

리포트는 지연 시간 외의 이점도 정성적 예시로 보여 줍니다.
아키텍처 고유의 특성을 보기 위해 DiffusionGemma와 Gemma AR 모두 thinking 모드를 끄고 비교했습니다.

#### 양방향 추론과 자기 교정

AR의 next-token prediction은 이미 생성된 앞쪽 토큰만 참조할 수 있지만, 텍스트 디퓨전은 캔버스 전체에 bidirectional attention을 적용하므로 캔버스 안에서 뒤쪽 토큰이 앞쪽 토큰 형성에 영향을 줄 수 있습니다.
반복 정제 구조이므로 초기 스텝의 성급한 결정을 이후 스텝에서 수정할 수 있습니다.

- 산술 문제: "답을 먼저 쓰고 풀이를 제시하라"는 조건의 계산 문제에서 Gemma 4는 첫 토큰부터 틀린 답 -1을 확정한 뒤 풀이 중에 오류를 발견하고 정정문을 덧붙입니다. DiffusionGemma도 처음에는 -1에 높은 확률을 주지만 다음 스텝에서 -15를 거쳐 정답 -25로 수렴하며, 디노이징은 5 스텝에 끝납니다.
- 개구리 퍼즐: 20피트 지점의 순간이동 연잎 때문에 무한 루프가 생기는 문제에서 Gemma 4는 "Yes"로 시작해 스스로 모순되는 답을 내고, DiffusionGemma는 처음에 "Yes"에 높은 확률을 주었다가 "No"로 수정하며 6 스텝에 디노이징을 마칩니다.

#### 동적·적응형 연산

AR 모델은 생성 토큰마다 고정된 연산을 쓰지만, 텍스트 디퓨전은 adaptive stopping으로 작업 난이도에 따라 연산량을 스스로 조절합니다.
최대 디노이징 스텝 수는 지연 시간과 품질의 트레이드오프를 조절하는 명시적 설정값 역할을 합니다.

부록 예시에서 각 토큰이 직전 두 생성 토큰에 의존하는 이진 수열 생성(순차 의존 과제)은 대체로 왼쪽에서 오른쪽 순서로 풀리며 40 토큰(숫자 20개와 공백 20개)을 7 스텝에 생성합니다.
AR 모델이라면 40번의 순차 스텝이 필요한 작업입니다.
같은 규칙을 주어진 입력 문자열에 적용하는 과제(합성곱형 의존)는 출력 간 의존성이 없어 모든 규칙을 병렬로 풀고 4 스텝에 수렴합니다.

#### 구조화·제약 출력

JSON 스키마처럼 형식이 엄격하거나 OCR, 코드 편집처럼 입력과의 어휘적 의존성이 큰 출력에서는 디퓨전이 구조적 사전 정보를 활용해 빠르게 수렴합니다.
AR 모델은 정형화된 상투 구문이나 입력의 그대로 복사인 경우에도 N개 토큰 생성에 O(N) 비용을 그대로 치러야 합니다.

- JSON 추출: 초기 캔버스부터 대부분 정확하고 토큰당 평균 신뢰도가 83%를 넘으며, 두 번째 스텝에서 신뢰도 100%로 수렴해 2 스텝 만에 응답을 반환합니다.
- Python 코드 디버깅: 대부분의 토큰이 입력에서 그대로 복사되고 국소적인 수정만 필요해 3 스텝 만에 수정된 코드를 생성합니다.

## 프롬프트 포맷

DiffusionGemma는 Jinja2 기반 chat template을 사용하며 대화 이력, 시스템 지시, 도구 스키마를 특수 제어 토큰으로 직렬화합니다.

- BOS/EOS: 다른 Gemma 모델처럼 모든 대화는 텍스트 표현이 없는 BOS 토큰(정수 2)으로 시작해야 하며, 토크나이저의 add_bos=True 옵션으로 붙일 수 있습니다. block-AR 생성이 끝나면 턴 종료 토큰 뒤를 EOS 토큰(정수 1)으로 패딩합니다.
- 역할: system, user, model, tool(도구 호출 응답) 네 가지 역할을 턴 시작·종료 토큰으로 감쌉니다.
- 멀티모달: 콘텐츠 배열을 파싱해 이미지 토큰을 삽입하고, forward pass에서 해당 토큰은 토큰 임베딩 대신 이미지 특징으로 대체됩니다.
- Thinking: Gemma 4와 마찬가지로 시스템 지시에 think 토큰을 추가하면 thinking 모드가 활성화되고, 내부 추론 채널 뒤에 최종 답변이 나옵니다. think 토큰이 없어도 빈 thought 채널을 출력하며, 멀티턴 대화에서는 이전 턴의 숨은 추론을 이력에 넣지 말고 최종 응답만 포함해야 합니다.
- 도구 호출: 도구 정의, 도구 호출, 도구 응답을 각각 전용 토큰 쌍으로 감싸고, JSON 형태 스키마에는 일반 큰따옴표 대신 전용 구분자 토큰을 사용합니다.

## 한계와 주의사항

리포트는 실험적 릴리스로서 다음 한계를 명시합니다.

- AR 기반 모델과의 성능 격차: 디퓨전 사전학습 없이 AR 가중치에서 warm-start했고, 연산 예산 때문에 SFT 단계가 비교적 짧았으며, SD²RL이 초저지연을 목표로 점근적 성능을 일부 양보하고, AR 기반 모델의 구조·최적화·데이터 구성 결정이 이산 디퓨전에 최적이 아닐 수 있다는 점이 원인입니다.
- 생성 길이와 간결성: 최종 체크포인트는 매우 간결한 출력을 내므로 추론 속도에는 유리하지만, 길고 정교한 추론 과정이 주는 품질 향상을 활용하지 못합니다.
- 간헐적 말더듬: 드물게 "the the the"처럼 흔한 토큰을 반복하는 루프에 빠집니다. SD²RL이 대부분을 완화했지만 디노이징 스텝을 공격적으로 줄인 초저지연 영역의 직접적 결과로 남아 있습니다.
- 멀티모달 작업에서 thought 종료 태그 누락: 멀티모달 프롬프트에서 추론이 맞더라도 thought 종료 태그를 항상 생성하지는 않아 thinking 모드 점수가 떨어질 수 있습니다. MMMU-Pro에서 thinking 점수가 non-thinking보다 낮은 것(54.3 vs 66.0)이 그 예이며, 파이프라인 후반에 발견되어 이번 릴리스에서는 수정하지 못했습니다.
- 높은 배치 크기에서의 처리량 한계: 동시 사용자 32명까지는 Gemma 4 AR(MTP)보다 사용자당·전체 처리량이 높지만, 그 이상에서는 토큰당 높은 연산 비용 때문에 AR 모델이 우위를 가집니다. 배치 크기에 대한 별도 최적화와 실제 트래픽 조건 분석은 향후 과제입니다.

리포트에는 별도의 안전성 평가 섹션이 포함되어 있지 않습니다.

## 결론

DiffusionGemma는 기존 Gemma 4 26B A4B AR 모델을 SFT와 SD²RL의 2단계 파이프라인으로 텍스트 디퓨전 모델로 전환하여, 연산 효율적인 초고속 텍스트 생성 경로를 보여 줍니다.
256 토큰 캔버스를 평균 약 12 스텝에 디노이징해 forward pass당 약 20 토큰을 생성하고, 단일 H100에서 약 1,500 TPS를 달성하면서 경쟁력 있는 추론·멀티모달 능력을 유지합니다.
causal encoder와 bidirectional decoder를 결합한 block-AR 구조, entropy-bounded 샘플러와 adaptive stopping, 보상 최대화와 sampler distillation을 한 번에 수행하는 SD²RL이 이 결과를 뒷받침합니다.
반면 원본 AR 모델 대비 성능 격차, 간결성으로 인한 긴 추론 이득 포기, 고배치 처리량 한계는 남아 있습니다.
Google DeepMind는 HuggingFace Transformers·vLLM 참조 구현, 파인튜닝 툴킷과 함께 Apache 2.0으로 모델을 공개했으며, 특화 작업 파인튜닝, 새로운 샘플링 알고리즘, 추론 효율 최적화 연구를 기대한다고 밝힙니다.

## Reference

- [DiffusionGemma Technical Report (arXiv 2608.00146)](https://arxiv.org/abs/2608.00146)
- [DiffusionGemma 26B A4B IT chat template (Hugging Face)](https://huggingface.co/google/diffusiongemma-26B-A4B-it/blob/main/chat_template.jinja)
- [DiffusionGemma: The first diffusion LLM natively supported in vLLM](https://vllm.ai/blog/2026-06-10-diffusion-gemma)
- [DiffusionGemma implementation in HuggingFace Transformers (PR)](https://github.com/huggingface/transformers/pull/46540)
- [Hackable Diffusion (GitHub)](https://github.com/google/hackable_diffusion)
