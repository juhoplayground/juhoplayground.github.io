---
layout: post
title: GraphQL - DataLoader
author: 'Juho'
date: 2025-04-07 09:00:00 +0900
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
1. [N+1 문제](#n1-문제)
 - [발생 원인](#발생-원인)
 - [성능 문제로 이어지는 이유](#성능-문제로-이어지는-이유)
2. [DataLoader](#dataloader)
3. [DataLoader 구현 코드](#dataloader-구현-코드)


## N+1 문제
GraphQL을 사용하다 보면 개발자들이 자주 마주치는 문제가 바로 N+1 문제입니다.<br/>
이 문제는 클라이언트가 하나의 요청으로 여러 데이터를 가져오려 할 때 의도치 않게 너무 많은 데이터베이스 쿼리가 발생해 성능 저하를 일으키는 현상을 말합니다. <br/>
예를 들어 게시글과 해당 게시글에 달린 댓글을 가져오는 GraphQL 쿼리를 생각해봅시다. <br/>
기본적으로 게시글 목록을 가져오기 위한 하나의 쿼리가 실행되고 그 후 각 게시글에 대해 댓글 데이터를 가져오는 쿼리가 개별적으로 실행됩니다. <br/>
만약 게시글이 N개라면 댓글 데이터를 가져오기 위해 N개의 별도 쿼리가 추가로 실행되므로 총 N +1 번의 쿼리가 발생하게 됩니다.

### 발생 원인
이와 같은 상황은 특히 다음과 같은 경우에 자주 발생합니다.<br/>

1) 연관된 데이터 조회<br>
- 한 객체에서 다른 객체를 조회하는 경우<br/>

2) 중첩 쿼리<br>
- 중첩된 필드마다 별도의 resolver가 호출되어 각기 데이터베이스 쿼리가 발생하는 경우<br/>

3) 비효율적인 resolver 설계<br/>
- 각 resolver에서 데이터베이스와 직접 통신하게 구현된 경우<br/>


### 성능 문제로 이어지는 이유
N+1 문제는 단순히 코드가 복잡해지는 문제뿐만 아니라 실제 운영 환경에서는 다음과 같은 성능 저하를 초래할 수 있습니다.<br/>

1) 데이터베이스 부하 증가<br/>
- 불필요하게 많은 쿼리가 데이터베이스에 전송되면 데이터베이스의 응답 시간이 늦어지고 서버에 과부하가 발생할 수 있습니다.<br/>

2) 네트워크 지연<br/>
- 각 쿼리마다 네트워크 왕복 시간이 발생하여 전체 응답 시간이 증가합니다. <br/>

3) 스케일링의 어려움<br/>
- 사용자가 많아질수록 N+1 문제로 인한 쿼리 수는 기하급수적으로 늘어나 서버 스케일링에 큰 영향을 미칩니다.<br/>

## DataLoader
이러한 N+1 문제를 해결하기 위한 대표적인 도구가 바로 DataLoader입니다. <br/>
DataLoader는 Meta에서 만든 GraphQL 서버에서 발생하는 N+1 문제를 효과적으로 해결할 수 있도록 도와줍니다. <br/>

DataLoader의 주요 기능<br/>

1) 배치 처리 <br/>
- 여러 개의 데이터 요청을 모아서 한 번에 처리할 수 있게 합니다.<br/>

2) 캐싱 <br/>
- 같은 데이터에 대한 중복 요청이 있을 경우 한 번의 데이터 조회 결과를 캐싱하여 재사용함으로써 불필요한 데이터베이스 접근을 줄입니다.<br/>

3) 비동기 처리<br/>
- 비동기적으로 데이터를 처리하므로 데이터 요청과 응답의 흐름을 개발자가 관리할 수 있습니다.<br/>

## DataLoader 구현 코드
`pip install aiodataloader`으로 필요 라이브러리 설치합니다.<br/>

