---
layout: post
title: "llmfit - 내 하드웨어에 맞는 LLM 모델을 찾아주는 터미널 도구"
author: 'Juho'
date: 2026-03-21 00:00:00 +0900
categories: [LLM]
tags: [LLM, GPU]
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
2. [주요 기능](#주요-기능)
   - [하드웨어 감지](#하드웨어-감지)
   - [다차원 스코어링](#다차원-스코어링)
   - [동적 양자화](#동적-양자화)
3. [설치 방법](#설치-방법)
4. [사용법](#사용법)
   - [TUI 모드](#tui-모드)
   - [CLI 모드](#cli-모드)
   - [REST API 서버](#rest-api-서버)
5. [동작 원리](#동작-원리)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

llmfit은 내 하드웨어에서 어떤 LLM 모델이 효과적으로 실행될 수 있는지 판단해주는 터미널 애플리케이션이다.
시스템 사양(RAM, CPU 코어, GPU VRAM)을 자동 감지하고 모델별 다차원 점수를 매겨 최적의 모델을 추천한다.
206개 이상의 모델 데이터베이스를 기반으로 Meta Llama, Mistral, Qwen, Gemma, Phi, DeepSeek 등 주요 모델을 지원한다.

## 주요 기능

### 하드웨어 감지

llmfit은 시스템의 RAM, CPU 코어 수, GPU 사양을 자동으로 감지한다.
NVIDIA, AMD, Intel Arc, Apple Silicon, Ascend 등 다양한 GPU를 지원한다.
감지된 하드웨어 정보를 바탕으로 각 모델의 실행 가능성을 평가한다.

### 다차원 스코어링

모델을 품질, 속도, 메모리 적합도, 컨텍스트 윈도우 능력의 네 가지 차원에서 평가한다.
용도별 가중치를 적용하여 사용 목적에 맞는 모델을 추천한다.
적합도는 Perfect, Good, Marginal, Too Tight 네 단계로 분류된다.

### 동적 양자화

하드웨어에 맞는 최적의 양자화 수준을 자동으로 선택한다.
Q8_0부터 Q2_K까지 다양한 양자화 레벨에 대해 메모리 요구량을 계산한다.
Mixture-of-Experts 아키텍처(Mixtral, DeepSeek)의 전문가 오프로딩도 지원한다.

## 설치 방법

| 플랫폼 | 설치 명령 |
|--------|-----------|
| macOS (Homebrew) | brew install llmfit |
| Windows (Scoop) | scoop install llmfit |
| Linux/macOS (스크립트) | curl -fsSL https://llmfit.axjns.dev/install.sh 파이프 sh |
| Docker | docker run ghcr.io/alexsjones/llmfit |
| 소스 빌드 | git clone 후 cargo build --release |

소스에서 빌드하려면 Rust와 Cargo가 필요하다.

## 사용법

### TUI 모드

기본 실행 시 대화형 터미널 UI가 실행된다.

```bash
llmfit
```

vim 스타일 키바인딩을 지원한다.

| 키 | 기능 |
|----|------|
| j/k 또는 위/아래 | 모델 탐색 |
| / | 이름, 제공자, 파라미터로 검색 |
| f | 적합도 필터 순환 (All, Runnable, Perfect, Good, Marginal) |
| s | 정렬 컬럼 변경 |
| v | 다중 모델 시각적 비교 |
| p | 플랜 모드 (하드웨어 요구사항 추정) |
| d | 선택한 모델 다운로드 |
| t | 테마 변경 (6종) |
| q | 종료 |

### CLI 모드

명령줄에서 직접 결과를 확인할 수 있다.

```bash
llmfit --cli                          # 전체 모델 테이블 뷰
llmfit fit --perfect -n 5             # 상위 5개 완벽 적합 모델
llmfit system                         # 감지된 하드웨어 정보 표시
llmfit recommend --json --limit 5     # 상위 5개 추천을 JSON으로 출력
llmfit search "llama 8b"              # 모델 검색
```

GPU 메모리를 수동으로 지정하거나 컨텍스트 길이를 제한할 수도 있다.

```bash
llmfit --memory=32G
llmfit --max-context 4096 --cli
```

### REST API 서버

HTTP 엔드포인트로 모델 추천을 제공할 수 있다.

```bash
llmfit serve --host 0.0.0.0 --port 8787
```

클러스터 스케줄링 등에 활용할 수 있는 API 엔드포인트를 제공한다.
Ollama, llama.cpp, MLX, Docker Model Runner 등 다양한 백엔드와 통합된다.

## 동작 원리

llmfit의 내부 동작은 5단계로 이루어진다.

| 단계 | 설명 |
|------|------|
| 하드웨어 감지 | 가용 메모리, CPU 수, GPU 능력 식별 |
| 모델 스코어링 | 네 가지 차원에 가중치를 적용하여 점수 산출 |
| 메모리 추정 | 양자화 레벨별 메모리 요구량 계산 |
| 속도 계산 | GPU 대역폭 기반 토큰 생성 처리량 추정 |
| 적합도 분석 | 메모리 적합도에 따라 4단계 분류 |

속도 계산 공식은 (bandwidth_GB_s / model_size_GB) × 0.55이다.
GPU 모드는 순수 GPU, MoE 전문가 오프로드, CPU+GPU 하이브리드, CPU 전용의 네 가지를 지원한다.
자동으로 하드웨어에 맞는 최적의 양자화와 실행 모드를 선택한다.

## 결론

llmfit은 로컬 LLM을 실행하려는 사용자에게 하드웨어 기반의 객관적인 모델 추천을 제공한다.
TUI, CLI, REST API 세 가지 모드를 지원하여 개인 사용부터 클러스터 스케줄링까지 다양한 시나리오에 대응한다.
MIT 라이선스로 공개되어 있다.

## Reference

- [llmfit GitHub Repository](https://github.com/AlexsJones/llmfit/)
