---
layout: post
title: "LM Studio CLI로 Google Gemma 4 로컬 실행: M4 Pro에서 51 tok/s 달성"
author: 'Juho'
date: 2026-04-09 00:00:00 +0900
categories: [LLM]
tags: [AI, LLM, GPU]
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
2. [Gemma 4 26B-A4B 모델 선택 이유](#gemma-4-26b-a4b-모델-선택-이유)
3. [LM Studio 0.4.0의 변화](#lm-studio-040의-변화)
4. [설치 및 실행](#설치-및-실행)
   - [기본 설치](#기본-설치)
   - [모델 로딩 튜닝](#모델-로딩-튜닝)
5. [성능 측정 결과](#성능-측정-결과)
   - [추론 성능](#추론-성능)
   - [메모리 요구사항](#메모리-요구사항)
   - [하드웨어 사용량](#하드웨어-사용량)
6. [Claude Code 연동](#claude-code-연동)
7. [한계점](#한계점)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

LM Studio 0.4.0의 새로운 헤드리스 CLI를 활용하여 Google Gemma 4 26B 모델을 로컬 환경에서 실행하는 방법을 다룬다.
MoE(Mixture-of-Experts) 아키텍처 덕분에 26B 파라미터 모델이면서도 실제로 약 4B 파라미터만 활성화되어, 저사양 하드웨어에서도 실용적인 속도를 달성한다.
M4 Pro MacBook에서 초당 51토큰 생성, 초기 응답 1.5초 수준의 성능이 확인되었다.

## Gemma 4 26B-A4B 모델 선택 이유

Gemma 4 26B-A4B는 MoE 아키텍처를 사용하여 128개 전문가 중 8개만 활성화한다.
실제 활성 파라미터는 약 3.8B에 불과하여 밀집(dense) 모델 대비 훨씬 빠르고 메모리 효율적이다.

| 항목 | 26B-A4B (MoE) | 31B (Dense) |
|------|---------------|-------------|
| MMLU Pro | 82.6% | 85.2% |
| 속도 | 빠름 | 느림 |
| 메모리 | 적음 | 많음 |
| 최대 컨텍스트 | 256K | 256K |

약간의 성능 차이를 감수하면 훨씬 빠른 응답과 적은 메모리 사용이 가능하다.
비전 지원과 함수 호출 기능도 포함되어 있다.

## LM Studio 0.4.0의 변화

LM Studio 0.4.0은 `llmster`라는 독립형 추론 엔진을 도입했다.
그래픽 인터페이스 없이 `lms` CLI만으로 모델 다운로드, 로딩, 채팅, API 서버 구동까지 모든 작업을 수행할 수 있다.
Continuous batching을 통한 병렬 요청 처리가 가능하며, OpenAI 호환 API와 Anthropic 호환 엔드포인트를 모두 지원한다.

## 설치 및 실행

### 기본 설치

```bash
# LM Studio 설치
curl -fsSL https://lmstudio.ai/install.sh | bash

# 데몬 시작
lms daemon up

# 모델 다운로드
lms get google/gemma-4-26b-a4b

# 대화 모드 실행
lms chat google/gemma-4-26b-a4b

# API 서버 구동
lms server start
```

API 서버는 `http://localhost:1234/v1`에서 OpenAI 호환 API를 제공한다.

### 모델 로딩 튜닝

컨텍스트 길이를 조절하여 메모리 사용량을 관리할 수 있다.

```bash
# 컨텍스트 길이 지정
lms load google/gemma-4-26b-a4b --context-length 128000

# GPU 오프로딩 비율 설정
lms load google/gemma-4-26b-a4b --gpu=1.0

# 자동 언로드 시간 설정 (초)
lms load google/gemma-4-26b-a4b --ttl 1800

# 메모리 사용량 사전 추정
lms load google/gemma-4-26b-a4b --estimate-only
```

`--estimate-only` 플래그로 실제 로딩 전에 메모리 요구량을 확인할 수 있어 OOM(Out of Memory) 상황을 예방할 수 있다.

## 성능 측정 결과

### 추론 성능

M4 Pro MacBook(48GB)에서 측정한 결과이다.

| 지표 | 수치 |
|------|------|
| 토큰 생성 속도 | 51.35 tok/s |
| 첫 토큰 응답 시간 | 1.551초 |
| 프롬프트 토큰 | 39개 |
| 생성 토큰 | 176개 |

### 메모리 요구사항

컨텍스트 길이에 따라 메모리 사용량이 크게 달라진다.

| 컨텍스트 길이 | 메모리 사용량 |
|---------------|---------------|
| 기본 | 약 17.6GB |
| 48K | 약 21GB |
| 128K | 약 28GB (추정) |
| 256K | 약 37.48GB |

### 하드웨어 사용량

M4 Pro 48GB 기준 실행 시 하드웨어 현황이다.

| 항목 | 수치 |
|------|------|
| 메모리 사용률 | 46.69GB / 48GB |
| GPU 사용률 | 90% |
| P-Core CPU | 35.96% |
| E-Core CPU | 82.42% |
| 전력 | 23.56W |
| 온도 | 약 91~92도 |

48GB 메모리 환경에서 256K 컨텍스트 사용 시 스왑 압박이 발생할 수 있다.

## Claude Code 연동

Shell 함수를 통해 로컬 Gemma 4 모델을 Claude Code의 백엔드로 사용할 수 있다.

```bash
claude-lm() {
    export ANTHROPIC_BASE_URL=http://localhost:1234
    export ANTHROPIC_AUTH_TOKEN=lmstudio
    export ANTHROPIC_MODEL="gemma-4-26b-a4b"
    export CLAUDE_CODE_MAX_OUTPUT_TOKENS="8000"
    claude "$@"
}
```

이를 통해 완전한 오프라인 환경에서 API 비용 없이 코딩 어시스턴트를 활용할 수 있다.
로컬 데이터가 외부로 전송되지 않으므로 보안 면에서도 유리하다.

## 한계점

몇 가지 한계점이 확인되었다.
Gemma 4가 자신의 모델명을 정확히 인식하지 못하는 경우가 있다.
복잡한 다단계 작업에서는 상용 모델 대비 제약이 있다.
48GB 메모리 환경에서 긴 컨텍스트 사용 시 스왑 압박이 발생할 수 있다.

## 결론

MoE 아키텍처를 채택한 Gemma 4 26B-A4B는 로컬 추론에 최적화된 모델이다.
LM Studio 0.4.0의 CLI 기반 아키텍처와 결합하면, GUI 없이도 완전한 로컬 LLM 환경을 구축할 수 있다.
적절한 컨텍스트 길이 설정과 메모리 추정으로 안정적인 운영이 가능하며, API 비용 없이 오프라인 코딩 어시스턴트를 구현할 수 있다는 점이 매력적이다.

## Reference

- [Running Google Gemma 4 locally with LM Studio - George Liu](https://ai.georgeliu.com/p/running-google-gemma-4-locally-with/)
