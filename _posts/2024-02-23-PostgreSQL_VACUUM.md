---
layout: post
title: PostgreSQL - VACUUM
author: 'Juho'
date: 2024-02-21 09:00:00 +0900
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
1. [VACUUM이란?](#VACUUM이란)
2. [VACUUM 옵션](#VACUUM-옵션)
3. [vacuumdb](#vacuumdb)
4. [비용 기반 VACUUM Delay](#비용-기반-vacuum-delay)
5. [Routine Vacuuming](#routine-vacuuming)
6. [VACUUM Progress Reporting](#vacuum-progress-reporting)
7. [VACUUM FULL Progress Reporting](#vacuum-full-progress-reporting)

## VACUUM이란?
VACUUM을 간단히 이야기하면 데이터베이스의 가비지 컬렉터, 선택적으로 분석하는 것이다.<br/>

VACUUM은 Dead Tuple에 의해 차지된 저장 공간을 회수한다.<br/>
일반적으로 PostgreSQL 작업에서 삭제되거나 업데이트에 의해 더 이상 사용되지 않는 튜플은 물리적으로 테이블에서 제거되지 않는다.<br/>
해당 튜플들은 VACUUM이 수행될 때까지 그대로 남아 있으므로 자주 업데이트되는 테이블에 대해 주기적으로 VACUUM을 수행하는 것이 필요하다.<br/>

테이블 및 열 목록이 없는 경우, VACUUM은 현재 사용자가 VACUUM 처리 권한을 가진 현재 데이터베이스의 모든 테이블 및 materialized view를 처리한다.<br/>
목록이 있는 경우, VACUUM은 그 목록에 있는 테이블만 처리한다.<br/>

VACUUM ANALYZE는 선택한 각 테이블에 대해 VACUUM을 수행한 후 ANALYZE를 수행한다.<br>
일상적인 유지 보수 스크립트에 유용한 결합 형태입니다.<br/>

일반적인 VACUUM (FULL 없음)은 단순히 공간을 회수하여 재사용할 수 있도록 한다.<br/>
이 명령 형태는 exclusive lock을 얻지 않으므로 테이블의 일반적인 읽기 및 쓰기와 병렬로 작동할 수 있다.<br/>
그러나 추가 공간은 대부분의 경우 운영 체제에 반환되지 않고 단지 동일한 테이블 내에서 재사용할 수 있도록 유지된다.<br/>
또한 인덱스를 처리하기 위해 여러 CPU를 활용할 수 있는데, 이것을 parallel VACUUM라고 한다.<br/>
이 기능을 비활성화하려면 PARALLEL 옵션을 사용하고 parallel workers를 0으로 지정하면 된다.<br/>
VACUUM FULL은 테이블의 전체 내용을 새로운 디스크 파일로 다시 작성하며 추가 공간 없이 사용되지 않는 공간을 운영 체제에 반환할 수 있도록 한다.<br/>
VACUUM FULL이 훨씬 느리며 처리되는 동안 각 테이블에 ACCESS EXCLUSIVE lock이 필요하다.<br/>

옵션 목록이 괄호로 둘러싸여 있을 때는 옵션을 어떤 순서로 작성해도 되지만 괄호가 없는 경우에는 옵션은 정확히 표시된 순서대로 지정되어야 한다.<br/>
```sql
VACUUM [ ( option [, ...] ) ] [ table_name [ ( column_name [, ...] ) ] ]
```
테이블을 VACUUM 처리하려면 일반적으로 테이블 소유자 또는 슈퍼유저여야 한다.<br/>
그러나 데이터베이스 소유자는 공유 카탈로그를 제외한 데이터베이스 내의 모든 테이블을 VACUUM 처리할 수 있습니다.<br/>
공유 카탈로그에 대한 제한은 진정한 데이터베이스 전체 VACUUM을 슈퍼유저만 수행할 수 있음을 의미한다.<br/>
VACUUM은 호출 사용자가 VACUUM 처리 권한이 없는 테이블을 건너뜁니다.<br/>

VACUUM은 트랜잭션 블록 내에서 실행할 수 없다.<br/>

GIN 인덱스가 있는 테이블의 경우 VACUUM(어떤 형태이든)은 대기 중인 인덱스 삽입도 완료한다.<br/>
이는 대기 중인 인덱스 항목을 기본 GIN 인덱스 구조의 적절한 위치로 이동시킴으로써 수행된다.<br/>

PostgreSQL은 모든 데이터베이스가 정기적으로 VACUUM 처리되어야 하며, Dead Tuple을 제거하기 위해 추천한다.<br/>
PostgreSQL에는 Routine VACUUM 유지 관리를 자동화할 수 있는 "autovacuum" 기능이 포함되어 있다.<br/>

VACUUM은 I/O 트래픽이 상당히 증가하여 다른 활성 세션에 대한 성능이 저하될 수 있다.<br/>
따라서 비용 기반 VACUUM 지연 기능을 사용하는 것이 때로는 바람직할 수 있다.<br/>
parallel vacuum의 경우 각 워커는 해당 워커가 수행한 작업에 비례하여 대기한다.<br/>

FULL 옵션이 없는 각 백엔드는 pg_stat_progress_vacuum 뷰에서 진행 상황을 보고한다.<br/>
FULL 옵션을 사용하는 백엔드는 대신 pg_stat_progress_cluster 뷰에서 진행 상황을 보고한다.<br/>

SELECT, INSERT, UPDATE, DELETE와 같은 명령은 정상적으로 작동하지만, VACUUM 작업 중에는 ALTER TABLE과 같은 명령으로 테이블의 정의를 수정할 수 없다.<br/>

일반적으로 관리자는 표준 VACUUM을 사용하고 VACUUM FULL을 피해야 한다.<br/>

## VACUUM 옵션
1) FULL<br/>
> Full은 VACUUM이 더 많은 공간을 회수할 수 있지만 훨씬 오랜 시간이 걸리고 테이블을 EXCLUSIVE lock 처리한다.<br/>
 이 방법은 작업이 완료될 때까지 이전 복사본을 해제하지 않기 때문에 추가 디스크 공간이 필요하다.<br/>
 FULL 옵션은 일반적으로 사용을 권장하지 않지만 특수한 경우에 유용할 수 있다.<br>
 보통은 테이블 내부에서 상당한 양의 공간을 회수해야 할 때만 사용해야 한다.<br/>
 예를 들어, 테이블의 대부분의 행을 삭제하거나 업데이트한 경우 테이블이 실제로 작은 디스크 공간을 차지하도록 하고 더 빠른 테이블 스캔을 허용하기 위해 테이블을 물리적으로 줄이고 싶을 때 VACUUM FULL은 일반 VACUUM보다 테이블을 보통 더 많이 줄인다.<br/>


2) FREEZE<br/>
> FREEZE를 지정하는 것은 VACUUM_freeze_min_age 및 VACUUM_freeze_table_age 매개변수를 모두 0으로 설정하여 VACUUM을 수행하는 것과 동일하다.<br/>
 테이블이 다시 작성될 때 항상 공격적인 freezing이 수행되므로, 이 옵션은 FULL이 지정된 경우 중복된다.<br/>

3) VERBOSE<br/>
> 각 테이블에 대한 자세한 VACUUM 활동 보고서를 출력한다.<br/>
  VACUUM은 진행 메시지를 출력하여 현재 처리 중인 테이블을 나타낸다.<br/>
  또한 테이블에 대한 다양한 통계 정보가 출력된다.<br/>

4) ANALYZE<br/>
> 쿼리를 실행하는 가장 효율적인 방법을 결정하기 위해 플래너가 사용하는 통계를 업데이트한다.<br/>

5) DISABLE_PAGE_SKIPPING<br/>
> 보통 VACUUM은  visibility map을 기반으로 페이지를 건너뛴다.<br/>
 모든 튜플이 freeze된 것으로 알려진 페이지는 항상 건너뛸 수 있으며, 모든 트랜잭션에게 모든 튜플이 보이는 것으로 알려진 페이지는 공격적인 VACUUM을 수행할 때를 제외하고는 건너뛸 수 있다.<br/>
 게다가, 공격적인 VACUUM을 수행하지 않을 때는 다른 세션에서 해당 페이지를 사용하는 것을 기다리지 않기 위해 일부 페이지를 건너뛸 수 있다.<br/>
 이 옵션은 모든 페이지 건너뛰기 동작을 비활성화하며, visibility map 내용에 의심이 있는 경우에만 사용되어야 합니다.<br/>
 이는 데이터베이스 손상을 일으키는 하드웨어 또는 소프트웨어 문제가 있을 경우에만 발생해야 한다.

6) SKIP_LOCKED<br/>
> VACUUM이 관계에 작업을 시작할 때 충돌하는 lock이 해제될 때까지 기다리지 않도록 지정한다.<br/>
 즉, relation을 즉시 lock할 수 없고 대기할 필요가 있는 경우 해당 relation은 건너뛰어진다.<br/>
 이 옵션을 사용하더라도 VACUUM이 여전히 relation의 인덱스를 열 때 차단될 수 있다.<br/>
 또한 VACUUM ANALYZE가 파티션, 테이블 상속 자식 및 일부 유형의 외래 테이블에서 샘플 행을 가져올 때 여전히 차단될 수 있다.
 또한, VACUUM은 일반적으로 지정된 파티션 테이블의 모든 파티션을 처리하지만, 이 옵션을 사용하면 파티션 테이블에 충돌하는 lock이 있는 경우 모든 파티션을 건너뛰게 된다.<br/>

7) INDEX_CLEANUP<br/>
> 일반적으로 테이블에 매우 적은 수의 Dead Tuple이 있을 때 VACUUM은 인덱스 VACUUM 처리를 건너뛴다.<br/>
 이런 경우 모든 테이블의 인덱스를 처리하는 비용이 Dead Index Tuple을 제거하는 이득을 크게 초과할 것으로 예상된다.<br/>
 이 옵션은 Dead Tuple이 하나 이상 있을 때 VACUUM이 인덱스를 처리하도록 강제할 수 있다.<br/>
 기본값은 AUTO이며, 적절할 때 VACUUM이 인덱스 진공 처리를 건너뛰도록 허용한다.<br/>
 INDEX_CLEANUP가 ON으로 설정된 경우 VACUUM은 보수적으로 모든 인덱스에서 Dead Tuple을 제거합니다.<br/>
 이는 PostgreSQL의 이전 릴리스에서 표준 동작이었던 것과의 역호환성을 위해 유용할 수 있다. <br/>
 INDEX_CLEANUP는 또한 OFF로 설정하여 VACUUM이 항상 인덱스 VACUUM 처리를 건너뛰도록 할 수 있다.<br/>
 이는 트랜잭션 ID wraparound를 피하기 위해 VACUUM을 가능한 빠르게 실행해야 할 때 유용할 수 있다.<br/>
 그러나 트랜잭션 ID wraparound  실패 방지 메커니즘인 vacuum_failsafe_age를 제어하는 것이 보통 자동으로 트리거되어 트랜잭션 ID wraparound 실패를 피하기 위해 선호되어야 한다.<br/>
 INDEX_CLEANUP이 정기적으로 수행되지 않으면 성능이 저하될 수 있다.<br/>
 테이블이 수정될수록 인덱스는 Dead Tuple을 축적하고 테이블 자체는 INDEX_CLEANUP이 완료될 때까지 제거할 수 없는  Dead Line 포인터를 축적한다.<br/>
 이 옵션은 인덱스가 없는 테이블에는 영향을 미치지 않으며, FULL 옵션을 사용하는 경우 무시된다.<br/>
 또한 트랜잭션 ID wraparound 실패 방지 메커니즘에는 영향을 미치지 않는다.<br/>
 트리거되면 INDEX_CLEANUP이 ON으로 설정되어 있더라도 인덱스 VACUUM 처리를 건너뛴다.<br/>

