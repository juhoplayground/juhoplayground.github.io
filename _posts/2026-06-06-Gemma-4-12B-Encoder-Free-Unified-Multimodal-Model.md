---
layout: post
title: "Gemma 4 12B: 인코더 없는 통합 멀티모달 모델"
author: 'Juho'
date: 2026-06-06 00:00:00 +0900
categories: [LLM]
tags: [LLM, AI, Benchmark, Embedding]
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
3. [기술적 혁신](#기술적-혁신)
   - [비전 처리](#비전-처리)
   - [오디오 처리](#오디오-처리)
4. [주요 역량](#주요-역량)
5. [접근성과 배포](#접근성과-배포)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

Google은 2026년 6월 3일 Gemma 4 12B를 발표했다.

정식 명칭은 "Introducing Gemma 4 12B: a unified, encoder-free multimodal model"이다.
엣지 중심의 E4B와 고성능 26B Mixture of Experts 사이를 잇는 중급 모델로 자리매김한다.

## 모델 사양

| 항목 | 내용 |
|------|------|
| 파라미터 | 120억(12B) |
| 메모리 요구 | 로컬 배포 시 16GB VRAM 또는 통합 메모리 |
| 메모리 풋프린트 | 26B 모델의 절반 미만이면서 유사 성능에 근접 |
| 아키텍처 | 통합(unified), 인코더 없는(encoder-free) 설계 |
| 라이선스 | Apache 2.0 |

비전과 오디오를 위한 별도의 멀티모달 인코더가 없다는 점이 가장 큰 특징이다.
26B 모델 대비 절반 미만의 메모리로 비슷한 성능에 다가서며, 소비자용 하드웨어에서 구동 가능한 "랩톱 레디(laptop ready)" 모델을 지향한다.

## 기술적 혁신

### 비전 처리

전통적인 비전 인코더를 행렬 곱셈과 정규화를 사용하는 경량 임베딩 모듈로 대체했다.

비전 처리는 위치 임베딩(positional embeddings)을 동반한 단일 행렬 곱셈으로 이루어진다.
이를 통해 언어 모델 백본이 시각적 이해를 직접 담당하게 된다.

### 오디오 처리

오디오 인코더는 완전히 제거했다.

원시 오디오 신호(raw audio signal)를 텍스트 토큰과 동일한 차원 공간으로 직접 투영(project)하여, 별도 인코더 없이 간결하게 처리한다.
Gemma 중급 모델 중 네이티브 오디오 입력을 지원하는 첫 모델이다.

## 주요 역량

| 역량 | 내용 |
|------|------|
| 네이티브 비전 입력 | 별도 인코더 없이 시각 입력 처리 |
| 네이티브 오디오 입력 | 중급 Gemma 최초로 오디오 입력 지원 |
| 고급 추론 | 다단계 워크플로우와 에이전트 애플리케이션 가능 |
| MTP 드래프터 | Multi-Token Prediction으로 지연시간 감소 |

벤치마크상 성능은 26B 모델에 근접하며 강력한 다단계 추론을 제공한다고 설명한다.
다중 토큰 예측(MTP) 드래프터를 통해 추론 지연을 줄이는 드래프터 레디(drafter-ready) 아키텍처도 갖췄다.

## 접근성과 배포

Gemma 4 12B는 Hugging Face와 Kaggle에서 제공되며, 다양한 프레임워크와 호환된다.

- 프레임워크: Hugging Face Transformers, llama.cpp, MLX, SGLang, vLLM, Unsloth
- 로컬 실행: LM Studio, Ollama, Google AI Edge Gallery App
- 클라우드: Google Cloud Run, GKE, Gemini Enterprise Agent Platform Model Garden

Gemma 4 모델군은 누적 다운로드 1억 5천만 건을 넘어섰으며, 로봇 보조 시스템부터 엔터프라이즈 보안 솔루션까지 다양하게 활용되고 있다.
에이전트 개발을 위한 전용 Skills Repository도 함께 제공된다.

## 결론

Gemma 4 12B는 멀티모달 역량과 온디바이스 효율성을 결합해 고급 AI를 민주화하려는 Google의 전략을 보여준다.

비전 인코더를 경량 임베딩 모듈로, 오디오 인코더를 직접 투영 방식으로 대체한 인코더 없는 통합 설계가 핵심이다.
이로써 작은 모델을 제약하던 아키텍처 병목을 제거하면서도 26B에 근접한 성능을 16GB 메모리 환경에서 구현한다.

## Reference

- [Introducing Gemma 4 12B](https://blog.google/innovation-and-ai/technology/developers-tools/introducing-gemma-4-12b/){:target="_blank"}
