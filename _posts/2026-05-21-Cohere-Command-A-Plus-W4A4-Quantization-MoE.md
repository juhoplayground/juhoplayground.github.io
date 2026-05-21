---
layout: post
title: "Cohere Command A+ 공개: W4A4 양자화로 단일 GPU에서 돌아가는 218B MoE 모델"
author: 'Juho'
date: 2026-05-21 00:00:00 +0900
categories: [LLM]
tags: [LLM, vLLM, Agent]
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
2. [모델 아키텍처](#모델-아키텍처)
   - [Sparse MoE 구성](#sparse-moe-구성)
   - [어텐션과 컨텍스트](#어텐션과-컨텍스트)
3. [W4A4 양자화](#w4a4-양자화)
   - [선택적 양자화 전략](#선택적-양자화-전략)
   - [하드웨어 요구사항](#하드웨어-요구사항)
4. [사용 방법](#사용-방법)
   - [Transformers 기본 사용](#transformers-기본-사용)
   - [vLLM 서빙](#vllm-서빙)
   - [도구 사용과 인용](#도구-사용과-인용)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Cohere와 Cohere Labs가 새로운 플래그십 모델 Command A+를 공개했다.
이번에 주목할 변형은 `command-a-plus-05-2026-w4a4`로, 4-bit 가중치와 4-bit 활성값을 함께 적용한 W4A4 양자화 버전이다.
라이선스는 Apache 2.0이며, 에이전트 작업, 다국어 처리, 추론 중심 작업에 최적화되어 있다.

가장 큰 특징은 총 218B 파라미터 규모의 Mixture-of-Experts(MoE) 모델을 단일 Blackwell B200 한 장에서 구동할 수 있다는 점이다.
즉 대규모 모델의 품질을 유지하면서도 하드웨어 풋프린트를 크게 줄인 것이 핵심이다.
입력으로 텍스트와 이미지를 모두 받을 수 있는 image-text-to-text 모델이며, 48개 언어를 지원한다.

## 모델 아키텍처

Command A+는 디코더 전용(decoder-only) Sparse Mixture-of-Experts 트랜스포머다.
모델 내부 구성은 추론 효율과 품질의 균형을 위해 설계되었다.

### Sparse MoE 구성

전체 파라미터는 218B이지만, 토큰 하나를 처리할 때 실제로 활성화되는 파라미터는 25B에 불과하다.
이 희소성(sparsity) 덕분에 대규모 모델의 표현력을 유지하면서 연산량을 절감한다.

| 항목 | 값 |
|------|------|
| 전체 파라미터 | 218B |
| 활성 파라미터 | 25B |
| 전문가(expert) 수 | 128개 |
| 토큰당 활성 전문가 | 8개 |
| 공유 전문가 | 1개 |
| 라우터 | 토큰 선택(token-choice), 정규화된 sigmoid 활성 |

라우터는 일반적인 softmax 대신 정규화된 sigmoid 활성을 사용한다.
또한 8개의 전문가 외에 항상 활성화되는 공유 전문가(shared expert) 1개를 두어 공통 표현을 담당하게 한다.

### 어텐션과 컨텍스트

어텐션은 회전 위치 임베딩(RoPE)을 적용한 슬라이딩 윈도우 어텐션과 글로벌 어텐션을 3:1 비율로 교차 배치한다.
이 인터리브 구조는 긴 문맥에서의 연산 효율과 전역 정보 통합을 동시에 노린 설계다.

| 항목 | 값 |
|------|------|
| 입력 컨텍스트 | 128K 토큰 |
| 출력 길이 | 64K 토큰 |
| 어텐션 패턴 | 슬라이딩 윈도우와 글로벌 어텐션 3:1 교차 |
| 위치 임베딩 | RoPE |

## W4A4 양자화

W4A4 변형의 핵심은 가중치와 활성값을 모두 4-bit로 표현하면서도 품질 저하를 최소화한 점이다.
양자화 포맷은 2단계 스케일링을 적용한 NVFP4를 사용한다.

### 선택적 양자화 전략

모든 부분을 4-bit로 줄이지 않고, 부분별로 다른 정밀도를 적용하는 선택적 양자화를 채택했다.
품질에 민감한 어텐션 경로는 풀 정밀도로 유지하고, 파라미터의 대부분을 차지하는 MoE 전문가만 4-bit로 양자화한다.

| 구성 요소 | 처리 |
|------|------|
| MoE 전문가 | 4-bit 양자화 |
| 어텐션 Q/K/V/O 프로젝션 | 풀 정밀도 |
| KV 캐시와 어텐션 연산 | 풀 정밀도 |

품질 유지를 위해 양자화 인식 증류(Quantization-Aware Distillation, QAD)를 사용한다.
학생 모델이 가짜 양자화(fake quantization) 연산자를 포함한 상태에서 풀 정밀도 교사 모델의 출력을 모방하도록 학습한다.
그 결과 풀 정밀도 대비 품질 저하가 거의 없으면서도 속도와 지연(latency), 메모리 측면에서 이점을 얻는다.
Cohere는 W4A4를 대부분의 사용 사례에서 권장하는 양자화로 제시한다.

### 하드웨어 요구사항

같은 모델을 정밀도별로 제공하며, 정밀도가 낮아질수록 필요한 GPU 수가 줄어든다.
벤치마크 차이는 모든 양자화 변형에서 무시할 만한 수준이라고 밝혔다.

| 양자화 | Blackwell | Hopper |
|------|------|------|
| BF16 (16-bit) | B200 4장 | H100 8장 |
| FP8 (8-bit) | B200 2장 | H100 4장 |
| W4A4 (4-bit, 권장) | B200 1장 | H100 2장 |

## 사용 방법

Command A+는 Hugging Face Transformers와 vLLM 양쪽에서 사용할 수 있다.
에이전트 활용을 위한 도구 호출(function calling)과 인용(citation) 기능도 기본 지원한다.

### Transformers 기본 사용

기본적인 텍스트 생성은 `apply_chat_template`으로 입력을 구성한 뒤 `generate`를 호출한다.

```python
from transformers import AutoTokenizer, AutoModelForImageTextToText

model_id = "CohereLabs/command-a-plus-05-2026-w4a4"
tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForImageTextToText.from_pretrained(model_id)

messages = [{"role": "user", "content": "What has keys but can't open locks?"}]
input_ids = tokenizer.apply_chat_template(
    messages,
    tokenize=True,
    add_generation_prompt=True,
    return_tensors="pt",
)

gen_tokens = model.generate(
    input_ids,
    max_new_tokens=4096,
    do_sample=True,
    temperature=0.6,
    top_p=0.95
)

gen_text = tokenizer.decode(gen_tokens[0])
print(gen_text)
```

### vLLM 서빙

W4A4 변형은 단일 GPU(`-tp 1`)로 vLLM 서버를 띄울 수 있다.
도구 호출과 추론 파서를 함께 지정하는 것이 권장 설정이다.

```bash
# 의존성 설치
uv pip install vllm>=0.21.0
uv pip install transformers
uv pip install cohere_melody>=0.9.0

# 서버 실행
vllm serve CohereLabs/command-a-plus-05-2026-w4a4 -tp 1 \
  --tool-call-parser cohere_command4 \
  --reasoning-parser cohere_command4 \
  --enable-auto-tool-choice
```

권장 샘플링 파라미터는 temperature 0.9, top_p 0.95, repetition_penalty 1.04다.

### 도구 사용과 인용

함수 정의를 `tools`로 전달하면 모델이 도구 호출을 생성한다.
또한 `enable_citations=True`를 주면 응답에 근거(grounding) 정보를 포함시킬 수 있다.

```python
from transformers import AutoTokenizer

model_id = "CohereLabs/command-a-plus-05-2026-w4a4"
tokenizer = AutoTokenizer.from_pretrained(model_id)

tools = [{
    "type": "function",
    "function": {
        "name": "query_daily_sales_report",
        "description": "Retrieves sales volumes and information for a given day.",
        "parameters": {
            "type": "object",
            "properties": {
                "day": {
                    "description": "Sales data date (YYYY-MM-DD format)",
                    "type": "string",
                }
            },
            "required": ["day"],
        },
    },
}]

conversation = [
    {"role": "user", "content": "Can you provide a sales summary for 29th September 2023?"}
]

input_ids = tokenizer.apply_chat_template(
    conversation=conversation,
    tools=tools,
    tokenize=True,
    add_generation_prompt=True,
    enable_citations=True,
    return_tensors="pt",
)
```

인용을 활성화하면 출력에 어떤 도구 결과를 근거로 삼았는지를 나타내는 스팬이 포함된다.
예를 들어 `<co>10000</co: 0:[0]>` 형태로, 인용된 텍스트와 도구 결과 인덱스가 함께 표시된다.

## 결론

Command A+의 W4A4 변형은 218B 규모 MoE 모델을 단일 GPU에서 운용할 수 있게 만들었다는 점에서 의미가 크다.
선택적 양자화와 양자화 인식 증류를 결합해, 어텐션 경로의 정밀도를 지키면서 전문가 레이어만 4-bit로 압축해 품질 손실을 억제했다.
128K 입력 컨텍스트, 48개 언어 지원, 도구 호출과 인용 기능까지 갖춰 에이전트와 다국어 애플리케이션에 폭넓게 활용할 수 있는 오픈 모델이다.

## Reference

- [CohereLabs/command-a-plus-05-2026-w4a4 (Hugging Face)](https://huggingface.co/CohereLabs/command-a-plus-05-2026-w4a4/)
