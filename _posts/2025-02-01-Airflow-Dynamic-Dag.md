---
layout: post
title: Airflow 동적으로 DAG 생성하기
author: 'Juho'
date: 2025-02-04 09:00:00 +0900
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
1. [Airflow 동적으로 DAG 생성](#airflow-동적으로-dag-생성)
 - 1) [동적 DAG 생성 코드](#1-동적-dag-생성-코드)

## Airflow 동적으로 DAG 생성
지난 글에서 설정한 Variable을 이용해서 동적으로 DAG을 생성하려고 합니다.<br/>


#### 1) 동적 DAG 생성 코드
```python
from datetime import datetime, timedelta
from airflow.settings import Session
from airflow.models import Variable


session = Session()
variables = session.query(Variable).all()
dag_info_list = {var.key: var.val for var in variables}

def create_dag(dag_id, schedule_interval, default_args, start_date, task_infos):
    with DAG(dag_id=dag_id,
            schedule=schedule_interval,
            default_args=default_args,
            start_date=start_date,
            tags=[task_infos[0]['project']],
            is_paused_upon_creation=False, # unpause 상태로 생성
            catchup=False) as dag:

      task_map = {}
      for task_info in task_infos:
        current_task = PythonOperator(task_id=task_id,
                                      python_callable=test_function1,
                                      op_args=[task_info],
                                      dag=dag,
                                      trigger_rule=TriggerRule.ALL_DONE,
                                      queue="airflow_queue")

        task_map[task_id] = current_task
    
    # dependency 설정
    task_ids = list(task_map.keys())
    for i in range(len(task_ids) - 1):
        task_map[task_ids[i]] >> task_map[task_ids[i + 1]]

    return dag

def main():
  for dag_info in dag_info_list:
    schedule_interval= '0 0 * * *'
    default_args = {'owner': 'owner',
                    'retries': 5,
                    'retry_delay': timedelta(minutes=1),
                    'execution_timeout': timedelta(hours=1),
                    'email_on_failure': True,
                    'email_on_retry': True,
                    'depends_on_past': True}
    start_date = datetime(2025, 1, 1, 0, 0)
    info = Variable.get(dag_info, deserialize_json=True)

    globals()[dag_info] = create_dag(dag_info, schedule_interval, default_args, start_date, info)

main()
```


---

<br/>

다음 글에서는 생성한 DAG을 중지, 삭제 하는 방법에 대해서 알아보겠습니다.<br/>
