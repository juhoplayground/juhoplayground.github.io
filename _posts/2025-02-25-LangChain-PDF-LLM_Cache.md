---
layout: post
title: Langchain PDF Chatbot 만들기 - 8- LLM Cache
author: 'Juho'
date: 2025-02-25 09:03:00 +0900
categories: [LangChain]
tags: [LangChain, PDF, Chatbot, Redis, Python]
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
1. [LLM Cache](#llm-cache)
2. [InMemoryCache](#inmemorycache)
3. [RedisCache](#rediscache)
4. [RedisSemanticCache](#redissemanticcache)
5. [Cache 확인 방법](#cache-확인-방법)

## LLM Cache
LangChain에서는 LLM 응답에 대한 캐시를 제공하고 있으며 다음의 이유 2가지로 유용합니다.<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- 1) 동일한 결과를 여러 번 요청하는 경우 API 호출 수를 줄여 비용을 절감할 수 있습니다.<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- 2) API 호출 수를 줄임으로써 애플리케이션의 속도를 향상시킬 수 있습니다.<br/>

`pip install -U langchain_openai langchain_community`로 필요한 라이브러리를 설치합니다.<br/>


## InMemoryCache
```python

from langchain_community.cache import InMemoryCache

set_llm_cache(InMemoryCache())
```
위의 코드로 간단하게 InMemoryCache설정이 가능합니다.<br/>


## RedisCache
```python

from langchain_community.cache import RedisCache

redis_client = redis.StrictRedis(host=REDIS_HOST, port=REDIS_PORT, db=REDIS_DB, password=REDIS_PASSWORD, decode_responses=True)
set_llm_cache(RedisCache(redis_client))
```

위의 코드로 RedisCache 설정이 가능합니다.<br/>

## RedisSemanticCache
RedisSemanticCache를 사용하기 위해서는 RediSearch를 설치해야합니다.<br/>
[RediSearch 설치방법](https://juhoplayground.github.io/posts/RedisSearch/){:target="_blank"}에 작성해두었으니 참고 부탁드립니다.<br/>
```python

from langchain_community.cache import RedisSemanticCache

embedding = OpenAIEmbeddings(model=OPENAI_API_EMBEDDING, openai_api_key=OPENAI_API_KEY)
cached_embedding = CacheBackedEmbeddings.from_bytes_store(underlying_embeddings=embedding,
                                                          document_embedding_cache=LocalFileStore(EMBEDDING_CACHE_FOLDER),
                                                          namespace=embedding.model)
semantic_cache = RedisSemanticCache(redis_url=f"redis://:{REDIS_PASSWORD}@{REDIS_HOST}:{REDIS_PORT}/{REDIS_DB}", embedding=cached_embedding, score_threshold=0.2)
set_llm_cache(semantic_cache)

```
위의 코드로 RedisSemanticCache 설정이 가능합니다.<br/>

## Cache 확인 방법
위에서 설정한 캐시를 확인할 수 있는 방법이 있습니다.<br/>

```python
from langchain.globals import get_llm_cache
llm_cache = get_llm_cache()
print(llm_cache)
```

캐시를 정상적으로 설정되었다면 `<langchain_community.cache.RedisSemanticCache object at 0x7c8a5e8fac00>` 이런식으로 print가 찍히고 <br/>
캐시가 설정되지 않았다면 None으로 print되게 됩니다.<br/>


## FastAPI + Chatbot에 사용한 방법
```python

from langchain_community.cache import InMemoryCache, RedisSemanticCache
from langchain_openai import OpenAIEmbeddings
from langchain.globals import set_llm_cache

from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    try:
        embedding = OpenAIEmbeddings(model=OPENAI_API_EMBEDDING, openai_api_key=OPENAI_API_KEY)
        semantic_cache = RedisSemanticCache(redis_url=f"redis://:{REDIS_PASSWORD}@{REDIS_HOST}:{REDIS_PORT}/{REDIS_DB}", embedding=embedding, score_threshold=0)
        set_llm_cache(semantic_cache)
    except Exception as e:
        set_llm_cache(InMemoryCache())
    yield
    
app = FastAPI(
    title="title",
    description="description",
    version="0.01",
    docs_url="/doc",
    redoc_url=None,
    lifespan=lifespan
)

```


<br/>

--- 
