---
layout: post
title: GraphQL - Complexity & Depth 제한
author: 'Juho'
date: 2025-04-14 09:00:00 +0900
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
1. [Complexity, Depth 제한을 하는 이유](#complexity-depth-제한을-하는-이유)
 - [서버 자원 보호 및 성능 유지](#서버-자원-보호-및-성능-유지)
 - [DoS 및 악의적 공격 방어](#dos-및-악의적-공격-방어)
 - [예측 가능한 API 동작과 유지 관리](#예측-가능한-api-동작과-유지-관리)
2. [Depth Limit 설정하는 방법](#depth-limit-설정하는-방법)
3. [Complexity Limit 설정하는 방법](#complexity-limit-설정하는-방법)


## Complexity, Depth 제한을 하는 이유
GraphQL은 클라이언트가 필요한 데이터만 선택해서 요청할 수 있도록 매우 유연한 쿼리 시스템을 제공하지만 이 유연성은 동시에 서버에 과도한 부하를 줄 수 있는 위험도 내포하고 있습니다.  
Complexity(복잡도)와 Depth(깊이) 제한은 이러한 위험을 완화하기 위해 도입되는 중요한 제어 수단입니다.

### 서버 자원 보호 및 성능 유지
- 자원 소모 관리  
GraphQL 쿼리는 매우 복잡하게 중첩되거나반복적인 필드 요청으로 인해 계산 비용이 기하급수적으로 증가할 수 있습니다.  
이러한 비용을 미리 측정하지 않고 실행하면 CPU, 메모리 등의 서버 자원을 과도하게 소모할 위험이 있습니다.


- 응답 속도 및 안정성 확보  
과도한 쿼리 요청은 서버 응답 시간을 지연시키고 다른 정상적인 요청 처리에도 영향을 줄 수 있습니다.  
복잡도와 깊이 제한을 통해 서버가 처리할 수 있는 쿼리의 "최대 부담"을 사전에 정의하고 이를 초과하는 쿼리는 차단함으로써 서비스의 안정성을 높일 수 있습니다.  

### DoS 및 악의적 공격 방어
- 악의적인 쿼리 방어  
GraphQL API는 클라이언트가 원하는 데이터를 자유롭게 선택할 수 있는 만큼 특정 사용자가 의도적으로 너무 깊은 쿼리나 과도한 복잡도를 가진 쿼리를 보내 서버를 마비시키려는 DoS 공격에 취약할 수 있습니다.  


- 비정상적인 사용 패턴 차단  
복잡도 제한은 각 필드에 비용을 할당한 후 전체 쿼리의 비용을 계산하여 허용 범위를 벗어나는 경우 쿼리를 거부합니다.  
이를 통해 우발적이거나 고의적인 리소스 낭비 상황을 미연에 방지할 수 있습니다.

### 예측 가능한 API 동작과 유지 관리
- 쿼리 예측 가능성  
사전에 정의한 제한 덕분에 개발자와 운영자는 API가 어떤 수준의 복잡도까지 처리 가능한지 명확히 파악할 수 있습니다.  
이는 시스템의 확장성과 모니터링, 문제 해결에 매우 중요한 요소입니다.  


- 유지 관리 및 비용 분석  
각 필드나 쿼리에 부여된 비용에 기반하여 사용 패턴을 분석하면, 어떤 요청이 빈번하게 리소스를 많이 사용하는지 파악할 수 있으며, 향후 스키마나 서버 설정을 조정하는 데 유용한 데이터를 제공할 수 있습니다.  


## Depth Limit 설정하는 방법
`pip install graphql-complexity[strawberry-graphql]` 설치했다.  
```python
from graphql_complexity.extensions.strawberry_graphql import build_complexity_extension
from graphql_complexity import SimpleEstimator

complexity_estimator = SimpleEstimator(complexity=2)
ComplexityLimitExtension = build_complexity_extension(estimator=complexity_estimator, max_complexity=100)
```
복잡도 체크를 위한 Estimator를 설정하는데 각 필드의 복잡도 값을 2로 설정하고 최대 복잡도를 100으로 설정한 것이다.  

### Complexity Limit 설정하는 방법
```python
from strawberry.extensions import QueryDepthLimiter

QueryDepthLimiterExtension = QueryDepthLimiter(max_depth=10)
```

이후 ComplexityLimitExtension, QueryDepthLimiterExtension를 적용하는 방법은  
```python
schema = strawberry.Schema(query=Query,
                        mutation=Mutation,
                        subscription=Subscription,
                        extensions=[ComplexityLimitExtension, QueryDepthLimiterExtension])
```
Schema에 extensions 파라미터의 값으로 넣어주면 적용이 된다.

<br/>

--- 

<br/>