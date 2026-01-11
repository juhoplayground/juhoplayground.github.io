---
layout: post
title: vLLM Custom Logits Processors로 특정 언어 토큰 차단하기
author: 'Juho'
date: 2026-01-11 09:00:00 +0900
categories: [LLM, vLLM]
tags: [Python, LLM, vLLM]
pin: True
toc: True
---

## 개요

대규모 언어 모델(LLM)을 서비스하다 보면 특정 언어나 문자를 생성하지 않도록 제어해야 하는 경우가 있습니다. 예를 들어, 한글 전용 서비스에서 중국어나 일본어 한자가 섞여 나오는 것을 방지하거나, 특정 토큰의 생성을 제한해야 하는 상황이 발생할 수 있습니다.

이번 포스트에서는 vLLM의 Custom Logits Processors 기능을 활용하여 특정 언어 토큰의 생성을 차단하는 방법을 알아보겠습니다.

## vLLM Custom Logits Processors란?

Custom Logits Processors는 vLLM에서 모델이 생성하는 로짓(logits) 출력을 사용자 정의 방식으로 처리할 수 있는 강력한 기능입니다.

### 로짓이란?

로짓(Logits)은 모델이 다음 토큰을 선택하기 전에 출력하는 원시 점수(raw scores)입니다. 이 값들은 소프트맥스(softmax) 함수를 거쳐 확률 분포로 변환되며, 최종적으로 가장 높은 확률을 가진 토큰이 선택됩니다.

### Custom Logits Processor의 역할

Custom Logits Processor는 모델이 다음 토큰을 선택하기 직전에 로짓 값을 수정할 수 있게 해줍니다. 이를 통해 다음과 같은 작업이 가능합니다:

- 특정 토큰의 생성 확률을 0으로 만들기 (차단)
- 특정 토큰의 생성 확률 조정하기
- 조건에 따른 동적 필터링 수행하기

## 작동 원리

로짓 프로세서의 핵심 원리는 간단합니다. 차단하려는 토큰의 로짓을 음의 무한대(`-inf`)로 설정하면, 소프트맥스 함수를 거친 후 해당 토큰의 확률이 0이 되어 절대 선택되지 않습니다.

```python
# 차단하려는 토큰의 로짓을 -inf로 설정
logits[blocked_token_ids] = float('-inf')
```

## 외국어 토큰 차단 구현

특정 언어(예: 중국어, 일본어)의 토큰을 차단하려면 다음 단계를 거쳐야 합니다:

### 1. 차단할 토큰 식별

먼저 토크나이저의 전체 어휘(vocabulary)에서 차단하려는 문자가 포함된 토큰을 식별해야 합니다. 유니코드 범위를 활용하면 특정 언어의 문자를 효율적으로 감지할 수 있습니다.

```python
# CJK(한중일) 문자 유니코드 범위 정의
CJK_RANGES = [
    (0x4E00, 0x9FFF),   # CJK Unified Ideographs
    (0x3400, 0x4DBF),   # CJK Extension A
    (0x20000, 0x2A6DF), # CJK Extension B
    # ... 기타 범위
]

def contains_cjk(text):
    for char in text:
        code = ord(char)
        for start, end in CJK_RANGES:
            if start <= code <= end:
                return True
    return False
```

### 2. Custom Logits Processor 클래스 구현

vLLM의 Custom Logits Processor는 다음 4가지 필수 메서드를 구현해야 합니다:

#### apply() 메서드

로짓 텐서를 받아 수정하고 반환하는 핵심 메서드입니다.

```python
def apply(self, logits: torch.Tensor, request_id: str) -> torch.Tensor:
    # 차단할 토큰의 로짓을 -inf로 설정
    logits[:, self.blocked_token_ids] = float('-inf')
    return logits
```

#### update_state() 메서드

요청 변경 시 내부 상태를 동기화합니다.

```python
def update_state(self, request_id: str, state: Any) -> None:
    # 상태 업데이트 로직
    pass
```

#### validate_params() 메서드

사용자 파라미터를 검증합니다.

```python
@classmethod
def validate_params(cls, params: Dict[str, Any]) -> None:
    # 파라미터 검증 로직
    pass
```

#### is_argmax_invariant() 메서드

최적화 가능 여부를 표시합니다.

```python
def is_argmax_invariant(self) -> bool:
    return True  # 최적화 가능
```

### 3. 전체 구현 예제

다음은 중국어와 한자를 차단하는 완전한 Custom Logits Processor 구현 예제입니다:

```python
from typing import Dict, Any
import torch
from vllm import LogitsProcessor

class NoChineseLogitsProcessor(LogitsProcessor):
    def __init__(self, tokenizer):
        self.tokenizer = tokenizer
        self.blocked_token_ids = self._identify_blocked_tokens()

    def _identify_blocked_tokens(self):
        """CJK 문자가 포함된 토큰 ID 목록 생성"""
        blocked_ids = []
        vocab = self.tokenizer.get_vocab()

        for token, token_id in vocab.items():
            if self._contains_cjk(token):
                blocked_ids.append(token_id)

        return torch.tensor(blocked_ids, dtype=torch.long)

    def _contains_cjk(self, text):
        """텍스트에 CJK 문자가 포함되어 있는지 확인"""
        CJK_RANGES = [
            (0x4E00, 0x9FFF),   # CJK Unified Ideographs
            (0x3400, 0x4DBF),   # CJK Extension A
            (0x20000, 0x2A6DF), # CJK Extension B
        ]

        for char in text:
            code = ord(char)
            for start, end in CJK_RANGES:
                if start <= code <= end:
                    return True
        return False

    def apply(self, logits: torch.Tensor, request_id: str) -> torch.Tensor:
        """차단할 토큰의 로짓을 -inf로 설정"""
        logits[:, self.blocked_token_ids] = float('-inf')
        return logits

    def update_state(self, request_id: str, state: Any) -> None:
        pass

    @classmethod
    def validate_params(cls, params: Dict[str, Any]) -> None:
        pass

    def is_argmax_invariant(self) -> bool:
        return True
```

