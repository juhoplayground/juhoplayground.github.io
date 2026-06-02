---
layout: post
title: "Motivation은 Novelty가 아니다 - 명작 논문 분석으로 본 박사 신입생 연구 가이드"
author: 'Juho'
date: 2026-06-01 00:00:00 +0900
categories: [Dev]
tags: [Documentation, Skill, Knowledge]
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
2. [Motivation과 Novelty의 차이](#motivation과-novelty의-차이)
   - [Motivation 단계의 한계](#motivation-단계의-한계)
   - [Novelty 도달의 조건](#novelty-도달의-조건)
3. [세 가지 명작 사례](#세-가지-명작-사례)
   - [ResNet](#resnet)
   - [Transformer](#transformer)
   - [NeRF](#nerf)
4. [자가진단 체크리스트](#자가진단-체크리스트)
5. [실제 미팅 대화에서 얻는 교훈](#실제-미팅-대화에서-얻는-교훈)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

박사 신입생이 가장 흔히 빠지는 함정 중 하나는 "기존 방법이 X를 못한다는 점을 발견했고, 우리가 그것을 해결하는 모듈을 붙였다"라는 식의 글쓰기다.
DGIST APRL의 가이드는 이런 구성이 motivation일 뿐 novelty가 아님을 분명히 한다.
이 글에서는 motivation과 novelty의 경계, 명작 논문에서 발견되는 공통 패턴, 그리고 박사 과정에서 사용할 수 있는 자가진단 체크리스트를 정리한다.

## Motivation과 Novelty의 차이

가이드의 핵심은 두 개념을 같은 선상에 두지 않는 것이다.
Motivation은 문제 제기까지의 단계이며, novelty는 그 문제의 해법이 분석으로부터 자연스럽게 도출되어야 한다는 단계다.

### Motivation 단계의 한계

| 특징 | 설명 |
|------|------|
| 문제 제기에서 멈춤 | "기존 방법이 X를 못한다"는 관찰만 있고 원리적 진단이 없다 |
| 모듈 부착 | 새 모듈을 붙이지만 왜 그 형태여야 하는지 설명하지 못한다 |
| 선행 연구 분석 미흡 | naive baseline의 실패 원인을 한 문장으로 설명하지 못한다 |

이 단계에서 멈추면 리뷰어가 "왜 그 형태인가"를 끝없이 물어보게 된다.

### Novelty 도달의 조건

| 특징 | 설명 |
|------|------|
| 원리적 진단 | 실패 원인을 수식이나 분석 결과로 명확히 보여준다 |
| 해법의 필연성 | 제안 방법의 형태가 진단으로부터 강제적으로 도출된다 |
| 광범위한 ablation | 각 component의 기여를 ablation으로 입증한다 |

가이드는 이 단계에 도달해야 비로소 탑티어 학회에 통할 수 있다고 말한다.

## 세 가지 명작 사례

가이드는 세 편의 대표 논문을 사례로 들며 motivation에서 novelty로 넘어가는 흐름을 보여준다.

### ResNet

ResNet은 "깊은 네트워크가 학습되지 않는다"는 motivation에서 출발하지만, 핵심은 "identity 매핑조차 학습하기 어렵다"는 원리적 진단이다.
이 진단에서 잔차 연결(skip connection)이라는 해법이 자연스럽게 도출된다.
모듈을 그냥 붙인 것이 아니라, 분석이 해법의 형태를 강제했다는 점이 핵심이다.

### Transformer

Transformer는 "순환 구조 때문에 병렬화가 안 된다"는 문제로 시작한다.
원리적 진단은 "long-range dependency를 다루기 위해 순환을 사용했지만, 순환이 병렬화를 막는다"는 것이다.
해법은 순환을 완전히 제거하고, 위치 정보를 별도로 인코딩하는 것으로 자연스럽게 도출된다.

### NeRF

NeRF는 MLP가 고주파 함수를 학습하지 못하는 "spectral bias"라는 원리적 진단에서 출발한다.
이 진단에서 입력 좌표의 주파수를 끌어올리는 positional encoding이 해법으로 도출된다.
역시 분석이 해법의 형태를 결정한 사례다.

## 자가진단 체크리스트

가이드는 연구자가 스스로 점검할 수 있는 세 가지 질문을 제시한다.

| 질문 | 점검 포인트 |
|------|------------|
| Naive baseline의 실패 원인을 한 문장으로 설명할 수 있는가 | 진단의 명료성 |
| 각 component를 제거했을 때 영향 범위를 ablation으로 입증했는가 | 해법의 필연성 |
| 방법을 한 줄의 수식으로 표현할 수 있는가 | 형식화 가능성 |

세 질문 모두에 자신 있게 답할 수 없으면 아직 motivation 단계에 머물러 있을 가능성이 높다.

## 실제 미팅 대화에서 얻는 교훈

가이드는 교수가 학생에게 "인트로 두 번째 문단까지만 왔다"고 지적하는 실제 대화 사례를 소개한다.
모티베이션을 도출한 뒤에는 실제 실패 사례를 분류하는 단계를 거쳐야 챌린지 포인트가 보이며, 그 챌린지 포인트가 곧 novelty의 씨앗이 된다.
즉 motivation에서 멈추지 않고, 실패 사례를 모아 패턴을 찾고, 패턴이 가리키는 원인을 분석하는 일련의 작업이 필요하다.

## 결론

탑티어 논문은 "왜 기존 방식이 실패하는가"를 깊이 분석하고, 그 분석이 제안 방법의 형태를 강제적으로 도출하는 특징을 가진다.
박사 신입생이 처음 논문을 쓸 때 "기존이 X를 못한다 → 모듈을 붙였다" 구조에 머무는 것은 매우 흔하지만, 가이드의 체크리스트를 거치면 자신의 글이 motivation에서 멈췄는지 novelty에 도달했는지 빠르게 확인할 수 있다.

## Reference

- [Motivation is not Novelty - DGIST APRL](https://gisbi-kim.github.io/motivation-is-not-novelty/)
