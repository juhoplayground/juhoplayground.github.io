---
layout: post
title: "Needle In A Haystack: 장문 컨텍스트 LLM의 검색 능력을 재는 벤치마크"
author: 'Juho'
date: 2026-04-26 00:00:00 +0900
categories: [LLM]
tags: [LLM, Benchmark, Evaluation, Context]
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
2. [테스트 방법론](#테스트-방법론)
3. [지원 모델 제공자](#지원-모델-제공자)
4. [주요 기능](#주요-기능)
5. [설치와 실행](#설치와-실행)
6. [결과 해석](#결과-해석)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

Greg Kamradt가 공개한 LLMTest_NeedleInAHaystack은 장문 컨텍스트 LLM의 검색 정확도를 체계적으로 측정하는 벤치마크다.
긴 문서 안에 특정 문장을 심어두고, 모델이 그 문장을 찾아낼 수 있는지를 컨텍스트 길이와 위치별로 측정한다.
단순한 요약 능력이 아니라 "길이가 늘어날 때 정보를 얼마나 안정적으로 보존하는가"를 검증한다.

## 테스트 방법론

벤치마크는 세 단계로 구성된다.

1. **Needle 삽입**: 무작위 사실이나 문장을 긴 텍스트 패시지의 특정 위치에 배치
2. **검색 과제**: 모델에게 해당 정보를 찾아 추출하도록 지시
3. **성능 측정**: 다양한 컨텍스트 윈도우 크기와 needle 깊이에서 정확도 추적

테스트는 두 축을 체계적으로 변화시킨다.

| 축 | 의미 |
|------|------|
| Document depth | needle이 컨텍스트 내에서 등장하는 위치 (% 표현) |
| Context length | 모델이 처리해야 하는 전체 텍스트 양 |

두 축의 조합으로 만들어지는 매트릭스는 문서가 길어지고 needle이 깊이 숨겨질 때 성능이 어떻게 저하되는지를 히트맵 형태로 보여준다.

## 지원 모델 제공자

현재 다음 제공자를 지원한다.

- OpenAI (GPT-4-128K 포함)
- Anthropic (Claude 2.1 이후 버전)
- Cohere (Command-R 모델)

제공자별로 컨텍스트가 극단적으로 길어질 때 정확도 열화 패턴이 크게 달라진다는 점이 테스트의 핵심 발견이다.

## 주요 기능

멀티 needle 테스트를 지원해 여러 needle을 컨텍스트에 균등 분포시켜 난이도를 높일 수 있다.
평가는 다른 언어 모델을 심판으로 사용하거나 LangSmith 오케스트레이션 플랫폼을 활용하는 두 가지 옵션이 제공된다.
Jupyter 노트북을 통해 피벗 테이블과 히트맵으로 성능을 시각화할 수 있다.

## 설치와 실행

사전 준비 사항은 다음과 같다.

- Python 가상 환경
- 모델 제공자 API 키 및 선택적 평가자 API 키
- PyPI 패키지 매니저를 통한 설치

기본 실행 예시다.

```bash
needlehaystack.run_test --provider openai --model_name "gpt-3.5-turbo-0125" \
    --document_depth_percents "[50]" --context_lengths "[2000]"
```

`--document_depth_percents`와 `--context_lengths`에 리스트를 넘겨 여러 조합을 한 번에 실행할 수 있다.
결과는 JSON 로그로 저장되며 제공된 시각화 노트북으로 분석한다.

## 결과 해석

히트맵에서 녹색 영역은 정확한 검색, 적색은 실패를 나타낸다.
일반적으로 문서가 길어지고 needle이 중간 깊이에 있을 때 "lost in the middle" 현상이 두드러진다.
시작과 끝 부분의 needle은 상대적으로 잘 찾지만 중간에 묻힌 정보는 놓치는 경향이 모델별로 다르게 나타난다.
이러한 패턴은 RAG 설계 시 청크 순서와 재정렬 전략을 설계하는 데 직접적인 근거가 된다.

## 결론

Needle In A Haystack은 단순한 마케팅 수치인 "컨텍스트 윈도우 크기"와 실제 사용 가능한 길이 사이의 간극을 정량화한다.
모델 공급자가 주장하는 컨텍스트 한도가 실제로 유효한지 자사 워크로드에서 확인하려면 이 벤치마크를 먼저 돌려보는 것이 합리적이다.
장문 검색이 핵심인 RAG, 문서 분석, 로그 파싱 같은 시나리오에서 모델 선정의 객관적 근거로 활용할 수 있다.

## Reference

- [gkamradt/LLMTest_NeedleInAHaystack on GitHub](https://github.com/gkamradt/LLMTest_NeedleInAHaystack/)
