---
layout: post
title: "LLM Pretraining Loss Spike 진단과 처방: 배치 사이즈, Gradient Accumulation, 그리고 Muon 전환 판단"
author: 'Juho'
date: 2026-08-19 00:00:00 +0900
categories: [LLM]
tags: [LLM, GPU, Benchmark]
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
2. [진단: 이 스파이크는 정상인가](#진단-이-스파이크는-정상인가)
   - [개입하지 않아도 되는 스파이크와 개입해야 하는 스파이크](#개입하지-않아도-되는-스파이크와-개입해야-하는-스파이크)
   - [gradient norm은 loss보다 먼저 튄다](#gradient-norm은-loss보다-먼저-튄다)
   - [소규모 모델의 스파이크는 원인이 다르다](#소규모-모델의-스파이크는-원인이-다르다)
3. [원인: Adam과 weight decay가 만드는 불안정](#원인-adam과-weight-decay가-만드는-불안정)
   - [2차 모멘트 감쇠와 epsilon](#2차-모멘트-감쇠와-epsilon)
   - [weight decay가 유발하는 weight norm 급등](#weight-decay가-유발하는-weight-norm-급등)
   - [데이터 배치 단독 원인이 아니다](#데이터-배치-단독-원인이-아니다)
4. [배치 사이즈와 gradient accumulation](#배치-사이즈와-gradient-accumulation)
   - [critical batch size가 말하는 것](#critical-batch-size가-말하는-것)
   - [반론: 배치가 아니라 beta2가 문제다](#반론-배치가-아니라-beta2가-문제다)
   - [gradient accumulation이 주는 것과 주지 않는 것](#gradient-accumulation이-주는-것과-주지-않는-것)
5. [AdamW에서 Muon으로 갈아탈 것인가](#adamw에서-muon으로-갈아탈-것인가)
   - [Muon head의 실체는 하이브리드 옵티마이저다](#muon-head의-실체는-하이브리드-옵티마이저다)
   - [Moonlight와 MuonClip이 추가한 것](#moonlight와-muonclip이-추가한-것)
   - [전환 판단 기준](#전환-판단-기준)
6. [실무 처방 체크리스트](#실무-처방-체크리스트)
7. [한계와 주의사항](#한계와-주의사항)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

LLM을 from scratch로 pretraining하다 보면 loss가 갑자기 튀었다가 몇십 step 뒤에 원래 궤적으로 돌아오는 현상을 만나게 된다.
동시에 가중치의 L1 norm이 함께 급등하는 경우도 흔하다.
이 글은 그 현상을 "정상인가 아닌가"라는 이분법 대신, 진단 지표와 원인 가설, 그리고 개입 수단의 비용 대비 효과로 정리한다.

논의의 기준 설정은 다음과 같다.
2장의 GPU에서 micro-batch 26, gradient accumulation 4로 학습하는 경우 유효 배치는 26 × 4 × 2 = 208 시퀀스가 된다.
block size 1024를 가정하면 약 0.21M 토큰이며, 이는 nanoGPT의 GPT-2 124M 기본 설정인 0.5M 토큰의 절반 이하다.

| 항목 | nanoGPT 기본값 | 기준 설정 |
|------|----------------|-----------|
| micro-batch | 12 | 26 |
| gradient accumulation | 40 (8 GPU 환산 5) | 4 |
| GPU 수 | 8 | 2 |
| 유효 배치 (시퀀스) | 480 | 208 |
| 유효 배치 (토큰, block 1024) | 약 491,520 | 약 213,000 |

이 글이 답하려는 질문은 다섯 가지다.
스파이크를 무시해도 되는가, weight norm 급등은 왜 생기는가, 배치를 512에서 2048로 키우면 정말 안정화되는가, gradient accumulation은 배치를 키우는 것과 같은가, 그리고 AdamW에서 Muon으로 갈아타는 것이 지금 시점에 합리적인가.

## 진단: 이 스파이크는 정상인가

### 개입하지 않아도 되는 스파이크와 개입해야 하는 스파이크

대규모 학습에서 loss spike는 예외가 아니라 기본값에 가깝다.
PaLM 540B는 gradient clipping을 global norm 1.0으로 켜 둔 상태에서도 학습 중 약 20회의 스파이크를 겪었고, 발생 간격은 매우 불규칙했으며 학습 후반부에도 나타났다.
OPT-175B는 2개월에 걸쳐 최소 35회의 수동 재시작을 겪었고 12.xx 계보 전체로는 약 90회의 재시작이 기록되어 있다.
DeepSeek-V3가 "회복 불가능한 loss spike나 롤백이 전혀 없었다"고 명시적으로 밝힌 것 자체가, 스파이크 없는 학습이 얼마나 드문지를 역설한다.

반면 소규모 환경에서도 스파이크는 학습 예산을 실제로 태운다.
RTX 3090 한 장으로 GPT-2급 모델을 학습한 실측 연재에서는 global step 약 4,200과 13,000, 23,000 부근에서 스파이크가 관측되었고, 회복에 학습 시간의 10~15%가 소요되었다고 보고되었다.
즉 "알아서 회복하니 무시하라"는 조언은 컴퓨트 비용을 무시한 조언이다.

개입 여부를 판단하는 기준은 다음과 같이 정리된다.

| 상황 | 판단 |
|------|------|
| rolling average 대비 2~3배 수준이고 10~20 step 내 자가 회복 | 개입 불필요, 롤백은 오히려 파괴적 |
| 스파이크 후 loss가 원래 궤적으로 돌아오지 않음 | 즉시 개입 |
| gradient norm이 무한 또는 NaN으로 반복 발생 | 즉시 개입 |
| 회복에 전체 학습 시간의 10% 이상을 소모 | 하이퍼파라미터 재검토 |

회복하지 못한 스파이크의 실제 사례가 있다.
BERT-large급 모델(vocab 50,000, hidden 1024, intermediate 4096, heads 16, layers 20)을 A100 40GB 한 장에서 학습하던 중 loss가 8.0 근처로 튄 뒤 10시간 동안 전혀 내려오지 않았다.
레이어별 gradient 분포를 플롯한 결과 스파이크 이후 gradient가 붕괴한 vanishing gradient 상태였고, 스파이크는 원인이 아니라 증상이었다.
해결책은 배치를 키우는 것이 아니라 warmup을 전체 학습의 10%로 연장하고 learning rate를 5e-5에서 4e-5로 낮추는 것이었다.

### gradient norm은 loss보다 먼저 튄다

진단에서 가장 실용적인 사실은 gradient norm이 조기 경보 지표라는 점이다.
GLM-130B 팀은 gradient norm이 training collapse의 유용한 지표이며, collapse가 gradient norm 스파이크보다 몇 step 뒤에 따라온다고 명시했다.
같은 팀은 임베딩 레이어의 gradient가 다른 레이어보다 수 자릿수 크고 심하게 요동친다는 것도 관측했다.

OPT-175B 로그북에서도 같은 패턴이 반복된다.
run 12.43에서는 gradient norm과 activation norm이 튀기 시작하는 것을 먼저 포착해 learning rate를 10% 낮추는 것으로 안정화했고, run 12.45.2에서는 큰 gradient norm 스파이크 몇 번 뒤에 perplexity가 발산하기 시작해 learning rate를 GPT-3 175B가 쓴 값의 2/3까지 낮췄다.
논문 쪽에는 loss divergence와 dynamic loss scalar가 0으로 떨어지는 것, 최종 레이어 activation의 L2 norm이 튀는 것 사이에 상관관계가 있다고 기록되어 있다.

따라서 loss만 로깅하고 있다면 진단이 늦어진다.
최소한 global gradient norm, 임베딩 레이어의 gradient norm, 최종 레이어 activation의 L2 norm, max attention logit을 함께 기록해야 한다.
mixed precision을 쓰는 경우 GradScaler가 non-finite gradient 업데이트를 조용히 스킵하면서 스파이크를 가려버리는 문제도 보고되어 있어, 스킵된 step 수 자체를 로깅하는 것이 좋다.

### 소규모 모델의 스파이크는 원인이 다르다

PaLM 논문에는 자주 인용되지 않지만 소규모 학습에 결정적인 단서가 있다.
스파이크가 "더 작은 모델을 학습할 때는 관측되지 않았다"는 문장이다.
즉 1B 미만 모델에서 스파이크가 보인다면 PaLM류의 "데이터 배치와 파라미터 상태의 결합"보다 초기화, learning rate, beta2, 수치 정밀도를 먼저 의심하는 것이 순서상 맞다.

한편 Spike No More 논문은 반대 방향의 관측을 내놓는다.
기본 초기화에서는 hidden dimension d가 커질수록 임베딩의 표준편차가 작아지므로, 350M보다 1.7B에서 스파이크가 더 자주 발생했다는 것이다.
두 관측을 종합하면, 스케일 자체가 단조롭게 스파이크 확률을 결정하는 것이 아니라 초기화 스킴과 스케일의 조합이 결정한다는 해석이 자연스럽다.

수치 정밀도도 무시할 수 없는 변수다.
nanoGPT 기반 실험에서 동일 설정 188 run 중 18건(약 10%)이 BF16에서 발산한 반면, 서로 다른 시드로 학습한 TF32 70 run에서는 발산 사례가 0건이었다는 보고가 있다.
반면 BLOOM은 V100과 IEEE float16 조합에서 되돌릴 수 없는 발산을 겪은 뒤 A100 확보 후 bf16으로 전환했고, 최종 학습에서 스파이크는 단 1회에 그쳤다.
정리하면 fp16의 좁은 dynamic range는 명확한 위험이고, bf16은 그보다 안전하지만 무결점은 아니다.

## 원인: Adam과 weight decay가 만드는 불안정

### 2차 모멘트 감쇠와 epsilon

Adam 계열에서 스파이크가 발생하는 인과 사슬은 최근 연구에서 비교적 명확하게 정식화되었다.
gradient가 작은 구간이 오래 지속되면 2차 모멘트 추정치가 beta2 비율로 계속 감쇠해 매우 작아진다.
이 상태에서 큰 gradient가 들어오면 분모가 지나치게 작아 비정상적으로 큰 업데이트가 발생하고 그것이 loss spike로 나타난다.
스파이크 도중에는 gradient norm이 커지는데도 2차 모멘트 추정치의 반응이 느려 계속 작게 유지되므로 스파이크가 증폭된다.
얕은 레이어와 임베딩 행렬이 특히 취약한데, 작은 gradient 구간이 길게 지속되기 때문이며 이는 GLM-130B의 임베딩 관측과 일치한다.
해당 연구가 제시한 처방은 스파이크 시점에 Adam의 epsilon을 기본값 1e-8에서 0.1로 크게 올리는 것이다.

방향이 정반대인 처방도 존재한다.
소규모 프록시 실험을 다룬 Wortsman 등의 연구는 가장 큰 스케일과 learning rate 조건에서 gradient RMS가 AdamW 기본 epsilon 값 근처까지 내려간다는 것을 관측하고, 4.8B 모델에 learning rate 0.3을 적용한 조건에서 epsilon을 1e-15로 낮추면 loss가 개선된다고 보고했다.
Molybog 등의 이론 분석도 gradient가 epsilon보다 훨씬 작아지면 얕은 층 파라미터가 사실상 얼어붙고, 드물게 큰 gradient가 들어올 때 연쇄가 시작된다고 설명한다.

두 처방은 서로 다른 조건을 전제한다.
epsilon 상향은 "작은 2차 모멘트에 의한 업데이트 폭주"를 막으려는 것이고, epsilon 하향은 "epsilon이 분모를 지배해 업데이트가 죽는 것"을 막으려는 것이다.
따라서 조정 전에 gradient RMS의 실제 분포를 로깅해 어느 쪽 영역에 있는지 확인하는 것이 순서다.

Molybog 등의 분석은 또 하나의 진단 지표를 제공한다.
Adam은 파라미터 업데이트 벡터의 norm이 상대적으로 크면서도 하강 방향과 사실상 무상관인 상태에 진입할 수 있으며, 이때 업데이트 비율의 분포가 종 모양에서 쌍봉 형태로 전이한다는 것이다.
저자들의 결론은 "은탄환은 없다"였고, 권고 목록에는 learning rate 인하, epsilon 조정, beta1과 beta2 조정, PaLM식 배치 스킵이 함께 들어 있다.

### weight decay가 유발하는 weight norm 급등

가중치의 L1 norm이 갑자기 튀는 증상은 최근 연구에서 직접 다뤄진다.
LayerNorm 같은 정규화 레이어가 만드는 scale-invariant 가중치는 weight decay에 의해 norm이 계속 0쪽으로 밀리고, 그 결과 Hessian의 최대 고유값이 norm의 제곱에 반비례해 폭증한다.
손실 지형이 급격히 sharp해지고 임계선을 넘으면 스파이크로 나타나며, 실증적으로도 scale-invariant 레이어의 norm이 다른 파라미터보다 훨씬 빠르게 감쇠하는 것이 관측되었다.
처방은 MLP 모듈의 weight decay를 해제하는 것으로, 187M Transformer를 100B 토큰으로 학습한 실험에서 스파이크 빈도가 크게 감소하고 수렴도 개선되었다.

같은 현상을 다른 각도에서 설명하는 관측도 있다.
Transformer 출력의 RMS가 커지면 최종 LayerNorm에 들어가는 입력 RMS가 커지고, LayerNorm의 gradient는 입력 RMS의 역수로 스케일되므로 Transformer가 받는 gradient가 줄어든다.
gradient가 줄면 Adam의 분모가 작아지고, 그 상태에서 큰 gradient가 들어오면 업데이트가 폭주할 조건이 만들어진다.
즉 weight norm 급등과 loss spike는 별개의 두 사건이 아니라 하나의 되먹임 고리로 연결된다.

실무적으로 확인할 지점은 weight decay 값 자체다.
scale-invariant 가중치의 norm 증가 여부를 가르는 임계값이 weight decay 0.01과 0.001 사이에 있으며, 널리 쓰이는 기본값 0.01이 unbounded growth를 막지 못한다는 관측이 반복된다.
"AdamW를 쓰고 있으니 weight norm은 통제된다"는 가정은 성립하지 않는다.
weight decay를 learning rate와 독립적으로 파라미터화하면 learning rate 민감도가 줄어든다는 보고도 함께 참고할 만하다.

### 데이터 배치 단독 원인이 아니다

"나쁜 데이터가 원인"이라는 가설은 PaLM 팀의 ablation에서 직접 반증되었다.
스파이크 주변의 데이터 배치들을 가져와 더 이른 시점의 다른 체크포인트에서 학습시켰을 때는 스파이크가 발생하지 않았다.
즉 스파이크는 특정 데이터 배치 단독이 아니라 특정 데이터 배치와 특정 파라미터 및 옵티마이저 상태의 결합에서만 발생한다.

PaLM의 대응은 스파이크 시작 약 100 step 이전 체크포인트로 롤백한 뒤, 스파이크 직전과 직후에 소비된 약 200~500개의 데이터 배치를 건너뛰는 것이었다.
이 조치 후 같은 지점에서 스파이크가 재현되지 않았다.

이 방식은 확실하지만 무겁다.
loss를 사람이 계속 지켜봐야 하고, 롤백에 드는 컴퓨트와 잦은 체크포인트 저장 비용이 발생하며, 배치 스킵은 학습 초반에만 잘 통한다는 관찰도 있다.
2장 GPU 규모에서 우선 시도할 수단은 아니다.

참고로 PaLM의 하이퍼파라미터는 다음과 같다.

| 항목 | 값 |
|------|-----|
| Optimizer | Adafactor (factorization 미사용) |
| Learning rate | 초기 1e-2, 이후 1/sqrt(k) 감쇠 |
| beta1 | 0.9 |
| beta2 | 1 − k의 -0.8제곱 |
| Gradient clipping | global norm 1.0 |
| Auxiliary loss | z_loss 계수 1e-4 |
| Weight decay | 동적, learning rate의 제곱 |
| Batch size 스케줄 | 512에서 시작해 step 50k에 1024, step 115k에 2048 |
| Dropout | 미사용 |

## 배치 사이즈와 gradient accumulation

### critical batch size가 말하는 것

"배치를 512에서 2048로 키워라"는 조언에는 명확한 1차 근거가 있다.
Pre-training에서 critical batch size가 어떻게 스케일되는지를 다룬 2024년 연구는 linear scaling 대비 20% 오버헤드가 발생하는 지점을 CBS로 정의하고, 85M부터 1.2B까지의 모델을 3.072B에서 375B 토큰 구간에서 학습해 측정했다.
결론은 "1B 미만 모델은 통상적인 compute-optimal 배분에서 2의 9제곱부터 2의 11제곱, 즉 512에서 2048이 적정하다"는 것이다.

같은 연구의 더 중요한 발견은 CBS가 모델 크기가 아니라 데이터 크기에 스케일한다는 점이다.
데이터 크기를 고정한 채 모델만 키우면 CBS는 거의 불변이고, 모델을 고정한 채 학습 기간을 늘리면 CBS가 크게 증가한다.
피팅된 스케일링 법칙은 데이터 크기 D(단위 billion)에 대해 B* = 22.91 × D의 0.47제곱이다.

Kaplan 등의 초기 스케일링 연구도 critical batch size가 모델 크기와 독립이며 loss에만 의존한다고 보았고, loss가 13% 감소할 때마다 critical batch size가 대략 두 배가 된다고 정리했다.
McCandlish 등의 gradient noise scale 분석은 그 이유를 설명한다.
노이즈 스케일의 분모가 gradient의 global norm 제곱이므로, 학습이 진행되며 gradient가 작아지면 critical batch size는 학습 전 구간에 걸쳐 한 자릿수 이상 커진다.
PaLM의 512에서 2048 스케줄, GLM-130B의 192에서 4224 스케줄이 모두 이 성질에 대응한다.

여기서 실무적으로 중요한 경고가 나온다.
OLMo 1B와 7B에서 CBS를 체크포인트 분기 학습으로 직접 측정한 결과, CBS는 0 근처에서 시작해 급격히 증가한 뒤 배치 4096 부근에서 정체했다.
그리고 처음부터 큰 고정 배치를 쓰면 학습 초기에 loss가 오히려 나빠졌다.
배치를 점진적으로 키우는 batch size warmup을 적용했을 때 OLMo 1B는 동등한 loss까지 43% 더 적은 gradient step으로 도달했다.
같은 연구는 gradient noise 기반 CBS 프록시가 신뢰할 수 없다고 결론지었다는 점도 함께 기억할 필요가 있다.

배치를 정하는 최신 관점은 모델 크기가 아니라 데이터셋 크기 기준이다.
weight decay와 배치 사이즈의 스케일링 법칙을 다룬 연구는 최적 배치와 critical batch size가 모두 데이터셋 크기의 멱법칙으로 스케일되고 모델 크기와는 독립이며, 최적 weight decay는 배치 사이즈에 선형으로 스케일된다고 정리했다.
Karpathy의 nanochat은 이 결과를 코드에 반영해 배치 사이즈 자동 스케일링을 구현했고, 최적 배치 사이즈 스케일 지수는 0.383이다.
토큰이 10배 늘어도 배치는 약 2.4배만 키우면 된다는 뜻이다.

learning rate 동반 조정에는 주의가 필요하다.
배치를 k배 키울 때 learning rate를 k배 키우는 linear scaling rule은 SGD 기준으로 유도된 것이고, 초기 학습처럼 네트워크가 빠르게 변하는 구간에서는 성립하지 않아 warmup이 필요하다.
Adam 계열에서는 learning rate가 배치의 alpha제곱에 비례하며 alpha가 0.5와 1.0 사이라는 경험식이 더 적절하고, 실무에서는 제곱근 스케일링이 흔히 쓰인다.
Ai2는 배치 2배당 learning rate를 sqrt(2)배로 키우는 규칙을 warmup 구간에 적용했다.

### 반론: 배치가 아니라 beta2가 문제다

정반대 방향의 최신 결과가 있다.
NeurIPS 2025에 채택된 소배치 학습 연구는 배치 1까지 내려가도 안정적으로 학습되며, 하이퍼파라미터 선택에 오히려 더 강건하고, per-FLOP 성능이 큰 배치와 같거나 낫다고 주장한다.
선행 연구가 보고한 소배치 loss spike와 심한 불안정은 하이퍼파라미터를 잘못 잡았을 때만 발생한다는 것이 이 논문의 반박이다.
실제로 배치 1에서 beta2를 0.9999로 재조정하자 loss 곡선이 매끄러워졌고, 흔히 인용되는 U자형 배치 사이즈 성능 곡선이 재현되지 않았다.

핵심 규칙은 beta2를 배치와 무관하게 0.95나 0.98로 고정하지 말고, 2차 모멘트의 half-life를 토큰 단위로 일정하게 유지하라는 것이다.
배치 B에서 B*로 바꿀 때 beta2를 다음처럼 다시 계산한다.

```python
# 2507.07101: token half-life를 고정하도록 beta2를 재계산한다.
# beta2_new = beta2_ref ** (batch_new / batch_ref)

beta2_ref, batch_ref = 0.95, 512
batch_new = 208                      # micro 26 x accum 4 x 2 GPU

beta2_new = beta2_ref ** (batch_new / batch_ref)
print(beta2_new)                     # 0.9794...
```

배치 512 기준으로 beta2가 0.95일 때, 배치를 208로 줄이면 beta2를 오히려 0.9794로 키워야 토큰 기준 half-life가 유지된다는 계산이다.
beta1은 0.9로 고정해도 배치 크기와 무관하게 잘 스케일된다고 같은 논문이 권고한다.
learning rate는 흔히 권장되는 제곱근보다 더 느리게 스케일해야 한다는 것도 이 논문의 결론이다.

여기서 실무자 사이의 통념과 충돌이 발생한다.
현대적 LLM 학습 레포에서 "beta1과 beta2를 모두 0.95로 두라, 기본값 0.9와 0.999는 시대에 뒤떨어졌다"는 권고가 널리 퍼져 있고, OPT-175B도 스파이크 대응 과정에서 beta2를 0.98에서 0.95로 낮췄다.
그러나 OPT의 배치는 2M 토큰이었고 소배치 논문이 다루는 영역은 그보다 훨씬 아래다.
같은 beta2 값이 배치에 따라 전혀 다른 의미를 갖는다는 것이 이 대립의 본질이다.

Molybog 등의 분석은 또 다른 축의 반론을 제공한다.
배치가 크면 gradient 평가의 시간축 분산이 작아지고, 연속 스텝 간 gradient가 비슷해지며 시간축 상관이 강해져 Adam이 쌍봉 영역에 진입한다는 논리다.
그래서 이 논문의 권고 목록에는 배치를 줄이라는 항목이 들어 있다.
다만 이는 175B 규모와 이미 매우 큰 배치를 전제한 논의이므로, 208 시퀀스 배치에 그대로 적용되지는 않는다.

조건별로 정리하면 다음과 같다.

| 조건 | 타당한 처방 | 근거 |
|------|-------------|------|
| 1B 미만 모델, 유효 배치가 512 미만 | 배치를 512~2048로 키우되 점진 증가 | CBS 스케일링 연구, Ai2 OLMo |
| 배치를 키울 VRAM이나 GPU가 없음 | beta2를 토큰 half-life 기준으로 재조정 | 소배치 학습 연구 |
| 배치를 바꿨는데 스파이크가 그대로 | beta2를 함께 바꾸지 않았을 가능성 | 소배치 학습 연구 |
| 이미 배치가 수 M 토큰 규모 | 배치 확대가 오히려 시간축 상관을 키울 수 있음 | Molybog 등의 Adam 불안정 분석 |

### gradient accumulation이 주는 것과 주지 않는 것

gradient accumulation이 배치를 키우는 것과 같은가에 대한 답은 세 층위로 나뉜다.

첫째, 수학적 등가성 층위다.
2024년 10월 HuggingFace Trainer의 gradient accumulation이 큰 배치와 수학적으로 동등하지 않다는 것이 발견되었다.
cross-entropy는 전체 non-padding 토큰 수로 정규화되는데, accumulation을 나이브하게 합하면 짧은 시퀀스가 과대 가중되고 긴 시퀀스가 과소 가중된다.
길이가 각각 1과 100인 두 텍스트는 full batch에서 1/(100+1)로 스케일되지만, accumulation 2단계에서는 각각 1/2과 1/200으로 가중된다.
최종 합은 원래 full batch loss의 G배가 되며, 여기서 G는 accumulation step 수다.
정량적으로는 accumulation step이 늘어남에 따라 L2 norm 차이가 0.0196에서 0.0286으로 증가했고, 수정 후 오차가 한 자릿수 이상 줄었다.

파급 범위가 특히 중요하다.
gradient accumulation을 명시적으로 쓰지 않아도 multi-GPU 학습이면 영향을 받았는데, 디바이스 간 gradient가 backprop 전에 누적되기 때문이다.
2GPU와 accumulation 4를 함께 쓰는 구성은 이 버그 계열이 정확히 저격하는 조합이다.
게다가 이 문제는 한 번으로 끝나지 않았고, 2025년 9월 transformers 4.56.0에서 logged loss가 gradient_accumulation_steps에 비례해 커지는 이슈가 다시 제기되었다.

다만 조건을 분명히 해야 한다.
이 비등가성은 시퀀스 길이가 가변일 때 발생한다.
고정 block size로 packing된 pretraining에서는 분모가 동일하므로 loss를 accumulation step 수로 나누는 것으로 충분하다.
그래도 프레임워크와 버전, 그리고 실제 tokens per iteration 로그는 반드시 확인해야 한다.
nanoGPT는 accumulation step을 DDP world size로 나누는 규약을 쓰기 때문에, 같은 설정값이 프레임워크마다 다른 유효 배치를 의미할 수 있다.

```python
# nanoGPT 규약: accumulation step을 GPU 수로 나눈 뒤 유효 배치를 로깅한다.
assert gradient_accumulation_steps % ddp_world_size == 0
gradient_accumulation_steps //= ddp_world_size

tokens_per_iter = gradient_accumulation_steps * ddp_world_size * batch_size * block_size
print(f"tokens per iteration will be: {tokens_per_iter:,}")

for micro_step in range(gradient_accumulation_steps):
    with ctx:
        logits, loss = model(X, Y)
        loss = loss / gradient_accumulation_steps   # accumulation 정규화
    X, Y = get_batch('train')
    scaler.scale(loss).backward()

if grad_clip != 0.0:
    scaler.unscale_(optimizer)                      # clip 전에 반드시 unscale
    torch.nn.utils.clip_grad_norm_(model.parameters(), grad_clip)
```

PyTorch AMP 공식 문서는 clipping 직전에 unscale_을 호출하라고 명시한다.
unscale_ 없이 clipping하면 스케일된 gradient를 클리핑하게 되어 임계값이 무의미해진다.

둘째, 실효 층위다.
RTX 3090 한 장에서 micro-batch 6과 accumulation 16으로 유효 배치 96을 구성해 학습한 결과 test loss는 3.683835였고, 8×A100 클라우드에서 같은 유효 배치 96으로 학습한 결과는 3.691526이었다.
차이는 0.0077로 사실상 동일하다.
저자의 결론은 명확하다.
큰 배치의 학습 안정화 이득은 얻지만 성능(속도) 이득은 얻지 못한다는 것이다.
다만 gradient norm 거동은 달랐고, 로컬 실행이 더 노이지했으며 global step 10,000 부근에 큰 스파이크가 있었던 반면 클라우드 실행은 step 6,000 부근부터 안정화되었다.

셋째, 권장 여부 층위다.
소배치 학습 연구는 "여러 디바이스에 모델 복제본이 있고 디바이스 간 대역폭이 병목인 경우가 아니면 gradient accumulation을 권장하지 않는다"고 명시한다.
누적 gradient를 저장하느라 메모리를 더 쓰면서 얻는 이득이 없다는 것이며, 권고는 "디바이스 처리량을 최대화하는 가장 작은 배치를 쓰라"이다.
HuggingFace 공식 문서도 accumulation이 추가 forward/backward pass를 유발해 학습을 느리게 만들 수 있으니 너무 많은 단계를 피하라고 경고하며, 같은 유효 배치 64를 만들 때 batch 1과 accum 64보다 batch 4와 accum 16이 낫다고 안내한다.
대규모 학습 현장에서는 accumulation 대신 데이터/파이프라인/텐서 병렬로 GPU를 더 붙이는 것이 정석이라는 관점도 반복된다.

정리하면 accumulation을 4에서 8이나 16으로 늘리는 것은 "안정화 측면에서는" 배치를 키우는 것과 사실상 같고, "처리량 측면에서는" 전혀 같지 않으며, "수학적으로는" 프레임워크 구현에 따라 다를 수 있다.

부수적으로 micro-batch 값 자체도 점검 대상이다.
HuggingFace 공식 문서는 Tensor Core 요건상 배치 크기를 2의 거듭제곱으로 두고 fp16은 8의 배수, A100은 64의 배수로 맞추라고 안내한다.
micro-batch 26은 8의 배수도 2의 거듭제곱도 아니므로, 24나 32로 바꾸면 GEMM 효율이 개선될 여지가 있다.

## AdamW에서 Muon으로 갈아탈 것인가

### Muon head의 실체는 하이브리드 옵티마이저다

Muon은 2D 파라미터 전용 옵티마이저다.
SGD-momentum 업데이트에 Newton-Schulz 반복을 후처리로 적용해 업데이트 행렬을 근사 직교화하며, 이는 SVD의 좌우 특이벡터 곱만 남기는 것과 등가다.
직교화의 의미는 업데이트에서 크기는 작지만 학습에 중요한 "희소한 방향"의 스케일을 키워주는 것으로 설명된다.

```python
# Newton-Schulz 5차 반복 (Muon 원본 구현)
a, b, c = (3.4445, -4.7750, 2.0315)
X = G.bfloat16()
X = X / (X.norm(dim=(-2, -1), keepdim=True) + 1e-7)   # spectral norm을 1 이하로
for _ in range(5):                                     # ns_steps = 5
    A = X @ X.mT
    B = b * A + c * A @ A
    X = a * X + B @ X
update = X * max(1, X.size(-2) / X.size(-1)) ** 0.5    # shape 보정
```

"Muon head 방식"이라 불리는 것의 실체는 옵티마이저 분리 설계다.
원문 docstring은 Muon을 hidden weight layer에만 써야 하며, 입력 임베딩과 최종 출력 레이어, 내부 gain과 bias는 AdamW 같은 표준 방법으로 최적화해야 한다고 명시한다.
근거는 modular norm 이론상 임베딩 레이어의 최적화 다이내믹스가 다른 레이어와 달라야 한다는 것이다.

```python
from muon import MuonWithAuxAdam

hidden_weights      = [p for p in model.body.parameters() if p.ndim >= 2]
hidden_gains_biases = [p for p in model.body.parameters() if p.ndim < 2]
nonhidden_params    = [*model.head.parameters(), *model.embed.parameters()]

param_groups = [
    dict(params=hidden_weights, use_muon=True,
         lr=0.02, weight_decay=0.01),                        # spectral norm 단위 LR
    dict(params=hidden_gains_biases + nonhidden_params, use_muon=False,
         lr=3e-4, betas=(0.9, 0.95), weight_decay=0.01),     # 기존 AdamW LR
]
optimizer = MuonWithAuxAdam(param_groups)
```

여기서 기존 Adam 실험과의 비교 가능성에 직결되는 사실이 드러난다.
Muon 기본 learning rate는 0.02이고 보조 AdamW는 3e-4로 약 66배 차이가 난다.
Muon의 learning rate는 "업데이트당 spectral norm 단위"로 정의되어 있어 단위 자체가 다르며, AdamW learning rate를 그대로 옮길 수 없다.
Muon 기타 기본값은 momentum 0.95, nesterov True, ns_steps 5, weight decay 0이다.

성능과 비용은 다음과 같이 보고되었다.

| 항목 | 결과 |
|------|------|
| CIFAR-10 speedrun 94% 도달 | 3.3에서 2.6 A100-초 |
| NanoGPT speedrun | 학습 속도 1.35배 개선 |
| 1.5B 모델로 GPT-2 XL 수준 도달 | 8×H100에서 10시간, AdamW는 13.3시간 |
| FLOP 오버헤드 (NanoGPT record) | 약 0.7% |
| FLOP 오버헤드 (Llama 405B 규모 가정) | 약 0.5% |
| Optimizer state 메모리 | AdamW의 절반 |

Newton-Schulz 반복 이전 단계에서 Muon은 그냥 SGD-momentum이므로 메모리 요구량이 같고, 결과적으로 optimizer state가 AdamW의 절반이다.
DeepSpeed 지원 문서의 실측으로는 Qwen2.5-3B 파인튜닝(8×A100 40GB, batch 32, ZeRO-2, bf16)에서 AdamW 34.5 GiB가 Muon 31.4 GiB로 약 9% 줄었다.
1~2 GPU 환경에서 Muon의 가장 실질적인 이점은 속도가 아니라 optimizer state 메모리라는 것이 실무자 다수 의견이며, 24GB VRAM에서 14B 모델을 full fine-tuning한 사례에서는 Muon을 4-bit로 양자화해 32-bit Adam 대비 optimizer state 메모리를 16배까지 줄였다는 보고도 있다.

### Moonlight와 MuonClip이 추가한 것

여기서 가장 중요한 균형점이 나온다.
Muon으로 갈아탄 것 자체가 안정성을 준 것이 아니라, 대규모에서 쓰려면 추가 장치가 필요했다.

Moonshot AI의 Moonlight는 두 가지를 추가했다.
첫째, 원본 Muon에는 weight decay가 없어 대규모 학습에서 가중치와 레이어 출력이 과도하게 커졌고, AdamW식 weight decay를 넣어 해결했다.
특히 RMSNorm의 gamma 파라미터에 weight decay를 거는 것이 레이어 출력 RMS 폭증 방지에 필수적이었다.
둘째, Muon의 업데이트 RMS가 행렬 shape에 따라 달라지는 문제를 보정했다.
행렬마다 max(A,B)의 제곱근을 곱한 뒤 전체 업데이트 RMS를 AdamW의 관측 범위인 0.2에서 0.4 사이에 맞췄으며, 최종 업데이트 규칙에 등장하는 상수 0.2가 이 정합의 핵심이다.
이 정합 덕분에 AdamW용으로 튜닝한 learning rate를 추가 탐색 없이 재사용할 수 있다는 것이 논문의 주장이다.

Moonlight의 정량 결과는 다음과 같다.

| 항목 | 값 |
|------|-----|
| 계산 효율 | AdamW 대비 약 52% FLOP으로 동등 성능 |
| 스케일링 실험 범위 | 399M에서 1.5B 파라미터, 8.92B에서 38.91B 토큰 |
| Moonlight 모델 | 총 16B, 활성 2.24B MoE, 5.7T 토큰 |
| 안정성 | loss 및 gradient norm 스파이크 관측되지 않음 |
| 초기 관측 | max attention logit이 100을 초과한 뒤 점차 정상화 |
| 분산 통신 오버헤드 | 분산 AdamW의 약 1.0~1.25배 |

Kimi K2는 여기서 한 걸음 더 나갔다.
논문은 "Muon 학습을 스케일업하면 attention logit 폭증으로 인한 학습 불안정이 드러나며, 이 문제는 실험상 Muon에서 더 자주 발생하고 AdamW에서는 덜 발생한다"고 명시한다.
중간 규모 실험에서 attention logit이 1000을 빠르게 초과했다.

QK-Clip은 사후 gradient clipping이 아니라 Q와 K 프로젝션 가중치 행렬을 직접 재스케일한다.
헤드별 최대 softmax 입력이 임계값 tau를 넘으면 해당 헤드에만 스케일을 적용해 개입을 최소화한다.

```python
tau = 100.0
# S_max[h] = (1/sqrt(d)) * max over batch, i, j of  Q_i^h · K_j^h^T
for h in heads:
    gamma = min(1.0, tau / S_max[h])
    W_qc[h] *= gamma ** 0.5     # non-rotary query
    W_kc[h] *= gamma ** 0.5     # non-rotary key
    W_qr[h] *= gamma            # rotary query
    # W_kr 는 헤드 간 공유되므로 손대지 않는다
```

결과는 15.5조 토큰 학습 전 구간에서 loss spike 0회다.
Kimi K2의 규모는 총 1.04조 파라미터 MoE에 활성 32.6B, global batch size 67M 토큰이며, learning rate는 warmup 후 2e-4 상수 구간을 거쳐 2e-5로 cosine decay하고 weight decay는 전 구간 0.1이다.
클리핑은 전체 학습 스텝의 약 30% 구간에서 활성화되었고 이후 max logit이 자연히 안정 범위로 수렴해 70k step 부근에서는 더 이상 발동하지 않았다.

즉 "Muon을 쓰면 스파이크가 없다"는 명제는 정확하지 않다.
정확한 명제는 "Muon에 weight decay와 RMS 정합, 그리고 QK-Clip 또는 qk-norm을 결합하면 대규모에서 스파이크 없는 학습이 가능했다"이다.

### 전환 판단 기준

이미 AdamW로 돌린 실험 자산이 있는 상태에서 Muon으로 갈아타는 것이 합리적인지는 다음 네 가지로 판단할 수 있다.

첫째, 배치 규모다.
배치 1 수준에서는 SGD, Adam, Adafactor, Muon이 비슷한 성능으로 수렴하고 옵티마이저 간 격차는 배치 512 이상에서 크게 벌어진다는 실증이 있다.
반대로 Muon의 데이터 효율 우위는 critical batch size를 한참 넘어선 대형 배치 영역에서도 유지되거나 오히려 커지며, 동일 loss 도달에 AdamW 대비 토큰을 10~15% 절약한다.
Newton-Schulz 오버헤드가 배치에 반비례하므로 배치가 커질수록 비용도 무시할 만해진다.
따라서 유효 배치 208 시퀀스 상태에서 옵티마이저만 교체하는 것의 기대 효과는 제한적이며, Muon의 이점은 배치를 키운 뒤에 커진다.

둘째, 모델 깊이다.
GNN 실험에서 2층 설정은 AdamW와 사실상 같았고, 8층 부근의 중간 깊이에서 Muon이 눈에 띄게 더 안정적이고 최종 결과도 좋았으며, 매우 깊은 경우에는 옵티마이저만으로는 충분하지 않았다는 균형 잡힌 보고가 있다.

셋째, 비교 가능성이다.
learning rate 단위가 spectral norm 기준으로 달라 AdamW 값을 그대로 옮길 수 없으므로, 기존 Adam 실험과의 직접 비교는 성립하지 않는다.
게다가 Adam으로 사전학습한 모델을 Muon으로 이어받거나 그 반대로 할 때 양 단계에서 같은 옵티마이저를 쓴 경우보다 성능이 떨어진다는 연구가 여럿 나와 있다.
전환한다면 새 실험 계보를 시작하는 것으로 보고 baseline을 다시 잡아야 한다.

넷째, 회의론의 무게다.
singular value를 랜덤 노이즈로 대체한 변형 옵티마이저가 Muon과 동등한 성능을 낸다는 결과는 Muon의 기하학적 서사가 성능의 실제 원인이 아닐 수 있음을 시사한다.
µP 정합 스케일링이 대중화 과정에서 잘못 구현되었고 그 오류가 프레임워크까지 전파되었다는 비판도 있으며, 이론적으로 올바른 스케일링은 fan-out과 fan-in 비의 제곱근이어야 하는데 실제 구현들이 다른 형태를 채택했다는 지적이다.
특정 아키텍처 분석에서 레이어별 실효 learning rate 오차가 +144.9%에서 +553.2%까지 벌어진다는 추정도 제시되었다.
"opaque training bug를 디버깅하는 데 쓸 시간이 옵티마이저로 아끼는 컴퓨트보다 크다"는 보수적 반론도 실무 감각으로는 무시하기 어렵다.

동시에 채택 곡선이 실험 단계를 지난 것도 사실이다.
NanoGPT speedrun에서 AdamW를 Muon으로 바꿔 35% 단축을 달성한 이후 7명의 연구자가 세운 12개 연속 기록 전부가 Muon을 유지했고, PyTorch 2.9가 Muon을 네이티브로 제공하며 DeepSpeed와 NVIDIA NeMo도 지원을 추가했다.
프로덕션 채택은 Kimi K2(1T), GLM-4.5(355B), INTELLECT-3(106B)로 이어졌고, PyTorch 공식 블로그는 Kimi K2와 GLM-5(744B)의 채택을 근거로 Muon이 Adam을 대체할 진지한 후보라고 표현했다.

## 실무 처방 체크리스트

우선순위는 비용 대비 효과 순이다.
0번부터 3번까지는 비용이 사실상 0이므로 배치나 옵티마이저를 바꾸기 전에 먼저 시도할 항목이다.

| 순서 | 처방 | 값 또는 방법 | 근거 |
|------|------|--------------|------|
| 0 | 진단 로깅 추가 | global grad norm, 임베딩 grad norm, 최종 레이어 activation L2 norm, max attention logit, GradScaler skip 횟수 | GLM-130B, OPT 로그북, Wortsman 2023 |
| 0 | 유효 배치 실측 | tokens per iteration을 직접 출력해 프레임워크 규약 확인 | nanoGPT 규약, HF accumulation 이슈 |
| 1 | Gradient clipping | global norm 1.0, 불안정하면 0.3까지 하향 | Megatron, DeepSpeed, TorchTitan, OPT-175B |
| 1 | Warmup 연장 | 최소 2000 step, 문제 시 전체 학습의 10%까지 | nanoGPT, Wortsman 2023, HF issue 26141 |
| 2 | beta2 재조정 | 토큰 half-life 고정 공식으로 재계산, beta1은 0.9 유지 | 소배치 학습 연구 |
| 2 | Adam epsilon 점검 | gradient RMS 분포를 먼저 확인한 뒤 상향 또는 하향 결정 | 2506.04805, Wortsman 2023 |
| 2 | Learning rate 인하 | 10% 하향부터, 심하면 2/3 수준까지 | OPT 로그북 |
| 3 | Weight decay 재검토 | 0.1 기준, MLP 모듈 해제 검토, learning rate와 독립 파라미터화 | Weight-norm Criticality, Wortsman 2023 |
| 3 | Dropout 해제 | pretraining에서는 0.0 | nanoGPT, PaLM, BLOOM |
| 3 | 정밀도 확인 | bf16 우선, fp16은 dynamic range 문제로 회피 | BLOOM, HF docs |
| 4 | z-loss 추가 | 계수 1e-4를 auxiliary loss로 | PaLM, Wortsman 2023 |
| 4 | qk-layernorm | Q와 K에 LayerNorm 적용 | Wortsman 2023 |
| 4 | 임베딩 처리 | Scaled Embed 또는 Embed LayerNorm | Spike No More, BLOOM |
| 5 | 적응형 clipping | ZClip 기준 EMA 계수 0.97, z-threshold 2.5, warmup 25 step | ZClip |
| 6 | 배치 사이즈 확대 | 1B 미만 모델은 512~2048, 단 고정이 아니라 점진 증가 | CBS 스케일링 연구, Ai2 OLMo |
| 7 | 옵티마이저 전환 | hidden 2D는 Muon, 임베딩과 head는 AdamW. QK-Clip 또는 qk-norm 동반 | Muon 원문, Moonlight, Kimi K2 |
| 8 | 롤백과 배치 스킵 | 스파이크 100 step 전 체크포인트 + 200~500 배치 스킵 | PaLM |

z-loss와 적응형 clipping은 다음처럼 구현된다.

```python
# z-loss: PaLM 및 Wortsman 계수 1e-4
# logits shape = (B, T, V)
logsumexp = torch.logsumexp(logits.float(), dim=-1)   # = log Z
z_loss = 1e-4 * (logsumexp ** 2).mean()
loss = ce_loss + z_loss
```

ZClip은 global gradient norm의 EMA 평균과 표준편차를 추적해 z-score 기반으로 이상을 탐지하고 적응적으로 클리핑한다.
LLaMA 1B를 SmolLM 코퍼스 50B 토큰으로 32×H100에서 학습한 실험에서 고정 임계 클리핑의 스파이크 6회가 0회로 줄었고, learning rate 3e-3 조건에서 고정 임계 5e-4 대비 동일 validation loss에 35% 더 빨리 도달해 18.6B 토큰을 절약했다.
텐서별 EMA 기반 적응 클리핑인 AdaGC는 Llama-2 7B의 spike score를 0.0333%에서 0%로, ERNIE 10B-A1.4B를 1T 토큰 학습에서 0.01%에서 0%로 낮추면서 GPU 시간을 1.24~4.48% 절감했다고 보고했다.

고정 임계 클리핑도 소규모에서 실제로 효과가 확인되었다.
관측된 unclipped norm이 1.1에서 3.0 범위였던 점을 보고 max_norm을 3.5로 설정한 실험에서 test loss가 3.692에서 3.678로 개선되었고, 큰 스파이크 3개가 작은 스파이크 2개로 줄었으며, 학습 시간 증가는 12,243초 중 100초에 그쳤다.
같은 실험에서 mixed precision의 GradScaler가 이미 non-finite gradient 업데이트를 조용히 스킵하고 있었다는 사실이 드러났고, clipping은 GradScaler가 잡지 못하는 "크지만 유한한" gradient를 잡아냈다.

## 한계와 주의사항

이 글에서 인용한 결과들은 서로 다른 스케일과 조건에서 얻어진 것이며, 그대로 이식하면 오히려 잘못된 결론에 도달할 수 있다.

첫째, 스케일 전이 문제다.
OPT-175B 로그북의 최종 노트는 "약 13B 미만의 작은 스케일에서 얻은 결과가 스케일업에서 반드시 성립하지는 않는다"고 기록한다.
역방향도 마찬가지다.
PaLM의 "데이터 배치와 파라미터 상태의 결합" 결론은 540B 규모의 관측이고, 같은 논문이 작은 모델에서는 스파이크가 관측되지 않았다고 명시했으므로 소규모 학습에 그대로 적용하기 어렵다.

둘째, 임베딩 처리에 대한 상충이다.
GLM-130B는 임베딩 gradient를 계수 0.1로 축소하는 Embedding Gradient Shrink를 처방했고 학습 로그에는 shrink 계수를 6000 step에 걸쳐 0.2에서 1로 워밍업한 복구 전략이 기록되어 있다.
반면 Spike No More는 임베딩 gradient를 분리하는 방식이 "large shortcut" 조건을 위반하기 때문에 여전히 스파이크가 발생한다고 반박하며, 임베딩의 표준편차를 1에 가깝게 키우는 Scaled Embed나 Embed LayerNorm을 권한다.
두 처방은 같은 문제를 반대 방향으로 다루므로 하나를 고르면 다른 하나를 함께 적용해서는 안 된다.

셋째, Adam epsilon 방향의 상충이다.
2차 모멘트 감쇠를 원인으로 보는 분석은 epsilon을 0.1까지 올리라 하고, gradient RMS가 epsilon 근처로 내려가는 것을 문제로 보는 분석은 1e-15까지 내리라 한다.
로깅 없이 값만 바꾸는 것은 도박에 가깝다.

넷째, 배치 확대 조언의 조건이다.
1B 미만 모델에 512~2048을 권하는 결론은 C4 데이터셋, context length 512, 고정 target validation loss까지 학습, 스케줄 대신 지수 가중 평균 사용, 배치별 beta2 튜닝이라는 특정 실험 프로토콜에서 도출되었다.
특히 "배치별로 beta2를 튜닝했다"는 조건이 중요하다.
beta2를 그대로 둔 채 배치만 키우면 그 결론이 재현되지 않을 수 있다.

다섯째, 소규모에서의 최적 배치가 항상 512 이상인 것도 아니다.
GPT-2급 모델을 로컬에서 학습한 실측 연재는 512부터 104, 6까지 시험한 뒤 해당 설정의 이상적 유효 배치를 약 97로 추정했다.
배치의 최적점은 모델과 데이터 스케일에 종속된다는 실증이다.
또한 배치를 키우는 것이 단조롭게 좋은 것도 아니어서, 매우 큰 배치 영역에서는 오히려 나빠지며 약간의 노이즈가 정규화 역할을 한다는 관점도 함께 제기된다.

여섯째, 이 글이 인용한 수치 중 일부는 확인이 필요하다.
토큰 half-life 기반 beta2 스케일링 공식은 원문 렌더링이 지수 표기와 곱셈 표기 사이에서 모호했으며, half-life 고정 원리상 지수 형태로 해석했다.
"7B는 loss spike 확률 약 6%, 1.3B는 약 15%"라는 수치는 검색 요약으로만 확보되어 원문에서 직접 확인되지 않았으므로 이 글에서는 결론 근거로 쓰지 않았다.
GLM-5의 Muon Split과 DeepSeek-V4의 Muon 채택은 PyTorch 공식 블로그의 인용을 경유한 정보다.

## 결론

loss가 튀었다가 회복되는 현상 자체는 대규모 학습에서 흔하지만, 소규모 pretraining에서 반복 관측된다면 무시할 신호가 아니다.
회복에 학습 시간의 10~15%가 소모된다는 실측이 있고, 회복하지 못하고 고착되는 사례도 존재한다.

weight의 L1 norm 급등은 별개 현상이 아니라 같은 되먹임 고리의 다른 얼굴이다.
정규화 레이어가 만드는 scale-invariant 가중치가 weight decay에 밀려 norm이 감쇠하면 손실 지형이 급격히 sharp해지고, 출력 RMS 증가는 LayerNorm을 통해 gradient를 축소시켜 Adam의 분모를 작게 만든다.
weight decay 기본값 0.01이 norm 증가를 막지 못한다는 관측이 있으므로 값 자체를 먼저 점검하고, MLP 모듈의 weight decay 해제를 실험 후보에 올릴 만하다.

유효 배치 208 시퀀스는 1B 미만 모델의 권장 범위인 512~2048보다 작으므로 배치 확대는 근거 있는 방향이다.
다만 고정된 큰 배치로 한 번에 옮기면 학습 초기 loss가 오히려 나빠질 수 있어 점진 증가가 낫고, 배치를 바꿀 때는 beta2를 토큰 half-life 기준으로 함께 조정해야 한다.
배치를 키울 자원이 없다면 beta2 재조정만으로도 상당 부분을 회수할 수 있다는 것이 NeurIPS 2025 결과의 요지다.

gradient accumulation을 8이나 16으로 늘리는 것은 안정화 측면에서 배치 확대와 거의 같은 효과를 주지만 처리량 이득은 주지 않는다.
그리고 2GPU와 accumulation을 함께 쓰는 구성은 loss 정규화 버그가 역사적으로 반복 발생한 영역이므로, 프레임워크 버전과 실제 tokens per iteration을 반드시 확인해야 한다.
고정 block size로 packing된 pretraining이라면 단순히 accumulation step 수로 나누는 것으로 충분하다.

Muon 전환은 지금 시점에서 우선순위가 높지 않다.
Muon head 방식의 실체는 hidden 2D 파라미터에만 Muon을 쓰고 임베딩과 LM head는 AdamW로 두는 하이브리드 구조이며, 대규모에서의 안정성은 옵티마이저 교체 단독이 아니라 weight decay와 RMS 정합, QK-Clip의 조합에서 나왔다.
Muon은 attention logit 폭증을 AdamW보다 더 자주 유발한다는 것이 Kimi K2 팀의 명시적 관측이므로, 도입한다면 qk-norm이나 QK-Clip을 함께 준비해야 한다.
배치 512 미만 영역에서는 옵티마이저 간 격차가 크지 않고 learning rate 단위가 달라 기존 Adam 실험과 직접 비교도 불가능하므로, 먼저 clipping과 warmup, beta2, weight decay를 정리하고 배치를 키운 뒤에 판단하는 순서가 합리적이다.
Muon을 지금 도입할 가장 설득력 있는 이유는 안정성이 아니라 optimizer state 메모리가 AdamW의 절반이라는 점이며, 이는 GPU 2장 환경에서 배치를 키울 여유를 만드는 수단이 될 수 있다.

## Reference

- [PaLM: Scaling Language Modeling with Pathways](https://ar5iv.labs.arxiv.org/html/2204.02311)
- [OPT: Open Pre-trained Transformer Language Models](https://ar5iv.labs.arxiv.org/html/2205.01068)
- [OPT-175B Chronicles: 10 percent update](https://github.com/facebookresearch/metaseq/blob/main/projects/OPT/chronicles/10_percent_update.md)
- [OPT-175B Chronicles: 56 percent update](https://github.com/facebookresearch/metaseq/blob/main/projects/OPT/chronicles/56_percent_update.md)
- [OPT-175B Chronicles: final update](https://github.com/facebookresearch/metaseq/blob/main/projects/OPT/chronicles/final_update.md)
- [GLM-130B: An Open Bilingual Pre-trained Model](https://ar5iv.labs.arxiv.org/html/2210.02414)
- [GLM-130B training log](https://github.com/zai-org/GLM-130B/blob/main/logs/main-log-en.md)
- [BLOOM: A 176B-Parameter Open-Access Multilingual Language Model](https://ar5iv.labs.arxiv.org/html/2211.05100)
- [Small-scale proxies for large-scale Transformer training instabilities](https://ar5iv.labs.arxiv.org/html/2309.14322)
- [A Theory on Adam Instability in Large-Scale Machine Learning](https://ar5iv.labs.arxiv.org/html/2304.09871)
- [Adaptive Preconditioners Trigger Loss Spikes in Adam](https://arxiv.org/pdf/2506.04805)
- [Spike No More: Stabilizing the Pre-training of Large Language Models](https://ar5iv.labs.arxiv.org/html/2312.16903)
- [Weight-norm Criticality and Loss Spikes](https://arxiv.org/html/2607.21005v1)
- [Initialization of LLMs via Reparameterization to Mitigate Loss Spikes](https://arxiv.org/pdf/2410.05052)
- [To FP8 and Back Again](https://arxiv.org/html/2405.18710v1)
- [ZClip: Adaptive Spike Mitigation for LLM Pre-Training](https://arxiv.org/html/2504.02507)
- [ZClip repository](https://github.com/bluorion-com/ZClip)
- [AdaGC: Enhancing LLM Pretraining Stability via Adaptive Gradient Clipping](https://arxiv.org/html/2502.11034v3)
- [An Empirical Model of Large-Batch Training](https://ar5iv.labs.arxiv.org/html/1812.06162)
- [Scaling Laws for Neural Language Models](https://ar5iv.labs.arxiv.org/html/2001.08361)
- [How Does Critical Batch Size Scale in Pre-training?](https://ar5iv.labs.arxiv.org/html/2410.21676)
- [Accurate, Large Minibatch SGD: Training ImageNet in 1 Hour](https://ar5iv.labs.arxiv.org/html/1706.02677)
- [Power Lines: Scaling Laws for Weight Decay and Batch Size in LLM Pre-training](https://arxiv.org/abs/2505.13738)
- [Revisiting critical batch size for large-batch OLMo pretraining](https://allenai.org/blog/critical-batch-size)
- [Small Batch Size Training for Language Models](https://arxiv.org/abs/2507.07101)
- [Bug Fixes in LLM Training: Gradient Accumulation](https://unsloth.ai/blog/gradient)
- [Fixing Gradient Accumulation](https://huggingface.co/blog/gradient_accumulation)
- [transformers issue 26141: training loss sudden spike](https://github.com/huggingface/transformers/issues/26141)
- [transformers issue 40564: gradient accumulation loss scaling](https://github.com/huggingface/transformers/issues/40564)
- [PyTorch AMP examples](https://docs.pytorch.org/docs/stable/notes/amp_examples.html)
- [HuggingFace Transformers: Methods and tools for efficient training on a single GPU](https://huggingface.co/docs/transformers/main/en/perf_train_gpu_one)
- [DeepSpeed Configuration JSON](https://www.deepspeed.ai/docs/config-json/)
- [Megatron-LM arguments.py](https://github.com/NVIDIA/Megatron-LM/blob/main/megatron/training/arguments.py)
- [TorchTitan configs.py](https://github.com/pytorch/torchtitan/blob/main/torchtitan/config/configs.py)
- [nanoGPT train.py](https://github.com/karpathy/nanoGPT/blob/master/train.py)
- [Muon: An optimizer for hidden layers in neural networks](https://kellerjordan.github.io/posts/muon/)
- [Muon repository](https://github.com/KellerJordan/Muon)
- [modded-nanogpt](https://github.com/KellerJordan/modded-nanogpt)
- [Muon is Scalable for LLM Training (Moonlight)](https://ar5iv.labs.arxiv.org/html/2502.16982)
- [Kimi K2: Open Agentic Intelligence](https://ar5iv.labs.arxiv.org/html/2507.20534)
- [Practical Efficiency of Muon for Pretraining](https://ar5iv.labs.arxiv.org/html/2505.02222)
- [Convergence Bound and Critical Batch Size of Muon Optimizer](https://arxiv.org/pdf/2507.01598)
- [Using Muon Optimizer with DeepSpeed](https://pytorch.org/blog/using-muon-optimizer-with-deepspeed/)
- [Gram Newton-Schulz](https://tridao.me/blog/2026/gram-newton-schulz/)
- [How Muon Lost Its Geometry](https://research.ligo.bio/posts/how-muon-lost-its-geometry/)
- [Muon is Not That Special: Random or Inverted Spectra Work Just as Well](https://arxiv.org/abs/2605.11181)
- [Can Muon Fine-tune Adam-Pretrained Models?](https://arxiv.org/abs/2605.10468)
- [Optimizer-Model Consistency](https://arxiv.org/abs/2605.06654)
- [Effects of Parameter Norm Growth During Transformer Training](https://arxiv.org/abs/2010.09697)
- [nanochat](https://github.com/karpathy/nanochat)
- [nanochat hyperparameter scaling and auto-configuration](https://deepwiki.com/karpathy/nanochat/3.2-hyperparameter-scaling-and-auto-configuration)
- [LLM from scratch: interventions baseline model](https://www.gilesthomas.com/2026/02/llm-from-scratch-32a-interventions-baseline-model)
- [LLM from scratch: interventions gradient clipping](https://www.gilesthomas.com/2026/02/llm-from-scratch-32b-interventions-gradient-clipping)
- [LLM from scratch: training our best model locally with gradient accumulation](https://www.gilesthomas.com/2026/04/llm-from-scratch-32k-interventions-training-our-best-model-locally-gradient-accumulation)
- [LLM360 K2-Spike-1 checkpoint](https://huggingface.co/LLM360/K2-Spike-1)
- [LLM360 K2-Spike-2 checkpoint](https://huggingface.co/LLM360/K2-Spike-2)
- [MuonClip deep dive](https://fireworks.ai/blog/muonclip)
