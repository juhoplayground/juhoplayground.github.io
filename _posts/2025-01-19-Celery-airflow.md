---
layout: post
title: Celery Airflow
author: 'Juho'
date: 2025-01-19 09:00:00 +0900
categories: [Celery]
tags: [Celery, Airflow, Python]
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
1. [Celery Airflow](#celery-airflow)
 - 1) [airflow 데이터베이스 초기화](#1-airflow-데이터베이스-초기화)
 - 2) [airflow 관리자 계정 생성](#2-airflow-관리자-계정-생성)
 - 3) [airflow. cfg 설정하기](#3-airflow-cfg-설정하기)
 - 4) [airflow 실행](#4-airflow-실행)
 - 5) [airflow dag 생성](#5-airflow-dag-생성)

## Celery Airflow
[Airflow Celery Executor](https://airflow.apache.org/docs/apache-airflow-providers-celery/stable/index.html){:target="_blank"}에 라이브러리 버전 요구사항을 확인해야한다.<br/>

공식 문서대로 설치한다.
그전에 AIRFLOW_HOME 환경변수를 설정한다. `export AIRFLOW_HOME=~/airflow`<br/>
`pip install "apache-airflow[celery]==2.10.4" --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-2.10.4/constraints-3.8.txt"`
airflow가 설치되었는지 `airflow version`으로 확인한다.<br/>

#### 1) airflow 데이터베이스 초기화
`airflow db migrate` 명령어를 실행한다.<br/>
다른 블로그에서는 `airflow db init` 명령어를 사용하는데 버전 업데이트 되면서 init 함수는 deprecated 되었다.<br/>

#### 2) airflow 관리자 계정 생성
```
airflow users create \
    --username admin \
    --firstname name \
    --lastname name \
    --role Admin \
    --email admin@example.com
```
user 생성이 잘 되었는지 확인하려면 `airflow users list` 명령어를 실행한다.<br/>

#### 3) airflow. cfg 설정하기
airflow.cfg 파일에서 executor를 Celery로 변경한다.<br/>
```
executor = CeleryExecutor
```

그리고 broker_url과 result_backend를 설정한다.<br/>
```
broker_url = amqp://user:id@localhost:port/vhost
result_backend = redis://:id@localhost:port/0
```

Celery Executor를 사용하려면 mysql이나 postgresql을 사용해야한다.<br/>
postgresql을 설치한다.<br/>
```
sudo apt update
sudo apt install postgresql postgresql-contrib
pip install psycopg2-binary
sudo -u postgres psql
```
데이터베이스 및 사용자 생성해준다.<br/>
```sql
CREATE DATABASE airflow_db;
CREATE USER airflow_user WITH PASSWORD 'your_password';
ALTER ROLE airflow_user SET client_encoding TO 'utf8';
ALTER ROLE airflow_user SET default_transaction_isolation TO 'read committed';
ALTER ROLE airflow_user SET timezone TO 'Asia/Seoul';
GRANT ALL PRIVILEGES ON DATABASE airflow_db TO airflow_user;
```

그리고 나서 다시 airflow.cfg 파일을 열고 수정한다.<br/>
`sql_alchemy_conn = postgresql+psycopg2://airflow_user:your_password@localhost/airflow_db`

airflow 시간대를 `default_timezone = Asia/Seoul`로 변경한다.<br/>

그리고 새 데이터베이스로 마이그레이션을 초기화한다.<br/>
`ariflow db migrate`

#### 4) airflow 실행
```
airflow scheduler -D
airflow webserver -p 8080 -d
airflow celery worker -D -q airflow_queue
```
`localhost:8080`에 접속해서 접속이 잘 되는지 확인하면 된다.<br/>
`localhost:5555`에 접속해서 flower에서 celery worker와 queue를 확인해본다.<br>
`localhost:15672`에 접속해서 rabbitmq에서 queue를 확인해본다.<br/>
이상이 없으면 잘 생성이 되었다.<br/>

나는 `-D` 옵션을 줘서 데몬으로 실행해도 되지만 systemd 서비스 파일을 만들기로 했다.<br/>
1) airflow scheduler
```
[Unit]
Description=Airflow Scheduler Daemon
After=network.target

[Service]
User={user}
Group={user}
WorkingDirectory=/home/{user}/{project_folder}
Environment="PATH=/home/{user}/{project_folder}/venv/bin:/usr/bin:/bin"
ExecStart=/home/{user}/{project_folder}/venv/bin/airflow scheduler
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

2) airflow celery worker
```
[Unit]
Description=Airflow Worker Daemon
After=network.target

[Service]
User={user}
Group={user}
WorkingDirectory=/home/{user}/{project_folder}
Environment="PATH=/home/{user}/{project_folder}/venv/bin:/usr/bin:/bin"
ExecStart=/home/{user}/{project_folder}/venv/bin/airflow celery worker -q airflow_queue
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

3) airflow webserver
```
[Unit]
Description=Airflow Webserver Daemon
After=network.target

[Service]
User={user}
Group={user}
WorkingDirectory=/home/{user}/{project_folder}
Environment="PATH=/home/{user}/{project_folder}/venv/bin:/usr/bin:/bin"
ExecStart=/home/{user}/{project_folder}/venv/bin/airflow webserver -p 8080
Restart=always
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

이 후 작성한 서비스를 systmed에 등록한다.<br/>
`sudo systemctl daemon-reload`

서버 재부팅 시 자동으로 실행되도록 설정한다.<br/>
```
sudo systemctl enable airflow-scheduler
sudo systemctl enable airflow-celery-worker
sudo systemctl enable airflow-webserver
```

각 데몬을 실행한다.<br/>
```
sudo systemctl start airflow-scheduler
sudo systemctl start airflow-celery-worker
sudo systemctl start airflow-webserver
```

서비스 상태를 확인한다.<br/>
```
sudo systemctl status airflow-scheduler
sudo systemctl status airflow-celery-worker
sudo systemctl status airflow-webserver
```

로그를 확인하고 싶으면 아래 명령어를 실행한다.<br/>
```
sudo journalctl -u airflow-scheduler -f
sudo journalctl -u airflow-celery-worker -f
sudo journalctl -u airflow-webserver -f
```

#### 5) airflow dag 생성
```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta
from random import randrange
import sys

# Python Path 추가
sys.path.append('/home/{user}/{project_folder}')

# Celery 작업 가져오기
from app.test.task import add

# Celery 작업을 호출하는 래퍼 함수
def trigger_celery_task(**kwargs):
    a= randrange(10)
    b = randrange(10)
    result = add.delay(a, b)  # Celery 작업 호출
    return result.id  # 작업 ID 반환

# DAG 정의
with DAG(
    dag_id='celery_integration_dag',
    schedule_interval=timedelta(minutes=1),  # 1분마다 실행
    start_date=datetime(2025, 1, 1),
    tags=['example'],
    catchup=False
) as dag:
    
    celery_task = PythonOperator(
        task_id='trigger_celery_task',
        python_callable=trigger_celery_task,
        queue="airflow_queue"
    )

celery_task
```

---

<br/>

다음 글로는 airflow retry 관련해서 작성하겠습니다. <br/>
