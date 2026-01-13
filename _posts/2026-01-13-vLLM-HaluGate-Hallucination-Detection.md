---
layout: post
title: vLLM HaluGate - 토큰 레벨 환각 탐지 시스템
author: 'Juho'
date: 2026-01-13 09:00:00 +0900
categories: [vLLM]
tags: [LLM, AI, Hallucination]
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
1. [들어가며](#들어가며)
2. [핵심 문제](#핵심-문제)
3. [HaluGate 아키텍처](#halugate-아키텍처)
4. [HaluGate Sentinel 상세 분석](#halugate-sentinel-상세-분석)
5. [HaluGate Sentinel 사용 방법](#halugate-sentinel-사용-방법)
6. [2단계 탐지 시스템](#2단계-탐지-시스템)
7. [기술 구현 및 성능](#기술-구현-및-성능)
8. [vLLM Signal-Decision 프레임워크 통합](#vllm-signal-decision-프레임워크-통합)
9. [응답 처리 방식](#응답-처리-방식)
10. [적용 범위와 한계](#적용-범위와-한계)
11. [주요 사용 사례](#주요-사용-사례)
12. [평가 프레임워크](#평가-프레임워크)
13. [핵심 요약](#핵심-요약)

## 들어가며

HaluGate는 프로덕션 환경에서 LLM이 생성하는 환각(Hallucination) 문제를 해결하기 위한 토큰 레벨 탐지 시스템입니다. vLLM Semantic Router 팀에서 개발한 이 시스템은 "정확한 도구 제공 데이터를 받았음에도 불구하고 잘못된 응답을 생성하는" 문제를 조건부 토큰 레벨에서 탐지하여 사용자에게 도달하기 전에 차단합니다.

## 핵심 문제

HaluGate가 해결하려는 문제를 실제 시나리오로 살펴보겠습니다.

**상황:** 도구가 정확한 에펠탑 건설 날짜(1887-1889, 높이 330m)를 반환했지만, LLM은 완전히 잘못된 정보(1950년 건설, 높이 500m)를 생성합니다.

이는 **외재적 환각(Extrinsic Hallucination)**으로, 모델이 제공된 정확한 컨텍스트를 무시하고 잘못된 정보를 생성하는 경우입니다.

## HaluGate 아키텍처

HaluGate는 크게 두 단계로 구성된 파이프라인입니다:

### Stage 1: HaluGate Sentinel (분류)
- **역할:** 프롬프트가 사실 검증이 필요한지 판단하는 이진 분류기
- **모델:** ModernBERT-base (0.1B 파라미터)
- **파인튜닝 방식:** LoRA (rank=16, alpha=32)
- **성능:** 96.4% 검증 정확도, F1 Score 0.965, 약 12ms 추론 지연
- **효율성:** 창의적/코딩/의견 관련 쿼리를 구분하여 약 35%의 쿼리에 대해 비용이 높은 탐지 단계를 건너뜀
- **입력 제한:** 최대 512 토큰, 영어만 지원

### Stage 2: 탐지 + 설명
두 개의 상호 보완적인 모델로 구성:

#### HaluGate Detector
- **기능:** 토큰 레벨 이진 분류
- **목적:** 컨텍스트에서 지원되지 않는 정확한 토큰 식별
- **장점:** 통합 멀티클래스 접근 방식보다 우수한 결과

#### HaluGate Explainer
- **기능:** 자연어 추론(NLI) 모델
- **환각 분류:**
  - **ENTAILMENT (0):** 컨텍스트가 주장을 지원
  - **NEUTRAL (2):** 검증 불가능
  - **CONTRADICTION (4):** 직접적인 사실 오류

## HaluGate Sentinel 상세 분석

HaluGate Sentinel은 프롬프트가 사실 검증이 필요한지 판단하는 첫 번째 관문입니다.

### 분류 레이블

| 레이블 | 값 | 의미 |
|--------|-------|------|
| NO_FACT_CHECK_NEEDED | 0 | 창의적, 코딩, 의견, 추론 작업 |
| FACT_CHECK_NEEDED | 1 | 외부/세계 지식이 필요한 정보 검색 쿼리 |

### 학습 데이터 상세
총 **50,000개의 균형 잡힌 프롬프트**로 학습:

#### FACT_CHECK_NEEDED (25,000 샘플)
- **NISQ-ISQ**: Information Seeking Queries
- **HaluEval**: 환각 평가 데이터셋
- **FaithDial**: 대화형 사실 검증
- **FactCHD**: 사실 확인 데이터
- **SQuAD**: Stanford Question Answering Dataset
- **TriviaQA**: 일반 지식 질의응답
- **HotpotQA**: 다중 홉 질의응답
- **TruthfulQA**: 진실성 평가
- **CoQA**: 대화형 질의응답

#### NO_FACT_CHECK_NEEDED (25,000 샘플)
- **NISQ-NonISQ**: 비정보 검색 쿼리
- **Databricks Dolly**: 지시 따르기 데이터셋
- **WritingPrompts**: 창의적 글쓰기 프롬프트
- **Alpaca**: 지시 데이터셋

### 성능 메트릭

| 지표 | 값 |
|------|-----|
| 검증 정확도 | 96.4% |
| F1 Score | 0.965 |
| 엣지 케이스 정확도 | 100% (27개 샘플) |
| 추론 지연 (P50) | 12ms |

### 입출력 형식
**입력:**
- 타입: 텍스트 (프롬프트 문자열)
- 최대 길이: 512 토큰
- 언어: 영어만 지원

**출력:**
- 예측 레이블: `NO_FACT_CHECK_NEEDED` 또는 `FACT_CHECK_NEEDED`
- 신뢰도 점수: Float [0, 1]

## HaluGate Sentinel 사용 방법

### Python 기본 사용법
```python
from transformers import AutoModelForSequenceClassification, AutoTokenizer
import torch

MODEL_ID = "llm-semantic-router/halugate-sentinel"

# 모델 및 토크나이저 로드
model = AutoModelForSequenceClassification.from_pretrained(MODEL_ID)
tokenizer = AutoTokenizer.from_pretrained(MODEL_ID)

id2label = model.config.id2label

def classify_prompt(text: str):
    """프롬프트가 사실 확인이 필요한지 분류"""
    inputs = tokenizer(
        text,
        return_tensors="pt",
        truncation=True,
        max_length=512,
    )

    with torch.no_grad():
        outputs = model(**inputs)

    probs = torch.softmax(outputs.logits, dim=-1)[0]
    pred_id = int(torch.argmax(probs).item())
    label = id2label.get(pred_id, str(pred_id))
    confidence = float(probs[pred_id].item())

    return label, confidence

# 사용 예시
print(classify_prompt("에펠탑은 언제 지어졌나요?"))
# → ('FACT_CHECK_NEEDED', 0.99...)

print(classify_prompt("봄에 대한 시를 써줘"))
# → ('NO_FACT_CHECK_NEEDED', 0.98...)
```

### 라우터/게이트웨이 통합
```python
def route_request(user_prompt: str):
    """사실 확인 필요 여부에 따라 라우팅"""
    label, prob = classify_prompt(user_prompt)

    FACT_CHECK_THRESHOLD = 0.6

    if label == "FACT_CHECK_NEEDED" and prob >= FACT_CHECK_THRESHOLD:
        # RAG/도구/검증기를 통한 환각 탐지 경로
        route = "hallucination_gatekeeper"
        return process_with_verification(user_prompt)
    else:
        # 직접 생성 경로
        route = "direct_generation"
        return process_direct(user_prompt)

# 사용 예시
response = route_request("Python에서 리스트를 정렬하는 방법은?")
```

## 2단계 탐지 시스템

### 1단계: 사실 확인 필요 여부 분류
```python
# HaluGate Sentinel이 쿼리 분석
query = "에펠탑은 언제 지어졌나요?"
needs_fact_check = sentinel.classify(query)  # True
```

### 2단계: 환각 탐지 및 설명
```python
# Detector: 토큰 레벨 검증
unsupported_tokens = detector.identify(response, context)

# Explainer: 환각 유형 분류
classification = explainer.categorize(claim, context)
# CONTRADICTION (4) - 직접적인 사실 오류
```

## 기술 구현 및 성능

### 아키텍처
- **기반 모델:** ModernBERT-base
- **실행 환경:** Candle (Rust ML 프레임워크) + Go 바인딩
- **배포 방식:** 네이티브 실행

### 성능 지표

| 구성 요소 | 지연 시간 (P50) |
|---------|----------------|
| Fact-check 분류기 | 12ms |
| Hallucination 탐지기 | 45ms |
| NLI 설명기 | 18ms |
| 총 지연 시간 | 76-162ms |

### 네이티브 Rust 실행의 주요 장점

| 항목 | Python 런타임 | Rust 네이티브 |
|------|--------------|--------------|
| 콜드 스타트 | 5-10초 | 500ms 미만 |
| 메모리 사용량 | 2-4GB | 500MB-1GB |
| 런타임 오버헤드 | 높음 | 거의 없음 |

## vLLM Signal-Decision 프레임워크 통합

HaluGate는 vLLM의 Signal-Decision 프레임워크 내에서 새로운 신호 타입으로 작동하여 조건부 라우팅을 가능하게 합니다.

### 설정 예시
```yaml
decisions:
  - name: "factual-query-with-verification"
    priority: 100
    rules:
      operator: "AND"
      conditions:
        - type: "fact_check"
          name: "needs_fact_check"
    plugins:
      - type: "hallucination"
        configuration:
          enabled: true
          hallucination_action: "header"
```

## 응답 처리 방식

### HTTP 헤더를 통한 탐지 결과 전달
탐지 결과는 다음 HTTP 헤더로 전달됩니다:
- `x-vsr-hallucination-detected`: 환각 탐지 여부
- `x-vsr-hallucination-spans`: 환각이 발견된 토큰 범위
- `x-vsr-nli-contradictions`: NLI 모순 정보
- `x-vsr-max-severity`: 최대 심각도 레벨

### 액션 설정

| 액션 | 동작 |
|------|------|
| header | 경고 추가, 응답 통과 |
| body | 경고를 응답 본문에 주입 |
| block | 오류 반환, LLM 출력 차단 |
| none | 로깅만 수행 |

### 사용 예시
```python
# header 액션 사용 시
response_headers = {
    'x-vsr-hallucination-detected': 'true',
    'x-vsr-hallucination-spans': '[[45, 52], [78, 85]]',
    'x-vsr-nli-contradictions': 'CONTRADICTION',
    'x-vsr-max-severity': '4'
}

# block 액션 사용 시
# LLM 출력이 차단되고 에러 응답 반환
```

## 적용 범위와 한계

### 탐지 가능한 환각
**외재적 환각 (Extrinsic Hallucination)**
- 도구/RAG 컨텍스트가 검증 근거를 제공하는 경우
- 제공된 정보와 모순되는 응답 생성

### 탐지 불가능한 환각
**내재적 환각 (Intrinsic Hallucination)**
- 컨텍스트 근거가 전혀 없는 경우
- 도구 정의 없이 사실을 묻는 쿼리

### 검증 불가능한 응답 처리
HaluGate는 검증할 수 없는 사실적 응답에 대해 조용히 통과시키는 대신 투명하게 플래그를 지정합니다.

### 주요 한계점

| 한계 | 설명 |
|------|------|
| 언어 제한 | 영어만 지원 |
| 경계 사례 | 철학적 프롬프트는 신뢰도가 낮을 수 있음 |
| 검증 범위 | 사실 확인 필요 여부만 판단, 직접 검증하지 않음 |
| 도메인 특화 | 일반 목적용, 특화 도메인은 미지원 |

### 신뢰도가 낮을 수 있는 쿼리 예시
```python
# 철학적/추상적 질문
classify_prompt("진리란 무엇인가?")
# → 신뢰도가 0.6-0.7 수준으로 낮을 수 있음

# 사실과 의견이 혼합된 질문
classify_prompt("최고의 프로그래밍 언어는 무엇이며 그 이유는?")
# → 경계 사례로 분류될 수 있음
```

## 주요 사용 사례

HaluGate Sentinel은 다양한 프로덕션 시나리오에서 활용될 수 있습니다:

### 1. LLM 게이트웨이/라우터 의사결정
```python
# 쿼리 유형에 따라 다른 처리 경로로 라우팅
if is_fact_check_needed:
    # 환각 탐지 및 검증 파이프라인
    response = rag_pipeline(query)
else:
    # 직접 생성 (빠른 경로)
    response = llm.generate(query)
```

### 2. 환각 게이트키퍼 프론트라인 라우팅
사실 확인이 필요한 쿼리만 비용이 높은 환각 탐지 시스템으로 전달하여 전체 처리 비용 절감

### 3. 트래픽 분석 및 위험 점수 산정
```python
# 사실 확인 필요 쿼리 비율 모니터링
fact_check_ratio = count_fact_checks / total_queries

# 위험도 기반 우선순위 설정
if confidence > 0.9 and label == "FACT_CHECK_NEEDED":
    priority = "HIGH_RISK"
```

### 4. 인프라 사이징 최적화
```python
# 사실 확인 필요 쿼리 비율에 따라 리소스 할당
if fact_check_ratio > 0.7:
    scale_up_verification_servers()
else:
    scale_down_verification_servers()
```

## 평가 프레임워크

HaluGate는 프로덕션 사용 외에도 오프라인 평가 도구로 활용될 수 있습니다.

### 벤치마크 데이터셋
- TriviaQA
- Natural Questions
- HotpotQA

### 활용 방법
기존 QA 벤치마크를 사용하여 다양한 모델의 환각 비율을 체계적으로 측정할 수 있습니다.

```python
# 평가 예시
benchmark_results = evaluate_hallucination_rate(
    model="llama-3.1-8b",
    dataset="TriviaQA",
    halugate_config={"threshold": 0.7}
)
```

## 핵심 요약

1. **문제 정의:** LLM이 정확한 컨텍스트를 제공받고도 잘못된 정보를 생성하는 외재적 환각 문제

2. **2단계 솔루션:**
   - **Stage 1 (Sentinel):** 사실 검증 필요 여부 분류
     - ModernBERT-base (0.1B 파라미터), LoRA 파인튜닝
     - 96.4% 정확도, F1 Score 0.965, 12ms 지연
     - 50,000개 균형 샘플로 학습
   - **Stage 2:** 토큰 레벨 탐지 + NLI 기반 설명

3. **성능:**
   - 총 지연 시간: 76-162ms
   - Rust 네이티브 실행으로 콜드 스타트 <500ms
   - 메모리 사용량: 500MB-1GB
   - Python 대비 5-10배 빠른 콜드 스타트

4. **통합:**
   - vLLM Signal-Decision 프레임워크와 완벽 통합
   - HTTP 헤더를 통한 유연한 응답 처리
   - 4가지 액션 모드 지원 (header, body, block, none)
   - Python/Transformers를 통한 직접 사용 가능

5. **주요 사용 사례:**
   - LLM 게이트웨이/라우터 의사결정
   - 환각 게이트키퍼 프론트라인 라우팅
   - 트래픽 분석 및 위험 점수 산정
   - 인프라 사이징 최적화

6. **한계점:**
   - 영어만 지원
   - 철학적 질문은 신뢰도 낮음
   - 사실 검증 자체는 수행하지 않음
   - 일반 목적용, 도메인 특화 미지원

7. **기여:**
   - LettuceDetect (KRLabs) 기반
   - ModernBERT NLI 모델 활용
   - TruthfulQA, HaluEval, FaithDial 등 다양한 데이터셋 사용

## 참고 자료
- [HaluGate: Token-Level Hallucination Detection for Production LLMs](https://blog.vllm.ai/2025/12/14/halugate.html)
- [HaluGate Sentinel Model - Hugging Face](https://huggingface.co/llm-semantic-router/halugate-sentinel)
