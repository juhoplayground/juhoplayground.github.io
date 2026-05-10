---
layout: post
title: "SQLGlot - 31개 SQL 방언을 다루는 Python 파서·트랜스파일러·옵티마이저"
author: 'Juho'
date: 2026-05-10 00:00:00 +0900
categories: [Python]
tags: [Python, Database, Compiler]
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
2. [핵심 기능](#핵심-기능)
   - [방언 지원과 트랜스파일](#방언-지원과-트랜스파일)
   - [에러 검출과 쿼리 분석](#에러-검출과-쿼리-분석)
   - [옵티마이저와 실행 엔진](#옵티마이저와-실행-엔진)
3. [설치와 사용 예시](#설치와-사용-예시)
4. [실제 도입 사례](#실제-도입-사례)
5. [버전 정책](#버전-정책)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

SQLGlot은 외부 의존성이 없는 Python 기반의 SQL 파서, 트랜스파일러, 옵티마이저, 실행 엔진이다.
다양한 SQL 입력을 읽어 들여 목표로 하는 방언으로 문법적·의미적으로 올바른 SQL을 출력하는 것이 핵심 기능이다.
순수 Python으로 구현되어 있으며, 성능이 필요한 경우 컴파일된 C 확장을 함께 설치할 수도 있다.

## 핵심 기능

### 방언 지원과 트랜스파일

SQLGlot은 31개의 SQL 방언을 지원한다.
지원 대상에는 DuckDB, Presto, Trino, Spark, Databricks, Snowflake, BigQuery, PostgreSQL, MySQL 등이 포함된다.
방언은 다음 세 가지 카테고리로 관리된다.

| 카테고리 | 설명 |
|----------|------|
| Official | 코어 팀이 직접 유지보수 |
| Community | 외부 컨트리뷰터가 주도 |
| Plugin | 서드파티 패키지로 제공 |

쿼리를 한 방언에서 다른 방언으로 변환할 수 있고, SQL 포맷팅도 가능하다.
주석은 베스트 에포트 기준으로 보존된다.
복잡한 변환에는 날짜/시간 함수 변환과 식별자 구분자 변환이 포함된다.

### 에러 검출과 쿼리 분석

파서는 괄호 불일치, 예약어 오용 같은 문법 오류를 식별하며, 프로그램에서 다루기 쉬운 구조화된 에러 정보를 제공한다.
쿼리 분석 측면에서는 표현식 트리를 순회할 수 있고, 컬럼과 테이블 참조 같은 메타데이터를 추출하거나, 두 SQL 표현식 간의 의미 차이를 비교하는 시맨틱 디핑이 가능하다.

### 옵티마이저와 실행 엔진

내장 옵티마이저는 다음과 같은 기법들을 적용한다.

| 최적화 기법 | 설명 |
|-------------|------|
| Boolean logic simplification | 불 논리 단순화 |
| Constant folding | 상수 폴딩 |
| Type-sensitive transformations | 스키마 정보가 주어졌을 때 타입에 민감한 변환 |

또한 Python 딕셔너리를 대상으로 SQL 쿼리를 직접 해석·실행할 수 있다.
별도 DB 없이도 SQL 표현식을 테스트하거나 Python 데이터에 SQL 인터페이스를 얹는 용도에 활용할 수 있다.

## 설치와 사용 예시

설치는 다음과 같이 수행한다.

```bash
pip3 install sqlglot
pip3 install "sqlglot[c]"
```

`sqlglot[c]`는 컴파일된 C 확장을 함께 설치하여 성능을 끌어올린다.
개발 의존성은 `make install-dev`로 설치할 수 있다.

다음은 DuckDB SQL을 Hive SQL로 트랜스파일하는 예시이다.

| 입력 (DuckDB) | 출력 (Hive) |
|---------------|-------------|
| SELECT EPOCH_MS(1618088028295) | SELECT FROM_UNIXTIME(1618088028295 / POW(10, 3)) |

빌더 API를 사용해 SQL을 프로그래밍 방식으로 구성하는 것도 가능하다.

```python
from sqlglot import select, condition

sql = (
    select("*")
    .from_("y")
    .where(condition("x=1").and_("y=1"))
    .sql()
)

print(sql)
```

위 코드는 다음 SQL을 생성한다.

```sql
SELECT * FROM y WHERE x = 1 AND y = 1
```

## 실제 도입 사례

SQLGlot은 데이터 엔지니어링 생태계의 다양한 도구의 기반으로 사용되고 있다.

| 도구 | 카테고리 |
|------|----------|
| SQLMesh | 데이터 변환 프레임워크 |
| Apache Superset | BI/시각화 |
| Dagster | 워크플로우 오케스트레이션 |
| Fugue | 분산 처리 추상화 |
| Ibis | 분석용 표현식 라이브러리 |

이렇게 폭넓은 도구들이 SQLGlot을 채택했다는 점은 단순 파서를 넘어 멀티 방언 SQL 처리 인프라의 표준급 라이브러리로 자리잡았음을 보여준다.

## 버전 정책

SQLGlot은 시맨틱 버저닝을 따른다.
PATCH 버전은 호환되는 수정 사항, MINOR 버전은 하위 호환성이 깨지는 변경, MAJOR 버전은 대규모 리비전을 의미한다.
프로덕션 도입 시에는 MINOR 업그레이드에서도 호환성 검증이 필요하다는 점을 주의해야 한다.

## 결론

SQLGlot은 외부 의존성 없는 Python으로 31개 SQL 방언을 다루는 종합 SQL 처리 라이브러리이다.
파싱, 트랜스파일, 포맷팅, 에러 검출, 옵티마이저, 실행까지 한 패키지에서 제공되며, SQLMesh, Superset, Dagster, Fugue, Ibis 같은 데이터 도구들의 기반으로 채택되어 있다.
멀티 방언 환경에서 SQL을 일관되게 다뤄야 할 때, 그리고 빌더 기반 SQL 생성과 시맨틱 분석이 필요할 때 매우 유용한 선택지이다.

## Reference

- [SQLGlot GitHub Repository](https://github.com/tobymao/sqlglot)
