---
layout: post
title: "AI 에이전트 파일 처리 성공률 33%→95% - 파일 네이티브 접근법의 발견"
author: 'Juho'
date: 2026-02-23 00:00:00 +0900
categories: [AI]
tags: [AI, LLM]
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
   - [File-native context의 부상](#file-native-context의-부상)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [실험 개요](#실험-개요)
   - [Format과 architecture 조건](#format과-architecture-조건)
   - [복잡도 tier와 scale tier](#복잡도-tier와-scale-tier)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [R1 File Agent vs Prompt](#r1-file-agent-vs-prompt)
   - [R2 Format 효과](#r2-format-효과)
   - [R3 모델 tier가 지배적](#r3-모델-tier가-지배적)
   - [R4 10,000 테이블 navigation](#r4-10000-테이블-navigation)
   - [R5 grep tax 발견](#r5-grep-tax-발견)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"Structured Context Engineering for File-Native Agentic Systems"는 LLM 에이전트가 외부 시스템을 운영할 때 어떤 컨텍스트 구조를 줘야 하는지 SQL 생성을 proxy로 체계적으로 검증한 연구다.
저자는 Damon McMillan(HxAI Australia)이며, 9,649회의 실험을 11개 모델 × 4 format × 2 architecture × 10–10,000 테이블 schema로 진행했다.

논문 abstract의 핵심은 다음과 같다.
첫째, file-based context retrieval이 frontier 모델(Claude/GPT/Gemini)에서는 정확도를 +2.7%(p=0.029) 개선하지만, open source 모델에서는 평균 −7.7%(p<0.001)로 모델 의존적이다.
둘째, format(YAML/Markdown/JSON/TOON)은 평균 정확도에 유의한 영향을 주지 않는다(chi-squared=2.45, p=0.484).
셋째, 모델 capability이 지배적이며 frontier와 open source tier 사이 21%p 격차가 어떤 format/architecture 효과보다 크다.
넷째, file-native agent는 도메인 분할로 10,000 테이블까지 navigation 정확도를 유지한다.
다섯째, 파일 크기가 runtime 효율을 예측하지 못하며 compact 또는 novel format은 grep output density와 패턴 unfamiliarity로 인한 토큰 오버헤드("grep tax")를 유발한다.

## 배경과 선행 연구

### File-native context의 부상

LLM 에이전트에 컨텍스트를 제공하는 방식이 retrieval-augmented나 prompt에 직접 임베딩하는 방식에서 file-based semantic layer로 옮겨가고 있다.
CLAUDE.md, AGENTS.md 같은 프로젝트 컨벤션 파일, llms.txt 표준, Cursor Rules, schema YAML/JSON/Markdown 파일이 그 예다.
에이전트는 grep, read 같은 native 파일 도구로 이런 문서에 접근한다.

### Related Work 정리

| 카테고리 | 주요 흐름 | 본 연구와의 차이 |
|----------|-----------|------------------|
| Agent framework | ReAct, Toolformer, Gorilla, ToolLLM | 도구 사용 방법 — 본 연구는 컨텍스트 표현 |
| Text-to-SQL | Spider, BIRD, DIN-SQL, MAC-SQL | prompt 기법 중심 — 본 연구는 schema retrieval |
| Context engineering | survey 1400+ 논문 | 일반론 — 본 연구는 file-native 정량화 |
| Format 비교 | Sui 2024 (HTML/XML/JSON/YAML), He 2024 (40% 변동) | 테이블 — 본 연구는 schema 파일 |
| TOON | Token-Oriented Object Notation | 25% smaller — 본 연구가 정량 검증 |
| Lost in the middle | Liu 2024 | 컨텍스트 위치 문제 — file-native가 selective retrieval로 우회 |

## 방법론

### 실험 개요

| 실험 | 초점 | 평가 수 | 주요 변수 |
|--------|--------|----------|-------------|
| Core | SQL 생성 정확도 | 8,401 | Format, Model, Architecture, Tier |
| Scale | Schema navigation | 928 | Format, Scale tier (S0–S5) |
| Partition | Enterprise navigation | 320 | Partitioning (S6–S9) |

### Format과 architecture 조건

4가지 schema 표현 format을 비교한다.

| Format | 특성 |
|----------|--------|
| YAML | hierarchical, grep-friendly |
| Markdown | documentation 스타일, 자연어 |
| JSON | machine-parseable, verbose |
| TOON | Token-Oriented Object Notation, ~25% smaller |

모든 format에 동일한 navigator.md(스키마 개요와 테이블 설명)가 포함된다.
모든 format이 동일한 system prompt와 generic grep guidance를 받으며 format-specific 검색 패턴은 제공하지 않는다.

architecture 조건은 두 가지다.
File Agent는 grep과 read tool로 schema 파일에서 필요한 정보만 retrieve한다.
Prompt Baseline은 전체 schema(약 6,000 토큰 for TPC-DS)를 system prompt에 임베딩한다.

### 복잡도 tier와 scale tier

| 복잡도 | 유형 | 테이블 수 | SQL feature |
|------|------|------------|----------------|
| L1 | Direct lookup | 1 | SELECT, COUNT |
| L2 | Single join | 2 | basic JOIN, aggregation |
| L3 | Multi-hop | 3-4 | chained JOIN, GROUP BY |
| L4 | Complex aggregation | 4+ | CTE, window function |
| L5 | Multi-step reasoning | 5+ | subquery, nested logic |

Scale은 S0(10 테이블, startup MVP)에서 S5(1,000 테이블, large enterprise)까지 single-file이고, S6–S9는 도메인 분할(약 250 테이블/도메인)로 10,000 테이블까지 확장된다.

## 실험 셋업

| 항목 | 값 |
|------|-----|
| 평가 모델 11종 | Frontier: Claude Opus 4.5, GPT-5.2, Gemini 2.5 Pro / Frontier Lab: Claude Haiku 4.5, GPT-5-mini, Gemini 2.5 Flash / Open Source: DeepSeek-V3.2, kimi-k2, llama-4-maverick, llama-4-scout, qwen3-32b |
| Schema 출처 | TPC-DS query pattern 기반 100 query (20/tier) |
| 성공 기준 | Jaccard 유사도 ≥ 0.9 |
| Temperature | 0 (Claude Agent SDK / GPT-5는 예외) |
| 통계 | paired t-test, chi-square, ANOVA + Benjamini-Hochberg 보정 |
| Scale 실험 | Claude 모델만 |

## 주요 결과

### R1 File Agent vs Prompt

| Tier | File Agent | Prompt | Diff | p-value | BH 유의 |
|------|--------------|----------|--------|------------|----------|
| Frontier Lab + Frontier (excl Opus) | 79.8% | 77.1% | +2.7% | 0.029 | Yes |
| Frontier Lab | 76.7% | 74.0% | +2.8% | 0.107 | No |
| Open Source | 64.6% | 72.3% | −7.7% | <0.001 | Yes |

Open Source 격차는 Qwen3-32B(−21.9%)와 Llama-4-Maverick(−13.9%)이 주도한다.
Kimi-k2(+0.3%)와 Llama-4-Scout(+0.5%)는 사실상 차이가 없다.

| 모델 | Tier | File | Prompt | Diff | Winner |
|------|------|--------|----------|--------|----------|
| claude-opus-4.5 | Frontier | 89.0% | N/A | N/A | File only |
| gpt-5.2 | Frontier | 83.8% | 82.0% | +1.8% | File |
| gemini-2.5-pro | Frontier | 85.1% | 81.4% | +3.7% | File |
| claude-haiku-4.5 | Frontier Lab | 76.1% | 74.5% | +1.6% | File |
| gpt-5-mini | Frontier Lab | 77.1% | 69.4% | +7.7% | File |
| gemini-2.5-flash | Frontier Lab | 77.0% | 78.0% | −1.1% | Prompt |
| DeepSeek-V3.2 | OSS | 75.3% | 78.6% | −3.3% | Prompt |
| kimi-k2 | OSS | 75.0% | 74.7% | +0.3% | File |
| llama-4-maverick | OSS | 60.9% | 74.8% | −13.9% | Prompt |
| llama-4-scout | OSS | 63.1% | 62.6% | +0.5% | File |
| qwen3-32b | OSS | 48.9% | 70.9% | −21.9% | Prompt |

### R2 Format 효과

Chi-squared p>0.05로 평균 정확도 효과 없음.
YAML 75.4%, MD 74.9%, JSON 72.3%, TOON 72.3%.

| 모델 | YAML | MD | JSON | TOON | Best | Spread |
|------|--------|------|--------|--------|--------|----------|
| claude-opus-4.5 | 92 | 86 | 88 | 90 | YAML | 5.4% |
| gemini-2.5-pro | 86 | 84 | 88 | 83 | JSON | 5.1% |
| claude-haiku-4.5 | 84 | 72 | 73 | 75 | YAML | 12.2% |
| llama-4-maverick | 58 | 71 | 51 | 63 | MD | 20.1% |
| qwen3-32b | 54 | 49 | 45 | 48 | YAML | 9.8% |

Open Source 모델 spread(9.8–20.1%)가 frontier(1.6–5.4%)보다 훨씬 크다.

### R3 모델 tier가 지배적

ANOVA F(10, 8390)=30.55, p<0.001.
Frontier 86.0%, Frontier Lab 76.7%, Open Source 64.6%.
21%p 격차가 어떤 다른 효과보다 크다.

복잡도별 분해.

| 모델 | L1 | L2 | L3 | L4 | L5 | Overall |
|------|-----|-----|-----|-----|-----|----------|
| claude-opus-4.5 | 100 | 93 | 94 | 94 | 64 | 89.0 |
| gpt-5.2 | 95 | 89 | 96 | 85 | 54 | 83.8 |
| qwen3-32b | 92 | 53 | 44 | 37 | 18 | 48.9 |

L1에서 모든 tier가 94–96%로 비슷하지만 L5에서 Frontier 64% vs Open Source 27%로 크게 벌어진다.

### R4 10,000 테이블 navigation

Single-file schema는 1,000 테이블까지 거의 완벽한 navigation 정확도를 유지한다.
도메인 partitioning을 적용하면 10,000 테이블까지 정확도가 유지된다.
Partitioned 구조는 query당 컨텍스트를 schema 전체 크기와 무관하게 bounded로 유지한다.

### R5 grep tax 발견

11개 모델 평균 토큰 사용(TPC-DS 24테이블).

| Format | 평균 토큰 | YAML 대비 |
|----------|-------------|---------------|
| YAML | 12,729 | 기준 |
| JSON | 16,320 | +28% |
| TOON | 17,625 | +38% |
| MD | 20,382 | +60% |

Markdown overhead는 grep 비효율이 아닌 verbose pipe-table format(column separator, alignment dash) 때문이며 +7%(Gemini Flash)에서 +105%(Claude Opus)까지 모델별 차이가 크다.

TOON overhead는 25% 작은 파일 크기에도 38% 더 많은 토큰을 소비하는데 두 요인이 결합된 결과다.
첫째, output density — TOON의 compact 문법이 grep match당 더 많은 텍스트를 반환한다.
둘째, pattern unfamiliarity — TOON의 custom keyword(TABLE, COL, FK)가 학습 데이터에 거의 없어 일부 모델이 DDL/JSON/YAML 패턴을 cycle해본 뒤 정확한 TOON 문법에 도달한다.
S5_PK03 case에서 Haiku가 TOON에 16회 grep 시도가 필요한 반면 같은 query에 YAML은 3회였다.
Opus는 동일 noisy output에서 max 4회로 처리해 grep tax도 모델 의존적이다.

## 한계와 디스커션

저자가 명시한 한계는 다음과 같다.
첫째, 100 query bank(20/tier)는 더 큰 query bank나 다른 구성에서 다른 결과가 나올 수 있다.
둘째, Scale 실험은 Claude 모델만 사용하고 schema navigation(metadata retrieval)에 한정되며 10,000 테이블에서의 SQL 생성 정확도는 미검증이다.
셋째, 모든 실험이 TPC-DS(retail data warehouse)에 의존하며 비-SQL 태스크 직접 검증이 추가로 필요하다.
넷째, TOON은 학습 데이터에 거의 없는 novel format이라 grep tax 일부는 학습 부재 때문일 수 있다.
다섯째, Claude Agent SDK는 temperature 제어를 노출하지 않고 GPT-5는 최소 temperature 1을 강제해 모든 조건이 동일하지 않다.

디스커션의 핵심 함의는 두 가지다.
첫째, architecture 선택은 모델 capability에 맞춰야지 universal best practice를 가정해서는 안 된다.
둘째, format 선택은 정확도가 아닌 token 효율성, 유지보수성, grep-ability 같은 운영 기준으로 해야 한다.

| Tier | 권장 architecture |
|------|-----------------------|
| Frontier | File Agent |
| Frontier Lab | File Agent (검증 후) |
| Open Source | Prompt Engineering |

## 결론

9,649회 실험으로 file-native agentic 시스템에서 컨텍스트 엔지니어링의 다섯 가지 핵심 결론을 도출했다.
File Agent는 frontier 모델에서 +2.7% 정확도 개선을 주지만 open source에서는 −7.7%로 architecture 선택이 모델 의존적이다.
Format은 평균 정확도에 영향이 없지만 open source 모델은 spread가 9.8–20.1%로 훨씬 크다.
모델 capability가 21%p 격차로 지배적이며 file-native partitioning이 10,000 테이블까지 확장 가능하다.
"grep tax"라는 새로운 현상을 정의해 compact format이 토큰 효율과 다를 수 있음을 정량화했다.
실용적으로는 YAML이 토큰 효율과 grep-friendliness 양면에서 우수한 default format으로 권장된다.

## Reference

- [Structured Context Engineering for File-Native Agentic Systems (arXiv:2602.05447)](https://arxiv.org/abs/2602.05447/)
