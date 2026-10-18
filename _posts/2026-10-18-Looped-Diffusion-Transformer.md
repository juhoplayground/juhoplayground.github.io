---
layout: post
title: "Looped Diffusion Transformer"
author: 'Juho'
date: 2026-10-18 00:00:00 +0900
categories: [AI]
tags: [AI, Benchmark, Evaluation, GPU]
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
2. [연구 배경과 동기](#연구-배경과-동기)
   - [Naive Looping의 한계](#naive-looping의-한계)
3. [방법론](#방법론)
   - [Looped-DiT 구조](#looped-dit-구조)
   - [Deep Supervision](#deep-supervision)
   - [Self-Modulating Attention](#self-modulating-attention)
4. [실험 설정](#실험-설정)
   - [모델 구성](#모델-구성)
   - [학습 설정과 데이터](#학습-설정과-데이터)
   - [평가 벤치마크와 추론 설정](#평가-벤치마크와-추론-설정)
5. [주요 결과](#주요-결과)
   - [파라미터 효율](#파라미터-효율)
   - [추론 효율](#추론-효율)
   - [추론 능력과 자기 교정](#추론-능력과-자기-교정)
6. [Looping은 꼭 필요한가](#looping은-꼭-필요한가)
   - [동일 파라미터와 동일 연산량 비교](#동일-파라미터와-동일-연산량-비교)
   - [Loop 깊이 vs 디노이징 스텝](#loop-깊이-vs-디노이징-스텝)
   - [Textual CoT와의 상보성](#textual-cot와의-상보성)
7. [Ablation Study](#ablation-study)
   - [구성 요소별 기여](#구성-요소별-기여)
   - [Deep Supervision과 Adaptive Looping](#deep-supervision과-adaptive-looping)
   - [Self-Modulating Attention 분석](#self-modulating-attention-분석)
8. [학습 및 추론 비용](#학습-및-추론-비용)
9. [관련 연구](#관련-연구)
10. [한계와 향후 과제](#한계와-향후-과제)
11. [결론](#결론)
12. [Reference](#reference)

## 개요

"Looped Diffusion Transformer"는 SenseTime Research, Tsinghua University LeapLab, Nanyang Technological University 연구진이 2026년 9월 arXiv에 공개한 논문입니다.
텍스트-이미지(T2I) 모델의 성능은 전통적으로 모델 크기를 키우거나 디노이징 스텝 수를 늘리는 방식으로 개선되어 왔습니다.
이 논문은 각 디노이징 스텝 안에서 공유된 Transformer 블록을 반복 실행하여, 파라미터 수는 그대로 둔 채 연산 깊이만 늘리는 대안적 스케일링 방식을 탐구합니다.
이러한 looped computation은 명시적인 reasoning 토큰 없이도 내부 표현을 반복적으로 정제할 수 있게 해 줍니다.

다만 단순한 looping(naive looping)은 이미지 품질을 일관되게 개선하지 못합니다.
연구진은 그 원인을 중간 loop에 대한 약한 supervision과, 로컬 정보를 점차 침식시키는 통제되지 않은 attention 업데이트로 진단했습니다.
이를 해결하기 위해 중간 loop 전반에 걸친 Deep Supervision과 loop 간 feature 업데이트를 안정화하는 Self-Modulating Attention을 결합한 Looped-DiT를 제안합니다.

핵심 결과는 다음과 같습니다.

| 항목 | 내용 |
|------|------|
| 파라미터 효율 | 260M 파라미터 Looped-DiT가 6.5배 큰 모델(InternVL-U, 1.7B)을 여러 T2I 벤치마크에서 능가 |
| 추론 연산량 | InternVL-U 대비 4.9배 낮은 추론 연산량(150 vs 732 TFLOPs), 지연 시간 3.2배 단축(0.65s vs 2.1s) |
| 추론 시간 스케일링 | 동일 추론 예산에서 loop 깊이를 늘리는 것이 디노이징 스텝을 늘리는 것보다 더 큰 성능 향상 |
| Latent reasoning | 깊은 loop가 앞선 loop의 오류를 점진적으로 교정하는 행동 관찰 |

코드는 [OpenSenseNova/Looped-DiT](https://github.com/OpenSenseNova/Looped-DiT){:target="_blank"}에 공개되어 있습니다.

## 연구 배경과 동기

Scaling law는 모델 크기, 데이터, 학습 연산량을 늘리면 언어 모델링 성능이 체계적으로 향상된다는 점을 보여 주었고, 이는 연산을 더 효율적으로 스케일링하는 방법에 대한 탐색으로 이어졌습니다.
지배적인 두 접근법은 각각 비용을 수반합니다.
더 깊거나 넓은 Transformer는 그만큼 더 많은 파라미터를 요구하고, chain-of-thought(CoT) reasoning은 더 긴 토큰 시퀀스를 통해 추론 시 연산량을 늘립니다.
Looped Transformer는 공유된 중간 블록을 반복 적용해 hidden state를 정제함으로써 파라미터 수나 시퀀스 길이를 늘리지 않고 연산 깊이를 늘리는 대안입니다.
이 특성은 Qwen-Image, FLUX.2처럼 수십억 파라미터로 커져 배포 비용이 큰 T2I 분야에서 특히 매력적입니다.

이미지 생성에서 looping의 의미는 연산 효율에만 그치지 않습니다.
일관된 이미지를 생성하려면 프롬프트를 문자 그대로 따르는 것 이상이 필요합니다.
모델은 (1) 암시된 시각적 내용을 추론하고, (2) 서로 의존적인 제약을 해소하며, (3) 생성 과정에서 불일치를 식별하고 교정해야 합니다.
이 과정들은 본질적으로 시각적이기 때문에, looping은 명시적 텍스트 reasoning 없이 hidden representation 안에서 이를 직접 수행할 수 있는 자연스러운 수단이 됩니다.
또한 중간 loop의 예측을 외부 annotation 없이 동일한 타깃 이미지로 supervise할 수 있어 표준 diffusion 학습과도 잘 결합됩니다.

연구진은 이를 검증하기 위해 단순한 구조와 학습 파이프라인을 가진 pixel-space MMDiT(Multimodal Diffusion Transformer)인 MiniT2I에 looping을 도입했습니다.
Transformer 블록을 pre-loop, looped, post-loop의 세 그룹으로 나누고, 각 디노이징 스텝 안에서 pre-loop과 post-loop 블록은 한 번씩, looped 블록은 파라미터를 공유한 채 N번 실행합니다.

### Naive Looping의 한계

초기 실험에서 naive looping은 생성 품질을 안정적으로 개선하지 못했습니다.
얕은 loop 깊이에서는 non-looped baseline보다 낮은 성능에 머물 수 있고, loop를 더 추가하면 성능이 포화되며, 학습 시 loop 깊이를 넘어서면 결국 하락합니다.
연구진은 동일한 변환을 반복 적용하면 중복된 업데이트가 이미지 토큰 표현의 정보를 덮어쓰거나 약화시킨다는 가설을 세웠습니다.

이를 확인하기 위해 각 loop 깊이에서 hidden state로부터 각 이미지 토큰의 2D patch-grid 좌표를 예측하는 ridge-regression probe를 학습했습니다.
그 결과 R² 값이 1번째 loop 이후 0.865에서 8번째 loop 이후 0.562로 절대값 0.303 감소했습니다.
즉 loop가 반복될수록 공간 위치 정보가 선형적으로 복원되기 어려워집니다.
연구진은 효과적인 looping을 위해 (1) 중간 loop 예측에 대한 의미 있는 supervision과 (2) 표현 변화에 따라 attention 업데이트 강도를 적응적으로 조절하는 능력이 모두 필요하다고 결론지었습니다.

논문이 밝힌 기여는 다음 세 가지입니다.

- Looping이 T2I 생성에서 파라미터 효율적인 스케일링 방식임을 입증
- 동일 추론 연산량에서 loop 깊이를 늘리는 것이 디노이징 스텝을 늘리는 것보다 효과적이라는, 추론 시간 스케일링의 상보적 축으로서의 loop 깊이 제시
- 깊은 loop가 이전 오류를 교정하고 상호 의존 제약을 해소하는, 명시적 텍스트 reasoning 없는 latent visual reasoning의 근거 제시

연구진은 이 연구가 T2I 생성에서 looped computation을 체계적으로 다룬 첫 연구라고 설명합니다.

## 방법론

### Looped-DiT 구조

Looped-DiT는 MMDiT 기반 pixel-space denoiser인 MiniT2I 위에 구축됩니다.
네트워크를 pre-loop 단계 A, looped 단계 B, post-loop 단계 C로 분할합니다.
노이즈 이미지 $$x_t$$와 텍스트 조건 $$y$$에 대응하는 이미지·텍스트 토큰 표현을 $$h_{\text{input} }$$이라 할 때 forward 연산은 다음과 같습니다.

$$
h^{(0)} = A(h_{\text{input} }), \quad h^{(r)} = B(h^{(r-1)}),\ r = 1, \dots, N, \quad \hat{x}_0 = C(h^{(N)})
$$

여기서 N은 loop 깊이입니다.
각 반복에서 B는 이미지와 텍스트 hidden state를 모두 업데이트하며, 그 결과가 다음 반복의 입력이 됩니다.
B의 파라미터는 반복 간 공유되므로 N을 늘려도 파라미터 수는 늘지 않고 유효 연산 깊이만 증가합니다.
Looped computation은 sampler가 다음 스텝으로 넘어가기 전 각 디노이징 스텝 내부에서 일어나므로, loop 깊이는 디노이징 스텝 수와 독립적으로 조절할 수 있습니다.

### Deep Supervision

최종 예측만 supervise하면 gradient가 이후의 모든 반복을 거쳐 역전파되어야 앞쪽 loop state에 도달합니다.
Loop 깊이가 깊어질수록 최적화 경로가 길고 간접적이 되어 앞쪽 state는 직접적인 학습 신호를 받지 못합니다.
연구진은 Deeply-supervised nets에서 영감을 받아 최종 state뿐 아니라 모든 loop 깊이의 예측에 flow-matching 목적함수를 적용하는 Deep Supervision을 도입했습니다.

각 loop 깊이 $$n = 1, \dots, N$$에 대해 해당 hidden state를 공유된 post-loop 단계로 디코딩합니다.

$$
\hat{x}_0^{(n)} = C\left(h^{(n)}\right)
$$

Flow-matching loss는 clean-image 예측 형태로 표현됩니다.
학습 이미지 $$x_0$$에 대해 노이즈 입력은 다음과 같이 구성됩니다.

$$
x_t = t x_0 + (1 - t)\epsilon, \quad \epsilon \sim \mathcal{N}(0, \sigma_{\text{noise} }^2 I), \quad t \in (0, 1)
$$

모든 loop 예측은 같은 노이즈 입력과 timestep을 공유하며, 같은 clean image $$x_0$$에 대해 supervise됩니다.

$$
\ell_n = \mathbb{E}\left[ \frac{\lVert \hat{x}_0^{(n)} - x_0 \rVert_2^2}{d_x \, c(t)^2} \right], \quad c(t) = \max\{1 - t, \tau\}
$$

$$d_x$$는 이미지 원소 수이고, $$\tau$$는 t가 1에 가까워질 때 loss 가중치가 발산하지 않도록 막는 양수입니다.
전체 학습 목적함수는 loop 깊이별 loss의 가중합입니다.

$$
\mathcal{L} = \sum_{n=1}^{N} w_n \ell_n, \quad w_n \ge 0
$$

$$w_n$$은 loop n에 대한 supervision 강도를 조절합니다.
중간 예측은 공유된 post-loop 단계 C를 재사용하고 학습 시에만 디코딩되므로, Deep Supervision은 추가 파라미터나 추론 오버헤드를 발생시키지 않습니다.

### Self-Modulating Attention

표준 softmax attention은 source 토큰들의 상대적 기여도는 조절하지만 결과 업데이트의 강도 자체는 조절하지 않습니다.
같은 공유 블록이 loop마다 반복 적용되면 attention 업데이트가 중복되어 이미지 토큰 표현을 덮어쓰거나 약화시킬 수 있습니다.
Self-Modulating Attention(SMA)은 현재 hidden state에 기반해 반복 업데이트를 조절합니다.

토큰 i의 attention head h 출력은 $$o_{i,h} = \sum_j \alpha_{ij,h} v_{j,h}$$이며, 합은 이미지와 텍스트 토큰 전체에 걸쳐 계산됩니다.
두 메커니즘 모두 head concatenation과 output projection 이전에 head 출력을 $$z_{i,h} = G_{i,h} o_{i,h}$$로 변조합니다.
따라서 SMA는 attention 가중치 자체는 바꾸지 않고 각 head가 residual stream에 쓰는 내용만 바꿉니다.

첫 번째 구현은 Gated Attention입니다.
정규화된 hidden state $$u_i$$로부터 토큰별 scalar gate를 계산합니다.

$$
G_{i,h}^{\text{gate} } = \sigma\left(w_{g,h}^\top u_i + b_{g,h}\right)
$$

$$w_{g,h}$$, $$b_{g,h}$$는 이미지와 텍스트 토큰에 대해 별도로 존재하는 modality-specific gate 파라미터이며, 이 scalar가 각 head의 residual 업데이트 기여 강도를 명시적으로 제어합니다.

두 번째 구현은 Exclusive Self Attention(XSA)입니다.
토큰 자신의 value 방향 성분을 제거하는 projection 연산자로 변조합니다.

$$
G_{i,h}^{\text{xsa} } = I - \hat{v}_{i,h} \hat{v}_{i,h}^\top, \quad \hat{v}_{i,h} = \frac{v_{i,h} }{\lVert v_{i,h} \rVert_2}
$$

$$G_{i,h}^{\text{xsa} } v_{i,h} = 0$$이므로 직접적인 self-value 기여가 제거되며, 결과는 다음과 같습니다.

$$
z_{i,h}^{\text{xsa} } = \sum_{j \neq i} \alpha_{ij,h} \, G_{i,h}^{\text{xsa} } v_{j,h}
$$

부록에 따르면 XSA는 attention 출력을 단순히 $$(1 - \alpha_{ii,h})$$배로 축소하는 것과 동등하지 않습니다.
자기 자신의 value 기여뿐 아니라 다른 value 벡터 중 자기 value 방향에 놓인 성분까지 제거하기 때문입니다.
또한 직교 projection이므로 $$\lVert z_{i,h}^{\text{xsa} } \rVert_2 \le \lVert o_{i,h} \rVert_2$$가 성립해, output projection 이전 개별 head 출력 수준에서 non-expansive합니다.

| 구분 | Gated Attention | Exclusive Self Attention (XSA) |
|------|-----------------|-------------------------------|
| 변조 방식 | 학습 가능한 명시적 scalar gate | 파라미터 없는 상태 의존 projection |
| 제어 대상 | head 출력의 크기 | 자기 value 방향 성분 제거 |
| 추가 파라미터 | modality별 gate 파라미터 | 없음 |
| 적용 위치 | looped 단계 B 내부만 | looped 단계 B 내부만 |

두 메커니즘 모두 반복 attention 업데이트를 조절하기 위해 설계되었으므로 looped 단계 B에만 적용되고, pre-loop과 post-loop 단계는 표준 attention을 사용합니다.
B의 파라미터가 공유되므로 SMA의 학습 파라미터도 공유되지만, 변조 값 자체는 매 loop마다 현재 hidden state로부터 다시 계산됩니다.

## 실험 설정

### 모델 구성

Patch size에 따라 B/32와 B/16 두 변형을 학습했습니다.
17개 MMDiT 블록을 6/5/6으로 분할하고 가운데 5개 블록을 N = 4회 반복합니다.
이로써 디노이징 스텝당 유효 블록 적용 횟수는 32회이지만 고유 파라미터 블록은 17개만 유지됩니다.
연구진은 자원 제약으로 이 분할과 학습 loop 깊이 4를 철저히 최적화된 값이 아닌 단순 기본값으로 사용했다고 밝혔습니다.

| 설정 | Looped-DiT B/32 | Looped-DiT B/16 |
|------|-----------------|-----------------|
| 이미지 해상도 | 512 × 512 | 512 × 512 |
| Patch size | 32 | 16 |
| 이미지 토큰 수 | 256 | 1024 |
| Joint 시퀀스 길이 | 512 | 1280 |
| Hidden dimension | 768 | 768 |
| 고유 MMDiT 블록 | 17 | 17 |
| Pre / loop / post 블록 | 6 / 5 / 6 | 6 / 5 / 6 |
| 학습 loop 깊이 | 4 | 4 |
| 유효 블록 적용 횟수 | 32 | 32 |
| Attention heads / head dim | 12 / 64 | 12 / 64 |
| SwiGLU hidden dimension | 2048 | 2048 |
| 최대 텍스트 길이 | 256 | 256 |
| 파라미터 | 260.2M | 258.1M |

원래 MiniT2I의 embedding layer, pre-RMSNorm, query/key normalization, rotary position embedding을 유지하며 명시적 timestep conditioning은 사용하지 않습니다.
이미지와 텍스트 스트림은 별도 경로로 처리되고 joint attention으로 상호작용합니다.

### 학습 설정과 데이터

모든 모델은 pixel space에서 flow-matching 목적함수로 직접 학습합니다.
노이즈는 $$\mathcal{N}(0, 4I)$$에서 샘플링하고, t는 μ = -0.8, σ = 0.8인 logit-normal 분포에서 샘플링합니다.
타깃과 예측 velocity는 다음과 같이 정의하고 둘 사이의 MSE를 최소화합니다.

$$
v = \frac{x - x_t}{\max(1 - t, 0.05)}, \quad \hat{v} = \frac{\hat{x}_0 - x_t}{\max(1 - t, 0.05)}
$$

학습 시 noise scale 2.0을 사용하고, classifier-free guidance를 위해 텍스트 조건을 확률 0.1로 drop합니다.
Deep Supervision은 loop 깊이 1, 2, 3의 예측에 적용되며, 각 중간 hidden state는 공유된 6개 post-loop 블록과 최종 정규화·예측 층을 거쳐 loop 깊이 4의 최종 예측과 같은 타깃으로 최적화됩니다.
B/32 실험에서 final+mean 가중치 (1/3, 1/3, 1/3, 1)가 exponential 가중치 (1/8, 1/4, 1/2, 1)보다 좋아 B/16에도 final+mean을 채택했습니다.

| 하이퍼파라미터 | B/32 Pretrain | B/32 Fine-tune | B/16 Pretrain | B/16 Fine-tune |
|---------------|---------------|----------------|---------------|----------------|
| 학습 스텝 | 250K | 40K | 500K | 80K |
| Global batch size | 1024 | 1024 | 1024 | 1024 |
| Nodes × GPUs | 2×8 | 2×8 | 4×8 | 4×8 |
| Optimizer | AdamW | AdamW | AdamW | AdamW |
| Adam (β1, β2) | (0.9, 0.95) | (0.9, 0.95) | (0.9, 0.95) | (0.9, 0.95) |
| Peak learning rate | 4e-4 | 4e-4 | 4e-4 | 4e-4 |
| Warmup | 5K (1e-6에서 시작) | 없음 | 5K (1e-6에서 시작) | 없음 |
| LR schedule | Warmup 후 Constant | Constant | Warmup 후 Constant | Constant |
| Weight decay | 0 | 0 | 0 | 0 |
| Gradient clipping | 0.1 | 0.1 | 0.1 | 0.1 |
| EMA decay | 0.99995 | 0.99995 | 0.99995 | 0.99995 |
| Condition dropout | 0.10 | 0.10 | 0.10 | 0.10 |
| Noise scale | 2.0 | 2.0 | 2.0 | 2.0 |

학습 데이터는 MiniT2I와 동일하게 CC12M으로 pretrain하고 BLIP3o-60K, DALL-E 3, ShareGPT-4o-Image 혼합 데이터로 fine-tune합니다.
연구진에 따르면 공개된 논문·코드·데이터 기준으로 Looped-DiT와 MiniT2I는 Table 1 비교 모델 중 가장 적은 학습 데이터를 사용하며, 학습 데이터 규모를 공개한 나머지 모델은 모두 최소 2배 이상의 데이터를 사용했습니다.
이미지는 짧은 변을 512 픽셀로 리사이즈한 뒤 512 × 512로 center-crop하고 [-1, 1]로 정규화하며, random flip 등 추가 augmentation은 사용하지 않습니다.

### 평가 벤치마크와 추론 설정

B/16은 6개 벤치마크로 평가합니다.

| 벤치마크 | 평가 대상 |
|---------|----------|
| GenEval | counting, color, spatial relation 등 객체 중심 compositional alignment |
| DPG-Bench | 여러 객체·속성·관계가 포함된 dense prompt 준수 |
| TIIF-Bench (Short) | 다양한 복잡도의 세밀한 instruction following, short-prompt split만 사용 |
| T2I-CoReBench | 복잡한 composition과 multi-step reasoning |
| PRISM-Bench | 다양한 고난도 생성 과제에서의 prompt-image alignment와 reasoning |
| SpatialGenEval | 정보 밀도가 높은 장면에서의 공간 이해와 reasoning |

TIIF는 학습이 최대 256 토큰의 짧은 캡션 위주이므로 long-prompt 설정이 학습 조건을 잘 대표하지 못한다는 이유로 short split만 사용했습니다.
B/32 분석과 ablation은 visual reasoning과 직접 관련된 DPG-Bench, T2I-CoReBench, PRISM-Bench, SpatialGenEval 4개 벤치마크로 수행하며, 평균은 B/16은 6개, B/32는 4개 벤치마크 기준입니다.

추론은 EMA 가중치 체크포인트를 사용하고, Euler sampling 100 디노이징 스텝, classifier-free guidance scale 6.0, loop 깊이 N = 4가 기본 설정입니다.
샘플링은 $$\mathcal{N}(0, 4I)$$에서 초기화하며, 서로 다른 seed로 3회 독립 평가한 평균을 보고합니다.
지연 시간은 단일 NVIDIA H100 GPU, batch size 1에서 측정하며 warm-up 생성 몇 개를 제외한 뒤 이후 100회 생성의 평균을 보고합니다.

## 주요 결과

### 파라미터 효율

각 모델의 공식 추론 설정으로 최신 T2I 모델들과 비교한 결과입니다.

| 모델 | Params | GenEval | DPG | PRISM | CoRe | Spatial | TIIF-Short | Average |
|------|--------|---------|-----|-------|------|---------|------------|---------|
| E-MMDiT | 0.30B | 69.1 | 80.4 | 51.6 | 30.7 | 45.7 | 63.4 | 56.8 |
| SANA-0.6B | 0.59B | 65.8 | 83.3 | 56.4 | 39.8 | 47.4 | 69.4 | 60.4 |
| DeCo-XXL/16 | 1.1B | 82.0 | 82.0 | 52.6 | 34.8 | 49.8 | 70.4 | 61.9 |
| URSA-0.6B | 0.86B | 64.1 | 85.6 | 59.6 | 43.4 | 52.0 | 67.6 | 62.1 |
| TiM-T2I | 0.87B | 82.8 | 83.2 | 51.4 | 36.7 | 48.2 | 70.6 | 62.2 |
| DreamLite | 0.39B | 71.6 | 85.2 | 55.5 | 42.2 | 53.9 | 72.6 | 63.5 |
| CogView4 | 6.4B | 73.0 | 85.1 | 60.9 | 45.0 | 53.3 | 66.9 | 64.0 |
| MiniT2I-B/16 | 0.26B | 87.5 | 84.1 | 55.6 | 44.0 | 52.0 | 75.0 | 66.4 |
| MiniT2I-L/16 | 0.91B | 88.3 | 84.8 | 58.9 | 44.4 | 52.2 | 75.0 | 67.3 |
| UniLiP-3B | 1.6B | 90.3 | 83.4 | 58.5 | 45.0 | 51.6 | 78.2 | 67.8 |
| InternVL-U | 1.7B | 85.0 | 85.2 | 63.5 | 48.2 | 54.5 | 77.7 | 69.0 |
| GoT-R1 (CoT) | 6.9B | 72.5 | 84.0 | 56.4 | 42.2 | 50.0 | 74.1 | 63.2 |
| T2I-R1 (CoT) | 6.9B | 78.8 | 84.7 | 57.1 | 35.5 | 49.7 | 77.6 | 63.9 |
| Uni-CoT (CoT) | 6.6B | 81.2 | 84.1 | 58.8 | 45.8 | 53.8 | 76.8 | 66.8 |
| Looped-DiT B/16 | 0.26B | 87.4 | 87.0 | 67.0 | 53.5 | 54.6 | 79.7 | 71.5 |

Looped-DiT B/16은 DPG-Bench, PRISM, T2I-CoReBench, SpatialGenEval, TIIF-Short에서 최고 성능을 기록했습니다.
차순위 모델인 InternVL-U 대비 평균 점수가 2.5점 높으면서 파라미터는 6.5배 적습니다.
특히 reasoning 성격이 강한 CoReBench(53.5 vs 48.2, +5.3)와 PRISM(67.0 vs 63.5, +3.5)에서 격차가 큽니다.
같은 0.26B 규모의 원래 MiniT2I-B/16과 비교해도 평균이 66.4에서 71.5로 상승했습니다.
GenEval에서는 UniLiP-3B(90.3), MiniT2I-L/16(88.3), MiniT2I-B/16(87.5)이 Looped-DiT(87.4)보다 소폭 높습니다.

### 추론 효율

Looped-DiT B/16은 추론 FLOPs와 지연 시간 두 지표 모두에서 성능-효율 Pareto frontier를 형성합니다.
Loop 깊이와 디노이징 스텝 수를 조합하여 추론 예산을 유연하게 조절할 수 있습니다.

| 비교 항목 | InternVL-U | Looped-DiT B/16 | 차이 |
|----------|-----------|-----------------|------|
| 파라미터 | 1.7B | 0.26B | 6.5배 적음 |
| 연산량 | 732 TFLOPs | 150 TFLOPs | 4.9배 적음 |
| 지연 시간 | 2.1s | 0.65s | 3.2배 빠름 |

### 추론 능력과 자기 교정

정성적으로 Looped-DiT B/16은 서론에서 제시한 reasoning 요구 중 제약 해소와 자기 교정에 해당하는 행동을 보였습니다.
연속된 loop에 걸쳐 누락된 내용을 추가하고, 공간 제약을 만족하도록 객체를 재배치하며, 렌더링 오류를 교정하거나 불필요한 객체를 제거합니다.
예를 들어 벽에 거울, 웨이트 랙, 벽걸이 수납장을 배치하라는 프롬프트나, 'STOP' 표지판·'HELLO' 문구 코트·'EXPRESS' 로고 트럭을 포함하라는 프롬프트, 다섯 가지 색의 찻잔을 요구하는 프롬프트에서 loop가 진행될수록 결과가 교정되었습니다.
반면 같은 프롬프트에서 더 큰 non-looped 모델(InternVL-U)과 명시적 CoT 모델(Uni-CoT)에는 오류가 남아 있었습니다.
정량적으로도 loop 수를 1에서 4로 늘릴수록 성능이 일관되게 향상되어 점진적 정제를 뒷받침합니다.

## Looping은 꼭 필요한가

Looping의 이점이 looped computation 고유의 것인지, 아니면 다른 전략으로도 얻을 수 있는지를 B/32로 검증했습니다.
비교 대상은 모델 깊이·폭 증가, 디노이징 스텝 추가, 명시적 CoT 도입입니다.

### 동일 파라미터와 동일 연산량 비교

Deeper baseline은 5개 공유 블록을 4번 통과하는 대신 20개의 서로 다른 블록을 사용해 유효 깊이 32를 맞췄고, Wider baseline은 hidden dimension을 늘려 forward 연산량을 근사적으로 맞췄습니다.
학습 연산량 매칭을 위해 Deeper 모델에 Deep Supervision을 추가한 baseline도 포함했습니다.

| 모델 | Train GFLOPs | Inference GFLOPs | Params (M) | 유효 깊이 | Hidden dim | Avg. |
|------|--------------|------------------|------------|----------|------------|------|
| MiniT2I B/32 | 441 | 146 | 260 | 17 | 768 | 55.2 |
| Deeper MiniT2I B/32 | 809 | 267 | 473 | 32 | 768 | 57.7 (+2.6) |
| Wider MiniT2I B/32 | 805 | 268 | 489 | 17 | 1056 | 58.6 (+3.5) |
| Deeper MiniT2I B/32 w. DeepSup | 1,246 | 267 | 473 | 32 | 768 | 58.1 (+3.0) |
| Looped-DiT B/32 | 1,246 | 267 | 260 | 32 | 768 | 59.1 (+4.0) |

Looped-DiT B/32는 파라미터가 같은 baseline 대비 평균 4.0점 높고, 더 적은 파라미터로 추론 연산량을 맞춘 두 baseline도 모두 능가합니다.
학습 연산량을 맞춘 경우에도 59.1로 Deeper baseline(58.1)보다 높습니다.

### Loop 깊이 vs 디노이징 스텝

4-loop 추론을, 같은 추론 FLOPs를 맞추도록 디노이징 스텝을 늘린 single-pass 추론과 비교했습니다.
Single-pass baseline은 두 가지로, 하나는 같은 체크포인트에서 looping만 끈 것(학습된 가중치와 학습 이력 고정)이고 다른 하나는 looping 없이 학습한 동일 아키텍처입니다.
25스텝에서 50스텝 설정 범위에서, 연산을 loop 깊이에 할당하는 쪽이 디노이징 스텝에 할당하는 쪽보다 DPG, PRISM, CoRe, Spatial 전반에서 일관되게 높은 성능을 보였습니다.
연구진은 이를 looping이 추론 연산을 더 효과적으로 사용하는 방식이라는 근거로 제시합니다.

### Textual CoT와의 상보성

Non-looped 모델에 대한 CoT 기반 프롬프트 재작성(CoT), 원래 프롬프트로 looping(Looping), 두 가지 결합(Both)을 비교했습니다.
프롬프트 재작성에는 Qwen3를 사용했으며, 관련 없는 디테일은 추가하지 말고 의도한 내용을 명시하고 공간 배치를 명확히 하도록 지시했습니다.
아래 수치는 원래 프롬프트를 쓴 non-looped 모델 대비 향상폭입니다.

| 능력 유형 | 세부 과제 | CoT | Looping | Both |
|----------|----------|-----|---------|------|
| 벤치마크 평균 | - | +2.1 | +4.0 | +5.5 |
| 제약 해소 (Looping 우세) | PRISM / Long Text | +3.1 | +9.1 | +9.4 |
| 제약 해소 (Looping 우세) | CoRe / Multi-Relation | +1.2 | +8.3 | +8.4 |
| 제약 해소 (Looping 우세) | CoRe / Procedural | +4.7 | +9.4 | +9.6 |
| 제약 해소 (Looping 우세) | Spatial / Orientation | +0.0 | +6.0 | +6.2 |
| 제약 해소 (Looping 우세) | Spatial / Motion | +0.1 | +6.0 | +6.3 |
| 암시된 내용 추론 (CoT 우세) | CoRe / Generalization | +22.2 | +7.5 | +24.5 |
| 암시된 내용 추론 (CoT 우세) | CoRe / Hypothetical | +9.4 | +3.5 | +11.6 |
| 암시된 내용 추론 (CoT 우세) | CoRe / Reconstructive | +6.2 | +3.4 | +8.9 |

관계적·절차적·공간적 제약을 동시에 만족해야 하는 과제에서는 looping의 이득이 크고, CoT를 더해도 추가 이득이 작습니다.
반대로 예시와 규칙으로부터 암시된 시각적 내용을 추론해야 하는 과제에서는 CoT의 이득이 큽니다.
둘을 결합하면 가장 높은 종합 점수를 얻어, 시각 생성의 서로 다른 측면에서 상보적임을 시사합니다.

## Ablation Study

### 구성 요소별 기여

B/32에서 looping, Deep Supervision, Self-Modulating Attention의 기여를 분리했습니다.
DeepSup 열의 "final only"는 최종 loop만 supervise하는 설정입니다.

| Looping | DeepSup | Attention | DPG | PRISM | CoRe | Spatial | Avg. |
|---------|---------|-----------|-----|-------|------|---------|------|
| 미적용 | - | Standard | 82.0 | 51.2 | 39.2 | 48.3 | 55.2 |
| 적용 | final only | Standard | 84.4 | 51.1 | 39.3 | 49.9 | 56.2 |
| 적용 | Exponential | Standard | 84.3 | 52.1 | 41.0 | 51.0 | 57.1 |
| 적용 | Final + Mean | Standard | 84.4 | 53.9 | 40.6 | 50.9 | 57.5 |
| 적용 | final only | Gated | 84.0 | 53.0 | 39.5 | 49.7 | 56.6 |
| 적용 | final only | XSA | 84.2 | 54.5 | 43.2 | 52.1 | 58.5 |
| 적용 | Final + Mean | XSA | 85.3 | 54.4 | 44.5 | 52.3 | 59.1 |

Exponential 가중치는 (w1, w2, w3, w4) = (1/8, 1/4, 1/2, 1)로 앞쪽 loop에 지수적으로 작은 가중치를 주고, Final + Mean은 (1/3, 1/3, 1/3, 1)로 최종 loop loss에 앞쪽 loop들의 평균 loss를 더합니다.
Looping만으로도 baseline보다 개선되며, 두 Deep Supervision 가중치 방식 모두 final-only보다 좋고 그중 Final + Mean이 더 우수합니다.
Self-Modulating Attention은 Gated와 XSA 모두 비변조 attention보다 개선되며 XSA의 이득이 더 큽니다.
Final + Mean과 XSA를 결합한 설정이 평균 59.1로 가장 높습니다.

### Deep Supervision과 Adaptive Looping

4 loop로 학습한 모델을 1에서 8 loop로 추론했을 때, Deep Supervision은 모든 loop 수에서 성능을 개선했습니다.
조기 종료(early exit) 시 성능 저하를 크게 줄이고, 학습 시 사용한 4 loop를 넘어서도 성능을 유지합니다.
특히 loop로 학습한 모델의 단일 loop 설정조차 표준 non-looped baseline을 능가하며, 같은 모델에서 추론 loop를 추가하면 성능이 더 오릅니다.

이처럼 loop 수에 강건한 특성은 adaptive looping을 가능하게 합니다.
각 loop 후 종료 여부를 결정하는 경량 gating network를 도입하는데, 4개의 learnable query로 post-loop activation에서 정보를 추출하는 cross-attention 층과, r번째 loop 이후 계속했을 때의 기대 이득 $$\hat{y}_r$$을 예측하는 MLP head로 구성됩니다.
학습 타깃은 추가 loop당 달성 가능한 최대 미래 loss 감소량으로 정의됩니다.

$$
y_r = \max\left(0, \max_{k = r+1, \dots, N} \frac{\ell_r - \ell_k}{k - r}\right), \quad r = 1, 2, \dots, N - 1
$$

$$\ell_k$$는 k번째 loop 예측과 정답 이미지 사이의 reconstruction error입니다.
이 타깃은 마지막 pass가 가장 정확하다고 가정하지 않으며, 이후 어떤 exit도 현재 예측보다 낫지 않으면 0으로 설정됩니다.
추론 시 threshold λ를 정하고 $$\hat{y}_r$$이 λ보다 큰 동안 loop를 계속하며, λ 이하가 되는 첫 loop에서 멈춥니다.
λ를 바꾸면 재학습 없이 연산 예산에 맞춰 평균 loop 수를 조절할 수 있습니다.

정성적으로 adaptive looping은 프롬프트 요구가 충족될 때까지 필요한 만큼만 loop를 사용하고 이득이 적은 추가 연산은 생략합니다.
논문의 예시에서는 프롬프트에 따라 adaptive looping이 사용한 loop 수가 2.84, 1.03처럼 달라졌습니다.
정량적으로는 같은 평균 loop 수에서 고정 loop 추론보다 성능이 높고, 적은 loop 설정(1.25 loop)에서 성능 하락을 절반으로 줄이며, 비슷한 성능 plateau에 더 일찍(2.70 loop) 도달했습니다.

### Self-Modulating Attention 분석

XSA를 looped 모델과 non-looped baseline에 각각 적용했을 때의 이득을 비교했습니다.
Non-looped 깊이 17과 32는 각각 looped 모델의 파라미터 수와 forward FLOPs에 맞춘 값입니다.

| 설정 | 유효 깊이 | XSA 미적용 | XSA 적용 | XSA 이득 |
|------|----------|-----------|---------|---------|
| Non-looped, parameter-matched | 17 | 55.2 | 55.1 | -0.1 |
| Non-looped, compute-matched | 32 | 58.1 | 58.8 | +0.7 |
| Looped | 32 | 57.5 | 59.1 | +1.6 |

XSA는 looped 모델에서 가장 큰 이득(+1.6)을 보여, 반복 업데이트 상황에서 변조가 특히 중요함을 시사합니다.
연구진은 비변조 attention이 loop마다 과도한 업데이트를 만들어 중요한 토큰 정보를 점진적으로 덮어쓴다는 가설을 세웠고, 두 가지 분석이 이를 뒷받침합니다.

- Position probe R²: loop 수가 늘수록 비변조 attention은 토큰 위치 복원력이 가장 크게 떨어지고, XSA는 가장 높은 위치 복원력을 유지하며, Gated는 그 중간
- 상대적 attention 업데이트 크기(attention update norm / residual stream norm): 비변조 attention이 가장 크고 XSA가 가장 작으며, Gated는 그 중간

이 순서는 Ablation 표의 벤치마크 성능 순서와도 일치합니다.
정성 예시로 "장난감 코끼리 2마리와 흰 풍선 4개" 프롬프트에서, XSA가 없으면 이미 올바른 이미지를 이후 loop가 계속 수정해 2번째 loop부터 풍선이 5개로 늘어났습니다.
XSA를 적용하면 이후 업데이트가 억제되어 4번째 loop까지 풍선 4개라는 기존 구조가 유지되었습니다.

## 학습 및 추론 비용

B/32 기반 각 설정의 학습·추론 비용입니다.
학습 GFLOPs는 한 샘플의 forward·backward와 Deep Supervision exit를 포함한 1 학습 스텝 기준이고, 추론 GFLOPs는 디노이징 forward pass 1회 기준입니다.
괄호 안은 parameter-matched MiniT2I-B/32 대비 배수입니다.

| 설정 | Params | Training GFLOPs | Inference GFLOPs |
|------|--------|-----------------|------------------|
| Compute-matched (Deeper) | 473M (1.82×) | 809 (1.83×) | 267 (1.83×) |
| Compute-matched (Wider) | 489M (1.88×) | 805 (1.83×) | 268 (1.84×) |
| Parameter-matched (MiniT2I-B/32) | 260M (1.00×) | 441 (1.00×) | 146 (1.00×) |
| + Looping | 260M (1.00×) | 809 (1.83×) | 267 (1.83×) |
| + XSA | 260M (1.00×) | 809 (1.83×) | 267 (1.83×) |
| + DeepSup | 260M (1.00×) | 1,246 (2.83×) | 267 (1.83×) |
| + DeepSup + XSA (ours) | 260M (1.00×) | 1,246 (2.83×) | 267 (1.83×) |

Deep Supervision은 loop 깊이 1에서 3의 중간 state를 공유된 6블록 post-loop 단계로 디코딩하므로 학습 연산량을 늘립니다.
전체 모델은 샘플당 학습 스텝 1,246 GFLOPs로 Deeper(809), Wider(805) baseline의 약 1.54배입니다.
다만 이 오버헤드는 학습에만 한정되며, 추론 시에는 중간 예측을 계산하지 않으므로 파라미터와 추론 연산량이 추가되지 않습니다.
XSA의 연산 오버헤드는 무시할 수준이며 보고된 GFLOPs를 바꾸지 않습니다.
본문의 compute-matched 비교는 학습이 아닌 추론 연산량 기준입니다.

## 관련 연구

| 분야 | 주요 연구 | 본 연구와의 관계 |
|------|----------|-----------------|
| T2I Diffusion Transformer | DiT, PixArt-α, Stable Diffusion 3(MMDiT), SANA, Qwen-Image | 아키텍처 설계·표현 학습·모델 스케일링 대신 공유 블록 반복으로 연산 깊이를 늘림 |
| Few-step / One-step 생성 | Progressive distillation, Consistency Models, LCM, InstaFlow, SDXL-Turbo, SDXL-Lightning, DMD, Hyper-SD, MeanFlow, Drifting Models | 외부 정제 스텝 수를 줄이는 대신 반복 연산을 loop 깊이로 내재화, 샘플링 스텝과 hidden-state 정제 간 추론 연산 배분의 상보적 축 제시 |
| Looped Transformer | Universal Transformers, ALBERT, Coconut, Relaxed Recursive Transformers, Mixture-of-Recursions, MeSH, SpiralFormer, SMELT | 주로 언어 모델링에서 연구된 looping을 T2I로 확장 |
| 시각 생성 Looping | Elastic Looped Transformers (ELT) | 클래스 조건부 이미지·비디오 생성 대상, intra-loop self-distillation 사용 |

가장 가까운 연구인 ELT는 클래스 조건부 이미지·비디오 생성에서 looping을 다루며 추론 시 loop 수 변경에 초점을 둡니다.
ELT도 이른 loop exit에서 유사한 성능 저하를 관찰했고, 얕은 loop 구성이 최대 loop 구성을 모방하도록 학습하는 intra-loop self-distillation으로 해결했습니다.
반면 Looped-DiT의 Deep Supervision은 teacher 구성이나 distillation 목적함수 없이 최종 예측과 같은 목적함수로 중간 loop 예측을 직접 최적화합니다.
또한 ELT는 T2I 생성을 다루지 않으며, 본 연구는 단일 클래스 레이블이 아닌 다양한 자연어 제약을 해석·충족해야 하는 open-ended T2I 생성으로 looping을 확장하고 latent visual reasoning 가능성을 탐구합니다.

## 한계와 향후 과제

- 규모와 설정의 제한: 실험은 약 260M 파라미터, 512 × 512 해상도의 MiniT2I 기반 MMDiT에 집중되어 있습니다. 훨씬 큰 모델, latent-space 아키텍처, 고해상도 생성으로 결과가 어떻게 확장되는지는 불분명합니다.
- 설계 공간 탐색 부족: 자원 제약으로 [6, 5, 6] 분할과 학습 loop 깊이 4를 고정해 사용했습니다. Loop 위치, 학습 깊이, 공유 블록 비율이 모델 규모·추론 예산과 어떻게 상호작용하는지는 향후 과제입니다.
- 학습 비용 증가: Deep Supervision은 중간 loop state를 추가로 디코딩하므로 학습 연산량이 늘어납니다. 추론에는 오버헤드가 없지만, 중간 supervision의 학습 비용을 줄이거나 더 효율적인 목적함수를 개발할 여지가 있습니다.
- Latent reasoning 근거의 성격: Latent visual reasoning에 대한 근거는 주로 행동적·표현적 분석입니다. Loop 간 점진적 오류 교정은 반복적 reasoning과 일치하지만, 이러한 행동이 어떻게 나타나는지에 대한 완전한 메커니즘 설명은 아닙니다. Loop 간 정보 흐름과 연산에 대한 직접 분석이 필요합니다.

## 결론

Looped-DiT는 각 디노이징 스텝 안에서 공유 Transformer 블록을 반복 적용해 파라미터 증가 없이 연산 깊이를 늘리는 T2I 모델입니다.
연구진은 naive looping이 불안정하고 공간 정보를 점진적으로 손상시킨다는 점을 보이고, 이를 Deep Supervision과 Self-Modulating Attention으로 해결했습니다.
통제된 비교를 통해 성능 향상이 모델 깊이·폭 확장, 같은 추론 연산의 디노이징 스텝 재배분, 명시적 텍스트 reasoning 도입만으로는 설명되지 않음을 확인했습니다.
260M 파라미터 모델이 6.5배 큰 InternVL-U를 평균 2.5점 앞서면서 추론 연산량은 4.9배 적다는 결과는 looping이 파라미터 효율적인 스케일링 축이 될 수 있음을 보여 줍니다.
또한 연속된 loop가 제약을 해소하고 이전 오류를 교정하는 모습은 latent visual reasoning을 시사하며, 향후 T2I 모델 개선을 위한 유망한 방향으로 looping을 제시합니다.

## Reference

- [Looped Diffusion Transformer (arXiv 2609.40305)](https://arxiv.org/abs/2609.40305)
