---
layout: post
title: ChatGPT 사용 시 뇌에 축적되는 인지 부채(Cognitive Debt) - MIT Media Lab 연구
author: 'Juho'
date: 2026-01-31 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Research]
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
   - [LLM과 학습의 인지적 비용](#llm과-학습의-인지적-비용)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [실험 설계와 그룹 구성](#실험-설계와-그룹-구성)
   - [Dynamic Directed Transfer Function](#dynamic-directed-transfer-function)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [Alpha band 연결성](#alpha-band-연결성)
   - [Beta, Delta, Theta band](#beta-delta-theta-band)
   - [세션 4 크로스오버](#세션-4-크로스오버)
   - [에세이 인용 능력과 소유감](#에세이-인용-능력과-소유감)
   - [에너지 비용](#에너지-비용)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

MIT Media Lab의 "Your Brain on ChatGPT: Accumulation of Cognitive Debt when Using an AI Assistant for Essay Writing Task"는 LLM 보조 에세이 작성이 인간 뇌 활동에 미치는 영향을 EEG, NLP, 교사 평가를 종합해 분석한 연구다.
저자는 Nataliya Kosmyna(MIT Media Lab, 교신저자), Eugene Hauptmann, Ye Tong Yuan(Wellesley), Jessica Situ, Xian-Hao Liao(MassArt), Ashly Vivian Beresnitzky, Iris Braunstein, Pattie Maes로 2025년 6월 arXiv에 공개되고 12월 v2로 업데이트되었다.

논문 abstract는 다음과 같은 핵심을 제시한다.
54명 참가자를 LLM 그룹, Search Engine 그룹, Brain-only 그룹으로 나누어 동일 SAT 에세이 과제를 4개월간 4개 세션에 걸쳐 수행하게 했다.
세션 1~3은 동일한 도구를 사용했고, 세션 4는 크로스오버 설계로 LLM 그룹은 도구 없이(LLM-to-Brain), Brain-only 그룹은 LLM을 사용해(Brain-to-LLM) 작성했다.
EEG 동적 연결성 분석 결과 외부 도구 의존도가 클수록 뇌 연결성이 체계적으로 감소했고, LLM 그룹은 4개월에 걸쳐 신경·언어·평가 세 차원 모두에서 일관되게 낮은 성과를 보였다.

## 배경과 선행 연구

### LLM과 학습의 인지적 비용

저자들은 LLM이 학습 환경에서 즉각적 편의성을 제공하는 동시에 cognitive offloading(인지 외주화)을 유발한다는 선행 연구 흐름에서 출발한다.
선행 메타분석은 ChatGPT 활용이 학생의 flow experience, self-efficacy, 학습 성과를 모두 떨어뜨릴 수 있다고 보고했다.
또한 Sparrow et al.의 Google effect 연구는 검색 엔진 의존이 정보 자체보다 정보의 위치만 기억하게 만들어 깊은 인지 처리를 저해한다고 입증했다.

본 연구는 이 흐름을 한 단계 진전시켜, LLM 사용이 단순히 학습 성과를 떨어뜨리는 데 그치지 않고 신경 회로 수준에서 측정 가능한 변화를 일으키는지, 그 변화가 도구 사용 종료 후에도 지속되는지를 EEG로 검증한다.

### Related Work 정리

| 카테고리 | 주요 흐름 | 본 연구와의 차이 |
|----------|-----------|------------------|
| LLM과 학습 | Yang 2025 (학습 성과 저하), Zhang 2024 (AI 의존성) | 행동 평가만 — 본 연구는 EEG 추가 |
| Cognitive Load Theory | Sweller (1988) — 내재·외재·적정 부하 | 본 연구는 LLM이 외재 부하를 줄이지만 적정 부하까지 줄이는지 검증 |
| Web search와 학습 | Sparrow 2011 (Google effect — 위치만 기억) | 검색 vs LLM 직접 대조 |
| Search as Learning | von Hoyer 2022 — 탐색적 검색의 학습 효과 | 검색을 중간 비교군으로 포함 |
| EEG와 글쓰기 | 베타·세타 대역과 working memory 관련 연구 | dDTF로 방향성 있는 연결성 분석 |

## 방법론

### 실험 설계와 그룹 구성

54명 참가자를 다음 세 그룹에 무작위로 배정한다.

| 그룹 | 도구 | 세션 1–3 | 세션 4 |
|------|------|----------|--------|
| LLM (그룹 1) | ChatGPT | LLM 사용 | 도구 없음 (LLM-to-Brain) |
| Search Engine (그룹 2) | Google 검색 | 검색 사용 | 세션 4 미참여 |
| Brain-only (그룹 3) | 도구 없음 | 도구 없음 | LLM 사용 (Brain-to-LLM) |

세션 1–3은 동일 그룹 조건으로 진행되고, 세션 4는 LLM과 Brain-only 그룹만 도구 조건을 교차시켰다.
참가자는 SAT 형식 에세이 9개 토픽 풀에서 세션마다 3개 중 1개를 선택해 20분간 작성한다.
세션 1–3에 54명 전원이 참여했고, 세션 4는 18명이 완주했다(LLM-to-Brain 9명, Brain-to-LLM 9명).

### Dynamic Directed Transfer Function

EEG 연결성은 dDTF(Dynamic Directed Transfer Function)로 분석한다.
dDTF는 다변량 자기회귀 모델에서 추출된 방향성 있는 정보 흐름 지표로, 32개 전극의 모든 쌍 992개에 대해 시간 가변적인 인과적 영향력을 추정한다.
4개 주파수 대역에서 분석된다: alpha(8–13Hz, 의미 처리·내적 주의), beta(13–30Hz, 능동 인지 통제), delta(0.5–4Hz, 대규모 통합 처리), theta(4–8Hz, 작업 기억과 실행 통제).
유의수준 p<0.05의 연결만 보고하며, 그룹 간 차이는 다중비교 보정된 p값으로 평가한다.

## 실험 셋업

| 항목 | 값 |
|------|-----|
| 참가자 | 54명 (세션 1–3), 18명 (세션 4) |
| 모집 지역 | 보스턴 인근 학술 기관 |
| EEG 장비 | Enobio 32채널 헤드셋 |
| 세션 수 | 4 |
| 세션 간격 | 약 4개월 |
| 세션당 작성 시간 | 20분 |
| 토픽 풀 | SAT 형식 9개 |
| 평가 | EEG dDTF + NLP(NER, n-gram, 토픽 온톨로지) + AI 심사관 + 영어 교사 + 사후 인터뷰 |

NLP 분석은 latent space 임베딩 클러스터링, 명명된 개체 인식(NER), n-gram 분포, 토픽 온톨로지로 구성되며, 평가는 별도 학습된 AI agent와 두 명의 인간 영어 교사 점수를 모두 활용한다.

## 주요 결과

### Alpha band 연결성

세션 1–3 누적 데이터에서 LLM 그룹과 Brain-only 그룹 간 가장 두드러진 차이가 alpha 대역에서 발견됐다.

| 연결 | p값 | Brain-only dDTF | LLM dDTF |
|------|-----|------------------|----------|
| P7 → T8 (좌두정 → 우측두) | 0.0002 | 0.053 | 0.009 |
| PO4 → AF3 (두정후두 → 전두) | 0.0025 | 0.024 | 0.009 |
| Oz → Fz (후두 → 정중전두) | 0.003 | 0.020 | 0.10 |

전체 유의 연결 수는 Brain-only 79개, LLM 42개로 큰 격차가 나타났다.
T8과 AF3가 Brain-only 그룹의 주요 수렴 허브로 기능한 반면, LLM 그룹은 fronto-parietal alpha 결합이 약하게 나타나 의미 검색과 창의적 발상의 신경 기반이 LLM 도구로 외주화된 것으로 해석된다.

### Beta, Delta, Theta band

저주파 beta(13–20Hz)에서는 Brain-only가 약한 우위(총 dDTF 2.854 vs 2.653)를, 고주파 beta(20–30Hz)에서는 두 그룹이 균형을 이루었다.
Delta 대역에서는 T7→AF3가 가장 강한 차이(p=0.0002, Brain-only 0.022 vs LLM 0.007)를 보였고 Brain-only 78개 vs LLM 31개의 유의 연결로 집행 통제 네트워크의 우위를 드러냈다.
Theta 대역에서는 Pz→T8 연결(p=0.0012, Brain-only 0.041 vs LLM 0.009)을 비롯해 Brain-only 65개 vs LLM 29개의 유의 연결이 관찰돼 작업 기억 부하 대비 결과로 해석된다.

| 대역 | Brain-only 유의 연결 수 | LLM 유의 연결 수 | 해석 |
|------|--------------------------|--------------------|------|
| Alpha (8–13Hz) | 79 | 42 | 내적 주의·의미 처리 |
| Beta low (13–20Hz) | 우위 (총 2.854) | 2.653 | 능동 인지 통제 |
| Theta (4–8Hz) | 65 | 29 | 작업 기억·실행 통제 |
| Delta (0.5–4Hz) | 78 | 31 | 대규모 통합·집행 |

LLM 그룹은 어느 대역에서도 Brain-only 그룹을 넘어선 유의 연결을 만들지 못했으며, 정보 흐름 방향성도 차이가 컸다.
Brain-only는 후두·측두에서 전두로 향하는 bottom-up 흐름이 강했고, LLM은 전두에서 후두로 향하는 top-down 흐름이 상대적으로 우세했다.
Search Engine 그룹은 전반적으로 Brain-only와 LLM 사이의 중간 수준 연결성을 보였으며 alpha 총 dDTF 0.288로 Brain-only 0.423 대비 약하지만 LLM보다는 강한 결합을 유지했다.

### 세션 4 크로스오버

세션 4의 크로스오버는 LLM 사용의 잔여 효과 검증이 핵심이다.
LLM-to-Brain 그룹(이전 LLM 사용자가 도구 없이 작성)은 alpha와 beta 네트워크의 약한 결합을 그대로 유지했고, n-gram 분석에서 COURAGE, FORETHOUGHT, PERFECT 등 토픽에서 좁은 어휘 집합을 반복 사용하는 패턴을 보였다.
이는 단순 도구 부재가 아닌, 이전 세션에서 형성된 인지 회로의 결손이 도구를 떼도 회복되지 않는다는 점을 의미한다.

반면 Brain-to-LLM 그룹(이전 Brain-only 사용자가 LLM을 처음 사용)은 alpha, beta, theta, delta 모든 주파수 대역에서 directed connectivity가 LLM 그룹의 세션 1–3보다 광범위하게 증가했다.
후두-두정 및 전두 노드의 광범위한 재활성화가 관찰되었고, 자발적 기억 회상 능력도 더 높았다.
이는 사전에 충분한 자기 주도 인지 노력을 거친 사용자가 LLM을 만나면 도구를 통합적으로 활용하는 반면, LLM에 먼저 노출된 사용자는 도구 부재 시 기존 회로 패턴이 회복되지 않음을 시사한다.

### 에세이 인용 능력과 소유감

세션 1 직후 인터뷰에서 자신이 방금 작성한 에세이의 일부를 정확히 인용해 보라는 과제에서, LLM 그룹은 18명 중 15명, 즉 83.3%가 정확한 인용에 실패했다.
Brain-only 그룹과 Search Engine 그룹은 대부분 정확한 인용을 제시했다.

| 그룹 | 인용 실패율 | 자기 소유감 (1=낮음, 5=높음) |
|------|-------------|------------------------------|
| LLM | 83.3% (15/18) | 낮음 |
| Search Engine | 낮음 | 중간 |
| Brain-only | 매우 낮음 | 높음 |

LLM 그룹은 또한 자신이 쓴 에세이에 대한 소유감이 가장 낮았고, AI 심사관과 인간 교사 점수가 높았음에도 글에서 개인성이 결여되어 교사가 AI 작성을 식별하기 쉬운 문체적 단서를 남겼다.

### 에너지 비용

저자는 인지 비용 외에 에너지 비용도 추가로 보고한다.

| 그룹 | 쿼리당 에너지 | 20시간 누적 쿼리 | 총 에너지 (Wh) |
|------|---------------|------------------|-----------------|
| LLM | 0.3 Wh | 600 | 180 |
| Search Engine | 0.03 Wh | 600 | 18 |

LLM 쿼리는 평균 검색 쿼리 대비 약 10배의 에너지를 소비하며, 환경 비용도 cognitive debt 논의와 함께 고려되어야 한다고 강조한다.

## 한계와 디스커션

저자가 명시한 한계는 여섯 가지다.
첫째, 표본 크기가 54명이며 보스턴 인근 학술 기관이라는 단일 지역에 편중된다.
둘째, ChatGPT만 평가했기에 다른 LLM(Claude, Gemini 등)으로의 일반화는 제한적이다.
셋째, 에세이 작성 한 가지 과제만 평가했으므로 다른 과제 유형으로의 일반화에 주의가 필요하다.
넷째, 작성 과제를 idea generation, drafting, revision 등 하위 단계로 분리해 분석하지 않았다.
다섯째, EEG는 spectral power 변화를 다루지 않고 connectivity만 분석했으며 hippocampus 같은 심부 영역의 직접 측정은 어려워 후속 fMRI 연구가 필요하다.
여섯째, 본 논문은 동료 심사를 거치지 않은 preprint이므로 모든 결론은 예비적이다.

디스커션의 핵심 개념은 cognitive debt(인지 부채)다.
LLM 의존이 단기 인지 노력을 줄여 주는 반면, 비판적 평가·창의적 발산·정보 내재화 같은 effortful process를 외주화함으로써 장기적 인지 능력을 갉아먹는다.
LLM-to-Brain 그룹의 좁은 n-gram 반복 패턴은 이 cognitive debt가 단순한 즉시적 수행 저하가 아니라 사고의 다양성과 깊이까지 잠식한다는 증거다.
저자들은 교육 모델 차원에서 학습자가 충분한 자기 주도 인지 노력을 경험한 뒤에 AI 통합을 도입하는 지연 통합 모델(delayed AI integration)을 권장한다.

또한 LLM이 만들어 내는 단일 합성 응답이 검색 엔진의 다양한 관점 노출과 달리 알고리즘적으로 편향된 echo chamber를 강화할 수 있다는 점, 데이터셋이 점차 AI 생성 콘텐츠로 오염되는 가운데 인간 작성 fingerprint 데이터의 가치가 커지고 있다는 점도 함께 논의된다.

## 결론

54명 4개월 종단 연구에서 EEG dDTF는 외부 도구 의존도가 커질수록 뇌 연결성이 모든 주파수 대역에서 감소함을 입증했다.
가장 두드러진 신호는 LLM 그룹의 alpha 79 vs 42, theta 65 vs 29, delta 78 vs 31 같은 유의 연결 격차와 P7→T8 dDTF 0.053 vs 0.009 같은 개별 경로 차이다.
83.3%가 자신이 방금 쓴 에세이를 정확히 인용하지 못했다는 행동 결과는 LLM 사용이 정보 내재화를 어떻게 약화시키는지를 단적으로 보여 준다.
세션 4의 크로스오버는 cognitive debt가 도구를 끄고도 회복되지 않는 잔여 효과로 이어진다는 점을 신경·언어 두 차원에서 입증해, 교육 현장에서 AI 통합 시점과 방식의 재고가 필요함을 시사하는 연구다.

## Reference

- [Your Brain on ChatGPT (arXiv:2506.08872)](https://arxiv.org/abs/2506.08872/)
- [Project Website - Brain on LLM](https://www.brainonllm.com/)
