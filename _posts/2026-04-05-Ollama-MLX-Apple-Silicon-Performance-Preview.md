---
layout: post
title: "Ollama, Apple Silicon에서 MLX 기반 구동 프리뷰 - 최대 2배 성능 향상"
author: 'Juho'
date: 2026-04-05 00:00:00 +0900
categories: [Ollama]
tags: [Ollama, LLM, GPU]
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
   - [성능 개선 수치](#성능-개선-수치)
   - [NVFP4 포맷 지원](#nvfp4-포맷-지원)
   - [캐시 개선](#캐시-개선)
4. [설치 및 사용법](#설치-및-사용법)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Ollama가 Apple의 머신러닝 프레임워크인 [MLX](https://ollama.com/blog/mlx){:target="_blank"} 기반으로 구동되는 프리뷰 버전을 공개했다.
Apple Silicon의 통합 메모리 아키텍처를 활용하여 Prefill 성능 1.5배 이상, Decode 성능 약 2배 향상을 달성했다.

## 배경

Apple Silicon Mac은 CPU와 GPU가 메모리를 공유하는 통합 메모리 아키텍처를 사용한다.
기존 Ollama는 이 아키텍처의 장점을 충분히 활용하지 못했지만, Apple의 MLX 프레임워크를 통합함으로써 M 시리즈 칩의 GPU Neural Accelerator를 직접 활용할 수 있게 되었다.
특히 M5 시리즈 칩에서 최적화된 성능을 보여준다.

## 핵심 내용

### 성능 개선 수치

Qwen3.5-35B-A3B 모델 기준 테스트 결과는 다음과 같다.

| 항목 | 수치 |
|------|------|
| Prefill 속도 | 초당 1,851 토큰 (기존 대비 1.5배 이상) |
| Decode 속도 | 초당 134 토큰 (기존 대비 약 2배) |
| TTFT 개선 | M5 시리즈 GPU Neural Accelerator 활용 |

첫 토큰 생성 시간(TTFT)과 토큰 생성 속도 모두 의미 있는 개선을 보여준다.

### NVFP4 포맷 지원

NVIDIA의 NVFP4 포맷을 지원하여 모델 정확도를 유지하면서 메모리 대역폭을 줄인다.
이를 통해 메모리 효율성이 향상되고 저장소 요구량도 감소한다.
기존 양자화 방식 대비 정확도 손실이 적은 것이 특징이다.

### 캐시 개선

캐시 재사용과 스마트 캐시 정책이 도입되었다.
지능형 체크포인트와 스마트 제거 기능으로 메모리 사용량을 줄이면서도 응답 속도를 향상시킨다.
반복적인 프롬프트나 유사한 요청에서 특히 효과적이다.

## 설치 및 사용법

32GB 이상의 통합 메모리를 갖춘 Mac이 필요하다.
Ollama 0.19 버전을 다운로드하여 사용할 수 있다.

```bash
# Claude Code와 함께 사용
ollama launch claude --model qwen3.5:35b-a3b-coding-nvfp4

# OpenClaw와 함께 사용
ollama launch openclaw --model qwen3.5:35b-a3b-coding-nvfp4

# 직접 실행
ollama run qwen3.5:35b-a3b-coding-nvfp4
```

향후 더 많은 모델 지원과 커스텀 모델 가져오기 기능이 추가될 예정이다.

## 결론

Ollama의 MLX 통합은 Apple Silicon Mac에서의 로컬 LLM 실행 경험을 크게 개선한다.
Prefill 1.5배, Decode 2배라는 실질적인 성능 향상과 함께, NVFP4 포맷 지원과 스마트 캐시로 메모리 효율도 높아졌다.
32GB 이상 Mac 사용자라면 온디바이스 LLM을 보다 빠르고 효율적으로 활용할 수 있게 되었다.

## Reference

- [Ollama MLX Blog Post](https://ollama.com/blog/mlx/)
