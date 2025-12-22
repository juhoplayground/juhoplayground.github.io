---
layout: post
title: GPT-OSS-120B MoE 모델에서 QLoRA 튜닝이 실패하는 이유와 NeMo의 해결책
author: 'Juho'
date: 2025-12-22 00:00:00 +0900
categories: [LLM]
tags: [LLM, MoE, QLoRA, NVIDIA-NeMo, Fine-tuning, Quantization]
pin: True
toc: True
---

## 목차
1. [GPT-OSS-120B는 Dense가 아닌 MoE 모델](#gpt-oss-120b는-dense가-아닌-moe-모델)
2. [Dense 모델과 MoE 모델의 근본적 차이](#dense-모델과-moe-모델의-근본적-차이)
3. [QLoRA × MoE 조합이 깨지는 지점](#qlora--moe-조합이-깨지는-지점)
4. [HuggingFace + PEFT 환경의 구조적 한계](#huggingface--peft-환경의-구조적-한계)
5. [그런데 왜 NVIDIA NeMo에서는 되는가?](#그런데-왜-nvidia-nemo에서는-되는가)
6. [결론](#결론)

## 들어가며

대규模 언어 모델을 효율적으로 파인튜닝하기 위해 QLoRA는 매우 유용한 기법입니다.
하지만 GPT-OSS-120B와 같은 MoE(Mixture of Experts) 모델에서는 QLoRA가 제대로 작동하지 않는 문제가 발생합니다.
이 글에서는 왜 MoE 모델에서 QLoRA 튜닝이 실패하는지 그리고 NVIDIA NeMo는 어떻게 이 문제를 해결하는지 기술적으로 살펴보겠습니다.

## GPT-OSS-120B는 Dense가 아닌 MoE 모델

GPT-OSS-120B는 일반적인 Dense Transformer가 아니라 MoE구조를 사용합니다.  
이것이 모든 문제의 출발점입니다.  

### MoE의 기본 동작 방식

MoE 모델은 다음과 같이 동작합니다:  

- 각 Transformer layer마다 128개의 FFN expert가 존재  
- 각 토큰마다 router가 해당 토큰을 어느 expert로 보낼지 결정  
- 일반적으로 Top-2 expert만 활성화  
- 나머지 126개 expert는 해당 토큰에 대해 완전히 비활성  
  
```
Forward pass  → 선택된 일부 expert만 계산
Backward pass → 선택된 expert + router만 gradient 수신
```

이 sparse activation + dynamic routing이 Dense 모델과의 결정적 차이입니다.  

## Dense 모델과 MoE 모델의 근본적 차이

| 구분 | Dense Transformer | MoE Transformer |
|------|------------------|-----------------|
| FFN 구조 | 단일 FFN | 다수의 expert FFN |
| 활성화 | 모든 파라미터 항상 활성 | 일부 expert만 활성 |
| Gradient | 전체에 균등 전달 | 선택된 expert에만 전달 |
| 성능 결정 요소 | weight 자체 | routing + expert 조합 |

Dense 모델에서는 "weight를 조금만 바꿔도" 전체 표현이 변합니다.  
MoE 모델에서는 "어떤 expert를 쓰느냐"가 성능의 상당 부분을 차지합니다.  

## QLoRA × MoE 조합이 깨지는 지점

### 1. Router + 4bit Quantization의 구조적 충돌  

QLoRA의 핵심 전제는 다음과 같습니다:

- Base weight는 4bit로 고정  
- LoRA adapter만 학습  
- 양자화 상태에서 바로 학습 수행  

문제는 MoE의 router가 민감하다는 점입니다.  

#### 실제로 발생하는 현상

1. Router는 단순한 linear projection  
2. 4bit quantization noise → router score 미세 변동  
3. 그 결과:  
   - Forward에서 선택된 expert와  
   - Backward에서 gradient가 흐르는 expert가 달라짐  

```
forward / backward routing 불일치
→ 학습 신호가 일관되지 않음
→ 수렴 실패 또는 생성 품질 급락
```

이 현상이 바로 "forward propagation과 back propagation의 라우팅이 안 맞는 느낌"의 정확한 기술적 원인입니다.  

### 2. LoRA는 Expert Routing을 바꿀 수 없다  

MoE 모델에서 중요한 것은 단순한 weight 조정이 아니라:  

> "이 토큰을 어떤 expert로 보내야 하는가"

하지만 QLoRA 환경에서는:  

- Router weight는 사실상 고정  
- LoRA는 선택된 expert 내부 weight만 미세 조정  

즉 LoRA로는 다음이 불가능합니다:  

- "이 도메인은 expert A 대신 B를 써야 한다"  
- Routing 정책 자체의 재구성   

#### 결과적으로 발생하는 문제  

- 특정 expert만 과도하게 사용  
- 잘못된 expert로 routing 고정  
- 새로운 데이터 분포에 적응 실패  
- Hallucination 증가 및 성능 저하  

### 3. Sparse Activation으로 인한 학습 신호 단절  

MoE의 구조적 특성상:  

- 한 토큰당 128개 중 2개 expert만 활성  
- 나머지 expert는 해당 토큰에서 완전히 학습되지 않음  

Dense 모델에서는:  
- 모든 파라미터가 매 step 학습됨  
- LoRA가 일관되게 수렴  

MoE 모델에서는:  
- 각 expert가 전체 데이터의 일부만 관측  
- LoRA adapter의 학습이 fragmented  
- Expert 간 품질 편차 확대  

### 4. Router–Expert Co-adaptation 붕괴  

원래 MoE 모델은:  

- Router와 expert가 함께 pre-training  
- 서로의 특성에 맞게 공진화(co-adaptation)  

하지만 LoRA 적용 시:  

- Router는 기존 expert 특성 기준으로 routing  
- Expert는 LoRA로 인해 성질이 변형  


```
Router가 "더 이상 맞지 않는 expert"로 토큰을 보냄  
→ 모델 내부 일관성 붕괴
→ 출력 품질 불안정
```

## HuggingFace + PEFT 환경의 구조적 한계  

HuggingFace의 QLoRA / PEFT는 기본적으로 Dense Transformer 기준으로 설계되었습니다.  

### 주요 한계점

1. Expert layer를 자동으로 LoRA 타겟팅하지 않음  
   - `mlp.experts.*`를 수동 지정 필요  

2. Router 학습, load-balancing loss, expert capacity 관리 부재  

3. Fused MoE kernel과 LoRA 충돌 가능성  

4. gpt-oss 전용 MXFP4 설정 미적용 시 성능 붕괴  

즉, HuggingFace에서는 "돌아가긴 하지만" MoE 학습에 필요한 전제가 빠져 있습니다.  

## 그런데 왜 NVIDIA NeMo에서는 되는가?  

이 부분이 핵심적인 대비 지점입니다.  

### NeMo는 MoE 학습을 전제로 설계됨  

NeMo (Megatron-LM 기반)는:  

- Expert Parallelism을 기본 지원  
- Router / expert 구조를 pretraining과 동일하게 유지  
- 대규모 MoE 학습이 기본 가정  

### NeMo의 QLoRA 접근 방식 차이  

| 항목 | HuggingFace QLoRA | NVIDIA NeMo |
|------|-------------------|-------------|
| 학습 precision | 4bit 상태에서 학습 | BF16으로 upcast 후 학습 |
| 학습 대상 | LoRA만 | LoRA 또는 full fine-tune |
| 양자화 시점 | 학습 중 | 학습 후 QAT로 복원 |

NeMo에서는:  

1. 안정적인 BF16 환경에서 MoE 학습  
2. Router score 변동 최소화  
3. Forward/Backward routing 일관성 유지  
4. 학습 완료 후에만 4bit로 축소  

결과적으로:  

```
Expert collapse 현상 감소
→ 수렴 안정성 확보
→ 생성 품질 유지
```

## 결론

GPT-OSS-120B와 같은 MoE 모델에서 QLoRA가 실패하는 이유는:

1. 4bit quantization이 router의 미세한 변동을 유발하여 forward/backward routing 불일치 발생  
2. LoRA는 routing 정책을 변경할 수 없어 새로운 도메인에 적응 실패  
3. Sparse activation으로 인해 학습 신호가 fragmented되어 expert 간 품질 편차 확대  
4. Router-Expert co-adaptation이 붕괴되어 모델 내부 일관성 상실  

NVIDIA NeMo가 이를 해결할 수 있는 이유는:  

- BF16 환경에서 안정적으로 학습하고 학습 후에 양자화  
- Expert Parallelism과 MoE 구조를 완벽히 지원  
- Forward/Backward routing 일관성을 유지   

따라서 MoE 모델을 파인튜닝할 때는 HuggingFace QLoRA 대신 NVIDIA NeMo를 사용하는 것이 권장됩니다.  
