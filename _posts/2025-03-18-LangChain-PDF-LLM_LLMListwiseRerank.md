---
layout: post
title: Langchain PDF Chatbot 만들기 - 19 - LLMListwiseRerank
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
1. [LLMListwiseRerank란?](#llmlistwisererank란)
2. [LLMListwiseRerank 사용 목적](#llmlistwisererank-사용-목적)
3. [LLMListwiseRerank 장,단점](#llmlistwisererank-장단점)
4. [LLMListwiseRerank + LLMChainFilter](#llmlistwisererank--llmchainfilter)
5. [LLMListwiseRerank Code](#llmlistwisererank-code)

## LLMListwiseRerank란?
[Zero-Shot Listwise Document Reranking](https://arxiv.org/pdf/2305.02156){:target="_blank"}을 인용한 Document compressor입니다.<br/>


## LLMListwiseRerank 사용 목적
주어진 쿼리와 가장 관련성이 높은 문서를 우선순위로 재정렬함으로써 전체 문서 목록을 압축하는 목적 <br/>
불필요한 문서를 걸러내고 사용자 질문에 답변하기에 가장 중요한 정보를 담은 문서들만 남도록 함<br/>
내부적으로는 with_structured_output 기능을 갖춘 언어 모델을 사용하여 각 문서의 관련성을 평가<br/>


## LLMListwiseRerank 장,단점
#### 1) 장점
1) 정밀한 관련성 평가<br/>
- 언어 모델을 활용하여 각 문서의 내용을 정교하게 분석하고 주어진 쿼리와의 관련성을 기반으로 문서를 재정렬합니다. 이를 통해 가장 중요한 문서들이 우선적으로 선택됩니다.<br/>

2) Zero-Shot 학습 기반<br/>
- 추가적인 학습 없이 다양한 도메인 및 상황에서 바로 활용할 수 있어 특정 데이터셋에 의존하지 않고 유연하게 적용 가능합니다.<br/>

3) 효율적 문서 압축<br/>
- 불필요한 문서를 걸러내고 핵심 정보를 담은 문서만 남김으로써 전체 문서 집합을 효과적으로 압축하고 검색 속도 및 응답 품질을 향상시킵니다.<br/>

4) 구조화된 출력 지원<br/>
- with_structured_output 기능을 통해 결과를 체계적으로 정리할 수 있어 후속 처리나 추가 분석에 유리합니다.<br/>


#### 2) 단점
1) 모델 요구사항 제약<br/>
underlying model이 with_structured_output 기능을 반드시 구현해야 하므로 해당 기능을 지원하지 않는 모델에서는 사용할 수 없습니다.<br/>

2) 계산 비용 및 자원 소모<br/>
대규모 언어 모델을 사용하기 때문에 처리 시간과 계산 자원이 많이 소모될 수 있으며 실시간 응답이나 대규모 데이터 처리에 제약이 있을 수 있습니다.<br/>

3) 미세 조정의 필요성<br/>
다양한 도메인이나 특수한 상황에서 최적의 성능을 발휘하려면 모델의 미세 조정이 필요할 수 있어 초기 설정 및 추가 비용이 발생할 수 있습니다.<br/>

4) 일반화 한계<br/>
모델이 특정 도메인이나 언어적 특수성에 대해 일반화하는 데 한계가 있을 수 있으며 경우에 따라 쿼리와 문서 간의 관계를 제대로 파악하지 못할 가능성도 있습니다.<br/>


## LLMListwiseRerank + LLMChainFilter
두 방법은 상호 보완적으로 사용할 수 있습니다.<br/>
LLMChainFilter를 통해 초기 문서 집합에서 쿼리와 관련성이 낮은 문서를 먼저 제거한 후 <br/>
남은 문서들을 LLMListwiseRerank로 재정렬하여 최종적으로 가장 관련성 높은 문서를 선택할 수 있습니다<br/>
다만 두 방법 모두 underlying model이 with_structured_output 기능을 지원해야 하며 각 단계의 출력이 다음 단계에 적합하도록 파이프라인을 잘 구성해야 합니다.<br/>
해당 파이프라인을 만들기 위해서는 `DocumentCompressorPipeline`를 사용하면 됩니다.

## LLMListwiseRerank Code
```python
from langchain.retrievers import ContextualCompressionRetriever, EnsembleRetriever
from langchain.retrievers.document_compressors import LLMChainFilter, LLMListwiseRerank, DocumentCompressorPipeline


ensemble_retriever = EnsembleRetriever(retrievers=[bm25_retriever, multi_query_retriever], weights=[0.4, 0.6])

pipeline_compressor = DocumentCompressorPipeline(transformers=[LLMChainFilter.from_llm(llm=llm),
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
