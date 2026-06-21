---
layout: post
title: "SQL to ER Diagram : 브라우저에서 SQL 스키마를 ER 다이어그램으로"
author: 'Juho'
date: 2026-06-20 00:00:00 +0900
categories: [Dev]
tags: [Database, Schema, MySQL]
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
2. [주요 기능](#주요-기능)
3. [지원 SQL 종류](#지원-sql-종류)
4. [사용 방법](#사용-방법)
5. [특징](#특징)
   - [프라이버시](#프라이버시)
   - [내보내기와 다중 지원](#내보내기와-다중-지원)
6. [시사점](#시사점)
7. [Reference](#reference)

## 개요

SQL to ER Diagram은 SQL 스키마를 브라우저에서 직접 대화형 ER 다이어그램으로 변환하는 무료 오픈소스 도구입니다.
별도 회원가입이나 설치 없이 즉시 사용할 수 있습니다.
SQL 스키마를 시각적으로 빠르게 확인하고자 하는 개발자에게 유용합니다.

## 주요 기능

CREATE TABLE 문을 입력하면 테이블, 컬럼, 기본 키, 외래 키 및 관계를 시각화합니다.
테이블을 드래그하여 원하는 위치에 배치할 수 있습니다.
완성된 다이어그램은 PNG 또는 SVG로 내보낼 수 있습니다.

| 기능 | 설명 |
|------|------|
| 스키마 시각화 | 테이블, 컬럼, 기본 키, 외래 키, 관계를 다이어그램으로 표현 |
| 레이아웃 조정 | 테이블을 드래그하여 배치 |
| 내보내기 | PNG 또는 SVG 형식으로 출력 |

## 지원 SQL 종류

PostgreSQL, MySQL, SQLite, SQL Server, Snowflake, BigQuery 문법을 포함한 표준 DDL을 지원합니다.
기본 키, 외래 키, UNIQUE, NOT NULL 등 주요 제약 조건도 인식합니다.

| 항목 | 지원 범위 |
|------|-----------|
| 데이터베이스 문법 | PostgreSQL, MySQL, SQLite, SQL Server, Snowflake, BigQuery |
| 표준 | 표준 DDL |
| 제약 조건 | 기본 키, 외래 키, UNIQUE, NOT NULL |

## 사용 방법

편집기에 SQL CREATE TABLE 문을 붙여넣기만 하면 자동으로 다이어그램이 렌더링됩니다.
별도의 설정이나 학습 곡선이 필요하지 않습니다.

## 특징

### 프라이버시

모든 처리가 로컬에서 진행됩니다.
입력한 데이터가 서버로 전송되지 않습니다.

### 내보내기와 다중 지원

완성된 다이어그램은 PNG 또는 SVG 이미지로 내보내거나 URL로 공유할 수 있습니다.
Prisma, SQLAlchemy, Sequelize 등 다양한 스키마 형식도 지원합니다.

| 특징 | 내용 |
|------|------|
| 프라이버시 | 모든 처리가 로컬에서 진행되어 데이터가 서버로 전송되지 않음 |
| 내보내기 | PNG 또는 SVG 이미지, URL 공유 |
| 다중 지원 | Prisma, SQLAlchemy, Sequelize 등 다양한 스키마 형식 지원 |

## 시사점

SQL to ER Diagram은 설치나 회원가입 없이 브라우저에서 즉시 스키마를 시각화할 수 있는 도구입니다.
모든 처리가 로컬에서 이루어지므로 민감한 스키마 정보도 안심하고 다룰 수 있습니다.
표준 DDL과 다양한 데이터베이스 문법, ORM 스키마 형식을 폭넓게 지원하여 활용 범위가 넓습니다.

## Reference

- [SQL to ER Diagram](https://sqltoerdiagram.com/)
