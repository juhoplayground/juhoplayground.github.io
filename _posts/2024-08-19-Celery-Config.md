---
layout: post
title: Celery Configuration
author: 'Juho'
date: 2024-08-19 09:00:00 +0900
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
1. [Celery Configuration](#celery-configuration)
 - 1) [enable_utc](#1-enable_utc)
 - 2) [timezone](#2-timezone)
 - 3) [broker_connection_retry_on_startup](#3-broker_connection_retry_on_startup)
 - 4) [task_time_limit](#4-task_time_limit)
 - 5) [task_soft_time_limit](#5-task_soft_time_limit)
 - 6) [result_backend](#6-result_backend)
 - 7) [result_persistent](#7-result_persistent)
 - 8) [worker_prefetch_multiplier](#8-worker_prefetch_multiplier)
 - 9) [task_reject_on_worker_lost](#9-task_reject_on_worker_lost)
 - 10) [task_acks_on_failure_or_timeout](#10-task_acks_on_failure_or_timeout)
 - 11) [task_acks_late](#11-task_acks_late)
 - 12) [accpet_content](#12-accpet_content)
 - 13) [task_serializer](#13-task_serializer)
 - 14) [result_serializer](#14-result_serializer)
 - 15) [worker_send_task_events](#15-worker_send_task_events)

`celery_app.py`에 꽤 많은 conf가 추가되어 있어서 해당 내용들이 무슨 내용인지 정리해보고자 한다.<br/>
이전 글에서 설명된 conf들은 생략한다.<br/>

## Celery Configuration
#### 1) enable_utc
celery.conf["enable_utc"] = False <br/>
기본적으로 Celery는 UTC를 사용하여 모든 시간을 처리하는데, 이 설정을 False로 변경하면 사용자가 지정한 로컬 타임존을 사용하게 된다.<br/>

#### 2) timezone
celery.conf["timezone"] = 'Asia/Seoul' <br/>
Celery에서 task를 예약하거나 스케줄링할 때, 특정 시간대를 기준으로 작업을 수행하게 된다.<br/>
그래서 서버가 위치한 타임존과 Celery의 작업 타임존이 일치하지 않으면 작업이 예상과 다른 시간에 실행될 수 있다.<br/>

#### 3) broker_connection_retry_on_startup
celery.conf["broker_connection_retry_on_startup"] = True <br/>
Celery가 시작될 때 브로커(broker)와의 연결에 실패하더라도 자동으로 연결될 때까지 일정 간격으로 연결 재시도를 하도록 한다.<br/>
메시지 브로커는 Celery의 핵심 구성 요소로, 작업 큐잉과 메시징을 담당한다.<br/>
브로커에 연결할 수 없는 상황이 발생할 경우, Celery는 정상적으로 작동하지 않을 수 있다.<br/>

#### 4) task_time_limit
celery.conf["task_time_limit"] = 600 <br/>
Celery에서 작업(task)이 수행될 수 있는 최대 시간을 지정한다.<br/>
이 설정은 개별 작업이 실행될 수 있는 최대 시간을 초 단위로 지정한다.<br/>
설정된 시간이 지나도 작업이 완료되지 않으면 작업이 강제로 중단된다.<br/>
작업이 무한정 실행되는 것을 방지하여 시스템 리소스를 보호할 수 있다.<br/>
논리적 오류나 외부 의존성 문제로 인해 작업이 예상보다 오래 실행되는 경우, 작업을 제한 시간 내에 강제로 종료함으로써 시스템 성능을 저하시킬 수 있는 상황을 방지할 수 있다.<br/>

#### 5) task_soft_time_limit
celery.conf["task_soft_time_limit"] = 300 <br/>
Celery에서 작업이 실행될 때, 지정된 시간 내에 작업이 종료되도록 경고를 보내는 역할을 한다.<br/>
이 설정은 task_time_limit과 유사하지만, 강제 종료 대신 작업이 스스로 종료할 수 있는 기회를 제공한다.<br/>
이 신호를 받으면 작업은 현재 진행 중인 작업을 완료하고, 가능한 빨리 종료되도록 설계되어야 한다.<br/>
작업이 종료될 시간을 미리 알고 있다면, 현재 상태를 저장하거나 적절하게 종료 절차를 밟을 수 있습니다. 이로 인해 데이터 손실이나 시스템 불안정성을 줄일 수 있다.<br/>

#### 6) result_backend
celery.conf["result_backend"] = 'rpc://' <br/>
Celery가 작업의 결과를 저장하거나 조회할 수 있는 위치를 지정한다.<br/>
백엔드는 Celery가 작업의 결과를 유지하고, 클라이언트가 이를 조회할 수 있도록 하는 데 사용된다.<br/>
비동기 작업을 실행하고 그 결과를 나중에 참조해야 하는 경우, 적절한 결과 백엔드를 설정해야 한다.<br/>

#### 7) result_persistent
celery.conf["result_persistent"] = True <br/>
Celery의 결과 백엔드에 저장된 작업 결과가 영구적으로 유지되도록 하는 옵션이다.<br/>
Celery가 재시작되거나 브로커가 재시작되더라도 작업 결과는 유지된다.<br/>
작업 결과를 영구적으로 보존할 필요가 있는 경우 유용하다.<br/>
예를 들어, 작업의 결과를 장기적으로 저장하거나, Celery가 재시작되더라도 결과를 유지하고 싶을 때 이 설정을 사용한다.<br/>
작업 결과를 지속적으로 저장함으로써 디버깅과 문제 추적을 용이하게 할 수 있다. <br/>

#### 8) worker_prefetch_multiplier
celery.conf["worker_prefetch_multiplier"] = 1 <br/>
Celery 워커가 동시에 가져오는 작업 수를 조절하는 옵션으로 워커가 큐에서 작업을 얼마나 미리 가져오는지에 대한 제어를 제공한다.<br/>
1로 설정하면 각 워커가 동시 작업을 최소화하여, 작업이 완료될 때까지 다른 작업을 가져오지 않도록 한다.<br/>
프리패치 값을 줄이면, 워커가 동시에 가져오는 작업 수가 줄어들어, 시스템 리소스를 보다 효율적으로 관리할 수 있다.<br/>
1로 설정할 경우 작업이 차례로 처리되도록 보장하지만, 병렬 처리가 줄어들 수 있습니다. <br/>
또한 작업이 큐에서 대기하는 시간이 증가할 수 있다.<br/>
높은 숫자로 설정할 경우 병렬 처리 성능을 높일 수 있지만, 시스템 리소스 사용량이 증가할 수 있다. <br/>
따라서 시스템의 요구 사항과 워커의 처리 능력에 따라 적절한 값을 설정하는 것이 중요하다. <br/>

#### 9) task_reject_on_worker_lost
celery.conf["task_reject_on_worker_lost"] = True <br/>
Celery 워커가 작업을 수행하는 동안 워커가 예기치 않게 종료되거나 연결이 끊길 경우, 해당 작업을 어떻게 처리할지를 지정하는 옵션이다.<br/>
True로 설정하면, 워커가 예기치 않게 종료되거나 연결이 끊길 경우, 그 워커가 처리 중이던 작업이 자동으로 해당 작업이 실패로 간주되어 다른 워커가 다시 가져가서 처리하게 된다.<br/>
워커가 작업을 처리하는 동안 문제가 발생해 워커가 종료되면, 작업이 손실되지 않고 다른 워커가 다시 처리할 수 있도록 보장한다.<br/>
이를 통해 시스템의 안정성을 높이고, 작업이 중단되거나 누락되는 것을 방지할 수 있다.<br/>
작업이 한 워커에만 의존하지 않고, 여러 워커가 작업을 처리할 수 있도록 보장하여 작업의 신뢰성을 높인다.<br/>
하지만 작업 재시도 로직이 별도로 설정되어 있어야 작업이 성공적으로 재처리될 수 있다.<br/>
해 task_acks_on_failure_or_timeout, task_retries, task_retry 등의 다른 설정을 조정해야한다.<br/>

#### 10) task_acks_on_failure_or_timeout
celery.conf["task_acks_on_failure_or_timeout"] = True <br/>
작업이 실패하거나 타임아웃이 발생했을 때, 작업에 대한 ack(확인 응답)를 보내는 방식을 결정한다.<br/>
ack는 작업이 성공적으로 처리되었음을 브로커에게 알리는 신호다.<br/>
작업이 실패하거나 타임아웃이 발생했을 때, Celery는 작업을 실패로 간주하고 ack를 보내지 않으며, 작업이 다시 큐로 돌아가도록 설정된다.<br/>
이를 통해 작업이 재시도되거나 다른 워커에 의해 처리될 수 있다.<br/>

#### 11) task_acks_late
celery.conf["task_acks_late"] = True <br/>
Celery 작업이 처리 완료 후에만 ack(확인 응답)를 보내도록 설정한다.<br/>
True로 설정하면, Celery는 작업을 성공적으로 처리한 후에만 브로커에 ack를 보낸다.<br/>
작업이 완료되지 않거나 실패할 경우, ack는 전송되지 않으며, 작업이 다시 큐로 돌아가서 다른 워커가 이를 재처리할 수 있다.<br/>

#### 12) accpet_content
celery.conf["accpet_content"] = ['application/json'] <br/>
Celery 작업이 수신할 수 있는 메시지의 콘텐츠 유형을 제어한다.<br/>
['application/json']로 설정하면 메시지의 콘텐츠 유형이 application/json인 경우에만 메시지를 수신하고 처리한다.<br/>
다른 콘텐츠 유형의 메시지는 무시한다.<br/>
작업에 전달되는 데이터의 형식을 일관되게 유지할 수 있다.<br/>
수신하는 데이터 형식을 제한하여 예상하지 못한 데이터 형식으로 인한 오류나 보안 문제를 줄일 수 있다.<br/>
잘못된 데이터 형식으로 인한 처리 오류를 방지하고, Celery 작업이 예상한 데이터 형식만 처리하도록 보장한다.<br/>

#### 13) task_serializer
celery.conf["task_serializer"] = 'json' <br/>
Celery 작업의 메시지를 직렬화할 때 사용할 직렬화 방식을 지정한다.<br/>
직렬화는 작업 데이터와 결과를 브로커를 통해 전송하기 위해 데이터를 바이트 스트림으로 변환하는 과정이다.<br/>
다른 직렬화 방식도 있다.
1) pickle : Python 객체를 직렬화하는 데 사용되는 기본 직렬화 방식<br/>
2) yaml : YAML 형식으로 직렬화할 수 있으며, 사람이 읽기 쉬운 데이터 형식<br/>
3) msgpack : 빠르고 효율적인 직렬화 형식으로, 이진 데이터 포맷<br/>
accpet_content 설정과 일치해야한다.<br/>

#### 14) result_serializer
celery.conf["result_serializer"] = 'json' <br/>
elery 작업의 결과를 직렬화할 때 사용할 직렬화 방식을 지정한다.<br/>
`task_serializer` 옵션처럼 다른 직렬화 방식도 지원하고 마찬가지로 accpet_content 설정과 일치해야한다.<br/>

#### 15) worker_send_task_events
celery.conf["worker_send_task_events"] = True <br/>
Celery 워커가 작업의 상태 변경 이벤트를 브로커에 전송할지 여부를 결정한다.<br/>
워커가 작업의 상태 변경 이벤트(예: 시작, 성공, 실패 등)를 브로커에 보고하는 기능을 활성화한다.<br/>
작업의 진행 상황을 모니터링하는 데 유용해진다.<br/>
이벤트 전송은 네트워크 및 브로커에 추가 부하를 줄 수 있다.<br/>
시스템의 성능 요구 사항에 따라 이 설정을 조정해야 할 수 있다.<br/>



---

<br/>

celery에 추가되어 있는 configuration을 확인해보았다. <br/>
제가 놓친 중요한 configuration이 있다면 알려주시면 감사하겠습니다. <br/>
