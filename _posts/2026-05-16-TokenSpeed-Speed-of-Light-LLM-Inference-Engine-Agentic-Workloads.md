---
layout: post
title: "TokenSpeed: 에이전트 워크로드를 위한 빛의 속도 LLM 추론 엔진"
author: 'Juho'
date: 2026-05-16 00:00:00 +0900
categories: [LLM]
tags: [LLM, vLLM, GPU, Benchmark]
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
3. [아키텍처 구성요소](#아키텍처-구성요소)
   - [모델링 레이어](#모델링-레이어)
   - [스케줄러](#스케줄러)
   - [커널 레이어](#커널-레이어)
   - [엔트리포인트](#엔트리포인트)
4. [성능 결과](#성능-결과)
5. [평가 방법론](#평가-방법론)
6. [개발 현황과 협력 생태계](#개발-현황과-협력-생태계)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

TokenSpeed는 에이전트 워크로드에 최적화된 LLM 추론 엔진이다.
LightSeek Foundation이 NVIDIA, AMD 등과 협력하여 2026년 3월 중순부터 개발을 시작했다.
TensorRT-LLM에 필적하는 성능과 vLLM 수준의 사용성을 동시에 제공하는 것을 목표로 한다.

프로젝트는 현재 "활발한 개발 중"이며, 향후 한 달간 프로덕션 하드닝이 계획되어 있다.
이번 릴리스는 특정 벤치마크 결과 재현에 초점을 맞춘 프리뷰 버전으로 명시되어 있다.

## 배경

Claude Code와 Cursor 같은 에이전트 코딩 시스템이 막대한 토큰 볼륨을 생성하면서 추론 인프라에 새로운 도전 과제가 등장했다.
한계 효율 개선조차 프로덕션 GPU 플릿 전반에서 상당한 용량 절감으로 직결되는 환경이 형성되었다.
TokenSpeed는 이러한 에이전트 트래픽 패턴을 일급 시민으로 가정하고 설계되었다.

기존 추론 엔진들은 일반적인 챗봇 워크로드를 전제로 만들어진 경우가 많아, 긴 컨텍스트와 도구 호출이 반복되는 에이전트 트래픽에서는 최적화 여지가 남아 있었다.
TokenSpeed는 이 간극을 메우기 위해 컴파일러, 스케줄러, 커널을 모두 새로 설계했다.

## 아키텍처 구성요소

TokenSpeed는 네 개의 주요 레이어로 구성된다.

| 레이어 | 역할 |
|--------|------|
| 모델링 | 로컬 SPMD 설계와 정적 컴파일 |
| 스케줄러 | C++ 제어 평면, Python 실행 평면 |
| 커널 | 플러그형 모듈 서브시스템 |
| 엔트리포인트 | SMG 통합 AsyncLLM |

### 모델링 레이어

TokenSpeed는 성능과 사용성의 균형을 맞춘 로컬 SPMD(Single Program, Multiple Data) 설계를 채택했다.
개발자는 I/O 배치 어노테이션만 명시하면 된다.
경량 정적 컴파일러가 모델 구성 시점에 필요한 collective 연산을 자동으로 생성한다.

이 방식은 사용자가 수동으로 병렬화 코드를 작성할 필요를 제거한다.
어노테이션 기반 접근으로 모델 정의는 단순하게 유지하면서, 멀티 GPU 분산 실행을 컴파일러가 책임진다.

### 스케줄러

스케줄러는 제어 평면과 실행 평면을 분리한다.
제어 평면은 유한 상태 머신으로 동작하며, 타입 시스템과 협력하여 컴파일 타임에 안전한 리소스 관리를 강제한다.
실행 평면은 개발 효율을 위해 Python으로 구현되었다.

요청 라이프사이클과 KV 캐시 관리를 모두 유한 상태 머신으로 모델링한 점이 핵심이다.
런타임이 아닌 컴파일 타임에 자원 안전성을 검증하므로, 프로덕션 환경에서 발생할 수 있는 자원 누수와 경합 문제를 사전에 차단할 수 있다.

### 커널 레이어

커널은 일급 모듈형 서브시스템으로 다뤄진다.
이식 가능한 공개 API, 중앙 레지스트리, 이기종 가속기를 위한 확장 가능한 플러그인 메커니즘을 제공한다.
NVIDIA Blackwell을 위한 특화 MLA(Multi-head Latent Attention) 커널이 포함되어 있다.

MLA 커널에는 다음 최적화가 적용되어 있다.

- 바이너리 prefill에서 정밀하게 튜닝된 softmax 구현
- 디코드 연산에서 Tensor Core 활용도를 높이는 query-sequence 축 그룹화
- NVIDIA 내부 노브 조정을 통한 성능 튜닝

### 엔트리포인트

CPU 측 요청 처리 오버헤드를 최소화하기 위해 SMG 통합 AsyncLLM이 도입되었다.
SMG는 CPU 측 요청 처리를 담당하여 GPU 자원이 토큰 생성에 집중할 수 있게 한다.

## 성능 결과

TokenSpeed는 Kimi K2.5 모델과 B200 하드웨어 환경에서 TensorRT-LLM과 비교 평가되었다.
코딩 에이전트 워크로드 트레이스 기반 결과는 다음과 같다.

| 지표 | TokenSpeed 우위 |
|------|------------------|
| 단일 배치 성능 | TensorRT-LLM 대비 약 9% 빠름 |
| 고처리량 시나리오 | 사용자당 100 TPS 부근에서 GPU당 TPM 약 11% 상승 |
| MLA 디코드 지연 | 투기적 디코딩 워크로드에서 거의 절반 수준 |

특히 MLA 커널은 투기적 디코딩(speculative decoding)을 사용하는 일반적인 디코드 워크로드에서 TensorRT-LLM 대비 지연 시간을 거의 절반으로 줄였다고 보고되었다.

## 평가 방법론

TokenSpeed 벤치마킹은 표준 벤치마크가 아닌 SWE-smith 트레이스를 사용했다.
SWE-smith 트레이스는 프로덕션 코딩 에이전트 트래픽을 가까이 반영하도록 구성되어 있다.

평가는 사용자당 TPS(tokens per second) 하한선을 유지하면서 GPU당 TPM(tokens per minute)을 최대화하는 방향으로 최적화되었다.
일반적인 TPS 하한은 70 TPS로 설정되었다.
이는 단순 throughput 극대화가 아니라, 사용자가 체감하는 응답 속도를 보장한 채로 처리량을 측정하는 방식이다.

## 개발 현황과 협력 생태계

TokenSpeed는 명시적으로 프리뷰 릴리스로 표기되어 있다.
개발 중인 주요 항목은 다음과 같다.

- 모델 커버리지 확장
- 런타임 기능: prefill disaggregation, KV storage, VLM 지원
- 플랫폼 최적화

프로덕션 배포는 현재 버전에서 권장되지 않는다.
Getting Started, 서버 실행, 모델 레시피, 설정, 병렬화 전략을 다루는 가이드 문서가 함께 제공된다.

프로젝트에는 NVIDIA, AMD, Qwen Inference, Together AI, Mooncake, LongCat, FluentLLM, EvalScope의 개발자가 참여했다.
컴퓨트 자원은 OpenAI, NVIDIA, AMD, Verda, Nebius에서 지원했다.

## 결론

TokenSpeed는 에이전트 워크로드라는 명확한 타겟을 잡고 추론 엔진 스택 전체를 재설계한 시도다.
컴파일러 기반 병렬화 모델링, 컴파일 타임 자원 안전성을 보장하는 스케줄러, 모듈형 커널 시스템이라는 세 축이 핵심이다.

Kimi K2.5와 B200 환경에서 TensorRT-LLM 대비 단일 배치 9%, 고처리량 시나리오 TPM 11% 우위, 그리고 MLA 디코드 지연 절반 수준의 결과를 보였다.
프리뷰 단계임을 감안하면, 프로덕션 하드닝과 모델 커버리지 확장이 완료된 이후의 행보가 더 중요한 평가 기준이 될 것이다.

에이전트 트래픽이 추론 워크로드의 중심으로 이동하는 흐름에서, TokenSpeed는 워크로드 특성에 맞춰 인프라를 재정의하는 접근의 대표 사례로 자리잡을 가능성이 있다.

## Reference

- [TokenSpeed: A Speed-of-Light LLM Inference Engine for Agentic Workloads](https://lightseek.org/blog/lightseek-tokenspeed.html)
- [lightseekorg/tokenspeed GitHub Repository](https://github.com/lightseekorg/tokenspeed)
