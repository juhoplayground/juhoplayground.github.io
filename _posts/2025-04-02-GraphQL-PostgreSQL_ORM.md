---
layout: post
title: GraphQL - ORM 
author: 'Juho'
date: 2025-04-01 09:00:00 +0900
categories: [GraphQL]
tags: [GraphQL, FastAPI, strawberry, PostgreSQL, ORM, Python]
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
1. [GraphQL과 관계형 데이터베이스의 조합이 중요한 이유](#graphql과-관계형-데이터베이스의-조합이-중요한-이유)
2. [GraphQL 스키마 객체 타입](#graphql-스키마-객체-타입)
3. [Query](#query)
4. [Mutation](#mutation)
5. [Subscription](#subscription)
6. [실행](#실행)

## GraphQL과 관계형 데이터베이스의 조합이 중요한 이유
1) 데이터 요청 효율성 향상<br/>
- GraphQL을 사용하면 클라이언트가 필요한 데이터만 정확히 요청할 수 있어 관계형 데이터베이스에서 과도한 데이터 조회나 여러 번의 API 요청을 방지할 수 있습니다.<br/>
- 이는 "오버페칭"과 "언더페칭" 문제를 해결합니다.<br/>

2) 스키마 정의와 타입 시스템 <br/>
- GraphQL의 강력한 타입 시스템은 관계형 데이터베이스의 스키마와 자연스럽게 매핑됩니다.<br/>
- 이는 데이터 일관성을 유지하고 개발 시 타입 안전성을 제공합니다.<br/>

3) 성능 최적화 가능성 <br/>
- N+1 문제와 같은 일반적인 GraphQL 성능 이슈는 DataLoader 패턴, 관계형 데이터베이스의 조인 최적화 등을 통해 효과적으로 해결할 수 있습니다.<br/>

4) 트랜잭션 지원 <br/>
- 관계형 데이터베이스의 트랜잭션 기능은 GraphQL 뮤테이션에서 여러 데이터 변경 작업의 원자성을 보장하는 데 필수적입니다.<br/>


## GrpahQL + ORM 예시 <br/>
1) Database 연결
```python
from app.config.database_config import POSTGRESQL_USER, POSTGRESQL_PASSWORD, POSTGRESQL_HOST, POSTGRESQL_PORT, POSTGRESQL_DB
from sqlalchemy import create_engine, orm

DATABASE_URL = f"postgresql+psycopg2://{POSTGRESQL_USER}:{POSTGRESQL_PASSWORD}@{POSTGRESQL_HOST}:{POSTGRESQL_PORT}/{POSTGRESQL_DB}"
engine = create_engine(DATABASE_URL, echo=True)
SessionLocal = orm.sessionmaker(bind=engine)
```

2) ORM을 이용한 Model 구현
```python
from sqlalchemy import Column, Integer, String, ForeignKey, Text, Table
from sqlalchemy.orm import declarative_base, relationship
from app.database.database import engine
Base = declarative_base()

book_tags = Table("book_tags",
                  Base.metadata,
                  Column("book_id", Integer, ForeignKey("books.id")),
                  Column("tag_id", Integer, ForeignKey("tags.id")))    

class PublisherModel(Base):
    __tablename__ = "publishers"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String)
    location = Column(String)
    published_year = Column(Integer)
    books = relationship("BookModel", back_populates="publishers")
    
class BookModel(Base):
    __tablename__ = "books"
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String)
    author = Column(String)
    publisher_id = Column(Integer, ForeignKey("publishers.id"))
    publishers = relationship("PublisherModel", back_populates="books")
    reviews = relationship("ReviewModel", back_populates="books")
    tags = relationship("TagModel", secondary=book_tags, back_populates="books")
    
class ReviewModel(Base):
    __tablename__ = "reviews"
    id = Column(Integer, primary_key=True, index=True)
    content = Column(Text)
    rating = Column(Integer)
    book_id = Column(Integer, ForeignKey("books.id"))
    books = relationship("BookModel", back_populates="reviews")

class TagModel(Base):
    __tablename__ = "tags"
    id = Column(Integer, primary_key=True)
    name = Column(String, unique=True)
    books = relationship("BookModel", secondary=book_tags, back_populates="tags")


# 테이블이 없으면 생성
Base.metadata.create_all(bind=engine)
```

