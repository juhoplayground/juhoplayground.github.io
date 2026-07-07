---
layout: post
title: "Anthropic이 개발자 신뢰를 잃는 방법: 과금과 벤더 락인 비판"
author: 'Juho'
date: 2026-07-07 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Management]
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
2. [지적된 문제들](#지적된-문제들)
   - [API 안정성 문제](#api-안정성-문제)
   - [Claude Code를 통한 벤더 락인](#claude-code를-통한-벤더-락인)
   - [서드파티 도구 과금](#서드파티-도구-과금)
   - [탐지와 소급 과금](#탐지와-소급-과금)
   - [소프트웨어 품질의 모순](#소프트웨어-품질의-모순)
3. [저자가 제안하는 대안](#저자가-제안하는-대안)
4. [결론](#결론)
5. [Reference](#reference)

## 개요

이 글은 Raheel Junaid가 Anthropic의 반소비자적(anti-consumer) 관행을 비판한 글을 정리한 것이다.
저자의 핵심 주장은, Anthropic이 경쟁력 있는 모델을 제공함에도 불구하고 벤더 락인, 가격 착취, 제약적 정책으로 개발자 신뢰를 스스로 훼손하고 있다는 것이다.

이 글은 특정 회사에 대한 한 개발자의 비판적 관점이며, 사용자 경험 기반의 주장이라는 점을 전제로 읽을 필요가 있다.

## 지적된 문제들

### API 안정성 문제

Claude API가 잦은 장애를 겪는다는 점을 지적한다.
저자는 구독 사용자에게 실질적인 대안이 없어, 업무를 API에 의존하는 개발자에게 의존성 리스크가 생긴다고 본다.

### Claude Code를 통한 벤더 락인

Claude 구독은 Anthropic의 독자 도구를 통해서만 사용할 수 있다.
Claude Code CLI, CoWork, Slack 연동이 그것이다.

저자는 이 구조가 사용자를 붙잡힌(captive) 상태로 만든다고 주장한다.
OpenCode 같은 더 나은 대안 인터페이스에 구독을 붙여 쓸 수 없다는 점을 문제로 든다.

### 서드파티 도구 과금

저자가 가장 강하게 비판하는 지점은 "Extra Usage" 과금이다.
서드파티 하네스(harness) 사용에 대해 API 요율로 요금을 부과하는 방식이다.

Anthropic이 Claude 구독을 퍼스트파티 접근과 서드파티 접근용 별도 풀(pool)로 분리했다는 것이 요지다.
저자는 이것이 사실상 에이전트 SDK 사용에 대해 추가 요금을 물리는 것이며, 사전 고지 없이 청구되었으므로 "전혀 extra가 아니다"라고 주장한다.

### 탐지와 소급 과금

저자는 Anthropic이 서드파티 도구 사용을 특정 파일명 확인 방식으로 탐지했다고 주장한다.
그 결과 사용자에게 소급하여(retroactively) 과금했으며, 정당한 프록시 도구까지 하룻밤 사이에 영향을 받았다고 한다.

### 소프트웨어 품질의 모순

Claude Code CLI에 수천 건의 미해결 버그가 있다는 점을 든다.
9,100개 이상의 이슈가 열려 있고, 6개월 넘게 동결된 기능도 있다는 것이다.
저자는 이것이 품질 개선을 내세우는 주장과 모순된다고 본다.

정리하면 다음과 같다.

| 지적 항목 | 저자의 주장 |
|-----------|-------------|
| API 안정성 | 잦은 장애, 대안 부재로 의존 리스크 |
| 벤더 락인 | 구독이 독자 도구에서만 동작 |
| Extra Usage 과금 | 서드파티 하네스에 API 요율 부과 |
| 탐지 방식 | 파일명 확인으로 탐지 후 소급 과금 |
| 품질 모순 | 9,100개 이상 이슈, 장기 동결 기능 존재 |

## 저자가 제안하는 대안

저자는 오픈소스 모델로의 전환을 권한다.
Qwen, GLM, Deepseek 같은 모델을 AI 게이트웨이를 통해 사용하는 방식이다.

이 방식의 장점으로는 비용 절감, 데이터 프라이버시 통제, 제약적 생태계로부터의 자유를 든다.

## 결론

이 글은 Anthropic이 기술적 우수성과 별개로, 과금 정책과 생태계 폐쇄성 때문에 개발자 커뮤니티의 선의(goodwill)를 잃고 있다는 비판을 담고 있다.
특히 사전 고지 없는 서드파티 사용 과금과 소급 청구가 신뢰 훼손의 핵심으로 지목된다.

저자의 결론은 명확하다.
제약적 생태계에 묶이는 대신, 호환 엔드포인트를 제공하는 오픈소스 모델과 AI 게이트웨이로 옮겨가라는 것이다.

## Reference

- [Anthropic's Method to Losing Goodwill in a Few Easy Steps](https://raheeljunaid.com/blog/anthropics-method-to-losing-goodwill-in-a-few-easy-steps/)
