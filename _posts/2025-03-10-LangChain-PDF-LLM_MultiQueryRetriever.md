---
layout: post
title: Langchain PDF Chatbot 만들기 - 15 - MultiQueryRetriever 
author: 'Juho'
date: 2025-03-10 09:00:00 +0900
categories: [LangChain]
tags: [LangChain, PDF, Chatbot, Python]
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
1. [MultiQueryRetriever란?](#multiqueryretriever란)
2. [MultiQueryRetriever 장,단점](#multiqueryretriever-장단점)
 - 1) [장점](#1-장점)
 - 2) [단점](#2-단점)
3. [MultiQueryRetriever Code](#multiqueryretriever-code)

## MultiQueryRetriever란?
기존 Vector Store Retriever는 거리 기반으로 유사한 임베딩을 가진 문서를 찾는 방식입니다.<br/>
하지만 임베딩 데이터의 의미를 제대로 찾지 못한 경우 검색된 문서 결과가 달라질 수 있습니다.<br/>
이러한 한계를 극복하기 위해 고안된 방법으로 MultiQueryRetriever는 주어진 사용자 질의에 대해 LLM을 사용하여 여러 쿼리를 자동으로 생성하는 Retriever입니다.<br/>
각 질의에 대해 문서를 검색하고 모든 문서를 합집합으로 추출하여 더 큰 문서 결과 집합을 반환하도록 합니다. <br/>

## MultiQueryRetriever 장,단점
#### 1) 장점
1) 높은 Context Recall <br/>
- 단일 질의로는 놓치기 쉬운 문맥을 다양한 질의를 통해 더 많은 관련 문서를 검색할 수 있어 context_recall 향상을 기대할 수 있음<br/>

2) LLM 기반의 자동화된 쿼리 확장<br/>
- 사용자가 직접 질의를 수정하지 않아도 LLM이 자동으로 비정형적인 질의나 표현이 어려운 질의도 자연스럽게 확장해줌<br/>

3) 검색 다양성 증대<br/>
- search_type="mmr"과 함께 사용하면 중복 없는 다양한 검색 결과를 얻을 수 있어 하나의 관점에 치우치지 않고 다양한 관점에서 정보를 얻을 수 있음<br/>

#### 2) 단점
1) 속도 저하<br/>
- LLM이 여러 질의를 생성하고 각 질의에 대해 별도로 검색을 진행하므로 검색 속도가 느려질 수 있어 실시간 응답이 중요한 서비스에서는 부적절할 수 있음<br/>

2) 비용 증가<br>
- 여러 질의를 생성하고 각각 검색을 진행하기 때문에 LLM API 호출 비용이 증가함<br/>

3) 정확하지 않은 쿼리 생성 가능성 <br/>
- LLM이 질문 의도와 관련 없는 쿼리 생성하여 잘못된 검색 결과 초래할 수 있음<br/>


## MultiQueryRetriever Code
```python
from langchain.retrievers.multi_query import MultiQueryRetriever

mqr_prompt = PromptTemplate(input_variables=["question"],
                            template="""You are an AI language model assistant. Your task is 
                                    to generate 3 different versions of the given user 
                                    question to retrieve relevant documents from a vector  database. 
                                    By generating multiple perspectives on the user question, 
                                    your goal is to help the user overcome some of the limitations 
                                    of distance-based similarity search. Provide these alternative 
                                    questions separated by newlines. Original question: {question}""")

multi_query_retriever = MultiQueryRetriever.from_llm(retriever=retriever, llm=llm, prompt=mqr_prompt, include_original=True)
multi_query_retriever.verbose = False
```

<br/>
`from_llm`은 LangChain에서 MultiQueryRetriever를 생성하는 메소드입니다.<br/>
`retriever`와 `llm` 파라미터는 이전에 생성한 값을 넣어주면 됩니다.<br/>
<br/>
위 코드의 `mqr_prompt`이 MultiQueryRetriever에서 사용하는 `DEFAULT_QUERY_PROMPT` 입니다.<br/>
더 세밀한 질의 생성을 위해서 사용자가 프롬프트를 수정할 수 있습니다.<br/>
<br/>
`include_original`는 원래 사용자의 질의를 최종 질의 목록에 포함시킬지 여부를 결정하는 값으로 기본값은 False입니다.<br/>
<br/>
`parser_key`는 **`DEPRECATED`**되어 더 이상 사용되지 않으며 지정해서는 안되는 값입니다.<br/>
<br/>
`verbose`는 로그에 생성된 질의를 출력하는 값으로 기본값은 True입니다.<br/>


<br/>

--- 

MultiQueryRetriever는 **정확도와 recall이 중요한 경우**에 적합하고  **속도와 비용이 중요한 경우**에는 비효율적일 수 있습니다.<br/>
