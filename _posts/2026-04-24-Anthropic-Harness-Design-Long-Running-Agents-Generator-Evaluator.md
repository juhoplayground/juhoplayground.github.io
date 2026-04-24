---
layout: post
title: "Anthropic의 Harness Design: 장시간 실행 에이전트를 위한 Generator-Evaluator 구조"
author: 'Juho'
date: 2026-04-24 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, LLM]
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
   - [나이브한 구현의 세 가지 한계](#나이브한-구현의-세-가지-한계)
   - [프론트엔드 디자인 실험: Generator-Evaluator 루프](#프론트엔드-디자인-실험-generator-evaluator-루프)
   - [풀스택 확장: Planner를 더한 3-Agent 구조](#풀스택-확장-planner를-더한-3-agent-구조)
   - [Opus 4.6 업그레이드 이후의 단순화](#opus-46-업그레이드-이후의-단순화)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Anthropic Labs의 Prithvi Rajasekaran이 2026년 3월 24일 공개한 글은 장시간 자율 소프트웨어 엔지니어링을 위한 "harness design" 기법을 다룬다.
단일 에이전트에 긴 작업을 통째로 맡기면 context 포화, context anxiety, 자가 평가 편향이 발생하는데, 이를 Generator-Evaluator 분리와 context reset으로 해결한 사례를 소개한다.
레트로 게임 메이커, 브라우저 DAW 같은 복잡한 프로젝트를 수 시간 단위로 자율 생성한 실험 결과와 함께, 모델 업그레이드에 따라 harness를 단순화해야 한다는 원칙을 제시한다.

## 배경

Claude 같은 모델에 "앱을 만들어줘"라는 긴 프롬프트를 한 번에 던지면 모델은 context window를 채워가며 작업하다가 중도에 일관성을 잃는다.
해결책으로 흔히 쓰이는 context compaction은 대화 이력을 요약해 이어붙이는 방식이지만, 이 글은 그것보다 더 공격적인 **context reset**(완전 초기화 + 구조화된 artifact로 상태 전달)을 주장한다.
핵심 아이디어는 모델이 할 수 없는 것을 외부 구조로 보완하되, 모델이 강해지면 그 구조를 다시 뜯어내야 한다는 것이다.

## 핵심 내용

### 나이브한 구현의 세 가지 한계

장시간 작업을 단일 에이전트에 맡겼을 때 관찰된 문제는 세 가지로 정리된다.

첫째, 일관성 손실이다.
context window가 차오르면 초반에 세운 설계 원칙을 후반에 스스로 위반하기 시작한다.

둘째, context anxiety다.
모델은 남은 context가 줄어드는 것을 인식하고, 마치 원고를 줄여 쓰듯 조기에 작업을 마무리하려는 경향을 보인다.

셋째, 자가 평가 편향이다.
생성자(generator)에게 자신의 결과물을 평가시키면 과도하게 긍정적인 평가가 나온다.
결과물이 실제로 얼마나 좋은지가 아니라 "내가 만든 것"이라는 사실이 평가를 왜곡한다.

### 프론트엔드 디자인 실험: Generator-Evaluator 루프

첫 번째 실험은 웹사이트 디자인이다.
HTML/CSS/JS를 생성하는 Generator agent와 Playwright MCP로 실제 페이지를 조작해보며 평가하는 Evaluator agent를 분리했다.

Evaluator는 네 가지 기준으로 채점한다.

| 기준 | 질문 |
|------|------|
| Design quality | 일관된 전체로 느껴지는가 |
| Originality | 템플릿 기본값이 아닌 맞춤 결정의 증거가 있는가 |
| Craft | 타이포그래피, 간격, 색상 조화의 기술적 실행 수준 |
| Functionality | 미학과 무관하게 실제로 사용 가능한가 |

5~15회 반복 사이클을 돌리면 결과물이 점진적으로 개선되다가, 특정 시점에 급격한 미적 전환이 일어난다.
흥미로운 점은 프롬프트에 "museum quality" 같은 표현을 넣었을 때 출력 스타일이 예상 이상으로 그 문구에 반응했다는 것이다.
풀 사이클은 최대 4시간까지 걸렸다.

### 풀스택 확장: Planner를 더한 3-Agent 구조

웹 앱 전체로 확장하면서 Planner를 추가했다.

Planner agent는 1~4문장짜리 요청을 상세한 제품 사양으로 확장한다.
예를 들어 16개의 기능 사양을 10개의 sprint로 분해하고, 필요하면 AI 기능 통합도 스스로 제안한다.

Generator agent는 React, Vite, FastAPI, SQLite/PostgreSQL 스택으로 sprint 단위로 구현하며 Git을 버전 관리에 사용한다.
각 sprint가 끝나면 Evaluator agent가 Playwright MCP로 실제 사용자 시나리오를 돌려 버그와 미구현을 탐지한다.

여기에는 한 가지 흥미로운 장치가 있다.
Sprint 시작 전 Generator와 Evaluator가 "완료"의 정의를 놓고 계약을 협상하고, Evaluator는 하드 임계값을 세워 하나라도 미달하면 sprint를 실패 처리한다.

첫 실험 프롬프트는 다음과 같았다.

```
Create a 2D retro game maker with features including a level editor,
sprite editor, entity behaviors, and a playable test mode.
```

Solo agent와 Full harness의 차이는 극명했다.

| 유형 | 소요 시간 | 비용 | 결과 |
|------|-----------|------|------|
| Solo agent | 20분 | $9 | UI 레이아웃 비효율, 게임 플레이 불가, entity-runtime 연결 끊김 |
| Full harness | 6시간 | $200 | UI 폴리시 양호, AI 기능 통합, 실제 플레이 가능, 물리 엔진 기본 동작 |

비용은 약 22배 늘었지만 결과물은 데모 가능한 수준과 작동 불가 수준의 차이였다.

### Opus 4.6 업그레이드 이후의 단순화

Claude Opus 4.5에서 4.6으로 올라간 뒤 팀은 harness를 오히려 줄였다.

Sprint 구조를 제거했다.
모델이 기본적으로 더 긴 작업을 native하게 지탱하므로 인위적 분할이 오히려 방해가 됐기 때문이다.
Sprint별 평가도 없애고 단일 최종 평가로 전환했다.

반면 Planner는 남겼다.
모델은 여전히 짧은 프롬프트의 scope을 충분히 넓게 해석하지 못하기 때문이다.
Evaluator는 작업이 경계선에 있을 때만 선택적으로 호출되는 구성으로 바뀌었다.

두 번째 실험은 브라우저 DAW였다.

```
Build a fully featured DAW in the browser using the Web Audio API.
```

- 소요 시간 3시간 50분, 비용 $124.70
- 첫 QA 라운드에서 "core DAW 기능 미완성" 지적
- 두 번째 라운드에서 음성 녹음, 클립 리사이즈, 효과 시각화 누락 발견
- 최종적으로 기능하는 음악 제작 환경 + 자율 에이전트 통합 완성

게임 메이커 실험보다 비용이 절반 이하로 떨어진 점이 핵심이다.
모델이 강해지면 harness의 가정(모델이 못 한다고 믿던 것들)이 만료되고, 같은 품질을 더 싸게 얻게 된다.

## 의미와 시사점

글 전체를 관통하는 네 가지 원칙이 있다.

**Task decomposition.** 복잡한 작업은 구성 가능한 청크로 쪼개고, 각 측면에 특화된 에이전트를 배정한다.

**Self-evaluation의 한계.** 생성자와 평가자를 분리하는 것은 단순한 역할 분담이 아니라 평가 신뢰도를 끌어올리는 구조적 장치다.

**Context reset이 compaction보다 강하다.** 요약해 이어붙이는 대신, 다음 에이전트에 clean slate와 구조화된 handoff를 주는 편이 context anxiety를 확실히 해결한다.

**Harness는 만료되는 가정의 집합이다.** 모든 harness 컴포넌트는 "모델이 지금은 이걸 못 한다"는 가정을 인코딩한다.
모델이 강해지면 그 가정 중 일부는 자동으로 거짓이 되므로, 새 모델이 출시될 때마다 harness를 재검토해 불필요한 계층을 제거해야 한다.

## 결론

이 글의 메시지는 간단하다.
장시간 자율 에이전트는 단일 모델만으로 만들 수 없으며, Generator-Evaluator 분리와 context reset 같은 구조가 필요하다.
동시에, 그 구조는 영구적이지 않다.
Opus 4.6 업그레이드로 sprint 구조가 사라진 것처럼, 모델이 강해질 때마다 harness를 덜어내는 용기가 필요하다.
AI 엔지니어링의 일상은 "새로운 harness를 쌓는 일"이 아니라 "어제의 harness를 오늘의 모델에 맞게 재조정하는 일"에 가까워지고 있다.

## Reference

- [Harness Design for Long-Running Application Development - Anthropic Engineering](https://www.anthropic.com/engineering/harness-design-long-running-apps)
