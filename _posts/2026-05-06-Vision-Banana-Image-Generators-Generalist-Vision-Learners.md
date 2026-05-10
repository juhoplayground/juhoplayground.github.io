---
layout: post
title: "Vision Banana, 이미지 생성 모델이 범용 비전 학습자가 된다"
author: 'Juho'
date: 2026-05-06 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Benchmark]
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
2. [배경과 선행 연구](#배경과-선행-연구)
   - [LLM과 비전 모델의 표현 학습 격차](#llm과-비전-모델의-표현-학습-격차)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [Instruction-tuning 접근](#instruction-tuning-접근)
   - [비전 태스크의 RGB 인코딩](#비전-태스크의-rgb-인코딩)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [2D semantic understanding](#2d-semantic-understanding)
   - [3D understanding (depth, normal)](#3d-understanding-depth-normal)
   - [참조 분할 (Referring Expression)](#참조-분할-referring-expression)
   - [생성 능력 보존](#생성-능력-보존)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"Image Generators are Generalist Vision Learners"는 이미지 생성 모델 사전학습이 LLM 사전학습과 유사한 역할을 해 비전 이해 태스크 전반에 강한 표현을 학습한다는 가설을 제시한 논문이다.
저자는 Valentin Gabeur, Shangbang Long, Songyou Peng, Paul Voigtlaender, Shuyang Sun, Yanan Bao, Karen Truong, Zhicheng Wang, Wenlei Zhou, Jonathan T. Barron, Kyle Genova, Nithish Kannen, Sherry Ben, Yandong Li, Mandy Guo, Suhas Yogin, Yiming Gu, Huizhong Chen, Oliver Wang, Saining Xie, Howard Zhou, Kaiming He, Thomas Funkhouser, Jean-Baptiste Alayrac, Radu Soricut를 포함한다.

논문 abstract의 핵심은 다음과 같다.
저자들은 Nano Banana Pro에 비전 태스크 데이터를 instruction-tuning해 Vision Banana를 만들었다.
비전 태스크 출력을 RGB 이미지로 parameterize해 perception을 image generation으로 재구성했다.
다양한 비전 태스크에서 SOTA를 달성하면서 생성 능력도 유지한다.

## 배경과 선행 연구

### LLM과 비전 모델의 표현 학습 격차

LLM은 next-token prediction이라는 단일 generative pretraining으로 광범위한 downstream 능력을 끌어낸다.
저자들은 비전에서도 image generation pretraining이 비슷한 generalist representation을 만들 수 있는지 묻는다.
Nano Banana Pro 같은 SOTA generative 모델이 이미 내부에 비전 이해를 위한 표현을 갖고 있고, 이를 instruction-tuning만으로 unlock 할 수 있다는 가설이다.

### Related Work 정리

| 카테고리 | 주요 선행 연구 | 본 연구와의 차이 |
|----------|----------------|------------------|
| Specialist 비전 모델 | SAM 3, DINO-X, Depth Anything V3, Lotus-2, MoGe-2 | 단일 태스크 특화 — 본 연구는 단일 generative 모델로 통합 |
| 비전 backbone | CLIP, DINO 계열 | 표현 학습 — 본 연구는 generation 자체를 표현 학습으로 |
| Diffusion-based perception | Marigold, Lotus | depth만 — 본 연구는 generalist |
| Multimodal | LLaVA, Gemini | 이해 중심 — 본 연구는 generation을 통한 이해 |

## 방법론

### Instruction-tuning 접근

Vision Banana는 Nano Banana Pro의 학습 mixture에 비전 태스크 데이터를 낮은 비율로 추가하는 lightweight instruction-tuning으로 만들어진다.
이 방식은 generative prior를 보존하면서 출력을 measurable 형식으로 정렬한다.

semantic segmentation prompt 예시.

```
"Generate a semantic segmentation visualization image,
 using this color mapping..."
```

출력은 decode 가능한 visualization으로 양적 벤치마크 평가가 가능하다.

### 비전 태스크의 RGB 인코딩

Depth는 metric depth와 RGB 간 bijection을 power transform으로 정의한다.

```
f(d, λ, c) = 1 - (1 - d / (λ * c))^(λ + 1)
```

저자는 λ=-3, c=10/3을 사용하고 RGB cube interpolation으로 Hilbert-curve 유사 traversal을 적용한다.
이로써 inference 시 RGB 출력을 metric depth로 정확히 역변환할 수 있다.

surface normal은 카메라 공간 좌표를 직접 RGB로 매핑한다.

| 방향 | RGB 색 |
|------|----------|
| -X (Left) | Pinkish red |
| +Y (Up) | Light green |
| +Z (outward) | Light blue/purple |

이 단순한 결정적 인코딩이 generation 모델이 다룰 수 있는 형태로 비전 태스크 출력을 변환한다.

## 실험 셋업

| 항목 | 값 |
|------|-----|
| Base 모델 | Nano Banana Pro |
| 2D 데이터 | in-house 모델 annotation, web-crawled images |
| 3D 데이터 | rendering engine synthetic (실측 depth annotation 미사용) |
| 평가 데이터 | 평가 벤치마크 학습 데이터 0% (strict zero-shot) |
| 평가 태스크 | Cityscapes, SA-Co/Gold, RefCOCOg, ReasonSeg, NYU, ETH3D, KITTI 등 |
| 생성 평가 | GenAI-Bench (text-to-image), ImgEdit (이미지 편집) |

## 주요 결과

### 2D semantic understanding

| 태스크 | 데이터셋 | Vision Banana | 비교 baseline |
|--------|------------|------------------|------------------|
| Semantic Segmentation | Cityscapes | 0.699 mIoU | 0.652 (SAM 3) |
| Instance Segmentation | SA-Co/Gold (500 sample) | 0.540 pmF1 | 0.552 (DINO-X) |
| Referring Seg | RefCOCOg UMD | 0.738 cIoU | 0.734 (SAM3 Agent) |
| Reasoning Seg | ReasonSeg | 0.793 gIoU | 0.770 (SAM3 Agent) |

Cityscapes에서 SAM 3 대비 4.7 mIoU 포인트 우위가 가장 두드러진다.

### 3D understanding (depth, normal)

Metric depth (4 데이터셋 평균).

| 모델 | δ_1 |
|------|-------|
| Vision Banana | 0.929 |
| Depth Anything V3 | 0.918 |

개별 데이터셋 결과.

| 데이터셋 | Vision Banana δ_1 | 비교 |
|----------|----------------------|--------|
| NYU | 0.948 | 0.961 (경쟁) |
| ETH3D | 0.935 | 0.908 (MoGe-2) |
| KITTI | 0.915 | 0.953 (Depth Pro) |

Surface Normal 평균 angular error.

| 모델 | mean error |
|------|-------------|
| Vision Banana | 18.928° |
| Lotus-2 | 19.642° |

Vision Banana가 indoor 데이터셋에서 mean과 median angular error 모두 가장 낮다.

### 참조 분할 (Referring Expression)

저자들은 free-form 텍스트 conditioning에 대해 명시적 학습이 없었음에도 ReasonSeg에서 0.793 gIoU로 SAM3 Agent(0.770)를 상회한다.
이는 generative pretraining이 cross-task transfer를 강하게 유도한다는 증거로 해석된다.

### 생성 능력 보존

instruction-tuning이 비전 이해 태스크를 추가했음에도 생성 품질이 유지된다.

| 평가 | Vision Banana 승률 |
|------|----------------------|
| GenAI-Bench (text-to-image) | 53.5% (vs Nano Banana Pro) |
| ImgEdit (image editing) | 47.8% (vs Nano Banana Pro) |

생성 능력의 회귀가 거의 없거나 오히려 약간 개선되어 instruction-tuning의 강점을 보였다.

## 한계와 디스커션

저자가 본문에서 인정하는 한계와 향후 방향은 다음과 같다.
첫째, lightweight specialist 모델 대비 계산 오버헤드가 크다.
둘째, instance segmentation은 SA-Co/Gold에서 SAM3에 다소 뒤졌다.
셋째, 본 연구는 단일 이미지 입력에 한정되며 multi-view, video, 시간 추론 확장은 향후 과제다.
넷째, 실용 배포를 위해서는 추론 가속 전략이 필요하다.

디스커션의 핵심 함의는 두 가지다.
첫째, image generation pretraining이 generalist vision learner라는 결론은 vision 분야에 NLP의 generative foundation 흐름과 유사한 패러다임 전환을 시사한다.
둘째, image generation을 통합 인터페이스로 사용하면 자연어 instruction으로 임의 비전 태스크를 표현할 수 있어 LLM의 인터페이스 통일과 유사한 효과를 낸다.

## 결론

Vision Banana는 Nano Banana Pro의 generative prior 위에 lightweight instruction-tuning을 더해 단일 모델로 2D segmentation, 3D depth, surface normal, 참조 분할을 통합 처리한다.
Cityscapes 0.699 mIoU(SAM 3 대비 +4.7), depth δ_1 평균 0.929, surface normal 18.928° mean error 같은 SOTA 결과와 함께 generation 능력도 유지(GenAI-Bench 53.5% 승률)했다.
"image generation pretraining이 비전 이해의 foundation이 될 수 있다"는 가설을 정량적으로 입증해 generative-first vision의 가능성을 제시한 연구다.

## Reference

- [Image Generators are Generalist Vision Learners (arXiv:2604.20329)](https://arxiv.org/abs/2604.20329/)
