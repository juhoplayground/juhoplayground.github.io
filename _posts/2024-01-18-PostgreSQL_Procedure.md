---
layout: post
title: PostgreSQL PL/pgSQL - Procedure
author: 'Juho'
date: 2024-01-18 16:12:00 +0900
categories: [PostgreSQL, Database, Procedure]
tags: [PostgreSQL, Database, Procedure]
pin: True
toc : True
---

## 목차
1. [PL/pgSQL?](#plpgsql)
2. [PL/pgSQL 특징](#plpgsql-특징)
3. [PL/pgSQL - 구조](#plpgsql-구조)
4. [PL/pgSQL - Procedure](#plpgsql---procedure)
 - 1) [기본적인 Procedure](#1-기본적인-procedure-생성)
 - 2) [Procedure 실행](#2-procedure-실행)
 - 3) [매개변수를 사용하는 Procedure](#3-매개변수를-사용하는-procedure)
 - 4) [Procedure 예외처리](#4-procedure-예외처리)


## PL/pgSQL?
PostgreSQL에서 Procedure는 PL/pgSQL(Procedural Language/PostgreSQL)이라는<br/> 내장된 프로시저 언어를 사용하여 작성됨
```sql
SELECT * FROM pg_catalog.pg_extensions;
```
<!-- ![Alt Text](/images/plpgsql.png) -->
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/34d6bdf8-5e0a-4b5d-a536-15c67b06eea7)

## PL/pgSQL 특징
1. 조건문과 반복문 <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;IF-THEN-ELSE, CASE문 등과 같은 조건문과 <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;FOR, WHILE 루프 등과 같은 반복문을 사용하여 제어할 수 있음
2. 예외 처리 <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;예외 처리 기능을 통해 Procedure나 Function 내에서 발생할 수 있는 예외를 처리할 수 있어<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;안정성읖 높이고 오류에 대한 적절한 대응을 할 수 있게 함
3. 트랜잭션 관리<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;여러 SQL문을 논리적인 블록으로 묶고 롤백 또는 커밋을 수행할 수 있음
4. 재사용<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;로직을 모듈화하여 재사용성을 높일 수 있음



## PL/pgSQL 구조
```sql
DECLARE
-- 선언 부분 : 변수, 상수 및 데이터 타입 정의
BEGIN
-- 실행 부분 : 실행 코드
EXCEPTION
-- 예외처리 부분 : 예외처리 코드 작성
END;
```
<br/>

## PL/pgSQL - Procedure
#### 1) 기본적인 Procedure 생성
```sql
CREATE OR REPLACE PROCEDURE sample_procedure()
AS $$
BEGIN
  UPDATE sample_table
  SET sample_columns = 'sample_values'
  WHERE sample_columns IS NULL;
END;
$$ LANGUAGE plpgsql;
```

#### 2) Procedure 실행
```sql
Call sample_procedure()
```

#### 3) 매개변수를 사용하는 Procedure
```sql
CREATE OR REPLACE PROCEDURE sample_procedure(smaple_parameters VARCHAR)
AS $$
BEGIN
  UPDATE sample_table
  SET sample_columns = 'sample_values'
  WHERE sample_columns = smaple_parameters;
END;
$$ LANGUAGE plpgsql;
```


#### 4) Procedure 예외처리
```sql
CREATE OR REPLACE PROCEDURE sample_procedure(smaple_parameters VARCHAR)
AS $$
DECLARE
    v_sqlstate          VARCHAR;
    v_message_text      VARCHAR;
    v_message_detail    VARCHAR;
    v_message_hint      VARCHAR;
    v_exception_context VARCHAR;
BEGIN
  UPDATE sample_table
  SET sample_columns = 'sample_values'
  WHERE sample_columns = smaple_parameters;
EXCEPTION
        WHEN OTHERS THEN
            GET STACKED DIAGNOSTICS v_message_text = MESSAGE_TEXT, v_message_detail = PG_EXCEPTION_DETAIL, v_message_hint = PG_EXCEPTION_HINT, v_sqlstate = RETURNED_SQLSTATE, v_exception_context = PG_EXCEPTION_CONTEXT;
            RAISE NOTICE '예외 발생: SQLSTATE=%, MESSAGE_TEXT=%, PG_EXCEPTION_DETAIL=%, PG_EXCEPTION_HINT=%, PG_EXCEPTION_CONTEXT=%', v_sqlstate, v_message_text, v_message_detail, v_message_hint, v_exception_context;
END;
$$ LANGUAGE plpgsql;
```