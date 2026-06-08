---
layout: post
title: "장기 에이전트 작업의 교차 벤치마크 일반화"
author: 'Juho'
date: 2026-06-08 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, Benchmark, Evaluation]
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
3. [주요 결과](#주요-결과)
4. [한계와 주의사항](#한계와-주의사항)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

이 글은 Surge AI가 공개한 장기 에이전트 작업의 교차 벤치마크 일반화 연구를 정리한 것이다.

핵심 주장은 명확하다.

RL 환경에서의 학습 개선은 학습 분포를 넘어 전이될 때만 의미가 있다는 것이다.

학습 환경에서만 점수가 오르고 그 밖의 환경으로 전이되지 않는다면, 이는 환경에 특화된 과적합일 뿐 진짜 역량의 발달로 보기 어렵다.

이를 검증하기 위해 연구진은 Qwen3.5-122B-A10B 모델을 자체 General Agent Tasks 스위트로 학습했다.

그리고 학습 데이터와 완전히 분리된 3개의 외부 벤치마크에서 성능을 측정했다.

측정에 사용한 외부 벤치마크는 Toolathlon, τ²-Bench, BFCL-V4이다.

세 벤치마크 모두 학습 데이터와 겹치지 않는 완전 외부 평가다.

## 방법론

학습 환경인 General Agent Tasks 스위트는 생산적 업무 시나리오를 시뮬레이션한다.

스프레드시트, 터미널, 캘린더, 검색, 파일시스템 등을 다루는 27개 작업 범주로 구성된다.

각 작업은 25개에서 40개 이상의 도구 호출 턴을 필요로 하는 장기 작업이다.

각 작업에는 다기준 완료를 채점하는 결정론적 Python grader가 포함된다.

이 grader는 sparse 보상과 dense 보상을 모두 생성할 수 있다.

학습 파이프라인은 2단계로 구성된다.

먼저 SFT를 수행하고, 그다음 GSPO 기반 RL을 적용한다.

LoRA는 rank 32, alpha 32 설정으로 attention과 MLP projection에 적용했다.

인프라는 slime, Megatron-Bridge, SGLang으로 구성했다.

핵심 설계 결정은 보상 희소성 문제를 다루는 방식이다.

베이스 모델의 pass rate가 부족하면 RL 단계에서 받을 수 있는 보상이 거의 없어 학습 신호가 희박해진다.

이 문제를 해결하기 위해 RL 이전에 SFT를 먼저 적용했다.

또한 per-criterion grader에서 유도한 dense reward를 사용해 학습 신호를 대폭 늘렸다.

## 주요 결과

학습 후 Pass@1 향상을 인분포 holdout과 3개 외부 벤치마크에서 측정했다.

향상폭은 다음과 같다.

| 평가 대상 | Pass@1 향상 |
| --- | --- |
| 인분포 holdout | +17.3pp |
| Toolathlon | +9.6pp |
| τ²-Bench | +5.3pp |
| BFCL-V4 | +3.5pp |

핵심은 학습 분포 밖인 세 외부 벤치마크 전반에서 일관된 개선이 나타났다는 점이다.

GPT-5.5와의 비교에서도 경쟁력을 보였다.

Toolathlon과 τ²-Bench에서는 pass@1 기준으로 GPT-5.5(medium reasoning)와 약 1pp 이내로 근접했다.

BFCL-V4에서는 pass@4 기준으로 GPT-5.5를 초과했다(72.2% vs 69.4%).

dense reward 도입은 학습 신호 자체를 크게 개선했다.

작업당 평균 보상은 0.30에서 0.51로 올랐다.

보상 신호를 내는 작업의 비율은 16.8%에서 82.7%로 증가했다.

Toolathlon 범주별로 보면 향상폭이 큰 영역이 두드러졌다.

Office 범주는 0%에서 16.5%로 올랐다.

Shopping은 +11.5pp, Finance는 +10.1pp, Tech는 +9.9pp 향상됐다.

가장 흥미로운 발견은 보상 함수에 명시적으로 안내하지 않은 행동 변화다.

학습된 모델은 두 가지 능력을 스스로 발달시켰다.

첫째는 병렬 도구 호출이다.

서로 독립적인 도구 호출을 동시에 실행해 컨텍스트 윈도우 소진을 줄였다.

둘째는 작업 마무리 규율이다.

중간 분석 단계에서 멈추지 않고 다단계 워크플로를 끝까지 완료하는 능력이다.

이러한 개선이 세 외부 벤치마크 전반으로 전이됐다.

이는 환경 특화 과적합이 아니라 진짜 역량 발달이 일어났음을 시사한다.

## 한계와 주의사항

이 연구는 블로그 형식의 개괄적 결과 공유에 해당한다.

상세 ablation과 trajectory 분석이 담긴 정식 기술 논문은 후속으로 예고됐다.

따라서 현재 공개된 수치는 정식 논문에서 더 엄밀하게 검증될 여지가 있다.

또한 평가는 Toolathlon, τ²-Bench, BFCL-V4 세 벤치마크에 한정된다.

다른 도메인이나 더 긴 장기 작업으로의 일반화는 별도의 검증이 필요하다.

## 결론

RL 환경의 가치는 학습 분포 너머로 역량이 일반화되는지에 달려 있다.

학습 환경에서만 점수가 오르는 것은 충분한 증거가 되지 못한다.

이 연구는 완전 외부 벤치마크에서 측정 가능한 개선을 보이며 전이 가능한 에이전트 역량을 입증했다.

인분포 holdout +17.3pp와 더불어 Toolathlon +9.6pp, τ²-Bench +5.3pp, BFCL-V4 +3.5pp의 외부 전이가 그 근거다.

명시적 안내 없이 발달한 병렬 도구 호출과 작업 마무리 규율 역시 역량 발달의 신호로 볼 수 있다.

## Reference

- [Cross-Benchmark Generalization for Long-Horizon Agentic Tasks](https://surgehq.ai/blog/cross-benchmark-generalization-for-long-horizon-agentic-tasks/)
