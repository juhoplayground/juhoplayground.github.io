---
layout: post
title: "Direct Corpus Interaction: 임베딩 없이 grep과 셸로 BRIGHT, BEIR 벤치마크 SOTA 달성"
author: 'Juho'
date: 2026-05-16 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, LLM, Benchmark]
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
   - [Top-k 인터페이스의 한계](#top-k-인터페이스의-한계)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [DCI 두 변형](#dci-두-변형)
   - [Runtime context management 5단계](#runtime-context-management-5단계)
   - [Coverage와 Localization 메트릭](#coverage와-localization-메트릭)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [BrowseComp-Plus](#browsecomp-plus)
   - [Multi-hop QA](#multi-hop-qa)
   - [IR Ranking](#ir-ranking)
   - [Ablation - 왜 DCI가 효과적인가](#ablation---왜-dci가-효과적인가)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"Beyond Semantic Similarity: Rethinking Retrieval for Agentic Search via Direct Corpus Interaction"은 LLM 에이전트가 임베딩 모델, 벡터 인덱스, retrieval API 없이 grep, find, bash 같은 일반 터미널 도구로 raw corpus를 직접 검색하는 새로운 retrieval 패러다임 DCI를 제안한 논문이다.
저자는 Zhuofeng Li, Haoxiang Zhang, Cong Wei, Pan Lu, Ping Nie, Yi Lu 등을 비롯한 Texas A&M, Waterloo, Stanford, Washington, UIUC, Verdent AI, Lambda 공동 연구진이다.

논문 abstract의 핵심은 다음과 같다.
Modern retrieval은 fixed top-k 인터페이스로 corpus 접근을 압축해 정확한 lexical 제약, 약한 단서 결합, local context 검사, multi-step 가설 정제가 어렵고 일찍 필터링된 증거는 회복 불가능하다.
DCI는 이 인터페이스 자체를 grep/bash 같은 high-resolution 인터페이스로 대체한다.
BrowseComp-Plus에서 Sonnet 4.6 + DCI가 69.0%→80.0%(+11%p)로 정확도를 끌어올리면서 비용은 $1,440→$1,016(−29.4%)로 절감했다.

## 배경과 선행 연구

### Top-k 인터페이스의 한계

표준 RAG는 chunk → index → top-k retrieval → reasoning 파이프라인이다.
Agentic 시대에는 multi-round 검색이 가능해졌지만 모델은 여전히 top-k slice만 보고 reasoning한다.
BrowseComp-Plus 같은 벤치마크는 intermediate entity 발견, 약한 단서 합성, 정확한 lexical 제약, plan 수정 같은 복합 행동을 요구하는데 fixed top-k 인터페이스는 이를 제약한다.
저자들은 retrieval bottleneck이 reasoning에만 있는 것이 아니라 인터페이스 자체에 있다고 진단한다.

### Related Work 정리

| 카테고리 | 주요 선행 연구 | 본 연구와의 차이 |
|----------|----------------|------------------|
| 표준 RAG | BM25, DPR, ColBERT | 고정 top-k — DCI는 raw corpus 직접 |
| Agentic search | R1-Searcher, Search-R1, ZeroSearch, ASearcher | retriever 인터페이스 의존 — DCI는 우회 |
| LLM reranker | RankR1, Rank1, ReasonRank | retrieval 후 재정렬 — DCI는 처음부터 grep |
| Coding agent | SWE-agent, Agentless, OpenHands, Aider | code 특화 — DCI는 retrieval 일반화 |
| Keyword agent | Subramanian 2025 | document QA — DCI는 broader retrieval |

DCI의 차별점은 semantic 이해를 인덱스(upward)가 아닌 LLM(downward)으로 옮긴다는 점이다.

## 방법론

### DCI 두 변형

**DCI-Agent-Lite (minimal scaffold)**
Pi 기반 lightweight terminal agent.
bash와 read tool만 노출하며 offline indexing, dense retriever, reranker 없음.
GPT-5.4 nano 사용.

**DCI-Agent-CC (stronger scaffold)**
Claude Code(Anthropic) 기반 off-the-shelf CLI agent.
강한 prompting과 robust tool orchestration 제공.
Claude Sonnet 4.6 사용.

두 변형 모두 raw corpus 직접 검색이라는 핵심 paradigm은 동일하다.

### Runtime context management 5단계

장기 horizon DCI 지원을 위한 lightweight context layer.

| Level | Truncation | Compaction | Summarization |
|--------|--------------|--------------|------------------|
| L0 | 없음 | 없음 | 없음 |
| L1 | max 50K chars | 없음 | 없음 |
| L2 | max 20K chars | 없음 | 없음 |
| L3 | max 20K chars | 있음 | 없음 |
| L4 | max 20K chars | 있음 | 있음 |

Truncation은 tool call 결과 텍스트 캡, Compaction은 zero-LLM in-memory 정리, Summarization은 model-generated summary로 history 교체다.

### Coverage와 Localization 메트릭

trajectory 평가를 위한 두 메트릭.

```
coverage_any   = 1[ |M| ≥ 1 ]
coverage_mean  = |M| / |D*|
coverage_all   = 1[ |M| = |D*| ]
```

```
localization(q, τ) = (1/|M|) * Σ_d max_{(d,σ)} ψ(ν(|σ|); ν(|d*|))
```

Coverage는 broad evidence reach를 측정하고, Localization은 한 gold 문서 안에서 얼마나 정확한 evidence span으로 좁혀가는지를 측정한다.

## 실험 셋업

| 항목 | 값 |
|------|-----|
| 모델 | DCI-Agent-Lite: GPT-5.4 nano (high reasoning) / DCI-Agent-CC: Claude Sonnet 4.6 (medium reasoning) |
| Max turn budget | 300 (양쪽) |
| 기본 context level | L3 (main result), L4 (ablation) |
| Agentic search | BrowseComp-Plus (Qwen3-Embed-8B FAISS index baseline) |
| Multi-hop QA | NQ, TriviaQA, Bamboogle, HotpotQA, 2WikiMultiHopQA, MuSiQue (Wikipedia 2018, E5 baseline index) |
| IR ranking | BRIGHT (Bio, Earth, Econ, Robotics) + BEIR (ArguAna, SciFact) |
| Retrieval baseline | R1-Searcher-7B, Search-R1-32B, ASearcher-Local-14B 등 |
| Ranking baseline | BM25, OpenAI text-emb-3-large, GTE-Qwen2-7B, RankR1, Rank1, ReasonRank-32B |

## 주요 결과

### BrowseComp-Plus

| 모델 + 방법 | 정확도 | 비용 |
|---------------|---------|--------|
| Sonnet 4.6 + Qwen3-Embed-8B retriever | 69.0% | $1,440 |
| Sonnet 4.6 + DCI-Agent-CC | 80.0% (+11%p) | $1,016 (−29.4%) |
| GPT-5 + Qwen3-Embed-8B (강한 baseline) | 71.7% | — |
| GPT-5.4 nano + DCI-Agent-Lite | 62.9% | $93 |
| o3 + Qwen3-Embed-8B | 66.0% | $740 |

DCI-Agent-CC가 동일 backbone retrieval baseline을 +11%p 능가하면서 cost는 −29.4%였다.
DCI-Agent-Lite는 $93으로 o3 retriever 조합($740)에 가까운 정확도를 보여 cost-performance trade-off에서도 강하다.

### Multi-hop QA

| 모델 | NQ | Trivia | Bam. | Hotpot | 2Wiki | MuSiQue | 평균 | Δ |
|--------|------|---------|--------|----------|--------|-----------|--------|------|
| ASearcher-Local-14B (강한 baseline) | 56 | 58 | 62 | 58 | 56 | 24 | 52.3 | – |
| DCI-Agent-Lite (GPT-5.4 nano) | 72 | 84 | 72 | 72 | 68 | 40 | 68.0 | +15.7 |
| DCI-Agent-CC (Sonnet 4.6) | 78 | 96 | 80 | 88 | 82 | 74 | 83.0 | +30.7 |

multi-hop 벤치마크에서 DCI-Agent-CC가 ASearcher-Local-14B 대비 평균 +30.7%p 향상.
HotpotQA +30, 2Wiki +26, MuSiQue +50으로 multi-hop 깊이가 깊을수록 격차가 커진다.

### IR Ranking

NDCG@10.

| 모델 | Bio. | Earth. | Econ. | Robotics | ArguAna | SciFact | 평균 | Δ |
|--------|--------|----------|---------|------------|-----------|-----------|--------|------|
| BM25 | 18.9 | 27.2 | 14.9 | 13.6 | 31.5 | 15.8 | 20.3 | −26.7 |
| OpenAI text-emb-3-large | 23.3 | 26.7 | 19.5 | 12.8 | 58.1 | 58.1 | 33.1 | −13.9 |
| ReasonRank-32B (강한 baseline) | 58.2 | 48.9 | 36.6 | 33.9 | 28.7 | 75.5 | 47.0 | – |
| DCI-Agent-Lite (GPT-5.4 nano) | 60.0 | 50.8 | 32.3 | 42.4 | 81.9 | 72.7 | 56.7 | +9.7 |
| DCI-Agent-CC (Sonnet 4.6) | 77.1 | 69.0 | 46.8 | 56.8 | 85.3 | 75.7 | 68.5 | +21.5 |

DCI-Agent-CC가 6개 데이터셋 모두에서 best NDCG@10이며 ReasonRank-32B 대비 +21.5%p.

### Ablation - 왜 DCI가 효과적인가

trajectory 분석(BrowseComp-Plus 100문제 subset).

| 메트릭 | Qwen3-Embed-8B | DCI-Agent-Lite (L4) |
|----------|------------------|------------------------|
| coverage_any | 74.0 | 70.0 |
| coverage_mean (recall) | 56.7 | 28.0 |
| coverage_all | 28.0 | 1.0 |
| Avg localization | 21.7 | 48.4 |
| Accuracy | 45.0 | 73.0 (+28) |

DCI-Agent-Lite는 mean coverage(28.0 vs 56.7)는 더 낮지만 localization(48.4 vs 21.7)이 압도적으로 높고 정확도는 +28%p였다.
DCI의 강점은 더 많은 gold 문서를 발견하는 데 있는 것이 아니라 발견한 문서를 fine-grained local 검사·검증 단계로 변환하는 것이다.

DCI-Agent-CC tool 분포에서 Bash 62.4%, Grep 33.0%였고 Bash 안에서는 chained search 22.3%, local context peek 18.0%, regex 17.0%, file localization 14.0% 순이었다.
full-document read는 9.1%에 그쳐 verbose read보다 fine-grained 패턴 매칭이 핵심임을 보여준다.

Tool-profile ablation.

| 방법 | Avg tools | Cost | Accuracy |
|--------|-------------|--------|------------|
| Qwen3-Embed-8B retriever | 18 | $0.0498 | 45 |
| DCI read+grep만 (L4) | 19 | $0.0355 | 61 (+16) |
| DCI open bash (L4) | 35 | $0.1021 | 73 (+12 추가) |

read+grep만으로도 retrieval baseline을 +16%p 능가하고, bash 추가로 +12%p 더 향상된다.

Corpus scaling: 100K→200K에서 tool call 38.5→86.9, 정확도 −13.6%p, 400K에서 정확도 37.5%로 더 큰 하락.
DCI의 operating envelope이 검색 깊이는 잘 확장되지만 검색 폭에는 비용이 가파르게 증가한다.

Context management ablation.

| Level | Avg tools | Latency | Cost | Retained cov | Accuracy |
|--------|-------------|-----------|--------|------------------|------------|
| L0 | 28.54 | 2226s | $0.0716 | 26.9 | 72 |
| L1 | 29.00 | 1819s | $0.0720 | 31.3 | 75 |
| L2 | 29.95 | 4412s | $0.0590 | 27.2 | 69 |
| L3 | 36.89 | 8711s | $0.1109 | 27.0 | 77 |
| L4 | 35.35 | 4531s | $0.1021 | 28.0 | 73 |

L3가 best accuracy(77)이며 retained coverage(27.0)는 L1보다 낮아도 정확도가 더 높다.
selective forgetting이 다단계 가설 수정에 유리하다는 비단조 패턴이 관찰된다.

## 한계와 디스커션

저자가 본문에서 인정한 한계와 주의사항은 다음과 같다.
첫째, corpus scale 확장 시 비용이 가파르게 증가한다(400K에서 latency·tool call 폭증).
둘째, Lite는 GPT-5.4 nano, CC는 Claude Sonnet 4.6에 의존해 결과가 backbone capability에 강하게 의존한다.
셋째, BrowseComp-Plus, multi-hop QA, IR ranking 외 도메인(시계열, 의료, multimodal)으로의 일반화는 추가 검증이 필요하다.
넷째, dense/sparse retrieval은 매우 큰 정적 corpus에 여전히 효과적이며 DCI는 corpus interface 설계 공간의 한 점이다.

디스커션의 핵심 함의는 두 가지다.
첫째, 핵심 질문은 "어떤 retriever를 쓸까"가 아니라 "어떤 인터페이스가 agent의 reasoning과 가장 잘 align되는가"라는 인터페이스 설계 문제로 재구성된다.
둘째, 진화하는 local corpus(개인 workspace, heterogeneous 파일)가 일반화될수록 offline indexing 없는 DCI가 자연스러운 paradigm이 된다.

## 결론

DCI는 grep/bash로 raw corpus를 직접 검색하는 새로운 retrieval 인터페이스를 형식화하고 BrowseComp-Plus 80%(+11%p, −29.4% cost), multi-hop QA 평균 83%(+30.7), IR ranking NDCG@10 68.5(+21.5)로 strong baseline을 모두 능가했다.
"retrieval interface resolution" 개념을 도입해 DCI의 핵심이 더 많은 gold 발견이 아니라 발견된 evidence를 high-value local 검증으로 변환하는 데 있음을 정량화했다.
agent capability가 강해질수록 retrieval 품질은 reasoning뿐 아니라 인터페이스 resolution에 의존하며, DCI는 향후 agentic search 인터페이스 설계의 broader 공간을 연 연구다.

## Reference

- [Beyond Semantic Similarity: Rethinking Retrieval for Agentic Search via Direct Corpus Interaction (arXiv:2605.05242)](https://arxiv.org/abs/2605.05242/)
- [DCI-Agent-Lite GitHub](https://github.com/DCI-Agent/DCI-Agent-Lite/)
