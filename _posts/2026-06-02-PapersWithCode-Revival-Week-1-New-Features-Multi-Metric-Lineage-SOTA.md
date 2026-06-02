---
layout: post
title: "PapersWithCode 부활 1주차 - 다중 메트릭, 논문 계보, 신규 메소드 등 6가지 신규 기능"
author: 'Juho'
date: 2026-06-02 00:00:00 +0900
categories: [Dev]
tags: [Documentation, Benchmark, Index, AI]
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
2. [신규 기능 6가지](#신규-기능-6가지)
   - [다중 메트릭 지원](#다중-메트릭-지원)
   - [외부 논문 등록](#외부-논문-등록)
   - [논문 계보](#논문-계보)
   - [신규 메소드 추가](#신규-메소드-추가)
   - [리더보드 스크린샷](#리더보드-스크린샷)
   - [평가 데이터 대량 추가](#평가-데이터-대량-추가)
3. [의미와 시사점](#의미와-시사점)
4. [결론](#결론)
5. [Reference](#reference)

## 개요

Hugging Face 오픈소스 팀의 Niels가 paperswithcode.co를 부활시킨 지 1주일이 지났다.
원래 우리가 사랑하던 paperswithcode를 다시 살리겠다는 프로젝트로, agent, computer vision, time-series forecasting 등 다양한 AI 도메인의 SOTA를 한곳에서 추적할 수 있게 한다.
첫 주에 추가된 여섯 가지 기능이 공개되었고, 반응이 좋아 향후 몇 달간 확장을 예고했다.

## 신규 기능 6가지

### 다중 메트릭 지원

같은 벤치마크에 여러 메트릭을 동시에 표시할 수 있게 되었다.
예를 들어 Open ASR Leaderboard는 음성 인식 정확도인 Word Error Rate(WER)와 추론 속도인 Inverse Real-Time Factor(RTFx)를 함께 보여준다.
Object Detection 리더보드는 COCO 데이터셋의 mAP뿐 아니라 frames-per-second(FPS)도 함께 제공한다.
정확도와 효율을 동시에 비교할 수 있어 실무용 모델 선택이 훨씬 쉬워졌다.

### 외부 논문 등록

Arxiv가 아닌 논문도 등록할 수 있게 되었다.
GitHub 저장소, 블로그 포스트, BiorXiv 등 다양한 형태를 paperswithcode.co/submit에서 받는다.
제출된 논문은 AI가 자동으로 task 태그, method 태그, GitHub 저장소, 평가 등을 enrich한다.
공개된 예시로는 Arxiv에 없는 DeepSeek-v4가 언급된다.

### 논문 계보

논문에 후속 또는 선행 작업이 있는 경우, abstract 위에 작은 배너로 표시한다.
공개된 예시는 Mamba-3, DINOv2, GLM-4.5다.
연구 흐름을 한눈에 따라가게 해주는 기능으로, 시리즈를 추적할 때 검색 비용을 크게 줄여 준다.

### 신규 메소드 추가

인기를 기반으로 새로운 메소드가 추가되었다.
공개된 항목은 Gated DeltaNet, Kimi Delta Attention, Mamba-2 등이다.
각 메소드 페이지는 해당 메소드를 인용한 모든 논문 목록을 함께 제공한다.
새로운 attention 변형이나 SSM 계열의 확산을 추적하기에 유용하다.

### 리더보드 스크린샷

리더보드 공유를 쉽게 하기 위한 "copy image" 버튼이 추가되었다.
산점도와 표 모두에 적용되어 소셜미디어에서 공유하기 좋은 형식으로 만들어 준다.
예시로 ClawEval에서 시도해 볼 수 있다.

### 평가 데이터 대량 추가

평가는 점진적으로 추가되고 있으며, Transformers 라이브러리가 지원하는 모델부터 시작했다.
현재까지 약 3,000개의 평가가 들어가 있다.
각 논문 페이지 하단에서 평가 결과를 확인할 수 있고, 예시로 Qwen 3.6이 언급된다.

## 의미와 시사점

| 항목 | 의미 |
|------|------|
| 다중 메트릭 | 정확도-효율 trade-off가 가시화 |
| 외부 논문 등록 | Arxiv 의존도 감소, 산업계 모델 포함 |
| 논문 계보 | 연구 흐름 추적 비용 감소 |
| 신규 메소드 카탈로그 | 아키텍처 변형 트렌드 모니터링 |
| 스크린샷 공유 | 결과 확산 속도 증가 |
| 평가 3,000개 | Transformers 생태계의 SOTA가 한 곳에 집결 |

또한 Niels는 더 쉬운 소통을 위해 Hugging Face Discord 서버에 채널을 만들 예정이며, GitHub thread에서도 피드백을 받는다고 밝혔다.

## 결론

기존 paperswithcode가 제공하던 가치(SOTA 추적, 코드 링크)는 그대로 유지하면서, 외부 논문 등록, 계보 표시, 다중 메트릭 등 실무자가 원하던 기능을 빠르게 추가하고 있다.
Hugging Face 산하라는 점에서 Transformers 평가 데이터와의 통합이 자연스럽다는 점도 강점이다.
1주차 만에 6가지 기능이 추가되었으니, 향후 몇 달간 발표될 업데이트도 챙겨볼 만하다.

## Reference

- 원문: Niels(Hugging Face 오픈소스 팀)의 r/MachineLearning 포스트 (사용자가 sample.txt에 제공한 텍스트)
- [paperswithcode.co](https://paperswithcode.co/)
