---
layout: post
title: Langchain PDF Chatbot 만들기 - 5 - Retriever
author: 'Juho'
date: 2025-02-24 09:00:00 +0900
categories: [LangChain]
tags: [LangChain, PDF, Chatbot, FAISS, Python]
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
1. [Retriever](#retriever)
 - 1) [Sparse Retriever](#1-sparse-retriever)
 - 2) [Dense Retriever](#2-dense-retriever)
2. [Vector Store Retriver](#vector-store-retriver)

## Retriever
Retriever는 벡터 저장소에서 사용자가 입력한 질문을 벡터화 → 유사도 검색 → 상위 문서 선정 → 문서 반환하여 문서를 검색하는 과정을 의미합니다.<br/>
간단한 의미 검색에서 부터 성능 향상을 위해 고려할 수 있는 다양한 기법까지 순차적으로 확인해보도록 하겠습니다. <br/>
VectorStore는 임베딩 벡터들을 효율적으로 저장하고 관련 문서를 신속하게 검색할 수 있는 데이터베이스를 의미합니다.<br/>
그렇기 때문에 응답 시간과 정확성에 큰 영향을 미칩니다. <br/>

#### 1) Sparse Retriever
Sparse Retriever는 데이터와 사용자의 질문을 BM25기법을 이용해서 키워드 벡터로 변환하여 처리합니다.<br/>
BM25는 특정 키워드의 존재 여부만 고려하기 때문에 계산 비용이 낮지만 키워드의 의미를 고려하지 않기 때문에 검색 결과의 품질이 상대적으로 떨어집니다.<br/>

#### 2) Dense Retriever
Dense Retriever는 데이터와 사용자의 질문을 임베딩 모델을 이용해서 고차원의 벡터값으로 변환하여 처리합니다.<br/>
키워드가 일치하지 않더라도 의미적으로 관련성이 가장 높은 문서를 제공하기 때문에 상대적으로 검색 결과의 품질이 좋을 수 있습니다.<br/>

두가지 방법의 장단점을 고려하여 목적과 요구사항에 맞게 선택하여 사용하면 됩니다.<br/>

## Vector Store Retriver
이전 게시글의 마지막에서 vector_store를 VectorStoreRetriever로 만드는 방법을 이야기했습니다.<br/>

```python
retriever = vector_store.as_retriever(search_type="similarity", search_kwargs={"k": 4})
```

간단한 chain을 만들어서 실행하는 방법을 확인해보겠습니다.<br/>
```python
from langchain_core.output_parsers import StrOutputParser
from langchain import hub

prompt = hub.pull("rlm/rag-prompt")

llm = ChatOpenAI(model=OPENAI_API_MODEL, temperature=0, openai_api_key=OPENAI_API_KEY)
chain = ({"context": itemgetter("question") | retriever,
         "question": itemgetter("question")}
         | prompt
         | llm
         | StrOutputParser())

response = chain.invoke({'question':question})
```
[prompt](https://smith.langchain.com/hub/rlm/rag-prompt?organizationId=0e147074-f7b1-4a59-92f0-ff22d80f8813){:target="_blank"}는 LangSmith의 Prompt Hub에서 받아왔습니다.<br/>
invoke 메소드를 이용해서 chain을 실행합니다.<br/>
invoke 메소드 외에도 다른 메소드를 이용해서 chain을 실행할 수 있습니다.<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- ainvoke(비동기 실행, 작업이 완료되면 전체 응답 반환), <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- stream(동기 스트리밍, 토큰 단위로 순차적 반환) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- astream (비동기 스트리밍, async iterator를 통한 순차적 반환) <br/>

stream을 사용하기 위해서는 한가지 설정을 추가로 해야합니다.<br/>
```
llm = ChatOpenAI(model=OPENAI_API_MODEL, temperature=0, openai_api_key=OPENAI_API_KEY, streaming=True)
```

<br/>

--- 
