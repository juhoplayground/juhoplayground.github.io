---
layout: post
title: "SuperGemma4 26B Uncensored MLX 4bit v2 - Apple Silicon용 고속 로컬 에이전트 모델"
author: 'Juho'
date: 2026-04-21 00:00:00 +0900
categories: [LLM]
tags: [LLM, Benchmark, AI]
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
   - [기본 정보](#기본-정보)
   - [특징](#특징)
3. [성능 벤치마크](#성능-벤치마크)
4. [사용법](#사용법)
   - [서버 실행](#서버-실행)
   - [빠른 테스트](#빠른-테스트)
5. [의미와 시사점](#의미와-시사점)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

Hugging Face에 Jiunsong이 공개한 `supergemma4-26b-uncensored-mlx-4bit-v2`는 Gemma 4 26B를 Apple Silicon용으로 튜닝한 MLX 4비트 양자화 모델이다.
텍스트 전용 플래그십 모델로, 로컬 에이전트 태스크의 속도와 역량을 동시에 겨냥한다.

원본 Gemma 4 26B IT 대비 속도와 품질 양쪽에서 개선을 보고하며, 한국어 처리와 코드·도구 사용 성능에 특히 강점을 둔다.

## 모델 스펙

### 기본 정보

| 항목 | 값 |
|------|------|
| Base Model | google/gemma-4-26B-A4B-it |
| Format | MLX 4-bit quantization |
| 파일 크기 | 약 13GB |
| 파라미터 | 25B |
| Tensor Type | BF16, U32 |
| License | Gemma |

### 특징

Uncensored 동작을 유지하면서도 출력이 불안정해지지 않도록 튜닝됐다.
멀티모달이 아닌 텍스트 전용 실행에 집중해 속도를 끌어올렸다.
코드, 브라우저 작업, 툴 사용, 플래닝, 한국어에서 강점을 갖도록 조정됐다.
최근 한 달 다운로드는 8,908회로 이미 상당한 커뮤니티 사용을 확보한 상태다.

## 성능 벤치마크

원본 Gemma 4 26B IT와 SuperGemma Fast의 Quick Bench 결과를 비교한 수치다.

| Metric | Gemma 4 26B IT | SuperGemma Fast | Improvement |
|--------|----------------|-----------------|-------------|
| Quick Bench Overall | 91.4 | 95.8 | +4.4 |
| Generation Speed | 42.5 tok/s | 46.2 tok/s | +8.7% |
| Code | 92.3 | 98.6 | +6.3 |
| Logic | 86.9 | 95.2 | +8.3 |
| Korean | 90.7 | 95.0 | +4.3 |
| Browser | 87.5 | 89.6 | +2.1 |
| System Design | 97.8 | 98.9 | +1.1 |

코드(+6.3)와 로직(+8.3) 지표의 개선 폭이 특히 크다.
토큰 생성 속도도 8.7% 빨라져 4비트 양자화 기본 구성 대비 품질과 속도 양쪽에서 우위를 보인다고 주장한다.

## 사용법

### 서버 실행

MLX의 OpenAI 호환 서버로 바로 띄울 수 있다.

```bash
mlx_lm.server \
  --model Jiunsong/supergemma4-26b-uncensored-mlx-4bit-v2 \
  --port 8080
```

채팅 템플릿이 자동 감지되므로 `--chat-template`에 리터럴 경로 문자열을 넘기지 말아야 한다.
리터럴 경로를 전달하면 응답이 깨진다고 모델 카드는 경고한다.

### 빠른 테스트

단일 프롬프트 생성은 `mlx_lm.generate`로 바로 확인할 수 있다.

```bash
mlx_lm.generate \
  --model Jiunsong/supergemma4-26b-uncensored-mlx-4bit-v2 \
  --prompt "Write a Python function that returns prime numbers up to n." \
  --max-tokens 512
```

배포 파일에는 `benchmark_quick_bench_20260412.json`, 응답 로그 jsonl, `SERVING_NOTES.md`가 포함되어 있어 재현성 검증이 가능하다.

## 의미와 시사점

Apple Silicon MLX 생태계에서 26B급 모델을 13GB 크기로 구동할 수 있다는 점은 로컬 에이전트 워크로드의 진입 장벽을 낮춘다.
기존 4비트 베이스라인 대비 품질이 저하되지 않고 오히려 개선되는 케이스가 늘어나고 있어, 로컬 모델 선택지가 빠르게 다양화되고 있다.

한국어 성능 지표가 별도 항목으로 측정·공개된 것도 주목할 만하다.
로컬 에이전트에서 한국어 도구 사용·플래닝 품질을 공식 수치로 제시하는 릴리즈는 여전히 드물다.

## 결론

SuperGemma4 26B Uncensored MLX 4bit v2는 Apple Silicon 사용자가 로컬에서 고속 에이전트 태스크를 돌리기 위한 실용적 옵션이다.
13GB 모델 크기와 46.2 tok/s 생성 속도, 그리고 한국어·코드·로직 벤치마크에서의 개선이 선택 근거가 된다.
MLX 4비트 양자화 환경에서 Gemma 4 계열을 찾고 있다면 기본 후보군에 올려볼 가치가 있다.

## Reference

- [Jiunsong/supergemma4-26b-uncensored-mlx-4bit-v2](https://huggingface.co/Jiunsong/supergemma4-26b-uncensored-mlx-4bit-v2)
