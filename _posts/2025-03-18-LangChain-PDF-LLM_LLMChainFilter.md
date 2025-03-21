---
layout: post
title: Langchain PDF Chatbot 만들기 - 18 - LLMChainFilter
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
1. [LLMChainFilter란?](#llmchainfilter란)
2. [LLMChainFilter 사용 목적](#llmchainfilter-사용-목적)
3. [LLMChainFilter 장,단점](#llmchainfilter-장단점)
4. [LLMChainFilter Code](#llmchainfilter-code)

## LLMChainFilter란?
LangChain의 LLMChainFilter는 검색된 문서 중에서 사용자의 쿼리와 관련성이 높은 문서만 필터링하는 문서 압축기입니다.<br/>

## LLMChainFilter 사용 목적
관련성 필터링: 검색된 많은 문서 중에서 실제로 쿼리와 관련된 문서만 선별<br/>
노이즈 감소: 검색 결과에서 관련 없는 정보를 제거하여 품질 향상<br/>
컨텍스트 최적화: LLM에 전달되는 context window를 효율적으로 사용<br/>

## LLMChainFilter 장,단점
#### 1) 장점
1) 정밀한 필터링 <br/>
- LLM의 언어 이해 능력을 활용하여 문서의 관련성과 중요도를 보다 정밀하게 평가할 수 있습니다.<br/>

2) 비용 및 시간 절감<br/>
- 불필요한 정보를 미리 제거함으로써 후속 처리(예: 검색, 질의응답 등)의 연산 비용과 시간을 줄여줍니다.<br/>


#### 2) 단점
1) LLM 의존성<br/>
- 결과의 정확성이 LLM의 성능에 크게 좌우되므로 모델이 부정확한 판단을 내릴 경우 중요한 정보가 잘못 필터링될 위험이 있습니다.<br/>

2) 추가 연산 비용<br>
- LLM을 사용하는 과정에서 추가적인 계산 자원이 필요해, 비용 및 처리 시간이 늘어날 수 있습니다.<br/>


## LLMChainFilter Code
```python
from langchain.retrievers import ContextualCompressionRetriever, EnsembleRetriever
from langchain.retrievers.document_compressors import LLMChainFilter


ensemble_retriever = EnsembleRetriever(retrievers=[bm25_retriever, multi_query_retriever], weights=[0.4, 0.6])
document_compressor = LLMChainFilter.from_llm(llm=llm)
compression_retriever = ContextualCompressionRetriever(base_compressor=document_compressor,
                                                        base_retriever=ensemble_retriever)

chat_input = (RunnablePassthrough.assign(docs=itemgetter("question") | compression_retriever)
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
