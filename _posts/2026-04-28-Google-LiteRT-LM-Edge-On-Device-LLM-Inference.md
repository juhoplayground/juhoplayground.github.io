---
layout: post
title: "Google LiteRT-LM: 엣지 디바이스용 고성능 온디바이스 LLM 추론 프레임워크"
author: 'Juho'
date: 2026-04-28 00:00:00 +0900
categories: [AI]
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
2. [핵심 기능](#핵심-기능)
   - [크로스 플랫폼 지원](#크로스-플랫폼-지원)
   - [멀티모달과 Function Calling](#멀티모달과-function-calling)
3. [설치와 사용](#설치와-사용)
4. [아키텍처와 기술 스택](#아키텍처와-기술-스택)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Google이 엣지 디바이스에 대형 언어 모델을 배포하기 위한 프로덕션 레디 오픈소스 추론 프레임워크 LiteRT-LM을 공개했다.
이 프로젝트는 Chrome, Chromebook Plus, Pixel Watch 등 Google의 실제 제품에서 이미 사용되고 있는 검증된 인프라다.
스마트폰, 태블릿, 웹, PC, IoT 기기 같은 다양한 엣지 환경에서 LLM을 실행하도록 설계되었으며, 최신 Gemma 4 모델을 기본 지원한다.
GPU와 NPU 하드웨어 가속을 활용하여 제한된 디바이스 자원 위에서도 최적화된 성능을 제공한다.

## 핵심 기능

### 크로스 플랫폼 지원

LiteRT-LM은 Android, iOS, Web, Desktop, IoT(라즈베리파이 등) 플랫폼을 아우른다.
하드웨어 가속 측면에서는 GPU와 NPU 가속기를 통해 최고 성능을 확보하며, 디바이스 특성에 맞춘 최적화를 지원한다.
지원 언어는 다음과 같다.

| 언어 | 상태 | 용도 |
|------|------|------|
| Kotlin | 안정화 | Android/JVM |
| Python | 안정화 | 프로토타이핑 |
| C++ | 안정화 | 고성능 네이티브 |
| Swift | 개발 중 | iOS/macOS |

지원하는 모델은 Gemma, Llama, Phi-4, Qwen 등 주요 오픈소스 LLM 계열을 망라한다.

### 멀티모달과 Function Calling

LiteRT-LM은 단순한 텍스트 생성 엔진을 넘어선다.
시각과 음성 입력을 처리하는 멀티모달 기능을 내장하고 있으며, 에이전트 워크플로우를 구현하기 위한 Function Calling도 지원한다.
이는 온디바이스 환경에서 도구를 활용하는 AI 에이전트를 구축할 수 있음을 의미한다.
Chrome 브라우저의 내장 AI 기능과 Pixel Watch 같은 웨어러블에서의 실시간 응답이 이 인프라 위에서 동작한다.

## 설치와 사용

LiteRT-LM은 간단한 CLI 명령으로 Hugging Face에서 모델을 다운로드하고 즉시 실행할 수 있다.
최신 Gemma 4 모델을 실행하는 예시는 다음과 같다.

```bash
litert-lm run \
  --from-huggingface-repo=litert-community/gemma-4-E2B-it-litert-lm \
  gemma-4-E2B-it.litertlm \
  --prompt="What is the capital of France?"
```

uv를 사용한 설치 방법도 제공된다.

```bash
uv tool install litert-lm
litert-lm run --from-huggingface-repo=google/gemma-3n-E2B-it-litert-lm gemma-3n-E2B-it-int4
```

코드 없이도 CLI 도구만으로 즉시 모델을 테스트할 수 있는 점이 진입 장벽을 낮춘다.

## 아키텍처와 기술 스택

프로젝트는 모듈식 구조를 채택하여 핵심 엔진과 언어별 바인딩을 분리한다.

| 디렉터리 | 역할 |
|---------|------|
| runtime | 핵심 추론 엔진 |
| c, python, kotlin | 언어별 API 바인딩 |
| tools/test | 테스트 스위트 |
| schema | 모델 포맷 정의 |

기술 스택 측면에서는 C++가 76.5%로 주요 언어이며, CMake 6.7%, Python 5.4%, Starlark 4.9%, Rust 3.9%, Kotlin 1.8% 순으로 구성된다.
빌드 시스템은 Bazel을 사용하며, 의존성으로는 TensorFlow, SentencePiece, 토크나이저 등이 포함된다.
라이선스는 Apache 2.0으로, 상업적 이용이 가능하다.
최신 버전은 v0.10.1로 Gemma 4 지원과 CLI 도구를 통해 개발자 경험을 강화했다.

GeekNews 커뮤니티에서는 메모리 부족 문제와 Mac 환경에서의 성능 개선 여부가 논의되었다.
엣지 디바이스 추론 엔진에서 흔히 발생하는 메모리 압박을 어떻게 해결하는지가 실무 도입의 관건으로 보인다.

## 결론

LiteRT-LM은 Google이 내부 제품에서 검증한 추론 인프라를 오픈소스로 공개한 프로젝트다.
크로스 플랫폼 지원, 하드웨어 가속, 멀티모달, Function Calling을 하나의 프레임워크로 통합하여 온디바이스 AI 개발의 진입 장벽을 크게 낮췄다.
Gemma, Llama, Phi-4, Qwen 등 주요 오픈소스 LLM을 모두 지원하는 범용성과, Chrome이나 Pixel Watch 같은 실제 제품에서의 운영 검증은 이 프레임워크의 신뢰성을 뒷받침한다.
온디바이스 AI 배포를 고려하는 팀에게 Apache 2.0 라이선스와 Bazel 기반 빌드 시스템은 도입을 현실적인 선택지로 만든다.

## Reference

- [LiteRT-LM GitHub Repository](https://github.com/google-ai-edge/LiteRT-LM)
