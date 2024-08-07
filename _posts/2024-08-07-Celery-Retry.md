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
2. [Celery 설치](#celery-설치)
3. [Celery Redis 연결](#celery-redis-연결)
4. [Celery Daemon](#celery-daemon)
5. [Celery 예시](#celery-예시)

## Celery 재시도
```
from celery.exceptions import MaxRetriesExceededError, WorkerShutdown

@celery.task(bind=True, max_retries=5, acks_late=True)
def some_task(self, params):
  try:
    do-something
  except MaxRetriesExceededError as e:
    self.update_state(state='FAILURE', meta={'exc_type': type(e).__name__, 'exc_message': e.__str__()})
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
 
`WorkerShutdown`는 Celery 워커가 종료되었을 때 발생한다. 워커 프로세스가 중지되거나 재시작되는 경우에도 이 예외가 발생할 수 있다.<br/>
워커 프로세스가 중지되거나, 재시작되어도 다시 해당 작업을 재시도한다. <br/>


---

<br/>

Celery task 재시도하는 방법을 작성했는데<br/>
혹시 잘못된 부분이나 더 나은 방법이 있다면 알려주시면 감사하겠습니다.<br/>