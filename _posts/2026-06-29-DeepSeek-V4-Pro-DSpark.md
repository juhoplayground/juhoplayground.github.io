---
layout: post
title: "DeepSeek-V4-Pro-DSpark: speculative decoding을 더한 1.6T MoE 모델"
author: 'Juho'
date: 2026-06-29 00:00:00 +0900
categories: [LLM]
tags: [LLM, vLLM, Benchmark]
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
2. [배경](#배경)
3. [핵심 내용](#핵심-내용)
   - [모델 스펙](#모델-스펙)
   - [아키텍처 혁신](#아키텍처-혁신)
   - [학습 방법](#학습-방법)
   - [추론 모드](#추론-모드)
   - [배포 방법](#배포-방법)
   - [벤치마크](#벤치마크)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

DeepSeek-V4-Pro-DSpark는 DeepSeek-V4-Pro에 speculative decoding 모듈을 부착한 특화 버전입니다.
베이스 모델은 Mixture-of-Experts(MoE) 구조의 언어 모델로, 효율적인 롱컨텍스트 처리를 목표로 설계되었습니다.
총 1.6조(1.6T) 파라미터 중 49B만 활성화되며, 최대 1M(100만) 토큰의 컨텍스트를 지원합니다.
라이선스는 MIT로 공개되었습니다.

## 배경

대규모 MoE 모델은 파라미터 수를 키우면서도 추론 비용을 억제할 수 있다는 장점이 있습니다.
다만 컨텍스트 길이가 1M 토큰 수준까지 늘어나면 어텐션 연산량과 KV 캐시 메모리가 급격히 증가하는 문제가 남습니다.
DeepSeek-V4-Pro는 하이브리드 어텐션과 압축 기법으로 이 비용을 크게 낮췄고, DSpark 변형은 여기에 speculative decoding 모듈을 추가해 추론 가속을 강화했습니다.

## 핵심 내용

### 모델 스펙

DeepSeek-V4-Pro-DSpark의 주요 사양은 다음과 같습니다.

| 항목 | 값 |
|------|------|
| 총 파라미터 | 1.6T |
| 활성 파라미터 | 49B |
| 컨텍스트 길이 | 1M 토큰 |
| 정밀도 | FP4 + FP8 혼합 |
| 라이선스 | MIT |

정밀도는 FP4와 FP8을 혼합해 사용합니다.
MoE 전문가(experts)는 FP4로, 그 외 대부분의 파라미터는 FP8로 저장됩니다.

### 아키텍처 혁신

DeepSeek-V4-Pro-DSpark는 여러 아키텍처 기법을 결합합니다.

하이브리드 어텐션은 Compressed Sparse Attention(CSA)과 Heavily Compressed Attention(HCA)을 결합한 방식입니다.
이 조합으로 롱컨텍스트 효율을 크게 끌어올렸습니다.
1M 토큰 컨텍스트에서 DeepSeek-V3.2 대비 단일 토큰 추론 FLOPs는 27%, KV 캐시는 10%만 사용한다고 설명합니다.

manifold-constrained hyper-connections(mHC)는 잔차 연결(residual connection)을 강화하면서 모델의 표현력을 유지합니다.
Muon optimizer는 더 빠른 수렴과 학습 안정성을 위해 채택되었습니다.

speculative decoding 모듈은 이 DSpark 변형에서 추가된 요소로, 추론 가속을 담당합니다.
세부 사항은 모델 저장소의 inference 폴더에 정리되어 있습니다.

| 기법 | 역할 |
|------|------|
| CSA + HCA 하이브리드 어텐션 | 롱컨텍스트 FLOPs와 KV 캐시 절감 |
| manifold-constrained hyper-connections | 잔차 연결 강화 및 표현력 유지 |
| Muon optimizer | 빠른 수렴과 학습 안정성 |
| speculative decoding 모듈 | 추론 가속 |

### 학습 방법

사전학습은 32T 이상의 다양하고 고품질의 토큰으로 진행되었습니다.

포스트트레이닝은 2단계 패러다임을 따릅니다.
1단계에서는 도메인별 전문가를 독립적으로 육성합니다.
이때 supervised fine-tuning(SFT)과 GRPO 기반 강화학습(RL)을 사용합니다.
2단계에서는 on-policy distillation을 통해 이들을 하나의 모델로 통합(consolidation)합니다.

### 추론 모드

DeepSeek-V4-Pro는 세 가지 추론 강도 모드를 제공합니다.

| 모드 | 특징 | 사용 사례 |
|------|------|-----------|
| Non-Think | 빠르고 직관적인 응답 | 일상적 작업, 저위험 결정 |
| Think High | 의식적인 논리 분석 | 복잡한 문제 해결 |
| Think Max | 최대 추론 강도 | 한계를 넘어서는 난이도 작업 |

Think Max 모드에서는 최소 384K 토큰 이상의 컨텍스트 윈도우 사용을 권장합니다.

로컬 배포 시 권장 샘플링 파라미터는 temperature = 1.0, top_p = 1.0입니다.

### 배포 방법

DeepSeek-V4-Pro-DSpark는 Transformers, vLLM, SGLang, Docker로 배포할 수 있습니다.

Transformers를 사용하는 예시입니다.

```python
from transformers import pipeline
pipe = pipeline("text-generation", model="deepseek-ai/DeepSeek-V4-Pro-DSpark")

from transformers import AutoTokenizer, AutoModelForCausalLM
tokenizer = AutoTokenizer.from_pretrained("deepseek-ai/DeepSeek-V4-Pro-DSpark")
model = AutoModelForCausalLM.from_pretrained("deepseek-ai/DeepSeek-V4-Pro-DSpark")
```

vLLM으로 서빙하는 예시입니다.

```bash
pip install vllm
vllm serve "deepseek-ai/DeepSeek-V4-Pro-DSpark"

curl -X POST "http://localhost:8000/v1/completions" \
  -H "Content-Type: application/json" \
  --data '{
    "model": "deepseek-ai/DeepSeek-V4-Pro-DSpark",
    "prompt": "Once upon a time,",
    "max_tokens": 512,
    "temperature": 0.5
  }'
```

SGLang으로 서버를 실행하는 예시입니다.

```bash
pip install sglang
python3 -m sglang.launch_server \
    --model-path "deepseek-ai/DeepSeek-V4-Pro-DSpark" \
    --host 0.0.0.0 \
    --port 30000
```

Docker로 실행하는 예시입니다.

```bash
docker model run hf.co/deepseek-ai/DeepSeek-V4-Pro-DSpark
```

채팅 메시지 인코딩은 Jinja 템플릿이 아니라 Python 인코딩 스크립트를 사용합니다.
OpenAI 호환 메시지 포맷을 다음과 같이 구성합니다.

```python
from encoding_dsv4 import encode_messages, parse_message_from_completion_text

messages = [
    {"role": "user", "content": "hello"},
    {"role": "assistant", "content": "Hello! I am DeepSeek.",
     "reasoning_content": "thinking..."},
    {"role": "user", "content": "1+1=?"}
]

prompt = encode_messages(messages, thinking_mode="thinking")

import transformers
tokenizer = transformers.AutoTokenizer.from_pretrained(
    "deepseek-ai/DeepSeek-V4-Pro"
)
tokens = tokenizer.encode(prompt)
```

### 벤치마크

모델 카드에 공개된 주요 벤치마크 결과는 다음과 같습니다.

| 영역 | 벤치마크 | 결과 |
|------|----------|------|
| 지식·추론 | MMLU-Pro | 87.5 |
| 지식·추론 | SimpleQA-Verified | 57.9 |
| 지식·추론 | Chinese-SimpleQA | 84.4 |
| 코딩 | LiveCodeBench | 93.5 |
| 코딩 | Codeforces Rating | 3206 |
| 롱컨텍스트 | MRCR 1M (MMR) | 83.5 |
| 롱컨텍스트 | CorpusQA 1M (ACC) | 62.0 |
| 에이전트 | SWE Verified (resolved) | 80.6 |
| 에이전트 | BrowseComp | 83.4 |

MMLU-Pro의 경우 Gemini-3.1-Pro High의 91.0과 비교해 경쟁력 있는 수준이라고 설명합니다.

## 의미와 시사점

DeepSeek-V4-Pro-DSpark는 대규모 MoE 모델의 롱컨텍스트 추론 비용을 구조적으로 낮추는 방향을 보여줍니다.
1M 토큰 컨텍스트에서 DeepSeek-V3.2 대비 FLOPs 27%, KV 캐시 10% 수준이라는 수치는 하이브리드 어텐션과 압축 기법의 효과를 정량적으로 드러냅니다.
여기에 speculative decoding 모듈까지 더해 실제 서빙 단계의 가속을 노린 점이 DSpark 변형의 차별점입니다.
MIT 라이선스와 Transformers, vLLM, SGLang, Docker 등 표준 배포 경로 지원은 도입 장벽을 낮춥니다.

## 결론

DeepSeek-V4-Pro-DSpark는 1.6T 총 파라미터, 49B 활성, 1M 컨텍스트의 MoE 모델에 speculative decoding을 결합한 버전입니다.
CSA와 HCA 하이브리드 어텐션, manifold-constrained hyper-connections, Muon optimizer, FP4+FP8 혼합정밀도 등으로 대규모·롱컨텍스트 추론의 효율을 끌어올렸습니다.
32T 이상 토큰 사전학습과 2단계 포스트트레이닝, 세 가지 추론 모드를 갖춰 다양한 활용 시나리오를 지원합니다.

## Reference

- [DeepSeek-V4-Pro-DSpark (Hugging Face)](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro-DSpark)
