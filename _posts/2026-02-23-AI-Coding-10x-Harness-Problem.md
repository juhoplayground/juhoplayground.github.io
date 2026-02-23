---
layout: post
title: "AI 코딩 성능 10배 개선한 방법 - 모델이 아닌 편집 도구를 바꿨다"
author: 'Juho'
date: 2026-02-23 02:00:00 +0900
categories: [AI]
tags: [AI, LLM, VibeCoding]
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
2. [하네스 문제란](#하네스-문제란)
3. [Hashline 솔루션](#hashline-솔루션)
4. [주요 성과](#주요-성과)
5. [업계의 반응](#업계의-반응)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

AI 코딩 성능은 모델을 바꿔야만 향상될까?
Can Boluk는 모델은 그대로 두고 하네스(Harness)만 바꿔서 16개 LLM의 코딩 성능을 획기적으로 개선했다.
일부 모델은 성공률이 6.7%에서 68.3%까지 올랐다.

## 하네스 문제란

하네스는 모델의 출력을 실제 코드 변경으로 변환하는 인터페이스다.
아무리 모델이 정확한 답을 생성해도 이 변환 과정에서 실패하면 코드는 제대로 수정되지 않는다.
현재 주요 에이전트들이 사용하는 세 가지 방식은 각각 명확한 한계가 있다.

| 방식 | 대표 에이전트 | 한계 |
|-----|------------|------|
| Patch 형식 | OpenAI Codex | diff 구조 파싱 오류 발생 |
| String Replace | Claude, Gemini | 공백 하나로도 실패 가능 |
| 신경망 기반 | Cursor | 70B 별도 모델 학습 필요 |

## Hashline 솔루션

Can Boluk가 제안한 Hashline은 각 코드 줄에 2~3자 해시태그를 부여하는 방식이다.
모델이 코드 내용 전체를 재현할 필요 없이 짧은 태그만 참조하면 된다.

```
11:a3|function hello() {
22:f1|  return "world";
33:0e|}
```

연구자는 이를 다음과 같이 설명했다.
"모델이 내용을 재현할 필요가 없다 - 태그 인식만으로도 이해도를 증명할 수 있다."

## 주요 성과

16개 LLM 모델을 대상으로 테스트한 결과 14개 모델에서 성능이 향상됐다.

| 메트릭 | 결과 |
|-------|------|
| 테스트 대상 | 16개 LLM 모델 |
| Grok Code Fast 1 개선도 | 6.7% → 68.3% (+61.6%p) |
| 토큰 사용 절감 | 평균 20~30% |
| 성공 모델 수 | 14/16개 |

## 업계의 반응

흥미롭게도 이 연구에 대한 업계의 반응은 달갑지 않았다.
Anthropic은 오픈소스 도구 접근을 차단했고, Google은 실험자 계정을 경고 없이 비활성화했다.
두 회사 모두 경쟁사 모델 최적화에는 협조하지 않은 것으로 보인다.

## 결론

AI 코딩 도구의 성능은 모델 자체만큼이나 그것을 둘러싼 하네스의 품질에 의해 결정된다.
더 크고 비싼 모델로 교체하기 전에, 현재 사용 중인 도구의 편집 메커니즘을 먼저 점검해볼 필요가 있다.
Hashline처럼 단순한 아이디어가 수십억짜리 모델 업그레이드보다 더 큰 효과를 낼 수 있다.

## Reference

- [I Improved 15 LLMs at Coding in One Afternoon - Can Boluk](https://blog.can.ac/2026/02/12/the-harness-problem/)
- [oh-my-pi 벤치마크 코드 - GitHub](https://github.com/can1357/oh-my-pi/tree/main/packages/react-edit-benchmark)
