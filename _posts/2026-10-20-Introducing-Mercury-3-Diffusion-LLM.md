---
layout: post
title: "Inception Mercury 3 - 초당 1,000토큰 이상을 내는 확산 기반 LLM"
author: 'Juho'
date: 2026-10-20 00:00:00 +0900
categories: [LLM]
tags: [LLM, AI, Benchmark, Agent]
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
2. [배경](#배경)
3. [핵심 내용](#핵심-내용)
   - [Mercury 3 주요 사양](#mercury-3-주요-사양)
   - [벤치마크 결과](#벤치마크-결과)
   - [세대별 품질과 규모](#세대별-품질과-규모)
   - [가격 정책](#가격-정책)
   - [파트너 활용 사례](#파트너-활용-사례)
   - [제품군 구성](#제품군-구성)
   - [제공 경로와 접근 방식](#제공-경로와-접근-방식)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Inception이 2026년 10월 19일 확산(diffusion) 기반 LLM의 신규 버전인 Mercury 3를 공개했습니다.
Inception은 Mercury 3를 지금까지 가장 뛰어난 확산 기반 LLM이라고 소개합니다.
회사는 Mercury 3가 여러 벤치마크에서 GPT-6 Luna와 Claude Haiku 5.5를 앞서면서도 4~8배 빠른 초당 1,000토큰 이상으로 동작한다고 주장합니다.
컨텍스트 길이는 100만 토큰으로 확대되었고, 가격은 입력 100만 토큰당 0.20달러, 출력 100만 토큰당 0.70달러입니다.

## 배경

일반적인 LLM은 토큰을 하나씩 순차적으로 생성하는 자기회귀(autoregressive) 방식을 사용합니다.
반면 Mercury 계열은 여러 위치의 텍스트를 병렬로 다듬어 생성하는 확산 방식을 사용합니다.
Mercury 3는 Mercury 1 대비 총 파라미터 수가 40배 이상 늘어났습니다.
회사 측 표준 벤치마크 기준으로 지능 지표는 Mercury 1(mini) 대비 3배 높아졌습니다.
Inception은 이를 근거로 자기회귀 모델의 발전을 이끈 스케일링 법칙이 확산 모델에도 그대로 적용된다고 주장합니다.

## 핵심 내용

### Mercury 3 주요 사양

| 항목 | 내용 |
|------|------|
| 품질 | GPT-6 Luna, Claude Haiku 5.5 등 속도 최적화 프런티어 모델과 동등 이상 |
| 속도 | NVIDIA GPU에서 초당 1,068토큰 |
| 상대 속도 | 비교 모델 대비 4~8배 빠름 |
| 컨텍스트 | 100만 토큰 (Mercury 2.5의 26만 토큰에서 확대) |
| 규모 | Mercury 1 대비 총 파라미터 40배 이상 |
| 지능 지표 | Mercury 1(mini) 대비 3배 |

### 벤치마크 결과

Inception은 같은 급의 속도 최적화 모델과 비교한 벤치마크 결과를 공개했습니다.
세 모델 모두 medium 설정 기준이며, 수치는 회사 측 발표 값입니다.

| 구분 | 항목 | Mercury 3 | GPT-6 Luna | Claude Haiku 5.5 |
|------|------|------|------|------|
| 속도와 비용 | 출력 속도 (tokens/s) | 1,068 | 약 130 | 약 240 |
| 속도와 비용 | 가격 ($/M, 입력/출력) | $0.10 / $0.35 | $0.10 / $0.50 | $0.10 / $0.50 |
| 능력 | GDPval (Elo) | 1,237.2 | 1,204.0 | 1,225.0 |
| 능력 | τ³-bench Banking | 30.24 | 23.00 | 13.06 |
| 능력 | MultiChallenge | 60.35 | 51.49 | 52.48 |
| 능력 | AA-Omniscience-Public (Accuracy) | 46.44 | 46.94 | 37.22 |
| 능력 | IFBench | 76.84 | 61.82 | 54.65 |
| 능력 | GPQA Diamond | 89.23 | 86.36 | 84.51 |
| 긴 컨텍스트 | AA-LCR v1.1 | 77.67 | 78.30 | 74.70 |
| 긴 컨텍스트 | MRCR v2 | 75.50 | 62.72 | 78.40 |
| 에이전틱 코딩 | SWE-Bench Atlas | 51.08 | 33.33 | 32.26 |
| 에이전틱 코딩 | DeepSWE v1 | 46.31 | 37.76 | 51.62 |
| 에이전틱 검색 | BrowseComp+ | 55.18 | 50.50 | 45.78 |
| 에이전틱 검색 | DeepSearchQA | 71.33 | 60.00 | 65.22 |

Mercury 3는 14개 항목 중 GDPval, τ³-bench Banking, MultiChallenge, IFBench, GPQA Diamond, SWE-Bench Atlas, BrowseComp+, DeepSearchQA 등 다수에서 가장 높은 점수를 기록했습니다.
특히 SWE-Bench Atlas에서는 51.08로 GPT-6 Luna(33.33), Claude Haiku 5.5(32.26)와 큰 차이를 보였습니다.
반면 AA-Omniscience-Public과 AA-LCR v1.1은 GPT-6 Luna가, MRCR v2와 DeepSWE v1은 Claude Haiku 5.5가 더 높았습니다.
출력 속도는 GPT-6 Luna 대비 약 8배, Claude Haiku 5.5 대비 약 4배로, 원문의 "4~8배 빠름"은 이 두 모델과의 비교입니다.
표의 가격은 출시 프로모션(50% 할인)이 적용된 Mercury 3 가격입니다.

### 세대별 품질과 규모

원문은 Mercury 1(mini)부터 Mercury 3까지 5개 세대의 모델 품질과 총 파라미터 규모를 비교한 차트도 공개했습니다.
모델 품질은 GDPval(Elo), SciCode, τ³-Banking, AA-Omniscience-Public, IFBench, SO Quality, AA-LCR v1.1, GPQA Diamond, Terminal-Bench v2.1, BrowseComp+, DeepSearchQA의 평균입니다.

| 세대 | 출시 | 모델 품질 (차트 기준 근사치) | 속도 비교 |
|------|------|------|------|
| M1 (mini) | 2025년 2월 | 약 16 | Gemini 2.0 Flash Lite 대비 약 5배 |
| M1 (small) | 2025년 2월 | 약 18 | Gemini 2.0 Flash Lite 대비 약 4배 |
| M2 | 2026년 2월 | 약 32 | Claude Haiku 4.5 대비 약 10배 |
| M2.5 | 2026년 9월 | 약 36 | GPT-5.6 Luna 대비 약 10배 |
| M3 | 2026년 10월 | 약 50 | GPT-6 Luna 대비 약 8배 |

M3는 Mercury 1(mini) 대비 총 파라미터가 40배 이상 크고, 품질 지표는 약 16에서 약 50으로 3배 가량 높아졌습니다.

### 가격 정책

| 구분 | 입력 (100만 토큰당) | 출력 (100만 토큰당) |
|------|------|------|
| 정가 | 0.20달러 | 0.70달러 |
| 출시 프로모션 (50% 할인) | 0.10달러 | 0.35달러 |

캐시된 입력 토큰은 입력 가격 대비 90% 할인이 적용됩니다.
API 플랫폼에서는 1억 토큰의 무료 티어를 제공합니다.

### 파트너 활용 사례

Inception은 실제 프로덕션 환경에서 Mercury 3를 사용한 파트너들의 결과를 함께 공개했습니다.

| 분야 | 파트너 | 보고된 결과 |
|------|------|------|
| 브라우저 에이전트 | TinyFish AI | 실시간 Mind2Web 내비게이션 과제 50개 중 50% 완료, GPT-6.1-Sol보다 높은 완료율 |
| 인터랙티브 교육 | General Learning / RevisionDojo | 힌트, 채점, 정의를 약 1초 만에 반환 |
| 실시간 검색 | Octen AI | 생성 시간 중앙값 0.9초, 전체 응답 시간 중앙값 2.2초 |
| 고객 지원 | Lorikeet | 초기 테스트에서 지연 시간과 평가 점수 모두 다른 모델보다 우수 |

TinyFish AI의 테스트에서 Mercury 3는 프런티어 모델과 동일한 토큰량과 에이전트 스텝 수를 사용했지만 토큰당 비용은 훨씬 낮았습니다.
RevisionDojo에서는 기존 모델과 같거나 더 나은 품질을 보이면서 기존 스택에 그대로 교체 투입(drop-in replacement)할 수 있었다고 합니다.
Octen AI의 Sense API 테스트에서는 검색 자체가 약 80ms에 끝나 모델 생성 단계가 병목이었습니다.
그 조건에서 Mercury 3는 테스트한 빠른 모델들 중 속도와 품질의 균형이 가장 좋았다고 보고되었습니다.
Lorikeet은 규제 산업을 위한 음성, 채팅, 이메일 컨시어지 에이전트를 만드는 회사이며, Mercury 3가 평가와 프로덕션 모두에서 더 높은 점수를 받았다고 밝혔습니다.

### 제품군 구성

| 제품 | 용도 | 특징 |
|------|------|------|
| Mercury 3 | 범용 | 고속 범용 확산 LLM |
| Mercury Voice | 음성 에이전트 | 전체 지연뿐 아니라 첫 답변 토큰까지의 시간(TTFAT) 최소화에 맞춘 튜닝 |
| Mercury Decide | 대량 분류 | JevBench, LegalBench에서 Jev보다 우수하다고 주장 |

Mercury Voice는 Audivi AI, Altur, OpenCall의 에이전트에 사용되고 있습니다.

### 제공 경로와 접근 방식

Mercury 3는 공개 당일부터 Inception API Platform, OpenRouter, Baseten, Vercel에서 사용할 수 있습니다.
OpenAI API와 호환되므로 기존 스택을 다시 작성하지 않고 연동할 수 있습니다.
엔터프라이즈 배포에서는 전용 용량, 오토스케일링, 컴플라이언스 제어, 데이터 보존 기간 설정을 제공합니다.
엔터프라이즈 평가 프로그램은 워크로드 적합성, 평가 설계, 서빙 제약 조건에서의 성능 검토를 다룹니다.
채팅 인터페이스는 [chat.inceptionlabs.ai](https://chat.inceptionlabs.ai){:target="_blank"}, API 플랫폼은 [platform.inceptionlabs.ai](https://platform.inceptionlabs.ai){:target="_blank"}에서 이용할 수 있습니다.

## 의미와 시사점

Mercury 3는 확산 기반 LLM이 파라미터 규모를 크게 늘리면서도 속도 이점을 유지할 수 있다는 점을 보여주려는 시도입니다.
100만 토큰 컨텍스트와 OpenAI API 호환성은 기존 서비스에 교체 투입하기 위한 진입 장벽을 낮춥니다.
브라우저 에이전트, 실시간 검색, 음성 상담처럼 지연 시간이 사용자 경험을 좌우하는 영역이 주요 타깃으로 제시되었습니다.
다만 공개된 성능 수치는 회사와 파트너 측 측정이므로 워크로드별 직접 검증이 필요합니다.

## 결론

Mercury 3는 초당 1,068토큰의 속도, 100만 토큰 컨텍스트, 입력 0.20달러/출력 0.70달러(100만 토큰당)의 가격을 내세운 확산 기반 LLM입니다.
Inception은 GPT-6 Luna, Claude Haiku 5.5 대비 동등 이상의 품질과 4~8배 빠른 속도를 주장하며, 확산 모델에도 스케일링 법칙이 적용된다고 강조합니다.
지연 시간에 민감한 에이전트나 실시간 서비스를 운영한다면 무료 티어 1억 토큰으로 직접 평가해 볼 만한 선택지입니다.

## Reference

- [Introducing Mercury 3 - Inception](https://www.inceptionlabs.ai/blog/introducing-mercury-3)
