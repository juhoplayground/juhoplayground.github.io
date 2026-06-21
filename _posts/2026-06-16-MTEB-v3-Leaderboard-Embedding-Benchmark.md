---
layout: post
title: "MTEB v3 리더보드: 느린 데모에서 본격 임베딩 벤치마크로"
author: 'Juho'
date: 2026-06-16 00:00:00 +0900
categories: [LLM]
tags: [MTEB, Embedding, Benchmark, Evaluation]
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
2. [무엇이 달라졌나](#무엇이-달라졌나)
   - [빠른 속도](#빠른-속도)
   - [탐색과 커스터마이징](#탐색과-커스터마이징)
3. [핵심 기능](#핵심-기능)
   - [투명성과 제로샷 표시](#투명성과-제로샷-표시)
   - [상위권이 아니라 프런티어 전체 개선](#상위권이-아니라-프런티어-전체-개선)
   - [모델 비교](#모델-비교)
   - [API와 CSV 내보내기](#api와-csv-내보내기)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

MTEB(Massive Text Embedding Benchmark)는 임베딩 모델을 평가하는 대표적인 벤치마크 플랫폼이다.
HuggingFace에 공개된 이번 글은 MTEB 리더보드를 전면 재설계한 v3 버전을 소개한다.
저자들은 이번 작업을 "느린 데모에서 기능이 풍부한 리더보드로(from a slow demo to feature-rich leaderboard)" 전환한 것이라고 설명한다.

기존 리더보드는 다루는 모델과 벤치마크의 수가 늘어나면서 속도와 가동 시간 양쪽에서 신뢰하기 어려워졌다.
새 버전은 FastAPI 백엔드와 Svelte 프런트엔드를 기반으로 이러한 문제를 해결하고, 필터링과 모델 비교 같은 탐색 기능을 대폭 강화했다.
이 글은 [MTEB v3 리더보드 소개 글](https://huggingface.co/blog/Samoed/mteb-v3-leaderboard){:target="_blank"}을 기반으로 v3에서 바뀐 점을 정리한다.

## 무엇이 달라졌나

기존 리더보드의 가장 큰 문제는 성능이었다.
저자들은 모델과 벤치마크 수가 증가하면서 벤치마크 자체가 속도와 가동 시간 측면에서 불안정해졌다고 지적한다.
v3는 이 문제를 기술 스택 교체와 기능 재설계로 풀어낸다.

작업은 Solomatin Roman(Samoed), Kenneth C. Enevoldsen, Isaac Chung을 비롯한 15명 이상의 기여자가 함께 진행했다.

### 빠른 속도

새 리더보드는 이전 벤치마크보다 훨씬 빠르다.
저자들은 모바일 기기에서도 무리 없이 탐색할 수 있을 만큼 속도가 개선되었다고 밝힌다.
이 속도 향상은 FastAPI와 Svelte 기반의 새 아키텍처에서 비롯된다.

### 탐색과 커스터마이징

v3는 미리 정의된 리더보드에만 의존하지 않고, 사용자가 직접 조건을 조합해 탐색하도록 설계되었다.
필터를 통해 도메인(domain), 언어(language), 모달리티(modality), 그리고 개별 태스크 단위까지 조건을 지정할 수 있다.
또한 태스크에 마우스를 올리면 해당 태스크의 세부 정보를 바로 확인할 수 있다.

| 필터 기준 | 설명 |
|------|------|
| 도메인 | 평가 대상 도메인별로 결과를 좁혀 본다 |
| 언어 | 특정 언어 기준으로 모델 성능을 비교한다 |
| 모달리티 | 모달리티별 벤치마크를 선택한다 |
| 개별 태스크 | 미리 정의된 묶음이 아닌 단일 태스크 단위로 필터링한다 |

## 핵심 기능

v3는 단순한 순위표를 넘어, 평가의 투명성과 모델 간 비교를 돕는 기능들을 갖췄다.

### 투명성과 제로샷 표시

v3는 평가의 투명성을 높이는 데 초점을 둔다.
태스크를 직접 들여다볼 수 있게 하고, HuggingFace 데이터셋 뷰어를 통합해 평가에 쓰인 데이터를 바로 확인할 수 있다.

특히 제로샷(zero-shot) 표시는 모델이 해당 태스크의 학습 셋으로 훈련되었는지, 아니면 처음 보는 데이터인지를 구분해 보여준다.
이 정보는 학습 데이터 오염 여부를 판단하고 공정한 비교를 하는 데 중요한 단서가 된다.

### 상위권이 아니라 프런티어 전체 개선

v3는 단순히 1위 모델만 주목하게 만들지 않는다.
성능 대 런타임(performance-by-runtime) 관계를 보여주는 차트를 제공해, 속도와 정확도의 균형을 함께 평가하도록 유도한다.
또한 모델 크기 구간(size-bracket)별 순위를 제공해, 상위권뿐 아니라 프런티어 전체에 걸친 개선을 장려한다.

### 모델 비교

v3는 관심 있는 모델들을 직접 비교하는 기능을 제공한다.
비교하고 싶은 모델을 고정(pin)하면, 해당 모델들이 재정렬되어 강조 표시된다.
이를 통해 여러 벤치마크에 걸쳐 모델 간 1대1 비교를 손쉽게 수행할 수 있다.

### API와 CSV 내보내기

v3는 결과를 로컬에서 활용할 수 있는 경로도 제공한다.
점수를 CSV로 내려받거나, 제공되는 API를 통해 프로그래밍 방식으로 조회할 수 있다.
API 문서는 [MTEB 리더보드 백엔드 API](https://mteb-leaderboard-backend.hf.space/docs){:target="_blank"}에서 확인할 수 있다.

| 접근 방식 | 용도 |
|------|------|
| CSV 다운로드 | 점수를 파일로 내려받아 직접 분석 |
| API 조회 | 프로그래밍 방식으로 점수를 쿼리 |

## 의미와 시사점

MTEB v3는 새로운 점수 체계나 모델 순위를 내세우기보다, 리더보드를 다루는 방식 자체를 개선한 업데이트다.
모델 수와 벤치마크가 계속 늘어나는 상황에서, 속도와 가동 시간은 평가 도구의 신뢰성을 좌우하는 기본 요건이 되었다.

제로샷 표시와 데이터셋 뷰어 통합은 임베딩 모델 평가에서 자주 제기되는 학습 데이터 오염 문제를 사용자가 직접 점검하도록 돕는다.
성능 대 런타임 차트와 크기 구간별 순위는 무조건 최고 점수만 좇기보다, 자신의 제약 조건에 맞는 모델을 고르도록 안내한다.

## 결론

MTEB v3 리더보드는 FastAPI와 Svelte 기반의 재설계를 통해 속도와 안정성을 끌어올리고, 필터링·제로샷 표시·모델 고정 비교·API 같은 실용적 기능을 더했다.
이는 임베딩 모델을 선택하거나 평가하려는 실무자에게 더 빠르고 투명한 도구를 제공한다.

저자들은 좋은 기능 제안이나 버그 제보를 enhancement 이슈로 남겨 달라고 요청하며, 이미 피드백과 개선에 참여한 이들에게 감사를 전한다.
리더보드는 [MTEB 리더보드 스페이스](https://huggingface.co/spaces/mteb/leaderboard){:target="_blank"}에서 직접 사용해 볼 수 있다.

## Reference

- [MTEB v3 Leaderboard 소개 글](https://huggingface.co/blog/Samoed/mteb-v3-leaderboard/)
- [MTEB Leaderboard Space](https://huggingface.co/spaces/mteb/leaderboard/)
- [MTEB GitHub Repository](https://github.com/embeddings-benchmark/mteb/)
- [MTEB Leaderboard Backend API](https://mteb-leaderboard-backend.hf.space/docs/)
