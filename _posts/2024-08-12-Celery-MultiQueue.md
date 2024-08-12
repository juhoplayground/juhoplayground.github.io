---
layout: post
title: Celery 다수의 Worker & Queue 사용하는 방법
author: 'Juho'
date: 2024-08-12 09:00:00 +0900
categories: [Celery]
tags: [Celery, Python]
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
1. [Celery Multiple Worker](#celery-multiple-worker)
2. [Celery Multiple Queue](#celery-multiple-queue)

## Celery Multiple Worker
다수의 Worker를 사용하려면<br/>
이전에 `/etc/systemd/system/celery.service`를 만들었는데 해당 파일을 수정해야 한다.<br/>
`/etc/systemd/system/celery-default.service`를 새로 생성한다.<br/>
```
[Unit]
Description=Celery Default Service
After=network.target

[Service]
Type=forking
User=user_name
Group=user_name
Environment="PATH=/home/user_name/project_folder/venv/bin:/usr/bin:/bin:/usr/sbin:/sbin"
WorkingDirectory=/home/user_name/project_folder
ExecStart=/home/user_name/project_folder/venv/bin/celery -A celery_app.celery worker --loglevel=info -f celery.logs -E -n work-default@%h
Restart=always

[Install]
WantedBy=multi-user.target
```

`/etc/systemd/system/celery-other.service`를 새로 생성한다.<br/>
```
[Unit]
Description=Celery Other Service
After=network.target

[Service]
Type=forking
User=user_name
Group=user_name
Environment="PATH=/home/user_name/project_folder/venv/bin:/usr/bin:/bin:/usr/sbin:/sbin"
WorkingDirectory=/home/user_name/project_folder
ExecStart=/home/user_name/project_folder/venv/bin/celery -A celery_app.celery worker --loglevel=info -f celery.logs -E -n work-other@%h
Restart=always

[Install]
WantedBy=multi-user.target
```

이후에 `systemctl daemon-reload`를 실행한 뒤에 <br/>
`systemctl restart celery-default.service`, `systemctl restart celery-other.service`를 각각 실행한다.<br/>
그렇게 하면 하나의 서버에서 `work-default@/root`, `work-other@/root`라는 워커가 생성된 것을 flower에서 확인할 수 있다. <br/>



## Celery Multiple Queue
이제 각각의 Worker가 서로 다른 Queue를 사용하게 설정을 할 것 이다.<br/>

`celery_app.py` 파일을 아래와 같이 수정한다.<br>

```python
from kombu import Exchange, Queue
from celery import Celery

CELERY_QUEUE_DEFAULT = 'default'
CELERY_QUEUE_OTHER = 'other'

celery = Celery(__name__,
                broker=f'amqp://{RABBITMQ_USER}:{RABBITMQ_PASSWORD}@{RABBITMQ_HOST}:{RABBITMQ_PORT}/{RABBITMQ_VHOST}',
                backend='rpc://')
     
celery.conf["task_default_queue"] = CELERY_QUEUE_DEFAULT
celery.conf["task_default_exchange_type"] = 'direct'

celery.conf["task_queues"] = (
    Queue(
        CELERY_QUEUE_DEFAULT,
        Exchange(CELERY_QUEUE_DEFAULT, type='direct'),
        routing_key=CELERY_QUEUE_DEFAULT,
    ),
    Queue(
        CELERY_QUEUE_OTHER,
        Exchange(CELERY_QUEUE_OTHER, type='direct'),
        routing_key=CELERY_QUEUE_OTHER,
    ),
)

celery.conf["task_routes"] = {
    'app.directory_a.function_a': {
        'queue': CELERY_QUEUE_OTHER,
        'routing_key': CELERY_QUEUE_OTHER,
        'exchnage': CELERY_QUEUE_OTHER
    },
    'app.directory_b.function_b': {
        'queue': CELERY_QUEUE_DEFAULT,
        'routing_key': CELERY_QUEUE_DEFAULT,
        'exchnage': CELERY_QUEUE_DEFAULT
    }
}
```
`task_default_queue`로 기본 queue를 `default`로 지정하고 `task_default_exchange_type`를 `direct`로 설정한다.<br/>
`task_default_exchange_type`는 Celery에서 작업 큐를 처리할 때 교환 타입을 설정해서 메시지를 어떻게 분배할지를 결정하는데 사용된다.<br/>
- Direct Exchange: 메시지가 라우팅 키와 일치하는 큐로 직접 전달된다. <br/>
- Topic Exchange: 메시지가 라우팅 키의 패턴과 일치하는 큐로 전달된다. <br/>
- Fanout Exchange: 모든 바인딩된 큐로 메시지를 브로드캐스트된다. <br/>
- Headers Exchange: 메시지 헤더에 기반해 큐로 라우팅한다. <br/>



그 다음 `systemctl restart celery-default.service` 파일을 수정한다.<br/>
```
[Unit]
Description=Celery Default Service
After=network.target

[Service]
Type=forking
User=user_name
Group=user_name
Environment="PATH=/home/user_name/project_folder/venv/bin:/usr/bin:/bin:/usr/sbin:/sbin"
WorkingDirectory=/home/user_name/project_folder
ExecStart=/home/user_name/project_folder/venv/bin/celery -A celery_app.celery worker --loglevel=info -f celery.logs -E -n work-default@%h --queues default --concurrency=4
Restart=always

[Install]
WantedBy=multi-user.target
```

`/etc/systemd/system/celery-other.service`파일도 수정해준다.<br/>
```
[Unit]
Description=Celery Other Service
After=network.target

[Service]
Type=forking
User=user_name
Group=user_name
Environment="PATH=/home/user_name/project_folder/venv/bin:/usr/bin:/bin:/usr/sbin:/sbin"
WorkingDirectory=/home/user_name/project_folder
ExecStart=/home/user_name/project_folder/venv/bin/celery -A celery_app.celery worker --loglevel=info -f celery.logs -E -n work-other@%h --queues other --concurrency=2
Restart=always

[Install]
WantedBy=multi-user.target
```

마찬가지로 `systemctl daemon-reload`를 실행한 뒤에 <br/>
`systemctl restart celery-default.service`, `systemctl restart celery-other.service`를 각각 실행한다.<br/>
그렇게 하고 flower에 들어가서 각 worker를 클릭해서 보면 <br/>
`Pool` 탭에서 `Max concurrency`각 데몬에서 설정한 것 처럼 4, 2로 설정된 것을 확인할 수 있다.<br/>
그리고 `Queue` 탭에서 데몬에서 설정한 Queue의 이름이 보이는 것을 확인할 수 있다.<br/>

celery.conf["task_routes"]에 디렉토리별로 queue, exchange를 설정하면 설정한 값대로 자동으로 큐를 선택한다.<br/>
하지만 task_routes에 지정되지 않는 작업을 특정 큐로 보내고 싶다면 <br/>
task에 `@celery.task(bind=True, max_retries=5, acks_late=True, queue="new", exchange="new")` 이런식으로 직접 설정을 하면 된다. <br/>
이렇게 하면 함수별로 명시적으로 지정이 된다.<br/>

---

<br/>

Celery에서 다수의 Worker & Queue 사용하는 방법하는 방법을 알아봤습니다. <br/>
상황과 서버의 CPU, 메모리 스펙을 고려해서 Worker, Queue, concurrency를 조절해서 사용하면 될 것 같습니다. <br/>