8) PROCESS_MAIN<br/>
> VACUUM이 주요 relation을 처리하도록 시도해야 함을 지정한다.<br/>
 이것은 일반적으로 원하는 동작이며 기본값이다.<br/>
 이 옵션을 false로 설정하는 것은 relation에 해당하는 TOAST 테이블을 VACUUM 처리해야 할 때 유용할 수 있다.<br/>

9) PROCESS_TOAST<br/>
> VACUUM이 각 relation에 대한 해당 TOAST 테이블을 처리하도록 시도해야 함을 지정한다.<br/>
 일반적으로 원하는 동작이며 기본값이다.<br/>
 이 옵션을 false로 설정하는 것은 주요 relation만 VACUUM 처리해야 할 때 유용할 수 있다.<br/>
 이 옵션은 FULL 옵션을 사용할 때 필요하다.<br/>

10) TRUNCATE<br/>
> VACUUM이 테이블 끝에 있는 빈 페이지를 자르고, 자른 페이지에 해당하는 디스크 공간을 운영 체제에 반환하도록 시도해야 함을 지정한다.<br/>
 이것은 보통 원하는 동작이며, 특별한 경우가 아닌 이상 기본값이다.<br/>
 vacuum_truncate 옵션이 테이블에 대해 false로 설정되지 않은 한 기본값으로 설정된다.<br/>
 이 옵션을 false로 설정하면 단축이 필요한 테이블에서 ACCESS EXCLUSIVE lock을 피하기 위해 유용할 수 있다.<br/>
 이 옵션은 FULL 옵션을 사용하는 경우 무시된다.<br/>

11) PARALLEL<br/>
> VACUUM의 index vacuum 및 index cleanup 단계를 정수 백그라운드 워커를 사용하여 병렬로 수행한다. <br/>
 연산을 수행하는 데 사용되는 워커의 수는 파티션 당 하나 이상의 인덱스를 지원하는 relation의 인덱스 수에 해당하며, 이 수는 PARALLEL 옵션으로 지정된 워커 수에 의해 제한된다.<br/>
 이것은 더 나아가 max_parallel_maintenance_workers로 제한된다. <br/>
 인덱스의 크기가 min_parallel_index_scan_size보다 큰 경우에만 인덱스가 parallel vacuum에 참여할 수 있다.<br/>
 정수로 지정된 병렬 워커의 수가 실행 중에 사용될 것을 보장하지 않는다.<br/>
 지정된 워커보다 적은 워커로 VACUUM이 실행되거나 아예 워커 없이 실행될 수 있다.<br/>
 한 번에 하나의 워커만 인덱스당 사용될 수 있습니다.<br/>
 따라서 테이블에 적어도 2개의 인덱스가 있는 경우에만 병렬 워커가 시작된다.<br/>
 VACUUM용 워커는 각 단계의 시작 전에 시작되고 단계가 끝나면 종료된다.<br/>
 이러한 동작은 향후 릴리스에서 변경될 수 있다.
 이 옵션은 FULL 옵션과 함께 사용할 수 없다.
 PARALLEL 옵션은 VACUUM 목적으로만 사용됩니다.<br/>
 이 옵션이 ANALYZE 옵션과 함께 지정된 경우 ANALYZE에는 영향을 미치지 않는다.<br/>

12) SKIP_DATABASE_STATS<br/>
> VACUUM이 데이터베이스 전체의 가장 오래된 freeze 되지 않은 XID에 대한 통계 업데이트를 건너뛰어야 함을 지정한다.<br/>
 보통 VACUUM은 명령의 끝에서 이러한 통계를 한 번 업데이트한다.<br/>
 그러나 매우 많은 테이블이 있는 데이터베이스의 경우 이 작업은 시간이 오래 걸릴 수 있으며, VACUUM 처리된 테이블 중 가장 오래된 오래된 freeze 되지 않은 XID를 포함하지 않는 경우 아무 것도 달성하지 못할 것이다.<br/>
 게다가, 병렬로 여러 VACUUM 명령이 발행되는 경우에는 한 번에 한 명령만 데이터베이스 전체 통계를 업데이트할 수 있다.<br/>
 따라서 응용 프로그램이 많은 수의 VACUUM 명령을 실행하려고 할 때, 마지막 명령을 제외한 모든 명령에 이 옵션을 설정하는 것이 유용할 수 있다.<br/>
 또는 모든 명령에 이 옵션을 설정하고 VACUUM (ONLY_DATABASE_STATS)를 별도로 실행할 수 있다.<br/>

13) ONLY_DATABASE_STATS<br/>
> VACUUM이 아무 작업도 수행하지 않고 오직 데이터베이스 전체의 가장 오래된 freeze 되지 않은 XID에 대한 통계만 업데이트해야 함을 지정한다.<br/>
 이 옵션이 지정된 경우, table_and_columns 목록은 비어 있어야 하며, VERBOSE를 제외한 다른 옵션은 활성화되지 않아야 한다.<br/>

