---
layout: post
title: "GOAT - 학습 가능한 사전 분포로 어텐션 메커니즘을 재설계하다"
author: 'Juho'
date: 2026-05-09 00:00:00 +0900
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
2. [방법론](#방법론)
   - [Entropic Optimal Transport 관점의 어텐션](#entropic-optimal-transport-관점의-어텐션)
   - [GOAT 메커니즘](#goat-메커니즘)
3. [주요 결과](#주요-결과)
   - [어텐션 싱크 문제 해결](#어텐션-싱크-문제-해결)
   - [길이 일반화](#길이-일반화)
4. [한계와 주의사항](#한계와-주의사항)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

"You Need Better Attention Priors"는 트랜스포머 어텐션 메커니즘을 Entropic Optimal Transport(EOT, 엔트로픽 최적 수송) 이론의 관점에서 재해석한 논문이다.
저자 Elon Litman과 Gabe Guo는 표준 어텐션이 사실상 "암묵적 균등 사전 분포(uniform prior)"에 의존하고 있음을 지적한다.
이를 학습 가능한 연속적 사전 분포로 대체하는 GOAT(Generalized Optimal transport Attention with Trainable priors) 메커니즘을 제안하며, 동시에 FlashAttention과 같은 가속 커널과의 호환성도 유지한다.
논문은 2026년 1월 21일 arXiv에 제출되었다.

## 방법론

### Entropic Optimal Transport 관점의 어텐션

저자들은 표준 어텐션을 두 분포 사이의 최적 수송 문제로 해석한다.
이 관점에서 보면 일반적인 softmax 기반 어텐션은 사전 분포가 균등하다는 가정 위에서 동작하는 특수한 사례에 해당한다.
즉 모든 토큰에 동일한 prior weight가 부여되고, 데이터 의존적인 분포 정보가 사전에 반영되지 않는 구조라는 것이다.

EOT 프레임워크는 이러한 균등 사전 가정을 명시적으로 드러내고, 더 나은 사전 분포가 성능을 본질적으로 끌어올릴 수 있음을 시사한다.

### GOAT 메커니즘

GOAT는 이 암묵적 균등 가정을 학습 가능한 연속 사전 분포로 대체한다.
주요 설계 원칙은 다음과 같다.

| 구성 요소 | 설명 |
|-----------|------|
| Trainable priors | 데이터로부터 학습되는 연속적 사전 분포 |
| Kernel compatibility | FlashAttention 등 가속 커널과 호환되는 형식 유지 |
| Spatial information | 어텐션 계산 안에 공간 정보를 직접 통합 |

기존 어텐션의 효율성을 깨지 않으면서 사전 정보를 모델 일부로 학습한다는 점이 핵심 기여이다.

## 주요 결과

### 어텐션 싱크 문제 해결

GOAT는 어텐션 싱크(특정 토큰에 비정상적으로 큰 attention weight가 집중되는 현상)에 대한 EOT 기반 설명을 제공한다.
표현력을 희생하지 않으면서 이 현상을 해소할 수 있는 방법을 제시하며, 사전 분포 자체가 문제의 원인이자 해법이라는 일관된 관점을 제공한다.

### 길이 일반화

GOAT는 공간 정보를 어텐션 계산에 직접 통합하여 "외삽 가능한 사전 분포(extrapolatable prior)"를 만든다.
이 사전 분포는 학습된 위치 임베딩의 유연성과 고정 인코딩(예: ALiBi, RoPE 계열의 일부 특성)의 일반화 능력을 결합한다.
결과적으로 학습 시 보지 못한 더 긴 시퀀스에 대해서도 안정적인 동작을 기대할 수 있다.

## 한계와 주의사항

본 논문의 요약 정보는 abstract 수준에서 정리된 내용이며, 구체적인 벤치마크 수치, 모델 규모, 학습 비용 비교 등의 정량 결과는 원문 본문을 확인해야 한다.
또한 GOAT가 가속 커널과 호환된다고 명시되어 있으나, 실제 구현 시 학습 가능한 사전 분포가 FlashAttention 커널의 특정 가정(예: 메모리 접근 패턴)과 어떻게 통합되는지는 별도의 검토가 필요하다.

## 결론

GOAT는 어텐션을 Entropic Optimal Transport 문제로 재해석하고, 표준 어텐션의 암묵적 균등 사전 분포를 학습 가능한 연속 사전 분포로 대체한 기여를 제시한다.
이 접근은 어텐션 싱크 문제 해결, 길이 일반화 능력 향상, 가속 커널과의 호환이라는 세 가지 실용적 이점을 동시에 노린다.
"더 나은 사전 분포"라는 단순한 메시지가 어텐션 메커니즘 설계의 다음 방향을 제시할 수 있을지가 관전 포인트이다.

## Reference

- [You Need Better Attention Priors (arXiv:2601.15380)](https://arxiv.org/abs/2601.15380)
