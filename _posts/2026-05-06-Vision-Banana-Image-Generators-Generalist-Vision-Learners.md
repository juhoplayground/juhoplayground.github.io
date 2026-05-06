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
2. [배경](#배경)
3. [핵심 내용](#핵심-내용)
   - [생성 사전학습 패러다임](#생성-사전학습-패러다임)
   - [2D 시각 이해 능력](#2d-시각-이해-능력)
   - [3D 시각 이해 능력](#3d-시각-이해-능력)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Google DeepMind가 [Vision Banana](https://vision-banana.github.io/#capabilities){:target="_blank"} 프로젝트와 함께 "Image Generators are Generalist Vision Learners"라는 기술 보고서를 공개했다.
이 연구는 이미지 생성 훈련이 LLM의 텍스트 사전학습과 유사한 역할을 하면서도 강력한 시각 표현을 학습할 수 있다고 주장한다.
Nano Banana Pro 위에 가벼운 지시튜닝을 더해 분할, 깊이 추정 같은 전형적 비전 태스크를 모두 RGB 이미지 생성으로 다시 정의한 것이 핵심이다.

## 배경

지금까지 시각 이해 모델은 태스크별로 별도의 헤드와 학습 절차를 사용해왔다.
의미론적 분할에는 분할 전용 헤드가, 깊이 추정에는 회귀 헤드가 필요했다.
반면 LLM은 텍스트 생성이라는 단일 인터페이스로 거의 모든 자연어 태스크를 흡수했다.
Vision Banana는 같은 일을 비전 영역에서 시도한다.
모델은 분할 마스크나 깊이 맵 같은 출력을 모두 RGB 이미지로 그리도록 학습된다.

## 핵심 내용

### 생성 사전학습 패러다임

Vision Banana는 Nano Banana Pro라는 이미지 생성 모델을 출발점으로 삼는다.
원본 학습 데이터에 소량의 비전 태스크 데이터를 섞어 지시튜닝을 수행한다.
지각 과제를 이미지 생성으로 매개변수화하는 것이 핵심 전략이다.
저자들은 이 접근이 "이미지 생성이 텍스트 생성처럼 통합 인터페이스" 역할을 할 수 있음을 보여준다고 평가한다.
가벼운 튜닝 후에도 기본 생성 능력이 유지된다는 점이 강조된다.

### 2D 시각 이해 능력

2D 영역에서 Vision Banana는 세 가지 분할 태스크를 수행한다.
의미론적 분할은 픽셀 단위로 클래스 라벨을 부여한다.
인스턴스 분할은 동일 클래스 객체도 개별 색상으로 분리한다.
참조 표현 분할은 자연어 설명에 해당하는 객체만 분할한다.

| 태스크 | 데이터셋 | 메트릭 | Vision Banana |
|--------|----------|--------|---------------|
| 의미론적 분할 | Cityscapes | mIoU | 0.842 |
| 인스턴스 분할 | SA-Co/Gold | pmF1 | 0.552 |
| 참조 분할 | RefCOCOg | cIoU | 0.838 |
| 추론 분할 | ReasonSeg | gIoU | 0.793 |

데모 사례로는 마카롱과 케이크 같은 음식 분할, 마늘과 가격 태그 인스턴스 분할, 셰프 이름이나 게임 컨트롤러를 가리키는 참조 분할 등이 제시된다.

### 3D 시각 이해 능력

3D 영역에서는 단안 메트릭 깊이 추정과 표면 법선 추정 두 가지를 수행한다.
메트릭 깊이는 카메라 내재 파라미터 없이 6개 벤치마크 평균으로 평가된다.

| 태스크 | 평가 범위 | 메트릭 | Vision Banana |
|--------|-----------|--------|---------------|
| 메트릭 깊이 추정 | 6 벤치마크 평균 | δ₁ | 0.882 |
| 표면 법선 추정 | 3 벤치마크 평균 | MAE 도 | 15.549 |

폭포, 라군, 농구 선수, 사원 등 다양한 자연 및 도심 장면에서 깊이 맵을 생성한다.
재즈 연주자나 실내 장면에서는 표면 법선 맵으로 객체 표면 방향을 시각화한다.

## 의미와 시사점

논문은 분할과 깊이 추정에서 Segment Anything Model 3, Depth Anything 시리즈와 경쟁할 수준의 성과를 zero-shot으로 달성했다고 보고한다.
이는 별도의 비전 헤드 없이 생성 모델만으로 SOTA에 근접할 수 있음을 의미한다.
저자들은 생성 사전학습이 파운데이션 비전 모델 구축의 중심 패러다임이 될 가능성을 제시한다.
이미지 생성기가 단순히 그림을 그리는 도구가 아니라 시각 표현 학습기로 재해석되는 셈이다.

## 결론

Vision Banana는 "이미지 생성기는 범용 비전 학습자"라는 아이디어를 25명 규모의 저자진이 실증한 결과물이다.
2026년 4월 22일 arXiv에 공개된 이 기술 보고서(arXiv:2604.20329)는 Valentin Gabeur, Shangbang Long, Songyou Peng이 동등 기여로 프로젝트를 이끌었다.
LLM 시대에 텍스트 생성이 모든 자연어 태스크의 통합 인터페이스가 된 것처럼, 이미지 생성도 비전 태스크의 통합 인터페이스로 자리잡을 수 있을지가 향후 관전 포인트다.

## Reference

- [Vision Banana 프로젝트 페이지](https://vision-banana.github.io/#capabilities){:target="_blank"}
- [arXiv:2604.20329 - Image Generators are Generalist Vision Learners](https://arxiv.org/abs/2604.20329){:target="_blank"}
