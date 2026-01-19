---
layout: post
title: sqlit - 터미널에서 데이터베이스를 다루는 경량 TUI SQL 클라이언트
author: 'Juho'
date: 2026-01-19 00:00:00 +0900
categories: [Database]
tags: [PostgreSQL, MySQL, SQLite, Database, CLI]
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
2. [sqlit이란](#sqlit이란)
3. [지원하는 데이터베이스](#지원하는-데이터베이스)
4. [핵심 기능](#핵심-기능)
5. [설치 방법](#설치-방법)
6. [사용 방법](#사용-방법)
7. [보안 기능](#보안-기능)
8. [테마 옵션](#테마-옵션)
9. [결론](#결론)
10. [참고 자료](#참고-자료)

## 개요

데이터베이스 작업을 할 때 SSMS, DBeaver, DataGrip 같은 무거운 GUI 도구를 사용하는 것이 부담스러울 때가 있습니다.
특히 간단한 쿼리 실행이나 데이터 확인만 필요한 경우 터미널에서 바로 작업할 수 있다면 훨씬 효율적입니다.

**sqlit**은 이런 요구를 충족시키는 경량 터미널 기반 SQL 클라이언트입니다.
lazygit에서 영감을 받아 만들어진 이 도구는 "The lazygit of SQL Databases"라는 슬로건처럼 빠르고 직관적인 데이터베이스 관리 경험을 제공합니다.

## sqlit이란

sqlit은 터미널 환경에서 데이터베이스를 효율적으로 관리하기 위한 TUI(Terminal User Interface) 도구입니다.

### 탄생 배경

개발자는 SSMS나 VS Code 확장과 같은 리소스 소모가 큰 도구 대신 경량이면서도 아름답고 키보드 중심의 TUI를 원했습니다.
데이터에 접근하는 것을 즐겁고 빠르며 쉽게 만들기 위해 sqlit을 개발했습니다.

### 기술 스택

sqlit은 Python의 Textual 프레임워크를 기반으로 구축되었습니다.
경량이면서 저메모리 구현으로 수백만 행의 데이터도 로드할 수 있습니다.
외부 문서 없이도 직관적인 상호작용이 가능하도록 설계되었습니다.

## 지원하는 데이터베이스

sqlit은 20개 이상의 로컬 및 클라우드 데이터베이스를 지원합니다.

### 로컬 데이터베이스

| 데이터베이스 | 지원 |
|-------------|------|
| PostgreSQL | O |
| MySQL | O |
| SQLite | O |
| SQL Server | O |
| MariaDB | O |
| Oracle | O |
| DuckDB | O |
| FirebirdSQL | O |

### 클라우드 데이터베이스

| 데이터베이스 | 지원 |
|-------------|------|
| Snowflake | O |
| BigQuery | O |
| Redshift | O |
| Supabase | O |
| Turso | O |
| CloudFlare D1 | O |
| CockroachDB | O |
| ClickHouse | O |

### 기타 지원

- IBM Db2
- SAP HANA
- Teradata
- Trino
- Presto
- Apache Flight SQL
- Athena

## 핵심 기능

### 빠른 연결

단순히 `sqlit` 명령만 입력하면 저장된 연결 목록이 표시됩니다.
UI에서 원하는 데이터베이스를 선택하면 즉시 연결됩니다.

### Docker 컨테이너 자동 탐색

Docker에서 실행 중인 데이터베이스 컨테이너를 자동으로 탐색합니다.
별도의 설정 없이 컨테이너에 바로 연결할 수 있습니다.

### Vim 스타일 키바인딩

Vim 사용자에게 익숙한 모달 편집과 키바인딩을 제공합니다.
키보드만으로 모든 작업을 빠르게 수행할 수 있습니다.

### 구문 하이라이팅 및 자동완성

SQL 쿼리 작성 시 구문 하이라이팅이 적용됩니다.
테이블명, 컬럼명 등에 대한 자동완성 기능도 제공됩니다.

### 쿼리 히스토리

연결별로 쿼리 히스토리가 관리됩니다.
퍼지 검색과 필터링 기능으로 이전에 실행한 쿼리를 쉽게 찾을 수 있습니다.

### 다양한 출력 형식

쿼리 결과를 CSV 또는 JSON 형식으로 출력할 수 있습니다.
스크립트나 다른 도구와의 연동이 용이합니다.

## 설치 방법

### pipx 설치 (권장)

```bash
pipx install sqlit-tui
```

### uv 설치

```bash
uv tool install sqlit-tui
```

### pip 설치

```bash
pip install sqlit-tui
```

### Arch Linux AUR

```bash
yay -S sqlit
```

### SSH 기능 포함 설치

SSH 터널링 기능이 필요한 경우 다음과 같이 설치합니다.

```bash
pipx install 'sqlit-tui[ssh]'
```

### Nix

Nix flake를 통한 설치도 지원합니다.

## 사용 방법

### 기본 실행

터미널에서 `sqlit` 명령을 실행하면 저장된 연결 목록이 표시됩니다.

```bash
sqlit
```

### 연결 생성

데이터베이스 URL을 사용하여 연결할 수 있습니다.

```bash
# PostgreSQL
sqlit connect postgresql://user:password@localhost:5432/dbname

# MySQL
sqlit connect mysql://user:password@localhost:3306/dbname

# SQLite
sqlit connect sqlite:///path/to/database.db
```

### CLI 쿼리 실행

프로그래밍 방식으로 쿼리를 실행할 수 있습니다.

```bash
# 기본 쿼리 실행
sqlit query -c "ConnectionName" -q "SELECT * FROM users"

# CSV 출력
sqlit query -c "ConnectionName" -q "SELECT * FROM users" --format csv

# JSON 출력
sqlit query -c "ConnectionName" -q "SELECT * FROM users" --format json
```

### 연결 관리

저장된 연결을 관리하는 서브커맨드를 제공합니다.

```bash
# 연결 목록 확인
sqlit connections list

# 연결 삭제
sqlit connections delete "ConnectionName"
```

## 보안 기능

### OS 키링 통합

비밀번호는 일반 텍스트가 아닌 시스템 키링에 암호화되어 저장됩니다.
macOS의 Keychain, Linux의 Secret Service, Windows의 Credential Locker를 지원합니다.

### SSH 터널링

원격 데이터베이스에 안전하게 연결하기 위한 SSH 터널 기능을 지원합니다.
bastion 호스트를 통한 연결도 가능합니다.

### 클라우드 CLI 통합

Azure, AWS, GCP CLI와 통합되어 클라우드 데이터베이스에 안전하게 연결할 수 있습니다.
기존 클라우드 인증 정보를 활용합니다.

## 테마 옵션

sqlit은 다양한 테마를 제공합니다.

| 테마 | 설명 |
|------|------|
| Rose Pine | 부드러운 파스텔 톤 |
| Tokyo Night | 어두운 네온 스타일 |
| Nord | 차가운 북유럽 색상 |
| Gruvbox | 따뜻한 레트로 색상 |

사용자의 취향에 맞게 테마를 선택하여 작업 환경을 꾸밀 수 있습니다.

## 결론

sqlit은 터미널에서 데이터베이스 작업을 빠르고 효율적으로 수행할 수 있게 해주는 훌륭한 도구입니다.

### 장점 요약

- 20개 이상의 데이터베이스 지원
- Docker 컨테이너 자동 탐색
- Vim 스타일 키바인딩으로 빠른 작업
- OS 키링을 통한 안전한 자격증명 저장
- SSH 터널링 지원
- 경량이면서 수백만 행 처리 가능

### 추천 대상

- 터미널 작업을 선호하는 개발자
- lazygit 같은 TUI 도구에 익숙한 사용자
- 가벼운 데이터베이스 클라이언트가 필요한 경우
- 여러 종류의 데이터베이스를 동시에 관리해야 하는 경우

무거운 GUI 도구 대신 터미널에서 빠르게 데이터베이스 작업을 하고 싶다면 sqlit을 사용해보는 것을 권장합니다.

## 참고 자료

- [sqlit GitHub Repository](https://github.com/Maxteabag/sqlit)
