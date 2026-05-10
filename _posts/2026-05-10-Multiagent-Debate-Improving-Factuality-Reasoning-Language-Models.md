---
layout: post
title: "Multiagent Debate - 다중 에이전트 토론으로 LLM 사실성과 추론 향상"
author: 'Juho'
date: 2026-05-10 00:00:00 +0900
categories: [LLM]
tags: [LLM, AI, Agent]
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
   - [단일 인스턴스 추론 기법의 한계](#단일-인스턴스-추론-기법의-한계)
   - [Society of Minds와 다중 에이전트 토론](#society-of-minds와-다중-에이전트-토론)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [토론 절차](#토론-절차)
   - [프롬프트 구조와 응답 집계](#프롬프트-구조와-응답-집계)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [추론 태스크 정확도](#추론-태스크-정확도)
   - [사실성 태스크 정확도](#사실성-태스크-정확도)
   - [에이전트 수와 라운드 수 ablation](#에이전트-수와-라운드-수-ablation)
   - [이종 모델 토론과 실패 사례 회복](#이종-모델-토론과-실패-사례-회복)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"Improving Factuality and Reasoning in Language Models through Multiagent Debate"는 동일한 LLM의 여러 인스턴스가 다중 라운드의 토론을 거치면서 답을 정제하는 기법을 제안한 논문이다.
저자는 Yilun Du, Shuang Li, Antonio Torralba, Joshua B. Tenenbaum, Igor Mordatch이며, 주된 실험은 gpt-3.5-turbo-0301에서 수행되었다.

논문 abstract의 요지는 다음과 같다.
프롬프팅, 검증(verification), self-consistency, 중간 스크래치패드 등 단일 모델 인스턴스 위에서 작동하는 기존 개선 기법과 달리, 본 논문은 다수의 LLM 인스턴스가 자신의 응답과 추론 과정을 제시하고 여러 라운드에 걸쳐 토론하여 공통의 최종 답에 도달하게 하는 보완적 접근법을 제안한다.
모델 학습이나 가중치 변경 없이 동일한 프롬프트 패턴만으로 추론 정확도와 사실성을 동시에 끌어올리는 것이 핵심이다.

## 배경과 선행 연구

### 단일 인스턴스 추론 기법의 한계

저자들은 LLM이 환각(hallucination)을 자신 있게 만들어 내거나, 추론 사슬에서 비현실적인 도약을 보이는 문제를 출발점으로 삼는다.
이 문제를 완화하기 위해 zero-shot chain-of-thought, intermediate scratchpad, verification, self-consistency, self-reflection 등이 제안되어 왔다.
그러나 저자들은 이런 기법이 모두 단일 모델 인스턴스 위에서 작동한다는 공통의 제약을 가진다는 점을 지적한다.
즉 한 개의 사고 흐름 안에서 자신의 답을 다듬는 데 한정되며, 모델이 첫 응답에서 잡힌 잘못된 가정에 갇혀 있을 경우 이를 깨고 나오기 어렵다.

### Society of Minds와 다중 에이전트 토론

본 논문은 Marvin Minsky의 The Society of Mind에서 영감을 얻어, 분산된 사고 흐름들이 서로 의견을 교환할 때 더 견고한 답이 나온다는 관점을 LLM에 도입한다.
다수의 동일한 LLM 인스턴스를 별도의 "agent"로 취급하고, 라운드를 거치며 서로의 응답을 입력으로 받아 자신의 답을 갱신하게 한다.
이 구조는 본질적으로 단일 인스턴스의 self-reflection과 다르다.
self-reflection은 자기 출력을 자기 입력으로 다시 넣는 자기 참조 루프인 반면, debate는 다른 인스턴스의 독립적 추론 경로를 외부 신호로 활용한다.

### Related Work 정리

논문이 정리하는 관련 연구는 크게 세 갈래다.

| 카테고리 | 핵심 흐름 | 본 연구와의 차이 |
|----------|-----------|------------------|
| Reasoning and Factuality in LMs | chain-of-thought, intermediate self-reflection, finetuning, RLHF, 외부 지식 retrieval | 단일 인스턴스 위에서 작동, 학습 또는 외부 지식 필요 |
| Multi-Agent / Debate | Irving et al.의 debate 절차 (안전성과 정확도 검증) | Irving은 인간 심판 기반, 본 논문은 모델끼리만 토론하여 정답을 결정 |
| Compositional Generation | Li et al. 등 다중 모델을 결합한 멀티모달 추론 | 모델 결합이 아닌 동일 모델 다중 인스턴스의 토론에 초점 |

세 갈래 모두에서 본 논문의 차별점은 학습 없이, 인간 심판 없이, 동일 모델 인스턴스만으로 정확도와 사실성을 동시에 끌어올린다는 점이다.

## 방법론

### 토론 절차

N개의 동일한 LLM 인스턴스가 같은 질문에 독립적으로 답을 낸 뒤, 다음 라운드에서 다른 인스턴스의 응답을 보고 자신의 답을 갱신하는 절차를 R회 반복한다.
주된 실험 구성은 N=3 에이전트, R=2 라운드이며, 라운드가 끝나면 다수결 또는 최종 라운드의 답변을 정답으로 채택한다.

| 구분 | 단일 응답 | Self-Reflection | Multiagent Debate |
|------|-----------|------------------|-------------------|
| 응답 주체 | 단일 LLM 1회 forward | 단일 LLM 자기 재검토 | N개 인스턴스 R회 토론 |
| 정보 통합 | 없음 | 자기 출력 재입력 | 타 인스턴스 응답 입력 |
| 학습 필요 | 없음 | 없음 | 없음 |
| 외부 자원 | 없음 | 없음 | 없음 |

### 프롬프트 구조와 응답 집계

다음 라운드의 프롬프트에는 직전 라운드의 다른 에이전트 응답이 함께 주입된다.
논문은 두 가지 변형을 비교한다.
짧은 프롬프트는 "Based off the opinion of other agents, can you give an updated response..." 형태이고, 긴 프롬프트는 "Using the opinion of other agents as additional advice..." 형태다.

응답 집계 방식은 두 가지가 있다.
N이 작을 때는 다른 에이전트 응답을 그대로 이어붙이는 직접 concatenation이 사용된다.
N이 5 이상으로 커지면 컨텍스트가 폭증하므로 ChatGPT로 다른 에이전트 응답을 요약한 뒤 주입하는 summarization 방식이 도입된다.

## 실험 셋업

모든 실험은 gpt-3.5-turbo-0301 모델에서 수행되었다.
주된 구성은 N=3 에이전트, R=2 라운드이며, 데이터셋별 표본 규모는 다음과 같다.

| 데이터셋 | 평가 표본 수 | 평가 형식 |
|----------|--------------|-----------|
| Arithmetic | 100문항 | 두 자리 정수 6개에 +, ×, − 연산을 섞은 식 |
| GSM8K | 100문항 | 초등 수학 서술형 |
| 체스 다음 수 예측 | 300국 | 14수 시점 ΔPS(centipawn 차이) |
| Biographies | 524명 | 컴퓨터 과학자 약력의 사실 정확도 |
| MMLU | 100문항 | 객관식 일반 지식 |
| 체스 합법 수 (Chess Validity) | 100문항 | 합법적 다음 수 생성 |

논문은 temperature 등 sampling 파라미터를 본문에서 명시하지 않으며, 결과는 위 표본을 기반으로 산출된다.

## 주요 결과

### 추론 태스크 정확도

| 태스크 | Single Agent | Self-Reflection | Multiagent Debate |
|--------|--------------|------------------|--------------------|
| 산술 | 67.0% | 72.1% | 81.8% |
| GSM8K | 77.0% | 75.0% | 85.0% |
| 체스 다음 수 (ΔPS) | 91.4 | 102.1 | 122.9 |

세 태스크 모두에서 토론 방식이 단일 응답 대비 큰 격차로 우위를 보였고, GSM8K에서는 단일 응답보다 self-reflection이 오히려 떨어지는 현상도 관찰됐다.
이는 자기 출력을 자기 입력으로 재주입하는 self-reflection이 본질적으로 같은 사고 사슬에 갇혀 있는 한계를 시사한다.

### 사실성 태스크 정확도

| 태스크 | Single Agent | Self-Reflection | Multiagent Debate |
|--------|--------------|------------------|--------------------|
| Biographies | 66.0% | 68.3% | 73.8% |
| MMLU | 63.9% | 57.7% | 71.1% |
| Chess Validity | 29.3% | 38.8% | 45.2% |

특히 MMLU에서 self-reflection은 63.9%에서 57.7%로 정확도를 떨어뜨리는 반면, debate는 71.1%까지 끌어올려 자기 재검토와 토론이 본질적으로 다른 메커니즘임을 시사한다.

### 에이전트 수와 라운드 수 ablation

산술 태스크에서 에이전트 수 N을 늘릴수록 정확도가 단조 증가했다.
라운드 수 R도 늘릴수록 성능이 향상되지만, 4라운드 이상부터는 추가 이득이 거의 사라지는 plateau가 관찰된다.
또한 N이 큰 설정에서는 응답 concatenation이 컨텍스트 길이를 잠식하므로, 다른 에이전트의 응답을 ChatGPT로 요약해 주입하는 summarization이 직접 concatenation보다 더 높은 정확도를 보였다.

### 이종 모델 토론과 실패 사례 회복

GSM8K에서 ChatGPT 단독은 20문제 중 14개, Bard 단독은 11개를 맞혔지만, 두 모델을 같은 토론에 참여시키자 17/20으로 상승했다.
또 흥미로운 패턴은 모든 에이전트가 첫 라운드에 동일한 오답을 내놓은 경우에도, 라운드를 거듭하면서 정답으로 수렴하는 사례가 다수 관찰된다는 점이다.
즉 토론은 이미 옳은 답을 선택지에 보유한 다수결로 환원되지 않고, 보편적 초기 오류를 능동적으로 교정하는 효과가 있다.

## 한계와 디스커션

저자가 Limitations and Discussion 절에서 명시한 항목은 세 가지다.
첫째, 다중 라운드 다중 인스턴스 생성으로 인해 단일 응답 대비 연산 비용이 N×R 배 증가한다.
둘째, 토론이 길어질수록 입력 길이가 폭증해 모델이 가장 최근 라운드 응답에만 집중하고 초기 라운드 정보는 사실상 누락되는 현상이 발생한다.
셋째, 토론은 대개 하나의 답으로 수렴하지만 그 수렴 답이 반드시 정답인 것은 아니며, 모델이 자신의 불확실성을 응답에 제대로 반영하지 못하는 한계가 있다.

디스커션에서 저자들이 제시한 향후 방향 중 주목할 점은, 토론 절차로부터 생성된 데이터를 베이스 모델에 다시 distillation하여 모델 자체를 자기 개선시키는 데 활용할 수 있다는 가능성이다.
즉 debate는 추론 시점의 정확도를 끌어올리는 도구일 뿐 아니라, 학습 시점에 사용할 고품질 합성 데이터의 생성기로도 기능할 수 있다.

이러한 한계를 고려하면, 실시간 응답이 중요한 시스템에서는 N과 R을 작게 유지하되 summarization을 도입해 컨텍스트를 압축하는 전략이 현실적이며, 수렴 결과를 그대로 신뢰하기보다 토론 과정 자체를 신뢰도 신호로 함께 활용하는 설계가 권장된다.

## 결론

Multiagent Debate는 N=3, R=2 같은 작은 설정에서도 산술 67.0%에서 81.8%, MMLU 63.9%에서 71.1%, Chess Validity 29.3%에서 45.2% 등 6개 벤치마크 전반에서 일관된 향상을 보였다.
chain-of-thought, self-consistency, self-reflection 같은 단일 인스턴스 기법과 달리, 다른 인스턴스의 독립적 추론 경로를 외부 신호로 활용한다는 점이 핵심 차별점이다.
모델 가중치 변경 없이 블랙박스 LLM에 그대로 적용 가능하며, 이종 모델끼리의 토론에서도 단일 모델 최고 성능을 능가하는 결과를 보여준다.

연산 비용과 컨텍스트 길이라는 트레이드오프는 분명하지만, 토론 데이터를 재활용한 self-distillation이라는 추가 활용 경로까지 고려하면 LLM 에이전트 시스템 설계의 기본 빌딩 블록 중 하나로 자리잡을 만한 기법이다.

## Reference

- [Improving Factuality and Reasoning in Language Models through Multiagent Debate (arXiv:2305.14325)](https://arxiv.org/abs/2305.14325/)