14) BUFFER_USAGE_LIMIT<br/>
> VACUUM의  Buffer Access Strategy ring buffer 크기를 지정한다.<br/>
 이 크기는 이 전략의 일부로 재사용될 공유 버퍼의 수를 계산하는 데 사용된다.<br/>
  0은 버퍼 액세스 전략의 사용을 비활성화한다.<br/>
  ANALYZE도 지정된 경우, BUFFER_USAGE_LIMIT 값은 VACUUM 및 ANALYZE 단계 모두에 사용된다.<br/>
  이 옵션은 ANALYZE가 지정된 경우를 제외하고 FULL 옵션과 함께 사용할 수 없다.<br/>
  이 옵션이 지정되지 않은 경우, VACUUM은 vacuum_buffer_usage_limit에서 값을 사용한다.<br/>
  더 높은 설정은 VACUUM을 더 빠르게 실행할 수 있게 할 수 있지만, 너무 큰 설정을 가지면 유용한 다른 페이지들이 공유 버퍼에서 제거될 수 있다.<br/>
  최소값은 128 kB이고 최대값은 16 GB다.<br/>

## vacuumdb
shell command상에서 vacuumdb 명령으로 VACUUM을 사용할 수 있다.<br/>
```
vacuumdb -h localhost -p 5432 -U postgres -d db_name
```
VACUUM을 정리하고 분석하는 것과 실질적인 차이는 없다.<br/>

vacuumdb는 PostgreSQL 서버에 여러 번 연결하여 각각의 연결마다 비밀번호를 요청한다.<br/>
이러한 경우에는 ~/.pgpass 파일이 편리하다.<br/>

## 비용 기반 VACUUM Delay
VACUUM 및 ANALYZE 명령을 실행하는 동안 시스템은 수행되는 다양한 I/O 작업의 예상 비용을 추적하는 내부 카운터를 유지한다.<br/>
누적된 비용이 vacuum_cost_limit으로 지정된 한도에 도달하면, 해당 작업을 수행하는 프로세스는 vacuum_cost_delay로 지정된 시간 동안 Sleep 상태가 된다.<br/>
그런 다음 카운터를 재설정하고 실행을 계속합니다.<br/>

이 기능의 목적은 관리자가 이러한 명령이 동시 데이터베이스 활동에 미치는 I/O 영향을 줄일 수 있도록 하는 것이다.<br/>
VACUUM 및 ANALYZE와 같은 유지 관리 명령이 빨리 완료되는 것은 중요하지 않은 경우가 많지만, 일반적으로 이러한 명령이 시스템이 다른 데이터베이스 작업을 수행하는 능력에 중대한 방해를 일으키지 않는 것이 매우 중요하다.<br/>
비용 기반 VACUUM Delay는 관리자가 이를 달성할 수 있는 방법을 제공한다.<br/>

이 기능은 수동으로 실행된 VACUUM 명령에 대해 기본적으로 비활성화이다.<br/>
이를 활성화하려면 vacuum_cost_delay 변수를 0이 아닌 값으로 설정해야 한다.<br/>
1) vacuum_cost_delay (floating point)<br/>
> cost limit를 초과했을 때 프로세스가 slepp될 시간의 양입니다.<br/>
 이 값이 단위 없이 지정된 경우 밀리초로 취급된다.<br/>
 기본값은 0으로, 이는 비용 기반 VACUUM Delay 기능을 비활성화한다.<br/>
 양수 값은 비용 기반 VACUUM을 활성한다.<br/>
 비용 기반 VACUUM을 사용할 때, 일반적으로 적절한 vacuum_cost_delay값으로 꽤 작은 값으로 1밀리초 미만으로 설정한다.<br/> 
 vacuum_cost_delay를 소수 밀리초 값으로 설정할 수 있지만, 이러한 딜레이는 오래된 플랫폼에서 정확하게 측정되지 않을 수 있다.<br/>
 이러한 플랫폼에서는 1밀리초보다 높은 VACUUM의 제어된 리소스 소비를 증가시키려면 다른 vacuum cost 매개변수를 변경해야 한다.<br/>
 그럼에도 불구하고, vacuum_cost_delay를 플랫폼이 일관되게 측정할 수 있는 최소값으로 유지해야 한다.<br/>
 큰 딜레이는 도움이 되지 않는다.<br/>

2) vacuum_cost_page_hit (integer)<br/>
> 공유 버퍼 캐시에서 찾은 버퍼를 VACUUM하는 데 예상되는 비용이다.<br/>
 버퍼 풀을 잠그는 비용, 공유 해시 테이블을 조회하는 비용 및 페이지 내용을 스캔하는 비용을 나타낸다.<br/>
 기본값은 1이다.<br/>

3) vacuum_cost_page_miss (integer)<br/>
> 디스크에서 읽어야 하는 버퍼를 VACUUM하는 데 예상되는 비용이다.<br/>
 버퍼 풀을 잠그는 비용, 공유 해시 테이블을 조회하는 비용, 디스크에서 원하는 블록을 읽어오는 비용 및 해당 내용을 스캔하는 데 필요한 노력을 나타낸다.<br/>
 기본값은 2다.<br/>

4) vacuum_cost_page_dirty (integer)<br/>
> VACUUM이 이전에 깨끗한 상태였던 블록을 수정할 때 부과되는 예상 비용이다.<br/>
 이는 변경된 블록을 다시 디스크로 플러시하는 데 필요한 추가 I/O를 나타냅니다.<br/>
 기본값은 20이다.<br/>

5) vacuum_cost_limit (integer)<br/>
> VACUUM 프로세스가 Sleep되도록 만드는 누적 비용이다.<br/>
 기본값은 200입니다.<br/>

<br>
일부 작업은 중요한 잠금을 유지하고 있으므로 가능한 한 빨리 완료되어야 한다.<br/>
이러한 작업 중에는 비용 기반 VACUUM Delay가 발생하지 않는다.<br/>
따라서 비용이 지정된 한도를 훨씬 초과하여 누적될 수 있다.<br/>
이러한 경우에 쓸데없이 긴 지연을 피하기 위해 실제 지연은 <br/>
`vacuum_cost_delay * accumulated_balance / vacuum_cost_limit`로 계산되며, 최대 `vacuum_cost_delay * 4`로 
제한된다.<br/>

## Routine Vacuuming
<!--https://www.postgresql.org/docs/current/routine-vacuuming.html#ROUTINE-VACUUMING-->
PostgreSQL 데이터베이스는 주기적인 유지 관리 작업인 VACUUM이 필요하다.<br/>
autovacuum daemon에 의해 VACUUM이 수행되도록 하는 것만으로 충분하다.<br/>
상황에 최적의 결과를 얻으려면 autovacuum daemon 파라미터를 조정해야 할 수 있다.<br/>
일부 데이터베이스 관리자는 daemon의 활동을 수동으로 관리하는 VACUUM 명령으로 보충하거나 대체할 수 있으며, 일반적으로 cron이나 Task Scheduler 스크립트에 의해 일정에 따라 실행된다.<br/>
수동으로 관리되는 VACUUM을 올바르게 설정하려면 몇 개의 문제를 이해하는 것이 중요하다.<br/>
autovacuum daemon에 의존하는 관리자라도 문제를 이해하면 autovacuum daemon을 이해하고 조정하는 데 도움을 받을 수 있다.<br/>

PostgreSQL의 VACUUM 명령은 여러 이유로 정기적으로 각 테이블을 처리해야 한다.<br/>
<br/>

1) 업데이트되거나 삭제된 행이 차지하는 디스크 공간을 회수하거나 재사용하기 위해.<br/>
> PostgreSQL에서 row의 UPDATE 또는 DELETE는 행의 이전 버전을 즉시 제거하지 않는다.<br/>
 이 접근 방식은 다중 버전 동시성 제어(MVCC)의 이점을 얻기 위해 필요하다.<br/>
 Row 버전은 여전히 다른 트랜잭션에서 잠재적으로 볼 수 있는 동안 삭제되어서는 안 된다.<br/> 
 그러나 결국 오래된 또는 삭제된 row 버전은 더 이상 어떤 트랜잭션에게도 관심이 없어진다.<br/>
 그런 후에는 새로운 row가 해당 공간을 재사용하기 위해 회수해야만 하며, 디스크 공간 요구 사항이 무한정으로 증가하는 것을 방지하기 위해 VACUUM을 실행한다.<br/>

> 표준 VACUUM 형태는 테이블과 인덱스에서 dead row 버전을 제거하고 향후 재사용을 위해 사용 가능한 공간을 표시한다.<br/>그러나 특별한 경우를 제외하고는 이 공간을 운영 체제로 반환하지 않는다.<br/>
 해당 특별한 경우는 테이블 끝의 하나 이상의 페이지가 완전히 비어 있고 exclusive table lock을 쉽게 얻을 수 있는 경우다.<br/>
 반면에 VACUUM FULL은 테이블을 활발하게 압축하여 dead space가 없는 테이블 파일의 완전히 새 버전을 작성한다.<br/> 이는 테이블의 크기를 최소화하지만 오랜 시간이 소요될 수 있다.<br/>
 또한 작업이 완료될 때까지 새로운 테이블 복사본에 대한 추가 디스크 공간이 필요하다.<br/>

