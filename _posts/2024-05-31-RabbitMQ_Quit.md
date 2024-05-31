---
layout: post
title: RabbitMQ에서 특정 작업 중단
author: 'Juho'
date: 2024-05-31 09:00:00 +0900
categories: [RabbitMQ]
tags: [RabbitMQ, Python]
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
1. [RabbitMQ 특정 작업 중단](#rabbitmq-특정-작업-중단)

## RabbitMQ 특정 작업 중단
```python
import pika

def rabbitmq_quit_task(queue_name, condition):
  try:
    check_list = []
    creds = pika.PlainCredentials(username=RABBITMQ_USER, password=RABBITMQ_PASSWORD)
    params = pika.ConnectionParameters(host=RABBITMQ_HOST, credentials=creds)
    connection = pika.BlockingConnection(params)
    channel = connection.channel()
    channel.queue_declare(queue=queue_name, passive=True, durable=True)


    while True:
      method_frame, header_frame, body  = channel.basic_get(queue=queue_name, auto_ack=False)
      if method_frame:
        message_id = method_frame.delivery_tag
        message_body = body.decode('utf-8')
        if message_body in check_list:
          pass
        else:
          check_list.append(message_body)
          if condition:
            channel.basic_ack(delivery_tag=message_id)
          else:
            channel.basic_nack(delivery_tag=message_id, requeue=True)
      else:
        break

  except pika.exceptions as e:
    print(f"Failed to Quit RabbitMQ Task : {e}")
  else:
    channel.close()
    connection.close()
```

`channel.basic_get(queue=queue_name, auto_ack=False)`으로 Queue에서 하나의 메세지씩 가져옴<br/>
메세지에 큐가 없으면 None을 반환하므로<br/>
`if method_frame`를 통해서 method_frame가 존재하면 while문을 반복하고 <br/> 존재하지 않으면 break를 통해서 반복문을 빠져나옴 <br/>

`method_frame.delivery_tag`을 통해서 message의 id값을 받고 <br/>
`body.decode('utf-8')`을 통해서 message의 body 값을 받음 <br/>

message_body값이 check_list에 포함되어 있으면 pass 하고<br/>
포함되어 있지 않다면 check_list에 포함을 시킴 <br/>

입력한 condition을 만족하는 경우에는 <br/>`channel.basic_ack(delivery_tag=message_id)`을 통해서 <br/>
메세지를 정상적으로 처리되었다고 하여 메세지를 큐에서 제거함<br/>
<br/>
만족하지 않는 경우에는 `channel.basic_nack(delivery_tag=message_id, requeue=True)`을 통해서 <br/>
메세지가 처리되지 않았다고 하고 `requeue=True` 옵션을 사용하여<br/>다시 메세지를 Queue에 넣어 메세지가 다른 Consumer에 의해 처리될 수 있도록 함<br/>
<br/>

---

<br/>

RabbitMQ의 Queue에 존재하는 작업을 삭제하는 혹시 더 나은 방법이 있다면 알려주시면 감사하겠습니다.<br/>