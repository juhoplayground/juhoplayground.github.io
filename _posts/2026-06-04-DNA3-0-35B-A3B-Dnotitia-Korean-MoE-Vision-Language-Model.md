---
layout: post
title: "DNA3.0-35B-A3B: Dnotitia의 한국어 특화 MoE 비전-언어 모델"
author: 'Juho'
date: 2026-06-04 00:00:00 +0900
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
2. [아키텍처와 파라미터](#아키텍처와-파라미터)
3. [주요 특징](#주요-특징)
   - [Dnotitia 포스트트레이닝](#dnotitia-포스트트레이닝)
   - [기반 모델에서 상속한 특징](#기반-모델에서-상속한-특징)
4. [사용 방법](#사용-방법)
5. [제한사항과 책임 있는 사용](#제한사항과-책임-있는-사용)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

DNA3.0-35B-A3B는 Dnotitia Inc.가 공개한 Mixture-of-Experts(MoE) 기반 인과 언어 모델이다.
Qwen3.6-35B-A3B를 기반으로 추가 포스트트레이닝을 거쳤으며, 비전 인코더를 포함한 멀티모달 모델이다.

전체 파라미터는 35B이지만 추론 시 활성화되는 파라미터는 3B에 불과해, 높은 처리량과 낮은 연산 비용을 동시에 노린다.
라이선스는 Apache-2.0이며, 한국어 응답 품질 개선에 초점을 맞춘 점이 특징이다.

## 아키텍처와 파라미터

이 모델은 희소 MoE 구조와 선형 주의(Gated DeltaNet)를 결합한 하이브리드 아키텍처를 사용한다.

| 항목 | 값 |
|------|-----|
| 전체 파라미터 | 35B |
| 활성 파라미터 | 3B |
| 숨겨진 차원 | 2048 |
| 레이어 수 | 40 |
| 전문가 수 | 256개 (8개 라우팅 + 1개 공유) |
| 게이트 어텐션 헤드 | 16(Q) / 2(KV), 헤드 차원 256 |
| 게이트 DeltaNet 헤드 | 32(V) / 16(QK), 헤드 차원 128 |
| 토큰 임베딩 | 248,320 |
| 기본 컨텍스트 길이 | 262,144 토큰 |
| 확장 컨텍스트 | YaRN 스케일링으로 약 1,010,000 토큰 |

기본 컨텍스트는 262,144 토큰이며, YaRN 스케일링을 적용하면 약 100만 토큰까지 확장된다.
저장 형식은 Safetensors(BF16)이고, 모델 크기는 약 36B 파라미터다.

## 주요 특징

### Dnotitia 포스트트레이닝

Dnotitia는 기반 모델 위에 자체 포스트트레이닝을 추가했다.

- 무검열 학습: 불필요한 거부 없이 더 넓은 범위의 프롬프트에 응답
- 페르소나 학습: Dnotitia의 기업 지식, 제품, 서비스, 내부 용어에 대한 감독 학습
- 장문 추론 보존: 다중 턴 세션에서 chain-of-thought 추적을 유지해 반복적 개발 지원

Qwen3.6-35B-A3B와 비교했을 때 네 가지 지표에서 개선을 보고한다.

| 지표 | 개선 내용 |
|------|-----------|
| 페르소나 식별 | Dnotitia 어시스턴트로서의 자기 인식과 기업 정보 정확도 |
| 무검열성 | 중국 원산 기반 모델의 검열 정책 회피 |
| 언어 혼동 감소 | 한국어 응답에서 중국 문자 침입 감소 |
| 반복 감소 | 장문 생성 중 무한 반복 루프 회피 |

### 기반 모델에서 상속한 특징

Qwen3.5/3.6에서 물려받은 특징도 모델의 핵심을 이룬다.

- 통합 비전-언어 기반: 멀티모달 토큰에 대한 초기 융합 학습으로 텍스트, 이미지, 비디오 크로스모달 추론
- 효율적 하이브리드 MoE: Gated DeltaNet과 희소 MoE 결합으로 높은 처리량과 낮은 활성 파라미터
- 확장 가능한 RL 일반화: 점진적으로 복잡해지는 작업 분배를 학습
- 전역 언어 지원: 201개 언어 및 방언 기본 지원
- 사고 모드 기본값: 최종 답변 전에 think 추론 블록 생성, 비활성화 가능

## 사용 방법

Transformers 라이브러리로 이미지-텍스트 입력을 처리하는 예시는 다음과 같다.

```python
from transformers import pipeline

pipe = pipeline("image-text-to-text", model="dnotitia/DNA3.0-35B-A3B")
messages = [
    {
        "role": "user",
        "content": [
            {"type": "image", "url": "https://example.com/image.jpg"},
            {"type": "text", "text": "What animal is on the candy?"}
        ]
    },
]
pipe(text=messages)
```

프로세서와 모델을 직접 로드해 생성하는 방식도 가능하다.

```python
from transformers import AutoProcessor, AutoModelForImageTextToText

processor = AutoProcessor.from_pretrained("dnotitia/DNA3.0-35B-A3B")
model = AutoModelForImageTextToText.from_pretrained("dnotitia/DNA3.0-35B-A3B")

inputs = processor.apply_chat_template(
    messages,
    add_generation_prompt=True,
    tokenize=True,
    return_dict=True,
    return_tensors="pt",
).to(model.device)

outputs = model.generate(**inputs, max_new_tokens=40)
print(processor.decode(outputs[0][inputs["input_ids"].shape[-1]:]))
```

프로덕션 서빙에는 전용 추론 엔진을 권장한다.
vLLM으로 멀티모달, 도구 호출, 텍스트 전용 모드를 각각 설정할 수 있다.

```bash
# 표준 멀티모달 서빙
vllm serve dnotitia/DNA3.0-35B-A3B \
  --reasoning-parser qwen3

# 도구 호출 활성화
vllm serve dnotitia/DNA3.0-35B-A3B \
  --reasoning-parser qwen3 \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder

# 텍스트 전용 모드 (비전 인코더 생략)
vllm serve dnotitia/DNA3.0-35B-A3B \
  --reasoning-parser qwen3 \
  --language-model-only
```

think 추론 블록을 끄려면 chat_template_kwargs를 사용한다.
`/think`, `/nothink` 같은 인라인 명령은 지원하지 않는다.

```bash
curl https://demo-api.dnotitia.ai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "DNA3.0-35B-A3B",
    "messages": [{"role": "user", "content": "Your prompt"}],
    "chat_template_kwargs": {"enable_thinking": false}
  }'
```

이 밖에 SGLang, KTransformers, Docker Model Runner를 지원하며, 양자화 모델은 Ollama, llama.cpp, LM Studio, Jan에서 사용할 수 있다.
권장 컨텍스트 길이는 128K 토큰 이상 유지다.

## 제한사항과 책임 있는 사용

모델 카드는 무검열 학습의 부작용과 상속된 편향을 명시한다.

- 감소된 거부 행동: 다른 모델보다 더 많은 프롬프트에 응답하므로 적절한 콘텐츠 조정 필요
- 페르소나 편향: Dnotitia 기업 지식 학습에서 비롯된 편향 가능성
- 상속된 편향: 기반 모델과 학습 데이터의 문화적, 언어적, 사실적 편향
- 환각: 틀린 정보를 높은 신뢰도로 생성 가능
- 고위험 자동화 부적합: 법률, 의료, 금융 결정에는 인간 감시 필요

## 결론

DNA3.0-35B-A3B는 Qwen3.6-35B-A3B를 기반으로 한국어 응답 품질, 페르소나 일관성, 반복 억제를 강화한 MoE 비전-언어 모델이다.
35B 전체 파라미터 중 3B만 활성화하는 효율적 구조와 100만 토큰까지 확장 가능한 컨텍스트가 핵심 강점이다.
무검열 학습이 적용된 만큼, 서비스에 투입할 때는 별도의 콘텐츠 조정과 인간 감시를 함께 설계하는 것이 중요하다.

## Reference

- [DNA3.0-35B-A3B - Hugging Face](https://huggingface.co/dnotitia/DNA3.0-35B-A3B)
