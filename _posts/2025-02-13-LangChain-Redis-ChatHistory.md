---
layout: post
title: Langchain Redis Chat History 기록하기
author: 'Juho'
date: 2025-02-13 09:00:00 +0900
categories: [LangChain]
tags: [LangChain, Redis, Python]
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
1. [LangCahin <-> Redis](#langcahin---redis)
 - 1) [설치](#1-설치)
 - 2) [테스트 내용](#2-테스트-내용)

## LangCahin <-> Redis
이전 메시지의 맥락을 유지해야 할 필요가 있어서 LangCahin과 Redis를 이용해서 Chat Message History를 기록하는 것이 있어서 테스트해봤다.<br/>
[RedisChatMessageHistory](https://python.langchain.com/api_reference/community/chat_message_histories/langchain_community.chat_message_histories.redis.RedisChatMessageHistory.html#langchain_community.chat_message_histories.redis.RedisChatMessageHistory){:target="_blank"} 관련 내용은 여기서 확인할 수 있다.<br/>


#### 1) 설치
이미 서버에는 redis가 설치되어 있고 다른 게시글에서 redis를 설치하는 내용은 했으니까 넘어가고 LangCahin 관련한 것만 설명하겠다.<br/>
`pip install langchain_community langchain_core langchain-openai redis`


#### 2) 테스트 내용
```python
from app.config.openai_config import OPENAI_API_KEY, OPENAI_API_EMBEDDING, OPENAI_API_MODEL
from app.config.redis_config import REDIS_HOST, REDIS_PORT, REDIS_DB, REDIS_PASSWORD
from langchain_community.chat_message_histories import RedisChatMessageHistory
from langchain_community.vectorstores import FAISS
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from operator import itemgetter
from uuid import UUID


async def get_chat_message_history(session_id: str):
    return RedisChatMessageHistory(session_id, url=f"redis://:{REDIS_PASSWORD}@{REDIS_HOST}:{REDIS_PORT}/{REDIS_DB}")


async def chat_message(session_id:UUID, query:str):
    embedding = OpenAIEmbeddings(model=OPENAI_API_EMBEDDING, openai_api_key=OPENAI_API_KEY)
    vector_store = FAISS.load_local("faiss_index", embedding, allow_dangerous_deserialization=True)
    retriever = vector_store.as_retriever()

    prompt = ""

    llm = ChatOpenAI(model=OPENAI_API_MODEL, temperature=0, openai_api_key=OPENAI_API_KEY, streaming=True)
    chat_input = {"context": itemgetter("question") | retriever,
                  "question": itemgetter("question"),
                  "chat_history": itemgetter("chat_history")}
            
    chain = (chat_input
            | prompt
            | llm
            | StrOutputParser())
            
        
    chain_with_history = RunnableWithMessageHistory(chain,
                                                    get_session_history=lambda session_id: get_chat_message_history,
                                                    input_messages_key="question",
                                                    history_messages_key="chat_history")

    config = {"configurable": {"session_id": session_id}}

    async for chunk in chain_with_history.astream({"question": query}, config=config):
        yield chunk

```

redis에서 확인해보면
`message_store:f795cc5d-2fd4-4598-bb71-70306e46b64f` 이러한 key를 가지는 list를 확인할 수 있다.<br/>
`lrange "message_store:f795cc5d-2fd4-4598-bb71-70306e46b64f" 0 -1`로 검색해서 전체 내용을 확인해볼 수 있다. <br/>


--
