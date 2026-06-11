---
layout: post
title: "OpenCV 5: DNN 엔진을 다시 쓴 20년 만의 대규모 현대화"
author: 'Juho'
date: 2026-06-11 00:00:00 +0900
categories: [Dev]
tags: [Dev, AI]
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
3. [DNN 엔진 재작성](#dnn-엔진-재작성)
4. [언어·API 현대화](#언어api-현대화)
5. [성능 벤치마크](#성능-벤치마크)
6. [새 역량](#새-역량)
7. [3D 비전 재구성](#3d-비전-재구성)
8. [하드웨어 가속](#하드웨어-가속)
9. [의미와 시사점](#의미와-시사점)
10. [결론](#결론)
11. [Reference](#reference)

## 개요

OpenCV 5가 2026년 6월에 출시됐다.
20여 년 만에 이뤄진 가장 큰 규모의 현대화다.
pip 버전은 2026년 6월 8일부터 가용하며, 덴버에서 열리는 CVPR 2026과 맞물려 공개됐다.
이번 릴리스의 핵심은 딥러닝 엔진의 완전한 재작성과 언어·API 현대화, 그리고 3D 비전·하드웨어 가속 전반의 재구성이다.

## 배경

OpenCV는 컴퓨터 비전 분야에서 가장 널리 쓰이는 라이브러리 중 하나다.
GitHub 스타가 86,000개를 넘고, 하루 100만 회 이상 설치된다.
OpenCV.org(비영리)를 비롯해 Big Vision, OpenCV China, OpenCV.ai가 프로젝트를 지원한다.
코드는 [github.com/opencv/opencv/tree/5.x](https://github.com/opencv/opencv/tree/5.x){:target="_blank"}에서 확인할 수 있다.

## DNN 엔진 재작성

OpenCV 5는 딥러닝 엔진을 완전히 재구축했다.
ONNX 연산자 지원이 약 22%에서 80% 이상으로 도약해, 최신 모델 로딩의 한계를 해소했다.
새 엔진은 typed operation graph를 기반으로 shape inference, constant folding, operator fusion을 지원한다.
이전 버전에는 없던 기능들이다.

엔진의 핵심 기능은 다음과 같다.
If·Loop 서브그래프로 control flow를 처리한다.
심볼릭·동적 shape를 지원한다.
FlashAttention 스타일의 attention fusion을 제공한다.
공격적 메모리 재사용을 위한 통합 버퍼 풀을 갖췄다.

엔진은 멀티 엔진 아키텍처로 설계됐다.
단일 API 뒤에 3개의 엔진이 동작한다.

멀티 엔진 구성

| 엔진 | 설명 |
| --- | --- |
| ENGINE_CLASSIC | 기존 4.x 엔진, CUDA·OpenVINO 등 비-CPU 백엔드 |
| ENGINE_NEW | fusion 포함 그래프 기반, 현재 CPU 전용 |
| ENGINE_AUTO | 기본값, 새 엔진 우선 시도 후 클래식 폴백 |
| ENGINE_ORT | 선택적 ONNX Runtime 래퍼 |

## 언어·API 현대화

Python 측면에서는 NumPy 2.x를 지원한다.
알고리즘 파라미터에 named/keyword 인자를 쓸 수 있고, 깊은 NumPy 통합을 제공한다.

C++ 측면에서는 C++17이 최소 권장 표준이다.
이후 5.x 버전에 C++20 모듈이 예정돼 있고, 레거시 C API는 공식적으로 deprecated 됐다.

데이터 타입도 확장됐다.
FP16·BF16·bool·64비트 정수를 네이티브로 지원해 최신 AI 워크로드에 대응한다.
0D(스칼라)·1D 배열을 제대로 지원해 어색한 reshape를 제거했다.

## 성능 벤치마크

Intel Core i9-14900KS 환경에서 ONNX Runtime 대비 성능을 측정했다.
탐지·세그멘테이션·대형 비전 모델 전반에서 일관된 경쟁력을 보였다.

ONNX Runtime 대비 속도 향상

| 모델 | 속도 향상 |
| --- | --- |
| XFeat | 31.25% 빠름 |
| OWLv2 | 36.6% 빠름 |
| BiRefNet | 32.4% 빠름 |
| YOLOv8n | 11.5% 빠름 |

## 새 역량

LLM/VLM 지원이 추가됐다.
내장 tokenizer와 KV-cache 기반 autoregressive 디코딩으로 언어·비전-언어 모델을 직접 실행한다.
지원 모델은 Qwen 2.5, Gemma 3, PaliGemma, GPT 계열 변형이다.

고급 기능도 늘었다.
LaMa inpainting으로 마스크를 활용해 객체를 제거할 수 있다.
ALIKED·DISK 기반 딥러닝 특징 검출을 제공한다.
LightGlueMatcher로 학습 기반 특징 매칭이 가능하며, 기존 stitching 모듈과 통합된다.

## 3D 비전 재구성

모놀리식이던 calib3d를 세 개의 모듈로 분리했다.

3D 비전 모듈 분리

| 모듈 | 역할 |
| --- | --- |
| 3d | 기본 기하·프리미티브·ICP·SLAM 요소 |
| calib | 단일·다중 카메라 캘리브레이션, hand-eye/robot-world |
| stereo | 스테레오 깊이 |

point cloud/mesh I/O, TSDF 볼류메트릭 융합, robust USAC 추정 프레임워크를 포함한다.

## 하드웨어 가속

재설계된 HAL로 벤더가 최적화 커널을 투명하게 플러그인할 수 있다.

HAL 벤더 지원

| 벤더 | 대상 |
| --- | --- |
| Intel IPP/IPPICV | x86/x64 |
| Arm KleidiCV | AArch64 NEON/SVE/SME |
| Qualcomm FastCV | Snapdragon Hexagon DSP/NPU |
| RISC-V Vector | RVV |

Universal Intrinsics 2.0이 SSE·AVX2/512·NEON·SVE·RVV 전반의 벡터 코드를 통합한다.
ARM 연산에서 3~4배 속도 향상을 제공한다.

## 의미와 시사점

문서 체계는 순수 Doxygen에서 Sphinx+Doxygen 파이프라인으로 이전됐다.
향후 로드맵에는 새 DNN 엔진의 네이티브 GPU 지원이 포함된다.
현재 CPU 전용인 한계를 넘어서는 방향이다.
가속기 전·후처리를 위한 비-CPU HAL도 예정돼 있다.

ONNX 연산자 지원이 22%에서 80% 이상으로 확대되면서, 최신 모델을 별도 변환 없이 곧바로 로딩하기 쉬워졌다.
Intel Core i9-14900KS 기준 ONNX Runtime 대비 XFeat 31.25%, OWLv2 36.6%, BiRefNet 32.4%, YOLOv8n 11.5%의 속도 향상은, 새 그래프 기반 엔진이 실사용 수준의 경쟁력을 갖췄음을 보여준다.

## 결론

OpenCV 5는 DNN 엔진 재작성, 언어·API 현대화, 3D 비전 모듈 분리, 하드웨어 가속 재설계를 한꺼번에 담은 대규모 현대화 릴리스다.
LLM/VLM 직접 실행과 새 특징 검출·매칭 기능까지 더해지면서, 전통적인 컴퓨터 비전 라이브러리의 범위를 최신 AI 워크로드로 확장했다.

## Reference

- [OpenCV 5](https://opencv.org/opencv-5/)
