---
layout: post
title: Celery Retry
author: 'Juho'
date: 2024-08-07 09:00:00 +0900
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
1. [Celery 재시도](#celery-재시도)

## Celery 재시도
```
from celery.exceptions import MaxRetriesExceededError, WorkerShutdown, WorkerLostError

@celery.task(bind=True, max_retries=5, acks_late=True)
def some_task(self, params):
  try:
    do-something
  except MaxRetriesExceededError as e:
    self.update_state(state='FAILURE', meta={'exc_type': type(e).__name__, 'exc_message': e.__str__()})
  except WorkerLostError as e:
    countdown = (2 * (self.request.retries + 1)) * 5
    raise self.retry(exc=e, countdown=countdown)
  except WorkerShutdown as e:
    countdown = (2 * (self.request.retries + 1)) * 5
    raise self.retry(exc=e, countdown=countdown)
  except Exception as e:
    countdown = (2 * (self.request.retries + 1)) * 5
    raise self.retry(exc=e, countdown=countdown)
```

`bind=True`는 작업이 `self` 인스턴스를 사용할 수 있도록 한다.<br/>
즉, 작업 메소드가 클래스의 인스턴스 메소드처럼 동작하게 한다.<br/>
이를 통해 작업의 메소드 내부에서 작업 인스턴스(self)를 통해 작업 상태나 메타데이터에 접근할 수 있다.<br/>
<br/>

`max_retries=5`는 이 옵션은 작업이 실패했을 때 최대 재시도 횟수를 설정한다.<br/>
<br/>

`acks_late=True`는  작업이 완료된 후에만 작업 큐에 ACK(acknowledgement)를 보낸다.<br/>
기본적으로 Celery는 작업을 처리한 후 즉시 ACK를 보내어 작업이 성공적으로 처리되었음을 큐에 알린다.<br/>
. acks_late=True는 작업이 성공적으로 완료될 때만 ACK를 보내므로, 작업이 실패하거나 예외가 발생하면 메시지가 다시 큐에 돌아가 재처리될 수 있도록 한다.<br/>
<br/>

`MaxRetriesExceededError`는 Celery 작업이 설정된 최대 재시도 횟수를 초과했을 때 발생한다.<br/>
이 예외는 작업이 재시도 한계를 넘어서서 실패했음을 나타낸다.<br/>
`@celery.task(bind=True, max_retries=5, acks_late=True)`에서 max_retries를 5로 설정했기 때문에<br/>
5회를 초과하면 해당 내용의 오류가 발생하게 되고 더 이상 재시도 하지 않고 `FAILURE` 처리를 한다.<br/>

`WorkerLostError`는 일반적으로 워커 프로세스가 작업을 완료하지 못하고 중간에 사라졌을 때 발생한다.<br/>
1) 메모리 부족 : 작업을 수행하는 동안 워커가 과도한 메모리를 사용하면 운영 체제가 워커 프로세스를 강제로 종료할 수 있다. <br/>
2) 시간 초과 : 작업이 지정된 시간 내에 완료되지 않으면 Celery가 워커를 강제로 종료시킬 수 있다.<br/>
-> `soft time limit`이나 `hard time limit`이 원인일 수 있다.<br/>
3) 프로세스 충돌 : 워커 프로세스가 충돌하거나 비정상적으로 종료되면, Celery는 작업이 사라진 것으로 간주하고 WorkerLostError를 발생한다.<br/>
4) 워커 재시작 : Celery 워커 프로세스가 수동 또는 자동으로 재시작될 때, 현재 실행 중이던 작업이 중단되면서 이 오류가 발생할 수 있다.<br/>
5) 네트워크 문제 : 분산된 Celery 환경에서 워커와 브로커(예: Redis, RabbitMQ) 간의 네트워크 연결이 끊기면 워커가 작업을 완료하지 못하고 이 오류가 발생할 수 있다.<br/>

 
`WorkerShutdown`는 Celery 워커가 정상적으로 종료되었을 때 발생한다.<br/>
워커가 의도적으로 종료되었으며, 시스템이 이를 인식하고 있는 상황이다.<br/>
하지만 `SIGTERM` 신호에 의해 종료되는 것을 막고자 추가했다.<br/>


---

<br/>

Celery task 재시도하는 방법을 작성했는데<br/>
혹시 잘못된 부분이나 더 나은 방법이 있다면 알려주시면 감사하겠습니다.<br/>