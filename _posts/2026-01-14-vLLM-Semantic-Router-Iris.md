---
layout: post
title: vLLM Semantic Router v0.1 Iris - MoM을 위한 시스템 레벨 라우터
author: 'Juho'
date: 2026-01-14 09:00:00 +0900
categories: [vLLM]
tags: [LLM, Semantic Router, MoM, HaluGate]
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
1. [vLLM Semantic Router 개요](#vllm-semantic-router-개요)
2. [핵심 아키텍처](#핵심-아키텍처)
3. [시스템 요구사항](#시스템-요구사항)
4. [설치 및 시작하기](#설치-및-시작하기)
5. [주요 기능](#주요-기능)
6. [HaluGate 환각 감지](#halugate-환각-감지)
7. [MoM 모델 패밀리](#mom-모델-패밀리)
8. [고급 설정](#고급-설정)
9. [커뮤니티 현황](#커뮤니티-현황)
10. [참고 자료](#참고-자료)

## vLLM Semantic Router 개요

vLLM Semantic Router v0.1(코드명: Iris)은 **Mixture-of-Models(MoM)을 위한 시스템 레벨 인텔리전트 라우터**입니다. 사용자와 AI 모델 사이의 중간 계층으로 동작하며, 요청 컨텍스트와 응답 신호를 기반으로 지능적인 라우팅 결정을 내립니다.

### 주요 특징

- **GPU 불필요**: CPU에서 추론 작업을 수행하므로 GPU 리소스가 필요하지 않습니다
- **시스템 레벨 라우터**: 여러 모델 간의 효율적인 트래픽 분산 및 라우팅
- **신호 기반 의사결정**: 6가지 신호 유형을 활용한 지능적 라우팅
- **쉬운 배포**: 단일 pip 명령으로 설치 가능

## 핵심 아키텍처

vLLM Semantic Router는 **Signal-Decision Driven Plugin Chain Architecture**를 기반으로 구축되었습니다. 이 아키텍처는 6가지 신호 유형을 추출하여 라우팅 결정에 활용합니다:

### 6가지 신호 유형

| 신호 유형 | 설명 | 기술 |
|---------|------|------|
| **Domain Signals** | 도메인 분류 | MMLU 학습 기반, LoRA 확장 가능 |
| **Keyword Signals** | 키워드 패턴 매칭 | Regex 기반 빠른 해석 |
| **Embedding Signals** | 의미론적 유사도 | 신경망 임베딩 기반 |
| **Factual Signals** | 환각 감지 | 분류 기반 환각 탐지 |
| **Feedback Signals** | 사용자 만족도 | 피드백 지표 |
| **Preference Signals** | 개인화 | 사용자 선호도 기반 |

### LoRA 기반 최적화

vLLM Semantic Router는 LoRA(Low-Rank Adaptation) 기반의 모듈식 아키텍처를 사용하여 계산 오버헤드를 크게 줄입니다:

- 기존: **O(n)** (각 분류 작업마다 별도 모델 실행)
- 최적화: **O(1) + O(n×ε)** (베이스 모델 공유 + 작은 LoRA 어댑터)

이를 통해 여러 분류 작업을 동시에 수행하면서도 메모리 사용량과 추론 시간을 최소화합니다.

## 시스템 요구사항

vLLM Semantic Router를 실행하기 위한 요구사항은 다음과 같습니다:

- **Python**: 3.10 이상
- **컨테이너 런타임**: Docker 또는 Podman (자동 감지)
- **선택사항**: HuggingFace 토큰 (gated 모델 사용 시)
- **GPU**: 필요 없음 (CPU에서 효율적으로 실행)

### Podman 설정

Podman을 사용하는 경우 최소 8GB 메모리 할당이 필요합니다:

```bash
podman machine init --memory 8192 --cpus 4 --disk-size 100
```

## 설치 및 시작하기

vLLM Semantic Router는 5단계만으로 간단하게 설치하고 실행할 수 있습니다.

### 1단계: 패키지 설치

```bash
python -m venv vsr
source vsr/bin/activate
pip install vllm-sr
vllm-sr --version
```

### 2단계: 설정 초기화

```bash
vllm-sr init
```

이 명령은 기본 설정 파일을 생성합니다.

### 3단계: 백엔드 설정 (config.yaml)

```yaml
providers:
  models:
    - name: "qwen/qwen3-1.8b"
      endpoints:
        - name: "my_vllm"
          weight: 1
          endpoint: "localhost:8000"
          protocol: "http"
      access_key: "optional-token"
  default_model: "qwen/qwen3-1.8b"
```

### 4단계: 라우터 시작

```bash
vllm-sr serve
```

이 명령은 자동으로:
- ML 모델 다운로드 (~1.5GB)
- 포트 8888에서 Envoy 프록시 시작
- 포트 9190에서 메트릭 활성화

### 5단계: 라우터 테스트

```bash
curl http://localhost:8888/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "MoM", "messages": [{"role": "user", "content": "Hello!"}]}'
```

### 유용한 명령어

```bash
# 로그 확인
vllm-sr logs router
vllm-sr logs envoy

# 실시간 로그 모니터링
vllm-sr logs router -f

# 상태 확인
vllm-sr status

# 서비스 중지
vllm-sr stop
```

## 주요 기능

vLLM Semantic Router는 다양한 플러그인 시스템을 통해 강력한 기능을 제공합니다.

### 플러그인 시스템

설정 가능한 모듈식 플러그인 아키텍처를 제공합니다:

- **Semantic Caching**: 유사한 요청에 대한 캐시 기반 응답
- **Jailbreak Detection**: 탈옥 시도 감지 및 차단
- **PII Protection**: 개인 식별 정보 보호
- **Hallucination Detection**: 환각 응답 감지 (HaluGate)
- **System Prompt Injection**: 시스템 프롬프트 주입 방지
- **HTTP Header Mutation**: HTTP 헤더 수정 및 관리

### 성능 최적화

- **LoRA 기반 아키텍처**: 베이스 모델 공유로 메모리 효율성 극대화
- **CPU 전용 실행**: GPU 없이도 빠른 추론 속도
- **분산 라우팅**: 여러 모델 엔드포인트 간 트래픽 분산

## HaluGate 환각 감지

HaluGate는 vLLM Semantic Router의 가장 강력한 기능 중 하나로, 3단계 환각 감지 파이프라인을 통해 LLM 응답의 신뢰성을 검증합니다.

### HaluGate 동작 원리

1. **토큰 레벨 분석**: 각 응답 토큰을 개별적으로 분석
2. **컨텍스트 검증**: 제공된 컨텍스트와 토큰 간의 관계 확인
3. **설명 생성**: 특정 토큰이 왜 환각으로 판단되었는지 설명

### 활용 사례

```python
# HaluGate를 통한 응답 검증
response = await router.chat(
    messages=[{"role": "user", "content": "What is the capital of France?"}],
    enable_halugate=True
)

if response.has_hallucination:
    print(f"환각 감지됨: {response.hallucination_tokens}")
    print(f"이유: {response.hallucination_explanation}")
```

HaluGate는 특히 다음과 같은 상황에서 유용합니다:

- RAG(Retrieval-Augmented Generation) 시스템에서 사실 관계 검증
- 의료, 법률 등 정확성이 중요한 도메인
- 사용자 신뢰도가 중요한 프로덕션 환경

## MoM 모델 패밀리

vLLM Semantic Router는 10개의 특화된 모델로 구성된 MoM(Mixture-of-Models) 패밀리를 제공합니다:

| 모델 | 용도 |
|------|------|
| Domain Classification | 도메인 분류 (과학, 비즈니스, 기술 등) |
| PII Detection | 개인정보 감지 및 마스킹 |
| Jailbreak Detection | 탈옥 시도 감지 |
| Hallucination Detection | 환각 응답 감지 |
| Semantic Similarity | 의미론적 유사도 계산 |
| Content Safety | 유해 콘텐츠 필터링 |
| Language Detection | 언어 감지 및 분류 |
| Intent Classification | 사용자 의도 분류 |
| Sentiment Analysis | 감성 분석 |
| Topic Modeling | 주제 모델링 |

각 모델은 특정 작업에 최적화되어 있으며, LoRA 어댑터를 통해 쉽게 확장하거나 커스터마이징할 수 있습니다.

## 고급 설정

### HuggingFace 설정

gated 모델이나 프라이빗 모델을 사용하는 경우:

```bash
export HF_ENDPOINT=https://huggingface.co
export HF_TOKEN=your_token_here
export HF_HOME=/path/to/cache
vllm-sr serve
```

### 커스텀 설정 파일 사용

```bash
vllm-sr serve --config my-config.yaml
```

### 커스텀 컨테이너 이미지 사용

```bash
vllm-sr serve --image ghcr.io/vllm-project/semantic-router/vllm-sr:latest
```

### Kubernetes 배포

프로덕션 환경에서는 Helm 차트를 사용하여 Kubernetes에 배포할 수 있습니다:

```bash
helm repo add vllm-sr https://vllm-project.github.io/semantic-router
helm install my-router vllm-sr/vllm-sr \
  --set config.defaultModel="qwen/qwen3-1.8b" \
  --set service.type=LoadBalancer
```

## 커뮤니티 현황

vLLM Semantic Router는 2025년 9월 실험적 출시 이후 급속도로 성장하고 있습니다:

- **600개 이상의 병합된 풀 리퀘스트**
- **300개 이상 해결된 이슈**
- **전 세계 50명 이상의 엔지니어 기여**

이는 활발한 오픈소스 커뮤니티와 프로덕션 환경에서의 실제 사용 사례가 증가하고 있음을 보여줍니다.

## 참고 자료

- [vLLM Semantic Router 공식 문서](https://vllm-semantic-router.com/docs/installation/){:target="_blank"}
- [vLLM SR Iris 공식 블로그 포스트](https://blog.vllm.ai/2026/01/05/vllm-sr-iris.html){:target="_blank"}
- [GitHub 저장소](https://github.com/vllm-project/semantic-router){:target="_blank"}
- [Hugging Face 모델 컬렉션](https://huggingface.co/LLM-Semantic-Router){:target="_blank"}
