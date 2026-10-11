---
layout: post
title: "Test-Time Training(TTT)과 LLM 메모리: fast weights, continual learning, 손실 압축으로 읽는 개념 정리"
author: 'Juho'
date: 2026-10-11 00:00:00 +0900
categories: [LLM]
tags: [LLM, AI, Context, RNN]
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
2. [TTT라는 용어의 세 가지 용법](#ttt라는-용어의-세-가지-용법)
   - [이 글이 다루는 계열](#이-글이-다루는-계열)
3. [TTT의 계보와 동작 방식](#ttt의-계보와-동작-방식)
   - [원래의 TTT와 dynamic evaluation](#원래의-ttt와-dynamic-evaluation)
   - [TTT layer](#ttt-layer)
   - [Titans와 Miras](#titans와-miras)
   - [LaCT](#lact)
   - [Nested Learning과 Hope](#nested-learning과-hope)
   - [TTT-E2E](#ttt-e2e)
   - [두 겹의 루프](#두-겹의-루프)
4. [fast weights와 TTT의 관계](#fast-weights와-ttt의-관계)
5. [continual learning과의 비교](#continual-learning과의-비교)
6. [어텐션과의 관계](#어텐션과의-관계)
7. [손실 압축 관점과 망각](#손실-압축-관점과-망각)
   - [압축이라는 서술](#압축이라는-서술)
   - [용량 상한](#용량-상한)
   - [세부 회상 실험 결과](#세부-회상-실험-결과)
   - [설계 요소로서의 망각](#설계-요소로서의-망각)
8. [가중치 안에 암묵적으로 저장된다는 비판](#가중치-안에-암묵적으로-저장된다는-비판)
9. [비용과 재현성 그리고 서빙과 안전](#비용과-재현성-그리고-서빙과-안전)
   - [비용](#비용)
   - [재현성과 공개 구현](#재현성과-공개-구현)
   - [서빙과 사업 모델](#서빙과-사업-모델)
   - [안전과 오염](#안전과-오염)
10. [결론](#결론)
11. [Reference](#reference)

## 개요

Test-Time Training(TTT)은 추론 중에 모델의 일부 가중치를 갱신해, 읽은 컨텍스트를 가중치 안에 저장하는 방식입니다.
최근에는 LLM의 메모리와 장기 컨텍스트 문제를 다루는 수단으로 연구되고 있습니다.
이 글에서는 용어 구분, 계보와 수식, fast weights 및 continual learning과의 관계, 어텐션과의 관계, 손실 압축과 망각, 가중치 저장의 암묵성 비판, 비용과 재현성을 차례로 정리합니다.

이 글에서 인용한 영어 구절은 원문에서 발췌된 짧은 조각이며, 표와 수식은 조사 시점에 확인된 값과 조건을 그대로 옮긴 것입니다.

## TTT라는 용어의 세 가지 용법

Test-Time Training이라는 이름은 서로 다른 세 가지 용법으로 쓰입니다.

| 계열 | 대표 논문 | 테스트 시점에 갱신되는 것 | 갱신의 수명 | 목적 |
|------|-----------|---------------------------|-------------|------|
| 분포 이동 대응 | Sun 외 2020 | 공유 특징 추출기 | 표준형은 샘플마다 리셋, 온라인형은 다음 샘플로 이월 | 테스트 분포 적응 |
| 태스크별 임시 미세조정 | Akyürek 외 2024 | 태스크별 LoRA 어댑터 | 태스크 단위의 일시적 갱신 | few-shot 추론 |
| 시퀀스 모델의 은닉 상태를 가중치로 사용 | Sun 외 2024, Titans, LaCT, TTT-E2E | 레이어 내부의 fast weight | 시퀀스 안에서 토큰 또는 청크마다 | 장기 컨텍스트 기억 |

두 번째 계열의 [Akyürek 외 논문](https://ar5iv.labs.arxiv.org/html/2411.07279){:target="_blank"}은 TTT를 "temporarily updating model parameters during inference using a loss derived from in-context examples."로 정의합니다.
이 논문은 각주에서 RNN 은닉 상태 용법, 즉 세 번째 계열과 자신들의 TTT가 다르다고 구분합니다.
이 논문의 현재 arXiv 제목은 "The Surprising Effectiveness of Test-Time Training for Few-Shot Learning"이며, 초기 버전 제목이 "for Abstract Reasoning"이었다는 점은 검색 결과로만 확인되었습니다.

같은 혼선은 개인 블로그에서도 지적됩니다.
원래의 TTT는 옵티마이저가 갱신하는 slow weights를 테스트 시점에 고치는 것이고, 최근의 TTT layer 계열은 순전파 중에 만들어지는 fast weights를 다룬다는 구분입니다.
이 글은 용어 사용이 뒤섞여 있다는 점을 "the general messiness around 'test time training'"이라고 표현합니다([Ian Barber, Test Time Training](https://ianbarber.blog/2026/09/02/test-time-training/){:target="_blank"}).

### 이 글이 다루는 계열

LLM의 메모리와 장기 컨텍스트 문제에서 말하는 TTT는 세 번째 계열입니다.
2020년의 분포 이동 대응 기법이 아니라, 2024년 이후의 시퀀스 모델링용 TTT를 가리킵니다.

이 계열은 TTT layer(2024), Titans(2024년 12월 arXiv 제출), LaCT(2025), Nested Learning과 Hope(2025), TTT-E2E(2025년 12월)로 이어집니다.

## TTT의 계보와 동작 방식

### 원래의 TTT와 dynamic evaluation

[Sun 외(2020)](https://ar5iv.labs.arxiv.org/html/1909.13231){:target="_blank"}의 원래 TTT는 분포 이동에 취약한 지도학습 모델을 대상으로 합니다.
라벨 없는 테스트 샘플 하나가 그 샘플이 뽑힌 분포에 대한 단서를 준다는 점을 이용합니다.
구조는 공유 특징 추출기 `θ_e`, 메인 태스크 헤드 `θ_m`, 자기지도 헤드 `θ_s`로 이루어지며, 보조 과제는 4방향 회전 예측입니다.

```text
결합 학습 (Eq. 2):
  min over (θ_e, θ_m, θ_s) of (1/n) Σ_i [ l_m(x_i, y_i; θ_m, θ_e) + l_s(x_i; θ_s, θ_e) ]

테스트 시점 갱신 (Eq. 3):
  min over θ_e of l_s(x; θ_s, θ_e)
```

표준형은 테스트 샘플마다 학습된 파라미터에서 다시 시작하고 예측 후 갱신을 버립니다.
온라인형은 직전 샘플에서 갱신된 파라미터를 다음 샘플의 초기값으로 씁니다.
결과로는 CIFAR-10.1 오류율이 baseline 17.4%에서 TTT 15.9%로 내려갔고, ImageNet-C level 5 Gaussian noise 정확도가 baseline 1.3%에서 TTT-Online 26.3%로 올랐습니다.
저자들은 테스트가 일반 추론보다 `2 × batch size × iteration 수`배 느리다는 점과 CIFAR-10.1 개선폭이 작다는 점을 한계로 적었습니다.

시퀀스 모델링 쪽의 직접적 선행 연구는 [dynamic evaluation](https://ar5iv.labs.arxiv.org/html/1709.07432){:target="_blank"}입니다.
테스트 시퀀스를 세그먼트로 나누고, 각 세그먼트를 현재 파라미터로 평가한 뒤 그 손실의 gradient로 파라미터를 갱신하고 다음 세그먼트로 넘어갑니다.

```text
θ_i ← θ_{i−1} − η ∇L(s_i) + λ (θ_g − θ_l^{i−1})
```

두 번째 항은 학습 완료 시점 파라미터 `θ_g` 쪽으로 되돌리는 감쇠 항이며, 과거의 적응 갱신을 지수적으로 감쇠시킵니다.
AWD-LSTM 기준으로 PTB test perplexity가 57.7에서 51.1로, WikiText-2가 66.1에서 44.3으로 내려갔습니다.

### TTT layer

[Sun 외(2024)](https://arxiv.org/html/2407.04620v4){:target="_blank"}의 TTT layer는 은닉 상태를 "a machine learning model itself, and the update rule a step of self-supervised learning"으로 만든다는 아이디어에서 출발합니다.
은닉 상태 `s_t`는 내부 모델 `f`의 가중치 `W_t`와 같고, 갱신 규칙은 자기지도 손실에 대한 경사하강 한 스텝입니다.

```text
갱신 (Eq. 2):          W_t = W_{t−1} − η ∇ℓ(W_{t−1}; x_t)
자기지도 손실 (Eq. 4):  ℓ(W; x_t) = ‖ f(θ_K x_t; W) − θ_V x_t ‖²
출력 (Eq. 5):          z_t = f(θ_Q x_t; W_t)
```

`θ_K`는 training view, `θ_V`는 label view, `θ_Q`는 test view입니다.
내부 루프에서는 `W`만 최적화되고 `θ_K`, `θ_V`, `θ_Q`는 외부 루프의 파라미터입니다.
논문은 `W`를 "merely a hidden state, not a parameter."라고 부릅니다.
테스트 시점에도 입력 시퀀스마다 서로 다른 가중치 열 `W_1`부터 `W_T`까지가 학습됩니다.

구현체는 두 가지입니다.

- TTT-Linear는 `f`가 선형이고 `W`가 정방행렬이며 LayerNorm과 residual을 포함합니다.
- TTT-MLP는 `f`가 hidden 4배의 2층 MLP와 GELU이며 LayerNorm과 residual을 포함합니다.

구현 장치로는 미니배치 TTT(b = 16), 토큰별 gradient를 실체화하지 않는 dual form, 학습 가능한 `W_0`, 학습 가능한 학습률 `η(x) = η_base · σ(θ_lr · x)`가 쓰였습니다.
개념적 흐름은 논문의 Eq. 2, 4, 5를 그대로 옮기면 다음과 같습니다.

```text
W = W_0                                   # 외부 루프에서 학습된 초기값
for x_t in sequence:
    loss = || f(θ_K x_t; W) - θ_V x_t ||^2   # Eq. 4
    W    = W - η * grad_W(loss)              # Eq. 2
    z_t  = f(θ_Q x_t; W)                     # Eq. 5
```

기존 TTT와의 차이는 재구성 과제가 사람이 설계한 것이 아니라 외부 루프에서 학습된다는 점입니다.
기존 메타러닝과의 차이로는 "each sequence itself is a dataset and defines its own generalization problem."이라는 서술이 제시됩니다.

### Titans와 Miras

[Titans](https://ar5iv.labs.arxiv.org/html/2501.00663){:target="_blank"}는 어텐션을 단기 기억으로, 테스트 시점에 갱신되는 신경 메모리(1층 이상의 MLP)를 장기 기억으로 해석합니다.
메모리는 연상 기억 손실로 학습되고, gradient를 surprise로 해석합니다.

```text
연상 기억 손실 (Eq. 12):   ℓ(M_{t−1}; x_t) = ‖ M_{t−1}(k_t) − v_t ‖₂²
순간 surprise만 (Eq. 8):   M_t = M_{t−1} − θ_t ∇ℓ(M_{t−1}; x_t)
모멘텀 누적 (Eq. 14):      S_t = η_t S_{t−1} − θ_t ∇ℓ(M_{t−1}; x_t)
망각 게이트 (Eq. 13):      M_t = (1 − α_t) M_{t−1} + S_t
인출 (Eq. 15):            y_t = M*(q_t)
```

`α_t`는 0과 1 사이 값으로 얼마나 잊을지를 정하며, 0이면 유지이고 1이면 초기화입니다.
`η_t`는 "a data-dependent surprise decay"이고 `θ_t`는 순간 surprise의 반영 비율입니다.
논문은 이 메커니즘이 "equivalent to optimizing a meta neural network with mini-batch gradient descent, momentum, and weight decay"이며 "the weight decay can be interpreted as a gating mechanism to forget the past data."라고 설명합니다.
어텐션과 메모리의 결합 방식은 MAC(Memory as a Context), MAG(Memory as a Gate), MAL(Memory as a Layer) 세 가지입니다.

언어모델링 Wiki perplexity는 760M 파라미터, 30B 토큰 설정에서 Transformer++ 25.21 대비 Titans(MAG) 18.61로 보고되었습니다.
"larger than 2M context window size"는 초록의 주장이며, 조사 범위의 본문 텍스트에서 해당 길이의 실험 수치는 확인되지 않았습니다.
BABILong 수치는 그림에만 존재합니다.

[Miras](https://ar5iv.labs.arxiv.org/html/2504.13173){:target="_blank"}는 같은 그룹의 후속 프레임워크로, 메모리 구조, attentional bias(내부 목적함수), retention gate, 메모리 학습 알고리즘의 네 축으로 시퀀스 모델을 분류합니다.

```text
W_t = argmin over W of  ℓ̃_t(W; k_t, v_t) + Ret_t(W, W_{t−1})
Ret_t(W, W_{t−1}) = (1/η_t) D_t(W, W_{t−1}) + (1/α_t) G_t(W)
```

이 분류에서 Titans는 ℓ2 bias, ℓ2 retention, 모멘텀 경사하강의 조합이고, softmax attention은 retention 없는 ℓ2 손실의 비모수 해로 정리됩니다.

### LaCT

[LaCT](https://arxiv.org/html/2505.23884){:target="_blank"}("Test-Time Training Done Right")의 블록은 "a window attention layer, a large-chunk TTT layer, and a feed-forward layer" 세 층으로 구성됩니다.
토큰 단위가 아니라 큰 청크 단위로 fast weight를 갱신하는 것이 특징이며, 초록 기준 청크 크기는 2K에서 1M 토큰입니다.

```text
fast weight 네트워크 (Eq. 6):  f_W(x) = W₂[ SiLU(W₁x) ∘ (W₃x) ]
손실 (Eq. 7):                 ℒ = −f_W(k_i)ᵀ v_i
청크 갱신 (Eq. 4):             g = ∇_W Σ_{i=1}^{b} η_i ℒ(f_W(k_i), v_i)
가중치 갱신 (Eq. 8):           W ← L2-Normalize(W − g)
적용 (Eq. 2):                 o = f_W(q)
```

논문은 "We do not use explicit weight-decay term as in previous methods."라고 밝힙니다.
단순 누적은 "suffer from magnitude explosion or decayed memory"가 되므로 L2 정규화로 대체했다는 설명입니다.
기존 TTT 방법은 최신 GPU에서 "often below 5% peak FLOPS"에 머물렀고 LaCT는 A100에서 최대 70%까지 활용한다고 보고합니다.
비선형 상태의 크기는 모델 파라미터의 최대 40%까지 확장되며, 기존 방법은 0.1%에서 5% 수준이었다고 적혀 있습니다.

### Nested Learning과 Hope

[Nested Learning](https://arxiv.org/html/2512.24695v1){:target="_blank"}은 갱신 주파수가 서로 다른 MLP 블록을 연결한 Continuum Memory System(CMS)을 제안합니다.

```text
Eq. 70:  y_t = MLP^(f_k)( MLP^(f_{k−1})( ⋯ MLP^(f_1)(x_t) ) )
청크 크기:  C^(ℓ) := max_i C^(i) / f_ℓ
Eq. 71:  θ^(f_ℓ)_{i+1} = θ^(f_ℓ)_i − Σ_{t=i−C^(ℓ)}^{i} η_t^(ℓ) f(θ^(f_ℓ)_t; x_t)   (i ≡ 0 mod C^(ℓ) 일 때만)
```

ℓ번째 MLP는 `C^(ℓ)` 스텝마다 갱신됩니다.
고주파 블록은 빠른 적응을, 저주파 블록은 지속 기억을 맡습니다.
한 블록을 갱신할 때 망각이 일어나더라도 지식이 다른 구성요소에 남아 있다는 것이 논문의 논리이며, 형식적 보장은 제시되지 않았습니다.
[Google Research 블로그](https://research.google/blog/introducing-nested-learning-a-new-ml-paradigm-for-continual-learning/){:target="_blank"}는 Hope를 Titans 변형에 CMS 블록을 더한 개념 증명용 아키텍처로 소개하며, 결과 주장은 모두 자체 보고입니다.

### TTT-E2E

[TTT-E2E](https://arxiv.org/html/2512.23675v2){:target="_blank"}는 앞의 계열과 출발점이 다릅니다.
초록은 "We formulate long-context language modeling as a problem in continual learning rather than architecture design."이라고 밝힙니다.
모델은 "a standard Transformer with sliding-window attention"이며 새 레이어를 설계하지 않습니다.

```text
내부 루프 손실 (Eq. 1):  ℓ_t(W) = CE( f(x_{t−1}; W), x_t )
갱신 (Eq. 2):           W_t = W_{t−1} − η ∇ℓ_t(W_{t−1})
미니배치 (Eq. 5):        W_i = W_{i−1} − η (1/b) Σ_{t=(i−1)b+1}^{ib} ∇ℓ_t(W_{i−1})
외부 루프 (Eq. 6):       L(W_0; X) = (1/T) Σ_i Σ_t ℓ_t(W_{i−1})
```

내부 루프의 손실은 레이어별 재구성이 아니라 네트워크 끝의 next-token 예측입니다.
학습 손실이 테스트 시점 손실과 같기 때문에 E2E라는 이름이 붙었고, 초기값 `W_0`는 "gradients of gradients"로 메타러닝됩니다.
테스트 시점에 갱신되는 것은 MLP 층뿐이며, 임베딩과 정규화와 어텐션 층은 고정됩니다.
그중에서도 마지막 1/4 블록의 MLP만 갱신하고, 해당 블록에는 사전학습 지식을 보존하는 정적 MLP를 하나 더 둡니다.
128K 주요 결과의 설정은 슬라이딩 윈도우 k = 8K, TTT 미니배치 b = 1K입니다.

논문은 레이어별 key-value 재구성 손실을 쓰는 선행 방식을 TTT-KVB(key-value binding)라 부르고 MesaNet, Titans, Nested Learning을 그 변형으로 분류합니다.
760M 모델, 8K 컨텍스트의 절제 실험에서 DCLM loss는 SWA 2.827, TTT-KVB 2.818, TTT-E2E 2.805입니다.
결론부는 갱신되는 가중치와 윈도우의 역할을 "the weights updated at test time can be interpreted as long-term memory and the sliding window as short-term memory."로 정리합니다.

[VentureBeat 보도](https://venturebeat.com/infrastructure/new-test-time-training-method-lets-ai-keep-learning-without-exploding){:target="_blank"}에 따르면 실험 범위는 125M에서 3B 파라미터이고, 8K 컨텍스트로 사전학습한 뒤 128K로 파인튜닝했습니다.
같은 보도에서 저자 Yu Sun은 "You should think of the model as an RNN with a huge hidden state."라고 설명합니다.

### 두 겹의 루프

세 번째 계열의 공통 구조는 두 겹의 루프입니다.

- 내부 루프는 추론 중에 라벨 없이 자기지도 손실로 fast weight를 갱신합니다.
- 외부 루프는 학습 중에 내부 루프가 잘 작동하도록 slow weight(투영 행렬, 초기값 `W_0`, 학습률 생성기 등)를 역전파로 학습합니다.

외부 루프는 내부 루프를 통과하는 미분을 필요로 하므로 이계 미분이 등장합니다.
갱신 규칙을 망각 항 중심으로 나열하면 다음과 같습니다.

| 모델 | 갱신 규칙의 형태 | 망각과 유지의 메커니즘 |
|------|------------------|------------------------|
| Fast weights (Ba 외 2016) | 감쇠된 이전 상태에 은닉 벡터의 외적을 더함 | 고정 감쇠율 λ |
| Linear attention, FWP | value와 key의 외적을 가산 누적 | 없음, 용량 초과 시 간섭 |
| Delta rule (Schlag 외 2021) | 기존 인출값과의 차이만큼 보정 | 같은 key의 기존 값을 덮어씀 |
| TTT layer (Sun 외 2024) | 자기지도 손실에 대한 경사하강 | 명시적 항 없음, 학습 가능한 학습률 |
| Titans | 모멘텀이 붙은 경사하강과 감쇠 | 데이터 의존 weight decay와 모멘텀 |
| Miras | 손실과 retention 항의 합을 최소화 | 일반화된 retention 정규화 |
| LaCT | 갱신 후 L2 정규화 | 명시적 감쇠 없이 노름 고정 |
| CMS (Nested Learning) | 블록별로 정해진 주기마다 갱신 | 갱신 주파수가 다른 블록에 지식 분산 |
| Dynamic evaluation | 경사하강과 학습 완료 파라미터로의 회귀 | 학습된 파라미터 쪽으로 감쇠 |

## fast weights와 TTT의 관계

fast weights의 원 개념은 [Hinton과 Plaut(1987)](https://ni.cmu.edu/~plaut/papers/txt/HintonPlaut87CogSciConf.fastWeights.txt){:target="_blank"}에서 나옵니다.
각 연결이 두 개의 가중치를 가지며, 하나는 "A slowly changing, plastic weight which stores long-term knowledge"이고 다른 하나는 임시 지식을 담고 0으로 감쇠하는 빠른 가중치입니다.
실효 가중치는 두 가중치의 합입니다.
이 논문의 실험은 새 연상을 학습해 흐려진 기존 연상의 일부만 재학습해도 나머지 연상이 거의 같은 수준으로 회복되는 현상을 보였습니다.
즉 fast weight의 원래 용도 자체가 간섭을 일시적으로 상쇄하는 임시 기억입니다.

[Ba 외(2016)](https://ar5iv.labs.arxiv.org/html/1610.06258){:target="_blank"}는 이 개념을 어텐션과 연결합니다.

```text
Eq. 1:  A(t) = λ A(t−1) + η h(t) h(t)ᵀ
Eq. 3:  A(t) = η Σ_{τ=1}^{t} λ^{t−τ} h(τ) h(τ)ᵀ
Eq. 4:  A(t) h_s(t+1) = η Σ_{τ=1}^{t} λ^{t−τ} h(τ) [ h(τ)ᵀ h_s(t+1) ]
```

`λ`는 감쇠율이고 `η`는 fast learning rate이며, 주 실험 값은 η = 0.5, λ = 0.95입니다.
Eq. 4는 과거 은닉 벡터들을 현재 벡터와의 내적과 감쇠 계수로 가중한 형태이며, 논문은 이를 "fast weights act like a kind of attention to the recent past."라고 해석합니다.

[Schlag 외(2021)](https://ar5iv.labs.arxiv.org/html/2102.11174){:target="_blank"}는 Fast Weight Programmer를 "A traditional slow net with slow weights continually changes or reprograms the fast weights of a fast net."으로 정의합니다.
이 논문은 softmax를 제거한 self-attention이 외적 기반 가산 갱신으로 쓰인다는 점을 보이고 "the core of linear Transformer variants are outer product-based Fast Weight Programmers."라고 결론짓습니다.
다만 1991년 연구가 곧 선형 트랜스포머라는 식의 우선권 서술은 주로 Schmidhuber 측의 입장이며, 합의된 역사로 단정하기는 어렵습니다.

TTT 논문들은 스스로를 이 계보 안에 놓습니다.

- Sun 외(2024)는 fast weights를 "The general idea of fast weights is to update the parameters of a "fast" model on only the most relevant data."로 요약하고 TTT를 그 특수한 경우로 서술합니다.
- 같은 논문은 차이점으로 "Compared to fast weights, TTT embraces the idea of formulating an explicit learning problem"을 듭니다.
- TTT-E2E는 "the inner-loop weights W can be viewed as 'fast' and the outer-loop weights θ as 'slow'."라고 적습니다.
- LaCT는 fast weight를 추론 중 고정되는 slow weight와 대비되는 "rapidly adaptable weights"로 설명합니다.
- Titans는 선형 층을 key-value 연상 기억으로 보는 관점이 fast weight programs로 거슬러 올라간다고 적습니다.

정리하면 fast weight는 빠르게 변하는 가중치에 임시 기억을 저장한다는 상위 개념입니다.
TTT는 그 갱신 규칙을 자기지도 손실에 대한 gradient step으로 구체화하고 외부 루프로 메타러닝한 것입니다.
즉 두 개념은 병렬이라기보다 포함 관계에 가깝습니다.

## continual learning과의 비교

[continual learning 서베이](https://ar5iv.labs.arxiv.org/html/2302.00487){:target="_blank"}는 전형적 목표를 "learn a sequence of contents one by one and behave as if they were observed simultaneously."로 정의합니다.
핵심 긴장은 "the trade-off between learning plasticity and memory stability."입니다.
Google Research 블로그는 continual learning을 "the ability for a model to actively acquire new knowledge and skills over time without forgetting old ones"로 정의합니다.

논문들의 서술을 바탕으로 두 개념을 비교하면 다음과 같습니다.

| 축 | 전통적 continual learning | TTT (시퀀스 모델링용) |
|----|---------------------------|------------------------|
| 갱신 대상 | 모델 파라미터 전체 또는 태스크별 일부 | 레이어 내부 fast weight, slow weight는 추론 중 고정 |
| 갱신 시점 | 배포 후 태스크나 분포가 바뀔 때의 학습 단계 | 추론의 forward pass 안에서 토큰 또는 청크마다 |
| 학습 신호 | 주로 라벨이 있는 새 태스크 데이터 | 자기지도 손실(재구성 또는 next-token) |
| 데이터 단위 | 시간에 따라 변하는 분포 | 하나의 시퀀스가 하나의 데이터셋 |
| 지속성 | 갱신이 영구적으로 누적 | 컨텍스트가 사라지면 유지되지 않는다는 서술이 있음 |
| 망각의 위상 | 풀어야 할 핵심 문제 | 설계 요소(감쇠, 게이트) 또는 용인되는 대가 |

논문마다 continual learning과의 거리를 다르게 서술합니다.

- Sun 외(2020)는 자신들의 설정을 continual learning과 구분하며 "We are not concerned about forgetting the past test samples since they have already been evaluated on."이라고 적습니다.
- TTT-E2E는 continual learning의 목표를 "to enable AI systems to keep changing with the world"로 정리하고, 기존 분야가 "focused on learning from a distribution that gradually changes over time"이라고 서술합니다.
- TTT-E2E는 같은 상위 목표를 공유하되 개인화와 학습 및 테스트 경계의 부재라는 두 측면에 집중한다고 밝히며, 장기 컨텍스트 모델링 자체를 continual learning 문제로 정식화합니다.
- Nested Learning은 "Test Time Training/Memorization are Instances of In-Context Learning"이라 분류하고, 이때 얻은 지식이 "does not persist once the current context is removed."라고 적습니다.
- Nested Learning은 continual learning 맥락에서 이를 test-time learning이라 부르는 것이 "can be misleading"이라고도 씁니다.
- Titans와 Akyürek 외 논문 본문에는 continual learning과의 관계 서술이 없습니다.

TTT-E2E는 기존 TTT에 대해 "the model is reset after each prediction because the test instances are independent."라고 적고, 시퀀스용 TTT는 시퀀스 내부에서 갱신을 이어간다고 설명합니다.
다만 TTT-E2E가 서로 다른 테스트 시퀀스 사이에 가중치를 `W_0`로 리셋하는지에 대한 명시적 문장은 조사 범위에서 확인되지 않았습니다.
메타러닝 구조상 시퀀스마다 `W_0`에서 시작하는 것으로 읽히지만 이는 추정입니다.

이 지점에서 의견이 갈립니다.
엄밀한 쪽은 fast weights가 시퀀스마다 초기화되므로 그 자체로는 continual learning 문제의 해법이 아니라고 봅니다.
Ian Barber의 글은 "the fast weights reset for every new sequence. So, no."라고 결론짓습니다.
비공식 Titans 구현에서도 메모리 모델의 기본 가중치가 영구적으로 수정되지 않고 각 데이터 예제의 시작점이 동일하다는 점이 구현자의 답변으로 확인됩니다([titans-pytorch 이슈 #6](https://github.com/lucidrains/titans-pytorch/issues/6){:target="_blank"}).
가중치가 추론 중에 바뀐다는 사실과 그 변화가 세션을 넘어 지속된다는 것은 별개의 문제라는 점이 쟁점입니다([관련 토론](https://news.ycombinator.com/item?id=46184405){:target="_blank"}).

반대로 느슨한 의미에서는 이 계열을 continual learning으로 가는 경로로 부르는 시각이 널리 보입니다.
E2E-TTT를 최근 컨텍스트를 추가 학습 데이터처럼 취급해 일부 가중치를 갱신하는 방식으로 요약하고 유연한 접근으로 평가하는 의견이 있습니다([관련 토론](https://news.ycombinator.com/item?id=48282135){:target="_blank"}).
정의부터 다르다는 지적도 있습니다.
단순한 암기와 고정 크기 메모리 채우기는 continual learning이 가리키는 바가 아니며, 지속적 일반화와 기억의 병합, catastrophic forgetting의 부재, graceful degradation까지 포함해야 한다는 의견입니다([관련 토론](https://news.ycombinator.com/item?id=49791267){:target="_blank"}).

문헌 서술을 종합해 해석하면, TTT는 언제 학습하는가(추론 시점)를 규정하고 continual learning은 무엇을 달성하는가(망각 없이 계속 습득)를 규정합니다.
이 구분은 특정 논문의 문장이 아니라 여러 소스를 종합한 해석입니다.

## 어텐션과의 관계

TTT를 어텐션 모듈을 바꾸는 방식으로 설명하는 것은 세 번째 계열의 초기 연구에는 잘 맞습니다.
TTT-E2E 논문은 선행 계열을 "TTT(-KVB) layers are drop-in replacements for self-attention layers"라고 부릅니다.
Sun 외(2024)는 두 정리로 어텐션과의 관계를 제시합니다.

- Theorem 1은 내부 모델이 선형 `f(x) = Wx`이고 갱신이 배치 경사하강이며 η = 1/2, `W_0` = 0일 때, TTT layer가 linear attention과 같은 출력 열을 낸다는 내용입니다.
- Theorem 2는 학습기를 Nadaraya-Watson 추정기로 두면 self-attention과 같은 출력을 낸다는 내용입니다.

조건을 정확히 읽을 필요가 있습니다.
배치 경사하강은 모든 gradient를 초기값 `W_0`에서 계산한다는 뜻이며, 직전 가중치에서 계산하는 온라인 경사하강으로 바꾸면 linear attention과 같지 않습니다.
softmax self-attention과의 동등성은 gradient 학습이 아니라 비모수 학습기로 해석했을 때 성립합니다.
즉 어텐션이 TTT의 특수한 경우라는 서술에는 학습기의 선택에 따라서라는 단서가 붙습니다.

같은 논문의 Table 1(125M 모델, perplexity)은 이 일반화가 성능 차이로 이어지는 과정을 보여줍니다.

| 구성 | Perplexity | 차이 |
|------|------------|------|
| Linear attention | 15.91 | 기준 |
| Linear attn. improved | 15.23 | −0.68 |
| TTT equivalence | 15.23 | 0 |
| learnable W_0 추가 | 15.27 | +0.04 |
| f 안에 LN과 residual 추가 | 14.05 | −1.22 |
| mini-batch TTT 추가 | 12.35 | −1.70 |
| learnable η 추가 | 11.99 | −0.36 |
| Mamba backbone 적용 | 11.09 | −0.90 |

같은 방향의 독립적 근거도 있습니다.
[von Oswald 외(2022)](https://ar5iv.labs.arxiv.org/html/2212.07677){:target="_blank"}는 1-head linear self-attention 층의 가중치를 특정하게 구성하면 제곱오차 손실에 대한 경사하강 1 스텝과 일치한다는 점을 보였고, 그 결과가 "restricted to small Transformers and simple regression problems."라고 한계를 적었습니다.
[Clark 외(2022)](https://ar5iv.labs.arxiv.org/html/2212.02475){:target="_blank"}는 fast weight의 gradient 갱신 합이 "can be interpreted as linear attention (i.e., without a softmax)."라고 서술합니다.

설계 패턴은 세 가지로 나뉩니다.

| 패턴 | 대표 | 어텐션과의 관계 |
|------|------|-----------------|
| TTT layer가 self-attention을 대체 | Sun 외 2024 | 어텐션 자리에 TTT layer를 넣음 |
| 윈도우 어텐션과 신경 메모리 병치 | Titans, LaCT, Hope | 어텐션은 가까운 범위, 메모리는 먼 범위 |
| 아키텍처는 그대로 두고 일부 MLP를 갱신 | TTT-E2E | 어텐션 층은 고정, 마지막 1/4 블록의 MLP만 갱신 |

따라서 "어텐션 모듈의 변경"이라는 설명은 부분적으로만 맞습니다.
TTT layer는 어텐션을 대체하고, Titans와 LaCT는 어텐션 옆에 메모리 층을 둡니다.
반면 TTT-E2E는 어텐션 층을 고정하고 MLP 층만 갱신하며, 원래의 TTT(2020)와 Akyürek 계열은 어텐션과 무관합니다.
어텐션 자리에 테스트 시점에 학습되는 가중치를 넣거나 붙이는 계열이 있고, 최신 연구는 어텐션을 건드리지 않는 방향으로도 나아갔다고 풀어 쓰는 편이 정확합니다.

## 손실 압축 관점과 망각

### 압축이라는 서술

압축이라는 표현은 논문들이 직접 사용합니다.

- Sun 외(2024)는 "Unlike self-attention, RNN layers have to compress context into a hidden state of fixed size."라고 적고 갱신 규칙을 "compression heuristic"이라 부릅니다.
- TTT-E2E는 "The key mechanism is compression."이라고 적고, full attention을 "designed for nearly lossless recall"로 대비합니다.
- Titans는 "a very long context cannot be properly compressed in a small vector-valued or matrix-valued states"라고 적습니다.
- Miras는 "these models compress the context into a fixed size memory"라고 적습니다.
- [Google Research 블로그](https://research.google/blog/titans-miras-helping-ai-have-long-term-memory/){:target="_blank"}는 선형 RNN 계열에 대해 "this fixed-size compression cannot adequately capture the rich information in very long sequences."라고 적습니다.

사람의 기억에 빗댄 설명도 저자들이 직접 사용합니다.
[NVIDIA 기술 블로그](https://developer.nvidia.com/blog/reimagining-llm-memory-using-context-as-training-data-unlocks-models-that-learn-at-test-time){:target="_blank"}는 사람이 "despite their imperfect recall of the exact details" 경험으로 나아진다고 서술합니다.
같은 글은 TTT를 "like updating the human brain"에, RAG를 "writing things down and looking things up in a notepad or calendar"에 비유합니다.

다만 조사된 범위의 논문에서 "lossy compression"이라는 정확한 표현이나 엔트로피, rate-distortion 같은 정보이론적 정식화는 확인되지 않았습니다.
논문의 표현은 compression과 nearly lossless recall의 대비, 그리고 아래의 용량 상한 명제입니다.
"lossy"라는 단어를 직접 쓰는 것은 논문이 아닌 에세이 쪽입니다.
[a16z 에세이](https://a16z.com/why-we-need-continual-learning/){:target="_blank"}는 "The lossy compression is the learning."이라고 적어 손실 압축을 일반화의 원천으로 봅니다.

### 용량 상한

고정 크기 가중치가 담을 수 있는 연상의 수에는 상한이 제시되어 있습니다.

- Ba 외(2016)는 N²개 가중치의 자기연상 기억이 실수 벡터 N개를 넘겨 저장할 수 없다고 적습니다.
- Schlag 외(2021)는 "With keys embedded in a d_dot space, there cannot be more than d_dot orthogonal vectors."라고 적고, 초과 시 인출 오류가 생기는 영역을 overcapacity regime이라 부릅니다.
- 같은 논문의 합성 실험에서 key 차원이 64인 linear attention은 연상 약 60개부터 오류가 누적됩니다.
- [ATLAS](https://ar5iv.labs.arxiv.org/html/2505.23735){:target="_blank"}의 Proposition 1은 ℓ2 attentional bias와 경사하강으로 학습되는 행렬 메모리가 선형 독립인 key를 가진 쌍을 최대 𝒪(d_k)개 저장한다고 서술합니다.
- ATLAS의 Theorem 1은 깊은 MLP 메모리의 상한이 깊이에 따라 커지지만 "remains subquadratic in key and value dimensions."라고 서술합니다.

LaCT는 상태 크기가 작을수록 긴 시퀀스에서 성능 격차가 벌어진다고 보고합니다.

### 세부 회상 실험 결과

가장 직접적인 정량 근거는 TTT-E2E의 Table 2(3B 모델, RULER S-NIAH, 정확도)입니다.
아래는 그중 S-NIAH-1(pass-key retrieval)의 값입니다.

| 방법 | 8K | 16K | 32K | 64K | 128K |
|------|-----|-----|-----|-----|------|
| Full attention | 1.00 | 1.00 | 1.00 | 1.00 | 0.99 |
| SWA | 1.00 | 0.50 | 0.26 | 0.13 | 0.07 |
| Hybrid SWA and full | 1.00 | 0.93 | 0.88 | 0.69 | 0.21 |
| Mamba 2 | 0.99 | 0.49 | 0.26 | 0.13 | 0.07 |
| Gated DeltaNet | 1.00 | 0.50 | 0.26 | 0.13 | 0.07 |
| TTT-KVB | 0.98 | 0.43 | 0.22 | 0.10 | 0.01 |
| TTT-E2E | 1.00 | 0.46 | 0.24 | 0.13 | 0.06 |

저자들은 "Transformer with full attention dramatically outperforms the other methods, including ours, especially in long context."라고 적습니다.
원인으로는 압축이 "leaves out seemingly irrelevant details, such as the target string."이라는 점을 듭니다.
윈도우(8K) 안에서는 모든 방법이 거의 만점이고, 윈도우를 벗어나면 TTT-E2E는 SWA와 사실상 같은 수준으로 떨어집니다.
언어모델링 loss는 full attention처럼 컨텍스트 길이에 따라 개선된다고 보고되지만 그 수치는 그림에만 있으며, 특정 문자열의 정확한 회상은 가중치에 남지 않는다는 것이 이 표의 의미입니다.

다른 모델에서도 같은 경향이 보고됩니다.
Nested Learning의 S-NIAH-3(4K, 8K, 16K)에서 Hope는 73.2, 46.2, 24.8이고 Transformer는 78.0, 69.2, 40.8입니다.
ATLAS 저자들도 "While Transformers still achieve the best results in in-context recall tasks..."라고 인정합니다.

실행 경험에 근거한 보고도 하나 있습니다.
TTT-E2E 공식 저장소에는 needle이 sliding-window attention 창 안에 없을 때 정확 회상이 성공한 사례를 관찰하지 못했다는 실험 보고가 올라와 있습니다([e2e 이슈 #8](https://github.com/test-time-training/e2e/issues/8){:target="_blank"}).
이는 한 사용자의 보고이며 조회 시점 기준으로 저자의 답변은 없습니다.

needle 과제 자체의 한계에 대한 지적도 있습니다.
고립된 정보를 찾는 일은 전체 컨텍스트를 종합해 과제를 푸는 일보다 훨씬 덜 까다로우므로, needle 과제가 장기 컨텍스트 성능의 좋은 척도가 아니라는 의견입니다([관련 토론](https://news.ycombinator.com/item?id=42730627){:target="_blank"}).

### 설계 요소로서의 망각

논문들은 망각을 해결 불가능한 문제가 아니라 관리 대상으로 다룹니다.

- Titans는 유한한 메모리를 다루기 위해 감쇠 메커니즘을 두며, 가산적 갱신만으로는 "results in memory overflow"가 된다고 적습니다.
- Miras는 forget gate 대신 Retention Gate라는 용어를 쓰며 "there is no memory erasing or forgetting process."라고 적습니다.
- LaCT는 명시적 감쇠 없이 가중치 노름을 고정하고, 상태 크기를 파라미터의 최대 40%까지 키웁니다.
- Nested Learning은 갱신 주파수가 다른 블록에 지식을 분산합니다.
- TTT-E2E는 정밀한 단기 기억을 슬라이딩 윈도우 어텐션에 맡기고, 사전학습 지식 보존용 정적 MLP를 병치합니다.
- Sun 외(2020)는 온라인형에서 "forgetting is not harmful and perhaps should even be taken advantage of."라고 적습니다.

망각이 원리적으로 사라지지 않는다는 서술도 있습니다.
Nested Learning의 결론부는 catastrophic forgetting이 "not “solved” in general"이며 "a natural consequence of compression"이라고 적습니다.
같은 대목은 용량이 한정되어 있어 "forces the model to forget so that it retains capacity for new information."이라고 설명합니다.

여기서 두 종류의 망각을 구분해 볼 수 있습니다.
하나는 압축에 따른 세부 누락으로, TTT-E2E와 Titans가 설계상 받아들이는 망각입니다.
다른 하나는 새 학습이 기존 표현을 파괴하는 catastrophic forgetting으로, a16z 에세이는 이를 "Catastrophic forgetting means models sensitive enough to learn from new data destroy existing representations."라고 설명합니다.
후자는 완화 대상인 미해결 과제로 다뤄지며, [MIT의 SEAL 연구](https://news.mit.edu/2025/teaching-large-language-models-to-absorb-new-knowledge-1112){:target="_blank"}도 반복 적응 시 이전 태스크 성능이 점차 저하되는 점을 한계로 명시합니다.
이 두 종류의 구분은 여러 소스를 종합한 해석이며 특정 논문이 이 형태로 정리한 것은 아닙니다.

실행 경험이 아닌 의견 수준에서는 두 방향의 반응이 있습니다.
망각이 필요하다는 쪽은 기억 계층에 빠져 있는 것이 통합과 우선순위화와 망각이라고 주장합니다([관련 토론](https://news.ycombinator.com/item?id=48283533){:target="_blank"}).
회의적인 쪽은 모든 학습이 가중치 갱신으로 일어난다면 "its just rearranging where the forgetting happens"라고 지적하며, 이 질문에는 답글이 달리지 않았습니다([관련 토론](https://news.ycombinator.com/item?id=46191332){:target="_blank"}).

## 가중치 안에 암묵적으로 저장된다는 비판

가중치 기반 메모리에는 새로 들어온 정보가 가중치 안에 지나치게 암묵적으로 내재된다는 비판이 있습니다.
비판은 네 갈래로 나뉩니다.

첫째는 불투명성과 검사 불가입니다.
[Letta의 글](https://www.letta.com/blog/continual-learning){:target="_blank"}은 "Weight updates, by contrast, are opaque and require systematic evals to detect problems."라고 적고, 대비되는 장점으로 "Token-space memories are human-readable."을 듭니다.
이 소스는 논문이 아니라 에이전트 메모리 회사의 블로그 글이며, TTT나 Titans를 이름으로 언급하지 않고 LoRA와 RL 기반 가중치 갱신을 대상으로 합니다.
[Fast-weight Product Key Memory](https://arxiv.org/html/2601.00671){:target="_blank"}는 슬롯 방식의 장점으로 "we can trace retrieved information back to specific input tokens."를 듭니다.

둘째는 얽힘과 선택적 삭제 불가입니다.
[Modular Memory 포지션 논문](https://arxiv.org/html/2603.01761){:target="_blank"}은 분산 파라메트릭 메모리에 대해 "entangled shared parameters make removing one item risk harming others."라고 적습니다.
이 논문은 "In neural memories, retrieval is implicit and learned as part of the forward pass"라고도 적는데, 여기서 implicit의 대상은 저장이 아니라 인출입니다.
[Panini](https://arxiv.org/html/2602.15156){:target="_blank"}는 "parametric storage provides no native mechanism for source attribution or selective retraction"이라고 적습니다.
Letta의 글은 반대편의 장점을 "Forgetting is trivial: just delete the tokens."로 요약합니다.

셋째는 손실은 줄어도 꺼내 쓰지 못한다는 점이며, 가장 정량적인 근거입니다.
[Beyond Perplexity](https://arxiv.org/html/2607.00368){:target="_blank"}는 Qwen3 세 모델에 LoRA 1-step 온라인 갱신으로 임의의 접근 코드 사실을 주입한 뒤, 컨텍스트를 제거하고 회상을 시험했습니다(모델당 48개 문항).

| 모델 | Support ΔNLL | 직접 질문 정답 ΔNLL | 바꿔 말한 질문 정답 ΔNLL | 생성 회상률 (직접, 바꿔 말함, 지연) |
|------|--------------|---------------------|--------------------------|--------------------------------------|
| Qwen3-1.7B | −1.151 | −0.674 | −0.714 | 0.0%, 0.0%, 0.0% |
| Qwen3-4B | −1.218 | −0.529 | −0.531 | 0.0%, 0.0%, 0.0% |
| Qwen3-8B | −0.966 | −0.504 | −0.463 | 0.0%, 0.0%, 0.0% |

NLL은 내려가지만 실제 생성으로는 한 번도 회상하지 못했습니다.
논문은 "A system can improve language-model loss while still failing to retrieve the new knowledge behaviorally."라고 적습니다.
갱신을 강하게 하면(1.7B stress update) 직접 및 지연 회상이 72.9%로 오르지만 locality가 97.9%에서 9.7%로 떨어져, 관련 없는 지식이 훼손됩니다.
저자들은 perplexity 근거가 배포 후 기억 주장으로 옮겨 쓰이는 현상을 "evidence migration"이라 부릅니다.

이 결과를 읽을 때의 한계는 분명합니다.
실험의 갱신 방식은 메타러닝된 TTT layer가 아니라 일반 LoRA 미세조정입니다.
저자들도 Qwen3와 LoRA 한 가지 갱신 방식만 실험했고 TTT-E2E 같은 대표적 TTT 시스템은 평가하지 않았다고 명시합니다.
따라서 이 결과를 TTT-E2E나 Titans에 그대로 적용하면 과잉 일반화입니다.

넷째는 정보 병목과 덮어쓰기입니다.
[SR-TTT](https://arxiv.org/html/2603.06642v1){:target="_blank"}는 "the fast weights aggressively compress the context into an information bottleneck"이라 적고, 드문 needle이 "overwritten by the continuous stream of updates from background tokens."라고 설명합니다.
이 논문은 surprisal이 높은 토큰만 정확 어텐션 캐시로 보내는 구조를 제안했고, 15.8M 파라미터 모델의 2048 토큰 NIAH에서 depth 0.50의 exact match가 10%에서 33%로 올랐다고 보고합니다.
4096 토큰에서는 두 구조 모두 0%였으며, 저자가 명시하듯 소규모 단일 저자 실험입니다.

반대 방향의 서술도 있습니다.
[LLM 메모리 서베이](https://arxiv.org/html/2607.25380v1){:target="_blank"}는 TTT 계열을 implicit이 아니라 explicit memory로 분류하며 "Titans exemplifies explicit memory because the stored information resides in persistent parameters."라고 적습니다.
전용 저장 영역과 쓰기 규칙이 있으면 기능적으로 명시적이라는 기준입니다.
즉 implicit과 explicit의 구분은 사람이 읽을 수 있는가를 기준으로 삼는지, 전용 저장소와 쓰기 규칙이 있는가를 기준으로 삼는지에 따라 논문마다 달라집니다.
Letta의 글도 가중치를 배제하지 않고 "We envision that future agents use both representations"라고 적습니다.
TTT-E2E의 Table 2는 윈도우 밖 세부 회상의 붕괴를 TTT 진영 내부에서 인정한 근거이기도 합니다.

## 비용과 재현성 그리고 서빙과 안전

### 비용

논문들이 직접 보고한 비용 수치는 다음과 같습니다.

| 항목 | 수치 | 출처 |
|------|------|------|
| 추론 지연 | 컨텍스트 길이와 무관하게 일정, 128K에서 full attention 대비 2.7배 빠름(H100) | TTT-E2E |
| 학습 지연 | 8K에서 full attention 대비 3.4배 느림, 128K에서는 1.2배 빠름 | TTT-E2E |
| 커널 제약 | 이계 미분 미지원으로 학습 시 cuDNN FlashAttention 사용 불가 | TTT-E2E |
| 하드웨어 활용률 | 기존 TTT 방법은 최신 GPU에서 peak FLOPS의 5% 미만인 경우가 많고, LaCT는 A100에서 최대 70% | LaCT |
| 학습 처리량 (3B, 32K) | Transformer 4.1K TPS, LaCT GD 5.0K, LaCT Momentum 4.9K, LaCT Muon 4.3K | LaCT |
| wall-clock | TTT-Linear 2k 컨텍스트 iteration 0.27s, Transformer 0.30s | Sun 외 2024 |
| dynamic evaluation | 표준 추론 대비 3배 이상 연산, Transformer-XL 1575 tok/s에서 510 tok/s로 감소 | Clark 외 2022 |

NVIDIA 기술 블로그는 2M 컨텍스트에서 35배 빠르다고 기재합니다.
다만 VentureBeat는 백만 토큰 규모의 이득을 실측이 아닌 추정(projection)이라고 적었으므로, 이 수치는 실측 여부가 확인되지 않은 값으로 보아야 합니다.
Sun 외(2024)는 TTT-MLP가 "still faces challenges in memory I/O"이며 시스템 최적화가 "preliminary at best"라고 인정합니다.
TTT-E2E 저자들은 일부 결과가 "merely artifacts of our experimental setup"일 가능성도 인정합니다.

태스크별 미세조정 계열의 비용도 보고되어 있습니다.
[MIT News](https://news.mit.edu/2025/study-could-lead-llms-better-complex-reasoning-0708){:target="_blank"}에 따르면 통상 1분 미만인 응답이 5분에서 10분으로 늘 수 있습니다.

### 재현성과 공개 구현

공개 구현의 상태는 계열마다 다릅니다.

| 저장소 | 내용 | 비고 |
|--------|------|------|
| test-time-training/ttt-lm-pytorch | TTT-Linear, TTT-MLP PyTorch 구현 | 튜토리얼용이며 학습 용도는 권장하지 않는다고 명시 |
| test-time-training/e2e | TTT-E2E 공식 JAX 구현 | 125M, 1B, 3B 체크포인트 공개 |
| a1600012888/LaCT | LaCT 코드 | MIT 라이선스 |
| Titans, Hope 공식 코드 | 확인되지 않음 | Titans 논문은 available soon으로만 기재 |

Titans의 공식 코드가 없다는 점은 재현 연구에서도 지적됩니다.
[Titans Revisited](https://arxiv.org/abs/2510.09551){:target="_blank"}는 공개 코드 부재로 자체 재구현을 했고, Titans가 청킹 때문에 기존 기준선을 항상 이기지는 못하지만 Neural Memory 구성요소는 어텐션 전용 모델 대비 일관되게 개선을 준다고 보고합니다.
이 논문은 초록 수준에서만 확인되었습니다.

비공식 구현을 실제로 돌려 본 사람들의 기록은 학습 불안정을 반복해서 보고합니다.

- 비공식 Titans 구현의 초기 실험에서 구현자는 norm과 adaptive step의 clamping을 추가하지 않으면 NaN이 발생했다고 적었습니다([titans-pytorch 이슈 #2](https://github.com/lucidrains/titans-pytorch/issues/2){:target="_blank"}).
- 같은 이슈에서 2층 MLP 메모리는 수렴했지만 gated delta net을 이기지 못했고 3층은 더 나빴다는 결과가 보고되었습니다.
- 시퀀스 길이 4096에서 neural memory 손실이 발산한다는 보고에 대해 구현자는 내부 네트워크와 외부 네트워크를 함께 최적화하므로 예상되는 일이라고 답했습니다([titans-pytorch 이슈 #29](https://github.com/lucidrains/titans-pytorch/issues/29){:target="_blank"}).
- 같은 이슈의 보고자는 학습률을 낮춰 NaN은 피했으나 "it's generally better to disable memory rather than use it."이라고 적었습니다.
- 모델 차원, 층 수, 메모리 갱신과 조회의 순서 등 구현 세부가 논문에 없다는 지적이 있습니다([titans-pytorch 이슈 #4](https://github.com/lucidrains/titans-pytorch/issues/4){:target="_blank"}).
- 구현자는 2025년 12월에 구현이 논문과 정확히 일치하지 않았음을 인정하면서 "think this line of research is overhyped"라고 적고 다른 주제로 옮겼습니다([titans-pytorch 이슈 #32](https://github.com/lucidrains/titans-pytorch/issues/32){:target="_blank"}).
- 비공식 Nested Learning 재현의 저자는 "I do not see a clean, consistent 'wins everywhere' story yet at current scale."이라고 적었고, 논문 최대 규모로는 연산 제약 때문에 실행하지 못했다고 밝혔습니다([nested_learning 이슈 #12](https://github.com/kmccleary3301/nested_learning/issues/12){:target="_blank"}).
- A100 8장에서 HOPE-mid 한 스텝에 약 5분이 걸린다는 보고가 있으며 조회 시점 기준으로 답변은 없습니다([nested_learning 이슈 #13](https://github.com/kmccleary3301/nested_learning/issues/13){:target="_blank"}).

모든 NaN 보고가 방법론의 결함은 아닙니다.
titans-pytorch의 또 다른 NaN 보고는 환경 문제로 결론이 났습니다.

TTT layer 공식 저장소에서는 저자가 안정화 방법을 설명합니다.
K와 V 투영이 외부 루프 손실로 학습되고, 안정성을 위해 `f(x) = x + LN(f_res(x))` 형태의 잔차 연결을 쓴다는 답변입니다([ttt-lm-pytorch 이슈 #21](https://github.com/test-time-training/ttt-lm-pytorch/issues/21){:target="_blank"}).
ARC용 TTT 공식 코드에서는 기본 Llama-3-8B-Instruct에 TTT를 적용해 400개 중 19개를 맞혔다는 보고에 대해, 저자가 논문의 사전 파인튜닝된 체크포인트를 쓰지 않은 것이 원인이라고 답했습니다([marc 이슈 #3](https://github.com/ekinakyurek/marc/issues/3){:target="_blank"}).
이 사례는 태스크별 TTT의 효과가 사전 파인튜닝에 크게 의존함을 보여줍니다.

논문 서술 자체에 대한 비판도 있습니다.
Titans 블로그 공개 시점에 논문 공개 후 11개월이 지나도록 코드와 가중치를 내려받을 수 없다는 지적이 있었습니다([관련 토론](https://news.ycombinator.com/item?id=46182168){:target="_blank"}).
Nested Learning 논문이 부록을 여러 번 참조하는데 PDF에 부록이 없어 재현이 어렵다는 지적도 있었습니다([관련 토론](https://news.ycombinator.com/item?id=45852806){:target="_blank"}).

### 서빙과 사업 모델

사용자별로 가중치가 달라지면 서빙 구조와 충돌한다는 의견이 있습니다.

- 가중치 수준의 continual learning은 공유 가중치를 포기해야 하므로 I/O와 저장 비용 때문에 실현이 어렵다는 의견이 있습니다([관련 토론](https://news.ycombinator.com/item?id=49709667){:target="_blank"}).
- 같은 토론의 답글은 역전파를 함께 수행하면 추론에 훨씬 많은 메모리가 필요해진다고 지적합니다.
- 클라우드 서빙은 모든 사용자가 같은 모델을 쓴다는 전제 위에서 요청을 배치 처리하므로, 사용자별 모델은 이 전제와 맞지 않는다는 의견이 있습니다([관련 토론](https://news.ycombinator.com/item?id=49052101){:target="_blank"}).
- 반론으로는 갱신되는 가중치가 전체의 작은 부분집합이므로, 대화마다 KV 캐시를 따로 저장하듯 fast weights를 따로 저장하면 된다는 의견이 있습니다([관련 토론](https://news.ycombinator.com/item?id=48287728){:target="_blank"}).

정밀한 회상이 필요한 경우의 보완재로는 RAG가 계속 언급됩니다.
VentureBeat 보도에서 연구진은 "We believe that these two classes of memory will continue to complement each other."라고 밝힙니다.

### 안전과 오염

추론 중에 가중치가 바뀌면 새로운 공격 표면이 생긴다는 우려가 있습니다.

- 무작위의 확률 낮은 입력을 계속 넣으면 모두 surprise로 기록되어 메모리가 오염되지 않느냐는 질문이 있습니다([관련 토론](https://news.ycombinator.com/item?id=46182122){:target="_blank"}).
- 이에 대한 반론은 외부 루프 학습에서 무의미한 토큰에 낮은 surprise를 주도록 학습되었을 것이라는 기대에 근거하며 실증은 아닙니다.
- 실시간 가중치 갱신이 모델의 영구적 탈옥을 허용하고, 학습한 내용을 다음 모델 세대로 효율적으로 옮길 방법이 없다는 지적이 있습니다([관련 토론](https://news.ycombinator.com/item?id=48282915){:target="_blank"}).
- Panini는 TTT 계열에 대해 "these methods may reinforce incorrect predictions, leading to error accumulation or catastrophic forgetting over time"이라고 적습니다.

이 절과 앞 절에서 다룬 의견은 Hacker News와 GitHub Issues에서 수집된 것입니다.
Reddit과 X의 반응은 수집되지 않았으므로 전체 개발자 커뮤니티의 여론으로 일반화할 수 없습니다.
또한 실행 경험에 근거한 발언은 GitHub 이슈의 보고들로 한정되며, 나머지 다수는 논문과 블로그를 읽고 쓴 의견입니다.

## 결론

LLM 메모리 맥락의 TTT는 추론 중에 fast weight를 갱신해 컨텍스트를 가중치에 압축하는 시퀀스 모델링 기법입니다.
이 계열은 fast weights라는 오래된 개념 위에서, 갱신 규칙을 자기지도 손실에 대한 경사하강으로 구체화하고 그 초기값과 규칙을 외부 루프로 메타러닝한 것입니다.

continual learning과는 상위 목표를 공유하지만, 현재 확인되는 실험은 단일 시퀀스 내부의 압축에 머물러 있습니다.
세션을 넘어 가중치가 지속되는 형태의 실증은 조사 범위에서 확인되지 않았습니다.
어텐션 모듈을 바꾸는 방식이라는 설명은 TTT layer와 Titans 계열에는 맞지만 어텐션을 고정하는 TTT-E2E에는 맞지 않습니다.

압축이 세부를 누락한다는 점은 저자들이 직접 인정하고 needle 과제의 수치로 드러납니다.
다만 논문들은 망각을 감쇠, retention, 정규화, 다중 주파수 메모리, 윈도우 어텐션과의 혼합으로 관리하는 설계 요소로 다룹니다.
가중치에 저장된 정보가 불투명하고 얽혀 있으며 선택적으로 지우기 어렵다는 비판은 여러 소스에서 확인되지만, 그중 가장 정량적인 실험은 일반 LoRA 갱신을 대상으로 한 것이어서 TTT 계열에 그대로 옮길 수 없습니다.

남은 쟁점은 컨텍스트 없이도 파라미터가 과거 입력을 보존하는가입니다.
컨텍스트가 제거된 뒤에도 가중치가 과거 입력을 얼마나 보존하고 꺼내 쓸 수 있는지, 그리고 그것을 perplexity가 아닌 행동 수준의 회상으로 어떻게 검증할지가 아직 열려 있습니다.

## Reference

- [Test-Time Training with Self-Supervision for Generalization under Distribution Shifts](https://ar5iv.labs.arxiv.org/html/1909.13231)
- [The Surprising Effectiveness of Test-Time Training for Few-Shot Learning](https://ar5iv.labs.arxiv.org/html/2411.07279)
- [Dynamic Evaluation of Neural Sequence Models](https://ar5iv.labs.arxiv.org/html/1709.07432)
- [Learning to (Learn at Test Time): RNNs with Expressive Hidden States](https://arxiv.org/html/2407.04620v4)
- [Titans: Learning to Memorize at Test Time](https://ar5iv.labs.arxiv.org/html/2501.00663)
- [It's All Connected: A Journey Through Test-Time Memorization, Attentional Bias, Retention, and Online Optimization](https://ar5iv.labs.arxiv.org/html/2504.13173)
- [Atlas: Learning to Optimally Memorize the Context at Test Time](https://ar5iv.labs.arxiv.org/html/2505.23735)
- [Test-Time Training Done Right (LaCT)](https://arxiv.org/html/2505.23884)
- [Nested Learning: The Illusion of Deep Learning Architecture](https://arxiv.org/html/2512.24695v1)
- [End-to-End Test-Time Training for Long Context](https://arxiv.org/html/2512.23675v2)
- [Using Fast Weights to Deblur Old Memories](https://ni.cmu.edu/~plaut/papers/txt/HintonPlaut87CogSciConf.fastWeights.txt)
- [Using Fast Weights to Attend to the Recent Past](https://ar5iv.labs.arxiv.org/html/1610.06258)
- [Linear Transformers Are Secretly Fast Weight Programmers](https://ar5iv.labs.arxiv.org/html/2102.11174)
- [Meta-Learning Fast Weight Language Models](https://ar5iv.labs.arxiv.org/html/2212.02475)
- [Transformers Learn In-Context by Gradient Descent](https://ar5iv.labs.arxiv.org/html/2212.07677)
- [A Comprehensive Survey of Continual Learning](https://ar5iv.labs.arxiv.org/html/2302.00487)
- [Google Research: Titans + MIRAS, Helping AI have long-term memory](https://research.google/blog/titans-miras-helping-ai-have-long-term-memory/)
- [Google Research: Introducing Nested Learning](https://research.google/blog/introducing-nested-learning-a-new-ml-paradigm-for-continual-learning/)
- [NVIDIA Technical Blog: Reimagining LLM Memory](https://developer.nvidia.com/blog/reimagining-llm-memory-using-context-as-training-data-unlocks-models-that-learn-at-test-time)
- [VentureBeat: New test-time training method lets AI keep learning without exploding](https://venturebeat.com/infrastructure/new-test-time-training-method-lets-ai-keep-learning-without-exploding)
- [MIT News: Study could lead to LLMs that are better at complex reasoning](https://news.mit.edu/2025/study-could-lead-llms-better-complex-reasoning-0708)
- [MIT News: Teaching large language models to absorb new knowledge](https://news.mit.edu/2025/teaching-large-language-models-to-absorb-new-knowledge-1112)
- [a16z: Why We Need Continual Learning](https://a16z.com/why-we-need-continual-learning/)
- [Titans Revisited: A Lightweight Reimplementation and Critical Analysis of a Test-Time Memory Model](https://arxiv.org/abs/2510.09551)
- [Letta: Continual Learning in Token Space](https://www.letta.com/blog/continual-learning)
- [Position: Modular Memory is the Key to Continual Learning Agents](https://arxiv.org/html/2603.01761)
- [Panini: Continual Learning in Token Space via Structured Memory](https://arxiv.org/html/2602.15156)
- [Memory for Large Language Models](https://arxiv.org/html/2607.25380v1)
- [Beyond Perplexity: A Behavioral Evaluation Framework for Deployment-Memory Claims in LLM Test-Time Training](https://arxiv.org/html/2607.00368)
- [SR-TTT: Surprisal-Aware Residual Test-Time Training](https://arxiv.org/html/2603.06642v1)
- [Fast-weight Product Key Memory](https://arxiv.org/html/2601.00671)
- [Ian Barber: Test Time Training](https://ianbarber.blog/2026/09/02/test-time-training/)
- [GitHub: test-time-training/e2e](https://github.com/test-time-training/e2e)
- [GitHub: test-time-training/e2e issue 8](https://github.com/test-time-training/e2e/issues/8)
- [GitHub: test-time-training/ttt-lm-pytorch](https://github.com/test-time-training/ttt-lm-pytorch)
- [GitHub: test-time-training/ttt-lm-pytorch issue 21](https://github.com/test-time-training/ttt-lm-pytorch/issues/21)
- [GitHub: a1600012888/LaCT](https://github.com/a1600012888/LaCT)
- [GitHub: lucidrains/titans-pytorch issue 2](https://github.com/lucidrains/titans-pytorch/issues/2)
- [GitHub: lucidrains/titans-pytorch issue 4](https://github.com/lucidrains/titans-pytorch/issues/4)
- [GitHub: lucidrains/titans-pytorch issue 6](https://github.com/lucidrains/titans-pytorch/issues/6)
- [GitHub: lucidrains/titans-pytorch issue 29](https://github.com/lucidrains/titans-pytorch/issues/29)
- [GitHub: lucidrains/titans-pytorch issue 32](https://github.com/lucidrains/titans-pytorch/issues/32)
- [GitHub: kmccleary3301/nested_learning issue 12](https://github.com/kmccleary3301/nested_learning/issues/12)
- [GitHub: kmccleary3301/nested_learning issue 13](https://github.com/kmccleary3301/nested_learning/issues/13)
- [GitHub: ekinakyurek/marc issue 3](https://github.com/ekinakyurek/marc/issues/3)
- [Hacker News: Google Titans architecture, helping AI have long-term memory](https://news.ycombinator.com/item?id=46181231)
- [Hacker News: Nested Learning, A new ML paradigm for continual learning](https://news.ycombinator.com/item?id=46182031)
- [Hacker News: Nested Learning 초기 제출 스레드](https://news.ycombinator.com/item?id=45849996)
- [Hacker News: Titans, Learning to Memorize at Test Time](https://news.ycombinator.com/item?id=42718166)
- [Hacker News: A sleep-like consolidation mechanism for LLMs](https://news.ycombinator.com/item?id=48281226)
- [Hacker News: What is the status on continual learning for LLMs?](https://news.ycombinator.com/item?id=49050360)
- [Hacker News: 가중치 수준 continual learning의 사업 모델 문제](https://news.ycombinator.com/item?id=49709667)
- [Hacker News: Show HN Mini-AGI 연속 학습 모델](https://news.ycombinator.com/item?id=49783133)
