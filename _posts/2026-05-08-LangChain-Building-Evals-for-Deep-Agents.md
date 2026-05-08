---
layout: post
title: "Deep Agents의 Evals 설계: 양보다 질, 행동 기반 평가 만들기"
author: 'Juho'
date: 2026-05-08 00:00:00 +0900
categories: [LangChain]
tags: [LangChain, Agent, Evaluation]
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
2. [Eval 큐레이션 접근](#eval-큐레이션-접근)
3. [Eval 분류 체계](#eval-분류-체계)
4. [메트릭 프레임워크](#메트릭-프레임워크)
5. [Trajectory 평가](#trajectory-평가)
6. [정확성 평가 방법](#정확성-평가-방법)
7. [구현 인프라](#구현-인프라)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

LangChain은 Deep Agents의 평가에 대해 명확한 입장을 제시한다.
"More evals ≠ better agents"이며, 대신 프로덕션에서의 desired behavior를 반영하는 타깃 evals를 구축해야 한다.
이는 종합 벤치마크 점수에 의존하기보다 "여러 파일에 걸쳐 콘텐츠를 검색하거나 5개 이상의 순차 도구 호출을 구성하는" 같은 구체적인 프로덕션 행동을 카탈로그화하는 접근이다.

## Eval 큐레이션 접근

평가 스위트의 소스는 세 가지다.

1. **Dogfooding 피드백**: 일상 에이전트 사용에서 발생한 오류가 eval 기회가 됨
2. **외부 벤치마크**: Terminal Bench 2.0과 BFCL(Berkeley Function Calling Leaderboard)에서 선별한 작업
3. **커스텀 artisanal evals**: 중요하다고 판단한 행동에 대한 손으로 작성한 테스트

각 eval은 자체 문서화된 docstring을 포함해 측정 방법론을 설명한다.
또한 `tool_use` 같은 카테고리 태그가 붙어 그룹별 실험 실행이 가능하다.

## Eval 분류 체계

평가는 출처가 아니라 테스트하는 능력별로 구성된다.

| 카테고리 | 측정 대상 |
|------|------|
| file_operations | 파일 도구, 병렬 호출, 페이지네이션 |
| retrieval | 다중 파일 정보 찾기, 종합 |
| tool_use | 도구 선택, 다단계 체이닝, 상태 추적 |
| memory | 컨텍스트 회상과 영속성 |
| conversation | 명확화 처리, 다중 턴 대화 |
| summarization | 컨텍스트 오버플로, compaction 복구 |

이러한 분류는 에이전트가 잘하는 것과 못하는 것을 능력 단위로 분해하게 만든다.
출처 기반 분류는 어떤 외부 벤치마크에서 왔는지를 알려주지만, 능력 기반 분류는 어디를 개선해야 하는지를 알려준다.

## 메트릭 프레임워크

정확성 측정을 보완하는 4가지 효율성 메트릭이 있다.

| 메트릭 | 계산 방식 |
|------|------|
| Step ratio | 관찰된 단계 수 ÷ 이상적 단계 수 |
| Tool call ratio | 관찰된 호출 수 ÷ 이상적 호출 수 |
| Latency ratio | 관찰된 지연 ÷ 이상적 지연 |
| Solve rate | 예상 단계 수 ÷ 관찰된 지연 |

비율 기반 메트릭은 작업 난이도에 정규화되어 서로 다른 작업 간 비교를 가능하게 한다.

## Trajectory 평가

방법론의 핵심은 "ideal trajectory"다.
이는 불필요한 행동이 최소화된 최적 시퀀스를 의미한다.

예시: 4단계, 4번 호출, 8초의 이상적 경로 vs 6단계, 5번 호출, 14초의 비효율적이지만 정확한 경로.
이 비교에서 1.75배 latency ratio 같은 메트릭이 산출된다.

이는 단순한 성공/실패 평가를 넘어, 어떻게 도달했는가를 측정하는 접근이다.
같은 정답이라도 더 적은 단계와 호출로 도달하면 더 효율적인 에이전트로 평가된다.

## 정확성 평가 방법

세 가지 방식이 사용된다.

- **커스텀 assertion**: 내부 evals용 (에이전트가 호출을 병렬화했는가?)
- **Exact matching**: BFCL 같은 외부 벤치마크에서 ground truth와 비교
- **LLM-as-judge**: 의미적 정확성 평가 (적절한 메모리 영속성 등)

평가 대상의 성격에 따라 방법을 달리하는 것이 핵심이다.
구조화된 도구 호출은 exact matching이 가능하지만, 메모리 영속성 같은 의미적 행동은 LLM 판단이 필요하다.

## 구현 인프라

평가는 GitHub Actions CI 환경에서 pytest를 통해 실행된다.
LangSmith가 트레이스 분석을 중앙화하며, Polly나 Insights 같은 내장 에이전트가 대규모 분석을 수행한다.
태그 기반 서브셋 실행으로 비용 효율적인 타깃 실험이 가능하다.

```bash
uv run pytest tests/evals --eval-category file_operations --eval-category tool_use
```

이 명령은 file_operations와 tool_use 카테고리에 해당하는 evals만 실행한다.
전체 스위트가 아닌 관련 서브셋만 돌리는 패턴은 빠른 반복과 비용 절감 모두에 기여한다.

eval 아키텍처는 Deep Agents 저장소 안에서 오픈소스로 공개되어 있다.

## 결론

좋은 evals는 양이 아니라 질의 문제다.
프로덕션 행동을 카탈로그화하고, 능력별로 분류하고, 정확성과 효율성을 함께 측정하고, 태그 기반 서브셋 실행으로 비용을 통제한다.
이상적 trajectory와의 비교는 단순한 성공률을 넘어 에이전트의 효율성을 정량화하는 강력한 도구다.
LangSmith와 pytest 기반 인프라는 이러한 접근을 일상적 개발 흐름 안에서 작동시킨다.

## Reference

- [How We Build Evals for Deep Agents](https://www.langchain.com/blog/how-we-build-evals-for-deep-agents/)
