---
layout: post
title: Langchain PDF Chatbot 만들기 - 9 - LLM PromptTemplate 설정
author: 'Juho'
date: 2025-02-26 09:00:00 +0900
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
1. [PromptTemplate](#prompttemplate)
2. [문자열 기반 프롬프트](#문자열-기반-프롬프트)
3. [PromptTemplate으로 변경](#prompttemplate으로-변경)
4. [ChatPromptTemplate으로 변경](#chatprompttemplate으로-변경)


## PromptTemplate
프롬프트는 LLM에게 원하는 답변을 얻기 위해 입력 내용을 의미합니다.<br/>
기존 문자열의 프롬프트를 사용해도 되지만 PromptTemplate을 이용하면 <br/>
프롬프트의 구조가 명확해지며 역할별 메세지를 쉽게 구분할 수 있어 가독성과 유지보수성이 향상됩니다.<br/>
LLM은 역할별 메세지를 다르게 해석을 하기 때문에 명확히 구분한다면 더 효과적인 응답을 생성할 수 있습니다. <br/>
PromptTemplate을 사용하면 이전 대화의 맥락을 유지할 수 있어 챗봇이 보다 자연스럽고 정교한 대화를 할 수 있습니다.<br/>


## 문자열 기반 프롬프트
```python

prompt = """You are an expert AI assistant that provides detailed and accurate answers.

AI: Hello! How can I assist you today?

User: {user_input}

AI: Sure! Here’s what I found: {ai_response}"""

```
최초에는 이러한 문자열 형태의 프롬프트를 작성했을 것 입니다.<br/>


## PromptTemplate으로 변경
```
prompt = PromptTemplate(
    input_variables=["user_input", "ai_response"],
    template="""You are an expert AI assistant that provides detailed and accurate answers.

AI: Hello! How can I assist you today?

User: {user_input}

AI: Sure! Here’s what I found: {ai_response}"""
)

```

문자열 형태의 프롬프트를 PromptTemplate으로 변경을 하였습니다.<br/>
하지만 PromptTemplate은 단순한 템플릿을 생성하는데 사용이 되며 역할 구분 없이 하나의 문자열로 프롬프트를 생성합니다.<br/>
역할 구분이 명확하지 않기 때문에 챗봇 보다는 일반적인 단일 질문 - 응답 형태의 프롬프트에 적합한 구성입니다.<br/>


## ChatPromptTemplate으로 변경
챗봇의 역할에 맞게 System, Human, AI 메시지를 구조적으로 관리하도록 변경을 해보겠습니다.<br/>

```python
from langchain_core.prompts import ChatPromptTemplate, SystemMessagePromptTemplate, HumanMessagePromptTemplate, AIMessagePromptTemplate

prompt = ChatPromptTemplate.from_messages([
    SystemMessagePromptTemplate.from_template("You are an expert AI assistant that provides detailed and accurate answers."),
    AIMessagePromptTemplate.from_template("Hello! How can I assist you today?"),
    HumanMessagePromptTemplate.from_template("{user_input}"),
    AIMessagePromptTemplate.from_template("Sure! Here’s what I found: {ai_response}")
])

```

이렇게 구성하게 되면 AI의 역할을 명확하게 정의하면서도 대화 흐름을 정리하는 데 도움이 됩니다.<br/>
<br/>

--- 
