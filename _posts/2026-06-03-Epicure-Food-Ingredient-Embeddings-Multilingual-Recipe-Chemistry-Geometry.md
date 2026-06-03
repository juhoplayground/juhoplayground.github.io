---
layout: post
title: "Epicure - 414만 다국어 레시피와 화학 정보로 학습한 식재료 임베딩 3종"
author: 'Juho'
date: 2026-06-03 00:00:00 +0900
categories: [AI]
tags: [Embedding, AI, Documentation]
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
2. [기존 연구의 한계](#기존-연구의-한계)
3. [방법론](#방법론)
   - [코퍼스](#코퍼스)
   - [정규화 어휘](#정규화-어휘)
   - [그래프 구성](#그래프-구성)
   - [세 가지 Epicure 모델](#세-가지-epicure-모델)
4. [평가 결과](#평가-결과)
   - [감독 학습 방향 품질](#감독-학습-방향-품질)
   - [내재적 기하학](#내재적-기하학)
   - [창발적 기하학](#창발적-기하학)
5. [한계와 주의사항](#한계와-주의사항)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

KAIKAKU.AI의 Jakub Radzikowski와 Josef Chen이 2026년 5월 21일 arXiv에 공개한 Epicure는 식재료에 대한 임베딩 모델 패밀리다.
414만 개 다국어 레시피와 FlavorDB 화학 정보를 결합해, 동일한 아키텍처와 하이퍼파라미터를 공유하면서 walk schema만 다른 세 가지 변형(Cooc, Core, Chem)을 학습했다.
요리 인공지능 도구를 위한 임베딩 기반이며, 화학 신호와 레시피 컨텍스트 신호의 혼합을 설계 축으로 노출한다는 점이 핵심 기여다.

## 기존 연구의 한계

기존 대표 모델인 FlavorGraph는 FlavorDB 화학 정보와 Recipe1M+ 공기 정보(co-occurrence)를 하나의 Metapath2Vec 모델로 결합했고, 6,653개 식재료와 1,645개 화합물로 구성된 가장 포괄적인 공개 식품 임베딩이었다.
하지만 세 가지 제약이 있었다.

| 제약 | 설명 |
|------|------|
| 단일 영어 코퍼스 | Recipe1M+ 기반으로 영어 중심 |
| 고정된 신호 혼합 | 화학과 레시피 컨텍스트가 inductive bias로 고정 |
| 분산된 어휘 | 조리 단계와 비식품 항목까지 포함 |

Epicure는 이 세 제약을 동시에 풀어낸다.

## 방법론

파이프라인은 다섯 단계로 구성된다.
다국어 레시피 코퍼스 집계, NER 용어의 정규화 어휘 매핑, 공기 그래프 및 typed compound 그래프 구성, 세 가지 Metapath2Vec 변형 학습, 그리고 임베딩 분석이다.

### 코퍼스

11개 공개 데이터셋에서 7개 언어를 가로지르는 4,135,189개 레시피를 집계했다.

| 언어 | 비중 |
|------|------|
| 영어 RecipeNLG | 53.9% |
| 중국어 XiaChuFang | 37.4% |
| 러시아어 Povarenok | 3.5% |
| 기타 다국어 (Vietnamese, Spanish, Turkish, Indonesian, German, Indian-English) | 나머지 |

비영어 식재료 용어는 Claude Opus(내부 ID 4.6) 패밀리로 결정론적으로 영어로 기계 번역했다.
최종 1,790 canonical 어휘와의 교집합 결과 4,103,118개 레시피(99.2%)가 최소 하나의 매칭 식재료를 포함했다.

### 정규화 어휘

NER 추출 결과는 약 200,000개 고유 식재료 문자열로, 철자 변형, 브랜드명, 비식품, 조리 modifier가 섞여 있었다.
LLM 기반 정규화 파이프라인(Claude Opus 4.6, Gemini Embedding `gemini-embedding-001`)을 통해 최종 1,790개 canonical 식재료로 축소했다.
이 중 523개는 FlavorDB의 typed I-C 엣지를 보유한 chemical-hub 식재료, 나머지 1,267개는 non-hub다.

### 그래프 구성

세 모델은 동일한 1,790개 식재료 노드와 203,508개 NPMI 공기 엣지를 공유한다.
Cooc 그래프는 순수 식재료-식재료 공기 그래프이며, Core/Chem 그래프는 추가로 2,247개 typed compound 노드와 80,019개 typed I-C 엣지를 가진다.
화합물은 balsamic, citrus, earthy 등 15개 향기 카테고리 중 하나 이상을 가지며, 카테고리별로 복제되어 Metapath2Vec의 typed walk가 같은 카테고리 다리(citrus-citrus)와 다른 카테고리 다리(citrus-earthy)를 구분할 수 있게 한다.

### 세 가지 Epicure 모델

세 모델은 모든 학습 하이퍼파라미터를 공유한다.

| 항목 | 값 |
|------|-----|
| 임베딩 차원 | 300 |
| Walks per node | 100 |
| Walk length | 50 |
| Context window | 7 |
| Negative samples | 5 |
| Batch size | 32,768 |
| Learning rate | 0.0025 |
| 옵티마이저 | SparseAdam |
| Epochs | 20 |

차이는 walk schema 한 가지뿐이다.

| 모델 | Walk 설계 |
|------|----------|
| Epicure-Cooc | Cooc 그래프만, NPMI 가중 I-I 랜덤 워크 |
| Epicure-Core | Typed compound 그래프 + ii_repeat=10 (혼합) |
| Epicure-Chem | Typed compound 그래프, I-I 워크 없음 (화학 극단) |

세 모델은 화학-vs-레시피 컨텍스트 스펙트럼을 단일 실험 설계 안에서 추적할 수 있게 한다.

## 평가 결과

### 감독 학습 방향 품질

27개 연속 probe와 8개 cuisine 매크로 지역을 5-fold 교차 검증으로 평가했다.
Cuisine 분리에 대한 평균 Cohen's d는 다음과 같다.

| 모델 | Cohen's d |
|------|-----------|
| Cooc | 2.43 |
| Core | 2.70 |
| Chem | 3.07 |

화학 극단인 Chem이 가장 높은 cuisine 분리도를 보였다는 점은, 화학 정보가 요리 전통을 잘 구분하는 데 기여한다는 해석을 뒷받침한다.

### 내재적 기하학

세 모델의 등방성(isotropy)은 sharply 다르며, Cooc은 isotropy가 높고 Chem은 집중된 기하를 가진다.
이는 입력 데이터가 아니라 walk schema의 결과다.

### 창발적 기하학

food-group residualised 임베딩에 multi-seed FastICA를 적용해 모델당 20개 해석 가능한 factor를 회수한 뒤, GMM partition으로 factor마다 150-200개의 명명된 culinary mode를 얻었다.

| 모델 | mean coherence | random-pair baseline |
|------|----------------|---------------------|
| Cooc | 0.611 | 0.097 |
| Core | 0.833 | 0.348 |
| Chem | 0.703 | 0.115 |

Core 모델이 화학과 레시피 컨텍스트를 균형 있게 결합해 가장 높은 mode coherence를 보인다.

또한 두 가지 operator가 같은 300차원 공간 위에서 동작한다.
하나는 nearest-neighbour pairings(top-K, mode-membership lookup)이고, 다른 하나는 SLERP direction arithmetic으로 seed를 supervised pole 벡터(예: rice + South-Asian → curry leaf, urad dal, chana dal, fenugreek seed) 또는 emergent factor-mode pole로 회전시킨다.
연속 각도 θ가 retrieval을 seed 우세에서 target 우세로 보간한다.

## 한계와 주의사항

- 일부 언어는 기계 번역 의존도가 높아 미묘한 의미가 손실될 수 있다.
- 1,790 canonical 어휘로 축소되면서 매우 지역적인 식재료가 누락될 수 있다.
- 비공개 자원: 논문은 코드와 학습된 artifact를 현재 시점에서 공개하지 않는다고 명시한다.
- Cuisine 매크로 지역 8개는 실용적 단순화이며, 세분화된 지역 차이는 다른 grouping이 필요하다.

## 결론

Epicure는 식재료 임베딩을 "단일 설계 결정"이 아니라 "화학과 레시피 컨텍스트의 가중치를 walk schema로 조절하는 설계 축"으로 다룬다.
세 자매 모델은 같은 300차원 공간 위에서 nearest-neighbour 추천과 SLERP 기반 방향 산술을 동시에 지원하며, 셰프용 도구가 명시적 supervised 방향(예: cuisine, 식이군)과 emergent factor를 자유롭게 회전·혼합·검색할 수 있게 한다.
414만 다국어 레시피와 LLM 기반 정규화 파이프라인이 결합되어, FlavorGraph 이후 가장 체계적으로 다국어 화학-요리 임베딩을 다룬 작업으로 평가할 수 있다.

## Reference

- [Epicure: Navigating the Emergent Geometry of Food Ingredient Embeddings - arXiv 2605.22391](https://arxiv.org/abs/2605.22391)
