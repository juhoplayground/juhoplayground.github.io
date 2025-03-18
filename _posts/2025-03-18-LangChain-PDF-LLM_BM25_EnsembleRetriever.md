---
layout: post
title: Langchain PDF Chatbot 만들기 - 17 - BM25Retriever (EnsembleRetriever)
author: 'Juho'
date: 2025-03-18 09:00:00 +0900
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
1. [BM25Retriever란?](#bm25retriever란)
2. [BM25Retriever Code](#bm25retriever-code)
3. [EnsembleRetriever란?](#ensembleretriever란)
4. [EnsembleRetriever Code](#ensembleretriever-code)

## BM25Retriever란?
BM25Retriever는 정보 검색 분야에서 널리 사용되는 BM25(Best Matching 25) 즉 Okapi BM25 알고리즘을 기반으로 하는 Retriever 입니다.<br/>
문서 내 단어 빈도와 역문서 빈도를 활용하여 주어진 검색 쿼리에 대해 각 문서의 관련성을 수치화합니다.<br/>
이를 통해 입력된 문서 집합과 검색 쿼리를 토대로 가장 관련성 높은 문서를 찾아내는 역할을 수행합니다.<br/>
`pip install -U rank_bm25`로 필요한 라이브러리를 설치합니다.

## BM25Retriever Code
```python
from langchain.retrievers import BM25Retriever
from nltk.tokenize import word_tokenize

bm25_retriever = BM25Retriever.from_documents(documents, k=4, preprocess_func=word_tokenize)
```

## EnsembleRetriever란?
EnsembleRetriever는 여러 Retriever의 결과를 결합합니다.<br/>
서로 다른 알고리즘의 강점을 활용함으로써 EnsembleRetriever는 단일 알고리즘보다 우수한 성능을 달성할 수 있습니다.<br/>
가장 일반적인 방식은 BM25와 같은 Sparse Retriever와 임베딩 유사도 기반의  Dense Retriever를 결합하는 것으로 이 두 방식은 상호 보완적인 강점을 가지고 있습니다.<br/>
Sparse Retriever는 키워드를 기반으로 관련 문서를 찾는 데 뛰어난 반면 Dense Retriever는 의미론적 유사성을 바탕으로 관련 문서를 효과적으로 검색합니다.<br/>
이를 `하이브리드 검색`이라고도 합니다.<br/>

## EnsembleRetriever Code
```python
from langchain_core.runnables import RunnableLambda, RunnablePassthrough
from langchain.retrievers import EnsembleRetriever


ensemble_retriever = EnsembleRetriever(retrievers=[bm25_retriever, multi_query_retriever], weights=[0.4, 0.6])

chat_input = (RunnablePassthrough.assign(docs=itemgetter("question") | ensemble_retriever)
              ) | {"context": itemgetter("docs") | RunnableLambda(reorder_documents),
                  "question": itemgetter("question"),
                  "chat_history": itemgetter("chat_history")}

chain = (chat_input
        | prompt
        | llm
        | StrOutputParser())
```



<br/>

--- 

<br/>
