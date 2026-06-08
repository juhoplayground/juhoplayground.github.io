---
layout: post
title: "Microsoft MAI: 언덕 오르기 기계로 만든 7개의 새 모델"
author: 'Juho'
date: 2026-06-08 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Benchmark]
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
2. [배경: 언덕 오르기 기계](#배경-언덕-오르기-기계)
3. [핵심 내용: 7개의 새 MAI 모델](#핵심-내용-7개의-새-mai-모델)
   1. [MAI-Thinking-1](#mai-thinking-1)
   2. [MAI-Code-1-Flash](#mai-code-1-flash)
   3. [MAI-Image-2.5](#mai-image-25)
   4. [MAI-Image-2.5-Flash](#mai-image-25-flash)
   5. [MAI-Transcribe-1.5](#mai-transcribe-15)
   6. [MAI-Voice-2](#mai-voice-2)
   7. [MAI-Voice-2-Flash](#mai-voice-2-flash)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Microsoft AI가 7개의 새로운 MAI 모델을 한꺼번에 공개했다.

이번 발표에는 추론, 코딩, 이미지 생성, 음성 전사, 음성 합성에 이르는 다양한 영역의 모델이 포함됐다.

함께 mai-thinking-1.pdf 기술 문서도 배포되어, 모델의 설계와 평가 방식을 공개했다.

Microsoft는 이번 모델군을 단순한 제품 출시가 아니라 자사가 구축한 개발 체계의 결과물로 설명한다.

## 배경: 언덕 오르기 기계

Microsoft는 이번 모델 개발의 핵심을 "Hill-Climbing Machine(언덕 오르기 기계)"이라는 개념으로 규정한다.

이는 "더 많은 컴퓨트, 더 나은 데이터, 더 날카로운 평가를 적용하면서 사이클마다 지속적으로 개선할 수 있는 조직"을 의미한다.

즉 한 번의 큰 도약이 아니라, 매 사이클마다 조금씩 더 높은 곳으로 올라가는 반복적 개선 과정을 강조한 것이다.

이 체계의 바탕에는 ablation 연구, 측정, 문서화를 통한 과학적 엄밀성이 자리한다.

각 구성 요소가 결과에 미치는 영향을 분리해 측정하고, 그 과정을 문서로 남기는 방식이다.

## 핵심 내용: 7개의 새 MAI 모델

### MAI-Thinking-1

중간 크기의 플래그십 추론 모델이다.

소프트웨어 엔지니어링 벤치마크에서 선도 시스템에 필적하는 성능을 보인다.

또한 고급 수학 추론 능력을 갖췄다.

### MAI-Code-1-Flash

추론 효율적인 코딩 모델이다.

활성 파라미터는 50억 규모다.

GitHub Copilot 및 VS Code에 통합되어 사용된다.

### MAI-Image-2.5

텍스트에서 이미지를 생성하고 이미지를 편집하는 모델이다.

월드클래스 수준의 성능 점수를 기록했다.

### MAI-Image-2.5-Flash

MAI-Image-2.5의 초효율 변형이다.

같은 이미지 작업을 더 효율적으로 처리하도록 설계됐다.

### MAI-Transcribe-1.5

Microsoft가 "세계 최고 전사(transcription) 모델"로 소개한 모델이다.

43개 언어에서 SOTA 정확도를 달성했다.

또한 경쟁 모델 대비 5배 빠른 속도를 보인다.

### MAI-Voice-2

15개 언어에 걸친 고품질 음성 생성 모델이다.

음성 적응 기능을 제공한다.

### MAI-Voice-2-Flash

MAI-Voice-2의 저비용 초효율 변형이다.

출시 예정 상태이며, 더 낮은 비용으로 음성 생성을 제공하는 것을 목표로 한다.

## 의미와 시사점

Microsoft는 이번 모델군과 함께 여러 정량적 결과를 제시했다.

Excel용으로 튜닝한 MAI 모델은 GPT-5.4에 필적하면서 최대 10배 효율적이라고 밝혔다.

엔터프라이즈 애플리케이션에서는 경쟁사 대비 약 10배 낮은 비용을 달성했다고 설명한다.

자체 설계한 Maia 200 실리콘을 활용해 1.4배 컴퓨팅 효율 향상을 이뤘다.

MAI-Transcribe-1.5는 경쟁 모델보다 5배 빠른 속도를 기록했다.

아래 표는 발표에서 강조된 주요 정량 지표를 정리한 것이다.

주요 정량 결과

| 항목 | 결과 |
| --- | --- |
| Excel 튜닝 MAI 모델 | GPT-5.4에 필적, 최대 10배 효율 |
| 엔터프라이즈 비용 | 경쟁사 대비 약 10배 낮음 |
| Maia 200 실리콘 | 1.4배 컴퓨팅 효율 향상 |
| MAI-Transcribe-1.5 속도 | 경쟁 모델 대비 5배 빠름 |

이러한 수치는 단일 모델의 우수성보다, 효율과 비용을 함께 끌어올리는 개발 체계의 성과를 보여준다.

## 결론

Microsoft AI는 추론, 코딩, 이미지, 전사, 음성을 아우르는 7개의 MAI 모델을 동시에 공개했다.

이 모델들은 "언덕 오르기 기계"라는 반복적 개선 체계의 산물로 제시됐다.

ablation 연구와 측정, 문서화를 통한 과학적 엄밀성, 그리고 효율과 비용 측면의 정량 결과가 이번 발표의 핵심이다.

## Reference

- [Building a hill-climbing machine: launching seven new MAI models](https://microsoft.ai/news/building-a-hillclimbing-machine-launching-seven-new-mai-models/){:target="_blank"}
