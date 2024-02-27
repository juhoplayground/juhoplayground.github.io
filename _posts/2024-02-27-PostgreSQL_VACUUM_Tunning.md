---
layout: post
title: PostgreSQL - VACUUM 튜닝
author: 'Juho'
date: 2024-02-27 09:00:00 +0900
categories: [PostgreSQL, Database, VACUUM]
tags: [PostgreSQL, Database, VACUUM]
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
1. [Vacuum 지연이 발생하는 테이블 확인](#vacuum-지연이-발생하는-테이블-확인)
2. [Autovacuum 발생할 것 같은 테이블 확인](#autovacuum-발생할-것-같은-테이블-확인)
3. [Vacuum 튜닝](#vacuum-튜닝)

## Vacuum 지연이 발생하는 테이블 확인
```sql
SELECT datname, usename, pid, state, wait_event, current_timestamp - xact_start AS xact_runtime, query
FROM pg_stat_activity
WHERE upper(query) LIKE '%VACUUM%'
ORDER BY xact_start;
```

해당 쿼리 세션 종료하기
```sql
pg_terminate_backend(pid)
```

문제가 되는 테이블 autovacuum 일시 정지
```sql
ALTER TABLE test.test_demo SET (autovacuum_enabled = false);
```

## Autovacuum 발생할 것 같은 테이블 확인
transaction wraparound 발생 방지를 위해서 autovacuum_freeze_max_age에 다다르면 autovacuum을 실행한다.<br/>
그렇기에 table의 age가 높으면 다음 actovacuum 대상으로 선정될 가능성이 높다.<br/>
또한, vacuum을 실행할 최소 dead tuple 수는 기본으로 50이고(autovacuum_vacuum_threshold)<br/>
dead tuple 비율은 기본으로 0.2다(autovacuum_vacuum_scale_factor).<br/>

다음 쿼리를 사용하면 age와 dead tuple의 크기와 비율이 높은 상위 테이블을 확인할 수 있다.<br/>
```sql
SELECT a.relname as table_name,
       greatest(age(b.relfrozenxid), age(c.relfrozenxid)) AS age,
       a.n_live_tup              AS live_tuples,
       a.n_dead_tup              AS dead_tuples,
       a.n_dead_tup / a.n_live_tup AS ratio,
       pg_size_pretty(pg_table_size(a.relid)) AS table_size,
       a.last_autovacuum         AS last_autovacuum,
       a.last_autoanalyze        AS last_autoanalyze
FROM pg_stat_user_tables AS a
LEFT JOIN pg_class AS b
ON a.relid = b.oid
LEFT JOIN pg_class AS c
ON b.reltoastrelid = c.oid
WHERE b.relkind = 'r'
ORDER BY age desc , ratio desc
LIMIT 10;
```

## Vacuum 튜닝
1) autovacuum_vacuum_scale_factor 변경
테이블이 커질수록 vacuum이 수행될 때 정리해야할 dead tuple의 수가 많아지므로 부하가 커지는 상황이 발생할 수 있다.
autovacuum_vacuum_scale_factor를 작은 값으로 변경하면 vacuum이 자주 실행되는 효과가 있다.<br/>
```
ALTER TABLE test.test_demo SET (autovacuum_vacuum_scale_factor = 0.1, autovacuum_vacuum_threshold = 50);
```

2) vacuum_cost_xx 변경
- autovacuum_vacuum_cost_limit = 200 : Autovacuum이 한번 실행될 때 마다 해당 프로세스는 200만큼의 credit을 가진다.<br/>
- autovacuum_vacuum_codt_delay = 2 : AutoVacuum이 autovacuum_vacuum_cost_limit 만큼 완료되면 다음 AutoVacuum은 2ms 동안 sleep한다.<br/>
- vacuum_cost_page_hit = 1 : page_hit (shared_buffer)에 있는 데이터를 Vacuum 할 때 마다 1 의 credit을 소모<br/>
- vacuum_cost_page_miss = 10: page_miss (디스크 영역)에 있는 데이터를 Vacuum할 때마다 10의 credit을 소모<br/>
- vacuum_cost_page_dirty = 20: Dead Tuple을 Vacuum할 때마다 20의 credit을 소모<br/>

주어진 credit을 모두 소진하면 autovacuum프로세스는 종료됨.<br/>
credit이 너무 작으면 autovacuum이 dead tuple을 다 정리하지 못한 채 종료되서 dead tuple이 계속 누적되는 경우가 발생할 수 있다.<br/>
그렇기 때문에 cost값을 증가시키면 좀 더 autovacuum이 오래 실행되어서 dead tuple을 정리못하는 경우가 줄어드는 효과가 있다.<br/>

AWS RDS 기본값은 `GREATEST({log(DBInstanceClassMemory/21474836480)600},200)` 계산식으로 결정된다.<br/>

3) autovacuum_max_workers 변경
큰 테이블의 vacuum으로 인해서 작은 테이블의 vacuum을 하지 못할때 이 값을 늘리면 효과적일 수 있다.<br/>
또 partitioning이 되어 있는 테이블의 경우 각 파티션별로 autovacuum worker가 할당되어 빠른 vacuum 작업을 수행할 수 있다.<br/>
AWS RDS 기본값은 `GREATEST({DBInstanceClassMemory/64371566592},3)` 계산식으로 결정된다.<br/>

