---
layout: post
title: "Google, I/O 2026에서 Gemini 3.5 정식 공개 - 에이전트 성능과 새 구독 체계"
author: 'Juho'
date: 2026-05-31 00:00:00 +0900
categories: [LLM]
tags: [LLM, AI, Subscription]
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
2. [Gemini 3.5의 핵심](#gemini-35의-핵심)
   - [성능 벤치마크](#성능-벤치마크)
   - [주요 기능과 배포](#주요-기능과-배포)
3. [새로운 구독 체계](#새로운-구독-체계)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Google은 2026년 5월 19일 Google I/O 2026에서 새 AI 모델 제품군 Gemini 3.5를 공개했습니다.
이번 발표는 지능형 에이전트 능력과 실제 작업 수행에 초점을 맞췄습니다.
Gemini 3.5 Flash는 즉시 사용 가능하며, 3.5 Pro는 다음 달 출시 예정입니다.

Gemini 3.5 Flash는 다단계 계획과 반복이 필요한 에이전트 워크플로우에 강점을 보입니다.
코드베이스 유지보수, 재무 문서 작성, 레거시 코드 현대화 같은 복잡한 장기 시야 작업을 처리합니다.
또한 향상된 멀티모달 이해를 바탕으로 인터랙티브 웹 UI와 그래픽을 생성합니다.

## Gemini 3.5의 핵심

### 성능 벤치마크

주요 벤치마크 결과는 다음과 같습니다.

| 벤치마크 | 결과 |
|----------|------|
| Terminal-Bench 2.1 | 76.2% |
| GDPval-AA | 1656 Elo |
| MCP Atlas | 83.6% |
| CharXiv Reasoning | 84.2% |

출력 토큰 생성 속도는 경쟁 프런티어 모델 대비 4배 빠릅니다.
이 모델은 Artificial Analysis Intelligence Index의 "오른쪽 상단 사분면"에 위치하여, 속도와 프런티어급 성능의 균형을 보입니다.
또한 비용은 경쟁 프런티어 모델의 절반 미만인 경우가 많습니다.

### 주요 기능과 배포

핵심 기능은 다음과 같습니다.
Antigravity 통합으로 대규모 문제 해결을 위해 협력형 서브에이전트(subagent)를 배치합니다.
멀티모달 기반으로 향상된 그래픽과 인터랙티브 UI 생성을 지원합니다.
코딩에서는 에이전트 및 코딩 벤치마크에서 Gemini 3.1 Pro를 능가합니다.

새로운 기능 "Gemini Spark"는 3.5 Flash를 사용하는 개인 AI 에이전트입니다.
지속적으로 작동하며 사용자의 디지털 생활을 탐색하고 지시에 따라 자율적으로 행동합니다.
신뢰할 수 있는 테스터에게 먼저 제공되며, 미국의 Google AI Ultra 구독자를 대상으로 베타 접근이 계획되어 있습니다.

실제 기업 배포 사례도 가치를 입증합니다.

| 기업 | 활용 사례 |
|------|-----------|
| Shopify | 병렬 서브에이전트 분석으로 가맹점 성장 예측 |
| Macquarie Bank | 문서 추론으로 고객 온보딩 가속 |
| Salesforce | Agentforce로 복잡한 기업 작업 자동화 |
| Ramp | 멀티모달 인보이스 분석으로 OCR 향상 |
| Xero | 세무 행정을 위한 자율 멀티위크 워크플로우 관리 |

배포 채널은 Gemini 앱(전 세계), Google 검색의 AI 모드, Google Antigravity 개발자 플랫폼, AI Studio와 Android Studio의 Gemini API, Gemini Enterprise Agent Platform 등입니다.
개발은 Google의 "Frontier Safety Framework"를 따랐으며, 사이버 및 CBRN 안전장치를 강화했습니다.

## 새로운 구독 체계

Google은 한국을 포함한 글로벌 시장에 새로운 AI 구독 플랜을 도입했습니다.

| 플랜 | 가격 | 특징 |
|------|------|------|
| AI Ultra (신규) | 월 100달러 | AI Pro 대비 5배 높은 사용 한도, 20TB 클라우드 저장공간 |
| AI Ultra (기존) | 월 200달러 | 기존 250달러에서 인하, AI Pro 대비 20배 높은 사용 한도 |
| AI Plus, AI Pro | 기존 | 새 기능과 모델 접근 확대 |

새로 도입된 기능들은 다음과 같습니다.
"제미나이 옴니(Gemini Omni)"는 AI Plus, Pro, Ultra에서 한국 포함 글로벌로 제공되며, 비디오·텍스트·이미지 등 여러 입력을 처리합니다.
"제미나이 3.5 프로"는 AI Pro와 Ultra에서 한국 포함 글로벌로 제공되며, Google의 가장 강력한 에이전트 및 코딩 모델로 자리매김합니다.
"프로젝트 지니(Project Genie)"는 200달러 AI Ultra 구독자에게 한국 포함 글로벌로 제공되는 실험적 프로토타입으로, 상상 속 세계를 직접 창조하고 탐험하게 합니다.
"제미나이 스파크"는 미국 시장의 100달러·200달러 AI Ultra 플랜으로 한정됩니다.

주목할 변화는 요금제 구조입니다.
하루 단위 프롬프트 한도에서 실제 소비된 연산 자원 기반의 "컴퓨트 사용량" 모델로 전환했으며, 사용량은 5시간마다 초기화됩니다.

## 의미와 시사점

이번 발표는 Google의 AI 전략이 단일 모델 성능을 넘어 에이전트 생태계와 구독 수익화로 확장되고 있음을 보여줍니다.
Gemini 3.5 Flash는 속도와 비용 효율을 앞세워 에이전트 워크플로우 시장을 겨냥하며, Antigravity와 서브에이전트로 대규모 작업을 분산 처리합니다.

구독 측면에서는 새 100달러 AI Ultra 티어를 신설하고 기존 200달러 티어 가격을 인하하여 전문가·개발자 시장을 공략합니다.
특히 프롬프트 횟수에서 연산 사용량 기반으로의 전환은 에이전트가 더 많은 연산을 소비하는 시대에 맞춘 과금 구조 변화로 읽힙니다.

## 결론

Google은 I/O 2026에서 Gemini 3.5를 정식 공개하며 에이전트 성능, 멀티모달 생성, 비용 효율을 전면에 내세웠습니다.
Terminal-Bench 2.1 76.2%, MCP Atlas 83.6% 등 벤치마크와 4배 빠른 출력 속도, 절반 미만의 비용이 핵심 강점입니다.
동시에 100달러 신규 AI Ultra 티어와 연산 사용량 기반 과금, 제미나이 옴니·프로·프로젝트 지니 등 새 구독 기능을 한국 포함 글로벌로 도입했습니다.

## Reference

- [A new era of intelligence with Gemini 3.5 (Google Blog)](https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-3-5/)
- [Google AI 구독 안내 (Google Korea Blog)](https://blog.google/intl/ko-kr/company-news/technology/google-ai-subscriptions-kr/)
