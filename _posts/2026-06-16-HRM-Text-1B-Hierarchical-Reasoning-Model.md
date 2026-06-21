---
layout: post
title: "HRM-Text-1B: 계층적 추론 모델 기반 1B 언어 모델"
author: 'Juho'
date: 2026-06-16 00:00:00 +0900
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
2. [아키텍처](#아키텍처)
   - [H-모듈과 L-모듈](#h-모듈과-l-모듈)
   - [기술 사양](#기술-사양)
3. [학습](#학습)
   - [학습 데이터](#학습-데이터)
   - [학습 설정](#학습-설정)
4. [사용 방법](#사용-방법)
   - [의도된 용도](#의도된-용도)
   - [사용 예시 코드](#사용-예시-코드)
5. [한계](#한계)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

HRM-Text-1B는 계층적 추론 모델(Hierarchical Reasoning Model, HRM) 아키텍처 위에 구축된 약 10억(1B) 파라미터 규모의 언어 모델 체크포인트입니다.
이 모델은 Sapient Intelligence가 구조화된 공개 데이터셋을 사용해 처음부터(from scratch) 학습했습니다.
HRM은 두 개의 Transformer 모듈이 서로 다른 시간 스케일로 동작하는 이중 시간 스케일(dual-timescale) 순환 설계를 채택합니다.
이 설계를 통해 파라미터 수를 제한하면서도 사실상 무한한 연산 깊이(unbounded compute depth)를 확보하는 것을 목표로 합니다.

## 아키텍처

HRM-Text-1B는 입력 임베딩 위에서 반복 동작하는 두 개의 Transformer 모듈로 구성됩니다.
하나는 느리게 동작하는 상위 수준 모듈이고, 다른 하나는 빠르게 동작하는 하위 수준 모듈입니다.

### H-모듈과 L-모듈

H-모듈은 상위 수준이며 느린(slow) 시간 스케일로 동작합니다.
L-모듈은 하위 수준이며 빠른(fast) 시간 스케일로 동작합니다.
두 모듈은 입력 임베딩 위에서 반복적으로 연산을 수행합니다.
반복 구성은 H_cycles와 L_cycles의 곱으로 표현되며, 값은 2 × 3입니다.
상태 주입(state injection)은 z_L과 z_H를 더하는 가산적(additive) 결합 방식으로 이루어집니다.
이러한 구조는 파라미터 수를 제한한 상태에서 사실상 무한한 연산 깊이를 제공합니다.

### 기술 사양

아래는 모델의 주요 기술 사양입니다.

| 항목 | 값 |
|------|------|
| 파라미터 | 약 1B |
| Hidden size | 1536 |
| 스택당 레이어 수 | 16 |
| Attention heads | 12 (head_dim: 128) |
| 최대 시퀀스 길이 | 4096 |
| 어휘 크기 | 65,536 |
| 위치 인코딩 | RoPE (theta 10000) |
| 활성화 함수 | SwiGLU |
| 정규화 | Parameterless Pre-RMSNorm |
| dtype | bfloat16 |

## 학습

HRM-Text-1B는 공개적으로 이용 가능한 텍스트 코퍼스를 기반으로 처음부터 사전 학습되었습니다.

### 학습 데이터

이 모델은 공개적으로 이용 가능한 텍스트 코퍼스를 샘플링한 혼합 데이터로 사전 학습되었습니다.
데이터 구성과 전처리 세부 사항은 프로젝트의 data_io 저장소를 통해 오픈소스로 공개되어 있습니다.

### 학습 설정

학습에는 약 400억(40B)개의 고유 토큰이 사용되었습니다.
옵티마이저로는 AdamATan2를 사용했으며, beta는 0.9/0.95, weight decay는 0.1입니다.
학습률은 2.2e-4이며 2000 스텝의 warmup을 적용했습니다.
글로벌 배치 크기는 196,608 토큰입니다.
학습 목표(objective)는 condition prefix 토큰을 사용하는 PrefixLM입니다.

아래는 학습 설정을 정리한 표입니다.

| 항목 | 값 |
|------|------|
| 학습 토큰 | 40B 고유 토큰 |
| 옵티마이저 | AdamATan2 (beta 0.9/0.95) |
| weight decay | 0.1 |
| 학습률 | 2.2e-4 |
| warmup | 2000 스텝 |
| 글로벌 배치 크기 | 196,608 토큰 |
| 학습 목표 | PrefixLM |

## 사용 방법

HRM-Text-1B를 사용할 때는 모델의 특성과 권장 조건(condition) 전략을 이해하는 것이 중요합니다.

### 의도된 용도

이 모델은 명시적으로 정렬(alignment) 이전 단계의 사전 학습 체크포인트이며, 챗봇이나 지시 수행(instruction-following) 어시스턴트가 아닙니다.
어시스턴트 형태의 응용을 위해서는 SFT나 RL 등 추가 정렬 과정이 필요합니다.

권장 조건 전략은 다음과 같습니다.
분류, 추출, QA 같은 NLP 작업에는 direct 조건과 2~8개의 few-shot 예시를 사용합니다.
추론이나 수학 작업에는 synth,cot 복합 조건을 사용해 chain-of-thought 동작을 유도하며, 다만 품질은 균일하지 않습니다.

### 사용 예시 코드

아래는 HRM-Text-1B를 로드하고 생성하는 예시 코드입니다.

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

model_id = "sapientinc/HRM-Text-1B"
tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(
    model_id,
    dtype=torch.bfloat16,
).cuda().eval()

condition = "<|quad_end|><|object_ref_end|>"
prompt = f"<|im_start|>{condition}Explain why the sky is blue.<|im_end|>"

inputs = tokenizer(prompt, return_tensors="pt").to(model.device)
inputs["token_type_ids"] = torch.ones_like(inputs["input_ids"])

with torch.no_grad():
    out = model.generate(**inputs, max_new_tokens=256, do_sample=False)
print(tokenizer.decode(out[0], skip_special_tokens=False))
```

이 모델은 PrefixLM 마스킹을 사용하며, token_type_ids의 값이 1인 위치는 양방향(bidirectional) prefix 위치를 표시합니다.
만약 token_type_ids를 생략하면 attention이 순수 causal 방식으로 동작합니다.
이는 사전 학습 분포와 일치하지 않으며 logits 품질이 눈에 띄게 나빠집니다.
또한 이 모델을 사용하려면 hrm_text를 기본 지원하는 transformers 5.9.0 이상 버전이 필요합니다.

## 한계

HRM-Text-1B에는 다음과 같은 한계가 있습니다.
학습 코퍼스가 영어로만 구성되어 있습니다.
코드 데이터셋으로 학습되지 않았기 때문에 코딩 성능이 약합니다.
출력물에는 부정확한 정보, 편향, 또는 안전하지 않은 내용이 포함될 수 있습니다.
또한 긴 문맥(long-context) 작업에는 최적화되어 있지 않습니다.

## 결론

HRM-Text-1B는 H-모듈과 L-모듈이라는 이중 시간 스케일 순환 구조를 통해 제한된 파라미터로 깊은 연산을 수행하려는 계층적 추론 모델 기반 1B 언어 모델입니다.
약 400억 고유 토큰으로 PrefixLM 목표를 사용해 학습되었으며, 정렬 이전 단계의 사전 학습 체크포인트입니다.
direct나 synth,cot 같은 조건 전략과 token_type_ids 설정을 올바르게 적용하는 것이 사전 학습 분포에 맞는 결과를 얻는 데 핵심입니다.
라이선스는 Apache License 2.0입니다.

## Reference

- [sapientinc/HRM-Text-1B (Hugging Face)](https://huggingface.co/sapientinc/HRM-Text-1B/)
