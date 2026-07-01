---
layout: post
title: "Wayfinder Router: 모델 호출 없이 결정론적으로 LLM을 라우팅하는 CLI"
author: 'Juho'
date: 2026-06-30 00:00:00 +0900
categories: [LLM]
tags: [LLM, Python, Agent]
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
2. [환경 설정](#환경-설정)
   - [설치](#설치)
   - [빠른 시작](#빠른-시작)
   - [설정 파일](#설정-파일)
3. [구현](#구현)
   - [프롬프트 점수 매기기](#프롬프트-점수-매기기)
   - [개별 요청 제어](#개별-요청-제어)
   - [보정과 피드백 루프](#보정과-피드백-루프)
   - [Python API](#python-api)
4. [주의사항](#주의사항)
   - [다른 라우터와의 비교](#다른-라우터와의-비교)
   - [배포와 보안](#배포와-보안)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Wayfinder Router는 로컬 및 클라우드 LLM 모델 간의 결정론적 라우팅을 위한 CLI 도구다.
모든 프롬프트에 대해 빠르고 오프라인으로 난이도를 판단하며, 모델 호출 없이 결정론적으로 점수를 매긴다.
간단한 요청은 저비용 모델로, 복잡한 요청은 고성능 모델로 라우팅한다.

이 도구가 내세우는 네 가지 장점은 다음과 같다.
첫째, 모델 호출이 필요 없는 의사결정이다.
둘째, 완전히 결정론적이고 오프라인으로 동작한다.
셋째, 사용자 데이터로 보정할 수 있다.
넷째, 자체 호스팅이 가능하다.

작동 원리는 간단하다.
프롬프트의 구조와 어휘를 분석하여 0.0부터 1.0 범위의 복잡도 점수를 생성한다.
구조 분석에는 길이, 제목, 목록, 코드가 포함되고, 어휘 분석에는 증명, 수학, 제약조건 같은 단서가 포함된다.
낮은 값은 로컬 모델로, 높은 값은 클라우드 모델로 자동 라우팅한다.

## 환경 설정

### 설치

Wayfinder Router는 pip으로 설치한다.
기본 설치 외에 게이트웨이와 UI를 포함하는 추가 옵션을 제공한다.

```bash
# 기본 설치
pip install wayfinder-router

# 게이트웨이 포함
pip install "wayfinder-router[gateway]"

# UI 포함 전체 설치
pip install "wayfinder-router[all]"
```

### 빠른 시작

먼저 설정을 생성한 뒤 게이트웨이를 시작한다.
게이트웨이는 OpenAI 호환 엔드포인트로 동작하므로 기존 클라이언트를 그대로 사용할 수 있다.

```bash
# 설정 생성
wayfinder-router init

# 게이트웨이 시작
export ANTHROPIC_API_KEY=sk-...
wayfinder-router serve --port 8088
```

클라이언트는 게이트웨이의 주소를 base_url로 지정하면 된다.

```python
import openai

client = openai.OpenAI(base_url="http://localhost:8088/v1", api_key="unused")
```

### 설정 파일

라우팅 기준값과 각 모델의 엔드포인트는 TOML 설정 파일에서 정의한다.
threshold 값을 기준으로 로컬 모델과 클라우드 모델 중 하나를 선택한다.

```toml
[routing]
threshold = 0.5

[gateway.models.local]
base_url = "http://localhost:11434/v1"
model = "llama3.2"

[gateway.models.cloud]
base_url = "https://api.openai.com/v1"
model = "gpt-4o"
api_key_env = "OPENAI_API_KEY"
```

## 구현

### 프롬프트 점수 매기기

route 명령으로 개별 프롬프트의 복잡도 점수를 확인할 수 있다.
표준 입력으로 프롬프트를 전달하면 라우팅 결과를 반환한다.

```bash
echo "Summarise this paragraph." | wayfinder-router route -
```

### 개별 요청 제어

라우터의 자동 판단을 요청 단위로 재정의할 수 있다.
강제 지정과 선호도 지정, 그리고 헤더를 통한 기준값 재설정을 지원한다.

| 방식 | 값 | 동작 |
|------|-----|------|
| 강제 지정 | local 또는 cloud | 해당 모델로 강제 라우팅 |
| 선호도 지정 | prefer-local 또는 prefer-hosted | 선호 모델 우선 선택 |
| 헤더 재설정 | X-Wayfinder-Threshold | 단일 요청의 기준값 재설정 |

### 보정과 피드백 루프

사용자 데이터로 라우팅 기준값을 보정할 수 있다.
보정, 온보딩, 자동 라벨링을 위한 명령이 준비되어 있다.

```bash
# 데이터 기반 기준값 보정
wayfinder-router calibrate data.jsonl --mode threshold

# 피드백 루프 온보딩
wayfinder-router onboard prompts.jsonl --arms local,cloud --calibrate

# 자동 라벨링
wayfinder-router judge prompts.jsonl --arms local,cloud --gold gold.jsonl
```

### Python API

CLI 외에 Python API로도 복잡도 점수를 계산할 수 있다.
score_complexity 함수에 프롬프트와 라우팅 설정을 전달하면 추천 모델과 점수를 반환한다.

```python
from wayfinder_router import score_complexity, RoutingConfig

result = score_complexity(prompt_text, config=RoutingConfig.binary(threshold=0.7))
print(result.recommendation, result.score)
```

Wayfinder Router는 OpenAI 호환 엔드포인트를 사용하는 다양한 제공자와 함께 동작한다.
OpenAI, Claude, Gemini, Mistral, Ollama, Groq, Together, OpenRouter, Fireworks, DeepSeek, vLLM, LM Studio, llama.cpp 등을 지원한다.

## 주의사항

### 다른 라우터와의 비교

Wayfinder Router는 학습된 분류기 대신 결정론적 구조 점수를 사용한다는 점에서 다른 라우터와 구분된다.
아래는 대표적인 라우터와의 비교다.

| 도구 | 방식 | 모델 호출 | 자체 호스팅 | 보정 |
|------|------|-----------|-------------|------|
| Wayfinder | 결정론적 구조 점수 | 아니오 | 예 | 예 |
| RouteLLM | 학습된 분류기 | 예 | 예 | 재학습 필요 |
| NotDiamond | 학습된 호스팅 | 예 | 아니오 | 플랫폼을 통해 |

성능 측면에서 라우팅 결정은 1밀리초 미만으로 이뤄지며 네트워크 호출이 없다.
동일한 입력에는 항상 동일한 결과를 반환한다.
저비용 모델로 처리할 수 있는 쉬운 프롬프트를 로컬에서 처리하므로 고비용 API 호출을 줄일 수 있다.

### 배포와 보안

로컬 서비스로 설치하거나 Docker로 배포할 수 있다.
웹 UI와 대시보드도 별도 포트로 실행할 수 있다.

```bash
# 로컬 서비스 설치와 상태 확인
wayfinder-router service install
wayfinder-router service status

# Docker 빌드와 실행
docker build -t wayfinder-router .
docker run -p 8088:8088 wayfinder-router

# 웹 UI 실행
wayfinder-router ui --port 8099
```

보안 측면에서 비밀키는 요청 시점에만 환경 변수에서 읽으며 설정 파일에 저장하지 않는다.
선택적으로 vault 통합을 지원하며 op read, macOS Keychain, Linux secret-tool을 사용할 수 있다.
비밀키는 메모리에만 유지된다.

## 결론

Wayfinder Router는 모델 호출 없이 결정론적으로 프롬프트 복잡도를 판단하여 로컬과 클라우드 모델 사이를 라우팅하는 CLI 도구다.
1밀리초 미만의 오프라인 의사결정과 동일 입력에 대한 동일 결과라는 특성을 통해 예측 가능한 라우팅을 제공한다.
사용자 데이터로 기준값을 보정할 수 있고, OpenAI 호환 엔드포인트를 사용하는 여러 제공자와 함께 자체 호스팅할 수 있다.
라이선스는 Apache 2.0이다.

## Reference

- [Wayfinder Router GitHub](https://github.com/itsthelore/wayfinder-router/)
