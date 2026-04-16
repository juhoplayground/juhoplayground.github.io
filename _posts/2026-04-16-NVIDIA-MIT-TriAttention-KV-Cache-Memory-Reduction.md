---
layout: post
title: "NVIDIA·MIT TriAttention, KV 캐시 메모리를 10배 줄이다"
author: 'Juho'
date: 2026-04-16 00:00:00 +0900
categories: [LLM]
tags: [LLM, GPU, Benchmark, Caching]
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
   - [TriAttention 작동 원리](#triattention-작동-원리)
   - [벤치마크 결과](#벤치마크-결과)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

NVIDIA와 MIT 연구진이 대규모 언어 모델의 KV 캐시 메모리 사용량을 10.7배 줄이는 새로운 기법 TriAttention을 공개했다.
기존 구글 TurboQuant의 6배 압축률을 뛰어넘는 결과다.
2026년 4월 6일 발표된 이 기술은 32B 파라미터 모델을 단일 24GB 소비자급 GPU에서 돌릴 수 있게 한다.

## 배경

KV 캐시는 LLM이 이전 대화 맥락을 기억하기 위해 키와 값 텐서를 저장하는 메모리 구조다.
컨텍스트가 길어질수록 KV 캐시 크기는 지수적으로 증가해 GPU 메모리 병목을 일으킨다.
기존 압축 기법은 대부분 정확도 저하를 동반했고, 특히 RoPE(회전 위치 인코딩) 적용 이후의 텐서 분포가 뒤섞여 압축 효율이 떨어지는 문제가 있었다.

## 핵심 내용

### TriAttention 작동 원리

TriAttention은 RoPE 이후 흐트러진 데이터에서도 중요 피처를 식별하는 수학적 필터를 사용한다.
이를 통해 단순히 데이터를 양자화하는 방식이 아니라, 어텐션 계산에 기여도가 높은 성분만을 선별적으로 보존한다.
결과적으로 메모리 사용량을 크게 줄이면서도 원본 모델의 추론 품질을 유지한다.

### 벤치마크 결과

AIME25 수학 추론 벤치마크 기준, TriAttention은 메모리를 10.7배 줄이면서 전체 정확도를 그대로 유지했다.
동일한 기법을 처리 속도 관점으로 활용하면 기존 대비 2.5배 빠른 추론이 가능하다.
32,000 토큰 수준의 초장문 컨텍스트에서도 정확도가 유지됨을 확인했다.

| 지표 | 기존 방식 | TriAttention |
|------|-----------|--------------|
| KV 캐시 압축 | TurboQuant 6배 | 10.7배 |
| AIME25 정확도 | 감소 | 유지 |
| 처리 속도 | 기준 | 2.5배 |
| 32K 토큰 대응 | 제한적 | 유지 |

## 의미와 시사점

32B 파라미터 모델이 다중 GPU 없이 24GB VRAM 단일 소비자 GPU에서 구동 가능해진다.
이는 로컬 LLM 추론, 개인 개발자 환경, 엣지 디바이스 배포의 경제성을 크게 개선한다.
RoPE 이후의 어려운 분포에서도 작동한다는 점은 기존 오픈소스 모델 대부분에 적용 가능함을 의미한다.

## 결론

TriAttention은 KV 캐시 병목이라는 LLM 추론의 고질적 문제에 대한 실질적 해법을 제시했다.
정확도를 포기하지 않고 10배 이상의 메모리 압축을 달성했다는 점에서, 향후 장문 추론과 소형 하드웨어 배포 전략을 재편할 가능성이 높다.

## Reference

- [NVIDIA, 'KV 캐시' 문제 해결... 'TriAttention'으로 메모리 10배 감소 (AI타임스)](https://www.aitimes.com/news/articleView.html?idxno=208924)
