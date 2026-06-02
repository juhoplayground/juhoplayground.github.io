---
layout: post
title: "JEPA - 얀 르쿤이 제시한 자기회귀 생성 모델의 대안 아키텍처"
author: 'Juho'
date: 2026-06-02 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Embedding]
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
2. [LLM의 한계](#llm의-한계)
3. [해결책의 방향](#해결책의-방향)
   - [월드 모델](#월드-모델)
   - [자기지도 학습](#자기지도-학습)
   - [추상적 표현](#추상적-표현)
   - [목적 기반 AI 아키텍처](#목적-기반-ai-아키텍처)
4. [JEPA의 작동 방식](#jepa의-작동-방식)
   - [입력과 인코더](#입력과-인코더)
   - [예측 모듈](#예측-모듈)
   - [불확실성 처리](#불확실성-처리)
   - [계층적 JEPA](#계층적-jepa)
5. [JEPA 기반 모델들](#jepa-기반-모델들)
   - [I-JEPA 이미지 처리](#i-jepa-이미지-처리)
   - [MC-JEPA 멀티태스킹](#mc-jepa-멀티태스킹)
   - [V-JEPA 비디오 처리](#v-jepa-비디오-처리)
   - [JEPA의 확장 가능성](#jepa의-확장-가능성)
6. [JEPA에서 영감을 받은 모델들](#jepa에서-영감을-받은-모델들)
7. [의미와 시사점](#의미와-시사점)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

Ksenia Se와 Ben Eum이 Turing Post Korea에 공개한 글은, 얀 르쿤이 자기회귀 생성 모델의 대안으로 제시한 JEPA(Joint Embedding Predictive Architecture)를 체계적으로 소개한다.
얀 르쿤은 X에서 이 글을 두고 "JEPA에 대해 설명한 아주 훌륭한 글"이라며 "JEPA는 트랜스포머가 아니라 '자기회귀형 생성 모델(Autoregressive Generative Model)'의 대안이다"라고 명확히 정리했다.
JEPA는 트랜스포머 모듈을 사용하면서도, 다음 토큰을 픽셀/문자 수준에서 예측하는 자기회귀 방식 대신 잠재 공간에서 추상적 표현을 예측한다는 점에서 결이 다르다.

## LLM의 한계

얀 르쿤은 거대언어모델의 한계를 두 축으로 비판한다.

| 한계 | 설명 |
|------|------|
| 상식의 부재 | 텍스트 너머의 실재(reality)에 대한 지식이 제한적이고, 환각(hallucination) 같은 실수를 저지른다 |
| 기억과 계획의 부재 | PlanBench 같은 벤치마크는 GPT-4 같은 SOTA조차 계획 생성 능력이 부족함을 보여준다 |

'Dissociating Language and Thought in Large Language Models' 논문은 LLM이 언어의 규칙·패턴 같은 '형식적 언어 능력'은 뛰어나지만, 세상에서 언어를 실제로 이해하고 사용하는 '기능적 언어 능력'은 불안정함을 보여준다.
이런 한계는 단순히 모델 크기를 키우거나 데이터를 더 넣는 방식으로 풀리지 않을 가능성이 크다는 것이 얀 르쿤의 진단이다.

또한 모라벡의 역설(Moravec's Paradox)은 복잡한 수학적 연산은 컴퓨터가 잘 처리하지만, 인간이 자연스럽게 하는 지각·감각 처리는 기계가 잘 못한다는 점을 짚는다.
다섯 살짜리 어린이 수준의 퍼즐을 까마귀가 풀고, 범고래가 무리 사냥을 하며, 코끼리가 협동을 한다는 사실은 현재 AI가 학습 효율에서 동물보다 한참 떨어진다는 점을 보여준다.

## 해결책의 방향

얀 르쿤은 1960년대 AI 창시자들이 그랬듯, 인지 과학·심리학·신경 과학·엔지니어링의 기초로 돌아가 비전을 다듬었다.
그 비전의 핵심 개념이 월드 모델, 자기지도 학습, 추상적 표현, 그리고 목적 기반 AI 아키텍처다.

### 월드 모델

월드 모델은 "세계가 작동하는 방식·원리를 내부적으로 표현한 것"이다.
얀 르쿤은 다음과 같이 말한다.

> "인간, 동물, 그리고 지능형 시스템이 '월드 모델'을 사용한다는 생각은, 심리학이라든가 제어공학, 로봇공학 같은 분야에서는 이미 수십년 전부터 받아들여진 아이디어예요."

AI 모델에게 주변 세계의 맥락(월드 모델)을 부여하면 모델 성능을 끌어올릴 수 있다는 것이 핵심 가설이다.

### 자기지도 학습

주변을 관찰하면서 학습하는 아기처럼, 자기지도 학습이 또 하나의 축이다.
GPT, BERT, LLaMA 같은 파운데이션 모델이 모두 이 방법론을 기반으로 했고, 머신러닝의 활용 방식을 바꿔 놓았다.

### 추상적 표현

모델은 자기지도 학습과 별도로, 센서가 캡처해야 할 정보와 그렇지 않은 정보를 구분할 수 있어야 한다.
사람의 눈은 매 순간 모든 것을 똑같이 보지 않으며, 필요한 정보의 차이를 잡아낸다.
1999년 '보이지 않는 고릴라(Invisible Gorilla)' 연구는 사람이 자기가 보고 싶은 것에 집중하느라 중요한 것을 놓치는 '무주의 맹시(Inattentional Blindness)'를 보여주는 대표 사례다.

이 비유에서, 얀 르쿤은 모델이 이미지의 픽셀 하나하나를 비교하지 말고 이미지의 추상적 표현(Abstract Representation)을 사용해야 한다고 제안한다.
추상적 표현은 복잡한 정보를 특정 작업이나 분석에 더 적합하고 의미 있게 단순화한 것으로, 시스템이 정보를 더 효율적이고 효과적으로 처리할 수 있게 해준다.

### 목적 기반 AI 아키텍처

얀 르쿤은 자율 인공지능(Autonomous Intelligence)을 위한 모듈화된, 제어 가능한 아키텍처를 제안한다.

| 모듈 | 역할 |
|------|------|
| 제어(Configurator) | 작업에 맞춰 다른 구성 요소의 파라미터를 동적으로 조정하는 제어 센터 |
| 인식(Perception) | 센서 데이터를 수집·해석해 현재 상태를 추정 |
| 월드 모델(World Model) | 미래 상태를 예측하고 누락된 정보를 채움. 시뮬레이터 역할로 가설적 추론·계획 실행 |
| 비용(Cost) | 작업의 잠재 결과를 사전 정의된 비용으로 평가 (Intrinsic Cost + Critic) |
| 행위자(Actor) | 예측·평가를 기반으로 작업을 결정. 최적 제어 이론과 유사하게 예측 비용을 최소화 |
| 단기 기억(Short-term Memory) | 시스템과 환경 간 상호작용 이력을 추적해 실시간 의사결정에 참조 |

비용 모듈은 두 하위 모듈을 가진다.
즉각적인 불편·위험을 계산하는 Intrinsic Cost와, 훈련을 통해 변경 가능하고 현재 행동을 기반으로 미래 비용을 추정하는 Critic이다.
이 아키텍처는 엄청난 양의 레이블 데이터 없이도 AI가 월드 모델을 학습할 수 있는 자기지도 학습의 중요성을 전제한다.

## JEPA의 작동 방식

JEPA는 위에서 설명한 여러 모듈의 핵심 아이디어를 집약해 구현한 개념이다.
예측에 필요한 필수 정보는 유지하면서 관련 없는 세부 정보는 무시하고, 불확실성을 처리할 수 있도록 설계되었다.

### 입력과 인코더

JEPA는 서로 관련 있는 입력 쌍을 받는다.
예를 들어 비디오라면 순차 프레임(x는 현재, y는 다음)을 받는다.
인코더는 입력의 필수 특징만 포착하고 관련 없는 세부 사항을 생략해 추상적 표현 sx와 sy로 변환한다.

### 예측 모듈

예측 모듈은 현재 프레임의 추상적 표현 sx로부터 다음 프레임의 추상적 표현 sy를 예측하도록 훈련된다.
이름 그대로, 인코딩한 임베딩을 결합(Joint)해 예측(Predictive) 모듈을 트레이닝하는 아키텍처(Architecture)다.

| 단계 | 내용 |
|------|------|
| Inputs | 관련 있는 입력 쌍(x, y) |
| Encoders | x, y를 추상적 표현 sx, sy로 인코딩 |
| Predictor | sx로부터 sy를 예측 |

### 불확실성 처리

JEPA는 두 가지 방법으로 불확실성을 처리한다.

첫째, 인코딩 단계에서 인코더가 관련 없는 정보를 삭제한다.
입력 데이터의 어떤 feature가 너무 불확실하거나 노이즈가 있다면, 추상적 표현에 포함하지 않는다.

둘째, 인코딩 후 잠재 변수(Latent Variable; z)를 활용한다.
z는 sy에는 있지만 sx에서는 관찰할 수 없는 요소다.
미리 정의된 값 집합 내에서 여러 값을 가질 수 있고, 각각의 값은 미래 상태 y에 나타날 수 있는 가설적 시나리오를 표현한다.
z값을 변경해 가면서 보이지 않는 요소의 작은 변화가 이후 상태에 어떤 영향을 미치는지 예측 모델이 시뮬레이션할 수 있다.

### 계층적 JEPA

여러 개의 JEPA를 다단계(multistep) 또는 반복(recurrent) 구조로 결합하거나, 계층적 JEPA(Hierarchical JEPA)로 쌓아 올려 여러 단계의 추상화 수준과 다중 타임 스케일에서 예측을 수행할 수 있다.
이 점은 단순한 next-frame 예측을 넘어, 다른 시간 척도에서 동시에 추론을 수행하는 시스템으로 확장될 가능성을 보여준다.

## JEPA 기반 모델들

얀 르쿤과 메타 AI 연구원들이 JEPA 아키텍처를 기반으로 발표한 대표 모델은 다음과 같다.

### I-JEPA 이미지 처리

2023년 6월 발표된 I-JEPA(Image-based JEPA)는 JEPA 아키텍처를 기반으로 한 최초의 모델이다.
비생성형(Non-Generative) 자기지도 학습 프레임워크이며, 이미지의 일부를 가리고 그 부분이 어떤 모습인지 예측하도록 훈련한다.

| 단계 | 설명 |
|------|------|
| Masking | 이미지를 다수의 패치로 분할하고, 일부 '타겟 블럭'을 마스킹 |
| Context Sampling | 마스킹하지 않은 '컨텍스트 블럭'을 통해 이미지 구성을 이해 |
| Prediction | 예측 모델이 컨텍스트 블럭의 정보로 마스킹된 부분의 표현을 예측 |
| Iteration | 마스킹된 실제 패치와 예측 패치의 간극을 줄이며 파라미터 업데이트 |

구성 요소 측면에서 I-JEPA는 세 부분으로 이루어져 있고, 각각 비전 트랜스포머(ViT; Vision Transformer)다.

| 컴포넌트 | 역할 |
|----------|------|
| Context Encoder | 마스킹되지 않은 컨텍스트 블럭을 처리 |
| Predictor | 컨텍스트 인코더 출력으로 마스킹된 부분을 예측 |
| Target Encoder | 마스킹된 타겟 블럭으로 학습·예측에 사용할 추상적 표현 생성 |

핵심은 마스킹된 이미지의 추상적 표현을 정확하게 예측하도록 예측 모듈을 트레이닝하는 것이다.
명시적 레이블 없이도 모델이 학습하는 자기지도 학습 구조다.

### MC-JEPA 멀티태스킹

MC-JEPA(Motion-Content JEPA)는 인코더 하나로 '동적' 요소(Motion; 움직임)와 '정적' 세부 정보(Content; 컨텐츠)를 동시에 해석하도록 설계되었다.
I-JEPA 발표 한 달 뒤인 2023년 7월에 공개되었다.
자율 주행, 영상 감시(Video Surveillance) 같은 실제 어플리케이션에서 더 강력하고 안정적인 성능을 보여주는 모델이다.

### V-JEPA 비디오 처리

V-JEPA(Video-JEPA)는 동영상 컨텐츠를 AI가 더 잘 이해하도록 설계된 JEPA다.

| 컴포넌트 | 역할 |
|----------|------|
| Encoder | 입력된 비디오 프레임을 고차원 공간에 투사해 핵심 시각 특징을 포착 |
| Predictor | 비디오의 한 부분의 인코딩된 특징으로 다른 부분의 특징을 예측 |

비디오 내 시간적·공간적 변환을 학습하기 때문에 시간 흐름에 따른 움직임과 변화를 이해할 수 있다.
명시적 주석 없이 관찰을 통해 학습하는 사람처럼 비지도 방식으로 비디오를 학습한 뒤, 다양한 다운스트림 태스크로 연결할 수 있어 동적 시각 자료를 다루는 강력한 도구가 된다.

### JEPA의 확장 가능성

2024년 3월에 발표된 'Learning and Leveraging World Models in Visual Representation Learning'은 '이미지 월드 모델(IWM; Image World Models)'이라는 개념을 도입한다.
마스킹뿐 아니라 더 다양한 이미지 손상(색상 흔들림, 흐림 등)에도 잘 대응하도록 JEPA 아키텍처를 일반화·확장하는 방향이다.

| 유형 | 설명 |
|------|------|
| 불변 모델(Invariant Models) | 시나리오가 달라져도 변하지 않는 '불변의 특징'을 인식·유지 |
| 등변량 모델(Equivariant Models) | 입력 데이터가 변할 때 따라오는 관계와 변환 내용을 보존 |

이 연구는 직접적인 감독·지도 없이 머신러닝 모델의 효율성을 높일 수 있는 새로운 아이디어를 제시한다.
시각적 변화가 있을 때 AI가 더 잘 적응하고 더 정확히 예측할 수 있다는 점을 실증했다.

## JEPA에서 영감을 받은 모델들

JEPA 개념에서 영감을 받은 모델은 어플리케이션 영역별로 다음과 같이 정리된다.

### 오디오와 스피치 영역

| 모델 | 설명 |
|------|------|
| A-JEPA | 오디오·음성 분류 작업에서 문맥 의미 이해를 위해 마스크 모델링 원리 사용 |
| Investigating Design Choices in JEPAs for General Audio Representation Learning | 자기지도 오디오 표현 학습에서 마스킹 전략과 샘플 길이 분석 |

### 비전과 공간 데이터 영역

| 모델 | 설명 |
|------|------|
| S-JEA | 스택형 조인트 임베딩 아키텍처에서 계층적 의미 표현으로 시각 표현 학습 향상 |
| DMT-JEPA | 분류·객체 감지·세그먼테이션에 적용 가능한 Local Semantic Understanding 중심 이미지 모델링 |
| JEP-KD | 시각적 음성 인식 모델을 오디오 기능에 맞춰 조정해 인식 성능 향상 |
| Point-JEPA | JEPA를 포인트 클라우드 데이터에 적용해 공간 데이터셋 효율·표현 학습 향상 |
| Signal-JEPA | EEG 신호 처리를 위한 Cross-Dataset Transfer 및 분류 성능 개선 |

### 그래프와 동적 데이터 영역

| 모델 | 설명 |
|------|------|
| Graph-JEPA | 서브그래프 표현을 위해 쌍곡선 좌표 예측을 사용하는 최초의 그래프용 JEPA |
| ST-JEMA | 높은 수준의 의미론적 표현에 초점, fMRI 데이터에서 동적 기능적 연결성 학습 |

### 시계열 데이터와 원격 센싱 영역

| 모델 | 설명 |
|------|------|
| LaT-PFN | 시계열 예측과 JEPA를 결합해 강력한 ICL(In-Context Learning) 지원 |
| Time-Series JEPA | 센서 데이터의 시공간 상관관계로 제한된 용량 네트워크에서 원격 제어 최적화 |

오디오부터 EEG, 포인트 클라우드, 그래프, 시계열에 이르기까지 JEPA의 핵심 아이디어(추상 공간에서의 예측)가 광범위하게 확산되고 있음을 알 수 있다.

## 의미와 시사점

JEPA가 지금 주목받는 이유는 세 가지로 정리된다.

첫째, 자기회귀 생성 모델의 한계를 정면으로 짚는다.
픽셀/문자 단위의 token-by-token 예측은 의미적으로 중요하지 않은 정보까지 모두 예측해야 하므로 학습 효율과 추상화 측면에서 비효율적일 수 있다.
JEPA는 예측 대상을 잠재 공간으로 끌어올려 의미 있는 정보만 다루도록 설계한다.

둘째, 월드 모델·자기지도 학습·추상적 표현이라는 세 축을 통합한다.
이전까지 LLM은 자기지도 학습에 강점이 있었지만 월드 모델은 약했고, RL 기반 시스템은 그 반대였다.
JEPA는 이 두 흐름을 잠재 공간 예측이라는 하나의 메커니즘으로 묶으려는 시도다.

셋째, I-JEPA → MC-JEPA → V-JEPA → IWM으로 이어지는 확장 경로가 명확하다.
이미지 → 멀티태스킹 → 비디오 → 일반화된 손상 처리로 이어지는 시리즈는, 단발성 아이디어가 아니라 한 비전을 단계적으로 구현하는 연구 프로그램임을 보여준다.

## 결론

JEPA는 트랜스포머라는 모듈을 부정하는 것이 아니라, '다음 토큰을 픽셀/문자 단위로 예측한다'는 자기회귀 생성 패러다임에 대한 대안이다.
얀 르쿤이 비전으로 제시한 월드 모델, 자기지도 학습, 추상적 표현, 목적 기반 AI 아키텍처를 잠재 공간 예측이라는 하나의 메커니즘으로 묶고, I-JEPA·MC-JEPA·V-JEPA·IWM 등으로 단계적으로 구현해 가고 있다.
A-JEPA, Graph-JEPA, Point-JEPA, Signal-JEPA, Time-Series JEPA 등 다양한 도메인 확장은 JEPA 아이디어가 메타 외부에서도 확산되고 있음을 보여준다.
LLM의 한계(상식 부재, 계획 부재)를 구조적으로 풀기 위한 또 하나의 진지한 시도라는 점에서, 자기회귀 생성 모델 일변도로 흐르는 현재 흐름과 대비해 따라가 볼 만한 연구 프로그램이다.

## Reference

- [Topic #4: 얀 르쿤의 최애? 역작? JEPA란 무엇인가? - Turing Post Korea (Ksenia Se & Ben Eum, 2024-07-19)](https://turingpost.co.kr/p/jepa-joint-embedding-predictive-architecture)
- [Yann LeCun on a Vision to Make AI Systems Learn and Reason like Animals and Humans - Meta AI](https://ai.meta.com/blog/yann-lecun-advances-in-ai-research/)
- [I-JEPA: Self-Supervised Learning from Images with a Joint-Embedding Predictive Architecture](https://arxiv.org/abs/2301.08243)
- [MC-JEPA: A Joint-Embedding Predictive Architecture for Self-Supervised Learning of Motion and Content Features](https://arxiv.org/abs/2307.12698)
- [V-JEPA: The Next Step toward Advanced Machine Intelligence - Meta AI](https://ai.meta.com/blog/v-jepa-yann-lecun-ai-model-video-joint-embedding-predictive-architecture/)
- [Learning and Leveraging World Models in Visual Representation Learning](https://arxiv.org/abs/2403.00504)
