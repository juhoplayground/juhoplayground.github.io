---
layout: post
title: "Gemma 4 로컬 모델로 Codex CLI 돌려보기 실전 테스트"
author: 'Juho'
date: 2026-04-18 00:00:00 +0900
categories: [LLM]
tags: [LLM, Ollama, Benchmark]
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
2. [실험 배경과 하드웨어](#실험-배경과-하드웨어)
   - [동기](#동기)
   - [테스트 머신](#테스트-머신)
3. [구성 과정에서의 난관](#구성-과정에서의-난관)
   - [Apple Silicon 환경](#apple-silicon-환경)
   - [NVIDIA Blackwell 환경](#nvidia-blackwell-환경)
4. [성능 비교 결과](#성능-비교-결과)
5. [실용적 권장 사항](#실용적-권장-사항)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

Daniel Vaughan은 Google Gemma 4를 클라우드 API 대신 로컬에서 실행하여 일상적인 에이전트 코딩 작업에 사용할 수 있는지 검증했다.
이번 실험의 핵심 동기는 누적되는 API 비용, 민감한 코드베이스의 프라이버시 제약, 외부 서비스 의존성이라는 세 가지였다.
결정적으로 Gemma 4는 함수 호출 정확도가 Gemma 3의 6.6%에서 86.4%로 향상되어, 비로소 로컬 도구 사용이 현실적으로 가능해졌다.

## 실험 배경과 하드웨어

### 동기

저자는 클라우드 LLM에 의존한 코딩 워크플로에서 비용과 프라이버시 부담을 느껴 왔다.
Gemma 4의 도구 호출 능력이 비약적으로 개선되면서 로컬 에이전트 코딩이 실제로 작동하는지 확인할 시점이 되었다고 판단했다.

### 테스트 머신

두 가지 구성을 비교했다.

| 머신 | 사양 | 모델 |
|------|------|------|
| MacBook Pro | M4 Pro, 24GB | Gemma 4 26B MoE |
| Dell GB10 | NVIDIA Blackwell, 128GB 통합 메모리 | Gemma 4 31B Dense |

두 환경 모두 Codex CLI의 커스텀 프로바이더로 등록했고 `wire_api = "responses"` 설정을 사용했다.

## 구성 과정에서의 난관

### Apple Silicon 환경

Ollama는 스트리밍 버그와 Flash Attention 프리징 문제로 사용할 수 없었다.
저자는 llama.cpp로 전환해 6개의 핵심 플래그를 적용했다.
컨텍스트 양자화를 적용하자 KV 캐시 요구량이 940MB에서 499MB로 줄었다.
Codex CLI에서 웹 검색 지원은 비활성화해야 거부 오류를 피할 수 있었다.

### NVIDIA Blackwell 환경

vLLM은 PyTorch 버전이 신형 GPU 아키텍처와 호환되지 않아 실패했다.
결국 Ollama v0.20.5가 가장 안정적으로 작동했고, SSH 터널링을 통해 Codex CLI에 연결했다.

## 성능 비교 결과

동일한 과제로 CSV 파싱 함수와 테스트 코드 작성을 지시했다.

| 환경 | 결과 | 소요 |
|------|------|------|
| GPT-5.4 (클라우드) | 5개 테스트 한 번에 통과 | 65초 |
| GB10 (Gemma 4 31B) | 도구 호출 3회로 정상 동작 | 7분 |
| MacBook (Gemma 4 26B MoE) | 도구 호출 10회, 테스트 5회 재작성 | 더 느림 |

토큰 생성 속도는 Mac이 초당 52토큰, GB10이 초당 10토큰으로 Mac이 5.1배 빨랐다.
그러나 첫 시도 신뢰성이 더 중요해서 느린 GPU 쪽이 전체 작업을 더 빨리 끝냈다.
MoE 아키텍처는 26B 중 토큰당 3.8B 파라미터만 활성화하여 더 작은 하드웨어에서 더 높은 처리량을 보였다.

## 실용적 권장 사항

Apple Silicon 사용자는 Gemma 4의 도구 템플릿 처리를 위해 llama.cpp의 `--jinja` 플래그를 사용해야 한다.
컨텍스트는 32,768 토큰으로 설정하고 KV 캐시 양자화를 적용한다.
NVIDIA 환경에서는 Ollama v0.20.5가 가장 신뢰할 만하다.
긴 추론 사이클 중 세션이 조기 종료되지 않도록 `stream_idle_timeout_ms`를 1,800,000ms로 늘려야 한다.

## 결론

로컬 모델은 여전히 복잡한 작업에서 클라우드 대안보다 느리고 덜 신뢰할 만하다.
하지만 격차가 좁혀지면서 하이브리드 워크플로가 실용적 선택이 되었다.
저자는 반복 작업과 프라이버시가 중요한 작업에는 로컬 추론을, 까다로운 작업에는 클라우드를 유지하는 방식을 권장한다.

## Reference

- [I Ran Gemma 4 as a Local Model in Codex CLI](https://medium.com/google-cloud/i-ran-gemma-4-as-a-local-model-in-codex-cli-7fda754dc0d4/)
