---
layout: post
title: Celery  Worker가 주기적 Sync가 재시작되는 문제 확인 
author: 'Juho'
date: 2025-07-01 09:00:00 +0900
categories: [Celery]
tags: [Celery, RabbitMQ]
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
1. [mingle: sync이 반복됨](#mingle-sync이-반복됨)
2. [heartbeat 문제인가?](#heartbeat-문제인가)
3. [진짜 문제는...](#진짜-문제는)

## mingle: sync이 반복됨
```
[2025-06-23 09:05:47,325: INFO/MainProcess] mingle: searching for neighbors
[2025-06-23 09:05:48,374: INFO/MainProcess] mingle: sync with 4 nodes
[2025-06-23 09:05:48,374: INFO/MainProcess] mingle: sync complete
```
celery.logs에서 위의 내용이 지속적으로 반복되는 현상을 확인했다.  

## heartbeat 문제인가?  
`grep "closing AMQP connection" /var/log/rabbitmq/*`을 확인해보니  
```
/var/log/rabbitmq/rabbit@ubuntu.log:2025-06-23 06:23:00.288055+09:00 [warning] <0.3241533.0> closing AMQP connection <0.3241533.0> (10.0.2.22:57550 -> 10.0.2.21:5672, vhost: 'myvhost', user: 'ubuntu', duration: '1M, 29s'):
/var/log/rabbitmq/rabbit@ubuntu.log:2025-06-23 06:23:14.211604+09:00 [info] <0.3241697.0> closing AMQP connection (10.0.2.21:55570 -> 10.0.2.21:5672, vhost: 'myvhost', user: 'ubuntu', duration: '
```
그래서 hearbeat문제가 있는지 확인하기로 했다.  

우선 `--without-heartbeat` 옵션을 사용하고 있었나? 싶어서 `ps aux | grep celery`로 확인해보니 그렇지는 않았다.  

그 다음은 `sudo rabbitmqctl environment | grep heartbeat`로 heartbeat을 확인해봤다.  
```
{heartbeat_interval,100},
      {heartbeat,60}
```
그래서 celery 설정을 아래와 같이 변경해서 heartbeat를 맞추기로 했다.  
```python
app.conf.broker_transport_options = {
    'heartbeat': 60,
}
```
그런데도 여전히 Celery Worker가 주기적으로 재시작되는 문제가 반복되어서 heartbeat가 문제가 아니라고 생각이 들었다.  

## 진짜 문제는...
`celery.logs`에 Error trace가 있는지 다시 확인해봤다.  
```
grep -i error celery.logs
grep -i exception celery.logs
```
Error trace는 검색되지 않았다.  

그래서 Celery Worker가 시스템에 의해서 죽는지 아래의 명령어로 확인을 해봤다.  
```
dmesg | grep -i kill
```

그랬더니
```
oom-kill: ... task=node, pid=..., UID=1000
Out of memory: Killed process ... (node) ...
```

OOM 이슈로 Celery Worker가 계속 죽고 systemd 설정에 의해서 계속 restart되고 있었다;;


---  

메모리를 증량해서 다행히 쉽게 해결했다.  
