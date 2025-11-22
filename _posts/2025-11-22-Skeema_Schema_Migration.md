---
layout: post
title: Skeema를 활용한 DB 스키마 관리 및 마이그레이션
author: 'Juho'
date: 2025-11-22 09:00:00 +0900
categories: [Skeema]
tags: [Skeema, Database, Migration, Schema]
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
1. [Skeema란?](#skeema란)
2. [설치 및 준비](#설치-및-준비)
3. [Skeema 설정](#skeema-설정)
4. [디렉터리 구조](#디렉터리-구조)
5. [기본 워크플로](#기본-워크플로)
6. [스키마 Pull](#스키마-pull)
7. [스키마 Diff](#스키마-diff)
8. [스키마 Push](#스키마-push)
9. [마이그레이션 SQL 생성](#마이그레이션-sql-생성)
10. [안전 옵션](#안전-옵션)
11. [무시/제외 규칙](#무시제외-규칙)
12. [뷰/루틴 처리](#뷰루틴-처리)
13. [검증 및 테스트](#검증-및-테스트)
14. [트러블슈팅](#트러블슈팅)

## Skeema란?
**Skeema**는 MySQL/MariaDB 스키마를 파일 시스템 기반으로 관리할 수 있게 해주는 CLI 도구이다.  
- 데이터베이스 스키마를 SQL 파일로 추출(pull)  
- 환경 간 스키마 차이를 비교(diff)  
- 변경 사항을 데이터베이스에 적용(push)  

Git과 함께 사용하면 스키마 변경 이력을 버전 관리할 수 있어 협업 환경에서 유용하다.  

## 설치 및 준비

### 설치 방법

**macOS (Homebrew)**
```bash
brew install skeema
```

**Linux (apt)**
```bash
# Debian/Ubuntu
curl -LO https://github.com/skeema/skeema/releases/download/v1.13.1/skeema_1.13.1_linux_amd64.deb
sudo dpkg -i skeema_1.13.1_linux_amd64.deb
```

**바이너리 직접 설치**
```bash
# Linux
curl -LO https://github.com/skeema/skeema/releases/download/v1.13.1/skeema_1.13.1_linux_amd64.tar.gz
tar -xzf skeema_1.13.1_linux_amd64.tar.gz
sudo mv skeema /usr/local/bin/
```

### 지원 버전
- MySQL 5.5, 5.6, 5.7, 8.0, 8.4
- MariaDB 10.1 ~ 10.11, 11.x
- Percona Server 5.5 ~ 8.0

### 필요 권한
Skeema가 제대로 동작하려면 다음 권한이 필요하다.  

| 작업 | 필요 권한 |
|------|-----------|
| pull/diff | `SELECT`, `SHOW VIEW`, `SHOW DATABASES` |
| push | `ALTER`, `CREATE`, `DROP`, `INDEX`, `REFERENCES` |
| 뷰 관리 | `CREATE VIEW`, `SHOW VIEW` |
| 루틴 관리 | `CREATE ROUTINE`, `ALTER ROUTINE`, `EXECUTE` |
| 안전한 작업 | `LOCK TABLES` (pt-online-schema-change 사용 시) |

```sql
-- 권한 부여 예시
-- 글로벌 권한
GRANT SHOW DATABASES ON *.* TO 'skeema_user'@'%';

-- 스키마별 권한
GRANT SELECT, SHOW VIEW,
      ALTER, CREATE, DROP, INDEX, REFERENCES,
      CREATE VIEW, LOCK TABLES
ON mydb.* TO 'skeema_user'@'%';
```

## Skeema 설정
`.skeema` 파일에 기본 설정과 환경별 설정을 작성한다.  

```ini
# db/skeema/.skeema

default-character-set=utf8mb4
default-collation=utf8mb4_unicode_ci
generator=skeema:1.13.1-community
schema={db_schema}
allow-auto-inc=int, int unsigned, bigint unsigned, bigint
lint-reserved-word=ignore
lint-dupe-index=ignore

[dev]
skip-allow-unsafe
flavor={$database}
host={$db_host}
password={$db_pwd}
port={$db_port}
user={$db_user}

[prod]
skip-allow-unsafe
flavor={$database}
host={$db_host}
password={$db_pwd}
port={$db_port}
user={$db_user}
```

### 주요 설정 항목

| 옵션 | 설명 |
|------|------|
| `default-character-set` | 기본 문자셋 |
| `default-collation` | 기본 콜레이션 |
| `schema` | 대상 스키마(데이터베이스) 이름 |
| `flavor` | MySQL 또는 MariaDB 버전 지정 (예: `mariadb:10.6`, `mysql:8.0`) |
| `skip-allow-unsafe` | unsafe 검사 건너뛰기 (allow-unsafe=true와 동일) |
| `lint-reserved-word` | 예약어 사용 경고 무시 |
| `lint-dupe-index` | 중복 인덱스 경고 무시 |

### 환경 섹션 상속
`[dev]`, `[prod]` 섹션은 상위 설정을 상속받는다.  
```ini
# 공통 설정
default-character-set=utf8mb4

[dev]
# dev는 공통 설정 + dev 전용 설정 적용
host=dev-db.example.com

[prod]
# prod는 공통 설정 + prod 전용 설정 적용
host=prod-db.example.com
safe-below-size=10M  # prod만 안전 옵션 적용
```

## 디렉터리 구조
Skeema는 스키마별로 하위 폴더를 생성한다.  

```
db/skeema/
├── .skeema              # 루트 설정 (환경별 접속 정보)
├── mydb/                # 스키마(데이터베이스) 폴더
│   ├── .skeema          # 스키마별 설정 (schema=mydb)
│   ├── users.sql        # 테이블 DDL
│   ├── orders.sql
│   └── products.sql
└── another_db/
    ├── .skeema
    └── ...
```

각 `.sql` 파일에는 `CREATE TABLE` 문이 포함된다.  
```sql
-- users.sql
CREATE TABLE `users` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `email` varchar(255) NOT NULL,
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_email` (`email`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## 기본 워크플로
Skeema를 활용한 스키마 관리 워크플로는 다음과 같다.  

```
1. pull (dev)     : 개발 서버 스키마를 파일로 추출
       ↓
2. Git 커밋/리뷰   : 변경 사항을 Git으로 관리, PR 리뷰
       ↓
3. diff (staging) : 스테이징 환경과 비교
       ↓
4. push (staging) : 스테이징에 먼저 적용
       ↓
5. 검증           : 애플리케이션 테스트
       ↓
6. diff (prod)    : 운영 환경과 비교, SQL 검토
       ↓
7. 백업           : 운영 DB 백업
       ↓
8. push (prod)    : 운영에 적용
```

## 스키마 Pull
개발 서버의 스키마를 파일로 추출한다.  

```bash
skeema pull dev
```

실행 결과:
- 각 테이블별 `.sql` 파일이 생성됨
- 테이블 구조, 인덱스, 제약조건 등이 포함

```
db/skeema/
├── .skeema
├── users.sql
├── orders.sql
└── products.sql
```

### pull 옵션
```bash
# 특정 스키마만 pull
skeema pull dev --schema=mydb

# 뷰 포함
skeema pull dev --include-views

# 루틴(프로시저/함수) 포함
skeema pull dev --include-routines
```

## 스키마 Diff
운영 서버와 현재 스키마 파일을 비교한다.  

```bash
skeema diff prod
```

출력 예시:
```sql
-- diffing mydb

ALTER TABLE `users` ADD COLUMN `phone` varchar(20) DEFAULT NULL;
ALTER TABLE `orders` MODIFY COLUMN `status` enum('pending','completed','cancelled') NOT NULL;
```

diff 명령은 실제로 변경을 적용하지 않고 차이점만 출력한다.  

### diff 종료 코드
- `0`: 차이 없음
- `1`: 차이 있음 (적용 가능)
- `2`: 오류 발생

CI/CD에서 활용:
```bash
skeema diff prod
if [ $? -eq 1 ]; then
  echo "스키마 변경 감지됨"
fi
```

## 스키마 Push
diff로 확인한 변경 사항을 실제 데이터베이스에 적용한다.  

```bash
# 운영 서버에 적용
skeema push prod

# dry-run (실제 적용 없이 확인만)
skeema push prod --dry-run

# 특정 스키마만 적용
skeema push prod --schema=mydb
```

### push 전 확인사항
```bash
# 1. diff로 변경 내용 확인
skeema diff prod

# 2. dry-run으로 실행 계획 확인
skeema push prod --dry-run

# 3. 실제 적용
skeema push prod
```

## 마이그레이션 SQL 생성
diff 결과를 SQL 파일로 저장하여 마이그레이션 스크립트로 활용한다.  

```bash
skeema diff prod > db/migration/V$(date +%Y%m%d%H%M)__{$description}.sql
```

### 네이밍 규칙
Flyway/Liquibase와 호환되는 네이밍:  
```
V{VERSION}__{DESCRIPTION}.sql
```

- `VERSION`: 정렬 가능한 버전 (YYYYMMDDHHMM 권장)  
- `DESCRIPTION`: 변경 내용 설명 (snake_case)  

```
db/migration/
├── V202511211030__add_users_table.sql
├── V202511211145__add_phone_column.sql
└── V202511220900__create_orders_index.sql
```

### Flyway 연동 팁
```bash
# Flyway 마이그레이션 디렉터리에 직접 생성
skeema diff prod > flyway/sql/V$(date +%Y%m%d%H%M)__schema_update.sql

# Flyway 실행
flyway migrate
```

### Idempotency 고려
Skeema가 생성하는 DDL은 기본적으로 idempotent하지 않다.  
수동으로 조건문을 추가하거나, Flyway/Liquibase의 버전 관리에 의존한다.  

## 안전 옵션
운영 환경에서는 안전 옵션을 적극 활용한다.  

### safe-below-size
지정된 크기 이하의 테이블에만 unsafe 작업 허용:  
```ini
[prod]
safe-below-size=100M  # 100MB 이하 테이블만 unsafe 허용, 초과 시 거부
```

### allow-unsafe
데이터 손실 가능성 있는 작업 허용 여부:
```ini
# 기본값: false (안전 모드)
allow-unsafe=false

# 위험한 작업도 허용
allow-unsafe=true
```

위험한 작업 예시:
- 컬럼 삭제
- 테이블 삭제
- 컬럼 타입 축소

### skip-verify
스키마 동기화 검증 건너뛰기:
```bash
skeema push prod --skip-verify
```

### alter-wrapper (pt-online-schema-change)
대용량 테이블에 온라인 DDL 적용:
```ini
[prod]
alter-wrapper=/usr/bin/pt-online-schema-change --execute --alter {CLAUSES} D={SCHEMA},t={TABLE},h={HOST},P={PORT},u={USER},p={PASSWORD}
```

또는 gh-ost 사용:
```ini
alter-wrapper=/usr/bin/gh-ost --alter "{CLAUSES}" --database={SCHEMA} --table={TABLE} --host={HOST} --user={USER} --password={PASSWORD} --execute
```

### alter-wrapper-min-size
특정 크기 이상 테이블에만 wrapper 적용:
```ini
alter-wrapper-min-size=50M
```

## 무시/제외 규칙
자동 생성 테이블이나 특정 객체를 제외할 수 있다.  

### ignore-table
특정 테이블 제외:
```ini
# 단일 테이블
ignore-table=schema_migrations

# 패턴 사용 (정규식)
ignore-table=/^tmp_/
ignore-table=/^backup_/
ignore-table=/_bak$/
```

### ignore-schema
특정 스키마 제외:
```ini
ignore-schema=information_schema
ignore-schema=performance_schema
ignore-schema=mysql
ignore-schema=sys
```

### 루틴 제외
```ini
# 프로시저/함수 무시 (기본값)
include-routines=false
```

> 참고: 트리거는 Skeema Community에서 지원하지 않음  

### .skeema 파일에서 무시 설정
```ini
# db/skeema/.skeema
ignore-table=/^flyway_/
ignore-table=/^DATABASECHANGELOG/
ignore-table=schema_version
```

## 뷰/루틴 처리

### 뷰 포함
```bash
skeema pull dev --include-views
skeema diff prod --include-views
skeema push prod --include-views
```

설정 파일에 추가:
```ini
include-views=true
```

### 주의사항
- 뷰 정의 시 참조하는 테이블이 먼저 존재해야 함  
- `DEFINER` 절이 환경마다 다를 수 있음  
- 뷰 간 의존성 순서 문제 발생 가능  

```ini
# .skeema 설정에서 DEFINER 제거
strip-definer=true
```

### 루틴(프로시저/함수) 포함
```bash
skeema pull dev --include-routines
```

```ini
include-routines=true
```

루틴 주의사항:  
- `DEFINER`, `SQL SECURITY` 설정 확인  
- 실행 권한 필요 (`EXECUTE` privilege)  
- 루틴 간 의존성 순서 수동 관리 필요  
 
## 검증 및 테스트

### skeema lint
SQL 파일의 문법 및 스타일 검사:  
```bash
skeema lint

# 특정 규칙 무시
skeema lint --lint-pk=ignore  
```

lint 규칙:  
- `lint-pk`: PRIMARY KEY 필수 여부  
- `lint-engine`: 스토리지 엔진 검사  
- `lint-charset`: 문자셋 검사  
- `lint-dupe-index`: 중복 인덱스 검사  

### 스테이징 먼저 적용
```bash
# 1. 스테이징에 적용
skeema push staging

# 2. 애플리케이션 테스트
./run_tests.sh

# 3. 성공 시 운영 적용
skeema push prod
```

### 적용 후 확인 쿼리
```sql
-- 테이블 크기 확인
SELECT
  table_name,
  ROUND(data_length/1024/1024, 2) AS data_mb,
  ROUND(index_length/1024/1024, 2) AS index_mb
FROM information_schema.tables
WHERE table_schema = 'mydb'
ORDER BY data_length DESC;

-- 인덱스 확인
SHOW INDEX FROM mydb.users;

-- 쿼리 플랜 확인
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
```

### CI/CD 통합
```yaml
# .gitlab-ci.yml 예시
schema-diff:
  stage: test
  script:
    - skeema diff prod
  allow_failure: true
  only:
    changes:
      - db/skeema/**/*
```

## 트러블슈팅

### Collation/Charset 드리프트
환경 간 기본 collation이 다를 때 발생:  
```
-- 오류 예시
ALTER TABLE `users` CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

해결:
```ini
# .skeema에 명시적 지정
default-character-set=utf8mb4
default-collation=utf8mb4_unicode_ci
```

### DEFAULT CURRENT_TIMESTAMP 차이  
MySQL 버전에 따라 표현 방식이 다름:  
```sql
-- MySQL 5.6
DEFAULT CURRENT_TIMESTAMP

-- MySQL 5.7+
DEFAULT CURRENT_TIMESTAMP()
```

해결:
```ini
# flavor 정확히 지정
flavor=mysql:5.7
```

### AUTO_INCREMENT 불일치  
AUTO_INCREMENT 값은 diff에서 무시됨 (정상 동작):  
```ini
# 이미 기본 동작
allow-auto-inc=int, int unsigned, bigint unsigned, bigint
```

특정 AUTO_INCREMENT 값 강제 필요 시 수동 처리.  

### file-per-table 가정  
Skeema는 테이블당 하나의 파일을 가정함.  
파티션 테이블 사용 시 주의:  
```sql
-- 파티션 정보도 함께 포함됨
CREATE TABLE `logs` (
  ...
) PARTITION BY RANGE (YEAR(created_at)) (
  PARTITION p2024 VALUES LESS THAN (2025),
  PARTITION p2025 VALUES LESS THAN (2026)
);
```

### 빈 diff인데 push 실패  
```bash
# 캐시 문제일 수 있음
skeema pull dev --skip-format
skeema diff prod
```

### 대용량 테이블 ALTER 타임아웃  
```ini
# 타임아웃 늘리기
connect-timeout=30

# 또는 pt-osc 사용
alter-wrapper=/usr/bin/pt-online-schema-change ...
```

### 권한 오류
```
Error: Access denied for user 'admin'@'%'
```

필요 권한 확인:
```sql
SHOW GRANTS FOR 'admin'@'%';
```

---

Skeema를 사용하면 스키마 변경을 코드처럼 관리할 수 있다.  
개발/운영 환경 간 스키마 차이를 쉽게 파악하고 안전하게 마이그레이션을 진행할 수 있다.
