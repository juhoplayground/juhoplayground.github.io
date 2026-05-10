---
layout: post
title: "감정적 프롬프트가 AI 성능을 바꿀까? EmotionRL 적응형 감정 프레이밍 연구"
author: 'Juho'
date: 2026-04-11 00:00:00 +0900
categories: [AI]
tags: [LLM, Prompt, Benchmark, Agent]
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
   - [감정 프레이밍 연구의 공백](#감정-프레이밍-연구의-공백)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [감정 분류와 자극 생성](#감정-분류와-자극-생성)
   - [EmotionRL 정책 학습](#emotionrl-정책-학습)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [정적 감정 prefix는 효과 약함](#정적-감정-prefix는-효과-약함)
   - [감정 강도 영향](#감정-강도-영향)
   - [인간 vs LLM 작성 prefix](#인간-vs-llm-작성-prefix)
   - [EmotionRL의 적응적 우위](#emotionrl의-적응적-우위)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"Do Emotions in Prompts Matter? Effects of Emotional Framing on Large Language Models"는 사용자 쿼리의 감정적 어조가 LLM 정확도에 어떤 영향을 주는지 6개 벤치마크에서 체계적으로 조사한 논문이다.
저자는 Minda Zhao, Yutong Yang, Chufei Peng, Rachel Gonsalves, Weiyue Li, Ruyi Yang, Zhixi Liu, Mengyu Wang(Harvard University, Bryn Mawr College)이다.

논문 abstract의 핵심은 다음과 같다.
정적 감정 prefix는 정확도에 작은 변화만 유발하며 신뢰할 만한 개입이 되지 못한다.
저자들은 입력별로 감정을 적응적으로 선택하는 EmotionRL 프레임을 도입해 정적 감정 prompting보다 일관된 개선을 얻는다.
결론은 감정 어조가 약하지만 입력 의존적인 신호이며 적응적 통제로만 의미 있게 활용 가능하다는 것이다.

## 배경과 선행 연구

### 감정 프레이밍 연구의 공백

저자들은 인간 의사소통이 본질적으로 감정 가치를 내포하지만 LLM 응답이 이에 어떻게 반응하는지는 충분히 조사되지 않았다고 본다.
선행 연구는 명시적 역할 부여나 전략적 시나리오 재구성에서 감정을 다뤘지만 다음 세 가지 공백이 남아 있다.
첫째, 1인칭 감정 prefix가 표준 벤치마크 전반에서 체계적 효과를 내는지.
둘째, 효과가 prefix 작성자(인간 vs LLM)에 따라 달라지는지.
셋째, 감정을 고정 조건이 아닌 적응적으로 다뤄야 하는지.

### Related Work 정리

| 카테고리 | 주요 흐름 | 본 연구와의 차이 |
|----------|-----------|------------------|
| 역할 기반 prompting | "act as a doctor" 같은 persona | 명시적 역할 — 본 연구는 1인칭 감정 |
| Chain-of-thought | 추론 강화 prompting | 인지 측면 — 본 연구는 affective 측면 |
| Persuasion | 전략적 설득 시나리오 | 도메인 특화 — 본 연구는 일반 벤치마크 |
| Robustness | 적대적 프롬프트 | 공격 — 본 연구는 자연 발화 |

본 연구의 차별점은 일반 벤치마크 6개에 대해 6개 감정 prefix를 체계적으로 비교하고, 적응형 학습 정책 EmotionRL을 추가한 것이다.

## 방법론

### 감정 분류와 자극 생성

Plutchik 기본 감정 중 6가지를 채택한다(happiness, sadness, fear, anger, disgust, surprise).
GPT-4o가 단일 문장 감정 표현을 생성하되, 수치값·entity·범위·난이도·사실 내용을 변경하지 않도록 제약한다.

### EmotionRL 정책 학습

action space.

```
A = { ANGER, DISGUST, FEAR, HAPPINESS, SADNESS, SURPRISE }
```

state는 frozen sentence embedding이다(backbone LLM과 무관).

```
s_i = f_emb(x_i) ∈ R^d
```

reward는 binary correctness signal.

```
r_i^(k) = 1[ ŷ_i^(k) = y_i ]
```

정책은 2-layer MLP다.

```
z_i = W_2 * σ( W_1 * s_i + b_1 ) + b_2
```

soft supervision weight (centered reward 기반).

```
w_i^(k) = exp( (r_i^(k) - r̄_i) / τ ) / Σ_j exp( (r_i^(j) - r̄_i) / τ )
```

손실은 weighted cross-entropy.

```
L = - Σ_i Σ_k w_i^(k) * log π_θ(a_k | s_i)
```

τ=1로 적당한 smoothing을 준다.
test 시 단일 greedy 선택 후 frozen LLM에 1회 query한다.

```
a* = argmax π_θ(a_k | s)
```

## 실험 셋업

| 항목 | 값 |
|------|-----|
| Inference | zero-shot, temperature 0.0 |
| 모델 | Qwen3-14B, Llama 3.3-70B, DeepSeek-V3.2 |
| 데이터셋 | GSM8K, BBH, MedQA, BoolQ, OpenBookQA, SocialIQA |
| 감정 수 | 6 (Plutchik 기본 감정) |
| 정책 | 2-layer MLP, softmax(temperature τ=1) |
| EmotionRL train | GSM8K, OpenBookQA, SocialIQA, MedQA, BoolQ (BBH는 표준 train split 부재로 제외) |
| 강도 검증 | MedQA-US, slight/moderate/extreme |
| 작성자 검증 | MedQA-US 250문항(219개 사용 가능), 인간 vs LLM |

## 주요 결과

### 정적 감정 prefix는 효과 약함

Figure 3에서 6개 벤치마크와 3개 모델 조합 전반에서 감정 prefix는 정확도에 큰 변화를 주지 못한다.
GSM8K와 MedQA-US는 baseline과 거의 동일하다.
유일한 명확한 예외는 SocialIQA로, 사회적 추론이 본질이므로 감정 컨텍스트가 더 큰 변동을 유발한다.

### 감정 강도 영향

MedQA-US에서 slight, moderate, extreme 강도를 비교한 결과 모든 모델에서 변화가 0 부근에 머물렀다.
강한 표현이 약간의 하향 추세만 유발해, 강도가 효과의 크기는 조정하지만 regime 변화는 일으키지 않는다.

### 인간 vs LLM 작성 prefix

Qwen3-14B를 MedQA-US에 적용한 비교에서 인간 작성과 LLM 작성 prefix가 거의 동일한 정확도를 보였다.
이는 prefix 작성 출처가 결과를 좌우하지 않음을 입증해 본 연구 결론의 robustness를 뒷받침한다.

### EmotionRL의 적응적 우위

Figure 6에서 EmotionRL은 5개 태스크 전반에서 일관되게 경쟁력 있고 종종 더 우수한 결과를 보였다.
정적 감정 prompting은 일관성이 없으며 일부 태스크에서는 중립적이거나 오히려 성능을 떨어뜨린다.
입력 조건부 선택이 감정 프레이밍을 신뢰할 만한 개입으로 변환한다는 점을 보여주는 결과다.

## 한계와 디스커션

저자가 명시한 한계와 주의사항은 다음과 같다.
첫째, 감정 분류는 Plutchik 6개로 제한되며 micro-expression이나 mixed emotion은 다루지 않는다.
둘째, 평가는 영어 중심이며 다국어 감정 표현으로의 일반화는 검증되지 않았다.
셋째, EmotionRL은 정답 라벨이 있는 데이터셋에 한해 학습 가능하므로 open-ended 생성 태스크에는 직접 적용이 어렵다.
넷째, sentence embedding이 frozen이므로 감정 의미가 충분히 표현되지 않는 임베딩에서는 정책 학습이 제한된다.

디스커션의 핵심은 두 가지다.
첫째, 감정 프레이밍은 약하지만 입력 의존적인 신호이며 단일 보편 감정으로 모든 입력을 다룰 수 없다.
둘째, EmotionRL의 reward-weighted cross-entropy는 binary correctness를 soft 분포로 변환해 상대적 유용성을 학습하는 우아한 설계이며, 다른 input-conditioned prompting 선택 문제에도 일반화 가능하다.

## 결론

본 연구는 6개 벤치마크와 3개 LLM에 대해 정적 감정 prefix가 정확도에 작은 변화만 유발한다는 점을 정량적으로 입증했다.
강도 변화는 효과를 미세 조정만 할 뿐 regime을 바꾸지 않으며, 인간 vs LLM 작성 prefix는 거의 동일한 결과를 낸다.
EmotionRL은 입력별 감정 선택을 학습해 정적 prompting의 비일관성을 해소하고 5개 태스크 전반에서 일관된 우위를 보였다.
감정 프레이밍을 적응적 control 문제로 재정의했다는 점이 본 연구의 핵심 기여다.

## Reference

- [Do Emotions in Prompts Matter? Effects of Emotional Framing on Large Language Models (arXiv:2604.02236)](https://arxiv.org/abs/2604.02236/)
