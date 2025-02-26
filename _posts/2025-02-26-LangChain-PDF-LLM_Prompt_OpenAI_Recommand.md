---
layout: post
title: Langchain PDF Chatbot 만들기 - 11 - 프롬프트 엔지니어링 With OpenAI
author: 'Juho'
date: 2025-02-26 09:02:00 +0900
categories: [LangChain]
tags: [LangChain, PDF, Chatbot, Prompt, OpenAI, Python]
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
1. [OpenAI가 권장하는 프롬프트 엔지니어링](#openai가-권장하는-프롬프트-엔지니어링)
 - 1) [최신 모델을 사용](#1-최신-모델을-사용)
 - 2) [프롬프트의 시작 부분에 지시사항을 배치](#2-프롬프트의-시작-부분에-지시사항을-배치)
 - 3) [구체적이고 자세하게 설명](#3-구체적이고-자세하게-설명)
 - 4) [원하는 출력 형식을 예시를 통해 명확하게 설명](#4-원하는-출력-형식을-예시를-통해-명확하게-설명)
 - 5) [zero-shot, few_shot, fine-tuning](#5-zero-shot-few_shot-fine-tuning)
 - 6) [모호하고 불필요하게 장황한 표현을 줄이기](#6-모호하고-불필요하게-장황한-표현을-줄이기)
 - 7) [하지 말아야 할 것만 말하는 대신 해야 할 것을 구체적으로 제시](#7-하지-말아야-할-것만-말하는-대신-해야-할-것을-구체적으로-제시)
 - 8) [리딩 워드(leading words) 사용](#8-리딩-워드leading-words-사용)
 - 9) ["Generate Anything" 기능을 사용](#9-generate-anything-기능을-사용)


## OpenAI가 권장하는 프롬프트 엔지니어링
[이전 글](https://juhoplayground.github.io/posts/LangChain-PDF-LLM_Prompt_Engineering/){:target="_blank"}에서는 보편적인 프롬프트 엔지니어링에 대해서 작성했습니다.<br/>
이번 글은 [OpenAI가 권장하는 프롬프트 엔지니어링 방법](https://help.openai.com/en/articles/6654000-best-practices-for-prompt-engineering-with-the-openai-api){:target="_blank"}에 대해서 이야기해보겠습니다.<br/>

#### 1) 최신 모델을 사용
최상의 결과를 얻기 위해 일반적으로 최신의 가장 성능이 뛰어난 모델을 사용하는 것을 권장합니다.<br/>
최신 모델일수록 프롬프트 엔지니어링이 더 용이한 경향이 있습니다. <br/>

#### 2) 프롬프트의 시작 부분에 지시사항을 배치
지시사항과 컨텍스트를 구분할 때 `###` 또는 `"""`를 사용하세요. <br/>

추천하지 않는 방법 ❌
```
Summarize the text below as a bullet point list of the most important points.

{text input here}
```

추천하는 방법 ✅
```
Summarize the text below as a bullet point list of the most important points.

Text: """
{text input here}
"""
```

#### 3) 구체적이고 자세하게 설명
원하는 컨텍스트, 결과, 길이, 형식, 스타일 등에 대해 가능한 한 구체적이고 자세하게 설명하세요.

추천하지 않는 방법 ❌
```
Write a poem about OpenAI. 
```

추천하는 방법 ✅
```
Write a short inspiring poem about OpenAI, focusing on the recent DALL-E product launch (DALL-E is a text to image ML model) in the style of a {famous poet}
```

#### 4) 원하는 출력 형식을 예시를 통해 명확하게 설명


추천하지 않는 방법 ❌
```
Extract the entities mentioned in the text below. Extract the following 4 entity types: company names, people names, specific topics and themes.

Text: {text}
```

추천하는 방법 ✅
```
Extract the important entities mentioned in the text below. First extract all company names, then extract all people names, then extract specific topics which fit the content and finally extract general overarching themes

Desired format:
Company names: <comma_separated_list_of_company_names>
People names: -||-
Specific topics: -||-
General themes: -||-

Text: {text}
```

#### 5) zero-shot, few_shot, fine-tuning
처음에는 zero-shot 방식으로 시작하고 few-shot 방식을 시도하세요.<br/>
둘 다 효과가 없을 경우 fine-tuning을 진행하세요.

Zero-shot
```
Extract keywords from the below text.

Text: {text}

Keywords:
```

Few-shot
```
Extract keywords from the corresponding texts below.

Text 1: Stripe provides APIs that web developers can use to integrate payment processing into their websites and mobile applications.
Keywords 1: Stripe, payment processing, APIs, web developers, websites, mobile applications
##
Text 2: OpenAI has trained cutting-edge language models that are very good at understanding and generating text. Our API provides access to these models and can be used to solve virtually any task that involves processing language.
Keywords 2: OpenAI, language models, text processing, API.
##
Text 3: {text}
Keywords 3:
```

#### 6) 모호하고 불필요하게 장황한 표현을 줄이기 

추천하지 않는 방법 ❌
```
The description for this product should be fairly short, a few sentences only, and not too much more.
```

추천하는 방법 ✅
```
Use a 3 to 5 sentence paragraph to describe this product.
```

#### 7) 하지 말아야 할 것만 말하는 대신 해야 할 것을 구체적으로 제시
추천하지 않는 방법 ❌
```
The following is a conversation between an Agent and a Customer. DO NOT ASK USERNAME OR PASSWORD. DO NOT REPEAT.

Customer: I can’t log in to my account.
Agent:
```

추천하는 방법 ✅
```
The following is a conversation between an Agent and a Customer. The agent will attempt to diagnose the problem and suggest a solution, whilst refraining from asking any questions related to PII. Instead of asking for PII, such as username or password, refer the user to the help article www.samplewebsite.com/help/faq

Customer: I can’t log in to my account.
Agent:
```

#### 8) 리딩 워드(leading words) 사용
특정 패턴을 유도하기 위해 리딩 워드(leading words)를 사용하세요. <br/>
추천하지 않는 방법 ❌
```
# Write a simple python function that
# 1. Ask me for a number in mile
# 2. It converts miles to kilometers
```

추천하는 방법 ✅
아래 코드 예제에서 "import"를 추가하면 모델이 Python 코드 작성을 시작해야 한다는 힌트를 얻습니다. <br/>
마찬가지로 "SELECT"는 SQL 문 작성을 시작하는 좋은 힌트가 됩니다.
```
# Write a simple python function that
# 1. Ask me for a number in mile
# 2. It converts miles to kilometers
 
import
```

#### 9) "Generate Anything" 기능을 사용
개발자는 'Generate Anything' 기능을 사용하여 특정 작업이나 원하는 자연어 출력을 설명하고 이에 맞춘 프롬프트를 받을 수 있습니다.<br/>
- Parameters<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;일반적으로 model과 temperature가 모델 출력을 조정하는 데 가장 많이 사용되는 매개변수입니다.<br/>

- model<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;성능이 높은 모델일수록 비용이 더 많이 들고 지연 시간이 증가할 수 있습니다.<br/>

- temperature<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;모델이 덜 확률적인 토큰을 선택할 가능성을 조절하는 값입니다.<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;값이 높을수록 출력이 더 무작위적(창의적)으로 변합니다.<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;하지만 이는 "진실성(truthfulness)"과는 다른 개념입니다.<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;데이터 추출이나 사실 기반 질의응답(Q&A) 같은 정확성이 중요한 경우, temperature = 0 이 가장 적합합니다.<br/>

- max_tokens(최대 토큰 길이)<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 출력 길이를 직접 제어하는 것이 아니라 토큰 생성을 위한 최대 한도를 설정하는 역할을 합니다.<br/>
이상적으로는 모델이 자체적으로 종료되거나 사용자가 지정한 stop sequence(중단 시퀀스)에 도달하면 멈추므로 이 한도에 도달하는 일이 드물어야 합니다.<br/>

- stop (중단 시퀀스)<br/>
특정 문자(토큰) 세트를 설정하면 모델이 해당 토큰을 생성할 때 출력을 자동으로 중단합니다.<br/>

<br/>

--- 