## vLLM 서버에 적용하기

작성한 Custom Logits Processor를 vLLM 서버에 적용하는 방법은 3가지가 있습니다:

### 1. FQCN 방식 (Fully-Qualified Class Name)

초기화 시 클래스의 전체 경로를 전달하는 방식입니다.

```bash
export PYTHONPATH=$PYTHONPATH:$(pwd)
vllm serve meta-llama/Llama-3.2-3B-Instruct \
    --logits-processors no_chinese_plugin:NoChineseLogitsProcessor
```

### 2. Entry Points 방식

Python 환경에 설치된 Custom Processors를 자동으로 감지하는 방식입니다. `setup.py` 또는 `pyproject.toml`에 entry point를 등록하면 됩니다.

```python
# setup.py
setup(
    name="vllm-no-chinese",
    entry_points={
        "vllm.logits_processors": [
            "no_chinese = my_package.processors:NoChineseLogitsProcessor"
        ]
    }
)
```

### 3. 오프라인 전용 방식

vLLM 생성자에 Python 클래스 객체를 직접 전달하는 방식입니다.

```python
from vllm import LLM
from my_processors import NoChineseLogitsProcessor

llm = LLM(
    model="meta-llama/Llama-3.2-3B-Instruct",
    logits_processors=[NoChineseLogitsProcessor]
)
```

## 성능 고려사항

### 초기 지연 (TTFT)

Custom Logits Processor를 적용하면 첫 토큰 생성 시간(Time To First Token, TTFT)이 증가할 수 있습니다. 실제 측정 결과에 따르면 약 1534ms의 지연이 발생할 수 있습니다.

### 웜업 프로세스

프로덕션 환경에서는 서버 시작 후 더미 요청을 보내는 웜업(warm-up) 프로세스를 구현하는 것이 좋습니다.

```python
# 웜업 요청 예제
def warmup_server():
    llm.generate("Hello", max_tokens=10)
    print("Server warmed up!")
```

### 차단 토큰 수

모델과 토크나이저에 따라 차단되는 토큰 수가 다릅니다. Llama 3.2 기준으로 약 4,426개의 토큰이 CJK 문자를 포함하고 있습니다. 이는 전체 어휘의 약 3-4% 정도입니다.

## 다양한 구현 방식 비교

### PyTorch 기반 구현

가장 기본적인 방식으로, PyTorch의 텐서 연산을 직접 활용합니다.

**장점:**
- 구현이 간단하고 직관적
- 다른 라이브러리 의존성 없음

**단점:**
- 추론 속도가 상대적으로 느림
- 배치 처리 최적화가 어려움

### Transformers LogitProcessor 활용

Hugging Face Transformers 라이브러리의 `LogitsProcessor` 인터페이스를 활용하는 방식입니다.

**장점:**
- Hugging Face 생태계와의 호환성
- 다양한 기본 제공 프로세서와 조합 가능

**단점:**
- vLLM만큼 빠르지 않음
- 대규모 배치 처리 시 성능 제약

### vLLM Custom Logits Processor (권장)

vLLM 프레임워크를 활용한 고성능 구현 방식입니다.

**장점:**
- 가장 빠른 추론 속도
- 대규모 배치 처리 최적화
- 프로덕션 환경에 적합

**단점:**
- vLLM에 대한 이해 필요
- 초기 설정이 다소 복잡

## 실제 사용 사례

### 한글 전용 챗봇 서비스

한글만 지원하는 챗봇 서비스에서 중국어나 일본어 한자가 섞여 나오는 것을 방지할 수 있습니다.

### 다국어 필터링

특정 시장(예: 중동, 동남아시아)에서 특정 언어의 노출을 제한해야 하는 경우에 활용할 수 있습니다.

### 민감 토큰 차단

욕설이나 부적절한 표현이 포함된 토큰을 사전에 차단할 수 있습니다.

## 결론

vLLM의 Custom Logits Processors는 LLM의 출력을 세밀하게 제어할 수 있는 강력한 도구입니다. 특정 언어나 토큰을 차단함으로써 서비스 품질을 향상시키고, 사용자 경험을 개선할 수 있습니다.

다만 초기 지연(TTFT)과 같은 성능 이슈를 고려하여 웜업 프로세스를 구현하고, 실제 운영 환경에서 충분한 테스트를 거친 후 적용하는 것이 중요합니다.

vLLM의 고성능 추론 기능과 Custom Logits Processors를 결합하면, 프로덕션 레벨의 LLM 서비스를 더욱 안정적이고 효율적으로 운영할 수 있습니다.

## 참고 자료

- [vLLM Custom Logits Processors 공식 문서](https://docs.vllm.ai/en/stable/features/custom_logitsprocs/)
