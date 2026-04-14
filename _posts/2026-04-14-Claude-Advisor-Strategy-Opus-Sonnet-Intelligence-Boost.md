---
layout: post
title: "Claude Advisor 전략: Opus와 Sonnet의 지능형 협업 모델"
author: 'Juho'
date: 2026-04-14 00:00:00 +0900
categories: [AI]
tags: [Agent, AI, LLM]
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
2. [핵심 개념](#핵심-개념)
3. [성능 지표](#성능-지표)
   - [Sonnet + Opus Advisor](#sonnet--opus-advisor)
   - [Haiku + Opus Advisor](#haiku--opus-advisor)
4. [구현 방법](#구현-방법)
   - [Advisor 도구 구현](#advisor-도구-구현)
   - [비용과 제어](#비용과-제어)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Anthropic이 Claude Platform에 [Advisor 전략](https://claude.com/blog/the-advisor-strategy){:target="_blank"}을 공식 도입했다.
Opus를 고급 조언자로, Sonnet 또는 Haiku를 실행자로 활용하는 새로운 아키텍처이다.
Opus 수준에 가까운 지능을 유지하면서 Sonnet 단독 배포와 유사한 비용 효율성을 달성하는 것이 목표이다.

## 핵심 개념

Advisor 전략은 전통적인 오케스트레이션 모델을 뒤집는다.
기존 방식은 대형 모델이 작업을 분해하고 작은 모델에 위임하는 구조였다.
Advisor 전략에서는 비용 효율적인 실행자 모델(Sonnet/Haiku)이 전체 작업을 처음부터 끝까지 수행한다.
실행자가 독립적으로 합리적으로 해결할 수 없는 결정에 직면할 때만 Opus에 조언을 요청한다.

핵심 운영 흐름은 다음과 같다.
실행자가 도구 호출을 처리하고 결과를 처리하며 솔루션을 향해 반복한다.
어려운 결정에 직면하면 실행자가 Opus에 지침을 요청한다.
Advisor는 계획, 수정 또는 중지 신호를 제공하되 직접 도구를 호출하지 않는다.
실행자는 조언 입력을 받아 실행을 재개한다.

## 성능 지표

### Sonnet + Opus Advisor

| 벤치마크 | 결과 |
|----------|------|
| SWE-bench Multilingual | Sonnet 단독 대비 2.7%p 향상 |
| 작업당 비용 | Sonnet 단독 대비 11.9% 절감 |
| BrowseComp, Terminal-Bench 2.0 | 작업당 비용 절감과 함께 점수 향상 |

Sonnet에 Opus Advisor를 결합하면 성능이 향상되면서 동시에 비용도 줄어드는 결과를 보여준다.

### Haiku + Opus Advisor

| 벤치마크 | 결과 |
|----------|------|
| BrowseComp | 41.2% (Haiku 단독 19.7% 대비 2배 이상) |
| 작업당 비용 | Sonnet 단독 대비 85% 저렴 |
| 절충점 | Sonnet 단독 대비 29% 낮은 점수 |

Haiku에 Opus Advisor를 결합하면 극적인 비용 절감과 함께 성능이 대폭 향상된다.

## 구현 방법

### Advisor 도구 구현

Anthropic은 Messages API에 직접 통합되는 서버 측 도구로 advisor_20260301을 도입했다.

```python
response = client.messages.create(
    model="claude-sonnet-4-6",  # executor
    tools=[
        {
            "type": "advisor_20260301",
            "name": "advisor",
            "model": "claude-opus-4-6",
            "max_uses": 3,
        },
        # ... other tools
    ],
    messages=[...]
)
```

단일 /v1/messages 요청이 내부적으로 핸드오프를 처리한다.
추가 라운드트립이나 수동 컨텍스트 관리가 필요 없다.
실행자 모델이 Advisor를 언제 호출할지 결정한다.
컨텍스트가 Advisor로 라우팅되고, 계획이 반환되면 동일한 요청 내에서 실행이 계속된다.

### 비용과 제어

Advisor 토큰은 Opus 요금으로, 실행자 토큰은 각 모델 요금으로 청구된다.
Advisor는 일반적으로 400~700개의 텍스트 토큰으로 짧은 계획을 생성한다.
전체 비용은 Opus를 처음부터 끝까지 실행하는 것보다 크게 낮다.
max_uses 파라미터로 요청당 Advisor 호출 횟수를 제한할 수 있다.
Advisor 토큰 사용량이 별도로 보고되어 비용 추적이 투명하다.

웹 검색 도구, 코드 실행 도구와 호환되며 에이전트 루프 내에서 원활하게 통합된다.
Messages API 요청에 도구를 추가하는 것만으로 사용이 가능하다.

## 결론

Advisor 전략의 핵심 장점은 프론티어 수준의 추론을 실행자가 필요로 할 때만 적용하여 나머지 작업은 낮은 비용으로 실행되는 것이다.
기업들은 프론티어 수준 품질을 5배 낮은 비용으로 달성하고 간단한 작업에서는 오버헤드가 없다고 보고하고 있다.
현재 Claude Platform에서 베타로 제공 중이다.

## Reference

- [The Advisor Strategy — Claude Blog](https://claude.com/blog/the-advisor-strategy/)
