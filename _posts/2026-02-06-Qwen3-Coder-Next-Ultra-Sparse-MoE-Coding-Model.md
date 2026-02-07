---
layout: post
title: Qwen3-Coder-Next - 80B 파라미터 중 3B만 활성화하는 초희소 코딩 에이전트 모델
author: 'Juho'
date: 2026-02-06 02:00:00 +0900
categories: [AI]
tags: [AI, LLM, Coding, MoE, HuggingFace, vLLM]
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
2. [모델 사양](#모델-사양)
3. [아키텍처 상세](#아키텍처-상세)
4. [벤치마크 성능](#벤치마크-성능)
5. [보안 코드 생성 벤치마크](#보안-코드-생성-벤치마크)
6. [학습 방법론](#학습-방법론)
7. [배포 및 사용법](#배포-및-사용법)
8. [Tool Calling 지원](#tool-calling-지원)
9. [로컬 추론 및 양자화](#로컬-추론-및-양자화)
10. [시사점](#시사점)
11. [Reference](#reference)

## 개요

Alibaba의 Qwen 팀이 코딩 에이전트와 로컬 개발 환경을 위해 설계한 오픈 웨이트 언어 모델 Qwen3-Coder-Next를 공개했다.
이 모델은 80B 전체 파라미터 중 토큰당 3B만 활성화하는 초희소(ultra-sparse) Mixture-of-Experts(MoE) 아키텍처를 채택했다.

SWE-Bench Verified에서 70.6%를 달성하여 DeepSeek-V3.2(70.2%)를 능가했으며, SWE-Bench Pro에서는 44.3으로 DeepSeek-V3.2(40.9)와 GLM-4.7(40.6)을 모두 앞섰다.
Apache 2.0 라이선스로 공개되어 상업적 활용이 자유롭다.

## 모델 사양

| 항목 | 상세 |
|------|------|
| 전체 파라미터 | 80B |
| 활성 파라미터 | 3B (토큰당) |
| 비임베딩 파라미터 | 79B |
| 컨텍스트 길이 | 256K (네이티브), Yarn으로 1M 확장 가능 |
| 지원 언어 | 358개 프로그래밍 언어 |
| 데이터 타입 | BF16 |
| 라이선스 | Apache 2.0 |
| Thinking 모드 | 미지원 |

### 모델 변형

| 모델 | 유형 | 용도 |
|------|------|------|
| Qwen3-Coder-Next | Instruct | 표준 추론 |
| Qwen3-Coder-Next-FP8 | Instruct (양자화) | 효율적 추론 |
| Qwen3-Coder-Next-GGUF | Instruct (GGUF) | 로컬 추론 |
| Qwen3-Coder-Next-Base | Base | 파인튜닝용 기반 모델 |
| Qwen3-Coder-480B-A35B-Instruct | Instruct | 대규모 변형 |
| Qwen3-Coder-30B-A3B-Instruct | Instruct | 소규모 변형 |

## 아키텍처 상세

### 하이브리드 레이어 구조

Qwen3-Coder-Next는 Gated DeltaNet과 Gated Attention을 결합한 하이브리드 레이어 구조를 채택한다.

전체 48개 레이어는 12개의 반복 블록으로 구성되며, 각 블록의 구조는 다음과 같다.

3 x (Gated DeltaNet -> MoE) -> 1 x (Gated Attention -> MoE)

4개 서브레이어 중 3개는 선형 어텐션 기반의 Gated DeltaNet을, 1개는 전통적인 Gated Attention을 사용한다.
이 설계를 통해 효율적인 시퀀스 처리와 정밀한 어텐션의 균형을 달성한다.

### Gated Attention 구성

| 항목 | 값 |
|------|-----|
| Query 헤드 수 | 16 |
| KV 헤드 수 | 2 |
| 헤드 차원 | 256 |
| RoPE 차원 | 64 |

### Gated DeltaNet 구성

| 항목 | 값 |
|------|-----|
| Value 헤드 수 | 32 |
| QK 헤드 수 | 16 |
| 헤드 차원 | 128 |

Gated DeltaNet은 선형 어텐션의 일종으로, 기존 Softmax 어텐션의 O(n^2) 복잡도를 피하면서도 효과적인 시퀀스 모델링을 수행한다.

### Mixture-of-Experts 구성

| 항목 | 값 |
|------|-----|
| 전체 Expert 수 | 512 |
| 토큰당 활성 Expert | 10 |
| 공유 Expert | 1 |
| Expert 중간 차원 | 512 |
| 은닉 차원 | 2,048 |

512개 Expert 중 토큰당 10개만 활성화되는 초희소 설계이다.
1개의 공유 Expert는 모든 토큰에 대해 항상 활성화되어 공통 지식을 제공한다.
이 구조 덕분에 밀집 모델 대비 최대 10배 높은 처리량을 달성한다.

## 벤치마크 성능

### SWE-Bench 결과

| 벤치마크 | Qwen3-Coder-Next | DeepSeek-V3.2 | GLM-4.7 |
|----------|------------------|---------------|---------|
| SWE-Bench Verified | 70.6% | 70.2% | 74.2% |
| SWE-Bench Pro | 44.3 | 40.9 | 40.6 |
| SWE-Bench Multilingual | 62.8 | 62.3 | 63.7 |

SWE-Bench Verified에서 DeepSeek-V3.2(671B 파라미터)를 능가한 것이 주목할 만하다.
SWE-Bench Pro에서는 세 모델 중 최고 점수를 기록했다.

활성 파라미터가 3B에 불과하다는 점을 고려하면, 파라미터 효율성 측면에서 압도적인 결과이다.

### 효율성 비교

밀집 모델 대비 최대 10배 높은 처리량을 제공한다.
FP8 네이티브 버전 기준 단일 NVIDIA DGX Spark에서 약 43 tokens/sec의 디코딩 속도를 보인다.

## 보안 코드 생성 벤치마크

### SecCodeBench

| 모델 | 점수 |
|------|------|
| Qwen3-Coder-Next | 61.2% |
| Claude Opus 4.5 | 52.5% |

Qwen3-Coder-Next가 Claude Opus 4.5 대비 8.7% 포인트 높은 보안 코드 생성 점수를 달성했다.

### CWEval

CWEval 벤치마크에서 func-sec@1 점수 56.32%를 기록하며 DeepSeek-V3.2와 GLM-4.7을 모두 능가했다.
이 결과는 코드 생성 시 보안 취약점을 최소화하는 능력이 뛰어남을 보여준다.

## 학습 방법론

### 에이전틱 학습 파이프라인

Qwen3-Coder-Next는 대규모 에이전틱 학습(agentic training) 파이프라인을 통해 개발되었다.

핵심 학습 과정:

1. GitHub Pull Request에서 실제 버그 수정 시나리오를 마이닝
2. 완전히 실행 가능한 환경과 쌍을 이루는 800,000개의 검증 가능한 코딩 태스크 합성
3. 실행 가능한 태스크 합성, 환경 상호작용, 강화학습을 결합한 학습

이 접근법을 통해 장기 추론, 복잡한 도구 사용, 실행 실패 후 복구 등의 에이전틱 능력을 강화했다.

### 핵심 능력

- 장기 추론(long-horizon reasoning): 여러 단계에 걸친 복잡한 코딩 문제 해결
- 복잡한 도구 사용: 다양한 개발 도구와의 상호작용
- 실행 실패 복구: 오류 발생 시 자동 복구 및 재시도
- 레포지토리 규모 이해: 256K 컨텍스트를 활용한 대규모 프로젝트 파악

## 배포 및 사용법

### 기본 사용법 (Transformers)

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

model_name = "Qwen/Qwen3-Coder-Next"

tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype="auto",
    device_map="auto"
)

prompt = "Write a quick sort algorithm."
messages = [
    {"role": "user", "content": prompt}
]
text = tokenizer.apply_chat_template(
    messages,
    tokenize=False,
    add_generation_prompt=True,
)
model_inputs = tokenizer([text], return_tensors="pt").to(model.device)

generated_ids = model.generate(
    **model_inputs,
    max_new_tokens=65536
)
output_ids = generated_ids[0][len(model_inputs.input_ids[0]):].tolist()
content = tokenizer.decode(output_ids, skip_special_tokens=True)
print("content:", content)
```

### SGLang 배포

sglang 0.5.8 이상이 필요하다.

```bash
pip install 'sglang[all]>=v0.5.8'

python -m sglang.launch_server \
    --model Qwen/Qwen3-Coder-Next \
    --port 30000 \
    --tp-size 2 \
    --tool-call-parser qwen3_coder
```

엔드포인트: `http://localhost:30000/v1`

### vLLM 배포

vllm 0.15.0 이상이 필요하다.

```bash
pip install 'vllm>=0.15.0'

vllm serve Qwen/Qwen3-Coder-Next \
    --port 8000 \
    --tensor-parallel-size 2 \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_coder
```

엔드포인트: `http://localhost:8000/v1`

OOM 오류 발생 시 컨텍스트 길이를 32,768 토큰으로 줄일 수 있다.

### 권장 샘플링 파라미터

| 파라미터 | 권장값 |
|----------|--------|
| temperature | 1.0 |
| top_p | 0.95 |
| top_k | 40 |

## Tool Calling 지원

Qwen3-Coder-Next는 OpenAI 호환 API를 통한 Tool Calling을 지원한다.

```python
from openai import OpenAI

client = OpenAI(
    base_url='http://localhost:8000/v1',
    api_key="EMPTY"
)

tools = [
    {
        "type": "function",
        "function": {
            "name": "square_the_number",
            "description": "output the square of the number.",
            "parameters": {
                "type": "object",
                "required": ["input_num"],
                "properties": {
                    "input_num": {
                        "type": "number",
                        "description": "input_num is a number that will be squared"
                    }
                },
            }
        }
    }
]

messages = [{"role": "user", "content": "square the number 1024"}]

completion = client.chat.completions.create(
    messages=messages,
    model="Qwen3-Coder-Next",
    max_tokens=65536,
    tools=tools,
)
```

Claude Code, Qwen Code, Cline 등 다양한 IDE/CLI 플랫폼과 호환된다.
Fill-in-the-Middle(FIM) 기능도 지원하며, `<|fim_prefix|>`, `<|fim_suffix|>`, `<|fim_middle|>` 특수 토큰을 사용한다.

## 로컬 추론 및 양자화

### 지원 로컬 추론 프레임워크

- Ollama
- LMStudio
- MLX-LM
- llama.cpp
- KTransformers

### 하드웨어 요구사항

64GB MacBook, RTX 5090, AMD Radeon 7900 XTX 등 소비자 하드웨어에서 256K 컨텍스트로 실행 가능하다.
FP8 양자화 버전과 GGUF 포맷을 활용하면 더 낮은 사양에서도 구동할 수 있다.

커뮤니티에서 34개 이상의 양자화 버전이 제공되고 있어 다양한 환경에 맞는 선택이 가능하다.

## 시사점

### 초희소 MoE의 효용성 입증

80B 파라미터 중 3B만 활성화하면서 671B 파라미터의 DeepSeek-V3.2를 SWE-Bench에서 능가한 결과는 초희소 MoE 아키텍처의 효용성을 강력히 입증한다.
밀집 모델 대비 10배 높은 처리량은 실제 서비스 배포 비용에 직접적인 영향을 미친다.

### 에이전틱 학습의 가능성

800,000개의 실행 가능한 코딩 태스크를 활용한 에이전틱 학습 방식은 단순한 코드 완성을 넘어 실제 개발 환경에서의 문제 해결 능력을 크게 향상시켰다.
실행 실패 후 복구 능력은 자율 코딩 에이전트로서의 실용성을 높인다.

### 오픈 소스 코딩 모델의 성장

Apache 2.0 라이선스와 다양한 양자화 옵션의 조합은 개발자들이 로컬 환경에서 고성능 코딩 에이전트를 활용할 수 있는 길을 열어준다.
Claude Code, Cline 등 기존 도구와의 호환성도 채택 장벽을 낮추는 요소이다.

### 고려사항

- Thinking 모드를 지원하지 않아 추론 과정 투명성이 제한됨
- 80B 전체 파라미터 로딩에 상당한 메모리가 필요하므로 양자화 선택이 중요
- Gated DeltaNet 기반 하이브리드 어텐션은 아직 생태계 지원이 초기 단계

## Reference

- [Qwen3-Coder-Next - Hugging Face](https://huggingface.co/Qwen/Qwen3-Coder-Next)
- [Qwen3-Coder - GitHub](https://github.com/QwenLM/Qwen3-Coder)
- [Qwen3-Coder-Next Technical Report](https://github.com/QwenLM/Qwen3-Coder/blob/main/qwen3_coder_next_tech_report.pdf)
