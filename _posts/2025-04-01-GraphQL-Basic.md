---
layout: post
title: GraphQL - Query, Mutation, Subscription
author: 'Juho'
date: 2025-04-01 09:00:00 +0900
categories: [GraphQL]
tags: [GraphQL, FastAPI, strawberry, Python]
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
1. [FastAPI + Strawberry](#fastapi--strawberry)
2. [GraphQL 스키마 객체 타입](#graphql-스키마-객체-타입)
3. [Query](#query)
4. [Mutation](#mutation)
5. [Subscription](#subscription)
6. [실행](#실행)

## FastAPI + Strawberry
Strawberry를 설치하기 위해서 `pip install 'strawberry-graphql[fastapi]'`를 실행합니다.<br/>
간단한 예시를 통해서 Graph의 Query, Mutation, Subscription에 대해서 알아보도록 하겠습니다.<br/>

## GraphQL 스키마 객체 타입
```python
@strawberry.type
class Book:
    id: int
    title: str
    author: str
    publisher_id: int
```
Python 타입 힌트를 이용해서 class를 GraphQL 스키마 객체 타입으로 사용이 가능합니다.<br/>
`@strawberry.type` 데코레이터를 사용하면 class에 정의된 필드들이 자동으로 GraphQL 필드로 변환되어 API에서 사용할 수 있게 됩니다.<br/>
이를 통해서 GraphQL 스키마를 보다 직관적이고 간편하게 작성할 수 있습니다.

## Query
GraphQL에서의 Query는 클라이언트가 서버에 데이터를 요청하는 진입점 역할을 합니다. <br/>
클라이언트는 이 Query를 통해 원하는 데이터의 구조와 필드를 명시적으로 선택하여 요청할 수 있습니다. <br/>
```python
@strawberry.type
class Query:
    @strawberry.field
    def books(self) -> list[Book]:
        """모든 책 목록을 반환하는 쿼리"""
        return books

    @strawberry.field
    def book(self, id: int) -> Book | None:
        """ID로 특정 책을 찾는 쿼리"""
        for book in books:
            if book.id == id:
                return book
        return None
```

## Mutation
GraphQL에서 Mutation은 데이터를 수정, 추가 또는 삭제하는 작업을 수행하기 위한 진입점입니다.<br/>
Mutation은 서버의 상태를 변경하는 작업(생성, 수정, 삭제)을 수행합니다.<br/>
각 Mutation 필드에는 해당 작업을 실행할 수 있는 Resolver 함수가 연결되어 있어서 클라이언트가 요청한 작업을 처리한 후 결과 데이터를 반환할 수 있습니다.<br/>
Mutation은 순차적으로 실행되며 이를 통해 데이터 무결성과 일관성을 유지할 수 있습니다.<br/>
```python
@strawberry.type
class Mutation:
    @strawberry.mutation
    async def insert_book(self, book_input: BookInput) -> Book:
        new_id = len(books) + 1
        new_book = Book(id=new_id,
                        title=book_input.title,
                        author=book_input.author,
                        publisher_id=book_input.publisher_id)
        books.append(new_book)
        await book_event_queue.put(books.copy())
        return new_book
    
    @strawberry.mutation
    async def update_book(self, id: int, book_update: BookUpdate) -> Book | None:
        for i, book in enumerate(books):
            if book.id == id:
                if book_update.title is not None:
                    book.title = book_update.title
                if book_update.author is not None:
                    book.author = book_update.author
                if book_update.publisher_id is not None:
                    book.publisher_id = book_update.publisher_id
                await book_event_queue.put(books.copy())
                return book
        return None
    
    @strawberry.mutation
    async def delete_book(self, id: int) -> bool:
        for i, book in enumerate(books):
            if book.id == id:
                books.pop(i)
                await book_event_queue.put(books.copy())
                return True
        return False
```
Subscription을 위해서 Mutation에 `await book_event_queue.put(books.copy())`를 추가해놓았습니다.<br/>

## Subscription
GraphQL에서 Subscription은 클라이언트와 서버 간의 실시간 통신을 위한 진입점입니다. <br/>
클라이언트는 특정 이벤트나 데이터 변경 사항에 대해 구독할 수 있으며 해당 이벤트가 발생할 때마다 서버가 자동으로 업데이트 정보를 푸시합니다. <br/>
주로 WebSocket과 같은 프로토콜을 사용하여 지속적인 연결을 유지하며 실시간 데이터 피드가 필요할 때 유용하게 활용됩니다.<br/>
```python
@strawberry.type
class Subscription:
    @strawberry.subscription
    async def on_book_event(self) -> AsyncGenerator[List[Book], None]:
        while True:
            event = await book_event_queue.get()
            yield event
```

## 실행
```python

schema = strawberry.Schema(query=Query, mutation=Mutation, subscription=Subscription)

graphql_app = GraphQLRouter(schema)

app = FastAPI()
app.include_router(graphql_app, prefix="/graphql")
```

이렇게 한 다음 FastAPI 서버를 올려서 /graphql 경로에 가면 Strawberry GraphiQL을 확인할 수 있다.<br/>
```graphql
query GetBook{
     books {
        title
  }
}
```
이런 내용을 입력해서 GraphQL 구현 테스트를 간단하게 해볼 수 있다.<br/>

<br/>

--- 

<br/>
다음글에는 PostgreSQL ORM과 GrpahQL을 연동해보겠습니다.<br/>
