---
layout: post
title: GraphQL - Query(Arguments, Aliases, Variables, Fragments, Directives)
author: 'Juho'
date: 2025-04-03 09:00:00 +0900
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
1. [Query](#query)
2. [Arguments](#arguments)
3. [Aliases](#aliases)
4. [Variables](#variables)
5. [Fragments](#fragments)
6. [Directives](#directives)

## Query
이전 게시글에서 만든 내용을 Query로 조회하려면 아래와 같습니다.
```graphql
query {
  publishers {
    id
    name
    location
    publishedYear
    books {
      id
      title
      author
      publisherId
      reviews {
        id
        content
        rating
        bookId
      }
      tags {
        id
        name
      }
    }
  }
}
```
이 기본적인 쿼리에 선택 사항을 하나씩 추가해보도록 하겠습니다.<br/>

## Arguments
Arguments를 추가하하여 Query를 수정하면 아래와 같습니다.
```graphql
query {
  publisher(name:"테스트1"){
    id
    name
    location
    publishedYear
    books {
      id
      title
      author
      publisherId
      reviews {
        id
        content
        rating
        bookId
      }
      tags {
        id
        name
      }
    }
  }
}
```
위와 같이 수정을 하면 모든 publishers가 아닌 특정 publishers(name이 "테스트1")만 호출하게 되어 데이터 필터링이 이루어집니다.<br/>
Arguments를 사용하면 필요한 데이터만 요청하므로 네트워크 트랙픽이 감소하고 서버의 부하도 줄일 수 있습니다.<br/>
또한 클라이언트도 불필요한 데이터를 처리할 필요가 없어서 클라이언트의 메모리 사용량이 감소하고 데이터 처리 시간도 단축될 수 있습니다.<br/>


## Aliases
Aliases는 동일한 필드를 다른 인자로 여러번 쿼리할 떄 유용합니다.<br/>
```graphql
query {
  test:publisher(name:"테스트1"){
    id
    name
    location
    city:location
    publishedYear
    books {
      id
      title
      author
      publisherId
      reviews {
        id
        content
        rating
        bookId
      }
      tags {
        id
        name
      }
    }
  }
  
}
```
Aliases를 사용하면 서버의 필드 이름과 상관없이 클라이언트가 원하는 이름으로 데이터를 받을 수 있어 응답 데이터 구조를 커스터마이징이 가능합니다.<br/>
그렇기에 프론트엔드 코드의 구조를 조금 더 유연하게 설계할 수 있습니다.<br/>
또한 동일한 타입의 여러 객체를 한번의 쿼리로 가져올 수 있어 다중 쿼리 실행이 용이해집니다.<br/>
```graphql
{
     test_1:publisher(name: "테스트1") { ... }
     test_2:publisher(name: "테스트2") { ... }
   }
```


## Variables
Variables가 전달되지 않았을 때 기본값으로 "테스트999"를 사용할 수 있게 설정합니다.<br/>
```graphql
query GetPublisher($publisher_name: String = "테스트999") {
  publisher(name: $publisher_name) {
    id
    name
    location
    city: location
    publishedYear
    books {
      id
      title
      author
      publisherId
      reviews {
        id
        content
        rating
        bookId
      }
      tags {
        id
        name
      }
    }
  }
}
```

Variables는 따로 설정을 아래와 같이 합니다.<br/>
```
{
  "publisher_name": "테스트1"
}
```
Variables를 사용하면 변수의 타입을 명시적으로 선언할 수 있습니다.<br/>
잘못된 타입의 데이터가 전달되면 즉시 에러를 발생시켜 런타임 에러를 사전에 방지할 수 있습니다.<br/>
클라이언트에서 변수만 바꿔서 다양한 데이터를 요청할 수 있어 동적 쿼리 작성이 용이합니다.<br/>
또한 SQL 인젝션과 같은 보안 취약점을 방지할 수 있습니다. <br/>


## Fragments
```
fragment PublisherDetails on Publisher {
  id
  name
  location
  city: location
  publishedYear
}

query GetPublisher($publisher_name: String = "테스트999",
                  $publisher_name2: String = "테스트998") {
  test1:publisher(name: $publisher_name) {
    ...PublisherDetails
  }
  test2:publisher(name: $publisher_name2) {
    ...PublisherDetails
  }
}
```

```
{
  "publisher_name": "테스트1",
  "publisher_name2": "테스트2"
}
```
Fragment를 사용하면 코드 재사용성이 높아지고, 필드 구조가 변경될 때 Fragment만 수정하면 됩니다.<br/>


## Directives
```
fragment PublisherDetails on Publisher {
  id
  name
  location
  city: location
  publishedYear
}

query GetPublisher(
  $publisher_name: String = "테스트999",
  $publisher_name2: String = "테스트998",
  $is_include: Boolean = true,
  $is_skip: Boolean = false
) {
  test1: publisher(name: $publisher_name) @include(if: $is_include) {
    ...PublisherDetails
  }
  test2: publisher(name: $publisher_name2) @skip(if: $is_skip) {
    ...PublisherDetails
  }
}
```

```
{
  "publisher_name": "테스트1",
  "publisher_name2": "테스트2",
  "is_include": true,
  "is_skip": false
}
```
`is_include`와 `is_skip`을 true, false로 변경하면서 실행 결과를 확인해보는게 좋습니다.<br/>

1) @include 지시어<br/>
if 조건이 true일 때만 필드를 포함합니다<br/>
includeFirst: true → test1 데이터 포함<br/>
includeFirst: false → test1 데이터 제외<br/>
<br/>

2) @skip 지시어<br/>
if 조건이 true일 때 필드를 제외합니다<br/>
skipSecond: true → test2 데이터 제외<br/>
skipSecond: false → test2 데이터 포함<br/>


이러한 지시어를 사용하면 클라이언트에서 필요한 데이터만 선택적으로 가져올 수 있어 효율적인 데이터 요청이 가능합니다.<br/>

<br/>

--- 
