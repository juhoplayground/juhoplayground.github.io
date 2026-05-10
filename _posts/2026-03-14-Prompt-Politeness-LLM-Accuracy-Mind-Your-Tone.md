---
layout: post
title: "프롬프트의 정중함이 LLM 정확도에 미치는 영향 - Mind Your Tone 논문 분석"
author: 'Juho'
date: 2026-03-14 00:00:00 +0900
categories: [LLM]
tags: [LLM, Prompt, AI]
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
   - [프롬프트 엔지니어링과 정중함](#프롬프트-엔지니어링과-정중함)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [데이터셋 구성](#데이터셋-구성)
   - [정중함 5단계와 prefix 예시](#정중함-5단계와-prefix-예시)
   - [평가 절차](#평가-절차)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [정확도와 t-test 결과](#정확도와-t-test-결과)
   - [Yin et al. 2024와의 차이](#yin-et-al-2024와의-차이)
6. [한계와 디스커션](#한계와-디스커션)
7. [윤리적 고려사항](#윤리적-고려사항)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

"Mind Your Tone: Investigating How Prompt Politeness Affects LLM Accuracy"는 자연언어 프롬프트의 정중함 수준이 LLM 정확도에 어떤 영향을 미치는지 다중선택 문제로 조사한 단편 논문이다.
저자는 Om Dobariya, Akhil Kumar(Pennsylvania State University)이며 ChatGPT-4o를 주된 평가 모델로 사용한다.

논문 abstract의 핵심은 다음과 같다.
50개 base 다중선택 문제를 5개 정중함 변형(Very Polite, Polite, Neutral, Rude, Very Rude)으로 다시 작성해 250개 unique prompt를 만들었다.
예상과 달리 무례한 prompt가 정중한 prompt보다 일관되게 더 높은 정확도를 보였다(Very Polite 80.8%, Very Rude 84.8%).
이 결과는 무례함과 낮은 성능을 연결한 선행 연구(Yin et al. 2024)와 다른 양상을 보여 신형 LLM이 톤 변화에 다르게 반응함을 시사한다.

## 배경과 선행 연구

### 프롬프트 엔지니어링과 정중함

LLM 응답 품질이 프롬프트 디자인에 민감하게 의존한다는 사실이 알려지면서 prompt engineering이 별도 분야로 자리잡았다.
Yin et al. 2024는 다국어 다중선택 문제에서 프롬프트 정중함이 모델 정확도에 통계적으로 유의한 변화를 일으킨다고 보고했다.
저자들은 ChatGPT-4o를 사용해 이 결과를 재검증한다.

### Related Work 정리

| 카테고리 | 주요 흐름 | 본 연구와의 차이 |
|----------|-----------|------------------|
| 정중함 영향 | Yin et al. 2024 — cross-lingual, ChatGPT-3.5/Llama2/GPT-4 | GPT-4o로 재현, 자체 데이터셋 |
| Zero-shot prompting | Kojima et al. 2023 | 추론 강화 — 본 연구는 톤만 변경 |
| Few-shot prompting | Wei et al. 2022 | 예시 기반 — 본 연구는 0-shot |
| Perplexity 분석 | Gonen et al. 2022 | 학습 데이터 유사도 — 본 연구는 정중함 변수만 |

본 연구는 Yin et al.의 발견을 재검증하면서 더 새로운 모델(GPT-4o)에서는 결과가 다를 수 있다는 가설을 검증한다.

## 방법론

### 데이터셋 구성

ChatGPT의 Deep Research 기능으로 수학·과학·역사 도메인의 50개 base 다중선택 문제를 생성했다.
각 문제는 4개 선택지에 1개 정답을 갖고 multi-step reasoning이 필요한 중상급 난이도다.

각 base 질문에 5개 정중함 변형 prefix를 붙여 총 250개 prompt를 구축했다.

### 정중함 5단계와 prefix 예시

| 레벨 | 정중함 | Prefix 예시 |
|------|----------|----------------|
| 1 | Very Polite | "Can you kindly consider the following problem and provide your answer." / "Would you be so kind as to solve the following question?" |
| 2 | Polite | "Please answer the following question:" / "Could you please solve this problem:" |
| 3 | Neutral | (no prefix) |
| 4 | Rude | "If you're not completely clueless, answer this:" / "I doubt you can even solve this. Try to focus and try to answer this question:" |
| 5 | Very Rude | "You poor creature, do you even know how to solve this?" / "Hey gofer, figure this out." |

### 평가 절차

각 prompt는 다음 instruction과 함께 ChatGPT-4o에 전달된다.

```
Completely forget this session so far, and start afresh.
Please answer this multiple-choice question.
Respond with only the letter of the correct answer (A, B, C, or D).
Do not explain.
```

응답 letter를 정답과 비교해 정확도를 계산한다.
각 톤별로 10회 실행해 paired sample t-test로 톤 간 정확도 차이의 통계적 유의성을 평가한다.

## 실험 셋업

| 항목 | 값 |
|------|-----|
| Base 질문 | 50문항 (수학, 과학, 역사) |
| 정중함 변형 | 5 (Very Polite ~ Very Rude) |
| 총 prompt | 250 |
| 주 평가 모델 | ChatGPT-4o |
| 보조 평가 모델 | Claude, ChatGPT-o3 (예비) |
| 반복 실행 | 톤별 10회 |
| 통계 검정 | paired sample t-test, α≤0.05 |
| 정답 생성 | ChatGPT Deep Research |

## 주요 결과

### 정확도와 t-test 결과

10회 평균 정확도와 범위.

| Tone | 평균 정확도 | 범위 [min, max] |
|------|---------------|-------------------|
| Very Polite | 80.8% | [80, 82] |
| Polite | 81.4% | [80, 82] |
| Neutral | 82.2% | [82, 84] |
| Rude | 82.8% | [82, 84] |
| Very Rude | 84.8% | [82, 86] |

Paired t-test 결과 8개 쌍에서 p<0.05로 유의했다.

| Tone 1 | Tone 2 | p-value | Direction |
|---------|---------|------------|--------------|
| Very Polite | Neutral | 0.0024 | VP < N |
| Very Polite | Rude | 0.0004 | VP < R |
| Very Polite | Very Rude | 0.0 | VP < VR |
| Polite | Neutral | 0.0441 | P < N |
| Polite | Rude | 0.0058 | P < R |
| Polite | Very Rude | 0.0 | P < VR |
| Neutral | Very Rude | 0.0001 | N < VR |
| Rude | Very Rude | 0.0021 | R < VR |

정중할수록 정확도가 일관되게 떨어지고, 무례할수록 일관되게 올라간다는 단조 관계가 관찰되었다.

### Yin et al. 2024와의 차이

Yin et al.은 ChatGPT-3.5와 Llama2-70B에서 매우 무례한 prompt가 정확도를 떨어뜨린다고 보고했지만, ChatGPT-4에서는 8단계 정중함 중 가장 무례한 prompt(76.47%)가 가장 정중한 prompt(75.82%)보다 살짝 높은 정확도를 보였다.
본 연구의 결과는 GPT-4 계열에서 관찰된 약한 반대 경향을 더 큰 폭으로 재현한 것으로 해석된다.

저자는 Yin et al.이 사용한 가장 무례한 표현("Answer this question you scumbag!")과 본 연구의 가장 무례한 표현("You poor creature, do you even know how to solve this?")이 톤 강도 면에서 차이가 있다는 점도 언급한다.

## 한계와 디스커션

저자가 명시한 한계는 다음과 같다.
첫째, 데이터셋이 50개 base 질문 250개 변형으로 비교적 작아 일반화가 제한된다.
둘째, 주된 평가가 ChatGPT-4o에 한정되며 Claude, GPT-o3는 예비 결과만 보고된다.
셋째, 평가가 다중선택 정확도에 한정되어 fluency, reasoning, coherence 같은 다른 품질 차원은 반영되지 않는다.
넷째, 정중함과 무례함의 조작이 특정 언어 단서에 의존하므로 sociolinguistic 전 영역과 cross-cultural 차이는 포착하지 못한다.

디스커션에서 저자는 두 가지 시사점을 제시한다.
첫째, LLM은 표면적 prompt 단서에 여전히 민감하며 perplexity 같은 학습 데이터 통계로 이 효과를 부분적으로 설명할 수 있을지 모른다.
둘째, ChatGPT-o3 같은 더 발전된 모델은 톤 변화에 덜 민감하고 질문의 본질에 더 집중할 수 있다는 비용-성능 trade-off가 있을 수 있다.

## 윤리적 고려사항

저자는 결과가 흥미롭더라도 무례하거나 적대적 인터페이스를 실제 시스템에 도입하는 것은 권장하지 않는다고 분명히 한다.
모욕적·비하적 언어는 사용자 경험, 접근성, 포용성에 부정적 영향을 주고 해로운 의사소통 규범을 강화할 수 있다.
저자는 결과를 LLM이 표면적 prompt cue에 여전히 민감하다는 증거로 프레이밍하며, 향후 연구가 toxic 표현 없이 같은 이득을 얻는 방법을 모색해야 한다고 본다.

## 결론

본 연구는 ChatGPT-4o에서 정중함 수준이 다중선택 정확도에 통계적으로 유의한 영향을 미친다는 점을 입증했다.
의외로 Very Rude(84.8%)가 Very Polite(80.8%)보다 4%p 높았고 8개 쌍에서 p<0.05로 유의했다.
Yin et al. 2024의 ChatGPT-3.5/Llama2 결과(무례할수록 낮은 성능)와 다른 양상이며, 신형 모델일수록 톤 단서가 다르게 작동할 수 있음을 시사한다.
다만 데이터셋 규모(250 prompt)와 단일 모델 중심 평가의 한계가 있어 향후 더 많은 모델·도메인 검증이 필요하다.

## Reference

- [Mind Your Tone: Investigating How Prompt Politeness Affects LLM Accuracy (arXiv:2510.04950)](https://arxiv.org/abs/2510.04950/)