> 일반적인 routine vacuuming 작업의 목표는 VACUUM FULL이 필요하지 않도록 충분히 자주 표준 VACUUM을 수행하는 것이다.<br/>
 autovacuum daemon은 이와 같은 방식으로 작동하려고 시도하며 사실상 VACUUM FULL을 실행하지 않는다.<br/>
 이 접근 방식에서의 아이디어는 테이블을 최소 크기로 유지하는 것이 아니라 디스크 공간의 일정한 상태를 유지하는 것이다.<br/>
 즉, 각 테이블은 최소 크기에 해당하는 공간과 VACUUM 작업 간 사용된 공간을 합한만큼의 공간을 차지한다.<br/>
 VACUUM FULL을 사용하여 테이블을 최소 크기로 줄이고 디스크 공간을 운영 체제로 반환할 수 있지만, 향후 테이블이 다시 커질 경우 이 작업의 의미가 크지 않는다.<br/>
 따라서 빈번한 표준 VACUUM 작업이 자주 업데이트되는 테이블을 유지하는 데 드물게 VACUUM FULL 작업보다 나은 접근 방식이다.<br/>

> 예를 들어 부하가 낮은 밤에 모든 작업을 수행하는 등 직접 VACUUM 작업을 예약하는 것을 선호할때 고정된 일정에 따라 VACUUM 작업을 수행하는 것의 어려움은 테이블에 예상치 못한 업데이트 활동의 급증이 발생할 경우, 해당 테이블이 공간을 회수하기 위해 VACUUM FULL이 실제로 필요할 수 있다는 점이다.<br/>
 autovacuum daemon을 사용하면 이 문제가 완화된다.<br/>
 왜냐하면 daemon은 업데이트 활동에 대응하여 동적으로 VACUUM 작업을 예약하기 때문이다.<br/>
 매우 예측 가능한 작업 부하가 있는 경우를 제외하고는 daemon을 완전히 비활성화하는 것은 현명하지 않다.<br/>
 한 가지 가능한 타협안은 daemon의 매개변수를 설정하여 이상하게 높은 업데이트 활동에만 반응하도록 하는 것이다.<br/> 따라서 상황이 무리하지 않도록 유지하는 동시에 예약된 VACUUM이 일반적인 부하일 때 대부분의 작업을 수행할 것으로 기대된다.<br/>

> autovacuum을 사용하지 않는 경우, 전체 데이터베이스에 대한 일일 VACUUM을 낮은 사용량 기간에 한 번 예약하고 필요한 경우 업데이트가 많이 이루어지는 테이블에 대해 더 자주 VACUUM 작업을 보충하는 것이 일반적인 접근 방식이다.<br/>
 클러스터에 여러 데이터베이스가 있는 경우 각각의 데이터베이스에 대해 VACUUM을 실행하는 것을 잊으면 안된다.<br/>

> 대량 업데이트 또는 삭제 활동으로 인해 테이블에 많은 수의 dead row 버전이 포함되어있는 경우 일반 VACUUM은 만족스럽지 않을 수 있다.<br/>
 이러한 테이블이 있고 해당 테이블이 차지하는 과도한 디스크 공간을 회수해야하는 경우 VACUUM FULL 또는 대안으로 CLUSTER 또는 ALTER TABLE의 테이블 재작성 변형 중 하나를 사용해야한다.<br/>
 이러한 명령은 테이블의 새로운 전체 복사본을 다시 작성하고 새 인덱스를 구축한다.<br/>
 이러한 옵션은 모두 ACCESS EXCLUSIVE LOCK이 필요하다.<br/>
 이전 복사본이 새로운 복사본이 완료될 때까지 해제되지 않으므로 이들은 임시로 테이블 크기와 거의 같은 추가 디스크 공간을 사용한다.<br/>

> 주기적으로 전체 내용이 삭제되는 테이블이 있다면, DELETE 후에 VACUUM을 사용하는 대신 TRUNCATE를 고려하는것이 좋다.<br/>
TRUNCATE는 즉시 테이블의 전체 내용을 제거하며, 이제 사용되지 않는 디스크 공간을 회수하기 위해 후속으로 VACUUM 또는 VACUUM FULL이 필요하지 않지만 단점은 엄격한 MVCC 의미론이 위배된다는 것이다.<br/>

2) PostgreSQL 쿼리 플래너에서 사용되는 데이터 통계를 업데이트하기 위해.<br/>
> PostgreSQL의 쿼리 플래너는 쿼리에 대한 좋은 계획을 생성하기 위해 테이블 내용에 대한 통계 정보를 활용한다.<br/>
 이러한 통계 정보는 ANALYZE 명령을 사용하여 수집되며, 이는 VACUUM의 선택적 단계로서도 호출될 수 있다.<br/>
 충분히 정확한 통계 정보를 보유하는 것이 중요하다.<br/>
그렇지 않으면 부적절한 계획 선택으로 인해 데이터베이스 성능이 저하될 수 있다.<br/>

> autovacuum daemon이 활성화되어 있다면 테이블 내용이 충분히 변경될 때마다 자동으로 ANALYZE 명령을 실행합니다.<br/> 
 그러나 특히 "흥미로운" column의 통계에 영향을 미치지 않을 것으로 알려진 경우에는 수동으로 예약된 ANALYZE 작업을 의지할 수 있다.<br/>
 daemon은 삽입 또는 업데이트된 row 수의 함수로서 ANALYZE를 엄격하게 예약하며, 그것이 의미 있는 통계적 변화로 이어질 것인지에 대한 지식이 없다.<br/>

> 파티션 및 상속 자식에서 변경된 row는 부모 테이블에서 ANALYZE를 트리거하지 않는다.<br/>
 부모 테이블이 비어 있거나 드물게 변경된 경우, autovacuum에 의해 처리되지 않을 수 있으며 상속 트리 전체의 통계가 수집되지 않을 수 있다.<br/>
 통계를 최신 상태로 유지하려면 부모 테이블에서 ANALYZE를 수동으로 실행해야 한다.<br/>

> 공간 회수를 위한 VACUUM 작업과 마찬가지로, 통계의 빈번한 업데이트는 드물게 업데이트되는 테이블보다 빈번하게 업데이트되는 테이블에 더 유용하다.<br/>
 그러나 많이 업데이트되는 테이블의 경우라도 데이터의 통계적 분포가 크게 변하지 않는다면 통계 업데이트가 필요하지 않을 수 있다.<br/>
 간단한 경험에 의한 규칙은 테이블의 col의 최솟값과 최댓값이 얼마나 변하는지를 생각하는 것이다.<br/>
 예를 들어, row 업데이트 시간을 포함하는 타임스탬프 열은 행이 추가되고 업데이트됨에 따라 지속적으로 증가하는 최대값을 갖게 된다.<br/>
 이러한 row는 웹사이트에서 접근된 페이지의 URL을 포함하는 열과 달리 상대적으로 느리게 통계값이 변경될 것이다.<br/> URL 열은 변경이 같이 발생할 수 있지만, 값의 통계적 분포가 비교적 느리게 변경될 것으로 예상된다.<br/>

> 특정 테이블 또는 테이블의 특정 column에 대해 ANALYZE를 실행하는 것이 가능하므로, 응용 프로그램이 필요한 경우 일부 통계 정보를 더 자주 업데이트할 수 있다.<br/>
 그러나 실제로는 대개 전체 데이터베이스를 ANALYZE하는 것이 가장 좋다.<br/>
 ANALYZE는 모든 row을 읽는 대신 테이블의 통계적으로 무작위로 표본을 추출한다.<br/>
 따라서 데이터베이스 전체를 ANALYZE하는 것은 빠른 작업이다.<br/>

> ANALYZE 빈도를 column마다 조정하는 것은 생산적이지 않을 수 있지만, ANALYZE에 의해 수집된 통계의 상세 수준을 열마다 조정하는 것은 가치가 있을 수 있다.<br/>
 WHERE 절에서 빈번하게 사용되고 데이터 분포가 매우 불규칙한 열은 다른 column보다 더 세밀한 데이터 히스토그램을 요구할 수 있다.<br/>
 ALTER TABLE SET STATISTICS를 참조하거나 default_statistics_target 구성 매개변수를 사용하여 데이터베이스 전체의 기본 설정을 변경할 수 있다.<br/>
 또한, 기본적으로 함수의 선택성에 대한 제한된 정보만 사용할 수 있다.<br/>
 그러나 함수 호출을 사용하는 통계 개체나 표현식 인덱스를 생성하면 함수에 대한 유용한 통계가 수집되며, 이는 표현식 인덱스를 사용하는 쿼리 계획을 크게 개선할 수 있다.<br/>

