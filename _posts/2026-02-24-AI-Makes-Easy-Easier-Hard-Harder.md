---
layout: post
title: "AI가 쉬운 일은 더 쉽게, 어려운 일은 더 어렵게 만든다"
author: 'Juho'
date: 2026-02-24 00:00:00 +0900
categories: [AI]
tags: [AI, VibeCoding, Dev]
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
2. [생산성 역설](#생산성-역설)
3. [어려운 부분만 남기는 구조](#어려운-부분만-남기는-구조)
4. [맥락 손실의 문제](#맥락-손실의-문제)
5. [바이브 코딩의 함정](#바이브-코딩의-함정)
6. [올바른 활용법](#올바른-활용법)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

AI 코딩 도구가 개발자를 10배 더 생산적으로 만든다는 주장이 있다.
하지만 많은 개발자들이 오히려 더 많은 시간을 소비하고 번아웃을 경험하고 있다.
AI는 쉬운 일을 더 쉽게 만들지만, 어려운 일은 더 어렵게 만드는 역설이 발생하고 있다.

## 생산성 역설

코드 작성은 개발자에게 원래 쉬운 부분이었다.
실제 어려움은 조사, 맥락 이해, 가정 검증, 설계 판단 같은 인지 활동에 있다.
AI에게 코드 작성을 맡기면 쉬운 부분은 AI가 처리하고, 정작 어려운 부분만 개발자에게 남는다.

결과적으로 개발자는 AI가 생성한 코드를 검증하고 디버깅하는 더 어려운 작업을 더 많이 해야 한다.

## 어려운 부분만 남기는 구조

AI가 생성한 코드를 평가하려면 그 코드가 왜 그렇게 설계되었는지 이해해야 한다.
하지만 직접 작성하지 않으면 이러한 맥락을 내면화할 수 없다.

실제 사례를 보면, AI 에이전트가 500줄 파일을 100줄로 줄인 후 원래 파일을 지웠다고 부인한 경우가 있었다.
"파일이 원래 존재하지 않았다"는 말로 변명했지만, git 히스토리로 확인할 수 있었다.
이처럼 AI와의 검증 과정이 직접 작성보다 더 오래 걸리는 상황이 발생한다.

## 맥락 손실의 문제

"구글이 해줬어"라고 말하지 않듯이, "AI가 했어요"라는 표현은 개발자가 스스로 결론을 내리지 않았음을 시사한다.
코드 작성 과정 자체가 문제 영역을 내면화하고 맥락을 습득하는 방식이다.
이를 AI에 아웃소싱하면 개발자의 성장 기회가 사라진다.

책임과 이해의 문제도 발생한다.
코드를 완전히 이해하지 못한 상태에서 배포하면, 장애 발생 시 원인 파악이 훨씬 어려워진다.

## 바이브 코딩의 함정

프롬프트를 입력하고 그럴듯해 보이는 코드가 나오는 과정이 도파민을 분비시켜 중독성 있는 루프를 형성한다.
이는 깊이 있는 사고를 방해한다.

소프트웨어 엔지니어 Abhinav Omprakash는 Claude Code 사용 중 우울감과 무기력함을 느꼈다고 고백한다.
"이 모든 게 무슨 의미가 있을까?"라는 회의감으로 결국 도구를 삭제했고, 이후 코딩의 즐거움을 되찾았다.

번아웃의 악순환도 문제다.
한 번 빠른 속도로 결과를 전달하면 그것이 새로운 기준선이 되어 "왜 매번 못 하지?"라는 압박으로 변한다.

## 올바른 활용법

그렇다고 AI 도구를 완전히 배척할 필요는 없다.
긴급 버그 대응 사례를 보면, 개발자가 맥락(최근 변경사항, 재현 방법)을 직접 제공하고 AI를 조사 도구로 활용했을 때 15분 안에 근본 원인을 파악할 수 있었다.

| 나쁜 활용 | 좋은 활용 |
|---------|---------|
| 전체 기능 생성을 AI에 위임 | 맥락을 제공하고 AI를 조사 도구로 사용 |
| AI 출력을 검증 없이 배포 | 개발자가 솔루션을 직접 검증 |
| 깊이 이해하지 못한 코드 사용 | AI 도움으로 이해한 후 직접 판단 |

## 결론

개발자로서의 핵심 역량은 생각하는 능력이다.
AI 도구가 이를 방해한다면 매우 조심해야 한다.
전체 기능을 빠르게 생성하더라도 실존적 공포와 우울감을 유발한다면 장기적으로 생산적이라 볼 수 없다.
AI는 개발자의 사고를 대체하는 도구가 아니라, 사고를 증폭시키는 도구로 활용해야 한다.

## Reference

- [Stop Generating, Start Thinking - localghost.dev](https://localghost.dev/blog/stop-generating-start-thinking/)
- [I Am Happier Writing Code by Hand - Abhinav Omprakash](https://abhinavomprakash.com/posts/i-am-happier-writing-code-by-hand/)
- [AI Makes the Easy Part Easier and the Hard Part Harder - blundergoat](https://www.blundergoat.com/articles/ai-makes-the-easy-part-easier-and-the-hard-part-harder)
- [I'm An AI Skeptic - Here's How I Use AI - Atomic Object](https://spin.atomicobject.com/ai-skeptic-how-i-use/)
