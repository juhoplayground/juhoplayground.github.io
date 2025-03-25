---
layout: post
title: Langchain PDF Chatbot 만들기 - 20 - LLMChainExtractor
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
1. [LLMChainExtractor란?](#llmchainextractor란)
2. [LLMChainExtractor 사용 목적](#llmchainextractor-사용-목적)
3. [LLMChainExtractor 장,단점](#llmchainextractor-장단점)
4. [LLMChainExtractor + LLMListwiseRerank + LLMChainFilter](#llmchainextractor--llmlistwisererank--llmchainfilter)
5. [LLMChainExtractor Code](#llmchainextractor-code)

## LLMChainExtractor란?
문서의 관련 부분을 추출하기 위해 LLM 체인을 사용하는 Document compressor입니다.<br/>


## LLMChainExtractor 사용 목적
LLMChainExtractor의 주된 목적은 LLM 체인을 활용하여 문서 내에서 필요한 핵심 정보나 관련 내용을 추출하는 것입니다.<br/>
이를 통해 전체 문서에서 불필요한 부분을 제거하고 핵심적인 내용만을 압축 및 요약하여 보다 효과적인 정보 처리가 가능하도록 돕습니다.<br/>

## LLMChainExtractor 장,단점
#### 1) 장점
1) 관련성 향상<br/>
- 문서에서 쿼리와 관련된 부분만 추출하여 검색 결과의 정확도를 높입니다.<br/>

2) 토큰 효율성<br/>
- 불필요한 텍스트를 제거하여 LLM 처리에 필요한 토큰 수를 줄임으로써 비용과 처리 시간을 절약합니다.<br/>

3) 컨텍스트 창 최적화<br/>
- 제한된 컨텍스트 창을 더 효율적으로 사용할 수 있게 해줍니다.<br/>

4) 노이즈 감소<br/>
- 관련 없는 정보를 필터링하여 LLM이 더 집중된 응답을 생성하도록 합니다.<br/>


#### 2) 단점
1) 추가 지연 시간<br/>
문서 압축 과정이 전체 검색 파이프라인에 추가 지연을 발생시킵니다.<br/>

2) 추가 비용<br/>
압축을 위해 LLM 호출이 추가되므로 API 비용이 증가할 수 있습니다.<br/>

3) 정보 손실 위험<br/>
압축 과정에서 중요한 정보가 실수로 제거될 수 있습니다.<br/>

4) LLM 의존성<br/>
압축 품질이 기반 LLM의 성능에 크게 의존합니다.<br/>

5) 복잡성 증가<br/>
검색 파이프라인에 추가적인 구성 요소를 도입하여 시스템 복잡성이 증가합니다.<br/>


## LLMChainExtractor + LLMListwiseRerank + LLMChainFilter
LLMChainExtractor를 먼저 사용해서 각 문서에서 관련 부분만 추출하여 다음 단계에서 처리할 텍스트 양을 줄입니다. <br/>
이는 후속 단계들의 효율성을 높이고 토큰 사용량을 감소시킵니다.<br/>
그 다음 LLMChainFilter를 사용해서 압축된 문서 중에서 관련성이 낮은 것들을 완전히 제거합니다. <br/>
이렇게 하면 재정렬이 필요한 문서 수가 줄어들어 다음 단계의 계산 비용을 절감할 수 있습니다. <br/>
마지막으로 LLMListwiseRerank를 사용해서 필터링된 관련 문서들만을 대상으로 재정렬을 수행합니다. <br/>
하지만 이 방법은 너무 과도하게 필터링이 될 경우 잠재적으로 중요한 정보를 함께 제거할 위험이 있습니다.
<br/>
그래서 LLMChainExtractor → LLMListwiseRerank → LLMChainFilter 순서를 사용해도 됩니다.<br/>
먼저 가능한 한 많은 후보 항목을 추출하여 보존한 다음 이 후보들을 재정렬로 우선순위를 부여하고 마지막으로 필터링을 통해 최종적으로 정제하는 과정을 거치게 됩니다 <br/>
이 접근 방식은 초기 추출 단계에서 정보를 최대한 보존한 후 이후 단계에서 정밀하게 정제할 수 있는 장점이 있습니다.<br/>
<br/>
데이터의 특성과 처리 환경 그리고 최종 결과에 대한 요구사항에 따라 달라질 수 있습니다.<br/>
각 방식의 트레이드오프를 고려하여 상황에 맞게 조정하는 것이 중요합니다.<br/>

## LLMChainExtractor Code
```python
from langchain.retrievers import ContextualCompressionRetriever, EnsembleRetriever
from langchain.retrievers.document_compressors import LLMChainFilter, LLMListwiseRerank, DocumentCompressorPipeline


ensemble_retriever = EnsembleRetriever(retrievers=[bm25_retriever, multi_query_retriever], weights=[0.4, 0.6])

pipeline_compressor = DocumentCompressorPipeline(transformers=[LLMChainExtractor.from_llm(llm=llm),
                                                               LLMChainFilter.from_llm(llm=llm),
                                                               LLMListwiseRerank.from_llm(llm=llm, top_n=4)])
compression_retriever = ContextualCompressionRetriever(base_compressor=pipeline_compressor,
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