> autovacuum daemon은 foreign 테이블에 대해 ANALYZE 명령을 실행하지 않는다.<br/>
 foreign 테이블에서 얼마나 자주 이것이 유용할지를 결정할 방법이 없기 때문이다.<br/>
 쿼리가 적절한 계획을 위해 foreign 테이블에 대한 통계를 필요로 한다면, 적절한 일정으로 해당 테이블에 대해 수동으로 관리되는 ANALYZE 명령을 실행하는 것이 좋다.<br/>

> autovacuum daemon은 파티션된 테이블에 대해 ANALYZE 명령을 실행하지 않습니다.<br/>
 상속 부모는 부모 자체가 변경될 때만 ANALYZE한다.<br/>
 자식 테이블의 변경 사항은 부모 테이블의 autoanalyze를 트리거하지 않는다.<br/>
 쿼리가 적절한 계획을 위해 부모 테이블에 대한 통계를 필요로 한다면, 주기적으로 해당 테이블에 대해 수동 ANALYZE를 실행하여 통계를 최신 상태로 유지해야 한다.<br/>

3) 인덱스 전용 스캔을 가속화하는 visibility map을 업데이트하기 위해.<br/>
> VACUUM은 각 테이블에 대한 visibility map을 유지하여 모든 활성 트랜잭션(및 모든 향후 트랜잭션, 페이지가 다시 
 수정되기 전까지)에서 알려진 visible이 있는 튜플만 포함된 페이지를 추적한다.<br/>
 이에는 두 가지 목적이 있다.<br/>
 첫째, VACUUM 자체는 다음 실행에서 정리할 것이 없기 때문에 visible하지 않는 페이지를 건너뛸 수 있다.<br/>
 둘째, PostgreSQL은 일부 쿼리를 인덱스만 사용하여 응답할 수 있도록 허용한다.<br/>
 PostgreSQL 인덱스에는 튜플 visibility 정보가 포함되어 있지 않기 때문에 일반적인 인덱스 스캔은 각 일치하는 인덱스 항목에 대해 힙 튜플을 가져와야 한다.<br/>
 현재 트랜잭션이 해당 튜플을 볼 수 있는지 여부를 확인합니다.<br/>
 반면에 인덱스 전용 스캔(index-only scan)은 visibility map을 먼저 확인한다.<br/>
 페이지의 모든 튜플이 visible하다고 알려진 경우, 힙 검색을 건너뛴다.<br/>
 이는 visibility map이 디스크 액세스를 방지할 수 있는 대규모 데이터 세트에서 가장 유용하다.<br/>
 visibility map은 힙보다 훨씬 작기 때문에 힙이 매우 큰 경우에도 쉽게 캐시될 수 있다.<br/>

4) 트랜잭션 ID wraparound 또는 multixact ID wraparound로 인한 매우 오래된 데이터의 손실로부터 보호하기 위해.<br/>
> PostgreSQL의 MVCC 트랜잭션 의미론은 트랜잭션 ID(XID) 번호를 비교할 수 있는 데 의존한다.<br/>
 현재 트랜잭션의 XID보다 큰 삽입 XID를 가진 row 버전은 "미래"에 있으며 현재 트랜잭션에는 보이지 않아야 한다.<br/> 그러나 트랜잭션 ID는 제한된 크기(32비트)를 갖기 때문에 오랜 시간 동안 실행되는 클러스터(40억개 이상의 트랜잭션)는 트랜잭션 ID 랩어라운드에 영향을 받는다.<br/>
 XID 카운터가 0으로 다시 랩어라운드되면, 과거에 있던 트랜잭션이 갑자기 미래에 있는 것처럼 보이게 된다.<br/>
 이는 해당 트랜잭션의 출력이 보이지 않게 됨을 의미하며, 즉 치명적인 데이터 손실을 의미한다.<br/>
 사실 데이터는 여전히 존재하지만, 접근할 수 없는 것 이다.<br/>
 이러한 문제를 피하기 위해 모든 데이터베이스의 모든 테이블을 최소한 20억 개의 트랜잭션마다 적어도 한 번씩 VACUUM해야 한다.<br/>

> 주기적인 VACUUM 작업이 이 문제를 해결하는 이유는 VACUUM이 row를 frozen으로 표시하기 때문이다.<br/>
 이는 해당 row가 충분히 과거에 커밋된 트랜잭션에 의해 삽입되어 해당 삽입 트랜잭션의 효과가 현재 및 향후 모든 트랜잭션에 대해 보이는 것이 확실함을 나타낸다.<br/>
 일반적인 XID는 모듈로 2^32 산술을 사용하여 비교된다.<br/>
 이것은 일반적인 XID마다 "과거"인 20억 개의 XID와 "최신"인 20억 개의 XID가 있음을 의미한다.<br/>
 다른 말로 하면, 일반적인 XID 공간은 끝점이 없는 원형으로 구성된다.<br/>
 따라서 한 특정 일반적인 XID로 행 버전이 생성된 후, 해당 행 버전은 어떤 일반적인 XID를 사용하든 다음 20억 개의 트랜잭션 동안 "과거"로 표시될 것이다.<br/>
 row 버전이 20억 개 이상의 트랜잭션 후에도 존재하는 경우, 갑자기 미래에 있다고 나타날 것이다.<br/>
 이를 방지하기 위해 PostgreSQL은 특별한 XID인 FrozenTransactionId를 예약한다.<br/>
 이 XID는 일반적인 XID 비교 규칙을 따르지 않으며 항상 모든 일반적인 XID보다 오래된 것으로 간주된다.<br/>
 frozen row 버전은 삽입 XID가 FrozenTransactionId로 간주되도록 처리되므로, wraparound 문제와 관계없이 모든 일반적인 트랜잭션에 대해 "과거"로 표시되며, 따라서 해당 row 버전은 삭제될 때까지 유효하다.<br/>

> PostgreSQL 9.4 이전 버전에서는 실제로 row의 삽입 XID를 FrozenTransactionId로 대체하여 frozen을 구현했다.<br/>
 이 FrozenTransactionId는 행의 xmin 시스템 열에 표시되었다.<br/>
 그러나 더 최신 버전에서는 행의 원래 xmin을 가능한 법의적 사용을 위해 보존하기 위해 플래그 비트를 설정한다.<br/> 그러나 xmin이 FrozenTransactionId(2)와 같은 행은 여전히 이전 9.4 버전에서 pg_upgrade된 데이터베이스에서 찾을 수 있다.<br> 
 또한, 시스템 카탈로그에는 BootstrapTransactionId(1)와 같은 xmin을 가진 행이 포함될 수 있으며, 이는 initdb의 첫 번째 단계에서 삽입되었음을 나타낸다.<br/>
 FrozenTransactionId와 마찬가지로, 이 특별한 XID는 모든 일반적인 XID보다 오래된 것으로 간주된다.<br/>

> vacuum_freeze_min_age는 해당 XID 값을 보유한 row를 frozen될 때까지 얼마나 오래 기다려야 하는지를 제어한다.<br/>
 이 설정을 늘리면 그 XID가 부착된 row가 곧 다시 수정될 것이기 때문에 frozen될 필요가 없는 작업을 피할 수 있지만, 이 설정을 줄이면 테이블을 다시 VACUUM해야 하는 트랜잭션 수가 증가한다.<br/>

> VACUUM은 테이블의 어떤 페이지를 스캔해야 하는지 결정하기 위해 visibility map을 사용한다.<br/>
 보통은 페이지에 더 이상 사용되지 않는 row 버전이 없는 경우에도 해당 페이지를 건너뛸 것이다.<br/>
 그러나 해당 페이지에는 여전히 오래된 XID 값이 있는 row 버전이 있을 수 있다.<br/>
 따라서 일반적인 VACUUM은 항상 테이블의 모든 오래된 row 버전을 frozen하지는 않는다.<br/>
 이런 경우에는 VACUUM이 결국 공격적인 VACUUM을 수행해야 할 수 있으며, 이는 모든 대상이 아직 frozen되지 않은 XID 및 MXID 값을 frozen할 것이다.<br>
 이는 모든 visible하지만 모든 동frozen되지 않은 페이지를 포함한다.<br/>
 실제로 대부분의 테이블은 주기적인 공격적인 VACUUM이 필요하다.<br/>
 vacuum_freeze_table_age는 VACUUM이 그렇게 하는 시점을 제한다: 마지막 스캔 이후 지난 트랜잭션 수가 vacuum_freeze_table_age에서 vacuum_freeze_min_age를 뺀 값보다 큰 경우에는 모든 visible이지만 모든 frozen되지 않은 페이지가 스캔된다.<br/>
 vacuum_freeze_table_age를 0으로 설정하면 VACUUM이 항상 공격적인 전략을 사용하도록 강제한다.<br/>

