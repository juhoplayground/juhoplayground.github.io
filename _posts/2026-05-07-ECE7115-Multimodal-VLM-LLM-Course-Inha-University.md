---
layout: post
title: "ECE7115, 인하대 Multimodal VLM 강의가 Stanford CS336을 따라가는 법"
author: 'Juho'
date: 2026-05-07 00:00:00 +0900
categories: [LLM]
tags: [LLM, AI, GPU]
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
   - [LLM 아키텍처 트랙](#llm-아키텍처-트랙)
   - [GPU와 시스템 트랙](#gpu와-시스템-트랙)
   - [추론, 평가, 에이전트 트랙](#추론-평가-에이전트-트랙)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

인하대학교 전기전자공학부 안남혁 교수가 2026년 봄학기 [ECE7115 Multimodal VLM (LLM)](https://gcl-inha.github.io/ece7115/){:target="_blank"} 강의 페이지를 공개했다.
강의는 Stanford CS336을 기반으로 구성되었으며, 슬라이드와 YouTube 강의 영상이 주차별로 제공된다.
Transformer 기초부터 GPU 시스템, RL 기반 사후학습, 도구 사용 에이전트까지 LLM 풀스택을 한 학기에 다루는 점이 특징이다.

## 배경

Stanford CS336은 "Language Models from Scratch"라는 슬로건으로 LLM 학계에서 표준 교과 자리를 잡아가고 있다.
한국어로 진행되는 정규 학기 강의 중 CS336 커리큘럼을 따르면서 자체 자료까지 공개하는 사례는 드물다.
ECE7115는 그 빈자리를 메우는 시도로, 영문 슬라이드를 단순히 이식하는 대신 한국어 강의 영상을 함께 제공한다.

## 핵심 내용

### LLM 아키텍처 트랙

전반부 5주는 LLM 아키텍처를 단계적으로 다룬다.
Transformer 기초에서 출발해 사전학습/사후학습/미세조정의 학습 파이프라인 전반을 정리한다.
이후 Attention 변형과 Mixture-of-Experts, Scaling Laws까지 현대 모델 설계의 핵심 의사결정을 다룬다.

| 주차 | 주제 | 핵심 내용 |
|------|------|-----------|
| 1주 | Introduction + Transformer | 과정 소개, 자원 계산, Transformer |
| 2주 | LLM 기초 | 사전학습, 사후학습, 미세조정 |
| 3주 | LLM 아키텍처 1 | 현대 모델, Attention 변형 |
| 4주 | LLM 아키텍처 2 | Mixture-of-Experts, Scaling Laws |
| 5주 | LLM 사례 연구 | 최근 모델 아키텍처 |

5주 차 사례 연구는 최근 공개된 모델 아키텍처를 골라 분석하는 형식으로 운영된다.

### GPU와 시스템 트랙

6~7주는 모델이 아니라 시스템에 초점을 둔다.
GPU의 동작 방식과 FlashAttention의 메모리 접근 패턴을 다루는 6주차는 추론 효율을 이해하기 위한 기반이 된다.
7주차는 다중 GPU와 다중 머신 학습에서 사용하는 병렬화 기법을 정리한다.

| 주차 | 주제 | 핵심 내용 |
|------|------|-----------|
| 6주 | GPU 이해 | GPU 아키텍처, FlashAttention |
| 7주 | 병렬화 | 다중 GPU/머신 학습 |

이 두 주차는 알고리즘만 아는 학생과 시스템까지 아는 학생을 가르는 분기점에 해당한다.

### 추론, 평가, 에이전트 트랙

8주차 이후는 학습이 끝난 모델을 어떻게 쓰는가에 집중한다.
강의 페이지는 8~12주를 묶어 추론, 평가, 데이터셋, RLHF, 추론 능력, 도구와 에이전트로 정리한다.
이 단계에서 RL 기반 사후학습이 본격적으로 등장하며, 도구 사용과 에이전트 패러다임까지 다루어 강의는 단순 모델 학습 강의를 넘어선다.

## 의미와 시사점

ECE7115의 가장 큰 자산은 자료 공개 정책이다.
주차별 슬라이드와 YouTube 강의 영상이 함께 제공되므로, 인하대 재학생이 아니어도 동일한 흐름으로 학습할 수 있다.
Stanford CS336을 한 번이라도 펼쳤다가 영어 강의에서 멈춰본 학습자에게는 한국어로 같은 커리큘럼을 따라갈 수 있는 길이 열린 셈이다.
강의 후반부에 RLHF와 에이전트가 포함된 점은, 2026년 시점의 LLM 강의가 단순 사전학습 모델 강의를 넘어 시스템과 활용까지 한 학기에 다루어야 한다는 현실을 반영한다.

## 결론

ECE7115는 Transformer부터 에이전트까지를 한 학기에 묶은 한국어 LLM 풀스택 강의다.
인하대학교 ECE 대학원 강의로 진행되지만, 강의 자료가 공개 페이지에 그대로 노출되어 외부 학습자도 같은 자료로 따라갈 수 있다.
"한국어로 들을 수 있는 CS336"이 필요했던 학습자라면, 이번 학기 강의 페이지가 그 출발점이 될 가능성이 높다.

## Reference

- [ECE7115 Multimodal VLM (LLM) - Inha University](https://gcl-inha.github.io/ece7115/){:target="_blank"}
