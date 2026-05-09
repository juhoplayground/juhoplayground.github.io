---
layout: post
title: "Better Harness: Evals를 학습 신호로 삼는 하네스 힐 클라이밍"
author: 'Juho'
date: 2026-05-09 00:00:00 +0900
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
2. [핵심 방법론](#핵심-방법론)
3. [6단계 힐 클라이밍 레시피](#6단계-힐-클라이밍-레시피)
4. [발견된 하네스 변경 예시](#발견된-하네스-변경-예시)
5. [과적합 방지 설계](#과적합-방지-설계)
6. [인프라와 트레이싱](#인프라와-트레이싱)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

LangChain의 Better-Harness 접근은 "evals as training data for agents"라는 관점을 취한다.
머신러닝에서 학습 데이터가 모델 개발을 안내하듯, 각 평가 케이스는 에이전트의 하네스(프롬프트, 도구, 지시사항)에 대한 반복적 개선을 안내하는 신호가 된다.
이는 모델 가중치를 학습시키는 대신, 모델 주변 시스템을 evals 신호에 맞춰 등반하듯 개선해 나간다는 의미다.

## 핵심 방법론

힐 클라이밍은 evals에서 나오는 신호를 따라 하네스를 점진적으로 개선하는 과정이다.
각 반복마다 단일한 변경을 제안하고, evals로 검증한 뒤, 회귀가 없는지 확인한다.
이는 머신러닝의 그래디언트 디센트와 유사한 직관을 가지지만 변경 단위가 프롬프트와 도구라는 점에서 다르다.

## 6단계 힐 클라이밍 레시피

| 단계 | 행동 |
|------|------|
| 1 | Evals 소스화와 태깅: 손으로 작성한 케이스, 프로덕션 트레이스 마이닝, 외부 데이터셋 결합. 각 평가에 행동 카테고리 태그 부여(tool selection, multi-step reasoning 등) |
| 2 | 카테고리별 데이터 분할: Optimization과 Holdout 세트 분리로 과적합 방지, 변경의 일반화 보장 |
| 3 | 베이스라인 실행: 수정 전 두 세트 모두에서 성능 메트릭 수립 |
| 4 | 자율 최적화: 트레이스에서 실패 진단, 타깃 하네스 변경 제안(반복당 1개), 개선 검증 |
| 5 | 변경 검증: 새 평가 통과 확인과 동시에 이전 통과 케이스의 회귀 감지. 시스템이 이득과 트레이드오프 모두 추적 |
| 6 | 인간 검토: 제안된 변경을 수동으로 검사해 과적합과 작업 특화 지시사항에 대한 토큰 낭비 방지 |

5단계의 회귀 추적이 특히 중요하다.
힐 클라이밍은 단순히 새 케이스를 통과시키는 게 아니라, 전체 분포에서의 순 개선을 측정해야 한다.

## 발견된 하네스 변경 예시

이 방법론은 Claude Sonnet 4.6과 GLM-5에서 테스트되었으며, 두 모델 모두에서 공유되는 개선이 발견되었다.

| 변경 | 컨텍스트 | 효과 |
|------|------|------|
| Use reasonable defaults when clearly implied | tool_indirect_email_report | 누락된 문구로 인한 차단 감소 |
| Do not ask for details already supplied | followup evaluations | 중복 일정 질문 제거 |
| Bound exploration before acting | 검색 체인 | 중복 검색 루프 방지 |
| Ask domain-defining before implementation questions | 새 도구 통합 | 도구 조합 신뢰성 개선 |

이 변경들은 모두 프롬프트의 한 줄 추가지만, 트레이스 분석을 통해 발견되었고 evals로 검증되었다.
인간이 직관으로 작성하기보다 데이터에서 도출된 변경이라는 점이 핵심이다.

## 과적합 방지 설계

힐 클라이밍은 보상 해킹의 위험을 내포한다.
LangChain은 네 가지 방어 장치를 둔다.

- **Holdout 세트를 일반화 프록시로**: 프로덕션 분포와 일치하는 미공개 데이터에서 평가
- **인간 검토를 두 번째 신호로**: evals가 놓칠 수 있는 보상 해킹 행동 포착
- **태그 기반 서브셋 평가**: 비용 절감을 위해 전체 스위트가 아닌 타깃 서브셋 실행
- **정기적 eval 유지보수**: 테스트 스위트 spring-clean, 포화되거나 obsolete한 케이스 제거

게시물의 표현대로 "Quality > quantity, a small set of well-tagged evals"가 수천 개의 noisy 케이스를 능가한다.

## 인프라와 트레이싱

LangSmith는 다음을 가능하게 한다.

- 프로덕션 에이전트 상호작용에서의 dense 피드백 신호
- 최적화 중의 trace-level 진단
- 프로덕션 실패를 새 평가 케이스로 마이닝
- 버전 간 나란한 비교

저자의 표현으로 "evals are the foundation that power the harness hill-climbing process"이다.
좋은 평가 인프라가 없다면 힐 클라이밍은 임의의 추측에 그친다.

## 결론

Better-Harness는 모델 가중치를 학습시키지 않고도 evals 신호를 따라 하네스를 체계적으로 개선하는 방법론이다.
6단계 레시피, holdout 분리, 인간 검토, 태그 기반 실행이 결합되어 과적합을 방지하면서 실제 성능 향상을 만든다.
이 연구 스캐폴드는 DeepAgents 저장소에 오픈소스로 공개되었으며, 동일한 하네스에 대한 모델별 프로파일을 더 넓은 평가 스위트에서 캡처하는 작업이 진행 중이다.

## Reference

- [Better Harness: A Recipe for Harness Hill-Climbing with Evals](https://www.langchain.com/blog/better-harness-a-recipe-for-harness-hill-climbing-with-evals/)
