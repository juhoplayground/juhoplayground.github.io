---
layout: post
title: PostgreSQL - Benchmark
author: 'Juho'
date: 2024-02-21 09:00:00 +0900
categories: [PostgreSQL, Database, Benchmark]
tags: [PostgreSQL, Database, Benchmark]
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
1. [밴치마크란?](#밴치마크란)
2. [pgbench란?](#pgbench란)
3. [pgbench 사용하는 방법](#pgbench-사용하는-방법)
4. [pgbench 실습](#pgbench-실습)

## 밴치마크란?
벤치마크(Benchmark)는 컴퓨팅에서 특정 오브젝트(하드웨어 또는 소프트웨어 등)에 대해 일반적으로 수많은 표준 테스트와 시도를 수행함으로써 오브젝트의 상대적인 성능 측정을 목적으로 컴퓨터 프로그램을 실행하는 행위다.<br/>

## pgbench란?
PostgreSQL에서는 설치시 기본적으로 `pgbench`라는 Benchmark tool을 제공한다.<br/>
기본적으로 pgbench는 각 트랜잭션에 SELECT, UPDATE 및 INSERT 명령이 다섯 번 포함된 일련의 SQL 명령어를 여러번 실행하여 TPC-B라 불리는 평균 트랜잭션 속도(초당 트랜잭션 수)를 계산하여 성능을 측정한다.<br/>
하지만 사용자가 자신만의 트랜잭션 스크립트 파일을 작성하여 다른 케이스를 테스트할 수도 있다.<br/>

pgbench는 지정된 목록에서 무작위로 선택된 테스트 스크립트를 실행한다.<br/>
이 스크립트는 -b로 지정된 내장 스크립트와 -f로 지정된 사용자 제공 스크립트가 포함될 수 있다.<br/>
각 스크립트는 @ 뒤에 상대적인 가중치를 지정하여 선택 확률을 변경할 수 있다.<br/>
기본 가중치는 1이고 가중치가 0인 스크립트는 무시된다.<br/>

기본 내장 트랜잭션 스크립트(-b tpcb-like로도 호출됨)는 각 트랜잭션마다 임의로 선택된 aid, tid, bid 및 delta를 사용하여 7개의 명령을 실행한다.<br/>
이 시나리오는 TPC-B 벤치마크에서 영감을 받았지만 실제로는 TPC-B가 아니기 때문에 이와 같은 이름이 지어졌다.<br/>
```sql
BEGIN;

UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;

SELECT abalance FROM pgbench_accounts WHERE aid = :aid;

UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;

UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;

INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);

END;
```
간단한 업데이트 내장 스크립트(-N로도 선택 가능)를 선택하면 단계 4와 5가 트랜잭션에 포함되지 않는다.<br/>
이렇게 하면 이러한 테이블에서 업데이트 충돌이 발생하지 않지만, 테스트 케이스가 실제 TPC-B와 더 이상 유사하지 않게 된다.<br/>

select-only 내장 스크립트(-S로도 선택 가능)를 선택하면 SELECT만 실행된다.<br/>

pgbench의 일반적인 실행 결과는 아래와 같다.<br/>
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
number of transactions per client: 1000
number of transactions actually processed: 10000/10000
number of failed transactions: 0 (0.000%)
latency average = 11.013 ms
latency stddev = 7.351 ms
initial connection time = 45.758 ms
tps = 896.967014 (without initial connection time)
```

첫 일곱 줄은 가장 중요한 매개변수 설정 중 일부를 보고합니다.<br/>
여섯 번째 줄은 직렬화 또는 데드락 오류가 발생한 트랜잭션에 대한 시도 횟수의 최댓값을 보고합니다.<br/>
여덟 번째 줄은 완료된 트랜잭션의 수와 의도된 트랜잭션 수를 보고합니다.<br/>
후자는 클라이언트 수와 클라이언트 당 트랜잭션 수의 곱이다.<br/>
이 값들은 실행이 완료되기 전에 실패했거나 일부 SQL 명령이 실패한 경우에만 동일합니다.<br/>
-T 모드에서는 실제 트랜잭션 수만 인쇄된다.<br/>
다음 줄은 직렬화 또는 데드락 오류로 인한 실패한 트랜잭션 수를 보고합니다.<br/>
마지막 줄은 초당 트랜잭션 수를 보고합니다.<br/>


## pgbench 사용하는 방법
기본 TPC-B와 유사한 트랜잭션 테스트를 위해서는 미리 특정 테이블이 설정되어 있어야 한다.<br/>
이러한 테이블을 생성하고 채우기 위해 pgbench를 -i(초기화) 옵션과 함께 호출해야 한다.<br/>
사용자 정의 스크립트를 테스트하는 경우 위의 단계가 필요하지 않지만, 대신 테스트에 필요한 설정을 수행해야 한다.<br/>

```
pgbench -i [ options ] dbname
```
주의할 사항으로는 pgbench -i는 pgbench_accounts, pgbench_branches, pgbench_history 및 pgbench_tellers 네 개의 테이블을 생성하며, 이와 같은 이름의 기존 테이블을 삭제한다.<br/>

기본 "스케일 팩터"가 1인 경우<br/>
```
table                   # of rows
---------------------------------
pgbench_branches        1
pgbench_tellers         10
pgbench_accounts        100000
pgbench_history         0
```
테이블에 처음에는 이만큼의 행이 포함되어 있다.<br/>
필요한 경우 -s(스케일 팩터) 옵션을 사용하여 행 수를 늘릴 수 있다.<br/>
또한 -F(fillfactor) 옵션도 이 시점에 사용할 수 있다.<br/>
필요한 설정을 완료한 후에는 -i를 포함하지 않는 명령을 사용하여 아래의 명령어로 벤치마크를 실행할 수 있다.<br/>
```
pgbench [ options ] dbname
```
대부분의 경우, 유용한 테스트를 수행하기 위해 몇 가지 옵션이 필요하다.<br/>
가장 중요한 옵션은 -c(클라이언트 수), -t(트랜잭션 수), -T(시간 제한), 그리고 -f(사용자 정의 스크립트 파일 지정)이다.<br/>
다양한 옵션은 [`pgbench`](https://www.postgresql.org/docs/current/pgbench.html){:target="_blank"}문서를 확인하면 좋을 것 같다.<br/>

가장 중요한 옵션은 -c(클라이언트 수), -t(트랜잭션 수), -T(시간 제한), 그리고 -f(사용자 정의 스크립트 파일 지정)이다.<br/>

성공적인 실행은 상태 코드 0으로 종료된다.<br/>
상태 코드 1은 잘못된 명령줄 옵션이나 절대 발생하지 않아야 하는 내부 오류와 같은 정적 문제를 나타낸다.<br/>
초기 연결 실패와 같이 벤치마크를 시작할 때 발생하는 초기 오류도 상태 코드 1로 종료된다.<br/>
스크립트에서 발생하는 데이터베이스 오류나 문제와 같이 실행 중에 발생하는 오류는 상태 코드 2로 종료됩니다.<br/>
후자의 경우 pgbench는 부분 결과를 출력한다.<br/>

유용한 결과를 얻기 위해서는 몇 초 동안 실행되는 테스트를 절대로 믿으면 안된다.<br/>
노이즈를 평균화하기 위해 실행 시간을 최소한 몇 분 이상 유지하려면 -t 또는 -T 옵션을 사용해야한다.<br/>
일부 경우에는 재현 가능한 숫자를 얻으려면 몇 시간이 걸릴 수 있다.<br/>
테스트를 몇 번 실행하여 숫자가 재현 가능한지 확인하는 것이 좋다.<br/>

기본 TPC-B와 유사한 테스트 시나리오의 경우, 초기화 스케일 팩터(-s)는 테스트하려는 가장 큰 클라이언트 수(-c)와 적어도 같아야 한다.<br/>
그렇지 않으면 대부분의 경우 업데이트 충돌만 측정하게 된다.<br/>
pgbench_branches 테이블에는 -s 개의 행만 있으며, 각 트랜잭션은 그 중 하나를 업데이트하려고 한다.<br/>
따라서 -c 값이 -s를 초과하면 다른 트랜잭션을 기다리는 트랜잭션이 많이 발생한다.<br/>

기본 테스트 시나리오는 테이블을 초기화한 후 경과된 시간에도 상당히 민감하다.<br/>
테이블에 불필요한 행과 공간이 축적되면 결과가 변경된다.<br/>
결과를 이해하려면 업데이트의 총 수와 진행 상황을 추적해야 한다.<br/>
autovacuum이 활성화되어 있는 경우 측정된 성능에 예측할 수 없는 변경이 발생할 수 있다.<br/>

pgbench의 한계는 많은 클라이언트 세션을 테스트할 때 본인이 병목이 될 수 있다는 것이다.<br/>
데이터베이스 서버와 다른 컴퓨터에서 pgbench를 실행하여 이를 완화할 수 있다.<br/>
그러나 낮은 네트워크 지연이 필수적이게 된다.<br/>
동일한 데이터베이스 서버에 대해 여러 클라이언트 머신에서 동시에 여러 pgbench 인스턴스를 실행하는 것이 유용할 수도 있다.<br/>

## pgbench 실습
1. 테스트용 DB를 생성한다.
```sql
CREATE DATABASE pgbench_test OWNER TO posgres;
```
```
\c pgbench_test postgres
```

2. 초기화 이전 상태
```
\dt+
```

3. 초기화 실행
```
pgbench -h localhost -p 5432 -U postgres -i -s 10 pgbench_test
```

4. 초기화 이후 상태
```
\dt+
```
```sql
SELECT schemaname,relname,n_live_tup FROM pg_stat_user_tables;
```
```
\l+
```

5. 벤치마크 성능 테스트
<br/>
-c : 시뮬레이션할 클라이언트 수로, 동시에 실행되는 데이터베이스 세션 수, 기본 값은 1<br/>
-j : pgbench 내에서의 워커 스레드 수<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; c는 항상 j보다 같거나 커야한다. 그렇지 않으면 j값은 무시되고, c값으로 대체된다.<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; c는 j의 배수여야한다.
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 기본 값은 1<br/>
-t : 각 클라리언트가 실행하는 트랜잭션의 수. 기본 값은 10<br/>

```
pgbench -h localhost -p 5432 -U postgres -c 8 -j 4 -t 10 pgbench_test
```



<br/>

---

위와 같은 결과를 바탕으로 postgres.conf에서 DB 성능에 영향을 주는 parameter를 바꿔가면서 pgbench를 반복적으로 실행하면서 튜닝을 하면 된다.<br/>
pgbench는 간단하게 PostgreSQL의 성능을 측정하기 좋은 것 같다.<br/>
하지만 TPC-B라는 측정 방법이 실제 Application에서 DB를 이용하는 방법과 차이가 크고, 튜닝에 필요한 자세한 정보를 확인하기 어려운 것 같다.<br/>