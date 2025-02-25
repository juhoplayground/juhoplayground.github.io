---
layout: post
title: Langchain PDF Chatbot 만들기 - 7 - PostgreSQL Chat Message History
author: 'Juho'
date: 2025-02-25 09:02:00 +0900
categories: [LangChain]
tags: [LangChain, PDF, Chatbot, PostgreSQL, Python]
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
1. [LangCahin <-> PostgreSQL](#langcahin---postgresql)
 - 1) [설치](#1-설치)
 - 2) [테스트 내용](#2-테스트-내용)

## LangCahin <-> PostgreSQL
이전 게시글에서 Redis를 이용해서 Chat History를 기록했는데 이번에는 PostgreSQL을 이용해서 진행하려고 한다.
[PostgreSQLChatMessageHistory](https://python.langchain.com/api_reference/community/chat_message_histories/langchain_community.chat_message_histories.postgres.PostgresChatMessageHistory.html){:target="_blank"} 관련 내용은 여기서 확인할 수 있다.<br/>
또는 [langchain-postgres](https://github.com/langchain-ai/langchain-postgres){:target="_blank"}에서 확인할 수 있다.<br/>



#### 1) 설치
PostgreSQLChatMessageHistory을 구현하기 위해서 아래의 라이브러리를 설치한다.<br/>
`pip install langchain_community langchain_core langchain-openai langchain-postgres`


#### 2) 테스트 내용
```python
from app.config.openai_config import OPENAI_API_KEY, OPENAI_API_EMBEDDING, OPENAI_API_MODEL
from app.config.redis_config import REDIS_HOST, REDIS_PORT, REDIS_DB, REDIS_PASSWORD
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_postgres import PostgresChatMessageHistory
from langchain_community.vectorstores import FAISS
from operator import itemgetter
from uuid import UUID


async def connect_database(session_id: str, user_id: str) -> PostgresChatMessageHistory:
    conn_info = f"postgresql://{POSTGRESQL_USER}:{POSTGRESQL_PASSWORD}@{POSTGRESQL_HOST}:{POSTGRESQL_PORT}/{POSTGRESQL_DB}"
    async_connection = await psycopg.AsyncConnection.connect(conn_info, autocommit=True)
    await PostgresChatMessageHistory.acreate_tables(async_connection, user_id) # 한번만 실행하면 된다.
    return PostgresChatMessageHistory(user_id, session_id, async_connection=async_connection)


async def chat_message(session_id:UUID, query:str):
    embedding = OpenAIEmbeddings(model=OPENAI_API_EMBEDDING, openai_api_key=OPENAI_API_KEY)
    cached_embedding = CacheBackedEmbeddings.from_bytes_store(underlying_embeddings=embedding,
                                                          document_embedding_cache=LocalFileStore(EMBEDDING_CACHE_FOLDER),
                                                          namespace=embedding.model)
    vector_store = FAISS.load_local("faiss_index", cached_embedding, allow_dangerous_deserialization=True)
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
            
    session_history = await connect_database(chat_id, user_info["id"])
    chain_with_history = RunnableWithMessageHistory(chain,
                                                    get_session_history=lambda session_id: session_history,
                                                    input_messages_key="question",
                                                    history_messages_key="chat_history")

    config = {"configurable": {"session_id": session_id}}

    async for chunk in chain_with_history.astream({"question": query}, config=config):
        yield chunk

```

생성된 테이블에서 결과를 확인해볼 수 있다.<br/>


---
