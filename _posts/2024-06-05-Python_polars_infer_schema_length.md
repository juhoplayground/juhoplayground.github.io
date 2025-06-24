---
layout: post
title: Python - Polars infer_schema_length 오류
author: 'Juho'
date: 2024-06-04 09:00:00 +0900
categories: [Polars]
tags: [Polars, Python]
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
1. [Polars 오류 발생](#polars-오류-발생)
2. [Polars 오류 원인](#polars-오류-원인)
3. [Polars 오류 해결방법](#polars-오류-해결방법)

## Polars 오류 발생
PostgreSQL에서 데이터를 받아와 pl.DataFrame()을 이용해서 DataFrame을 만들고 특정 작업을 할때 오류가 발생했다.<br/>
오류 내용은 아래와 같았다.
```
could not append value: "col" of type: str to the builder; make sure that all rows have the same schema or consider increasing `infer_schema_length`
```


## Polars 오류 원인
infer_schema_length는 스키마 추론을 위해 스컌할 최대 행수를 의미하며 기본 값은 100이다.<br/>
문제가 되는 col에 Null값과 float가 같이 존재하였는데 기본 값인 100 행 이내에는 전부 Null 값이 들어가게 되는 경우가 존재하였고 <br/>
이 떄 특정 작업을 진행을 하게 되면 type 문제가 발생하는 것이였다.<br/>

## Polars 오류 해결방법
모든 컬럼이 100행 이내에 Null이 존재하지 않는다는 보장을 할 수 없었다<br/>
DB에서 Null을 다른 값으로 치환할 수 있는 상황도 아니였다.<br/>
<br/>
간편하게 pl.DataFrame(, infer_schema_length=None)으로 설정하면 전체 데이터를 스캔하도록 할 수 있다.<br/>
그렇게 되면 해당 col에 대한 정확한 type으로 스키마 추론을 해서 특정 작업에서 오류가 발생하지 않았다.<br/>
다만 그 과정에서 소폭의 속도 저하는 발생하였다.<br/>