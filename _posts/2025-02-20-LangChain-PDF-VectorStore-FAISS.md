---
layout: post
title: Langchain PDF Chatbot 만들기 - 3 - VectorStore(FAISS)
author: 'Juho'
date: 2025-02-20 09:00:00 +0900
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
1. [VectorStore](#vectorstore)
 - 1) [VectorStore 종류](#1-vectorstore-종류)
 - 2) [VectorStore 선택 과정](#2-vectorstore-선택-과정)
 - 3) [VectorStore Indexing](#3-vectorstore-indexing)
2. [FAISS](#faiss)
 - 1) [FAISS 생성](#1-faiss-생성)
 - 2) [FAISS 삽입 및 삭제](#2-faiss-삽입-및-삭제)
 - 3) [FAISS 저장 및 호출](#3-faiss-저장-및-호출)
 - 4) [FAISS 병합](#4-faiss-병합)
 - 5) [FAISS 유사도 검색](#5-faiss-유사도-검색)
 - 6) [FAISS VectorStoreRetriever ](#6-faiss-vectorstoreretriever)

## VectorStore
VectorStore는 임베딩 벡터들을 효율적으로 저장하고 관련 문서를 신속하게 검색할 수 있는 데이터베이스를 의미합니다.<br/>
그렇기 때문에 응답 시간과 정확성에 큰 영향을 미칩니다. <br/>

#### 1) VectorStore 종류
Apache Doris, Azure Cosmos DB Vector Search, BigQuery Vector Search, Cassandra, Chroma, Clickhouse<br/>
DuckDB, Elastic Vector Search, FAISS, Milvus, MongoDB Atlas Vector Search, Neo4j Vector<br/>
OpenSearch Vector Search, Oracle AI Vector Search, PGVector, Pinecone, Redis Vector Search, Weaviate와 같은 다양한 VectorStore가 있습니다.<br/>

#### 2) VectorStore 선택 과정
후보군을 좁히기 위해, 벡터 검색 기능이 핵심인 전용 벡터 데이터베이스와 기존 데이터베이스에 벡터 검색 기능을 추가한 솔루션을 분류했습니다.<br/>
- 전용 벡터 데이터베이스<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Chroma, FAISS, Milvus, OpenSearch Vector Search, Oracle AI Vector Search, Pinecone, Weaviate<br/>

- 기존 DB 확장형<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Apache Doris, Azure Cosmos DB Vector Search, BigQuery Vector Search, Cassandra, ClickHouse, DuckDB<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Elastic Vector Search, MongoDB Atlas Vector Search, Neo4j Vector, PGVector, Redis Vector Search<br/>

전용 벡터 데이터베이스 → 벡터 검색 성능이 뛰어나며 최적화가 잘되어 있음.<br/>
기존 DB 확장형 → 기존 데이터와 함께 활용 가능하지만, 성능 최적화가 덜 되어 있음.<br/>
저는 기존 DB와 연결할 필요가 없고 검색 정확도와 속도를 최우선으로 고려했기 때문에 전용 벡터 데이터베이스를 선택했습니다.<br/>

비용이 발생하지 않는 오픈 소스 솔루션을 선택하기 위해 라이선스를 확인했습니다.<br/>
무료 벡터 스토리지<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Chroma (MIT License) – 무료, 로컬 및 자체 호스팅 가능<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;FAISS (MIT License) – 무료, 빠른 벡터 검색 라이브러리<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Milvus (Apache 2.0) – 무료, 분산 및 클러스터링 지원<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;OpenSearch Vector Search (Apache 2.0) – 무료, Elasticsearch 기반<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Weaviate (BSD-3-Clause) – 무료, 자체 호스팅 가능<br/>

유료 벡터 스토리지<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Oracle AI Vector Search – 유료 (Oracle Cloud 서비스)<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Pinecone – 관리형 유료 (제한된 무료 요금제 제공, 자체 호스팅 불가능)<br/>
<br/>
결과적으로 Oracle AI Vector Search, Pinecone은 제외했습니다.<br/>
<br/>
무료 전용 벡터 데이터베이스 중에서, 최종적으로 실습할 2개를 선택하기 위해 추가적으로 검토했습니다.<br/>
제외한 스토리지<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;OpenSearch Vector Search → Elasticsearch 환경이 필요 없으면 굳이 사용할 이유가 없고, 검색 속도 최적화가 부족.<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Milvus → 대규모 데이터셋을 다루기에 적합하지만, 클러스터 설정이 복잡하여 운영 부담이 있음.<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Weaviate → 메타데이터 필터링이 강점이지만, 메타데이터 필터링을 활용할 계획이 없으며 초기 설정이 복잡함.<br/>

최종 선택: Chroma, FAISS<br/>
사용법 자체는 크게 다르지 않으며, 특정 벡터 스토리지로 변경하는 것이 불가능하지는 않았습니다.<br/>
하지만 Chroma를 사용하면서 관련성이 낮은 문서가 조회되는 경우가 있었습니다.<br/>

#### 3) VectorStore Indexing
Chroma → ANN(Approximate Nearest Neighbor) 방식 사용 (HNSW 인덱싱)<br/>
HNSW의 Recall: 0.5 ~ 0.95<br/>

FAISS → 기본적으로 IP(Inner Product) 방식을 사용하며 LSH, HNSW, IVF 등의 인덱싱 기법 지원<br/>
IP 방식의 Recall: 1.0<br/>
OpenAI 임베딩 모델은 정규화된 벡터를 반환하므로 IP 방식이 코사인 유사도 방식과 동일하게 동작함.<br/>
단점: Brute force 방식이라 처리 속도가 저하될 가능성이 있음.<br/>

IP 방식으로 인한 처리 속도가 지나치게 늘어날 경우 다른 인덱싱을 고려하기 위해 다른 인덱싱 기법을 비교하였습니다.<br/>
- LSH (Locality Sensitive Hashing)<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;비슷한 데이터 포인트를 동일한 해시 버킷에 매핑하여 검색 속도를 향상.<br/>

- HNSW (Hierarchical Navigable Small World Graphs)<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;계층적인 그래프를 활용하여 빠르고 정확한 검색 가능.<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;대규모 데이터셋에서도 성능이 우수함.<br/>

- IVF (Inverted File Index)<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;데이터를 미리 클러스터링한 후, 특정 클러스터 내에서만 탐색하여 속도를 개선.<br/>
<br/>
다양한 인덱싱을 지원하는 FIASS를 사용하기로 최종 결정을 하였습니다.<br/>

## FAISS
FAISS(Facebook AI Similarity Search)는 Facebook AI Research에서 개발한 라이브러리로 벡터의 효율적인 유사도 검색과 클러스터링을 지원합니다.<br/>
FAISS RAM에 맞지 않을 정도로 대규모인 벡터 집합도 처리할 수 있는 알고리즘을 포함하고 있으며 평가 및 매개변수 튜닝을 위한 지원 코드도 제공하여<br/>
벡터의 압축된 표현을 활용함으로써 메모리 사용량을 최소화하고 검색 속도를 극대화하는 특징을 지니고 있습니다.<br/>
`pip install faiss-cpu langchain langchain_community langchain_openai`로 필요한 라이브러리 설치합니다.<br/>

#### 1) FAISS 생성
```python
from app.config.openai_config import OPENAI_API_KEY, OPENAI_API_EMBEDDING
from app.config.server_config import EMBEDDING_CACHE_FOLDER

from langchain_community.docstore.in_memory import InMemoryDocstore
from langchain_community.vectorstores.utils import DistanceStrategy
from langchain.embeddings import CacheBackedEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings
from langchain.storage import LocalFileStore
import faiss

embedding = OpenAIEmbeddings(model=OPENAI_API_EMBEDDING, openai_api_key=OPENAI_API_KEY)
cached_embedding = CacheBackedEmbeddings.from_bytes_store(underlying_embeddings=embedding,
                                                          document_embedding_cache=LocalFileStore(EMBEDDING_CACHE_FOLDER),
                                                          namespace=embedding.model)


vector_store = FAISS(embedding_function=cached_embedding,
                     index=faiss.IndexFlatIP(3072),
                     docstore=InMemoryDocstore(),
                     index_to_docstore_id={},
                     distance_strategy=DistanceStrategy.COSINE)
```
`3072`는 임베딩 모델의 차원 값으로 사용하는 임베딩 모델에 맞게 설정하면 됩니다. <br/>
`IndexFlatIP`로 index를 설정하였는데 [FAISS Index](https://github.com/facebookresearch/faiss/wiki/Faiss-indexes){:target="_blank"}를 확인하고 원하는 값으로 설정하면 됩니다.<br/>
`docstore`는 사용할 문서 저장소로 저는 `InMemoryDocstore()`로 설정했습니다.<br/>
`index_to_docstore_id`는 `{}`로 우선 설정하고 VectorStore를 생성하였습니다.<br/>
`distance_strategy`를 이용해서 유사도를 측정하는 방법을 설정할 수 있는데 `COSINE`으로 설정하였습니다. (기본 값 : DistanceStrategy.EUCLIDEAN_DISTANCE)<br/>
[OpenAI Distance Guide](https://platform.openai.com/docs/guides/embeddings/#which-distance-function-should-i-use){:target="_blank"}에서 코사인 방식을 권장한다고 합니다.<br/>

#### 2) FAISS 삽입 및 삭제
- 삽입 <br/>

```python
uuids = [str(uuid4()) for _ in range(len(documents))]
vector_store.add_documents(documents=documents, ids=uuids)
```

- 삭제 <br/>

```python
vector_store.delete(ids=ids_to_delete)
```

#### 3) FAISS 저장 및 호출
- 저장 <br/>

```python
vector_store.save_local(folder_path=FAISS_PATH, index_name=FAISS_INDEX_NAME)
```

- 호출 <br/>

```python
vector_store = FAISS.load_local(folder_path=FAISS_PATH, index_name=FAISS_INDEX_NAME, embeddings=cached_embedding, allow_dangerous_deserialization=True)
```

#### 4) FAISS 병합
```python
vector_store1.merge_from(vector_store2)
```

#### 5) FAISS 유사도 검색
```python

vector_store.similarity_search("검색할 내용", filter={"source": ""}, fetch_k=20 , k=4) # 

vector_store.similarity_search_with_score("검색할 내용", filter={"source": ""}, fetch_k=20, k=4)
```
<br/>
`similarity_search`는 주어진 쿼리와 가장 유사한 문서 리턴합니다.<br/>
`similarity_search_with_score`는  주어진 쿼리와 가장 유사한 문서, 유사도 점수와 함께 리턴합니다.<br/>

`filter` 파라미터를 이용해서 metadata 필터링이 가능합니다. <br/>
`fetch_k` 파라미터를 이용해서 필터링 전 검색할 문서 수를 조절할 수 있습니다. (기본 값 : 20)<br/>
`k` 파라미터를 이용해서 리턴할 문서의 수를 조절할 수 있습니다. (기본 값 : 4)<br/>

그리고 MMR(Maximal Marginal Relevance) 방식으로 검색할 수도 있습니다.<br/>
특정 쿼리에 대해 관련성이 높으면서도 서로 다양한 문서들을 선택해 검색 결과의 다양성과 관련성 사이의 균형을 맞출 수 있습니다.<br/>

```python
vector_store.max_marginal_relevance_search("검색할 내용", filter={"source": ""}, fetch_k=20, k=4)
```

#### 6) FAISS VectorStoreRetriever 
`as_retriever` 메서드로 VectorStore를 VectorStoreRetriever로 생성할 수 있습니다.<br/>
- similarity : 유사도 기반 검색 (기본값)<br/>

```python
retriever = vector_store.as_retriever(search_type="similarity", search_kwargs={"k": 4, "fetch_k" : 20})
```

- mmr : Maximal Marginal Relevance 검색<br/>

```python
retriever = vector_store.as_retriever(search_type="mmr", search_kwargs={"k": 4, "lambda_mult": 0.5, "fetch_k" : 20})
```

- similarity_score_threshold : 임계값 기반 유사도 검색<br/>

```python
retriever = vector_store.as_retriever(search_type="similarity_score_threshold", search_kwargs={"k": 4, "score_threshold": 0.5, "fetch_k" : 20})
```

<br/>

--- 
