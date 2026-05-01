---
layout: post
title: "DeepSeek-V4-Pro 공개 - 1.6T MoE, 49B 활성 파라미터, 1M 컨텍스트, FP4/FP8 혼합 정밀도"
author: 'Juho'
date: 2026-04-30 00:00:00 +0900
categories: [LLM]
tags: [LLM, Benchmark, Context, GPU]
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
2. [아키텍처와 학습](#아키텍처와-학습)
   - [하이브리드 어텐션](#하이브리드-어텐션)
   - [mHC와 Muon 옵티마이저](#mhc와-muon-옵티마이저)
3. [추론 모드와 벤치마크](#추론-모드와-벤치마크)
   - [세 가지 추론 모드](#세-가지-추론-모드)
   - [주요 벤치마크 결과](#주요-벤치마크-결과)
4. [모델 배리언트와 사용법](#모델-배리언트와-사용법)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

DeepSeek-AI가 DeepSeek-V4-Pro를 Hugging Face에 공개했다.
총 1.6조 파라미터 중 49억이 활성화되는 MoE 구조이며, 컨텍스트 길이는 100만 토큰까지 지원한다.
MoE 전문가는 FP4로, 나머지는 FP8로 혼합 정밀도를 적용해 효율을 끌어올렸다.
모델은 MIT 라이선스로 공개되어 상업적 활용까지 허용된다.

## 아키텍처와 학습

### 하이브리드 어텐션

DeepSeek-V4는 Compressed Sparse Attention(CSA)과 Heavily Compressed Attention(HCA)을 결합한 하이브리드 어텐션을 도입한다.
단일 토큰 추론 FLOPs는 DeepSeek-V3.2 대비 약 27% 수준으로 줄었다.
1M 토큰 컨텍스트에서의 KV 캐시도 약 10% 수준으로 압축됐다.
긴 컨텍스트를 다룰 때 계산과 메모리 비용을 동시에 낮춘 것이 이번 버전의 가장 큰 특징이다.

### mHC와 Muon 옵티마이저

Manifold-Constrained Hyper-Connections(mHC)는 잔차 연결을 강화해 신호가 층 사이에서 더 안정적으로 전파되도록 설계됐다.
Muon 옵티마이저는 수렴 속도와 학습 안정성을 동시에 개선하는 역할을 한다.
사전학습 데이터는 32조 토큰 이상의 고품질 데이터로 구성됐다.
사후학습은 두 단계로 진행된다.
1단계는 도메인별 전문가 모델을 SFT와 GRPO 기반 RL로 양성하고, 2단계에서 on-policy distillation으로 단일 모델로 통합한다.

## 추론 모드와 벤치마크

### 세 가지 추론 모드

| 모드 | 특징 | 사용 사례 |
|------|------|----------|
| Non-Think | 빠르고 직관적 응답 | 일상적 작업, 저위험 결정 |
| Think High | 의식적 논리 분석 | 복잡한 문제 해결, 계획 |
| Think Max | 최대 추론 노력 | 모델 추론 경계 탐색 |

Think Max 모드의 권장 컨텍스트 윈도우 최소치는 384K 토큰이다.
일반 모드의 권장 샘플링 파라미터는 temperature 1.0, top_p 1.0이다.

### 주요 벤치마크 결과

Think Max 모드의 지식과 추론 벤치마크 결과다.

| 벤치마크 | 점수 |
|---------|------|
| MMLU-Pro | 87.5 |
| SimpleQA-Verified | 57.9 |
| GPQA Diamond | 90.1 |
| Chinese-SimpleQA | 84.4 |

코드와 수학 벤치마크도 최상위권이다.

| 벤치마크 | 점수 |
|---------|------|
| LiveCodeBench | 93.5 |
| Codeforces Rating | 3206 |
| HMMT 2026 Feb | 95.2 |
| IMOAnswerBench | 89.8 |

1M 토큰 롱 컨텍스트와 에이전트 과제도 공개됐다.

| 벤치마크 | 점수 |
|---------|------|
| MRCR 1M | 83.5 |
| CorpusQA 1M | 62.0 |
| SWE Verified (Resolved) | 80.6 |
| Terminal Bench 2.0 | 67.9 |
| BrowseComp | 83.4 |

## 모델 배리언트와 사용법

네 가지 배리언트가 제공된다.

| 모델 | 전체 파라미터 | 활성 파라미터 | 컨텍스트 | 정밀도 |
|------|-------------|-------------|---------|--------|
| DeepSeek-V4-Flash-Base | 284B | 13B | 1M | FP8 Mixed |
| DeepSeek-V4-Flash | 284B | 13B | 1M | FP4 + FP8 Mixed |
| DeepSeek-V4-Pro-Base | 1.6T | 49B | 1M | FP8 Mixed |
| DeepSeek-V4-Pro | 1.6T | 49B | 1M | FP4 + FP8 Mixed |

채팅 템플릿은 별도의 인코딩 모듈을 사용한다.

```python
from encoding_dsv4 import encode_messages, parse_message_from_completion_text

messages = [
    {"role": "user", "content": "hello"},
    {"role": "assistant", "content": "Hello! I am DeepSeek.", "reasoning_content": "thinking..."},
    {"role": "user", "content": "1+1=?"}
]

prompt = encode_messages(messages, thinking_mode="thinking")

import transformers
tokenizer = transformers.AutoTokenizer.from_pretrained("deepseek-ai/DeepSeek-V4-Pro")
tokens = tokenizer.encode(prompt)
```

로컬 배포는 리포지토리의 `/encoding`과 `/inference` 디렉터리에서 모델 가중치 변환과 대화형 데모 스크립트를 제공한다.

## 결론

DeepSeek-V4-Pro는 1M 토큰 컨텍스트를 최소한의 계산 비용으로 처리할 수 있도록 설계된 개방형 MoE의 새로운 지표다.
MMLU-Pro, GPQA Diamond, LiveCodeBench에서 최상위권 오픈소스 점수를 보이며 SWE Verified도 80.6으로 높은 에이전트 성능을 증명했다.
MIT 라이선스와 FP4/FP8 혼합 정밀도 설계는 자체 호스팅 환경에서의 비용 효율성을 고려한 선택이다.

## Reference

- [DeepSeek-V4-Pro on Hugging Face](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro)
