---
layout: post
title: PostgreSQL PL/pgSQL - Role(User, Group)
author: 'Juho'
date: 2024-02-07 13:00:00 +0900
categories: [PostgreSQL, Database, Role, User, Group]
tags: [PostgreSQL, Database, Role, User, Group]
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
1. [사용자 확인](#사용자-확인)
2. [사용자 생성](#사용자-생성)
 - 1) [Option Parameters](#option-parameters)
3. [사용자 생성](#사용자-생성)

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

그룹으로 사용되는 Role의 멤버를 추가하고 제거하는 선호되는 방법은 GRANT와 REVOKE를 사용하는 것이다.<br/>
`ALTER ROLE`를 사용하여 Role의 속성을 변경하고, `DROP ROLE`을 사용하여 Role을 제거한다.<br/>
`CREATE ROLE`에서 지정한 모든 속성은 이후 `ALTER ROLE` 명령으로 수정할 수 있습니다.<br/>


<br/>


### Option Parameters
<table>
    <tr>
        <th >옵션</th>
        <th>값</th>
        <th>설명</th>
    </tr>
    <tr>
        <td rowspan="2" >SUPERUSER | NOSUPERUSER</td>
        <td>SUPERUSER</td>
        <td>데이터베이스 내에서 모든 제한을 무시할 수 있는 슈퍼 유저인지 결정한다.<br/>
        새로운 슈퍼 유저를 만드려면 생성자가 슈퍼유저여야 한다.
        </td>
    </tr>
    <tr>
        <td>NOSUPERUSER</td>
        <td>기본 값</td>
    </tr>
    <tr>
        <td rowspan="2" >CREATEDB | NOCREATEDB</td>
        <td>CREATEDB</td>
        <td>새 데이터베이스를 생성할 수 있다.<br/>
        슈퍼 유저 역활 또는 CREATEDB를 가진 역활만이 CREATEDB를 지정할 수 있다.
        </td>
    </tr>
    <tr>
        <td>NOCREATEDB</td>
        <td>기본 값</td>
    </tr>
    <tr>
        <td rowspan="2" >CREATEROLE | NOCREATEROLE</td>
        <td>CREATEROLE</td>
        <td>다른 Role을 생성, 변경, 삭제, 주석 및 보안 라벨 변경 허용 여부를 결정한다.
        </td>
    </tr>
    <tr>
        <td>NOCREATEROLE</td>
        <td>기본 값</td>
    </tr>
    <tr>
        <td rowspan="3" >INHERIT | NOINHERIT</td>
        <td>INHERIT</td>
        <td>기본 값, GRANT를 사용하여 한 Role의 멤버십을 다른 Role에 부여할 때, GRANT는 새 맴버에게 권한이 상속될지 여부를 지정하기 위해 WITH INHERIT를 사용할 수 있습니다.<br/>
        멤버 역활이 INHERIT로 설정되어 있으면 WITH INHERIT TRUE로 생성된다.
        </td>
    </tr>
    <tr>
        <td>NOINHERIT</td>
        <td>NOINHERIT로 설정되어 있으면 WITH INHERIT FALSE로 생성된다.</td>
    </tr>
    <tr>
        <td>비고</td>
        <td>PostgreSQL 16 이전 버전에서는 GRANT문이 WITH INHERIT을 지원하지 않았다.
        그래서 Role 수준의 속성을 변경하면 이미 존재하는 GRANT의 동작도 변경되었다.</td>
    </tr>
    <tr>
        <td rowspan="2" >LOGIN | NOLOGIN</td>
        <td>LOGIN</td>
        <td>클라이언트 연결 중에 초기 세션 승인 이름으로 역활을 지정할 수 있는지 여부이다. 로그인 속성을 가진 역활은 사용자로 간주할 수 있다.
        </td>
    </tr>
    <tr>
        <td>NOLOGIN</td>
        <td>기본 값<br/>
        데이터베이스 권한을 관리하는데 유용하지만, 일반적인 의미에서는 사용자가 아니다.</td>
    </tr>
    <tr>
        <td rowspan="3" >BYPASSRLS | NOBYPASSRLS</td>
        <td>BYPASSRLS</td>
        <td>Role이 모든 행 수준의 보안 정책(RLS)을 우회하는지 여부를 결정한다.
        슈퍼 유저 또는 BYPASSRLS를 가진 역활만이 BYPASSRLS 지정할 수 있다.
        </td>
    </tr>
    <tr>
        <td>NOBYPASSRLS</td>
        <td>기본 값<br/>
        </td>
    </tr>
    <tr>
        <td>비고</td>
        <td>pg_dump는 기본적으로 row_security를 OFF로 설정하여 테이블의 모든 내용이 덤프되도록 한다. pg_dump를 실행하는 사용자에게 적절한 권한이 없는 경우 오류가 반환된다.<br/>
        </td>
    </tr>
    <tr>
        <td>CONNECTION LIMIT connlimit</td>
        <td></td>
        <td>Role이 동시에 몇개의 연결을 만들 수 있는지를 지정, 기본 값은 -1로 제한이 없음을 의미한다. 이 제한은 일반 연결에만 적용이 된다. 준비된 트랜잭션 또는 백그라운드 워커 연결은 이 제한에 포함되지 않는다.
        </td>
    </tr>
    <tr>
        <td rowspan="2" >[ ENCRYPTED ] PASSWORD 'password' | PASSWORD NULL</td>
        <td></td>
        <td>Role의 암호를 설정. 암호 인증을 사용할 계획이 없는 경우 이 옵션을 생략할 수 있다. 암호가 지정되지 않으면 null로 설정되고 해당 사용자의 암호 인증은 항상 실패한다. null 암호는 명시적으로 PASSWORD NULL로 작성할 수 있다. 
        </td>
    </tr>
    <tr>
        <td>비고</td>
        <td> PostgreSQL 10 이전버전에서는 그렇지 않았지만, 빈 문자열을 지정하면 암호가 NULL로 설정된다. 이전 버전에서는 빈 문자열을 사용할 수도 있고 아닐 수도 있었으며, 인증 방법 및 정확한 버전에 따라 libpq가 어떠한 경우에도 사용하지 않도록 거부할 수 있었다. 모호성을 피하기 위해 빈 문자열을 지정하는 것은 피해야한다.
        </td>
    </tr>
    <tr>
        <td>VALID UNTIL 'timestamp'</td>
        <td></td>
        <td>VALID UNTIL은 Role의 암호가 더 이상 유효하지 않은 날짜와 시간을 설정합니다. 생략되면 암호는 언제나 유효합니다.
        </td>
    </tr>
    <tr>
        <td>IN ROLE 'role_name'</td>
        <td></td>
        <td>IN ROLE은 새로운 Role이 기존 Role의 멤버로 자동으로 추가되도록 합니다. 새로운 역활을 관리자로 추가하는 옵션은 없으므로, 별도의 GRANT를 사용해야한다.
        </td>
    </tr>
    <tr>
        <td>IN GROUP 'role_name'</td>
        <td></td>
        <td>IN GROUP은 IN ROLE의 구식 내용이다.
        </td>
    </tr>
    <tr>
        <td>ROLE 'role_name'</td>
        <td></td>
        <td>ROLE은 하나 이상의 지정된 기존 Role이 새로운 Role의 멤버로 자동으로 추가되도록합니다. 새로운 Role이 Group이 되는 효과가 있습니다.
        </td>
    </tr>
    <tr>
        <td>ADMIN 'role_name'</td>
        <td></td>
        <td>ADMIN은 ROLE과 유사하지만, 지정된 ROLE들이 새로운 역할에 ADMIN OPTION과 함께 추가됩니다. 이 Role의 멤버십을 다른 사람에게 부여할 권한을 부여합니다.
        </td>
    </tr>
    <tr>
        <td>USER 'role_name'</td>
        <td></td>
        <td>USER은 ROLE의 구식 내용이다.
        </td>
    </tr>
    <tr>
        <td>SYSID 'uid'</td>
        <td></td>
        <td>SYSID은 무시되지만 이전 버전과의 호환성을 위해 허용된다.
        </td>
    </tr>
</table>