> 테이블이 VACUUM 작업을 받지 않는 최대 기간은 마지막 공격적인 VACUUM 시간의 vacuum_freeze_min_age 값을 제외한 20억 개의 트랜잭션이다.<br/>
 이 기간을 초과하여 VACUUM 작업이 수행되지 않으면 데이터 손실이 발생할 수 있다.<br/>
 이를 방지하기 위해 autovacuum_freeze_max_age 구성 매개변수에서 지정된 age보다 오래된 XID를 포함할 수 있는 row가 있는 테이블에서 autovacuum이 호출된다.
 autovacuum이 비활성화되어 있더라도 실행된다.<br/>

> 이것은 특정 테이블이 그렇지 않은 경우에도 autovacuum이 대략적으로 autovacuum_freeze_max_age에서 vacuum_freeze_min_age를 뺀 트랜잭션마다 한 번씩 호출됨을 의미한다.<br/>
 공간 회수 목적으로 정기적으로 VACUUM 실행되는 테이블의 경우에는 이는 그다지 중요하지 않다.<br/>
 그러나 정적 테이블(삽입은 받지만 업데이트나 삭제는 받지 않는 테이블을 포함하여)의 경우 공간 회수를 위해 VACUUM을 할 필요가 없으므로, 매우 큰 정적 테이블에서 강제 autovacuum 간격을 최대화하려는 시도가 유용할 수 있다.<br/>
 이는 autovacuum_freeze_max_age를 증가시키거나 vacuum_freeze_min_age를 감소시킴으로써 수행할 수 있다.<br/>

> vacuum_freeze_table_age의 유효한 최대값은 0.95 * autovacuum_freeze_max_age이다.<br>
 이보다 높은 설정은 최대값으로 제한된다.<br/>
 autovacuum_freeze_max_age보다 높은 값은 의미가 없다.<br/>
 왜냐하면 그 시점에서 이미 anti-wraparound autovacuum이 트리거되고 있기 때문이다.<br/>
 그리고 0.95의 배수를 사용하는 것은 해당 시점이 오기 전에 수동 VACUUM을 실행할 여유 공간을 남겨두는 역할을 한다.<br/> 짧은 설명으로, vacuum_freeze_table_age는 autovacuum_freeze_max_age보다 약간 낮은 값으로 설정되어야 합니다. 충분한 여유 공간을 남겨두어 정기적으로 예약된 VACUUM이나 보통의 삭제 및 업데이트 활동으로 트리거된 autovacuum이 해당 시간대에 실행될 수 있도록 한다.
 이 값을 너무 가까이 설정하면 공간 회수를 위해 테이블이 최근에 VACUUM 처리되었더라도 anti-wraparound autovacuum이 발생할 수 있으며, 더 낮은 값은 더 자주 공격적인 VACUUM 작업을 유발할 수 있다.<br/>

> autovacuum_freeze_max_age를 증가시키면(그리고 이에 따라 vacuum_freeze_table_age도 증가한다) 유일한 단점은 데이터베이스 클러스터의 pg_xact 및 pg_commit_ts 하위 디렉터리가 더 많은 공간을 차지할 것이라는 것이다.<br/>
 왜냐하면 모든 트랜잭션의 커밋 상태와 (track_commit_timestamp가 활성화된 경우) autovacuum_freeze_max_age 지점까지의 타임스탬프를 저장해야 하기 때문이다.<br/>
 커밋 상태는 트랜잭션 당 두 개의 비트를 사용하므로, autovacuum_freeze_max_age를 최대 허용 값인 20억으로 설정하면 pg_xact의 크기는 약 반 기가바이트, pg_commit_ts의 크기는 약 20GB로 예상된다.<br/>
 이것이 전체 데이터베이스 크기에 비해 미미하다면, autovacuum_freeze_max_age를 최대 허용 값으로 설정하는 것이 좋다. 그렇지 않으면, pg_xact 및 pg_commit_ts 저장소에 할당하려는 공간에 따라 설정해야한다.<br/>
 기본값인 2억 개의 트랜잭션은 약 50MB의 pg_xact 저장소와 약 2GB의 pg_commit_ts 저장소를 사용한다.<br/>

> vacuum_freeze_min_age를 줄이는 것의 단점 중 하나는 VACUUM이 무의미한 작업을 수행할 수 있다는 것이다.<br/>
 row 버전을 frozen하는 것은 해당 row이 곧 수정될 경우(새로운 XID를 획득하게 되는 경우) 시간 낭비다.<br/>
 따라서 설정은 row가 더 이상 변경되지 않을 정도로 크게 설정되어야 한다.

> 데이터베이스의 가장 오래된 frozen되지 않은 XID의 age를 추적하기 위해 VACUUM은 시스템 테이블 pg_class와 pg_database에 XID 통계를 저장한다.
 특히, 테이블의 pg_class 행의 relfrozenxid column에는 가장 최근의 VACUUM의 끝에서 남은 가장 오래된 frozen되지 않은 XID가 포함되어 있다(일반적으로 가장 최근의 공격적인 VACUUM).
 마찬가지로, 데이터베이스의 pg_database 행의 datfrozenxid column은 해당 데이터베이스에 나타나는 frozen되지 않은 XID의 하한이다.<br/>
 이는 데이터베이스 내의 각 테이블의 relfrozenxid 값의 최솟값이다.<br/>
 이 정보를 확인하는 편리한 방법은 아래의 쿼리를 실행하는 것이다.<br/>
 
```sql
SELECT c.oid::regclass                                    as table_name,
       greatest(age(c.relfrozenxid), age(t.relfrozenxid)) as age
FROM pg_class c
         LEFT JOIN pg_class t ON c.reltoastrelid = t.oid
WHERE c.relkind IN ('r', 'm');

SELECT datname, age(datfrozenxid)
FROM pg_database;
```

> age 열은 현재 트랜잭션의 XID부터 cutoff XID까지의 트랜잭션 수를 측정한다.

> VACUUM 명령의 VERBOSE 매개변수가 지정된 경우, VACUUM은 테이블에 대한 여러 가지 통계를 출력한다.<br/>
 이에는 relfrozenxid와 relminmxid가 어떻게 진행되었는지에 대한 정보와 새로 frozen된 페이지 수가 포함된다.<br/>
 autovacuum 로깅(log_autovacuum_min_duration로 제어됨)이 autovacuum에 의해 실행된 VACUUM 작업에 대해 보고할 때 서버 로그에도 동일한 세부 정보가 나타난다.<br/>

> VACUUM은 일반적으로 마지막 VACUUM 이후 수정된 페이지만 스캔하지만, relfrozenxid는 frozen되지 않은 XID를 포함할 수 있는 테이블의 모든 페이지가 스캔될 때만 진행될 수 있다.<br/>
 이는 relfrozenxid가 vacuum_freeze_table_age 트랜잭션 이상으로 오래되었거나, VACUUM의 FREEZE 옵션이 사용된 경우, 또는 이미 모든 페이지가 모두 frozen되어 있지 않은 경우에 발생한다.<br/>
 VACUUM이 이미 모두 frozen된 페이지가 아닌 테이블의 모든 페이지를 스캔할 때, age(relfrozenxid)를 사용한 값을 설정해야 합니다. 이 값은 사용된 vacuum_freeze_min_age 설정 값보다 약간 더 큰 값이어야 합니다 (VACUUM이 시작된 이후의 트랜잭션 수만큼 더 크게). VACUUM은 테이블에 남아 있는 가장 오래된 XID를 relfrozenxid로 설정한다.<br/> 따라서 최종 값이 엄격히 요구되는 값보다 훨씬 최근일 수 있다.<br/>
 autovacuum_freeze_max_age에 도달할 때까지 테이블에 대해 relfrozenxid를 진행하는 VACUUM이 없는 경우, 테이블에 대해 곧 autovacuum이 강제로 실행된다.<br/>

> 어떤 이유에서든지 autovacuum이 테이블에서 오래된 XID를 정리하지 못하는 경우, 시스템은 데이터베이스의 가장 오래된 XID가 wraparound 지점으로부터 4천만 트랜잭션에 도달할 때부터 다음과 같은 경고 메시지를 발생시킨다.<br/>

```
WARNING:  database "mydb" must be vacuumed within 39985967 transactions
HINT:  To avoid a database shutdown, execute a database-wide VACUUM in that database.
```

