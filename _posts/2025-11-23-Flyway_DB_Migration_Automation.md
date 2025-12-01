---
layout: post
title: Flyway를 활용한 DB Migration 자동화
author: 'Juho'
date: 2025-12-01 09:00:00 +0900
categories: [Flyway]
tags: [Flyway, Skeema, Database, Migration, CI/CD]
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
1. [Flyway란?](#flyway란)
2. [Flyway + Skeema 조합의 장점](#flyway--skeema-조합의-장점)
3. [Flyway 설치 및 설정](#flyway-설치-및-설정)
4. [마이그레이션 파일 생성 워크플로](#마이그레이션-파일-생성-워크플로)
5. [자동화 스크립트 작성](#자동화-스크립트-작성)
6. [CI/CD 파이프라인 구성](#cicd-파이프라인-구성)
7. [롤백 전략](#롤백-전략)
8. [모니터링 및 알림](#모니터링-및-알림)
9. [실전 팁과 주의사항](#실전-팁과-주의사항)
10. [트러블슈팅](#트러블슈팅)

## Flyway란?

Flyway는 데이터베이스 마이그레이션 도구로 SQL 기반 버전 관리를 제공한다.  

### 주요 특징
- 버전 관리: 각 마이그레이션에 고유 버전 번호 부여  
- 순차 실행: 버전 순서대로 자동 실행  
- 상태 추적: 실행 이력을 `flyway_schema_history` 테이블에 기록  
- Idempotency: 이미 실행된 마이그레이션은 재실행하지 않음  
- 다양한 DB 지원: MySQL, PostgreSQL, Oracle, SQL Server 등  

### Flyway 마이그레이션 파일 네이밍 규칙  
```
V{VERSION}__{DESCRIPTION}.sql
```
- `V`: Versioned 마이그레이션 (필수)  
- `VERSION`: 버전 번호 (예: 1, 1.1, 20251123001)  
- `__`: 구분자 (언더스코어 2개)  
- `DESCRIPTION`: 설명 (snake_case 권장)  

예시:
```
V1__create_users_table.sql
V2__add_email_index.sql
V20251123001__add_orders_table.sql
```

## Flyway + Skeema 조합의 장점

| 도구 | 역할 | 장점 |
|------|------|------|
| Skeema | 스키마 diff 생성 | • 환경 간 차이 자동 감지<br>• DDL 자동 생성<br>• Git 기반 버전 관리 |
| Flyway | 마이그레이션 실행 및 추적 | • 버전별 실행 이력 관리<br>• 순차적 적용 보장<br>• 재실행 방지 |

### 워크플로 개요

```
[개발 DB]
    ↓
(Skeema pull) → [SQL 파일]
    ↓
(Git 커밋/리뷰)
    ↓
(Skeema diff) → [마이그레이션 SQL]
    ↓
[Flyway 마이그레이션 폴더]
    ↓
(Flyway migrate) → [스테이징/운영 DB]
```

## Flyway 설치 및 설정

### Gradle 프로젝트에 Flyway 추가

**build.gradle** (Gradle Groovy DSL)

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'org.flywaydb.flyway' version '10.8.1'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '17'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'

    // Flyway Core
    implementation 'org.flywaydb:flyway-core:10.8.1'
    implementation 'org.flywaydb:flyway-mysql:10.8.1'

    // Database Driver
    runtimeOnly 'com.mysql:mysql-connector-j'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

// Flyway 설정
flyway {
    url = System.getenv('DB_URL') ?: 'jdbc:mysql://localhost:3306/mydb'
    user = System.getenv('DB_USER') ?: 'root'
    password = System.getenv('DB_PASSWORD') ?: ''
    locations = ['classpath:db/migration']
    baselineOnMigrate = true
    baselineVersion = '1'
    validateOnMigrate = true
    cleanDisabled = true
}

tasks.register('flywayMigrateDev', org.flywaydb.gradle.task.FlywayMigrateTask) {
    group = 'flyway'
    description = 'Migrate dev database'
    url = 'jdbc:mysql://dev-db.example.com:3306/mydb'
    user = System.getenv('DEV_DB_USER')
    password = System.getenv('DEV_DB_PASSWORD')
    locations = ['classpath:db/migration']
    baselineOnMigrate = true
}

tasks.register('flywayMigrateStaging', org.flywaydb.gradle.task.FlywayMigrateTask) {
    group = 'flyway'
    description = 'Migrate staging database'
    url = 'jdbc:mysql://staging-db.example.com:3306/mydb'
    user = System.getenv('STAGING_DB_USER')
    password = System.getenv('STAGING_DB_PASSWORD')
    locations = ['classpath:db/migration']
    baselineOnMigrate = true
}

tasks.register('flywayMigrateProd', org.flywaydb.gradle.task.FlywayMigrateTask) {
    group = 'flyway'
    description = 'Migrate production database'
    url = 'jdbc:mysql://prod-db.example.com:3306/mydb'
    user = System.getenv('PROD_DB_USER')
    password = System.getenv('PROD_DB_PASSWORD')
    locations = ['classpath:db/migration']
    validateOnMigrate = true
    outOfOrder = false
    cleanDisabled = true
}

test {
    useJUnitPlatform()
}
```


### 프로젝트 구조

```
project/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/myapp/
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/
│   │           ├── migration/           # Flyway 마이그레이션
│   │           │   ├── V1__initial_schema.sql
│   │           │   ├── V2__add_orders_table.sql
│   │           │   └── V3__add_email_index.sql
│   │           └── skeema/              # Skeema 스키마 정의
│   │               ├── .skeema
│   │               └── mydb/
│   │                   ├── users.sql
│   │                   └── orders.sql
│   └── test/
├── scripts/
│   ├── generate_migration.sh
│   └── deploy.sh
├── build.gradle
└── gradle.properties
```

### Spring Boot application.yml 설정

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?useSSL=false&serverTimezone=UTC
    username: ${DB_USER:root}
    password: ${DB_PASSWORD:}
    driver-class-name: com.mysql.cj.jdbc.Driver

  jpa:
    hibernate:
      ddl-auto: validate  # Flyway가 스키마 관리하므로 validate만
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.MySQLDialect

  flyway:
    enabled: true
    baseline-on-migrate: true
    validate-on-migrate: true
    locations: classpath:db/migration
    table: flyway_schema_history
    baseline-version: 1
    encoding: UTF-8
    placeholder-replacement: true
    placeholders:
      appUser: app_user

---
# Development Profile
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:mysql://dev-db.example.com:3306/mydb
    username: ${DEV_DB_USER}
    password: ${DEV_DB_PASSWORD}
  flyway:
    baseline-on-migrate: true

---
# Staging Profile
spring:
  config:
    activate:
      on-profile: staging
  datasource:
    url: jdbc:mysql://staging-db.example.com:3306/mydb
    username: ${STAGING_DB_USER}
    password: ${STAGING_DB_PASSWORD}

---
# Production Profile
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:mysql://prod-db.example.com:3306/mydb
    username: ${PROD_DB_USER}
    password: ${PROD_DB_PASSWORD}
  flyway:
    validate-on-migrate: true
    out-of-order: false
    clean-disabled: true
```

### gradle.properties (환경변수 관리)

```properties
# Local Development
DB_URL=jdbc:mysql://localhost:3306/mydb
DB_USER=root
DB_PASSWORD=

# 또는 환경변수로 관리
# export DB_URL=jdbc:mysql://localhost:3306/mydb
# export DB_USER=root
# export DB_PASSWORD=secret
```

## 마이그레이션 파일 생성 워크플로

### 1. 개발 환경에서 스키마 변경

```bash
# 개발 DB에 직접 변경 적용
mysql -h dev-db -u dev_user -p mydb

# 또는 애플리케이션 ORM으로 변경
```

### 2. Skeema로 변경사항 추출

```bash
# 1. 개발 DB에서 현재 스키마 pull
skeema pull dev

# 2. Git diff로 변경사항 확인
git diff db/skeema/

# 3. 운영 DB와 비교하여 마이그레이션 SQL 생성
skeema diff prod > /tmp/migration.sql
```

### 3. Flyway 마이그레이션 파일로 변환

```bash
# 타임스탬프 기반 버전 생성
VERSION=$(date +%Y%m%d%H%M%S)
DESCRIPTION="add_user_phone_column"

# 마이그레이션 파일 생성
cat /tmp/migration.sql > src/main/resources/db/migration/V${VERSION}__${DESCRIPTION}.sql
```

### 4. 마이그레이션 파일 검증

```bash
# Gradle로 검증
./gradlew flywayValidate

# 마이그레이션 상태 확인
./gradlew flywayInfo
```

출력 예시:
```
> Task :flywayInfo
Schema version: 2

+-----------+---------+---------------------+------+---------------------+
| Category  | Version | Description         | Type | Installed On        |
+-----------+---------+---------------------+------+---------------------+
| Versioned | 1       | initial schema      | SQL  | 2025-11-20 10:00:00 |
| Versioned | 2       | add orders table    | SQL  | 2025-11-21 09:30:00 |
| Pending   | 3       | add email index     | SQL  |                     |
+-----------+---------+---------------------+------+---------------------+

BUILD SUCCESSFUL
```

### 5. 마이그레이션 실행

```bash
# 로컬 환경에서 실행
./gradlew flywayMigrate

# 개발 환경
./gradlew flywayMigrateDev

# 스테이징 환경에 먼저 적용
./gradlew flywayMigrateStaging

# 검증 후 운영 적용
./gradlew flywayMigrateProd

# Spring Boot 프로필 사용
./gradlew bootRun --args='--spring.profiles.active=dev'
./gradlew bootRun --args='--spring.profiles.active=staging'
./gradlew bootRun --args='--spring.profiles.active=prod'
```

## 자동화 스크립트 작성

### generate_migration.sh

마이그레이션 파일 자동 생성 스크립트 (Gradle 통합)

```bash
#!/bin/bash

set -e

# 설정
SKEEMA_DIR="./src/main/resources/db/skeema"
MIGRATIONS_DIR="./src/main/resources/db/migration"
TEMP_SQL="/tmp/migration_$(date +%s).sql"

# 인자 확인
if [ $# -lt 2 ]; then
    echo "Usage: $0 <environment> <description>"
    echo "Example: $0 prod add_user_phone_column"
    exit 1
fi

ENV=$1
DESCRIPTION=$2
VERSION=$(date +%Y%m%d%H%M%S)

echo "=== Skeema + Flyway Migration Generator ==="
echo "Environment: $ENV"
echo "Description: $DESCRIPTION"
echo "Version: $VERSION"
echo ""

# 1. Skeema로 diff 생성
echo "[1/6] Generating diff with Skeema..."
cd $SKEEMA_DIR
skeema diff $ENV > $TEMP_SQL

# 변경사항 확인
if [ ! -s $TEMP_SQL ]; then
    echo "No schema changes detected."
    rm -f $TEMP_SQL
    exit 0
fi

echo "Changes detected:"
cat $TEMP_SQL
echo ""

# 2. Flyway 마이그레이션 파일 생성
cd - > /dev/null
MIGRATION_FILE="${MIGRATIONS_DIR}/V${VERSION}__${DESCRIPTION}.sql"
echo "[2/6] Creating Flyway migration file: $MIGRATION_FILE"

# SQL 파일 생성 (헤더 추가)
cat > $MIGRATION_FILE <<EOF
-- Flyway Migration
-- Version: V${VERSION}
-- Description: ${DESCRIPTION}
-- Generated: $(date '+%Y-%m-%d %H:%M:%S')
-- Environment: ${ENV}
--
-- This migration was automatically generated by Skeema
-- Review carefully before deploying to production

EOF

# Skeema diff 결과 추가
cat $TEMP_SQL >> $MIGRATION_FILE

# 3. SQL 문법 체크
echo "[3/6] Validating SQL syntax..."
if ! grep -qE "ALTER|CREATE|DROP|INSERT|UPDATE|DELETE" $MIGRATION_FILE; then
    echo "Warning: No DDL/DML statements found in migration file"
fi

# 4. Gradle Flyway 검증
echo "[4/6] Running Gradle Flyway validation..."
./gradlew flywayValidate || {
    echo "Flyway validation failed"
    exit 1
}

# 5. 마이그레이션 정보 출력
echo "[5/6] Checking migration info..."
./gradlew flywayInfo

# 6. Git 추가
echo "[6/6] Adding to Git..."
git add $MIGRATION_FILE
git add $SKEEMA_DIR

echo ""
echo "=== Migration file created successfully ==="
echo "File: $MIGRATION_FILE"
echo ""
echo "Next steps:"
echo "1. Review the migration file: cat $MIGRATION_FILE"
echo "2. Test locally: ./gradlew flywayMigrate"
echo "3. Commit: git commit -m 'feat: $DESCRIPTION'"
echo "4. Test on staging: ./scripts/deploy.sh staging"
echo "5. Deploy to prod: ./scripts/deploy.sh prod"

# Cleanup
rm -f $TEMP_SQL
```

### deploy.sh

환경별 배포 스크립트 (Gradle 기반)

```bash
#!/bin/bash

set -e

# 인자 확인
if [ $# -lt 1 ]; then
    echo "Usage: $0 <environment>"
    echo "Environments: dev, staging, prod"
    exit 1
fi

ENV=$1

# 환경에 따른 Gradle Task 설정
case $ENV in
    dev)
        GRADLE_TASK="flywayMigrateDev"
        ;;
    staging)
        GRADLE_TASK="flywayMigrateStaging"
        ;;
    prod)
        GRADLE_TASK="flywayMigrateProd"
        ;;
    *)
        echo "Invalid environment: $ENV"
        echo "Valid environments: dev, staging, prod"
        exit 1
        ;;
esac

echo "=== Flyway Deployment to $ENV ==="
echo "Gradle Task: $GRADLE_TASK"
echo ""

# 1. 현재 상태 확인
echo "[1/5] Checking current migration status..."
./gradlew flywayInfo

# 2. Pending 마이그레이션 확인
PENDING=$(./gradlew flywayInfo --quiet | grep -c "Pending" || true)

if [ "$PENDING" -eq 0 ]; then
    echo "No pending migrations. Nothing to do."
    exit 0
fi

echo ""
echo "Found $PENDING pending migration(s)"
echo ""

# 3. 운영 환경이면 확인 요청
if [ "$ENV" = "prod" ]; then
    echo "⚠️  WARNING: Deploying to PRODUCTION"
    echo ""

    # Pending 마이그레이션 상세 출력
    echo "Pending migrations:"
    ./gradlew flywayInfo --quiet | grep "Pending"
    echo ""

    read -p "Are you sure you want to proceed? (yes/no): " CONFIRM
    if [ "$CONFIRM" != "yes" ]; then
        echo "Deployment cancelled."
        exit 0
    fi

    # 백업 확인
    echo ""
    read -p "Have you backed up the database? (yes/no): " BACKUP
    if [ "$BACKUP" != "yes" ]; then
        echo "Please backup the database first."
        echo "Run: mysqldump -h \$DB_HOST -u \$DB_USER -p \$DB_NAME > backup_\$(date +%Y%m%d_%H%M%S).sql"
        exit 1
    fi
fi

# 4. 마이그레이션 실행
echo ""
echo "[2/5] Running migrations on $ENV..."
./gradlew $GRADLE_TASK

# 5. 검증
echo ""
echo "[3/5] Validating after migration..."
./gradlew flywayValidate

# 6. 최종 상태 확인
echo ""
echo "[4/5] Final migration status:"
./gradlew flywayInfo

# 7. 알림 (선택사항)
echo ""
echo "[5/5] Sending notification..."
if [ -n "$SLACK_WEBHOOK_URL" ]; then
    ./scripts/notify_slack.sh "$SLACK_WEBHOOK_URL" "$ENV" "success" "DB migration completed successfully"
fi

echo ""
echo "=== Deployment completed successfully ==="
echo ""
echo "Summary:"
echo "- Environment: $ENV"
echo "- Migrations applied: $PENDING"
echo "- Timestamp: $(date '+%Y-%m-%d %H:%M:%S')"
```

### Gradle Task로 스크립트 실행

**build.gradle에 커스텀 태스크 추가**

```groovy
tasks.register('generateMigration', Exec) {
    group = 'flyway'
    description = 'Generate migration file from Skeema diff'

    doFirst {
        if (!project.hasProperty('env') || !project.hasProperty('desc')) {
            throw new GradleException('Usage: ./gradlew generateMigration -Penv=<env> -Pdesc=<description>')
        }
    }

    commandLine './scripts/generate_migration.sh', project.property('env'), project.property('desc')
}

tasks.register('deployMigration', Exec) {
    group = 'flyway'
    description = 'Deploy migration to specific environment'

    doFirst {
        if (!project.hasProperty('env')) {
            throw new GradleException('Usage: ./gradlew deployMigration -Penv=<environment>')
        }
    }

    commandLine './scripts/deploy.sh', project.property('env')
}

tasks.register('backupDatabase', Exec) {
    group = 'database'
    description = 'Backup database before migration'

    doFirst {
        if (!project.hasProperty('env')) {
            throw new GradleException('Usage: ./gradlew backupDatabase -Penv=<environment>')
        }

        def timestamp = new Date().format('yyyyMMdd_HHmmss')
        def backupFile = "backup_${project.property('env')}_${timestamp}.sql"

        println "Creating backup: ${backupFile}"
    }

    commandLine 'sh', '-c', '''
        mysqldump -h $DB_HOST -u $DB_USER -p$DB_PASSWORD $DB_NAME > backup_$(date +%Y%m%d_%H%M%S).sql
    '''
}
```

### 사용 예시

```bash
# 1. 마이그레이션 파일 생성
./gradlew generateMigration -Penv=prod -Pdesc=add_user_phone_column

# 또는 스크립트 직접 실행
./scripts/generate_migration.sh prod add_user_phone_column

# 2. 생성된 파일 확인
cat src/main/resources/db/migration/V20251123093000__add_user_phone_column.sql

# 3. 로컬 테스트
./gradlew flywayMigrate

# 4. Git 커밋
git commit -m "feat: add user phone column migration"

# 5. 스테이징 배포
./gradlew deployMigration -Penv=staging
# 또는
./scripts/deploy.sh staging

# 6. 백업 후 운영 배포
./gradlew backupDatabase -Penv=prod
./gradlew deployMigration -Penv=prod

# 7. Spring Boot로 마이그레이션 실행
./gradlew bootRun --args='--spring.profiles.active=prod'
```

## CI/CD 파이프라인 구성

### GitHub Actions

.github/workflows/db-migration.yml

```yaml
name: DB Migration with Gradle

on:
  push:
    branches: [main, develop]
    paths:
      - 'src/main/resources/db/migration/**'
      - 'src/main/resources/db/skeema/**'
  pull_request:
    paths:
      - 'src/main/resources/db/migration/**'
      - 'src/main/resources/db/skeema/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: gradle

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Validate Migration Files
        run: ./gradlew flywayValidate

      - name: Check Migration Status
        run: ./gradlew flywayInfo

  deploy-staging:
    needs: validate
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: gradle

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Run Migrations on Staging
        env:
          STAGING_DB_URL: ${{ secrets.STAGING_DB_URL }}
          STAGING_DB_USER: ${{ secrets.STAGING_DB_USER }}
          STAGING_DB_PASSWORD: ${{ secrets.STAGING_DB_PASSWORD }}
        run: |
          ./gradlew flywayMigrateStaging
          ./gradlew flywayInfo

      - name: Notify Slack
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Staging DB migration completed'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}

  deploy-production:
    needs: validate
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: gradle

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Backup Production DB
        env:
          PROD_DB_HOST: ${{ secrets.PROD_DB_HOST }}
          PROD_DB_USER: ${{ secrets.PROD_DB_USER }}
          PROD_DB_PASSWORD: ${{ secrets.PROD_DB_PASSWORD }}
          PROD_DB_NAME: ${{ secrets.PROD_DB_NAME }}
        run: |
          sudo apt-get update && sudo apt-get install -y mysql-client
          mysqldump -h $PROD_DB_HOST -u $PROD_DB_USER -p$PROD_DB_PASSWORD $PROD_DB_NAME > backup_$(date +%Y%m%d_%H%M%S).sql

      - name: Upload backup to S3
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-2
      - run: aws s3 cp backup_*.sql s3://my-db-backups/

      - name: Run Migrations on Production
        env:
          PROD_DB_URL: ${{ secrets.PROD_DB_URL }}
          PROD_DB_USER: ${{ secrets.PROD_DB_USER }}
          PROD_DB_PASSWORD: ${{ secrets.PROD_DB_PASSWORD }}
        run: |
          ./gradlew flywayMigrateProd
          ./gradlew flywayInfo

      - name: Notify Slack
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Production DB migration completed'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### GitLab CI

.gitlab-ci.yml

```yaml
image: gradle:8.5-jdk17

stages:
  - validate
  - deploy-staging
  - deploy-production

variables:
  GRADLE_OPTS: "-Dorg.gradle.daemon=false"

cache:
  paths:
    - .gradle/wrapper
    - .gradle/caches

before_script:
  - export GRADLE_USER_HOME=`pwd`/.gradle

validate-migrations:
  stage: validate
  script:
    - ./gradlew flywayValidate
    - ./gradlew flywayInfo
  only:
    changes:
      - src/main/resources/db/migration/**/*
      - src/main/resources/db/skeema/**/*

deploy-staging:
  stage: deploy-staging
  environment:
    name: staging
  variables:
    STAGING_DB_URL: $STAGING_DB_URL
    STAGING_DB_USER: $STAGING_DB_USER
    STAGING_DB_PASSWORD: $STAGING_DB_PASSWORD
  script:
    - ./gradlew flywayMigrateStaging
    - ./gradlew flywayInfo
  only:
    - develop
  when: manual

deploy-production:
  stage: deploy-production
  environment:
    name: production
  variables:
    PROD_DB_URL: $PROD_DB_URL
    PROD_DB_USER: $PROD_DB_USER
    PROD_DB_PASSWORD: $PROD_DB_PASSWORD
  before_script:
    - apt-get update && apt-get install -y mysql-client
    - |
      mysqldump -h $PROD_DB_HOST \
                -u $PROD_DB_USER \
                -p$PROD_DB_PASSWORD \
                $PROD_DB_NAME > backup_$(date +%Y%m%d_%H%M%S).sql
    # S3에 백업 업로드 (선택)
    - apt-get install -y awscli
    - aws s3 cp backup_*.sql s3://my-db-backups/
  script:
    - ./gradlew flywayMigrateProd
    - ./gradlew flywayInfo
  only:
    - main
  when: manual
  needs:
    - validate-migrations

# 롤백 작업 (수동)
rollback-production:
  stage: deploy-production
  environment:
    name: production
  variables:
    PROD_DB_URL: $PROD_DB_URL
    PROD_DB_USER: $PROD_DB_USER
    PROD_DB_PASSWORD: $PROD_DB_PASSWORD
  script:
    - echo "Rolling back production database..."
    # 백업에서 복원
    - |
      mysql -h $PROD_DB_HOST \
            -u $PROD_DB_USER \
            -p$PROD_DB_PASSWORD \
            $PROD_DB_NAME < backup_latest.sql
  only:
    - main
  when: manual
```

### Jenkins Pipeline

Jenkinsfile

```groovy
pipeline {
    agent any

    tools {
        jdk 'JDK17'
        gradle 'Gradle8'
    }

    environment {
        STAGING_DB_URL = credentials('staging-db-url')
        STAGING_DB_USER = credentials('staging-db-user')
        STAGING_DB_PASSWORD = credentials('staging-db-password')
        PROD_DB_URL = credentials('prod-db-url')
        PROD_DB_USER = credentials('prod-db-user')
        PROD_DB_PASSWORD = credentials('prod-db-password')
        SLACK_WEBHOOK = credentials('slack-webhook-url')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Migrations') {
            steps {
                sh './gradlew flywayValidate'
                sh './gradlew flywayInfo'
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    try {
                        sh './gradlew flywayMigrateStaging'
                        sh './gradlew flywayInfo'

                        slackSend(
                            color: 'good',
                            message: "Staging DB migration completed successfully",
                            channel: '#deployments',
                            tokenCredentialId: 'slack-token'
                        )
                    } catch (Exception e) {
                        slackSend(
                            color: 'danger',
                            message: "Staging DB migration failed: ${e.message}",
                            channel: '#deployments',
                            tokenCredentialId: 'slack-token'
                        )
                        throw e
                    }
                }
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to Production?', ok: 'Deploy'

                script {
                    // JDBC URL에서 호스트와 DB 이름 추출
                    def dbUrl = env.PROD_DB_URL
                    def dbHost = dbUrl.replaceAll(/jdbc:mysql:\/\/([^:\/]+).*/, '$1')
                    def dbName = dbUrl.replaceAll(/.*\/([^?]+).*/, '$1')

                    // 백업
                    sh """
                        mysqldump -h ${dbHost} \
                                  -u \$PROD_DB_USER \
                                  -p\$PROD_DB_PASSWORD \
                                  ${dbName} > backup_\$(date +%Y%m%d_%H%M%S).sql
                    """

                    // S3 업로드
                    sh 'aws s3 cp backup_*.sql s3://my-db-backups/'

                    try {
                        sh './gradlew flywayMigrateProd'
                        sh './gradlew flywayInfo'

                        slackSend(
                            color: 'good',
                            message: "Production DB migration completed successfully",
                            channel: '#deployments',
                            tokenCredentialId: 'slack-token'
                        )
                    } catch (Exception e) {
                        slackSend(
                            color: 'danger',
                            message: "Production DB migration failed: ${e.message}",
                            channel: '#deployments',
                            tokenCredentialId: 'slack-token'
                        )
                        throw e
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
```

## 롤백 전략

### Flyway Undo 마이그레이션 (Pro 버전)

Flyway Pro/Enterprise에서는 `U` 접두사로 Undo 스크립트 작성 가능

```
V1__create_users.sql
U1__drop_users.sql
```

### Community 버전 롤백 방법

#### 1. 수동 롤백 스크립트

db/rollbacks/R1__rollback_add_phone_column.sql
```sql
-- Rollback for V20251123093000__add_user_phone_column.sql
-- Execute manually if needed

ALTER TABLE users DROP COLUMN phone;
```

#### 2. 백업 복원

```bash
# 백업 생성
mysqldump -h prod-db -u admin -p mydb > backup_before_migration.sql

# 문제 발생 시 복원
mysql -h prod-db -u admin -p mydb < backup_before_migration.sql
```

#### 3. Flyway 상태 수정

```bash
# 마이그레이션 이력에서 제거 (주의!)
# flyway_schema_history 테이블에서 해당 버전 삭제
mysql -h prod-db -u admin -p mydb -e \
  "DELETE FROM flyway_schema_history WHERE version='20251123093000';"

# 또는 Gradle Flyway repair 사용
./gradlew flywayRepair
```

### 안전한 롤백 전략

**scripts/rollback.sh**

```bash
#!/bin/bash
# rollback.sh - Gradle 기반 롤백

set -e

if [ $# -lt 2 ]; then
    echo "Usage: $0 <environment> <version>"
    echo "Example: $0 prod 20251123093000"
    exit 1
fi

ENV=$1
VERSION=$2
ROLLBACK_SQL="src/main/resources/db/rollbacks/R${VERSION}__rollback.sql"

if [ ! -f $ROLLBACK_SQL ]; then
    echo "Rollback script not found: $ROLLBACK_SQL"
    echo "Please create rollback script first."
    exit 1
fi

echo "=== Rolling back version $VERSION on $ENV ==="
echo ""
echo "Rollback SQL:"
cat $ROLLBACK_SQL
echo ""

read -p "Execute rollback? (yes/no): " CONFIRM
if [ "$CONFIRM" != "yes" ]; then
    echo "Rollback cancelled."
    exit 0
fi

# 1. 백업
echo "[1/4] Creating backup before rollback..."
BACKUP_FILE="backup_before_rollback_$(date +%Y%m%d_%H%M%S).sql"
mysqldump -h $DB_HOST -u $DB_USER -p$DB_PASSWORD $DB_NAME > $BACKUP_FILE
echo "Backup created: $BACKUP_FILE"

# 2. 롤백 SQL 실행
echo ""
echo "[2/4] Executing rollback SQL..."
mysql -h $DB_HOST -u $DB_USER -p$DB_PASSWORD $DB_NAME < $ROLLBACK_SQL

# 3. Flyway 이력 제거
echo ""
echo "[3/4] Removing migration from Flyway history..."
mysql -h $DB_HOST -u $DB_USER -p$DB_PASSWORD $DB_NAME -e \
  "DELETE FROM flyway_schema_history WHERE version='$VERSION';"

# 4. Flyway 상태 확인
echo ""
echo "[4/4] Checking Flyway status..."
./gradlew flywayInfo

echo ""
echo "=== Rollback completed ==="
echo "Backup file: $BACKUP_FILE"
```

**build.gradle에 롤백 태스크 추가**

```groovy
tasks.register('rollbackMigration', Exec) {
    group = 'flyway'
    description = 'Rollback a specific migration version'

    doFirst {
        if (!project.hasProperty('env') || !project.hasProperty('version')) {
            throw new GradleException('Usage: ./gradlew rollbackMigration -Penv=<env> -Pversion=<version>')
        }
    }

    commandLine './scripts/rollback.sh', project.property('env'), project.property('version')
}

tasks.register('createRollbackScript') {
    group = 'flyway'
    description = 'Create template rollback script for latest migration'

    doLast {
        def migrationsDir = file('src/main/resources/db/migration')
        def rollbacksDir = file('src/main/resources/db/rollbacks')

        if (!rollbacksDir.exists()) {
            rollbacksDir.mkdirs()
        }

        // 최신 마이그레이션 파일 찾기
        def latestMigration = migrationsDir.listFiles()
            ?.findAll { it.name =~ /^V.*\.sql$/ }
            ?.max { it.lastModified() }

        if (latestMigration) {
            def version = (latestMigration.name =~ /V(\d+)__/)[0][1]
            def rollbackFile = new File(rollbacksDir, "R${version}__rollback.sql")

            rollbackFile.text = """-- Rollback for ${latestMigration.name}
-- Created: ${new Date().format('yyyy-MM-dd HH:mm:ss')}
--
-- WARNING: Review and modify this template before using!
--
-- Example rollback operations:
-- ALTER TABLE users DROP COLUMN phone;
-- DROP INDEX idx_email ON users;
-- DROP TABLE orders;

-- Add your rollback SQL here:

"""
            println "Rollback template created: ${rollbackFile.path}"
        } else {
            println "No migration files found"
        }
    }
}
```

**사용 예시**

```bash
# 1. 롤백 스크립트 템플릿 생성
./gradlew createRollbackScript

# 2. 생성된 템플릿 수정
vim src/main/resources/db/rollbacks/R20251123093000__rollback.sql

# 3. 롤백 실행
./gradlew rollbackMigration -Penv=prod -Pversion=20251123093000

# 또는 스크립트 직접 실행
./scripts/rollback.sh prod 20251123093000
```

## 모니터링 및 알림

### Slack 알림 스크립트

scripts/notify_slack.sh

```bash
#!/bin/bash

WEBHOOK_URL=$1
ENV=$2
STATUS=$3
MESSAGE=$4

COLOR="good"
if [ "$STATUS" = "failed" ]; then
    COLOR="danger"
fi

curl -X POST $WEBHOOK_URL \
  -H 'Content-Type: application/json' \
  -d @- <<EOF
{
  "attachments": [
    {
      "color": "$COLOR",
      "title": "DB Migration - $ENV",
      "text": "$MESSAGE",
      "fields": [
        {
          "title": "Environment",
          "value": "$ENV",
          "short": true
        },
        {
          "title": "Status",
          "value": "$STATUS",
          "short": true
        }
      ],
      "footer": "Flyway Migration",
      "ts": $(date +%s)
    }
  ]
}
EOF
```

### 마이그레이션 모니터링 쿼리

```sql
-- 최근 마이그레이션 이력 조회
SELECT
  installed_rank,
  version,
  description,
  type,
  script,
  installed_on,
  execution_time,
  success
FROM flyway_schema_history
ORDER BY installed_rank DESC
LIMIT 10;

-- 실패한 마이그레이션 조회
SELECT *
FROM flyway_schema_history
WHERE success = 0;

-- 전체 마이그레이션 통계
SELECT
  COUNT(*) as total_migrations,
  SUM(execution_time) as total_execution_time_ms,
  AVG(execution_time) as avg_execution_time_ms
FROM flyway_schema_history
WHERE success = 1;
```

### Grafana 대시보드 메트릭

```sql
-- Prometheus exporter용 쿼리
SELECT
  COUNT(*) as flyway_migrations_total
FROM flyway_schema_history;

SELECT
  COUNT(*) as flyway_migrations_failed
FROM flyway_schema_history
WHERE success = 0;

SELECT
  MAX(installed_on) as flyway_last_migration_timestamp
FROM flyway_schema_history;
```

## 실전 팁과 주의사항

### 1. 마이그레이션 작성 원칙

좋은 예시
```sql
-- V20251123001__add_user_email_index.sql
-- 단일 목적, 명확한 설명

ALTER TABLE users
ADD INDEX idx_email (email);
```

나쁜 예시
```sql
-- V20251123001__various_changes.sql
-- 여러 테이블, 여러 작업 혼재

ALTER TABLE users ADD COLUMN phone VARCHAR(20);
ALTER TABLE orders MODIFY status VARCHAR(50);
CREATE TABLE products (...);
-- 실패 시 롤백 복잡
```

### 2. 대용량 테이블 처리

```sql
-- V20251123002__add_index_large_table.sql
-- pt-online-schema-change 또는 gh-ost 사용

-- Online DDL 활성화 (MySQL 8.0+)
ALTER TABLE large_table
ADD INDEX idx_created_at (created_at),
ALGORITHM=INSTANT;

-- 또는 Skeema의 alter-wrapper 활용
```

### 3. 데이터 마이그레이션 포함

```sql
-- V20251123003__migrate_user_status.sql
-- DDL과 DML 분리 권장

-- 1. 새 컬럼 추가
ALTER TABLE users
ADD COLUMN status_new ENUM('active', 'inactive', 'suspended') DEFAULT 'active';

-- 2. 기존 데이터 마이그레이션
UPDATE users
SET status_new = CASE
  WHEN old_status = 1 THEN 'active'
  WHEN old_status = 0 THEN 'inactive'
  ELSE 'suspended'
END;

-- 3. 기존 컬럼 삭제는 다음 마이그레이션에서
-- ALTER TABLE users DROP COLUMN old_status;
```

### 4. 환경별 조건부 실행

```sql
-- V20251123004__add_dev_test_data.sql
-- Flyway placeholder 사용

INSERT INTO users (email, name)
VALUES ('test@example.com', 'Test User')
WHERE '${flyway.placeholders.environment}' = 'dev';
```

flyway.conf:
```properties
flyway.placeholders.environment=${ENV}
```

### 5. 멱등성 보장

```sql
-- V20251123005__create_audit_table.sql
-- IF NOT EXISTS 사용

CREATE TABLE IF NOT EXISTS audit_logs (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  action VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 인덱스도 체크
CREATE INDEX IF NOT EXISTS idx_user_id
ON audit_logs(user_id);
```

### 6. Git 관리 전략

```bash
# 마이그레이션 파일은 절대 수정하지 않기
# 잘못된 마이그레이션은 새로운 마이그레이션으로 수정

# 나쁜 예시
git commit --amend  # 이미 배포된 마이그레이션 수정

# 좋은 예시
# V20251123006__fix_previous_migration.sql 생성
```

### 7. Skeema ignore 설정

```ini
# db/skeema/.skeema
# Flyway 관련 테이블 제외
ignore-table=flyway_schema_history
```

## 트러블슈팅

### 1. Checksum 불일치

문제
```
Migration checksum mismatch for migration version 20251123001
```

원인
- 이미 실행된 마이그레이션 파일을 수정함
- 파일 인코딩 변경
- 줄바꿈 문자 변경 (CRLF vs LF)

해결
```bash
# 1. Gradle Flyway repair로 checksum 업데이트
./gradlew flywayRepair

# 운영 환경의 경우 (환경변수 설정 후)
PROD_DB_URL=jdbc:mysql://prod-db.example.com:3306/mydb \
PROD_DB_USER=admin \
PROD_DB_PASSWORD=secret \
./gradlew flywayRepair

# 2. 또는 수동으로 수정
mysql -h prod-db -u admin -p -e \
  "UPDATE flyway_schema_history
   SET checksum = NULL
   WHERE version = '20251123001';"
```

### 2. Out of Order 실행

문제
```
Detected resolved migration not applied to database: 20251123001
```

원인
- 버전 순서가 뒤바뀐 마이그레이션 파일

해결
```properties
# flyway.conf
flyway.outOfOrder=true
```

또는 타임스탬프 버전 사용:
```bash
VERSION=$(date +%Y%m%d%H%M%S)  # 20251123093045
```

### 3. 마이그레이션 실패 후 복구

문제
```
Migration V20251123002 failed
SQL State: 42S21, Error Code: 1060
Duplicate column name 'email'
```

해결
```bash
# 1. 실패한 마이그레이션 확인
./gradlew flywayInfo

# 2. 수동으로 문제 해결
mysql -h prod-db -u admin -p mydb

# 3. Flyway 상태 수정
./gradlew flywayRepair

# 4. 재시도
./gradlew flywayMigrate
# 또는 환경별 태스크
./gradlew flywayMigrateProd
```

### 4. Skeema diff가 빈 결과

문제
- Skeema diff 실행 시 변경사항이 없다고 나옴
- 실제로는 테이블이 다름

원인
- `.skeema` 설정의 `schema` 불일치
- `flavor` 설정 누락

해결
```ini
# db/skeema/.skeema
schema=mydb  # 정확한 스키마 이름
flavor=mysql:8.0  # DB 버전 명시
```

### 5. 대용량 마이그레이션 타임아웃

문제
```
Timeout waiting for migration to complete
```

해결
```properties
# flyway.conf
flyway.lockRetryCount=10
flyway.connectRetries=10
```

또는 배치 처리:
```sql
-- V20251123007__bulk_update.sql
-- 작은 배치로 분할

UPDATE users SET status = 'active' WHERE status IS NULL LIMIT 1000;
-- 여러 번 실행
```

### 6. Baseline 설정

문제
- 기존 운영 DB에 Flyway 도입 시 모든 테이블을 마이그레이션으로 만들기 어려움

해결
```bash
# 1. 현재 상태를 V1으로 baseline 설정
./gradlew flywayBaseline

# 운영 환경의 경우
PROD_DB_URL=jdbc:mysql://prod-db.example.com:3306/mydb \
PROD_DB_USER=admin \
PROD_DB_PASSWORD=secret \
./gradlew flywayBaseline

# 2. 이후 마이그레이션은 V2부터 시작
# V2__add_new_feature.sql
```

**build.gradle에 baseline 설정 추가**
```groovy
flyway {
    baselineOnMigrate = true
    baselineVersion = '1'
}
```

---

Skeema와 Flyway를 조합하면 스키마 변경의 감지부터 적용, 이력 관리까지 전체 워크플로를 자동화할 수 있다.  