---
layout: post
title: Langchain PDF Chatbot 만들기 - 10 - 프롬프트 엔지니어링
author: 'Juho'
date: 2025-02-26 09:01:00 +0900
categories: [LangChain]
tags: [LangChain, PDF, Chatbot, Prompt, Python]
pin: True
toc : True
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
1. [Prompt Engineering](#prompt-engineering)
 - 1) [Role(Persona) 부여](#1-rolepersona-부여)
 - 2) [명확한 지시](#2-명확한-지시)
 - 3) [예시 제공](#3-예시-제공)
 - 4) [Chain of Thought](#4-chain-of-thought)
 - 5) [Self-Consistency](#5-self-consistency)
 - 6) [Prompt Chaining](#6-prompt-chaining)
 - 7) [Tree of Thought](#7-tree-of-thought)
 - 8) [ReAct Prompting](#8-react-prompting)
 - 9) [Response Enforcement](#9-response-enforcement)
 - 10) [Response Enforcement](#10-context-provisioning)
 - 11) [Clarification Prompting](#11-clarification-prompting)


## Prompt Engineering
프롬프트 엔지니어링은 LLM에게 원하는 답변을 얻기 위해서 입력하는 프롬프트를 설계하고 최적화하는 방법을 의미합니다.<br/>
많은 프롬프트 엔지니어링 기법이 있는데 일반적으로 많이 사용되고 있고 제가 사용하고 있는 방법을 소개해보겠습니다.<br/>

#### 1) Role(Persona) 부여
LLM에게 특정 역할을 부여하면 보다 일관된 응답을 얻을 수 있습니다.<br/>

#### 2) 명확한 지시
모호한 질문보다는 구체적인 지시를 제공해야 원하는 답변을 얻을 수 있습니다. <br/>

#### 3) 예시 제공
예제를 제공하면 모델이 기대하는 형식을 학습할 수 있습니다.<br/>
##### a) [zero-shot](https://arxiv.org/pdf/2109.01652){:target="_blank"}
예시 없이 모델에게 질문이나 작업 지시만 던지는 방식입니다. 모델이 내부적으로 학습된 일반 지식을 활용해 답변합니다.<br/>
##### b) one-shot
예시를 1개만 제공하여 모델이 해당 예시를 기반으로 다음 답변을 유추하게 하는 방식입니다.<br/>
##### c) [few-shot](https://arxiv.org/pdf/2302.13971){:target="_blank"}
2~5개 정도의 예시를 제공해 모델이 답변 패턴을 학습할 수 있도록 하는 방식입니다. 예시를 통해 모델이 “이런 식으로 답변하라”는 패턴을 익히게 됩니다.<br/>
##### d) many-shot
few-shot보다 더 많은 예시를 제공하는 방식 <br/>

#### 4) [Chain of Thought](https://arxiv.org/abs/2201.11903){:target="_blank"}
복잡한 문제에 대해 한번에 답변을 요구하는 것이 아닌 1단계 → 2단계 → 3단계 형태로 결론을 도출하는 형식 <br/>

#### 5) Self-Consistency
Few-shot CoT를 통해 동일한 질의에 대해 여러 번 답변을 생성하거나 스스로 검토해 가장 신뢰도 높은 답을 고르도록 유도하는 기법 <br/>
“답변을 하기 전 답변에 대하여 검토를 한 다음 모순되는 부분이 있다면 교정한 뒤 최종 답변을 제공해야 합니다.”<br/>

#### 6) Prompt Chaining
여러 개의 프롬프트를 순차적으로 연결하여 복잡한 작업을 수행하는 기법으로 이전 단계의 답변을 다음 프롬프트의 입력으로 활용하여 점진적으로 문제 해결하는 방식입니다. <br/>
```
"피보나치 수열이 무엇인지 설명해주세요."
"파이썬을 사용하여 피보나치 수열을 생성하는 함수를 작성해주세요."
"이제 해당 함수를 최적화하는 방법을 설명해주세요."
```

#### 7) [Tree of Thought](https://arxiv.org/pdf/2305.10601){:target="_blank"}
모델이 여러 분기점을 생성하고 다양한 해결책을 평가하여 가지치기를 하여 최적의 답변을 생성합니다. <br/>

#### 8) [ReAct Prompting](https://arxiv.org/pdf/2210.03629){:target="_blank"}
LLM이 Act(행동)와 Reason(추론)을 번갈아가며 수행하도록 만드는 방식입니다.<br/>

#### 9) Response Enforcement
응답의 형식이나 길이를 제어하는 방식입니다.<br/>

#### 10) Context Provisioning
모델의 맥락 한계를 보완하기 위해 과더 대화, 여러 문단의 추가 자료나 관련 정보를 프롬프트 안에 포함하는 방식입니다.<br/>

#### 11) Clarification Prompting
사용자가 의도한 질문의 범위나 내용을 더욱 명확히 설정하기 위해, 모델 스스로 “추가 질문”을 제시하게 하여 정보를 확실히 한 뒤 답변하도록 유도하는 방식입니다.<br/>
“만약 질문이 불분명하다면 필요한 정보를 추가로 물어보세요”라는 지시를 추가함으로써 사용자의 질문 의도를 재정의합니다.<br/>


<br/>

--- 

더 많은 내용은 [프롬프트 엔지니어링 가이드](https://www.promptingguide.ai/kr){:target="_blank"}를 참고하는 것도 좋을 것 같습니다.<br/>