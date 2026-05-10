---
layout: post
title: "Trie 기반 빔 서치 - LLM 디코딩의 메모리와 속도를 동시에 잡다"
author: 'Juho'
date: 2026-03-23 00:00:00 +0900
categories: [LLM]
tags: [LLM, AI]
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
   - [배치 빔 서치의 메모리 병목](#배치-빔-서치의-메모리-병목)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [Trie 기반 디코딩 핵심 아이디어](#trie-기반-디코딩-핵심-아이디어)
   - [Trie attention mask와 분기 격리](#trie-attention-mask와-분기-격리)
   - [위치 ID와 가비지 컬렉션](#위치-id와-가비지-컬렉션)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [메모리 절감](#메모리-절감)
   - [생성 속도](#생성-속도)
   - [출력 품질](#출력-품질)
   - [빔 폭에 따른 확장성](#빔-폭에-따른-확장성)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"Efficient Beam Search for Large Language Models Using Trie-Based Decoding"은 LLM 빔 서치에서 분기마다 KV 캐시를 중복 보관하는 비효율을 trie 구조로 제거하는 디코딩 기법을 제안한다.
저자는 Brian J Chan, Jui-Hung Cheng, Mao Xun Huang, Chao-Ting Chen, Hen-Hsen Huang(국립정치대학교, 대만 / Academia Sinica)이며 2025년 2월 arXiv에 공개되었다.

논문 abstract의 핵심은 다음과 같다.
trie 기반 디코딩은 공유 prefix를 가진 빔들을 단일 KV 캐시로 통합해 추론 속도를 유지하면서 메모리를 크게 절약한다.
같은 빔 폭에서 표준 batch 빔 서치와 거의 동일한 출력을 내며, 메모리 제약 환경과 대규모 모델 배포에 특히 유용하다.

## 배경과 선행 연구

### 배치 빔 서치의 메모리 병목

저자들은 Llama 3 8B float16 모델이 모델 파라미터에 약 15.7GB GPU 메모리를 쓰고 8k 토큰 시퀀스 처리에 추가 2.5GB를 요구한다는 사례로 메모리 병목을 정량화한다.
배치 기반 빔 서치는 분기마다 독립 KV 캐시를 할당하는데, 디코딩이 진행되면서 대부분의 분기는 동일한 prefix를 공유한다.
이 prefix 토큰의 KV 캐시가 분기 수만큼 중복 저장된다.

### Related Work 정리

| 카테고리 | 주요 흐름 | 본 연구와의 차이 |
|----------|-----------|------------------|
| Beam search 효율화 | length normalization, diverse beam search | 출력 품질 개선 — 메모리 동일 |
| KV 캐시 압축 | quantization, paged attention | 캐시 자체 압축 — 본 연구는 중복 제거 |
| Speculative decoding | 작은 draft 모델 가속 | 정확도 trade-off — 본 연구는 동등 출력 |
| Tree decoding | speculative tree, lookahead | 짧은 draft tree — 본 연구는 전체 빔 trie |

본 연구는 추가 학습이나 정확도 손실 없이 빔 서치의 메모리 효율을 근본적으로 개선한다는 점에서 다른 흐름과 구별된다.

## 방법론

### Trie 기반 디코딩 핵심 아이디어

Algorithm 2는 trie 기반 빔 서치의 절차를 정의한다.
프롬프트로 trie를 초기화하고 직렬화해 모델 입력으로 만든다.
각 디코딩 step에서 b개의 best token을 trie에 확장하고 attention mask를 갱신한다.
일정 간격으로 가비지 컬렉션을 수행해 제거된 분기를 정리한다.

논문의 Figure 1 예시에서 표준 배치 빔 서치는 21토큰을 저장해야 하지만 trie 디코딩은 12토큰만 필요로 한다.

### Trie attention mask와 분기 격리

여러 분기를 단일 텐서로 합치면 분기 간 토큰이 서로 attention할 위험이 생긴다.
이를 막기 위해 specialized causal mask를 사용한다.
같은 분기 내 토큰끼리만 attention하고, 다른 분기 토큰에 대한 attention은 마스킹된다.
이 mask가 각 빔의 일관성을 보존하면서 단일 텐서 처리의 이점을 모두 누릴 수 있게 한다.

### 위치 ID와 가비지 컬렉션

trie 안에서 같은 분기의 인접 토큰이 다른 분기 토큰에 의해 떨어질 수 있다.
이 경우 position id를 분기 기준으로 재번호 매겨 표준 빔 서치와 동일한 contextual 관계를 유지한다.

가비지 컬렉션은 즉시 토큰 제거(GPU에서 비용이 큼)를 피하고 일정 간격으로 일괄 처리한다.
제거 표시된 분기를 식별해 KV 캐시를 재구성하며, 이 간격은 사용자가 설정 가능한 hyperparameter다.

## 실험 셋업

| 항목 | 값 |
|------|-----|
| 모델 | Phi-3.5-mini-instruct |
| 데이터셋 | CNN/DailyMail (200, 평균 입력 910), QMSUM (90, 평균 입력 4,133), HumanEval (164, 평균 입력 181) |
| 빔 폭 b | 1, 3, 9, 15 |
| 하드웨어 | 8 × Tesla V100-SXM2-32GB |
| Mem/Tok | (peak memory − 모델 정적 메모리) / (입력 + 출력 길이 합) |
| 속도 | 토큰/초 (Tok/Sec) |
| 품질 | 요약 ROUGE-L, HumanEval accuracy ratio |

## 주요 결과

### 메모리 절감

CNN/DailyMail b=15에서 trie의 Mem/Tok은 1.55로 batch 빔 서치의 19.95 대비 92% 감소했다.

| 데이터셋 | 빔 폭 | Batch Mem/Tok | Trie Mem/Tok |
|-----------|--------|------------------|------------------|
| CNN/DM | 15 | 19.95 | 1.55 (-92%) |
| QMSUM | 9 | 27.14 | 3.27 (-88%) |
| QMSUM | 15 | OOM | 1.57 GB/토큰 |

QMSUM b=15에서 batch 빔 서치는 OOM으로 실패한 반면 trie는 토큰당 1.57GB로 정상 완료했다.

### 생성 속도

| 데이터셋 | 빔 폭 | Batch Tok/Sec | Trie Tok/Sec |
|-----------|--------|------------------|------------------|
| CNN/DM | 3 | 7.50 | 8.01 |
| QMSUM | 9 | 2.37 | 5.15 |
| HumanEval | 15 | 6.13 | 6.21 |

QMSUM b=9에서 trie가 2.17배 빠른 결과가 가장 인상적이다.
긴 입력에서 메모리 압박이 줄어들면서 throughput이 동시에 개선됨을 보여준다.

### 출력 품질

| 데이터셋 | 빔 폭 | Batch ROUGE-L | Trie ROUGE-L |
|-----------|--------|------------------|------------------|
| CNN/DM | 3 | 0.2066 | 0.2065 |
| CNN/DM | 9 | 0.2070 | 0.2071 |
| CNN/DM | 15 | 0.2099 | 0.2093 |

모든 비교에서 차이는 통계적으로 유의하지 않다(p>0.05).
trie 빔 서치는 표준 빔 서치와 사실상 동등한 출력을 낸다는 의미다.

### 빔 폭에 따른 확장성

Figure 3에서 빔 폭이 늘어날수록 batch 빔 서치는 가파른 메모리 증가를 보이는 반면 trie는 완만한 증가에 그친다.
이 격차는 빔 폭이 커질수록, 컨텍스트가 길수록 더 벌어진다.

## 한계와 디스커션

저자가 본문에서 인정하는 한계와 주의사항은 다음과 같다.
첫째, attention mask와 trie 관리에 약간의 추가 구현 복잡도가 따른다.
둘째, 가비지 컬렉션 간격이 너무 짧으면 GPU 작업 부담이 늘고, 너무 길면 메모리 이득이 줄어드는 trade-off가 있다.
셋째, 실험은 Phi-3.5-mini-instruct 단일 모델로 진행되었으며 다른 아키텍처(MoE, MLA 등)에서의 검증이 필요하다.
넷째, prefix 공유가 적은 디코딩 시나리오(예: 매우 짧은 컨텍스트의 다양한 분기)에서는 이득이 줄어든다.

디스커션의 핵심은 두 가지다.
첫째, 빔 서치의 본질적 비효율(prefix 중복)을 학습 변경 없이 제거할 수 있는 일반적 기법이라는 점이다.
둘째, 긴 컨텍스트, 큰 빔 폭, 코드 생성 같은 정밀도 요구 태스크에서 가장 큰 이득이 발생해 production LLM 배포에 즉시 도입 가치가 있다.

## 결론

trie 기반 빔 서치는 prefix 공유 분기를 단일 KV 캐시로 통합해 같은 출력 품질을 유지하면서 메모리를 92%까지 절감한다.
QMSUM b=9에서 2.17배 throughput 향상, b=15에서 OOM을 피한 사례, ROUGE-L의 사실상 동일한 결과(p>0.05)는 이 기법의 실용적 가치를 입증한다.
추가 학습이나 특수 하드웨어 없이 즉시 도입 가능하고, 빔 폭이 커질수록 이득이 커지므로 코드 생성과 긴 문서 요약에서 특히 유용하다.

## Reference

- [Efficient Beam Search for Large Language Models Using Trie-Based Decoding (arXiv:2502.00085)](https://arxiv.org/abs/2502.00085/)
