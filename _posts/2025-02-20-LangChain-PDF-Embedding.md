---
layout: post
title: Langchain PDF Chatbot 만들기 - 3 -  Embedding
author: 'Juho'
date: 2025-02-20 09:00:00 +0900
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
1. [Embedding](#embedding)
 - 1) [Embedding 모델 종류](#1-embedding-모델-종류)
 - 2) [OpenAI Embedding 사용하는 방법](#2-openai-embedding-사용하는-방법)
 - 3) [Cache Embedding](#3-cache-embedding)

## Embedding
임베딩이란 텍스트, 소리, 이미지 등의 데이터를 고정 길이의 실수 형태의 벡터로 표현하는 것을 의미합니다. <br/>
벡터 값들간의 거리 계산을 통해서 의미적 관계를 유추할 수 있습니다. <br/>
→ TensorFlow에서 제공하는 [Embedding Projector](https://projector.tensorflow.org/){:target="_blank"}를 통해서 벡터값을 3차원 공간에 투영한 결과를 확인해볼 수 있습니다. <br/>


#### 1) Embedding 모델 종류
```markdown
| 구분                             | 제공                                               | 장점                                                                 | 단점                                |
|----------------------------------|----------------------------------------------------|----------------------------------------------------------------------|-------------------------------------|
| Cloud Service Embedding Model    | OpenAI, Amazon, Anthropic과 같은 플랫폼에서 제공      | 개인의 하드웨어 사양에 대해 크게 신경쓰지 않아도 됨<br>다국어에 대한 임베딩 보장 | 유료                                |
| OpenSource Embedding Model       | HuggingFace에서 제공하는 무료 오픈소스                  | 무료                                                                 | 개인의 하드웨어 사양을 고려<br>다국어에 대한 지원 아쉬움 |
```
<br/>

처음에는 HuggingFace에서 Embedding Model을 사용했는데 유사도를 계산할 때 정확하지 못한 것이 많았습니다.<br/>
왠만하면 OpenSource로 작업을 하는데 유료 Embedding Model을 사용하기로 결정했습니다.<br/>
그 중에서도 OpenAI를 사용하기로 결정했습니다. (나중에 o1, o3 모델 사용 가능성을 위해서) <br/>

OpenAI에서는 아래와 같은 임베딩 모델을 제공한다.<br/>
```markdown
| Model                     | ~ Pages per dollar | Performance on MTEB eval | Max input |
|---------------------------|--------------------|--------------------------|-----------|
| text-embedding-3-small    | 62,500             | 62.3%                    | 8191      |
| text-embedding-3-large    | 9,615              | 64.6%                    | 8191      |
| text-embedding-ada-002    | 12,500             | 61.0%                    | 8191      |
```
MTEB(Massive Text Embedding Benchmark) 평가에서 모델의 성능을 나타내는 값으로<br/>
다양한 텍스트 임베딩 관련 태스크(문장 유사도, 검색, 분류, 클러스터링)를 얼마나 잘 수행하는지에 대한 백분율입니다.<br/>
text-embedding-3-large가 제공되는 임베딩 모델 중에서 가장 높은 값을 제공하기 때문에 선택하였습니다.<br/>
하지만 1달러로 9,615 페이지를 처리를 해 임베딩 모델 중에서 1달러 대비 처리할 수 있는 페이지 수가 가장 적습니다.<br/>
성능과 비용에 대한 트레이드 오프를 잘 비교해서 임베딩 모델을 선택하면 될 것 같습니다.<br/>
<br/>
<br/>
처음에는 어떻게든 비용을 줄이기 위해서 `text-embedding-3-small` 모델을 사용했는데 <br/>
임베딩 결과물의 품질이 만족스럽지가 않아서 결국에는 `text-embedding-3-large`을 사용하기로 하였습니다.<br/>


#### 2) OpenAI Embedding 사용하는 방법
`pip install -U langchain langchain_openai`로 필요한 라이브러리를 설치합니다. <br/>

```python
from langchain_openai import OpenAIEmbeddings
embedding = OpenAIEmbeddings(model=OPENAI_API_EMBEDDING, openai_api_key=OPENAI_API_KEY)
```

위의 형태로 사용이 가능합니다. <br/>
<br/>
text-embedding-3-large은 기본 값으로 3072 차원을 가지만 다른 모델은 1536 차원을 가집니다. <br/>

```python
embedding = OpenAIEmbeddings(model="text-embedding-3-large", openai_api_key=OPENAI_API_KEY)
dim = embedding.embed_query('테스트')
print(len(dim))

embedding = OpenAIEmbeddings(model="text-embedding-3-small", openai_api_key=OPENAI_API_KEY)
dim = embedding.embed_query('테스트')
print(len(dim))

embedding = OpenAIEmbeddings(model="text-embedding-ada-002", openai_api_key=OPENAI_API_KEY)
dim = embedding.embed_query('테스트')
print(len(dim))
```

하지만 dimensions 파라미터를 이용해서 임베딩의 크기를 원하는 값으로 수정할 수 있습니다.<br/>

```python
embedding = OpenAIEmbeddings(model="text-embedding-3-large", openai_api_key=OPENAI_API_KEY, dimensions=1536)
dim = embedding.embed_query('테스트')
print(len(dim))
```

<br/>
`OpenAIEmbeddings.embed_query()`를 이용해서 입력된 텍스트를 임베딩 벡터로 반환이 가능하며 <br/>
`OpenAIEmbeddings.embed_documents()`를 이용해서 텍스트 문서를 임베딩 벡터로 반환이 가능합니다.<br/>


#### 3) Cache Embedding
동일한 내용에 대해서 임베딩을 반복하지 하지 않기 위해서 `CacheBackedEmbeddings`을 이용해 캐싱을 사용할 수 있습니다. <br/>
CacheBackedEmbeddings의 주요 파라미터는 아래와 같습니다.<br/>
- underlying_embeddings : 임베딩을 위해 사용되는 embedder<br/>
- document_embedding_cache : 문서 임베딩을 캐싱하기 위한 ByteStore<br/>
- namespace : 동일한 텍스트가 다른 임베딩 모델을 사용하여 임베딩될 때 충돌을 피하기 위해 설정<br/>

1) 영구적인 Cache Embedding<br/>
```python
from langchain.embeddings import CacheBackedEmbeddings
from langchain_openai import OpenAIEmbeddings
from langchain.storage import LocalFileStore

embedding = OpenAIEmbeddings(model=OPENAI_API_EMBEDDING, openai_api_key=OPENAI_API_KEY)
cached_embedding = CacheBackedEmbeddings.from_bytes_store(underlying_embeddings=embedding,
                                                          document_embedding_cache=LocalFileStore(EMBEDDING_CACHE_FOLDER),
                                                          namespace=embedding.model)
```

2) 비영구적인 Cache Embedding<br/>
```python
from langchain.embeddings import CacheBackedEmbeddings
from langchain_openai import OpenAIEmbeddings
from langchain.storage import InMemoryByteStore


embedding = OpenAIEmbeddings(model=OPENAI_API_EMBEDDING, openai_api_key=OPENAI_API_KEY)
cached_embedding = CacheBackedEmbeddings.from_bytes_store(underlying_embeddings=embedding,
                                                          document_embedding_cache=InMemoryByteStore(),
                                                          namespace=embedding.model)
```

이렇게 설정할 경우 동일한 내용에 대해서는 재계산을 하지 않기 때문에 임베딩에 사용하는 비용을 소폭 줄일 수 있습니다. <br/>
<br/>

--- 
