---
layout: post
title: Langchain PDF Chatbot 만들기 - 13 - Example Selector
author: 'Juho'
date: 2025-02-28 09:00:00 +0900
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
1. [지난 Few-shot Prompt의 문제점](#지난-few-shot-prompt의-문제점)
2. [해결 방법 Example Selector](#해결-방법-example-selector)

## 지난 Few-shot Prompt의 문제점
[지난 게시글](https://juhoplayground.github.io/posts/LangChain-PDF-LLM_Few_Shot_Prompt/){:target="_blank"}에서 구현한 Few-shot Prompt에는 문제점이 있습니다.<br/>
그것은 의도와 다르게 질의와 관련이 없는 다수의 예제가 프롬프트에 포함이 되어 모델의 응답 품질이 떨어지거나 혼란이 발생하는 것 입니다.<br/>
또한 예제가 많아질 수록 컨텍스트 길이가 늘어나 모델이 주의해야 할 핵심 정보가 희석되는 현상이 발생하는 것 입니다.


## 해결 방법 Example Selector
해당 문제는 Example Selector를 사용하여 해결할 수 있습니다.<br/>
Example Selector는 사용자의 질의와 각 예제 간의 의미적 유사도를 정량적으로 평가하여 <br/>
상위 몇 개의 가장 관련성 높은 예제만을 선택함으로써 불필요한 예제가 모델의 응답에 혼입되어 발생할 수 있는 혼란을 최소화합니다. <br/>

```python
from langchain_core.prompts import ChatPromptTemplate, SystemMessagePromptTemplate, HumanMessagePromptTemplate, AIMessagePromptTemplate, FewShotChatMessagePromptTemplate
from langchain_core.example_selectors import SemanticSimilarityExampleSelector

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

example_selector = await SemanticSimilarityExampleSelector.afrom_examples(examples=examples, embeddings=embedding, k=4, vectorstore_cls=FAISS)
few_shot_prompt = FewShotChatMessagePromptTemplate(example_selector=example_selector, example_prompt=example_prompt, input_variables=["query"])

prompt = ChatPromptTemplate.from_messages([
  SystemMessagePromptTemplate.from_template("You are an expert AI assistant that provides detailed and accurate answers."),
  AIMessagePromptTemplate.from_template("Hello! How can I assist you today?"),
  few_shot_prompt.format(question=question),
  HumanMessagePromptTemplate.from_template("{question}")])
```

기존 코드에서 달라진 내용은 <br/>
```
example_selector = await SemanticSimilarityExampleSelector.afrom_examples(examples=examples, embeddings=embedding, k=4, vectorstore_cls=FAISS)
few_shot_prompt = FewShotChatMessagePromptTemplate(example_selector=example_selector, example_prompt=example_prompt, input_variables=["query"])
```
이 부분입니다.<br/>

자세히 설명하자면 <br/>
`examples`는 선태 가능한 전체 예제 목록<br/>
`embeddings`는 유사도를 측정하는데 사용되는 임베딩 클래스 <br/>
`k`는 선택할 예시의 개수로 기본 값은 4입니다.<br/>
`vectorstore_cls`는 유사도를 측정하는데 사용되는 VectorStore 클래스<br/>


<br/>

--- 