4) autovacuum_work_mem 변경 
테이블을 스킨할때 autovacuum은 메모리에서 dead tuple을 수집하는데, 수집할 수 있는 dead tuple의 수는 autovacuum_work_mem에 의해 결정된다.<br/>
이 값을 큰 값으로 변경하면 autovacuum 주기당 더 많은 dead tuple을 수집할 수 있다.<br/>
AWS RDS 기본값는 `GREATEST({DBInstanceClassMemory/32768},131072)` 계산식으로 결정된다.<br/>

5) maintenance_work_mem 변경
작은 규모의 테이블이 여러 개 있는 경우에는 autovacuum_max_workers는 많이, 그리고 maintenance_work_mem은 적게 설정하는 게 좋다.<br/>
크기가 큰 테이블이라면 autovacuum_max_workers는 적게, maintenance_work_mem은 크게 설정하는 것이 좋다.<br/>
일반적으로 1~2GB로 설정한다.<br/>
AWS RDS 기본값은 `GREATEST({DBInstanceClassMemory/63963136*1024},65536)` 계산식으로 결정된다.<br/>

6) max_parallel_maintenance_workers 변경
index 생성이나 vacuum 작업을 병렬로 수행해서 처리 속도에 영향을 줄 수 있다.<br/>
기본값은 2인데, 해당 작업이 너무 느리면 4개~8개 정도로 설정하면 도움이 될 수 있다.<br/>
parallel 관련 파라미터(max_parallel_workers, max_parallel_workers_per_gather)도 확인해야 한다.<br/>

<br/>

---
Autovacuum 튜닝에 정답은 없는 것 같다.<br/>
Autovacuum을 off 하고 새벽시간마다 vacuum하면 dead tuple이나 frozen 작업이 많아져서 더 큰 부하를 유발하기도 하고<br/>
Autovacuum을 자주 실행되도록 하면 vacuum의 부하가 쿼리 성능에 영향을 끼치고 예상하지 못한 lock 대기로 장애 상황이 발생하기도 한다.<br/>
Autovacuum이 적절한 빈도로 실행되도록 dead tuple 정리와 vacuum 부하 간 적절한 균형을 찾도록 모니터링을 하면서 확인 꾸준히 해야할 것 같다.<br/>
<br>

또 Vaccuum full 대신에 [pg_repack](https://github.com/reorg/pg_repack){:target="_blank"}라는 오픈소스도 있다.<br/>
vacuum full을 하지 못할 상황에서의 대안이 될 수 있을 것 같다.<br/>