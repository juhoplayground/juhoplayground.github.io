---
layout: post
title: PostgreSQL PL/pgSQL - Function
author: 'Juho'
date: 2024-01-30 09:00:00 +0900
categories: [PostgreSQL, Database, Function]
tags: [PostgreSQL, Database, Function]
pin: True
toc : True
---

## 목차
1. [PL/pgSQL - Function](#plpgsql---function)
 - 1) [기본적인 Function 생성 (Return 없음)](#1-기본적인-function-생성-return-없음)
 - 2) [Function 실행 방법](#2-function-실행-방법)
 - 3) [매개변수를 사용하는 Function](#3-매개변수를-사용하는-function)
 - 4) [Function 예외처리](#4-function-예외처리)
 - 5) [Function Return 타입](#5-function-return-타입)

## PL/pgSQL - Function
PostgreSQL에서 Function은 Procedure과 다르게 처리 결과를 Return 할 수 있다.<br/>
대신 Return하는 결과물의 형태를 꼭 지정해줘야한다.<br/>

#### 1) 기본적인 Function 생성 (Return 없음)
Return을 하지 않으려면 `Returns VOID`를 사용해야한다.
```sql
CREATE OR REPLACE Function sample_function()
RETURNS VOID
AS $$
BEGIN
  UPDATE sample_table
  SET sample_columns = 'sample_values'
  WHERE sample_columns IS NULL;
END;
$$ LANGUAGE plpgsql;
```

#### 2) Function 실행 방법
```sql
select sample_function();
```

#### 3) 매개변수를 사용하는 Function
```sql
CREATE OR REPLACE Function sample_function(smaple_parameters VARCHAR)
RETURNS VOID
AS $$
BEGIN
  UPDATE sample_table
  SET sample_columns = 'sample_values'
  WHERE sample_columns = smaple_parameters;
END;
$$ LANGUAGE plpgsql;
```

#### 4) Function 예외처리
```sql
CREATE OR REPLACE Function sample_function(smaple_parameters VARCHAR)
RETURNS VOID
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

#### 5) Function Return 타입
1) 스칼라 타입 (Scalar Types)<br/>
`INTEGER`, `VARCHAR`, `CHAR`, `BOOLEAN`, `NUMERIC`, `DATE`, `TIMESTAMP` 등과 같이 단일 값을 가지는 데이터 타입<br/>
```sql
CREATE OR REPLACE FUNCTION sample_function(x INTEGER, y INTEGER)
RETURNS INTEGER 
AS $$
DECLARE
    result INTEGER;
BEGIN
    result := x + y;
    RETURN result;
END;
$$ LANGUAGE plpgsql;
```
<br/>

2) 레코드 타입 (Record Types)<br/>
여러 필드를 포함하는 복합 데이터 타입<br/>
```sql
CREATE OR REPLACE FUNCTION sample_function(x VARCHAR, y INTEGER)
RETURNS RECORD AS $$
DECLARE
    result RECORD;
BEGIN
    result.x := x;
    result.y := y;
    
    RETURN user_info;
END;
$$ LANGUAGE plpgsql;
```
<br/>

3) 테이블 타입 (Table Types)<br/>
여러 행을 가지는 데이터 집합을 반환하는 데이터 타입<br/>
```sql
CREATE OR REPLACE FUNCTION sample_function()
RETURNS TABLE(sample_columns VARCHAR) AS $$
BEGIN
    RETURN QUERY SELECT * FROM sample_table;
END;
$$ LANGUAGE plpgsql;
```
<br/>


4) 커서 타입 (Cursor Types)<br/>
결과 집합을 가리키는 커서를 반환<br/>
```sql
CREATE OR REPLACE FUNCTION sample_function()
RETURNS REFCURSOR AS $$
DECLARE
    result_cursor REFCURSOR;
BEGIN
    OPEN result_cursor FOR SELECT * FROM sample_table;
    RETURN result_cursor;
END;
$$ LANGUAGE plpgsql;
```
```sql
BEGIN;
DECLARE
    my_cursor REFCURSOR;
    row RECORD;
BEGIN
    my_cursor := sample_function();
    FETCH ALL FROM my_cursor;
    CLOSE my_cursor;
END;
COMMIT;
```
<br/>

5) 다양한 특수 타입<br/>
```sql
CREATE TYPE sample_type AS (
    sample_col1 VARCHAR,
    sample_col2 VARCHAR
);
```
```sql
CREATE OR REPLACE FUNCTION sample_function(x INTEGER, y VARCHAR)
RETURNS sample_type AS $$
DECLARE
    smth sample_type;
BEGIN
    -- 예제를 위해 고정된 주소 값을 반환
    smth.sample_col1 := x;
    smth.sample_col2 := y;
    RETURN smth;
END;
$$ LANGUAGE plpgsql;
```