---
layout: post
title: vLLM Sleep Mode - 단일 GPU에서 다중 모델 전환을 위한 제로 리로드 솔루션
author: 'Juho'
date: 2026-01-30 00:00:00 +0900
categories: [vLLM]
tags: [vLLM, LLM, GPU, Performance, Python]
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
2. [기존 방식의 문제점](#기존-방식의-문제점)
3. [Sleep Mode란](#sleep-mode란)
4. [Sleep Level 비교](#sleep-level-비교)
5. [성능 이점의 원리](#성능-이점의-원리)
6. [Quick Start](#quick-start)
7. [성능 벤치마크](#성능-벤치마크)
8. [Ablation 연구](#ablation-연구)
9. [Sleep Level 선택 가이드](#sleep-level-선택-가이드)
10. [정리](#정리)

## 개요

vLLM 블로그에서 발표한 Sleep Mode는 단일 GPU에서 다중 LLM을 효율적으로 전환하며 서빙할 수 있는 기능이다.
두 개의 LLM이 각각 GPU에 들어갈 수 있지만 동시에 로드할 수 없는 상황에서, 기존 방식은 메모리 과다 사용 또는 긴 리로드 시간 중 하나를 감수해야 했다.
Sleep Mode는 모델을 몇 초 만에 "수면" 상태로 전환하고 빠르게 다시 깨울 수 있어, 효율성과 속도를 동시에 확보한다.

## 기존 방식의 문제점

단일 GPU에서 여러 모델을 서빙하려면 일반적으로 세 가지 방법이 있다.

첫 번째는 두 모델을 동시에 메모리에 올리는 방식이다.
이 경우 GPU 메모리를 두 배로 소비하게 된다.

두 번째는 하나의 모델만 유지하고 전환 시 전체 리로드를 수행하는 방식이다.
이 경우 모델 전환마다 30~100초 이상의 지연이 발생한다.

세 번째는 가중치만 빠르게 로드하는 방식이다.
하지만 이 방식도 CUDA 그래프 캡처, JIT 컴파일, 메모리 할당자 셋업 등의 오버헤드가 여전히 존재한다.

## Sleep Mode란

Sleep Mode는 모델을 수면 상태로 전환하여 GPU 리소스를 해제하고, 필요할 때 빠르게 깨워서 추론을 수행할 수 있는 기능이다.
기존 전체 리로드 대비 18~200배 빠른 모델 전환 속도를 제공한다.
Tensor Parallelism, Pipeline Parallelism, Expert Parallelism을 모두 지원한다.

### Sleep Level 1

Level 1은 모델 가중치를 CPU RAM으로 오프로드하는 방식이다.
가장 빠른 웨이크 타임을 제공하며, CPU 메모리가 충분한 환경에 적합하다.

### Sleep Level 2

Level 2는 모델 가중치를 완전히 삭제하는 방식이다.
CPU RAM 사용이 최소화되며, 웨이크 시 가중치를 다시 로드해야 하지만 인프라 구성요소가 보존되어 전체 리로드보다 훨씬 빠르다.

## Sleep Level 비교

### Sleep Level별 성능 비교

| Mode | 총 소요 시간 | Wake Time | CPU RAM 사용량 | 적합한 사용 사례 |
|------|-------------|-----------|---------------|----------------|
| No Sleep | 357.1s | N/A | 최소 | 단일 모델 서빙 |
| Level 1 | 112.6s | 0.26s / 0.82s | 높음 | 빈번한 모델 전환 |
| Level 2 | 124.6s | 0.85s / 2.58s | 최소 | 제한된 리소스 환경 |

Level 2는 가중치를 다시 로드해야 함에도 불구하고 No Sleep 방식보다 23~45배 빠르다.
이는 메모리 할당자, CUDA 그래프, JIT 커널 등의 인프라 구성요소가 보존되기 때문이다.

## 성능 이점의 원리

Sleep Mode가 빠른 이유는 모델의 인프라 구성요소를 수면 중에도 유지하기 때문이다.
전체 리로드 시 재구축이 필요한 다섯 가지 핵심 요소는 다음과 같다.

첫째, VRAM 로드 작업이다.
둘째, 메모리 할당자(CuMemAllocator) 상태 설정이다.
셋째, CUDA 그래프 캡처이다.
넷째, GPU 커널 JIT 컴파일이다.
다섯째, 캐시 워밍업 절차이다.

Sleep Mode는 이 중 가중치 로드를 제외한 모든 구성요소를 보존한다.
이를 통해 콜드 스타트 대비 61~88% 빠른 추론 성능을 달성한다.

## Quick Start

### 모델 서버 실행

두 개의 터미널에서 각각 다음과 같이 모델 서버를 실행한다.

```bash
vllm serve microsoft/Phi-3-vision-128k-instruct --enable-sleep-mode --port 8001
```

```bash
vllm serve Qwen/Qwen3-0.6B --enable-sleep-mode --port 8002
```

### Sleep / Wake 제어

모델을 수면 상태로 전환하려면 다음 REST API를 호출한다.

```bash
# Level 1 Sleep
curl -X POST http://localhost:8001/sleep

# Level 2 Sleep
curl -X POST http://localhost:8001/sleep?level=2
```

모델을 깨우려면 다음과 같이 호출한다.

```bash
# Wake
curl -X POST http://localhost:8001/wake
```

Level 2에서 깨울 때는 추가적으로 가중치 리로드와 프리픽스 캐시 리셋이 필요하다.

```bash
# Level 2 Wake 후 추가 작업
curl -X POST http://localhost:8001/wake
curl -X POST http://localhost:8001/reload_weights
curl -X POST http://localhost:8001/reset_prefix_cache
```

### 추론 요청

```bash
curl http://localhost:8001/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "microsoft/Phi-3-vision-128k-instruct",
    "prompt": "Hello, world!",
    "max_tokens": 50
  }'
```

보안 관련 주의사항으로, Sleep/Wake 엔드포인트를 사용하려면 `VLLM_SERVER_DEV_MODE=1` 환경변수 설정이 필요하며, 신뢰할 수 있는 네트워크 환경에서만 사용해야 한다.

## 성능 벤치마크

### A100 GPU 결과 (235B 모델)

5회 모델 전환 기준으로 측정한 결과이다.

| 항목 | 결과 |
|------|------|
| Level 1 총 소요 시간 | 112.6초 |
| Wake Time | 0.26초 / 0.82초 |
| 추론 속도 향상 | 콜드 스타트 대비 61~88% 빠름 |

### A4000 GPU 결과 (소형 모델)

| 항목 | 결과 |
|------|------|
| Sleep Mode 총 소요 시간 | 85초 |
| No Sleep 총 소요 시간 | 226초 |
| 시간 절감율 | 62% |
| Wake Time | 0.1~0.8초 |
| 모델 전환 속도 | 콜드 스타트 대비 58~203배 빠름 |

전체적으로 5회 모델 전환 워크로드에서 65~68%의 총 시간 절감 효과를 보여주었다.

## Ablation 연구

### Warm-Up 영향

워밍업 수행 여부에 따른 첫 번째 추론 성능 차이를 분석한 결과이다.

| 조건 | 첫 추론 소요 시간 |
|------|------------------|
| 워밍업 수행 | 0.45초 |
| 워밍업 미수행 | 2.59초 |

워밍업을 수행하지 않으면 첫 추론이 5.8배 느려진다.
컴파일된 커널은 Sleep/Wake 사이클에서도 유지되므로, 워밍업의 효과가 누적된다.

### FP8 양자화 효과

FP8 양자화를 적용한 경우의 성능 변화이다.

| 항목 | 결과 |
|------|------|
| Wake 작업 속도 향상 | 13~33% 빠름 |
| 대형 모델 추론 속도 향상 | 30% 빠름 |
| 초기 로드 시간 증가 | 7% 느림 |

FP8 양자화는 Wake 속도와 추론 성능을 개선하지만, 초기 로드 시간이 약간 증가하는 트레이드오프가 있다.

## Sleep Level 선택 가이드

### Level 1 선택 조건

CPU 메모리가 충분한 환경에서 사용한다.
6초 이내의 빠른 Wake Time이 필요한 경우에 적합하다.
빈번하게 모델을 전환하는 워크로드에 최적이다.

### Level 2 선택 조건

CPU RAM이 제한된 환경에서 사용한다.
비용 최적화가 중요한 경우에 적합하다.
10개 이상의 모델을 관리해야 하는 경우에 유용하다.

### Sleep Mode 미사용 조건

단일 모델만 서빙하는 경우에는 Sleep Mode가 불필요하다.
모델 전환이 극히 드문 경우에도 마찬가지이다.

## 정리

vLLM Sleep Mode는 단일 GPU 환경에서 다중 LLM 서빙의 핵심 과제를 해결하는 기능이다.
Level 1은 CPU RAM을 활용한 최고 속도의 전환을, Level 2는 최소 리소스 환경에서의 효율적 전환을 제공한다.
벤치마크 결과 18~200배 빠른 모델 전환과 61~88% 빠른 추론 성능을 보여주었다.
0.6B부터 235B 파라미터 모델까지 다양한 GPU 환경(A4000~A100)에서 테스트되어 확장성이 검증되었다.
비용 효율적인 다중 모델 서빙이 필요한 환경에서 실용적인 솔루션이 될 수 있다.

## 참고

- [Zero-Reload Model Switching with vLLM Sleep Mode](https://blog.vllm.ai/2025/10/26/sleep-mode.html)
