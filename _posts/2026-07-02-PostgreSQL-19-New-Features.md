---
layout: post
title: "PostgreSQL 19 미리보기: 시간 테이블과 주요 신기능"
author: 'Juho'
date: 2026-07-02 00:00:00 +0900
categories: [PostgreSQL]
tags: [PostgreSQL, Database, VACUUM]
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
2. [시간 테이블(Temporal Tables)](#시간-테이블temporal-tables)
   - [WITHOUT OVERLAPS 제약](#without-overlaps-제약)
   - [FOR PORTION OF](#for-portion-of)
3. [운영과 성능 개선](#운영과-성능-개선)
   - [REPACK CONCURRENTLY와 파티셔닝](#repack-concurrently와-파티셔닝)
   - [논리적 복제와 Autovacuum](#논리적-복제와-autovacuum)
   - [SQL 품질과 성능](#sql-품질과-성능)
4. [한계와 주의사항](#한계와-주의사항)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

PostgreSQL 19가 beta 단계에 접어들며 주요 신기능들이 공개됐다.
가장 눈에 띄는 변화는 SQL:2011 표준의 애플리케이션 시간(application-time)을 도입한 시간 테이블 지원이다.
이 외에도 무중단 테이블 재구성, 파티션 병합·분할, 논리적 복제 성숙화, autovacuum 병렬화 등 운영자와 개발자 모두를 겨냥한 개선이 폭넓게 포함됐다.

## 시간 테이블(Temporal Tables)

기존 PostgreSQL에서 시간 범위를 추적하려면 별도의 valid_from과 valid_to 열, btree_gist 확장, 복잡한 배제 제약 조건, 그리고 행을 수동으로 분할·결합하는 애플리케이션 로직이 필요했다.
GiST는 이해하기 어렵고 제약 조건 문법이 직관적이지 않으며, 테이블 자체가 시간 데이터임을 인식하지 못한다는 한계가 있었다.

### WITHOUT OVERLAPS 제약

PostgreSQL 19는 범위형 단일 열과 WITHOUT OVERLAPS 제약으로 이 과정을 단순화한다.
과거에는 다음과 같이 작성해야 했다.

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE TABLE products (
    valid_from DATE NOT NULL,
    valid_to DATE NOT NULL,
    EXCLUDE USING gist (product_id WITH =,
        daterange(valid_from, valid_to) WITH &&)
);
```

이제는 다음과 같이 간결하게 표현한다.

```sql
CREATE TABLE products (
    product_id INT NOT NULL,
    valid_at DATERANGE NOT NULL,
    PRIMARY KEY (product_id, valid_at WITHOUT OVERLAPS)
);
```

확장 프로그램을 자동으로 관리하고, 범위형 단일 열을 사용하며, PostgreSQL이 시간 데이터를 명확히 인식한다.

### FOR PORTION OF

FOR PORTION OF 구문은 특정 시간 구간에 대한 업데이트와 삭제를 자동 처리한다.

```sql
UPDATE products
   FOR PORTION OF valid_at FROM '2025-03-01' TO '2025-09-01'
   SET price = 10.99
 WHERE product_id = 1;
```

이 명령은 기존 행을 지정 기간만큼 분할하고, 중간 부분을 새 가격으로 교체하며, 앞뒤 부분을 보존한다.
수동 분할·결합 로직 없이 3개 행이 5개 행으로 자동 정렬되어 겹침이나 간격이 발생하지 않는다.

삭제도 같은 방식으로 동작한다.

```sql
DELETE FROM products
   FOR PORTION OF valid_at FROM '2025-06-01' TO '2025-10-01'
 WHERE product_id = 2;
```

또한 시간 외래키를 지원해, 참조 대상이 자식 행의 전체 유효 기간에 걸쳐 존재하도록 보장한다.

```sql
CREATE TABLE variants (
    variant_id INT NOT NULL,
    product_id INT NOT NULL,
    valid_at DATERANGE NOT NULL,
    PRIMARY KEY (variant_id, valid_at WITHOUT OVERLAPS),
    FOREIGN KEY (product_id, PERIOD valid_at)
        REFERENCES products (product_id, PERIOD valid_at)
);
```

## 운영과 성능 개선

### REPACK CONCURRENTLY와 파티셔닝

기존에 pg_repack 확장으로만 가능했던 테이블 재구성이 코어에 통합됐다.
REPACK CONCURRENTLY는 VACUUM FULL이나 CLUSTER와 달리 잠금 없이 테이블을 재구성하므로, 프로덕션 환경에서 다운타임 없이 블로트를 제거할 수 있다.

파티셔닝에는 병합과 분할 기능이 추가됐다.

```sql
ALTER TABLE customer_orders MERGE PARTITIONS
(orders_2026_q1, orders_2026_q2) INTO orders_2026_h1;

ALTER TABLE customer_orders SPLIT PARTITION orders_2026_q3 INTO (
    PARTITION orders_2026_07 FOR VALUES FROM ('2026-07-01') TO ('2026-08-01'),
    PARTITION orders_2026_08 FOR VALUES FROM ('2026-08-01') TO ('2026-09-01')
);
```

워크로드 변화에 맞춰 파티셔닝 전략을 동적으로 조정할 수 있다.

### 논리적 복제와 Autovacuum

논리적 복제가 한층 성숙해졌다.
발행자와 구독자 간 시퀀스 값을 동기화하고, ALL SEQUENCES로 모든 시퀀스를 자동 복제하며, EXCEPT 절로 특정 테이블을 제외할 수 있다.
effective_wal_level 뷰로 실제 복제 상태를 확인할 수 있다.

Autovacuum은 병렬 처리를 지원한다.

```sql
ALTER SYSTEM SET autovacuum_max_parallel_workers = 4;

ALTER TABLE application_logs SET (
    autovacuum_vacuum_insert_score_weight = 3.0,
    autovacuum_vacuum_score_weight = 0.5
);
```

테이블별 우선순위 조정이 가능하고, pg_stat_autovacuum_scores 뷰로 결정 과정을 추적할 수 있다.

### SQL 품질과 성능

SQL 문법 편의성도 개선됐다.
GROUP BY ALL로 그룹 키를 자동 추론하고, 윈도우 함수에 IGNORE NULLS와 RESPECT NULLS를 지원한다.
COPY는 JSON 배열 내보내기, 다중 헤더 스킵, ON_ERROR SET_NULL 오류 처리를 지원하며, 관계형 모델을 유지하면서 그래프 쿼리를 추가하는 SQL/PGQ 속성 그래프도 도입됐다.

성능 측면에서는 반조인·세미조인 최적화, 상수 폴딩, Radix 정렬, COPY FROM의 SIMD 활용, 조인 전 집계 처리 등이 반영됐다.

## 한계와 주의사항

시간 테이블은 아직 애플리케이션 시간만 지원하며, 시스템 시간(transaction time)은 구현되지 않았다.
시스템 시간은 트리거로 에뮬레이션할 수 있으나, 완전한 이중 시간(bitemporal) 지원은 향후 릴리스로 예정돼 있다.
또한 시간 외래키는 NO ACTION만 지원하고 CASCADE는 지원하지 않는다.
현재 beta 단계이므로 프로덕션 적용 전 정식 릴리스 및 충분한 검증이 필요하다.

## 결론

PostgreSQL 19는 개별 혁신보다 데이터베이스가 더 많은 작업을 자동으로 처리·최적화하도록 만드는 방향을 보여준다.
시간 테이블은 감사 추적 시스템을 크게 단순화하고, REPACK CONCURRENTLY와 autovacuum 병렬화는 운영 부담을 낮춘다.
표준 준수와 직관적 문법을 통해 개발자 경험과 운영 효율을 모두 끌어올린 릴리스다.

## Reference

- [Looking Forward to Postgres 19: It's About Time (pgEdge)](https://www.pgedge.com/blog/looking-forward-to-postgres-19-its-about-time)
- [PostgreSQL 19 Features in Beta (Snowflake)](https://www.snowflake.com/en/blog/engineering/postgresql-19-features-beta/)
