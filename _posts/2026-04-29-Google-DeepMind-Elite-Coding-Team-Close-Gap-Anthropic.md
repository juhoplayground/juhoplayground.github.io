---
layout: post
title: "Google DeepMind 엘리트 코딩 팀 구성: Anthropic과의 격차를 좁히는 2026년 전략"
author: 'Juho'
date: 2026-04-29 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Team, Management]
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
2. [팀 구성과 리더십](#팀-구성과-리더십)
3. [배경과 동기](#배경과-동기)
   - [내부 평가와 격차 인식](#내부-평가와-격차-인식)
   - [Jetski와 사용량 추적](#jetski와-사용량-추적)
4. [업계 맥락과 장기 비전](#업계-맥락과-장기-비전)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Google DeepMind가 Gemini 모델의 프로그래밍 능력을 강화하기 위해 Sebastian Borgeaud가 이끄는 전담팀을 구성했다.
Google 공동창립자 Sergey Brin과 DeepMind CTO Koray Kavukcuoglu가 직접 관여하는 등 경영진 수준의 우선순위가 부여되었다.
내부 평가 결과 Anthropic의 코딩 도구가 Google의 것보다 우월하다는 판단이 내려졌고, 이에 따라 새로운 소프트웨어 작성 같은 복잡한 장기 프로그래밍 작업에 집중하는 전략이 수립되었다.
The Decoder는 이 움직임을 2026년 코딩이 주요 AI 실험실 간 핵심 전장이 되었음을 보여주는 사례로 분석했다.

## 팀 구성과 리더십

이번 엘리트 코딩 팀은 Google DeepMind 내부에서 구성되었으며, 리더는 Sebastian Borgeaud다.
팀의 존재감을 뒷받침하는 것은 경영진 수준의 참여다.

| 역할 | 인물 |
|------|------|
| 팀 리더 | Sebastian Borgeaud |
| 경영진 참여자 | Sergey Brin(Google 공동창립자) |
| 기술 감독 | Koray Kavukcuoglu(DeepMind CTO) |

Sergey Brin의 직접 관여는 단순한 기술 프로젝트가 아닌 전사 우선순위임을 시사한다.
DeepMind CTO의 감독은 연구와 제품화 양측을 아우르는 리소스 배분을 의미하며, 팀의 결과물이 빠른 시간 내에 Gemini 제품군에 반영될 가능성이 높다.

## 배경과 동기

### 내부 평가와 격차 인식

이번 팀 구성의 직접적인 동기는 내부 평가 결과다.
Anthropic의 코딩 도구가 Google의 것보다 우월한 것으로 판단되었다는 내부 결론이 팀 구성의 출발점이다.
팀은 새로운 소프트웨어 작성 같은 복잡한 장기 프로그래밍 작업에 집중하고 있다.

이 영역은 단순 자동완성이나 짧은 함수 생성이 아니라 장시간 컨텍스트를 유지하며 새로운 코드베이스를 설계하는 능력이다.
Anthropic의 Claude Code가 강점을 보이는 장기 에이전트 워크플로우 영역에 Google이 정면으로 도전하는 그림이다.

### Jetski와 사용량 추적

Google은 내부 코드로 학습된 모델 활용을 강화하고 있다.
Brin은 모든 Gemini 엔지니어에게 복잡한 작업에 내부 에이전트 사용을 의무화했다.
이는 도구 개선과 실사용 데이터 수집을 동시에 달성하는 전략이다.

흥미로운 점은 "Jetski"라는 내부 코딩 도구의 사용량을 추적하고 팀을 순위 매기는 방식이 운영되고 있다는 것이다.

| 전략 요소 | 내용 |
|----------|------|
| 내부 코드 학습 | 구글 내부 코드베이스를 활용한 모델 훈련 강화 |
| 도구 사용 의무화 | Brin이 모든 Gemini 엔지니어에게 내부 에이전트 사용 지시 |
| Jetski 추적 | 내부 코딩 도구 사용량 팀별 순위 매김 |
| 집중 영역 | 새로운 소프트웨어 작성 같은 복잡한 장기 프로그래밍 |

이러한 내부 경쟁 유도 방식은 자체 도구 사용률을 높여 피드백 루프를 빠르게 돌리려는 의도로 읽힌다.
외부 벤치마크 경쟁력 확보에 앞서 내부 개발 생산성을 도구로 증명하는 구조다.

## 업계 맥락과 장기 비전

코딩은 2026년 주요 AI 실험실 간 핵심 전장이 되었다.
OpenAI는 Sora 중단 후 다른 모델 학습에 계산 자원을 집중한 것으로 알려져 있다.
영상 생성 우선순위를 조정하고 코딩 중심 워크로드에 투자를 재분배한 셈이다.

Google의 장기 비전은 단순 코딩 도구 경쟁을 넘어선다.
Brin은 강화된 코딩 능력이 자가 개선 AI로의 디딤돌이라고 평가했다.
이는 AI 연구자의 업무 자동화로 이어질 수 있음을 시사하는 발언이다.

이 관점은 코딩 능력이 AI 자체의 연구개발 속도를 가속시키는 메타적 능력이라는 산업 전반의 인식을 반영한다.
코드 작성이 가능한 AI는 자기 자신을 개선하는 도구를 만들 수 있고, 이것이 AI 연구의 병목을 해소하는 핵심 지렛대로 작용한다는 논리다.

## 결론

Google DeepMind의 이번 팀 구성은 단순한 조직 개편이 아니라 AI 경쟁의 축이 이동하고 있다는 명확한 신호다.
Anthropic이 장기 에이전트 코딩 영역에서 선점한 입지를 Google이 직접 겨냥하고 있으며, Brin과 Kavukcuoglu의 개입은 이 전환의 전사적 중요성을 보여준다.
내부 코드 학습 강화, Jetski 사용량 추적, 엔지니어 사용 의무화라는 세 축은 단기간 내에 제품 수준 결과로 이어질 수 있는 실행 전략이다.
개발자 관점에서 보면 2026년 하반기 Gemini 코딩 기능의 급격한 개선을 예상할 수 있으며, Claude Code 대 Gemini Code 경쟁 구도가 개발자 도구 시장을 재편할 가능성이 높다.

## Reference

- [Google Builds Elite Team to Close the Coding Gap with Anthropic](https://the-decoder.com/google-builds-elite-team-to-close-the-coding-gap-with-anthropic/)