`dataloaders.py`파일을 만듭니다.<br/>
```python
from app.database import SessionLocal, BookModel, ReviewModel
from aiodataloader import DataLoader
from typing import List, Dict


class ReviewsByBookLoader(DataLoader):
    async def batch_load_fn(self, book_ids: List[int]) -> List[List[dict]]:
        try:
            db = SessionLocal()
            
            all_reviews = db.query(ReviewModel).filter(ReviewModel.book_id.in_(book_ids)).all()
            
            reviews_by_book: Dict[int, List[dict]] = {book_id: [] for book_id in book_ids}
            for review in all_reviews:
                reviews_by_book[review.book_id].append({"id": review.id,
                                                        "content": review.content,
                                                        "rating": review.rating,
                                                        "book_id": review.book_id})
                
            return [reviews_by_book[book_id] for book_id in book_ids]
        finally:
            db.close()

class TagsByBookLoader(DataLoader):
    async def batch_load_fn(self, book_ids: List[int]) -> List[List[dict]]:
        try:
            db = SessionLocal()
            
            books_with_tags = db.query(BookModel).filter(BookModel.id.in_(book_ids)).all()
            
            tags_by_book: Dict[int, List[dict]] = {book_id: [] for book_id in book_ids}
            for book in books_with_tags:
                tags_by_book[book.id] = [{"id": tag.id, "name": tag.name} for tag in book.tags]
                
            return [tags_by_book[book_id] for book_id in book_ids]
        finally:
            db.close()

class BooksByPublisherLoader(DataLoader):
    async def batch_load_fn(self, publisher_ids: List[int]) -> List[List[dict]]:
        try:
            db = SessionLocal()
            
            all_books = db.query(BookModel).filter(BookModel.publisher_id.in_(publisher_ids)).all()
            
            books_by_publisher: Dict[int, List[dict]] = {publisher_id: [] for publisher_id in publisher_ids}
            for book in all_books:
                books_by_publisher[book.publisher_id].append({"id": book.id,
                                                              "title": book.title,
                                                              "author": book.author,
                                                              "publisher_id": book.publisher_id})
                
            return [books_by_publisher[publisher_id] for publisher_id in publisher_ids]
        finally:
            db.close() 
```


`context.py` 파일을 만듭니다.<br/>
```python
from typing import Any, Optional
from strawberry.types import Info
from strawberry.fastapi import BaseContext
from .dataloaders import ReviewsByBookLoader, TagsByBookLoader, BooksByPublisherLoader

class GraphQLContext(BaseContext):
    def __init__(self, request=None):
        super().__init__()
        self.request = request
        self.reviews_by_book_loader = ReviewsByBookLoader()
        self.tags_by_book_loader = TagsByBookLoader()
        self.books_by_publisher_loader = BooksByPublisherLoader()

async def get_context(request=None) -> GraphQLContext:
    return GraphQLContext(request=request)

def get_loader_context(info: Info) -> GraphQLContext:
    return info.context 
```

기존 스키마 정의를 변경해줍니다.<br/>
```python
@strawberry.type
class Book:
    id: int
    title: str
    author: str
    publisher_id: int
    
    @strawberry.field
    async def reviews(self, info: Info) -> List[Review]:
        context = get_loader_context(info)
        reviews = await context.reviews_by_book_loader.load(self.id)
        return [Review(**review) for review in reviews]
    
    @strawberry.field
    async def tags(self, info: Info) -> List[Tag]:
        context = get_loader_context(info)
        tags = await context.tags_by_book_loader.load(self.id)
        return [Tag(**tag) for tag in tags]
        
@strawberry.type
class Publisher:
    id: int
    name: str
    location: str
    published_year: int

    @strawberry.field
    async def books(self, info: Info) -> List[Book]:
        context = get_loader_context(info)
        books = await context.books_by_publisher_loader.load(self.id)
        return [Book(**book) for book in books]
```

Query의 내용을 변경해줍니다.<br/>
```python
@strawberry.field
    async def reviews(self, info: Info, book_id: Optional[int] = None) -> List[Review]:
        if book_id is not None:
            context = info.context
            reviews = await context.reviews_by_book_loader.load(book_id)
            return [Review(**review) for review in reviews]
        else:
            db = SessionLocal()
            reviews = db.query(ReviewModel).all()
            db.close()
            return [Review(id=r.id, content=r.content, rating=r.rating, book_id=r.book_id) for r in reviews]
    
```

<br/>

--- 

<br/>
GraphQL에서 N+1 문제는 중첩된 데이터를 가져올 때 필연적으로 발생할 수 있는 성능 저하 요인입니다. <br/>
이를 해결하기 위해 DataLoader와 같은 도구를 활용하면 배치 처리와 캐싱을 통해 데이터베이스에 보내는 쿼리의 수를 줄이고 응답 속도를 개선할 수 있습니다. <br/>
GraphQL 서버를 구축할 때 N+1 문제를 미리 인지하고 DataLoader를 적절히 활용함으로써 더욱 효율적이고 확장 가능한 API를 설계해야할 것 같습니다.<br/>