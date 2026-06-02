---
layout: post
title: "NuExtract3 - 4B 비전-언어 모델로 문서에서 JSON 추출과 마크다운 변환을 동시에"
author: 'Juho'
date: 2026-06-01 00:00:00 +0900
categories: [LLM]
tags: [LLM, Documentation, Schema]
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
2. [모델 스펙](#모델-스펙)
3. [지원하는 템플릿 타입](#지원하는-템플릿-타입)
4. [벤치마크 결과](#벤치마크-결과)
   - [구조화 추출](#구조화-추출)
   - [문서-마크다운 변환](#문서-마크다운-변환)
5. [사용 예시](#사용-예시)
   - [vLLM 배포](#vllm-배포)
   - [텍스트 추출](#텍스트-추출)
   - [이미지 추출](#이미지-추출)
   - [Pydantic 스키마 변환](#pydantic-스키마-변환)
6. [추론 모드](#추론-모드)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

NuMind가 공개한 NuExtract3는 문서 이해 작업을 위한 4B 비전-언어 모델이다.
구조화 추출과 이미지-마크다운 변환을 하나의 모델로 처리하며, 텍스트와 이미지 입력을 모두 다룬다.
다국어 문서를 지원하며, reasoning 모드와 non-reasoning 모드를 모두 제공한다.
자연어로 템플릿 자체를 생성하는 기능도 포함되어 있어, 별도의 스키마 설계 단계 없이 빠른 프로토타이핑이 가능하다.

## 모델 스펙

| 항목 | 값 |
|------|-----|
| 파라미터 수 | 4B |
| 베이스 모델 | Qwen/Qwen3.5-4B |
| 텐서 타입 | BF16 |
| 라이선스 | Apache 2.0 |
| 포맷 | Safetensors |

베이스 모델이 Qwen3.5-4B인 점은 다국어 지원과 reasoning 능력 측면에서 유리한 출발점이다.

## 지원하는 템플릿 타입

NuExtract3는 JSON 템플릿 안에 타입을 명시하면 그에 맞는 형식으로 결과를 반환한다.

| 타입 | 설명 |
|------|------|
| verbatim-string | 원문 그대로 추출 |
| string | 일반 텍스트 또는 paraphrased 결과 |
| integer | 정수 |
| number | 정수 또는 소수 |
| date-time | ISO-8601 형식 |
| date | 날짜만 |
| time | 시간만 |
| currency | 통화 값 |
| email | 이메일 주소 |
| country | 국가명 |

배열은 `["string"]`, enum은 `["yes", "no", "maybe"]`, multi-enum은 `[["A", "B", "C"]]` 형식으로 표현한다.

## 벤치마크 결과

### 구조화 추출

NuExtract3.4-4B-RL은 약 600개 문서를 대상으로 한 구조화 추출 평가에서 평균 0.651 ± 0.019를 기록했고, 실패 출력은 27건이었다.
같은 평가에서 Qwen3.5-9B는 0.479, Qwen3.5-4B는 0.417에 그쳐 명확한 우위를 보였다.
평가 메트릭은 verbatim-string과 string은 Levenshtein 거리를 사용하고, 다른 타입은 exact-match를 적용한다.
인보이스, 포스터, 평면도 등 다양한 문서가 평가에 포함되었다.

### 문서-마크다운 변환

문서를 마크다운으로 변환하는 작업은 Gemini 3 Flash 비교를 통해 100개 복잡 문서에서 평가했고, 결과는 사람의 선호도 투표와 정렬되었다.

## 사용 예시

### vLLM 배포

vLLM에서는 multimodal 입력과 speculative decoding을 함께 설정한다.

```bash
vllm serve numind/NuExtract3 \
  --trust-remote-code \
  --limit-mm-per-prompt '{"image": 99, "video": 0}' \
  --chat-template-content-format openai \
  --generation-config vllm \
  --max-model-len 131072 \
  --speculative-config '{"method": "qwen3_next_mtp", "num_speculative_tokens": 2}'
```

Multi-Token Prediction(MTP)를 활성화하면 speculative decoding으로 추론 속도가 향상된다.

### 텍스트 추출

영수증 같은 짧은 텍스트에서 구조화된 정보를 뽑는 예시다.

```python
import json
from openai import OpenAI

client = OpenAI(api_key="EMPTY", base_url="http://localhost:8000/v1")

template = {
    "store": "verbatim-string",
    "date": "date-time",
    "total": "number",
    "currency": ["USD", "EUR", "GBP", "JPY", "Other"],
    "items": [{"name": "verbatim-string", "price": "number"}]
}

response = client.chat.completions.create(
    model="numind/NuExtract3",
    temperature=0.2,
    messages=[{
        "role": "user",
        "content": [{
            "type": "text",
            "text": "Yesterday I bought apples and coffee at Trader Joe's for a total of $12.40."
        }]
    }],
    extra_body={
        "chat_template_kwargs": {
            "template": json.dumps(template),
            "enable_thinking": False
        }
    }
)
print(response.choices[0].message.content)
```

### 이미지 추출

이미지에서 추출할 때는 base64 인코딩 후 데이터 URL로 전달한다.

```python
import json
import base64
from openai import OpenAI

client = OpenAI(api_key="EMPTY", base_url="http://localhost:8000/v1")

def encode_image(image_path):
    with open(image_path, "rb") as image_file:
        return base64.b64encode(image_file.read()).decode("utf-8")

image_base64 = encode_image("receipt.png")
data_url = f"data:image/png;base64,{image_base64}"

template = {
    "store": "verbatim-string",
    "date": "date-time",
    "total": "number",
    "payment_method": "verbatim-string"
}

response = client.chat.completions.create(
    model="numind/NuExtract3",
    temperature=0.2,
    messages=[{
        "role": "user",
        "content": [{
            "type": "image_url",
            "image_url": {"url": data_url}
        }]
    }],
    extra_body={
        "chat_template_kwargs": {
            "template": json.dumps(template, indent=4),
            "enable_thinking": False
        }
    }
)
print(response.choices[0].message.content)
```

### Pydantic 스키마 변환

기존에 정의된 Pydantic 모델을 NuExtract 템플릿으로 변환하는 유틸리티가 제공된다.

```python
from typing import Literal
from pydantic import Field, BaseModel
from numind.nuextract_utils import convert_json_schema_to_nuextract_template

class HotelBooking(BaseModel):
    city: str
    check_in_date: str = Field(description="date")
    check_out_date: str = Field(description="date")
    number_of_guests: int
    room_type: Literal["single", "double", "suite"]

template, dropped_branches = convert_json_schema_to_nuextract_template(
    HotelBooking.model_json_schema()
)
```

기존 백엔드 코드의 Pydantic 모델을 그대로 추출 템플릿으로 활용할 수 있다.

## 추론 모드

| 모드 | 추천 온도 | 용도 |
|------|-----------|------|
| Non-thinking | 0.2 | 빠르고 결정론적인 추출 |
| Thinking | 0.6 | 복잡한 레이아웃, 애매한 필드 |

Thinking 모드는 reasoning을 활성화해 모호한 문서나 복잡한 레이아웃에서 정확도를 끌어올린다.
간단한 영수증 추출은 non-thinking 모드로 충분하다.

## 결론

NuExtract3는 4B라는 합리적 사이즈로 비전-언어 문서 이해를 한 모델에서 처리하며, Apache 2.0 라이선스로 자가 호스팅이 자유롭다.
JSON 스키마 기반 추출과 마크다운 변환을 동시에 지원하기 때문에, 문서 처리 파이프라인을 단일 엔드포인트로 통합할 수 있다.
Qwen3.5-4B 베이스 위에 reasoning 모드와 MTP를 결합한 구성은, 비싼 클로즈드 API를 쓰지 않고도 충분히 실용적인 문서 이해 시스템을 구축할 수 있게 한다.

## Reference

- [NuExtract3 모델 카드 - Hugging Face](https://huggingface.co/numind/NuExtract3)
- [NuExtract 플랫폼](https://nuextract.ai/)
- [NuMind GitHub](https://github.com/numindai/nuextract)
- [Demo Space](https://huggingface.co/spaces/numind/NuExtract-3-4B)