> 수동으로 VACUUM을 실행하면 문제가 해결된다.<br/>
 그러나 주의할 점은 VACUUM을 슈퍼유저가 수행해야만 시스템 카탈로그를 처리할 수 있기 때문에 그렇게 하지 않으면 실패할 것이다. 이는 데이터베이스의 datfrozenxid를 진행시킬 수 없게 된다.<br.>
 이러한 경고가 무시된다면, wraparound까지 남은 트랜잭션이 300만 개 미만이 될 때 새로운 XID를 할당하지 않겠다고 시스템이 거부한다.<br/>

```
ERROR:  database is not accepting commands to avoid wraparound data loss in database "mydb"
HINT:  Stop the postmaster and vacuum that database in single-user mode.
```

> 이 상태에서 이미 진행 중인 트랜잭션은 계속할 수 있지만, 읽기 전용 트랜잭션만 시작할 수 있다.<br/>
 데이터베이스 레코드를 수정하거나 관계를 잘라내는 작업은 실패한다.<br/>
 VACUUM 명령은 여전히 정상적으로 실행할 수 있다.<br/>
 postmaster를 중지하거나 단일 사용자 모드로 전환할 필요가 없으며, 정상 작동을 복원하기 위해 다음 단계를 따르면 된다.<br/>
 1. 오래된 준비된 트랜잭션을 해결해야한다.<br/>
    이를 확인하려면 age(transactionid)가 큰 행에 대해 pg_prepared_xacts를 확인하면 된다.<br/>
    이러한 트랜잭션은 커밋되거나 롤백되어야 한다.<br/>
 2. 긴 시간 동안 계속되는 열려 있는 트랜잭션을 종료해야한다.<br/>
    이를 확인하려면 age(backend_xid) 또는 age(backend_xmin)이 큰 row에 대해 pg_stat_activity를 확인하면 된다.<br/>
    이러한 트랜잭션은 커밋되거나 롤백되어야 하며, pg_terminate_backend를 사용하여 세션을 종료할 수도 있다.<br/>
 3. 오래된 복제 슬롯을 삭제해야한다.<br/>
    age(xmin) 또는 age(catalog_xmin)이 큰 슬롯을 찾으려면 pg_stat_replication을 사용하면 된다.<br/>
    많은 경우, 이러한 슬롯은 더 이상 존재하지 않는 서버로의 복제를 위해 생성되었거나 오랫동안 다운된 서버에 대한 것이다.<br/>
    해당 슬롯을 삭제하면 아직 존재하고 해당 슬롯에 연결을 시도할 수도 있는 서버를 위해 해당 복제본을 다시 구축해야 할 수 있다.<br/>
 4. 대상 데이터베이스에서 VACUUM을 실행한다.<br/>
    데이터베이스 전체에 대한 VACUUM이 가장 간단하다.<br/>
    소요 시간을 줄이기 위해 relminxid가 가장 오래된 테이블에 수동으로 VACUUM 명령을 실행할 수도 있다.<br/>
    이 시나리오에서는 XID를 필요로하는 VACUUM FULL을 사용하면 안된다.<br/>
    슈퍼 사용자 모드에서만 성공하며, 이 경우 트랜잭션 ID wraparound 위험을 높이기 때문에 XID를 소비한다.<br/>
    정상 작동을 복원하기 위해 필요한 최소한의 작업 이상을 수행하기 때문에 VACUUM FREEZE도 사용하면 안된다.<br/>
 5. 정상 작동이 복원되면 향후 문제를 피하기 위해 대상 데이터베이스에서 autovacuum이 올바르게 구성되어 있는지 확인하면 된다.<br/>

> 이전 버전에서는 postmaster를 중지하고 단일 사용자 모드에서 데이터베이스를 VACUUM해야 할 때가 있었다.<br/>
 전형적인 시나리오에서는 더 이상 이 작업이 필요하지 않으며 가능한 한 피해야 한다.<br/>
 왜냐하면 이는 시스템을 중단해야 하기 때문입니다.<br/>
 또한 데이터 손실을 방지하기 위해 설계된 트랜잭션 ID wraparound 안전장치를 비활성화하기 때문에 더 위험하다.<br/>
 이 시나리오에서 단일 사용자 모드를 사용해야 하는 유일한 이유는 VACUUM이 필요 없는 테이블을 TRUNCATE하거나 삭제하여 그들을 피하고 싶을 때이다.<br/>
 300만 개의 트랜잭션 안전 여유 공간은 관리자가 이를 수행할 수 있도록 만들어진 것이다.<br/>

Multixacts와 Wraparound
> Multixacts와 ID는 여러 트랜잭션에 의한 row lock을 지원하는 데 사용된다.<br/>
 튜플 헤더에는 lock 정보를 저장할 공간이 제한되어 있기 때문에 한 row를 동시에 lock을 하는 여러 트랜잭션이 있는 경우 해당 정보는 "multiple transaction ID" 또는 간단히 multixact ID로 인코딩된다.<br/>
 특정 multixact ID에 포함된 어떤 트랜잭션 ID인지에 대한 정보는 별도의 pg_multixact 하위 디렉터리에 저장되며, 튜플 헤더의 xmax 필드에는 multixact ID만 표시됩니다.<br/>
 트랜잭션 ID와 마찬가지로 multixact ID도 32비트 카운터 및 해당 저장소로 구현되어 있으며, 모두 신중한 노화 관리, 저장소 정리 및 wraparound 처리가 필요하다.<br/>
 각 multixact의 구성원 목록을 보유하는 별도의 저장소 영역이 있으며, 이 역시 32비트 카운터를 사용하며 관리된다.<br/>
 VACUUM이 테이블의 어느 부분이든 스캔할 때, vacuum_multixact_freeze_min_age보다 오래된 multixact ID를 만나면 해당 값을 다른 값으로 대체한다.<br/>
 이 값은 제로 값, 단일 트랜잭션 ID 또는 더 최신의 multixact ID일 수 있다.<br/>
 각 테이블마다 pg_class.relminmxid에는 해당 테이블의 튜플 중에 여전히 나타나는 가장 오래된 가능한 multixact ID가 저장된다.<br/>
 이 값이 vacuum_multixact_freeze_table_age보다 오래된 경우, 강력한 VACUUM이 강제된다.<br/>
 강력한 VACUUM은 모두 all-frozen으로 알려진 페이지만 건너뛴다.<br/>
 mxid_age() 함수를 사용하여 pg_class.relminmxid의 age를 확인할 수 있다.<br/>
 어떤 원인이든 상관없이 강력한 VACUUM은 테이블의 relminmxid를 진행시킬 수 있는 것이 보장된다.<br/>
 결국 모든 데이터베이스의 모든 테이블이 스캔되고 가장 오래된 multixact 값이 진행됨에 따라 디스크에 저장된 오래된 multixact 저장소를 제거할 수 있다.<br/>
 안전장치로, multixact age가 autovacuum_multixact_freeze_max_age보다 큰 테이블에 대해 강력한 VACUUM 스캔이 발생한다.<br/>
 또한 multixact 구성원이 차지하는 저장소가 2GB를 초과하는 경우, 모든 테이블에 대해 가장 오래된 multixact age부터 시작하여 더 자주 강력한 VACUUM 스캔이 발생한다.<br/>
 이러한 두 가지 종류의 강력한 스캔은 autovacuum이 명시적으로 비활성화된 경우에도 발생한다.<br/>
 XID 경우와 유사하게, autovacuum이 테이블에서 오래된 multixact id를 지우지 못하는 경우, 시스템은 데이터베이스의 가장 오래된 multixact id가 wraparound 지점에서 4천만 거래에 이를 때부터 경고 메시지를 발생시킨다.<br/>
 그리고 XID 경우와 마찬가지로, 이러한 경고를 무시하면 시스템은 wraparound까지 남은 거래가 300만 개 미만이 되면 새로운 multixact id를 생성하지 않도록 거부한다.<br/>
 multixact id가 고갈된 경우의 정상 작동은 XID가 고갈된 경우와 거의 동일한 방식으로 복원할 수 있다.<br/>
 하지만 아래와 같은 차이점이 있다.<br/>
 1. 실행 중인 트랜잭션 및 준비된 트랜잭션은 multixact에 나타날 가능성이 전혀 없다면 무시할 수 있다.<br/>
 2. multixact ID 정보는 pg_stat_activity와 같은 시스템 뷰에서 직접적으로 확인할 수는 없지만, 오래된 XID를 찾는 것은 여전히 multixact id wraparound 문제를 일으키는 트랜잭션을 결정하는 좋은 방법이다.<br/>
 3. XID 고갈은 모든 쓰기 트랜잭션을 차단하지만, multixact ID 고갈은 multixact ID를 필요로 하는 row lock을 포함하는 일부 쓰기 트랜잭션만 차단한다.<br/>

