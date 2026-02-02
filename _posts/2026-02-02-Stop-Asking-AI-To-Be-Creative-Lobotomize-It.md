---
layout: post
title: AI에게 창의적이라고 요청하지 마라 - 제약으로 창의성을 끌어내는 프롬프트 기법
author: 'Juho'
date: 2026-02-02 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Prompt, Productivity]
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
2. [왜 "창의적으로 답해줘"가 실패하는가](#왜-창의적으로-답해줘가-실패하는가)
3. [핵심 방법론 - 3단계 제약 기법](#핵심-방법론---3단계-제약-기법)
4. [실전 예시](#실전-예시)
5. [왜 이 방법이 작동하는가](#왜-이-방법이-작동하는가)
6. [유니버셜 템플릿](#유니버셜-템플릿)
7. [고급 기법](#고급-기법)
8. [Reference](#reference)

## 개요

"창의적으로 답해줘", "틀을 깨고 생각해줘"라는 프롬프트는 AI에서 가장 많이 사용되지만, 가장 비효과적인 지시 중 하나이다.
Humai 블로그의 Mark(Yahor Kamarou)가 3개월간 100회 이상의 프롬프트 실험을 통해 도출한 대안적 접근법을 소개한다.

핵심 아이디어는 AI에게 창의성을 요청하는 대신, 일반적인 답변에 사용되는 어휘를 금지하고 전혀 관련 없는 도메인의 비유로 답변을 강제하는 것이다.
이를 통해 고확률 토큰 대신 저확률 토큰에서 샘플링을 유도하여 진정으로 새로운 관점을 끌어낸다.

## 왜 "창의적으로 답해줘"가 실패하는가

AI 모델은 학습 데이터에서 가장 확률이 높은 토큰을 예측하는 방식으로 작동한다.
"창의적으로"라는 지시를 받으면, 모델은 학습 데이터에서 "창의적"이라고 라벨링된 콘텐츠의 통계적 평균을 출력한다.
이는 마케팅 카피, 자기계발서, 블로그 글에서 자주 등장하는 표현들의 조합에 불과하다.

결과적으로 "본질적으로 일반적이고 예측 가능한" 출력이 생성된다.
문제는 프롬프트가 아니라 고확률 토큰이 지배하는 생성 메커니즘 자체에 있다.

## 핵심 방법론 - 3단계 제약 기법

### Step 1 - 부정적 제약 (어휘 절단)

일반적인 응답에 등장하는 흔한 단어들을 금지한다.

예를 들어 "번아웃(업무 소진)"에 대한 문제를 다룬다면, 다음 용어를 금지한다.
- 우울, 피곤, 스트레스, 휴식, 균형, 치료

이 단어들은 번아웃 관련 질문에서 모델이 가장 높은 확률로 출력하는 토큰이다.
이를 금지하면 모델은 새로운 경로를 탐색할 수밖에 없다.

### Step 2 - 도메인 이동

원래 주제와 전혀 관련 없는 분야를 선택한다.

| 원래 주제 | 이동할 도메인 |
|----------|-------------|
| 심리학 문제 | 궤도 역학 |
| 인간관계 문제 | 양자 물리학 |
| 커리어 정체 | 와인 양조 |

조합이 터무니없을수록 결과가 더 강력하다.
직관적으로 연관성이 느껴지는 도메인은 기존 사고 패턴을 강화할 수 있으므로 피해야 한다.

### Step 3 - 강제 유추 출력

모델이 선택된 도메인의 용어만을 사용하여 기술적 진단과 행동 프로토콜을 생성하도록 강제한다.
직접적인 조언이나 원래 주제의 용어 사용을 금지한다.
비유(metaphor)를 벗어나는 것을 허용하지 않는다.

## 실전 예시

### 번아웃 문제를 궤도 역학 + 바이러스학으로 분석

금지 단어 : 우울, 피곤, 스트레스, 휴식, 균형, 치료

모델이 생성한 진단은 다음과 같다.
- "궤도 수정에 필요한 델타-V가 부족한 상태의 저하된 궤도를 보이고 있다"
- "중앙 노드에서 용균 주기(lytic cycle)가 감지되었다"

행동 프로토콜로는 "중력 보조 기동(gravity assist maneuver)" 수행과 72~96시간의 "바이러스 관해기(viral remission period)"가 제시되었다.

번아웃을 "에너지가 부족하다"로 설명하는 대신, "궤도 이탈"과 "바이러스 감염"의 프레임으로 재구성함으로써 완전히 새로운 해결 관점이 도출된다.

### 커리어 정체를 와인 양조로 분석

모델은 두 가지 커리어 경로를 두 개의 포도밭 부지로 묘사했다.
"스트레스가 깊이를 만든다"는 와인 양조의 원리를 커리어에 적용하여, 두 가지 옵션을 모두 시험한 후 결정하라는 조언을 제시했다.

결론은 다음과 같았다.
"올바른 부지란 없다. 포도나무가 제공하는 것을 다루는 양조가의 기술이 있을 뿐이다."

추상적인 커리어 불안을 구체적인 농업 비유로 변환함으로써 실행 가능한 통찰이 생성되었다.

## 왜 이 방법이 작동하는가

### 토큰 확률 분포 관점

제약(금지 단어)은 고확률 토큰을 차단한다.
모델은 확률 분포의 꼬리(tail) 영역에서 샘플링할 수밖에 없게 된다.
꼬리 영역의 토큰은 학습 데이터에서 덜 빈번하게 등장하므로 더 독창적인 조합이 만들어진다.

### 어텐션 메커니즘 관점

금지 조건은 트랜스포머의 어텐션 메커니즘에서 표준적인 패턴을 차단한다.
일반적인 경로가 막히면 저빈도 연결(low-frequency connections)로 주의가 재배치된다.
이는 학습 데이터에서 드물게 활성화되는 연결을 강제로 활용하게 만든다.

### 인지과학 관점

이 방법은 여러 이론적 근거를 가진다.
기능적 고착(Functional Fixedness)의 해소가 첫 번째이다.
익숙한 용어를 금지하면 문제를 새로운 프레임으로 바라보게 된다.

개념 혼합(Conceptual Blending)이 두 번째이다.
Fauconnier와 Turner의 이론에 따르면, 서로 다른 개념 공간의 요소를 혼합할 때 새로운 의미 구조가 생성된다.

낯설게 하기(Defamiliarization)가 세 번째이다.
Shklovsky의 문학 이론에서 말하는 것처럼, 익숙한 것을 낯설게 만들면 자동화된 인식에서 벗어나 대상을 다시 보게 된다.

Dr. Seuss의 "Green Eggs and Ham"은 50개의 단어만으로 쓰였다.
제약이 오히려 창의적 표현을 촉진한다는 것을 보여주는 대표적 사례이다.

## 유니버셜 템플릿

다양한 문제에 바로 적용할 수 있는 범용 프롬프트 템플릿이다.

```
[CONTEXT SETUP]
You are a specialized analytical system. Your knowledge base
is limited to ONLY the domain of [CHOSEN_DOMAIN].

You DO NOT have access to terms from [ORIGINAL_TOPIC].
You don't know these words: [LIST_OF_BANNED_WORDS].

[TASK]
Analyze this situation: "[PROBLEM_DESCRIPTION]"

Your goal:
1. Create a technical diagnosis using ONLY [CHOSEN_DOMAIN] terminology
2. Propose a solution protocol within [CHOSEN_DOMAIN] metaphors

[PROHIBITIONS]
FORBIDDEN to use: [LIST_OF_BANNED_WORDS]
FORBIDDEN to give direct advice from [ORIGINAL_TOPIC]
FORBIDDEN to exit the metaphor of [CHOSEN_DOMAIN]

[OUTPUT REQUIREMENTS]
1. Diagnosis (2-3 sentences in domain terms)
2. Action protocol (3-5 specific steps in domain metaphors)
3. Prognosis (what happens if protocol followed/ignored)

[ITERATIVE REFINEMENT]
After your initial output, perform 1-2 refinement iterations:
- Identify 3-5 most prominent words/concepts from previous output
- Add them to the banned list
- Regenerate entire output using updated prohibitions
- Label each iteration clearly
- Stop after specified iterations or if no new prominent words emerge
```

### 템플릿 사용법

1. CHOSEN_DOMAIN에 원래 주제와 무관한 도메인을 넣는다.
2. ORIGINAL_TOPIC에 실제 문제 영역을 넣는다.
3. LIST_OF_BANNED_WORDS에 해당 주제에서 가장 흔히 쓰이는 용어 5~10개를 넣는다.
4. PROBLEM_DESCRIPTION에 구체적인 상황을 기술한다.

### 반복 정제 프로세스

첫 번째 출력에서 가장 두드러진 단어 3~5개를 식별한다.
이 단어들을 기존 금지 목록에 추가한다.
업데이트된 금지 목록으로 전체 출력을 재생성한다.
각 반복을 명확하게 라벨링한다.
새로운 두드러진 단어가 없거나 수확 체감이 발생하면 중단한다.

반복할수록 모델은 더 깊은 꼬리 영역에서 토큰을 탐색하게 되어, 점점 더 독창적인 관점이 도출된다.

## 고급 기법

### 다중 도메인 분석

하나의 문제를 3~5개의 서로 다른 도메인으로 순차 분석한다.
각 도메인의 솔루션이 교차하는 지점을 찾는다.
여러 도메인에서 공통으로 나타나는 해결 방향은 해당 문제의 구조적 본질에 가까울 가능성이 높다.

### 연쇄 제약 (Cascading Constraints)

반복 단계마다 점진적으로 더 엄격한 제한을 추가한다.
첫 반복에서는 명백한 단어를 금지한다.
두 번째 반복에서는 첫 출력에서 나온 단어를 추가로 금지한다.
세 번째 반복에서는 모든 명사를 금지하는 등 극단적 제약을 적용할 수 있다.
제약이 강화될수록 모델은 더 추상적이고 구조적인 수준에서 문제를 분석하게 된다.

### 무작위 도메인 선택

직관 대신 무작위 생성기를 사용하여 도메인을 선택한다.
직접 선택하면 개인의 가정이 반영되어 기존 사고 패턴을 강화하게 된다.
무작위 선택은 이러한 편향을 제거하고 예상치 못한 연결을 만들어낸다.

## Reference
- [Stop Asking AI To Be Creative. Start Lobotomizing It Instead. - Humai Blog](https://www.humai.blog/stop-asking-ai-to-be-creative-start-lobotomizing-it-instead/)
