---
layout: post
title: Claude Code로 LLM 파인튜닝하기 - HuggingFace Skills
author: 'Juho'
date: 2025-12-15 00:00:00 +0900
categories: [HuggingFace]
tags: [HuggingFace, Claude Code, Fine-tuning, LLM, SFT, DPO, GRPO, LoRA]
pin: false
toc: true
---

## 목차

1. [개요](#개요)
2. [HuggingFace Skills란?](#huggingface-skills란)
3. [주요 기능](#주요-기능)
4. [설치 및 설정](#설치-및-설정)
5. [첫 번째 훈련 실행](#첫-번째-훈련-실행)
6. [훈련 방법론](#훈련-방법론)
7. [하드웨어 및 비용 가이드](#하드웨어-및-비용-가이드)
8. [데이터셋 검증](#데이터셋-검증)
9. [훈련 모니터링](#훈련-모니터링)
10. [GGUF 변환으로 로컬 실행](#gguf-변환으로-로컬-실행)
11. [핵심 기능 요약](#핵심-기능-요약)
12. [성공을 위한 팁](#성공을-위한-팁)
13. [참고 자료](#참고-자료)

## 개요

HuggingFace가 획기적인 도구를 출시했습니다.  
Claude Code와 같은 AI 코딩 에이전트를 사용하여 단순한 자연어 명령만으로 오픈소스 LLM을 파인튜닝할 수 있게 되었습니다.  
이 글에서는 HuggingFace Skills를 활용하여 코드 한 줄 작성하지 않고도 LLM 파인튜닝을 수행하는 방법을 알아보겠습니다.  
Claude Code, Codex, Gemini CLI 같은 코딩 에이전트가 GPU 선택, 스크립트 생성, 작업 제출, 모니터링, 그리고 모델 배포까지 모든 과정을 자동으로 처리합니다.  

## HuggingFace Skills란?

HuggingFace Skills는 AI 에이전트가 복잡한 머신러닝 작업을 수행할 수 있도록 도와주는 도구입니다.  
특히 LLM 파인튜닝 과정을 완전히 자동화하여, 다음과 같은 작업을 에이전트가 대신 처리합니다:

- 데이터셋 형식 검증
- 적절한 GPU 하드웨어 자동 선택
- 훈련 스크립트 구성 및 모니터링 설정
- HuggingFace GPU 인프라에 작업 제출
- 실시간 진행 상황 추적
- 완성된 모델을 Hub에 자동 배포

## 주요 기능

### 1. 완전 자동화된 파인튜닝
자연어 명령만으로 전체 파인튜닝 파이프라인을 실행할 수 있습니다.   
복잡한 설정이나 코드 작성 없이도 프로덕션급 모델을 훈련할 수 있습니다.  

### 2. 지능형 GPU 선택
모델 크기와 데이터셋에 따라 최적의 GPU 하드웨어를 자동으로 선택합니다.  
비용과 성능을 균형있게 고려합니다.  

### 3. 실시간 모니터링
Trackio 대시보드를 통해 훈련 손실, 학습률, 검증 메트릭을 실시간으로 추적할 수 있습니다.  

### 4. 비용 투명성
작업 제출 전에 예상 시간과 비용을 미리 확인할 수 있어 예산 관리가 용이합니다.  

### 5. Hub 자동 배포
훈련이 완료되면 자동으로 HuggingFace Hub에 모델을 업로드하여 즉시 사용 가능합니다.  

## 설치 및 설정

### Claude Code 설치

```bash
/plugin marketplace add huggingface/skills
/plugin install hf-llm-trainer@huggingface-skills
```

### Codex 설치

Codex는 `AGENTS.md` 파일을 통해 자동으로 Skills를 로드합니다.  

확인 방법:
```bash
codex --ask-for-approval never "Summarize the current instructions."
```

### Gemini CLI 설치

```bash
gemini extensions install https://github.com/huggingface/skills.git --consent
```

### HuggingFace 인증

HuggingFace에 로그인하여 API 토큰을 설정해야 합니다:

```bash
hf auth login
```

또는 환경 변수로 설정:

```bash
export HF_TOKEN=hf_your_write_access_token_here
```

**중요**: HuggingFace Jobs 기능은 Pro 또는 Team/Enterprise 플랜이 필요합니다.  

## 첫 번째 훈련 실행

### 명령 예시

Claude Code에게 다음과 같이 간단히 요청하면 됩니다:

```
Fine-tune Qwen3-0.6B on the open-r1/codeforces-cots dataset
for instruction following.
```

### 에이전트의 응답

에이전트는 자동으로 설정을 분석하고 다음과 같은 정보를 제공합니다:

```
Configuration:
- Hardware: t4-small (~$0.75/hour)
- Estimated time: ~20 minutes
- Estimated cost: ~$0.30
- Output: username/qwen-codeforces-cots-sft

The model will be pushed to Hub automatically. Should I submit?
```

### 작업 제출 결과

승인하면 작업이 제출되고 다음 정보를 받게 됩니다:

```
✅ Job submitted successfully!

Job ID: abc123xyz
Monitor: https://huggingface.co/jobs/username/abc123xyz

Expected time: ~20 minutes
Estimated cost: ~$0.30

View real-time metrics at: https://huggingface.co/spaces/username/trackio
```

### 훈련된 모델 사용

훈련이 완료되면 바로 모델을 사용할 수 있습니다:

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained("username/qwen-codeforces-cots-sft")
tokenizer = AutoTokenizer.from_pretrained("username/qwen-codeforces-cots-sft")
```

## 훈련 방법론

HuggingFace Skills는 세 가지 주요 훈련 방법을 지원합니다.  

### 1. Supervised Fine-Tuning (SFT)

**사용 시기**: 고품질의 입력/출력 예시 데이터가 있을 때

**적합한 사용 사례**:
- 고객 지원 대화
- 코드 입출력 쌍
- 도메인 특화 Q&A

**명령 예시**:
```
Fine-tune Qwen3-0.6B on my-org/support-conversations for 3 epochs.
```

### 2. Direct Preference Optimization (DPO)

**사용 시기**: 선호도 쌍 데이터가 있을 때 (선택된 응답 vs 거부된 응답)

**데이터 요구사항**: `chosen`과 `rejected` 컬럼이 필요합니다.

**특징**: 일반적으로 초기 SFT 단계 이후에 수행됩니다.

**명령 예시**:
```
Run DPO on my-org/preference-data to align the SFT model
```

### 3. Group Relative Policy Optimization (GRPO)

**사용 시기**: 검증 가능한 작업(수학, 코드)을 위한 강화학습

**작동 방식**: 모델이 응답을 생성하고 보상을 받습니다.

**명령 예시**:
```
Train a math reasoning model using GRPO on openai/gsm8k based on Qwen3-0.6B.
```

## 하드웨어 및 비용 가이드

| 모델 크기 | 권장 GPU | 훈련 비용 | 비고 |
|----------|----------|-----------|------|
| <1B | t4-small | $1-2 | 빠른 실험 실행 |
| 1-3B | t4-medium/a10g-small | $5-15 | 수 시간 훈련 |
| 3-7B | a10g-large/a100-large | $15-40 | LoRA 필요 |
| 7B+ | 권장하지 않음 | N/A | 이 스킬의 범위 초과 |

**LoRA 참고사항**: 3B 이상의 모델은 메모리를 절약하면서 품질을 유지하기 위해 자동으로 LoRA(Low-Rank Adaptation)를 사용합니다.  

## 데이터셋 검증

훈련 전에 데이터셋을 검증하여 형식 오류를 미리 확인할 수 있습니다:

```
Check if my-org/conversation-data works for SFT training.
```

에이전트는 데이터셋이 SFT, DPO, GRPO 중 어떤 방법과 호환되는지 알려줍니다.

### 데이터셋 형식 수정

컬럼 이름이 다른 경우 수정 방법을 물어볼 수 있습니다:

```
My DPO dataset uses 'good_response' and 'bad_response' instead
of 'chosen' and 'rejected'. How do I fix this?
```

## 훈련 모니터링

### 실시간 대시보드

Trackio 통합을 통해 실시간 메트릭을 제공합니다:

- URL: `https://huggingface.co/spaces/username/trackio`
- 추적 항목: 훈련 손실, 학습률, 검증 메트릭

### 상태 확인

훈련 중 언제든지 상태를 확인할 수 있습니다:

```
How's my training job doing?
```

응답 예시:
```
Job abc123xyz is running (45 minutes elapsed)

Current step: 850/1200
Training loss: 1.23 (↓ from 2.41 at start)
Learning rate: 1.2e-5

Estimated completion: ~20 minutes
```

## GGUF 변환으로 로컬 실행

훈련된 모델을 GGUF 형식으로 변환하면 llama.cpp, Ollama, LM Studio에서 로컬로 실행할 수 있습니다.  

### 변환 명령

```
Convert my fine-tuned model to GGUF with Q4_K_M quantization.
Push to username/my-model-gguf.
```

### 로컬 추론

```bash
llama-server -hf <username>/<model-name>:<quantization>

# 예시:
llama-server -hf unsloth/Qwen3-1.7B-GGUF:Q4_K_M
```

## 핵심 기능 요약

HuggingFace Skills의 핵심 장점:

- **자동 하드웨어 선택**: 모델 크기에 따라 최적 GPU 자동 선택
- **비용 투명성**: 사전 비용 예측 제공
- **비동기 훈련**: 터미널을 닫고 나중에 다시 확인 가능
- **실시간 모니터링**: Trackio 대시보드를 통한 실시간 추적
- **자동 Hub 배포**: 훈련된 모델 자동 업로드
- **LoRA 지원**: 대형 모델의 효율적인 훈련
- **다단계 파이프라인**: SFT → DPO → GRPO 순차 실행 가능

## 성공을 위한 팁

### 1. 항상 데모부터 시작

프로덕션 실행 전에 100개 샘플로 테스트하세요:

```
Fine-tune on first 100 examples of my-org/dataset for testing.
```

### 2. 데이터셋 검증

작업 제출 전 항상 데이터셋을 검증하세요:

```
Validate my-org/dataset for SFT compatibility.
```

### 3. 하드웨어 선택 검토

에이전트의 하드웨어 선택을 검토하고 필요시 조정하세요.

### 4. 실시간 모니터링

Trackio 대시보드로 훈련 진행 상황을 실시간으로 확인하세요.

### 5. 단계적 접근

필요에 따라 SFT로 시작한 후 DPO나 GRPO를 추가하세요.

## 참고 자료

- [HuggingFace Skills - 공식 블로그](https://huggingface.co/blog/hf-skills-training)
- [SKILL.md - 전체 문서](https://github.com/huggingface/skills/blob/main/hf-llm-trainer/skills/model-trainer/SKILL.md)
- [Training Methods Guide](https://github.com/huggingface/skills/blob/main/hf-llm-trainer/skills/model-trainer/references/training_methods.md)
- [Hardware Guide](https://github.com/huggingface/skills/blob/main/hf-llm-trainer/skills/model-trainer/references/hardware_guide.md)
- [TRL Documentation](https://huggingface.co/docs/trl)
- [HuggingFace Jobs](https://huggingface.co/docs/huggingface_hub/guides/jobs)
- [Trackio Monitoring](https://huggingface.co/docs/trackio)
