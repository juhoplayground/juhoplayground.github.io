---
layout: post
title: "NVIDIA AnyFlow - 추론 단계 수에 자유로운 14B 비디오 디퓨전 모델"
author: 'Juho'
date: 2026-05-20 00:00:00 +0900
categories: [AI]
tags: [AI, GPU, Pytorch, Benchmark]
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
2. [AnyFlow의 핵심 아이디어](#anyflow의-핵심-아이디어)
   - [Any-Step 생성](#any-step-생성)
   - [멀티 아키텍처와 멀티 태스크](#멀티-아키텍처와-멀티-태스크)
3. [모델 사양과 변형](#모델-사양과-변형)
4. [설치와 사용](#설치와-사용)
   - [환경 구성](#환경-구성)
   - [Text-to-Video 추론 예시](#text-to-video-추론-예시)
5. [한계와 라이선스](#한계와-라이선스)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

NVIDIA가 Hugging Face에 비디오 디퓨전 모델 AnyFlow를 공개했습니다.
첫 번째 any-step 비디오 디퓨전 모델로, 추론 예산을 4스텝부터 50스텝 이상까지 자유롭게 선택해도 품질이 매끄럽게 향상되는 것이 특징입니다.
이 글에서 다루는 모델은 Wan2.1-T2V-14B-Diffusers 백본 위에 구축된 14B 파라미터 양방향(bidirectional) 비디오 디퓨전 모델입니다.
하나의 모델로 다양한 추론 예산을 처리할 수 있어, 기존 distillation 모델이 고정된 스텝 수에 얽매이는 문제를 해결합니다.

## AnyFlow의 핵심 아이디어

AnyFlow는 flow map 기반의 새로운 distillation 프레임워크입니다.
한 번의 학습으로 다양한 추론 예산에 대응할 수 있도록 설계된 점이 핵심입니다.

### Any-Step 생성

전통적인 distilled 디퓨전 모델은 학습 시 정한 고정 스텝 수에서만 최적의 품질을 보입니다.
AnyFlow는 단일 모델이 임의의 추론 예산에 적응하며, 적은 스텝에서도 높은 품질을 유지하고 스텝이 늘어날수록 점진적으로 향상됩니다.
이는 on-policy flow map distillation 기법으로 학습된 결과로, 추론 시간을 자유롭게 조절할 수 있는 유연성을 제공합니다.

### 멀티 아키텍처와 멀티 태스크

AnyFlow는 causal과 bidirectional 두 가지 비디오 디퓨전 아키텍처를 모두 지원합니다.
지원 태스크는 Text-to-Video(T2V), Image-to-Video(I2V), Video-to-Video(V2V)입니다.
파라미터 규모는 1.3B에서 14B까지 검증됐으며, 이는 스케일링 가능성을 보여주는 결과입니다.

## 모델 사양과 변형

이 페이지에서 다루는 모델의 기본 사양은 다음과 같습니다.

| 항목 | 값 |
|------|------|
| 아키텍처 | Bidirectional video diffusion |
| 파라미터 | 14B |
| 해상도 | 480P (480x832) |
| 주요 태스크 | Text-to-Video |
| 백본 | Wan2.1-T2V-14B-Diffusers |
| 프레임워크 | Hugging Face Diffusers |
| 라이선스 | NVIDIA NSCLv1 (비상업용) |

AnyFlow 패밀리는 네 가지 변형으로 공개됐습니다.

| 모델 | 태스크 | 해상도 |
|------|--------|--------|
| AnyFlow-FAR-Wan2.1-1.3B-Diffusers | T2V, I2V, V2V | 480P |
| AnyFlow-FAR-Wan2.1-14B-Diffusers | T2V, I2V, V2V | 480P |
| AnyFlow-Wan2.1-T2V-14B-Diffusers | T2V | 480P |
| AnyFlow-Wan2.1-T2V-1.3B-Diffusers | T2V | 480P |

## 설치와 사용

AnyFlow는 Hugging Face Diffusers 생태계와 통합돼 있습니다.
NVIDIA가 제공하는 FAR 파이프라인을 통해 전체 기능을 활용할 수 있습니다.

### 환경 구성

기본 환경은 conda 기반으로 구성합니다.

```bash
conda create -n far python=3.10
conda activate far
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128
pip install -r requirements.txt --no-build-isolation
```

Diffusers 라이브러리만 빠르게 설치하려면 다음 명령을 사용합니다.

```bash
pip install -U diffusers transformers accelerate
```

모델 가중치는 Hugging Face CLI로 내려받습니다.

```bash
pip install "huggingface_hub[cli]"

hf download nvidia/AnyFlow-Wan2.1-T2V-14B-Diffusers \
    --repo-type model \
    --local-dir ./pretrained_models/
```

### Text-to-Video 추론 예시

AnyFlow 전용 파이프라인을 사용해 비디오를 생성하는 예시입니다.

```python
import torch
from diffusers.utils import export_to_video
from far.pipelines.pipeline_wan_anyflow import WanAnyFlowPipeline

model_id = "nvidia/AnyFlow-Wan2.1-T2V-14B-Diffusers"
pipeline = WanAnyFlowPipeline.from_pretrained(model_id).to('cuda', dtype=torch.bfloat16)

prompt = "CG game concept digital art, a majestic elephant with vibrant tusk and sleek fur running swiftly towards a herd."

video = pipeline(
    prompt=prompt,
    height=480,
    width=832,
    num_frames=81,
    num_inference_steps=4,
    generator=torch.Generator('cuda').manual_seed(0)
).frames[0]

export_to_video(video, "output.mp4", fps=16)
```

핵심은 `num_inference_steps`를 자유롭게 조절할 수 있다는 점입니다.
4스텝에서도 의미 있는 품질이 나오며, 스텝을 늘리면 안정적으로 품질이 향상됩니다.

기본 추론 파라미터는 다음과 같습니다.

| 파라미터 | 기본값 | 비고 |
|----------|--------|------|
| num_frames | 81 | 전체 비디오 프레임 수 |
| num_inference_steps | 4 이상 | 임의의 스텝 수 지원 |
| height | 480 | 출력 높이 |
| width | 832 | 출력 너비 |
| dtype | torch.bfloat16 | 효율성을 위한 권장 설정 |

## 한계와 라이선스

AnyFlow는 NVIDIA One-Way Noncommercial License(NSCLv1)로 배포돼 비상업적 용도로만 사용할 수 있습니다.
출력물의 소유권은 NVIDIA가 주장하지 않지만, 모델 사용 자체는 라이선스 조건을 따라야 합니다.
최대 해상도는 480P로 제한되며, 현 시점에서 inference provider는 배포되지 않았습니다.
최적 성능을 위해서는 CUDA GPU가 필요합니다.

학습 데이터의 상세 구성, 정량적 벤치마크, Wan2.1 대비 수치 비교는 공개된 문서에서 명시되지 않았습니다.
대신 "any-step에서도 품질이 안정적으로 향상된다"는 정성적 특성이 차별점으로 강조됩니다.

## 결론

AnyFlow는 비디오 디퓨전에서 "고정된 distillation 스텝"이라는 제약을 해체한 모델입니다.
14B 규모의 백본 위에서 4스텝부터 50스텝 이상까지 동일 모델로 추론할 수 있다는 점은, 서비스 운영 측면에서 추론 비용과 품질을 동적으로 조절할 수 있다는 가능성을 시사합니다.
FAR 파이프라인 기반의 멀티 태스크 변형까지 함께 공개돼 T2V/I2V/V2V를 단일 프레임워크로 다룰 수 있습니다.
다만 비상업 라이선스와 480P 해상도 한계는 상용 활용 전 검토가 필요한 부분입니다.

## Reference

- [nvidia/AnyFlow-Wan2.1-T2V-14B-Diffusers - Hugging Face](https://huggingface.co/nvidia/AnyFlow-Wan2.1-T2V-14B-Diffusers/)
