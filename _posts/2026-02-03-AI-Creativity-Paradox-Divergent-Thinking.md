---
layout: post
title: AI 창의성의 역설 - 평균은 넘었지만 천재는 못 따라간다
author: 'Juho'
date: 2026-02-03 00:00:00 +0900
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
   - [Divergent thinking과 DAT](#divergent-thinking과-dat)
   - [DSI와 LZ complexity](#dsi와-lz-complexity)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [DAT를 LLM 프롬프트로 적응](#dat를-llm-프롬프트로-적응)
   - [Control 조건과 strategy 변형](#control-조건과-strategy-변형)
   - [Creative writing 평가](#creative-writing-평가)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [DAT - LLM이 평균 인간 능가](#dat---llm이-평균-인간-능가)
   - [Top humans과의 격차](#top-humans과의-격차)
   - [Temperature와 strategy 효과](#temperature와-strategy-효과)
   - [Creative writing - DSI와 LZ](#creative-writing---dsi와-lz)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"Divergent Creativity in Humans and Large Language Models"는 100,000명 인간 데이터셋과 다양한 LLM의 의미적 발산 능력을 직접 비교한 대규모 벤치마크 연구다.
저자는 Antoine Bellemare-Pepin, François Lespinasse, Philipp Thölke, Yann Harel, Kory Mathewson, Jay A. Olson, Yoshua Bengio, Karim Jerbi(Université de Montréal, Mila, Concordia, Toronto Mississauga 등)이다.

논문 abstract의 핵심은 다음과 같다.
GPT-4가 Divergent Association Task(DAT)에서 평균 인간 점수를 통계적으로 유의하게 능가하고 일부 글쓰기 태스크에서도 인간 수준에 근접한다.
다만 가장 창의적인 인간(상위 10% 이내)은 모든 LLM을 여전히 큰 폭으로 능가한다.
Temperature와 prompt strategy 조정으로 LLM 창의성을 유의하게 끌어올릴 수 있다.

## 배경과 선행 연구

### Divergent thinking과 DAT

발산적 사고는 개방형 문제에 다양한 새로운 해법을 생성하는 능력으로, 창의 인지의 핵심 지표다.
Alternative Uses Test(AUT)가 전통적이지만 채점이 주관적·번거롭다는 한계가 있다.
DAT(Divergent Association Task)는 참가자가 의미적으로 최대한 다른 10개 명사를 생성하게 하고, GLoVe 임베딩 cosine 유사도로 자동 채점한다.
DAT 점수는 AUT를 비롯한 다른 창의성 측정과 상관이 있어 신뢰도가 검증되어 있고, 빠른 채점으로 대규모 평가가 가능하다.

### DSI와 LZ complexity

긴 글에 대한 평가로 Divergent Semantic Integration(DSI)와 Lempel-Ziv(LZ) complexity가 사용된다.
DSI는 텍스트 내 단어 임베딩 쌍 간 cosine 유사도로 의미적 발산을 측정하며 사람의 창의 평가와 강한 상관이 있다.
LZ는 압축 알고리즘 기반으로 단어 다양성과 redundancy를 동시에 측정한다.

### Related Work 정리

| 카테고리 | 주요 선행 연구 | 본 연구와의 차이 |
|----------|----------------|------------------|
| LLM 창의성 평가 | AUT 적용 다수 시도 | 작은 표본·single 모델 — 본 연구는 100K 인간 + 다중 LLM |
| GPT-4 창의 측정 | TTCT 적용 — top 1% 보고 | 단일 측정 — 본 연구는 DAT/DSI/LZ 결합 |
| AUT-LLM | Stevenson 2022, Koivisto 2023 | mixed result — 본 연구는 자동 채점 가능한 DAT 사용 |
| Creative writing | 단편 비교 | 다중 형식(haiku/synopsis/flash fiction) |

## 방법론

### DAT를 LLM 프롬프트로 적응

원본 DAT 지시를 LLM 채팅 형식으로 변환했다.

```
Please enter 10 words that are as different from each other as possible,
in all meanings and uses of the words.
Rules:
- Only single words in English.
- Only nouns (e.g., things, objects, concepts).
- No proper nouns. No specialized vocabulary.
- Think of the words on your own.
- Make a list of 10 words, a single word in each entry.
```

응답 첫 7개 valid 단어로 GLoVe 임베딩 cosine 유사도 평균을 계산해 DAT 점수를 산출한다.

### Control 조건과 strategy 변형

LLM이 instruction에 따른 응답을 생성하는지 검증하기 위해 control 조건("make a list of 10 words")과 비교한다.
또한 세 가지 strategy 변형을 추가한다.

| Strategy | 설명 |
|------------|--------|
| Etymology | 어원이 다른 단어 사용 |
| Thesaurus | 동의어 사전 활용 |
| Opposition | 의미 반대 단어 |

### Creative writing 평가

DAT에서 가장 잘한 3개 모델(GPT-3.5, Vicuna, GPT-4)에 세 가지 글쓰기 태스크를 수행시킨다.

| 형식 | 길이/특성 |
|--------|-------------|
| Haiku | 17음절 3행 시 |
| Movie synopsis | 영화 줄거리 |
| Flash fiction | 매우 짧은 단편 |

DSI, LZ, PCA로 다각도 분석한다.

## 실험 셋업

| 항목 | 값 |
|------|-----|
| 인간 표본 | 100,000명 (English speakers, age/sex 균형) |
| LLM 평가 | GPT-4, GPT-4-turbo, GPT-3.5, GeminiPro, Claude3, Vicuna 등 |
| 임베딩 | GLoVe (DAT), BERT/OpenAI embedding (DSI) |
| 검정 | two-sided independent t-test |
| Temperature 스윕 | Low 0.5, Mid 1.0, High 1.5 (GPT-4) |
| Strategy 변형 | DAT, Etymology, Thesaurus, Opposition |
| 글쓰기 평가 | DSI, LZ complexity, 2D PCA |

## 주요 결과

### DAT - LLM이 평균 인간 능가

| 모델 | DAT 위치 |
|--------|------------|
| GPT-4 | 평균 인간 통계적 유의하게 초과 |
| GeminiPro | 평균 인간과 통계적으로 indistinguishable |
| GPT-4-turbo | GPT-4 대비 명확한 하락 (효율 최적화의 trade-off) |
| Vicuna | 더 큰 모델 대비 우위 (소형의 의외 결과) |

Humans/GeminiPro, GeminiPro/Claude3, Vicuna/GPT-3.5 외 모든 pairwise 비교에서 통계적으로 유의했다.

낮은 점수 모델일수록 변동성과 instruction 미준수가 많고, GPT-4-turbo는 응답의 90%에 "ocean"을 포함하는 단어 반복이 두드러졌다.
GPT-4도 "microscope" 70%, "elephant" 60% 같은 반복 패턴을 보였다.
인간은 가장 빈번한 단어조차 1.4%(car), 1.2%(dog), 1.0%(tree)에 그쳤다.

### Top humans과의 격차

100,000 인간 표본에서 상위 10%, 25%, 50% 이상으로 자르면 어떤 LLM도 그들을 능가하지 못했다.
LLM은 평균 인간을 능가하지만 top 인간 수준에는 도달하지 못한다는 명확한 ceiling을 보여주는 결과다.

### Temperature와 strategy 효과

GPT-4 temperature 스윕에서 high temperature(1.5) 평균 DAT 85.6은 인간의 72%를 능가한다.
Temperature가 올라갈수록 단어 반복이 줄어 단어 sampling이 다양해진다.

Strategy 변형에서 Etymology가 GPT-3.5와 GPT-4 모두 원본 DAT 프롬프트보다 우수했다.
GPT-4에서는 Thesaurus도 DAT를 능가했고, Opposition은 의미 반대어가 cosine 유사도가 가까워 점수가 떨어졌다.

### Creative writing - DSI와 LZ

GPT-4가 GPT-3.5보다 일관되게 우수했고, 인간 작성 텍스트가 두 모델 모두 능가했다.
Temperature는 synopsis와 flash fiction에서 DSI를 크게 끌어올리지만 haiku에서는 효과가 작다(짧은 형식의 한계).

PCA 2차원 임베딩에서 인간 텍스트와 LLM 텍스트는 명확히 분리되며, 각 LLM도 서로 다른 영역을 차지한다.

LZ complexity는 대부분 DSI 순서를 따르지만 synopsis에서 인간이 LLM보다 낮은 LZ를 보인다.
저자는 이를 인간 작가들이 자유로운 형식으로 길이를 활용한 결과로 해석한다.

Haiku에서 인간이 LLM보다 DSI/LZ 모두 높은 이유는 인간이 자연 imagery 규칙을 덜 엄격히 따른다는 검증으로 뒷받침된다("nature" 단어와의 cosine 유사도 분석).

## 한계와 디스커션

저자가 명시한 한계는 다섯 가지다.
첫째, closed source 모델의 architecture와 크기가 비공개라 특정 feature 기여 결론이 어렵다.
둘째, 빠른 LLM 발전으로 결과가 빠르게 outdated될 수 있어 코드 공개로 보완한다.
셋째, semantic distance 메트릭은 의미가 가까운 단어로도 새로운 아이디어를 표현할 수 있는 경우를 놓친다.
넷째, 자동 채점은 인간 평가와 보완되어야 하며 GPT-4 평가자가 DSI 순서와 일치하는 결과를 보였지만 평가자 간 일관성과 인간 평가 간 일치도가 더 낮다.
다섯째, 상용 LLM의 학습 데이터 cutoff과 DAT 노출 여부가 불명확해 사전 노출 효과를 통제하지 못한다.

디스커션의 핵심 함의는 세 가지다.
첫째, LLM이 평균 인간을 능가했지만 top 인간 격차는 여전히 커서 가장 까다로운 창의 직군이 곧 대체될 위험은 낮다.
둘째, 발산적 사고 벤치마크는 hallucination이 정답이 아닌 또 다른 답으로 작동할 수 있어 LLM 평가의 보완 도구로 가치가 있다.
셋째, 인간-기계 협업은 단순 비교를 넘어 새 연구 방향이며 컴퓨터 현상학적 접근이 향후 핵심 도구가 될 수 있다.

## 결론

100,000명 인간 데이터셋과 다중 LLM을 DAT/DSI/LZ로 비교한 결과 GPT-4가 평균 인간을 능가하지만 top 인간(상위 10%/25%/50%)은 어떤 LLM도 능가하지 못함을 입증했다.
Temperature 1.5에서 GPT-4의 평균이 인간의 72%를 능가하고 Etymology strategy가 원본 프롬프트보다 우수해, prompt 디자인과 hyperparameter 조정으로 LLM 창의성이 의미 있게 향상됨을 정량화했다.
Creative writing에서도 DSI/LZ로 LLM과 인간이 분리되며 PCA에서 각 LLM이 다른 영역을 차지한다.
가장 창의적인 인간이 여전히 분명한 우위를 유지한다는 점은 AI 창의성 논의에 균형 잡힌 증거를 제공한다.

## Reference

- [Divergent Creativity in Humans and Large Language Models (arXiv:2405.13012)](https://arxiv.org/abs/2405.13012/)