3) graphQL 스키마 정의
```python
@strawberry.type
class Review:
    id: int
    content: str
    rating: int
    book_id: int

@strawberry.type
class Tag:
    id: int
    name: str
    
@strawberry.type
class Book:
    id: int
    title: str
    author: str
    publisher_id: int
    
    @strawberry.field
    async def reviews(self) -> List[Review]:
        try:
            db = SessionLocal()
            db_reviews = db.query(ReviewModel).filter(ReviewModel.book_id == self.id).all()
            return [Review(id=r.id, content=r.content, rating=r.rating, book_id=r.book_id) for r in db_reviews]
        finally:
            db.close()
    
    @strawberry.field
    async def tags(self) -> List[Tag]:
        try:
            db = SessionLocal()
            book = db.query(BookModel).filter(BookModel.id == self.id).first()
            return [Tag(id=tag.id, name=tag.name) for tag in book.tags] if book else []
        finally:
            db.close()
        
@strawberry.type
class Publisher:
    id: int
    name: str
    location: str
    published_year: int

    @strawberry.field
    async def books(self) -> List[Book]:
        try:
            db = SessionLocal()
            db_books = db.query(BookModel).filter(BookModel.publisher_id == self.id).all()
            return [Book(id=b.id, title=b.title, author=b.author, publisher_id=b.publisher_id) for b in db_books]
        finally:
            db.close()
```

4) Query 작성 <br/>
```python
@strawberry.type
class Query:
    @strawberry.field
    async def books(self) -> List[Book]:
        db = SessionLocal()
        books = db.query(BookModel).all()
        db.close()
        return [Book(id=b.id, title=b.title, author=b.author, publisher_id=b.publisher_id) for b in books]

    @strawberry.field
    async def book(self, title: str) -> Optional[Book]:
        db = SessionLocal()
        find_book = db.query(BookModel).filter(BookModel.title == title).first()
        db.close()
        if find_book:
            return Book(id=find_book.id, title=find_book.title, author=find_book.author, publisher_id=find_book.publisher_id)
        return None
```

5) Mutation 작성 예시
```python
@strawberry.type
class Mutation:
    @strawberry.mutation
    async def create_book(self, book_input: BookInput) -> Book:
        db = SessionLocal()
        publisher_model = db.query(PublisherModel).filter(PublisherModel.id == book_input.publisher_id).first()
        if not publisher_model:
            db.close()
            raise ValueError("유효하지 않은 출판사 ID입니다.")
        
        new_book = BookModel(title=book_input.title,
                             author=book_input.author,
                             publisher_id=book_input.publisher_id)
        db.add(new_book)
        db.commit()
        db.refresh(new_book)
    
        new_book = Book(id=new_book.id,
                        title=new_book.title,
                        author=new_book.author,
                        publisher_id=new_book.publisher_id)
        
        await change_event_queue.put(ChangeEvent(entity_type=EntityType.BOOK,
                                                 event_type=EventType.CREATE,
                                                 book=new_book))
        db.close()
        return new_book

    @strawberry.mutation
    async def update_book(self, id: int, update: BookUpdate) -> Optional[Book]:
        db = SessionLocal()
        book_model = db.query(BookModel).filter(BookModel.id == id).first()
        if not book_model:
            db.close()
            return None
        
        if update.title is not None:
            book_model.title = update.title
        if update.author is not None:
            book_model.author = update.author
        if update.publisher_id is not None:
            publisher_model = db.query(PublisherModel).filter(PublisherModel.id == update.publisher_id).first()
            if not publisher_model:
                db.close()
                raise ValueError("유효하지 않은 출판사 ID입니다.")
            book_model.publisher_id = update.publisher_id
            
        db.commit()
        db.refresh(book_model)
        update_book = Book(id=book_model.id,
                           title=book_model.title,
                           author=book_model.author,
                           publisher_id=book_model.publisher_id)
        await change_event_queue.put(ChangeEvent(entity_type=EntityType.BOOK,
                                                 event_type=EventType.UPDATE,
                                                 book=update_book))
        db.close()
        return update_book
```


<br/>

--- 

<br/>
다음글에는 DataLoader 관련해서 작성하도록 하겠습니다.<br/>
