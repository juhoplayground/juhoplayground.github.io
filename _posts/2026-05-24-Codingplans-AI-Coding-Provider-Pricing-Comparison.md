---
layout: post
title: "codingplans.cc — 52개 AI 코딩 공급자의 요금제와 사양을 한 화면에서 비교"
author: 'Juho'
date: 2026-05-24 00:00:00 +0900
categories: [AI]
tags: [AI, Dev, Benchmark]
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
2. [무엇을 비교하는가](#무엇을-비교하는가)
3. [공급자 카테고리](#공급자-카테고리)
4. [필터와 정렬 기능](#필터와-정렬-기능)
5. [요금제 범위와 벤치마크](#요금제-범위와-벤치마크)
6. [활용 대상](#활용-대상)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

codingplans.cc는 AI 코딩 관련 공급자의 요금제 정보를 한 화면에서 비교할 수 있게 만든 비교 플랫폼이다.
사이트의 슬로건은 단순하다.
"Compare providers and see every monthly plan tier at a glance."
공급자별로 월 단위 요금제 단계를 한눈에 보고, 그 안에서 가격과 성능, 프라이버시 정책을 동시에 살피라는 의도다.

## 무엇을 비교하는가

| 항목 | 설명 |
|------|------|
| 가격 | Lite/Entry부터 Scale/Ultra까지 월간 요금제 단계 |
| 무료 플랜 | 무료 등급 제공 여부 |
| 프라이버시 옵션 | 사용자 데이터 보호 정책 |
| 데이터 학습 정책 | 사용자 데이터의 학습 사용 여부 |
| 모델 정보 | 각 공급자가 제공하는 모델 |
| 접근 방식 | API, IDE 통합, 에이전트 등 접속 경로 |
| HQ 위치 | 공급자의 본사 소재 |
| 성능 지표 | Artificial Analysis Coding Index |

데이터는 2026년 5월 기준으로 업데이트되어 있다.

## 공급자 카테고리

총 52개 공급자가 다섯 카테고리로 분류된다.

| 카테고리 | 수량 | 예시 |
|----------|------|------|
| Direct Providers | 9 | OpenAI, Google Gemini, Anthropic Claude, Kimi, Xiaomi MiMo, Z.AI GLM, MiniMax, StepFun, Mistral |
| Aggregators & Routers | 20 | LLM Gateway, Nous Portal, ZenMux, Venice, NanoGPT, OpenCode |
| IDEs & Coding Tools | 12 | Cursor, GitHub Copilot, Warp, Windsurf, Tabnine, Zed |
| Autonomous Agents | 4 | Cosine, Factory.ai, Jules, Devin |
| App Builders | 4 | Bolt.new, Lovable, Replit, v0 |

직접 공급자, 라우터, 에디터/툴, 자율 에이전트, 앱 빌더라는 다섯 축으로 시장을 분류한 점이 특징이다.

## 필터와 정렬 기능

테이블 형식의 비교 화면에서 다음 항목을 기준으로 필터링과 정렬이 가능하다.

- 카테고리별 필터
- 무료 플랜 보유 여부
- 프라이버시 옵션
- 데이터 학습 정책
- 가격 정렬
- Artificial Analysis Coding Index 점수 정렬
- 공급자 이름 정렬

각 공급자 행에는 본사 위치, 사용 가능한 모델, 접근 방식, 프라이버시 정책이 같이 따라온다.
토큰 처리 속도, 컨텍스트 윈도우 크기, 모델 기능 등도 함께 추적되는 지표다.

## 요금제 범위와 벤치마크

| 항목 | 범위 |
|------|------|
| 무료 등급 | 대부분의 직접 공급자와 툴에서 제공 |
| 예산형 구독 | 월 3달러 수준부터 |
| 엔터프라이즈/에이전트 구독 | 월 1,000달러 수준까지 |
| 사용량 기반 | 일부 공급자는 토큰 기반 종량제 모델 제공 |

벤치마크 측면에서는 Artificial Analysis Coding Index를 기준으로 점수를 비교한다.
가격뿐 아니라 토큰 throughput, 컨텍스트 길이, 모델 기능까지 묶여서 관리된다.

## 활용 대상

| 사용자 | 주된 비교 포인트 |
|--------|-------------------|
| AI 코딩 어시스턴트를 도입하려는 개발자 | 가격 대비 모델 품질 |
| 엔터프라이즈 솔루션을 검토하는 팀 | 프라이버시 정책, 데이터 학습 옵션 |
| 비용 민감 개발자 | 무료 플랜, 저가 구독 |
| 특정 프라이버시 표준을 요구하는 조직 | 데이터 학습 비활성화 가능성 |

사이트는 운영자 정보나 창업자 정보는 노출하지 않는다.

## 결론

codingplans.cc는 AI 코딩 도구 시장이 직접 공급자, 라우터, 에디터/툴, 자율 에이전트, 앱 빌더로 분화되었음을 시각화해 보여준다.
공급자 52개와 다섯 카테고리, 가격/벤치마크/프라이버시 정책을 같은 화면에서 비교한다는 점에서, 도구 선택 단계의 의사결정을 단축하는 용도로 적합하다.
무료 플랜부터 월 1,000달러 수준 에이전트 구독까지 한 테이블에 들어와 있는 만큼, 자기 워크로드에 맞는 가격 곡선이 어디에 있는지 가늠하기 쉽다.

## Reference

- [codingplans.cc](https://codingplans.cc/){:target="_blank"}