Autovacuum Daemon
> PostgreSQL에는 VACUUM 및 ANALYZE 명령을 자동화하는 autovacuum이라는 선택적이지만 매우 권장되는 기능이 있다.<br/> 
 autovacuum은 삽입, 업데이트 또는 삭제된 튜플이 많은 테이블을 확인한다.<br/>
 이러한 확인은 통계 수집 기능을 사용하므로 track_counts가 true로 설정되지 않으면 autovacuum을 사용할 수 없다.<br/>
 기본 구성에서는 autovacuum이 활성화되어 있고 관련된 구성 매개변수가 적절하게 설정된다.<br/>

> "autovacuum daemon"은 실제로 여러 프로세스로 구성되어 있다.<br/>
 모든 데이터베이스에 대한 autovacuum worker processes를 시작하는 autovacuum launcher라는 영구적인 daemon process가 있다.<br/>
 launcher는 시간에 따라 작업을 분산하며, 각 데이터베이스에서 매 autovacuum_naptime 초마다 하나의 woker를 시작하려고 노력한다.<br/>
 따라서 N 개의 데이터베이스가있는 경우, 새로운 작업자는 autovacuum_naptime/N 초마다 시작된다.<br/>
 동시에 실행되는 tovacuum_max_workers worker processes의 최대 수가 있다.<br/>
 처리해야 할 데이터베이스가 autovacuum_max_workers보다 많은 경우 첫 번째 작업자가 작업을 완료하자마자 다음 데이터베이스가 처리된다.<br/>
 각 worker processes는 해당 데이터베이스 내의 각 테이블을 확인하고 필요에 따라 VACUUM 및/또는 ANALYZE를 실행한다.<br/>
 log_autovacuum_min_duration을 설정하여 autovacuum 작업자의 활동을 모니터링할 수 있다.<br/>

> 몇 개의 큰 테이블이 짧은 시간 내에 모두 VACUUM 처리 가능해지면, 모든 autovacuum woker가 그러한 테이블을 장기간 VACUUM 처리하는 데 바쁠 수 있다.<br/>
 이로 인해 다른 테이블 및 데이터베이스가 작업자가 사용 가능해질 때까지 VACUUM 처리되지 않을 수 있습니다.<br/>
 하나의 데이터베이스에있는 woker 수에는 제한이 없지만, woker는 이미 다른 woker가 수행 한 작업을 반복하지 않도록 노력한다.<br/>
 실행 중인 woker는 수는 max_connections 또는 superuser_reserved_connections 제한에 포함되지 않는다는 점을 유의해야 한다.<br/>

> 테이블의 relfrozenxid 값이 autovacuum_freeze_max_age 트랜잭션보다 오래된 경우 항상 VACUUM 처리된다.<br/>
 이는 스토리지 매개 변수를 통해 수정된 freeze max age를 가진 테이블에도 적용된다.<br/>
 그렇지 않으면, 마지막 VACUUM 이후에 만료 된 튜플 수가 "vacuum threshold"을 초과하면 테이블이 VACUUM 처리된다.<br/>
 vacuum threshold값은 `vacuum threshold = vacuum base threshold + vacuum scale factor * number of tuples`로 정의된다.<br/>
 vacuum base threshold값은 autovacuum_vacuum_threshold이며, vacuum scale factor값은 autovacuum_vacuum_scale_factor이고, 튜플 수는 pg_class.reltuples다.<br/>
 테이블은 또한 마지막 VACUUM 이후에 삽입된 튜플 수가 정의된 vacuum insert threshold값보다 초과된 경우에도 VACUUM되고 `vacuum insert threshold = vacuum base insert threshold + vacuum insert scale factor * number of tuples`로 정의된다.
 vacuum insert base threshold값은 autovacuum_vacuum_insert_threshold이고, vacuum insert scale factor는 autovacuum_vacuum_insert_scale_factor다.<br/>
 이러한 VACUUM 작업은 테이블의 일부를 모두 visible로 표시하고 튜플을 frozen할 수 있으며, 이는 후속 VACUUM 작업에서 필요한 작업을 줄일 수 있다.<br/>
 INSERT 작업을 수행하지만 UPDATE/DELETE 작업을 수행하지 않거나 거의 수행하지 않는 테이블의 경우, 테이블의 autovacuum_freeze_min_age를 낮추는 것이 유용할 수 있다.<br/>
 이렇게 하면 이전 VACUUM 작업에서 튜플을 frozen할 수 있다.<br/>
 구식 튜플과 삽입된 튜플의 수는 누적 통계 시스템에서 얻어지며, 각 UPDATE, DELETE 및 INSERT 작업에서 업데이트된다.<br/>
 이 값은 과도한 부하 하에 정보가 손실될 수 있으므로 반정확할 수 있다.<br/>
 테이블의 relfrozenxid 값이 vacuum_freeze_table_age 트랜잭션 이상으로 오래된 경우, 오래된 튜플을 frozen하고 relfrozenxid를 진행시키기 위해 공격적인 VACUUM이 수행된다.<br/>
 그렇지 않으면, 마지막 VACUUM이 이후 수정된 페이지만 스캔된다.<br/>

> analyze에 대해서도 비슷한 조건이 사용되고 임계값은 `analyze threshold = analyze base threshold + analyze scale factor * number of tuples`로 정의 된다.
 지난 ANALYZE 이후 삽입, 업데이트 또는 삭제된 총 튜플 수와 비교된다.<br/>

> 파티션 테이블은 튜플을 직접 저장하지 않기 때문에 autovacuum에서 직접 처리되지 않는다.<br/>
 autovacuum은 다른 테이블과 마찬가지로 테이블 파티션을 처리한다.<br/>
이는 autovacuum이 파티션 테이블에 대해 ANALYZE를 실행하지 않기 때문에 발생하는데, 이는 파티션 테이블 통계를 참조하는 쿼리의 최적화되지 않은 계획을 초래할 수 있다.<br/>
 파티션 테이블이 처음으로 채워질 때와 파티션의 데이터 분포가 중요하게 변경될 때마다 수동으로 ANALYZE를 실행하여이 문제를 해결할 수 있다.<br/>

> 임시 테이블은 autovacuum에서 액세스할 수 없습니다.<br/>
 따라서 적절한 vacuum 및 analyze 작업은 세션 SQL 명령을 통해 수행되어야 한다.<br/>

> 기본 임계값과 스케일 팩터는 postgresql.conf에서 가져오지만, 이를 테이블 단위로 재정의할 수 있다.<br/>
 테이블의 스토리지 매개변수를 통해 설정을 변경하면 해당 테이블을 처리할 때 해당 값을 사용하고, 그렇지 않으면 전역 설정이 사용된다.<br/>

> 여러 개의 worker가 실행될 때, 모든 실행 중인 woker에 대해 autovacuum 비용 지연 매개변수가 "balanced"로
 설정되어 있으므로 실제로 실행되는 woker의 수에 관계없이 시스템에 대한 총 I/O 영향은 동일하다.<br/>
 그러나 per-table autovacuum_vacuum_cost_delay나 autovacuum_vacuum_cost_limit 스토리지 매개변수가 설정된 테이블을 처리하는 worker는 균형 조정 알고리즘에서 고려되지 않는다.<br/>

> 일반적으로 Autovacuum worker는 다른 명령을 차단하지 않습니다.<br/>
 만약 프로세스가 autovacuum에 의해  SHARE UPDATE EXCLUSIVE lock과 충돌하는 lock을 얻으려고 시도하면, lock획득이 중단되고 autovacuum이 중단된다.<br/>
 그러나 autovacuum이 트랜잭션 ID wraparound(ex: pg_stat_activity view의 autovacuum 쿼리 이름이 (to prevent wraparound)로 끝남)를 방지하기 위해 실행 중인 경우, autovacuum이은 자동으로 중단되지 않는다.<br/>

> 정기적으로 SHARE UPDATE EXCLUSIVE lock과 충돌하는 lock을 획득하는 명령(ex: ANALYZE)을 실행하면 autovacuum이 영원히 완료되지 않을 수 있으므로 주의해야 한다.<br/>

## VACUUM Progress Reporting
<!-- https://www.postgresql.org/docs/current/progress-reporting.html#VACUUM-PROGRESS-REPORTING-->
## VACUUM FULL Progress Reporting
<!-- https://www.postgresql.org/docs/current/progress-reporting.html#CLUSTER-PROGRESS-REPORTING-->