---
layout: post
title: Langchain PDF Chatbot 만들기 - 16 - LongContextReorder
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
1. [LongContextReorder란?](#longcontextreorder란)
2. [LongContextReorder Code](#longcontextreorder-code)

## LongContextReorder란?
모델의 아키텍처와 관계없이 10개 이상의 검색된 문서를 포함할 경우 성능이 상당히 저하됩니다 <br/>
간단히 말해 모델이 긴 문맥 중간에서 관련 정보를 찾아야 할 때 제공된 문서를 무시하는 경향이 있습니다.<br/>
관련 논문 : [https://arxiv.org/abs/2307.03172](https://arxiv.org/abs/2307.03172){:target="_blank"} <br/>
이 문제를 피하기 위해서 검색 후 문서의 순서를 재배열하여 성능 저하를 방지할 수 있습니다.<br/>

## LongContextReorder 문서 정렬 알고리즘 <br/>
1) 먼저 입력 받은 문서 리스트를 reverse() 함수를 사용해 뒤집습니다. <br>
2) 뒤집힌 리스트를 순회하면서 인덱스에 따라 문서의 위치를 결정합니다. <br/>
- 인덱스가 짝수인 경우 : 해당 문서를 결과 리스트의 맨 앞에 삽입 <br/>
- 인덱스가 홀수인 경우 : 해당 문서를 결과 리스트의 맨 뒤에 추가 <br/>

이 과정을 통해 문서들이 중요한 결과가 리스트이 양 끝으로 몰리게 되고 중간에는 상대적으로 관련성이 낮은 문서들이 위치하게 됩니다. <br/>
이 방법을 이용하여 모델이 문서 중간에 있는 중요한 정보를 무시하는 문제를 완화합니다.<br/>

## LongContextReorder Code
```python
from langchain_core.runnables import RunnableLambda, RunnablePassthrough
from langchain_community.document_transformers import LongContextReorder

def reorder_documents(docs):
    long_context_reorder = LongContextReorder()
    reordered_docs = long_context_reorder.transform_documents(docs)
    return reordered_docs

chat_input = (RunnablePassthrough.assign(docs=itemgetter("question") | multi_query_retriever)
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
알고리즘이 간단한 역순 변환 및 짝수/홀수 인덱스 분리 방식을 사용하기 때문에 구현 및 이해가 쉽지만 <br/>
문서 간의 관련성이나 중요도를 미리 평가하지 않고 단순히 인덱스 기반으로 재정렬하므로 모든 상황에서 최적의 배치를 보장하지 못할 수 있습니다. <br/>
그래서 재정렬된 결과가 오히려 문서 간 연결성을 약화시키거나 전체 성능에 부정적 영향을 줄 수 있습니다. <br/>