---
layout: post
title: Airflow DAG 삭제 및 중지 시키기
author: 'Juho'
date: 2025-02-08 09:00:00 +0900
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
1. [Airflow DAG 삭제 혹은 중지](#airflow-dag-삭제-혹은-중지)
 - 1) [중지하는 방법](#1-중지하는-방법)
 - 2) [삭제하는 방법](#2-삭제하는-방법)

## Airflow DAG 삭제 혹은 중지
이전 글에서처럼 UI가 아니라 Variables를 이용해서 동적으로 DAG을 생성하는 것 뿐만 아니라 삭제 혹은 중지하는 방법도 필요했다.<br.>

#### 1) 중지하는 방법
```python
from airflow.settings import Session
from airflow.models import DagModel


session = Session()
dag_model = session.query(DagModel).filter(DagModel.dag_id == dag_id).first()
dag_model.is_paused = True
session.commit()
```
데이터베이스 세션을 열고 <br/>
DAG 정보를 저장하는 DagModel에서 dag_id와 일치하는 DAG을 조회하고 <br/>
is_paused 값을 True로 설정하여 일시 중지 상태로 만든 다음 <br/>
변경 사항을 반영한다.<br/>
그럼 특정 DAG이 일시 중지 상태가 되어 새로운 실행이 스케줄링되지 않는다.<br/>
<br/>

#### 2) 삭제하는 방법
```python
from airflow.models.taskinstance import TaskInstance
from airflow.models.dagrun import DagRun
from airflow.settings import Session
from airflow.models import DagModel


session = Session()
session.query(DagRun).filter(DagRun.dag_id == dag_id).delete()
session.query(TaskInstance).filter(TaskInstance.dag_id == dag_id).delete()
session.query(DagModel).filter(DagModel.dag_id == dag_id).delete()
session.commit()
```
데이터베이스 세션을 열고 <br/>
dag_id와 일치하는 모든 DAG 실행(DagRun) 기록을 삭제하고 <br/>
해당 DAG에서 실행된 모든 태스크 실행(TaskInstance) 기록을 삭제하고 <br/>
dag_id에 해당하는 DAG 자체의 메타데이터(DagModel)도 삭제하면 UI 상에서도 사라진다.<br/>
변경 사항을 반영한다.<br/>

DAG 파일이 여전히 디렉토리에 남아있다면 다음 Airflow 재시작시 감지되어 등록될 수 있으나<br/>
나는 동적으로 DAG을 생성했기 때문에 그럴 일 이 없다.<br/>
<br/>


---

<br/>

Celery <-> Airflow 연동과 Airflow에서 Variables를 이용해 동적으로 DAG을 생성, 중지, 삭제하는 것 까지 확인해보았습니다. <br/>
