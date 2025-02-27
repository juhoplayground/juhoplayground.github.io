---
layout: post
title: Langchain PDF Chatbot 만들기 - 12 - Few Shot Pormpt
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
1. [LangChin에서 FewShot Prompt 구현하기](#langchin에서-fewshot-prompt-구현하기)


## LangChin에서 FewShot Prompt 구현하기
이전 2개의 게시글에서 이야기한 것 처럼 LLM에게 예시를 제공하는 것은 매우 중요한 일입니다.<br/>
그렇다면 LangChain에서는 어떻게 FewShot Prompt를 구현할 수 있을지 알아보도록 하겠습니다.<br/>


```python
from langchain_core.prompts import ChatPromptTemplate, SystemMessagePromptTemplate, HumanMessagePromptTemplate, AIMessagePromptTemplate, FewShotChatMessagePromptTemplate

example_prompt  = ChatPromptTemplate.from_messages([HumanMessagePromptTemplate.from_template("""{human}"""),
                                                            AIMessagePromptTemplate.from_template("""**답변 :** {ai}""")])
examples = [{
                "human":  HumanMessagePromptTemplate.from_template("질문 1"),
                "ai": AIMessagePromptTemplate.from_template("""답변 : 답변 1""")
            },
            {
                "human": HumanMessagePromptTemplate.from_template("질문 2"),
                "ai": AIMessagePromptTemplate.from_template("""답변 : 답변 2""")
            },
            {
                "human": HumanMessagePromptTemplate.from_template("질문 3"),
                "ai": AIMessagePromptTemplate.from_template("""답변 : 답변 3""")
            }]
few_shot_prompt = FewShotChatMessagePromptTemplate(examples=examples, example_prompt=example_prompt)

prompt = ChatPromptTemplate.from_messages([
  SystemMessagePromptTemplate.from_template("You are an expert AI assistant that provides detailed and accurate answers."),
  AIMessagePromptTemplate.from_template("Hello! How can I assist you today?"),
  few_shot_prompt,
  HumanMessagePromptTemplate.from_template("{question}")])
```


위와 같이 Prompt를 구현한다면 예제 데이터를 별도로 관리할 수 있어 새로운 사례를 추가하거나 기존 예제를 업데이트하는 것이 용이합니다. <br/>
위의 코드에서는 하드 코딩으로 examples를 제공하였지만 회사 업무를 진행할때는 별도의 json 파일을 읽어서 처리하도록 하였습니다. <br/>



<br/>

--- 

<br/>
다음 게시글로는 현재 FewShot Prompt가 가지고 있는 문제점에 대해서 이야기해보고 <br/>
그것을 어떻게 개선할 수 있는지에 대해서 작성하도록 하겠습니다.<br/>
