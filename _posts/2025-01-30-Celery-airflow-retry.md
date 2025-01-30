---
layout: post
title: Airflow Retry 설정
author: 'Juho'
date: 2025-01-30 09:00:00 +0900
categories: [Airflow]
tags: [Airflow, Python]
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
1. [Celery Airflow Retry](#celery-airflow-retry)
 - 1) [재시도](#1-재시도)
 - 2) [일정 시간 이후 강제 종료되기](#2-일정-시간-이후-강제-종료되기)
 - 3) [이전 DAG이 실패했을 때 다음 DAG 실행 여부 설정하기](#3-이전-dag이-실패했을-때-다음-dag-실행-여부-설정하기)
 - 4) [DAG이 재시도하거나 실패했을때 이메일 받는 방법](#4-dag이-재시도하거나-실패했을때-이메일-받는-방법)

## Celery Airflow Retry
airflow는 DAG을 실행하는 과정에서 실패 했을 때 재시도 설정하는 것이 
#### 1) 재시도
```python
default_args = {'owner' : 'me',
                'retries' : 5,
                'retry_delay' : timedelta(minutes=5)}

with DAG(dag_id='dag_id',
       default_args=default_args
       schedule_interval="0 0 * * *",
       start_date=datetime(2025, 1, 1),
       tags=['test'],
       catchup=False) as dag:
```
DAG 정의 부분에 위와 같이 작성하면 된다. <br/>
위의 코드는 재시도를 5번 하고, 재시도 사이에 5분 간격을 두고 재시도 설정한 내용이다. <br/>

#### 2) 일정 시간 이후 강제 종료되기
```python
default_args = {'owner' : 'me',
                'retries' : 5,
                'retry_delay' : timedelta(minutes=5),
                'execution_timeout' : timedelta(hours=1)}

with DAG(dag_id='dag_id',
       default_args=default_args
       schedule_interval="0 0 * * *",
       start_date=datetime(2025, 1, 1),
       tags=['test'],
       catchup=False) as dag:
```
`execution_timeout` 옵션을 추가하면 재시도를 5번 하는 도중이라도 1시간 이후 강제 종료된다. <br/>


#### 3) 이전 DAG이 실패했을 때 다음 DAG 실행 여부 설정하기
```python
default_args = {'owner' : 'me',
                'retries' : 5,
                'retry_delay' : timedelta(minutes=5),
                'execution_timeout' : timedelta(hours=1),
                'depends_on_past' : True}

with DAG(dag_id='dag_id',
       default_args=default_args
       schedule_interval="0 0 * * *",
       start_date=datetime(2025, 1, 1),
       tags=['test'],
       catchup=False) as dag:
```
`depends_on_past` 옵션 True로 추가하면 이전 DAG이 실패했을 때 다음 DAG이 실행되지 않는다. <br/>
기본값은 False로 이전 DAG이 실패해도 다음에 실행된다. <br/>

#### 4) DAG이 재시도하거나 실패했을때 이메일 받는 방법
```python
default_args = {'owner' : 'me',
                'retries' : 5,
                'retry_delay' : timedelta(minutes=5),
                'execution_timeout' : timedelta(hours=1),
                'depends_on_past' : True,
                'email_on_failure' : True,
                'email_on_retry' : True}

with DAG(dag_id='dag_id',
       default_args=default_args
       schedule_interval="0 0 * * *",
       start_date=datetime(2025, 1, 1),
       tags=['test'],
       catchup=False) as dag:
```
`email_on_failure` 옵션 True로 추가하면 DAG이 실패했을 때 이메일을 받는다. <br/>
`email_on_retry` 옵션 True로 추가하면 DAG이 재시도 할 때 이메일을 받는다. <br/>
해당 이메일은 airflow.cfg 파일에서 설정한 이메일로 발송된다. <br/>

---

<br/>

다음 글로는 airflow Variables 관련해서 작성하겠습니다. <br/>
