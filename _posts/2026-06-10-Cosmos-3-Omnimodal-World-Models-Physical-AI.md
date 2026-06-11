---
layout: post
title: "Cosmos 3: Physical AI를 위한 옴니모달 월드 모델"
author: 'Juho'
date: 2026-06-10 00:00:00 +0900
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
2. [방법론(아키텍처)](#방법론아키텍처)
3. [주요 결과](#주요-결과)
4. [한계와 주의사항](#한계와-주의사항)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Cosmos 3는 NVIDIA가 2026년 6월 3일 공개한 옴니모달 월드 모델 패밀리다.
언어, 이미지, 비디오, 오디오, 액션 시퀀스를 단일 mixture-of-transformers(MoT) 아키텍처에서 함께 처리하고 생성한다.
유연한 입출력 구성을 지원해 Physical AI에 필요한 핵심 모달리티를 하나의 프레임워크로 통합한다.
즉 비전-언어 모델, 비디오 생성기, 월드 시뮬레이터, 월드-액션 모델을 동일한 프레임워크 안에 포섭한다.
다양한 이해와 생성 작업에서 SOTA를 확립했다.

포스트 학습된 Cosmos 3는 작성 시점 기준으로 Artificial Analysis가 선정한 최고 오픈소스 Text-to-Image 및 Image-to-Video 모델로 뽑혔다.
또한 RoboArena 최고 정책 모델로도 선정되었다.
코드, 체크포인트, 합성 데이터셋, 벤치마크가 OpenMDW-1.1 라이선스로 공개되었다.

Physical AI 에이전트는 지각, 추론, 행동으로 실세계와 상호작용한다.
실세계에서 직접 학습하는 방식은 느리고 비싸며 위험하다.
따라서 두 가지 결합된 능력이 필요하다.
하나는 understanding으로, 부분 관찰에서 잠재 표현과 의미, 동역학을 추론하는 능력이다.
다른 하나는 generation으로, 그럴듯한 미래를 예측하고 시뮬레이션하는 능력이다.

기존 패러다임은 이 둘을 분리했다.
이해는 VLM이, 생성은 Video Generation 또는 Forward Dynamics Model이, 행동은 VLA 또는 WAM이 담당하는 식이다.
그러나 이해는 미래와 행동 결과 추론을 요구하고, 생성은 세계와 행동의 구조화된 표현에 의존한다.
그래서 단일 프레임워크 통합이 필수적이다.
예를 들어 식탁을 치우는 가정용 로봇은 현재 VLM, VLA 또는 WAM, World Model을 짜깁기해야 한다.

Cosmos 3는 입출력 구성에 따라 아키텍처 변경 없이 역할을 전환한다.
VLM, text-to-image, 비디오 생성(text-to-video, image-to-video, video-to-video, 오디오-비디오 동기 생성), world-action model 등으로 전환할 수 있다.

## 방법론(아키텍처)

Cosmos 3는 액션을 핵심 모달리티로 취급해 전용 action token 클래스를 도입했다.
이 토큰은 물리 세계와 언어 추론, 비디오 월드 모델링을 잇고, 물리적으로 grounding된 제어 신호와 직결된다.

모달리티별 인코더가 입력을 통합 표현 공간에 투영한 뒤 MoT 백본이 처리한다.
추론 시 언어 토큰은 next-token prediction으로 생성한다.
그 외 모달리티는 반복적 denoising으로 생성한다.

인코더 구성은 다음과 같다.
시각 이해에는 vision-language 정렬로 사전학습한 ViT를 사용한다.
이 ViT는 16x16 패치, 2x2 토큰 병합 MLP, DeepStack 특징 집계를 사용하며, 비디오 프레임 사이에 텍스트 타임스탬프를 삽입한다.
시각 생성에는 Wan2.2-TI2V-5B의 video VAE를 사용하며, 시간 4배와 공간 32x32 압축을 수행한다.
ViT는 백본과 공동 학습하고, VAE는 동결한다.

MoT 백본은 Dual-Tower 레이어 구조와 Dual-Stream Joint Attention을 갖는다.
멀티모달 위치 임베딩은 위치 인덱스 할당과 절대 시간 변조를 사용한다.
모델 변형으로는 Cosmos3-Super와 Cosmos3-Nano가 있다.

Cosmos 3는 세 가지 방식으로 Physical AI 에이전트 학습을 가속한다.
첫째는 합성 데이터 생성이다.
둘째는 작업 및 임바디먼트 특화다.
셋째는 학습 환경 생성이다.

아키텍처 변경 없이 타깃 데이터로 포스트 학습이 가능하다.
예를 들어 DROID에서 robot policy로 포스트 학습할 수 있다.
SDG-PhyxSim, RobotSim, DriveSim, SynHuman, Warehouse 합성 데이터셋과 Cosmos-HumanEval(Cosmos-HUE) 벤치마크가 함께 공개되었다.

## 주요 결과

아래 결과는 Table 1의 개요로, Cosmos 3가 특화된 오픈소스 베이스라인을 전반적으로 능가함을 보여준다.
표기 중 별표는 포스트 학습 변형, 단검(†)은 클로즈드 모델을 의미한다.

Reasoning 결과는 General, Robotics, Smart infra., Driving 순서다.
Cosmos3-Super는 73.7 / 57.8 / 62.6 / 79.3을 기록했다.
Cosmos3-Nano는 69.6 / 55.1 / 61.0 / 76.0을 기록했다.
비교 모델인 Gemini 3.1 Pro는 77.5 / 58.2 / - / 58.6, Qwen3-VL-32B는 72.8 / 52.6 / 56.1 / 40.7, Gemma-4-31B는 69.8 / 51.0 / 51.3 / 36.6이다.

Reasoning 점수

| 모델 | General | Robotics | Smart infra. | Driving |
| --- | --- | --- | --- | --- |
| Cosmos3-Super | 73.7 | 57.8 | 62.6 | 79.3 |
| Cosmos3-Nano | 69.6 | 55.1 | 61.0 | 76.0 |
| Gemini 3.1 Pro (†) | 77.5 | 58.2 | - | 58.6 |
| Qwen3-VL-32B | 72.8 | 52.6 | 56.1 | 40.7 |
| Gemma-4-31B | 69.8 | 51.0 | 51.3 | 36.6 |

생성 작업에서도 Cosmos 3는 강세를 보였다.
Text2Image에서 Cosmos3-Super는 91.36(포스트 학습)으로 Gemini 3 Pro Image의 90.85를 앞섰고, Cosmos3-Nano는 84.61로 Qwen-Image-2512의 84.25를 앞섰다.
Text2Video에서 Cosmos3-Super는 80.0, Cosmos3-Nano는 79.4로 Veo-3.1의 79.1과 Wan2.2-A14B의 78.0을 능가했다.
Image2Video에서 Cosmos3-Super는 82.8, Cosmos3-Nano는 82.7로 Veo-3.1의 82.6과 Wan2.2-A14B의 81.3을 앞섰다.
Audio에서는 Cosmos3-Super 7.31, Cosmos3-Nano 7.34로 Veo-3.1의 7.45에 근접했다.

Generation 점수

| 모델 | Text2Image | Text2Video | Image2Video | Audio |
| --- | --- | --- | --- | --- |
| Cosmos3-Super | 91.36 (*) | 80.0 | 82.8 | 7.31 |
| Cosmos3-Nano | 84.61 | 79.4 | 82.7 | 7.34 |
| Veo-3.1 (†) | - | 79.1 | 82.6 | 7.45 |
| Gemini 3 Pro Image (†) | 90.85 | - | - | - |
| Qwen-Image-2512 | 84.25 | - | - | - |
| Wan2.2-A14B | - | 78.0 | 81.3 | - |

월드-액션 측면에서도 우위를 보였다.
Forward Dynamics(FD) 로봇 평가에서 Cosmos3-Super는 26.0, Cosmos3-Nano는 25.5(둘 다 포스트 학습)로 Ctrl-World의 23.0을 능가했다.
Policy 로봇 평가에서 Cosmos3-Nano는 39.7(포스트 학습)로 π0.5의 28.1을 크게 앞섰다.

World-Action 점수

| 모델 | FD: Robot | Policy: Robot |
| --- | --- | --- |
| Cosmos3-Super | 26.0 (*) | - |
| Cosmos3-Nano | 25.5 (*) | 39.7 (*) |
| Ctrl-World | 23.0 | - |
| pi0.5 | - | 28.1 |

## 한계와 주의사항

위 결과는 기술 보고서 작성 시점 기준 수치다.
합성 데이터와 환경의 스케일링은 Physical AI의 지속적 병목으로 남아 있다.
Cosmos 3는 이해와 생성을 단일 옴니모달 백본으로 통합해, 공유 표현과 공동 다중작업 감독을 통한 확장 가능한 학습을 지향한다.

## 결론

Cosmos 3는 옴니모달 월드 모델이 임바디드 에이전트를 위한 확장 가능하고 범용적인 백본으로 작동함을 입증했다.
지각, 시뮬레이션, 실행을 아키텍처 변경 없이 단일 프레임워크로 통합한다.
코드와 체크포인트, 합성 데이터셋, 벤치마크가 OpenMDW-1.1 라이선스로 공개되어 후속 연구의 기반이 된다.

## Reference

- [Cosmos 3: Omnimodal World Models for Physical AI](https://arxiv.org/abs/2606.02800/)
