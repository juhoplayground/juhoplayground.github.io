---
layout: post
title: "OpenPipe ART: 경험으로 학습하는 에이전트 강화학습 트레이너"
author: 'Juho'
date: 2026-06-18 00:00:00 +0900
categories: [LLM]
tags: [Agent, LLM, Evaluation]
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
2. [ART의 동작 방식](#art의-동작-방식)
   - [클라이언트와 서버 아키텍처](#클라이언트와-서버-아키텍처)
   - [W&B 서버리스 RL](#wb-서버리스-rl)
3. [RULER: 보상 함수 없는 강화학습](#ruler-보상-함수-없는-강화학습)
4. [설치와 사용 예시](#설치와-사용-예시)
5. [지원 모델과 활용 사례](#지원-모델과-활용-사례)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

ART(Agent Reinforcement Trainer)는 OpenPipe가 공개한 오픈소스 강화학습 프레임워크입니다.
언어 모델이 경험을 통해 스스로 개선되도록 만드는 것이 목표입니다.
ART는 GRPO 강화학습 알고리즘을 사용해 다단계(multi-step) 에이전트를 실제 작업으로 학습시킵니다.
Qwen, Llama 등 다양한 모델을 지원합니다.

핵심 가치는 "에이전트가 경험으로부터 학습하도록 해 신뢰성을 높인다"는 점입니다.
ART는 복잡한 인프라를 직접 관리하지 않고도 기존 파이썬 애플리케이션에 RL 학습을 통합할 수 있게 해줍니다.
라이선스는 Apache 2.0입니다.

## ART의 동작 방식

### 클라이언트와 서버 아키텍처

ART는 추론(inference)과 학습(training)을 분리한 클라이언트/서버 구조로 동작합니다.
추론 단계에서는 사용자 코드가 ART 클라이언트를 통해 에이전트 워크플로를 실행합니다.
이때 요청은 모델의 최신 LoRA를 vLLM에서 실행 중인 서버로 전달됩니다.
주고받은 메시지는 Trajectory(궤적)에 저장되며, 작업이 끝나면 각 궤적에 보상(reward) 값을 부여합니다.

학습 단계에서는 궤적들을 그룹으로 묶어 서버로 보냅니다.
GRPO가 최신 체크포인트에서 모델을 학습하고, 새로운 LoRA를 저장한 뒤 vLLM에 로드합니다.
이후 다시 추론을 재개하며, 이 루프가 지정한 반복 횟수만큼 이어집니다.

이 구조 덕분에 ART는 OpenAI 호환 클라이언트 인터페이스를 제공하면서도 학습 서버는 모듈화되고 추상화된 형태로 유지됩니다.
로컬 GPU에서도, 일시적인 클라우드 환경에서도 동작합니다.

### W&B 서버리스 RL

ART는 Weights & Biases와 연동한 서버리스 RL 옵션을 제공합니다.
다중화된(multiplexed) 추론 클러스터를 활용해 인프라를 완전 관리형으로 운영합니다.

다음은 W&B 서버리스 RL이 제시하는 주요 이점입니다.

| 항목 | 내용 |
|------|------|
| 비용 | 다중화 추론 클러스터로 40% 비용 절감 |
| 속도 | 2000개 이상 동시 요청으로 28% 빠른 학습 |
| 인프라 | 완전 관리형 인프라 |
| 배포 | 체크포인트 즉시 배포 |

관측성(observability) 측면에서는 W&B, Langfuse, OpenPipe와의 통합을 지원합니다.

## RULER: 보상 함수 없는 강화학습

RULER(Relative Universal LLM-Elicited Rewards)는 ART의 자동 보상 생성 시스템입니다.
강화학습에서 가장 번거로운 작업 중 하나인 보상 함수 엔지니어링(reward function engineering)을 제거하는 것이 목적입니다.
공식 문서는 이를 "RULER: Easy Mode for RL Rewards"라고 표현합니다.

RULER는 LLM을 심판(judge)으로 사용해 에이전트의 궤적을 평가합니다.
손으로 작성한 보상 함수를 요구하는 대신, 언어 모델이 다단계 에이전트 행동을 순위 매기고 점수화합니다.
즉, 궤적의 상대적 품질에 기반해 수치 보상을 부여합니다.

RULER의 주요 이점은 다음과 같습니다.

| 이점 | 설명 |
|------|------|
| 라벨 데이터 불필요 | 입력 생성과 평가를 자동으로 수행 |
| 손쉬운 일반화 | 전통적 보상 시스템의 드롭인 대체로 동작 |
| 빠른 개발 | 2-3배 빠른 개발 효율 |
| 강한 성능 | 여러 벤치마크에서 입증된 실증적 성능 |

라벨이 없는 데이터로도 학습이 가능하다는 점이 핵심 장점입니다.

## 설치와 사용 예시

설치는 pip로 간단히 진행합니다.

```bash
pip install openpipe-art
```

다음은 W&B 서버리스 백엔드를 사용하는 예시 코드입니다.

```python
from art.serverless.backend import ServerlessBackend

model = art.TrainableModel(
  project="voice-agent",
  name="agent-001",
  base_model="Qwen/Qwen3.6-27B"
)

backend = ServerlessBackend(api_key="your_wandb_api_key")
model.register(backend)
```

TrainableModel에 프로젝트명, 모델명, 베이스 모델을 지정하고 서버리스 백엔드에 등록하면 학습 준비가 완료됩니다.

## 지원 모델과 활용 사례

ART는 대부분의 vLLM/HuggingFace-transformers 호환 causal 언어 모델을 지원합니다.
특히 Unsloth가 지원하는 모델에 최적화되어 있습니다.
Qwen 2.5, Qwen 3.6, Llama 등이 포함됩니다.
다만 현재 Gemma 3은 지원되지 않습니다.

다음은 ART와 RULER 기반으로 학습된 대표적인 활용 사례입니다.

| 사례 | 모델 | 설명 |
|------|------|------|
| ART·E 이메일 에이전트 | Qwen 2.5 14B | 이메일 검색에서 OpenAI o3를 능가 |
| 2048 게임 | Qwen 3.6 27B | 2048 게임 플레이 학습 |
| Temporal Clue | Qwen 2.5 7B | 퍼즐 해결 |
| Tic Tac Toe | Qwen 2.5 3B | 틱택토 게임 플레이 |
| Codenames | Qwen 2.5 3B | 단어 게임 학습 |

가장 주목받는 사례는 ART·E 이메일 에이전트입니다.
Qwen 2.5 14B 모델이 이메일 검색 작업에서 OpenAI의 o3를 능가했으며, 학습 반복이 진행됨에 따라 측정 가능한 정확도 향상을 보였습니다.

## 결론

ART는 언어 모델이 실제 작업에서 경험을 쌓으며 개선되도록 하는 강화학습 프레임워크입니다.
클라이언트/서버 분리 구조로 기존 파이썬 애플리케이션에 RL을 통합하기 쉽게 만들었습니다.
RULER는 보상 함수 엔지니어링이라는 진입 장벽을 LLM 심판으로 대체해 라벨 없는 데이터로도 학습을 가능하게 합니다.
이메일 검색에서 소형 오픈 모델이 o3를 능가한 사례는 에이전트 강화학습의 실용적 가능성을 보여줍니다.

## Reference

- [OpenPipe ART (Agent Reinforcement Trainer)](https://github.com/OpenPipe/ART/)
