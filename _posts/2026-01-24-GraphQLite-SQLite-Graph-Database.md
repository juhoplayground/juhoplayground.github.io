---
layout: post
title: GraphQLite - SQLite에 그래프 데이터베이스 기능 추가하기
author: 'Juho'
date: 2026-01-24 00:00:00 +0900
categories: [Database]
tags: [SQLite, Database, Python, Rust]
pin: True
toc: True
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
1. [개요](#개요)
2. [주요 특징](#주요-특징)
3. [지원하는 Cypher 쿼리](#지원하는-cypher-쿼리)
4. [그래프 알고리즘](#그래프-알고리즘)
5. [설치 방법](#설치-방법)
6. [사용 예시](#사용-예시)
7. [기술 스택](#기술-스택)

## 개요

GraphQLite는 SQLite에 그래프 데이터베이스 기능을 추가하는 확장 프로그램이다.
단일 파일, 무설정 임베디드 데이터베이스의 단순성과 Cypher의 관계 모델링 성능을 결합하여 제공한다.

Neo4j와 같은 별도의 그래프 데이터베이스를 설치하지 않고도 SQLite에서 그래프 쿼리를 실행할 수 있다.
기존 SQLite 데이터베이스와 완전히 호환되며 서버 설정이 필요 없다.

## 주요 특징

### 무설정 구성

별도의 설정 없이 바로 사용할 수 있다.
SQLite의 장점인 단순함을 그대로 유지한다.

### SQLite 완전 호환

기존 SQLite 데이터베이스에 그래프 기능을 추가할 수 있다.
관계형 데이터와 그래프 데이터를 하나의 파일에서 함께 관리한다.

### 서버 불필요

임베디드 데이터베이스로 동작하므로 별도의 서버가 필요 없다.
애플리케이션에 직접 내장하여 사용할 수 있다.

### 다양한 언어 바인딩

Python, Rust, SQL 인터페이스를 제공한다.
익숙한 언어로 그래프 데이터베이스를 활용할 수 있다.

## 지원하는 Cypher 쿼리

GraphQLite는 Neo4j에서 사용하는 Cypher 쿼리 언어를 지원한다.

| 명령어 | 설명 |
|--------|------|
| MATCH | 패턴에 맞는 노드와 관계 검색 |
| CREATE | 새로운 노드와 관계 생성 |
| MERGE | 존재하면 매칭, 없으면 생성 |
| SET | 노드나 관계의 속성 설정 |
| DELETE | 노드나 관계 삭제 |
| WITH | 쿼리 중간 결과 전달 |
| UNWIND | 리스트를 개별 행으로 변환 |
| RETURN | 결과 반환 |

## 그래프 알고리즘

GraphQLite는 다양한 그래프 알고리즘을 내장하고 있다.

### PageRank

웹 페이지의 중요도를 측정하는 알고리즘이다.
소셜 네트워크에서 영향력 있는 사용자를 찾는 데 활용할 수 있다.

### Louvain

커뮤니티 탐지 알고리즘이다.
그래프에서 밀접하게 연결된 노드 그룹을 찾아낸다.

### Dijkstra

최단 경로를 찾는 알고리즘이다.
두 노드 사이의 최적 경로를 계산한다.

### BFS/DFS

너비 우선 탐색과 깊이 우선 탐색을 제공한다.
그래프 순회와 탐색에 사용된다.

### 연결 성분 탐지

그래프에서 서로 연결된 노드 집합을 찾는다.
네트워크 분석에 유용하다.

## 설치 방법

### macOS/Linux

```bash
brew install graphqlite
```

### Python

```bash
pip install graphqlite
```

### Rust

```bash
cargo add graphqlite
```

## 사용 예시

### Python에서 기본 사용

```python
import graphqlite

# 데이터베이스 연결
db = graphqlite.connect("my_graph.db")

# 노드 생성
db.execute("""
    CREATE (p:Person {name: 'Alice', age: 30})
""")

# 관계 생성
db.execute("""
    MATCH (a:Person {name: 'Alice'})
    CREATE (b:Person {name: 'Bob', age: 25})
    CREATE (a)-[:KNOWS]->(b)
""")

# 쿼리 실행
result = db.execute("""
    MATCH (p:Person)-[:KNOWS]->(friend)
    RETURN p.name, friend.name
""")

for row in result:
    print(row)
```

### 그래프 알고리즘 실행

```python
# PageRank 실행
pagerank_result = db.execute("""
    CALL algo.pageRank()
    YIELD nodeId, score
    RETURN nodeId, score
    ORDER BY score DESC
""")

# 최단 경로 찾기
shortest_path = db.execute("""
    MATCH (start:Person {name: 'Alice'}),
          (end:Person {name: 'Charlie'})
    CALL algo.shortestPath(start, end)
    YIELD path
    RETURN path
""")
```

### SQL과 함께 사용

GraphQLite는 기존 SQL 쿼리와 함께 사용할 수 있다.

```sql
-- 일반 SQL 테이블 생성
CREATE TABLE products (
    id INTEGER PRIMARY KEY,
    name TEXT,
    price REAL
);

-- 그래프 노드로 변환
SELECT graphqlite_create_node('Product', id, name, price)
FROM products;

-- Cypher 쿼리 실행
SELECT * FROM graphqlite_query("
    MATCH (p:Product)
    WHERE p.price > 100
    RETURN p.name, p.price
");
```

## 기술 스택

GraphQLite는 다음 언어로 구성되어 있다.

| 언어 | 비중 | 용도 |
|------|------|------|
| C | 80.8% | SQLite 확장 핵심 |
| Python | 7.4% | Python 바인딩 |
| Rust | 6.9% | Rust 바인딩 |
| Shell | 1.9% | 빌드 스크립트 |
| Yacc | 1.7% | Cypher 파서 |

C로 작성된 핵심 코드는 SQLite와 직접 통합되어 최적의 성능을 제공한다.
Python과 Rust 바인딩을 통해 각 언어 생태계에서 편리하게 사용할 수 있다.

## Reference

- [GraphQLite GitHub](https://github.com/colliery-io/graphqlite)
