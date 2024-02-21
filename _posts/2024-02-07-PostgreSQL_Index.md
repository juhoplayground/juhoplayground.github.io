---
layout: post
title: PostgreSQL - Index 기초
author: 'Juho'
date: 2024-02-07 13:00:00 +0900
categories: [PostgreSQL, Database, Index]
tags: [PostgreSQL, Database, Index]
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
1. [인덱스의 개념](#인덱스의-개념)
2. [PostgreSQL INDEX 설명](#postgresql-index-설명)
3. [인덱스가 사용되는 경우](#인덱스가-사용되는-경우)
4. [인덱스 확인 방법](#인덱스-확인-방법)
5. [지원하는 Index 종류](#지원하는-index-종류)
6. [파티셔닝된 테이블에서 Index 동작 방법](#파티셔닝된-테이블에서-index-동작-방법)
7. [Index의 정렬 순서](#index의-정렬-순서)
8. [Index와 통계정보](#index와-통계정보)
9. [Index 생성 속도](#index-생성-속도)
10. [병렬 Index 빌드 기능](#병렬-index-빌드-기능)


## 인덱스의 개념
인덱스는 테이블의 조회 속도를 높여주는 자료구조다.<br/>
인덱스가 설정되지 않았다면 Table Full Scan이 일어나 성능이 저하된다.<br/>
인덱스의 단점으로는 Select 속도는 빨라지지만 UPDATE, INSERT, DELETE의 속도는 저하가 된다는 점이다.<br/>
때문에 효율적인 인덱스 설계를 하여 단점을 최소화해야한다.<br/>

## PostgreSQL INDEX 설명
하지만 무조건적인 UPDATE, DELETE의 속도 저하를 제공하지는 않는다. <br/>
인덱스는 검색 조건이 있는 UPDATE 및 DELETE 명령에도 이점을 제공할 뿐만 아니라 조인 검색에도 사용할 수 있기 때문이다.<br/>

큰 테이블에 인덱스를 생성하는 것은 시간이 오래 걸릴 수 있다.<br/>

기본적으로 PostgreSQL은 SELECT를 인덱스 작성과 동시에 허용하지만 INSERT, UPDATE, DELETE는 인덱스 작성이 완료될 때까지 차단된다.<br/>
인덱스 생성과 동시에 INSERT, UPDATE, DELETE가 발생하도록 허용하는것도 가능하지만, 주의해야할 사항이 있다.<br/>

인덱스가 생성된 후 시스템은 테이블과 동기화를 유지해야 한다.<br/>
이로 인해 데이터 조작 작업에 오버헤드가 추가된다.<br/>
또한 인덱스는 heap-only 튜플의 생성을 방지할 수 있다. <br>
따라서 쿼리에서 드물게 또는 전혀 사용되지 않는 인덱스는 제거해야 한다.<br/>

인덱스의 주요 키 필드는 컬럼 이름으로 지정되거나, 괄호 안에 표현식으로 작성될 수 있다. <br/>
표현식을 사용하여 기본 데이터의 변형을 기반으로 데이터에 빠르게 엑세스할 수 있다.<br/>
예를 들어, upper(col)에서 계산된 인덱스는 WHERE upper(col) = 'JIM' 절을 사용하여 인덱스를 활용할 수 있다.<br/>

PostgreSQL은 B-tree, 해시, GiST, SP-GiST, GIN 및 BRIN 인덱스 방법을 제공한다.<br/>
사용자는 자체 인덱스 방법을 정의할 수도 있지만, 이는 꽤 복잡하다<br/>

## 인덱스가 사용되는 경우
쿼리가 순차적인 테이블 스캔보다 효율적일 것으로 판단되는 경우에는 인덱스를 사용한다. <br/>
쿼리 플래너가 합리적인 결정을 내릴 수 있도록 통계를 업데이트하기 위해 정기적으로 ANALYZE 명령을 실행해야 할 수도 있다.<br/>
PostgreSQL에서는 기본적으로 hint가 존재하지 않는다.<br/>
그래서 [`pg_hint_plan'](https://github.com/ossc-db/pg_hint_plan?tab=readme-ov-file){:target="_blank"}이라는 extension을 사용해야한다. <br/>



## 인덱스 확인 방법
`pg_indexes`를 통해서 생성된 인덱스를 확인할 수 있다.

```sql
select * from pg_indexes where tablename = 'test1';
```

<table>
    <tr>
        <td>schemaname</td>
        <td>tablename</td>
        <td>indexname</td>
        <td>tablespace</td>
        <td>indexdef</td>
    </tr>
    <tr>
        <td>test</td>
        <td>test1</td>
        <td>test1_id_index</td>
        <td></td>
        <td>CREATE INDEX test1_id_index ON test.test1 USING btree (id)</td>
    </tr>
</table>
B-TREE를 이용해 인덱스를 찾음을 알 수 있다.<br/>

## 지원하는 Index 종류
현재, 다중 키 컬럼 인덱스를 지원하는 인덱스 방법은 B-tree, GiST, GIN, BRIN만 있다. <br/>
다중 키 컬럼이 있는지 여부는 INCLUDE 열을 인덱스에 추가할 수 있는지와는 독립적이다.<br/>
인덱스에는 INCLUDE 열을 포함하여 최대 32개의 컬럼을 가질 수 있다.<br/>
PostgreSQL을 빌드할 때 이 한계를 변경할 수 있습니다.<br/> 
현재 B-tree만 고유한 인덱스를 지원한다. <br/>



## 파티셔닝된 테이블에서 Index 동작 방법
파티셔닝된 테이블에서 CREATE INDEX가 호출되면, 기본 동작은 모든 파티션에 해당하는 인덱스가 있는지 확인하기 위해 재귀적으로 이동한다.<br/>
각 파티션은 먼저 해당 파티션에 해당하는 인덱스가 이미 존재하는지 여부를 확인하고, 그렇다면 해당 인덱스가 생성 중인 인덱스의 파티션 인덱스로 첨부되어 해당 인덱스의 부모 인덱스가 된다.<br/>
각 파티션의 새 인덱스 이름은 명령에서 인덱스 이름이 지정되지 않은 것처럼 결정된다.<br/>
 ONLY 옵션이 지정된 경우 재귀가 수행되지 않고 인덱스가 무효로 표시된다.<br/>
`ALTER INDEX ... ATTACH PARTITION`은 모든 파티션이 일치하는 인덱스를 획득한 후에 인덱스를 유효하게 표시한다.<br/>
`CREATE TABLE ... PARTITION OF`를 사용하여 생성되는 모든 파티션은 ONLY가 지정되었는지 여부와 관계없이 자동으로 일치하는 인덱스를 갖게 된다.<br/>


## Index의 정렬 순서
정렬된 스캔을 지원하는 인덱스 방법은 현재는 B-tree만 해당한다.<br/>
선택적인 ASC, DESC, NULLS FIRST, NULLS LAST을 지정하여 인덱스의 정렬 순서를 수정할 수 있다.<br/>
정렬된 인덱스는 보통 앞뒤로 스캔될 수 있으므로, 단일 컬럼 DESC 인덱스를 생성하는 것은 일반적으로 유용하지 않다.<br/>
그 정렬 순서는 이미 일반적인 인덱스로 사용 가능하다.<br/>
이러한 옵션의 가치는 혼합된 순서화 쿼리 ex) `ORDER BY x ASC, y DESC`와 같은 정렬 순서와 일치하는 다중 컬럼 인덱스를 생성할 수 있다는 것 이다.<br/>
NULLS 옵션은 정렬 단계를 피하기 위해 인덱스에 의존하는 쿼리에서 `nulls sort high`가 아닌 `nulls sort low` 동작을 지원해야 할 때 유용하다.<br/>


## Index와 통계정보
시스템은 주기적으로 모든 테이블 컬럼에 대한 통계를 수집한다.<br/>
새로 생성된 비표현식 인덱스는 즉시 이러한 통계를 활용하여 인덱스의 유용성을 판단할 수 있다.<br/>
표현식 인덱스의 경우, 해당 인덱스에 대한 통계를 생성하기 위해 ANAZLYZE를 실행하거나 autovacuum 데몬이 테이블을 분석하도록 대기해야한다.<br/>


## Index 생성 속도
대부분의 인덱스 방식에서, 인덱스를 생성하는 속도는 maintenance_work_mem 설정에 따라 달라진다.<br/>
큰 값은 인덱스 생성에 필요한 시간을 줄일 수 있다.<br/>
하지만 실제로 사용 가능한 메모리 양보다 크게 설정하면, 시스템이 스와핑되어 성능이 저하될 수 있다. <br/>
max_parallel_maintenance_workers를 늘리면 더 많은 워커를 사용할 수 있어 인덱스 생성에 필요한 시간을 줄일 수 있다.<br/>
단, 이미 I/O 바운드 상태인 경우에는 해당되지 않는다. <br.>


## 병렬 Index 빌드 기능
PostgreSQL은 여러개의 CPU를 활용하여 테이블 row를 빠르게 처리하기 위해 인덱스를 구축할 수 있다. <br/>
현재는 B-tree만 병렬 인덱스를 지원하는데, maintenance_work_mem은 시작된 워커 프로세스의 수와 상관없이 각 인덱스 빌드 작업 전체에서 사용할 수 있는 최대 메모리 양을 지정한다.<br/>
일반적으로 비용 기반의 모델은 필요한 경우 얼마나 많은 워커 프로세스를 요청해야 하는지 자동으로 결정한다.<br/>
병렬 인덱스 빌드는 maintenance_work_mem을 늘리면 동등한 직렬 인덱스 빌드와 비교해 효과를 더 얻을 수 있다.<br/>
maintenance_work_mem은 병렬 워커 프로세스의 수에 영향을 미칠 수 있다.<br/>
병렬 워커는 총 maintenance_work_mem 예산에서 최소 32MB의 할당을 가져야한다.<br/>
또한 리더 프로세스에는 나머지 32MB의 할당이 있어야한다.<br/>
ALTER TABLE을 통한 parallel_workers 값 설정은 CREATE INDEX에 대한 테이블의 병렬 워커 프로세스 요청 수를 직접 제어한다.<br/>
비용 기반 모델을 완전히 우회하며, maintenance_work_mem이 얼마나 많은 병렬 워커를 요청할지에 영향을 미치지 않는다. <br/>
ALTER TABLE을 통해 parallel_workers를 0으로 설정하면 모든 경우에 해당 테이블에서 병렬 인덱스 빌드가 비활성화된다. <br/>
인덱스 빌드 조정 중에 parallel_workers를 설정한 후에는 다시 초기화하는 것이 좋다.<br/>
그 이유는 parallel_workers는 모든 병렬 테이블 스캔에 영향을 미치기 때문에 쿼리 계획에 무심코 변경이 일어나는 것을 방지할 수 있다.<br/>
CONCURRENTLY 옵션을 사용한 CREATE INDEX는 별도의 제한 없이 병렬 빌드를 지원하지만 실제로는 첫 번째 테이블 스캔만 병렬로 수행된다.<br/>

<br/>

---
다음글은 지원하는 Index 종류에 대해서 자세하게 확인해보려고 한다.<br/>