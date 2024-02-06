---
layout: post
title: PostgreSQL PL/pgSQL - Role, User, Group
author: 'Juho'
date: 2024-02-28 09:00:00 +0900
categories: [PostgreSQL, Database, Role, User, Group]
tags: [PostgreSQL, Database, Role, User, Group]
pin: True
toc : True
---

## 목차
1. [사용자 확인](#사용자-확인)
2. [사용자 생성](#사용자-생성)

## 사용자 확인
현재 사용자 정보를 확인할때는 `pg_user` 혹은`pg_shadow` 을 사용하면 된다.<br/>
두 내용의 차이점으로는 `pg_shadow`에서는 암호화된 비밀번호를 표시하기 때문에 슈퍼 유저나 권한을 가진 사람만 조회할 수 있다. <br/>
`pg_user`는 비밀번호를 `*`로 표기하기 때문에 일반 유저도 조회할 수 있다.<br/>
하지만 `pg_shadow`는 8.1 이하 버전의 하위 호환성을 위해서 남아 있기 때문에 최신 버전에서는 [`pg_user`](https://www.postgresql.org/docs/16/view-pg-shadow.html){:target="_blank"}를 사용하는것을 권장하고 있다.<br/>
<br/>

## 사용자 생성
PostgreSQL에서 [User](https://www.postgresql.org/docs/16/sql-creategroup.html){:target="_blank"}, [Group](https://www.postgresql.org/docs/16/sql-creategroup.html){:target="_blank"}은 [Role](https://www.postgresql.org/docs/16/sql-createrole.html){:target="_blank"}의 alias다.<br/>

`Role`은 데이터베이스 객체를 소유하고 권한을 가질 수 있는 개체로 <br/>
사용 방법에 따라 `User`, `Group` 또는 둘 다로 간주될 수 있다. <br/>

`CREATE ROLE name [ [ WITH ] option [ ... ] ]`을 사용하려면 Create Role 권한이 있거나, 데이터베이스 슈퍼 유저여야한다. <br/>
`CREATE ROLE`은 데이터베이스 클러스터에 새 역활을 추가한다.<br/>
`Role`은 클러스터 수준에서 정의되므로, 클러스터 내 모든 데이터베이스에서 유효하다.<br/>

