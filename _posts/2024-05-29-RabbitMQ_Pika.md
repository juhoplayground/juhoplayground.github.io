---
layout: post
title: RabbitMQ with Pika
author: 'Juho'
date: 2024-05-29 09:00:00 +0900
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
1. [Pika?](#pika)
2. [Pika 주요 개념](#pika-주요-개념)
3. [Pika 설치](#pika-설치)
4. [Pika 예제](#pika-예제)


## Pika?
Pika는 Python에서 RabbitMQ와 같은 메세지 브로커와 통신하기 위한 라이브러리로 AMQP를 지원한다.<br/>
이를 통해 메세지를 생성하고 큐에 넣거나, 큐에서 메세지를 소비하는 등의 작업을 수행할 수 있다.<br/>
또한 블로킹(동기) 모드와 논블로킹(비동기) 모드를 모두 지원한다.<br/>


## Pika 주요 개념
1) Connection (연결)
- RabbitMQ 서버와의 네트워크 연결을 의미한다. AMQP 프로토콜을 사용하여 서버와 통신한다.<br/>

2) Channel (채널)
- Connection을 통해 열리는 가상 연결로 대부분의 AMQP 작업은 채널을 통해서 이루어지고, 하나의 Connection에서 여러 개의 채널을 열 수 있다.<br/>

3) Exchange (익스체인지)
- 메세지를 적적한 큐로 라우팅하는 역활을 한다. 익스체인지는 다양한 유형이 있으며, 각 유형은 메세지를 라우팅하는 방식이 다르다.<br/>

4) Queue (큐)
- 메세지가 저장되는 버퍼, 메세지는 큐에 저장되며 컨슈머에 의해 소비된다.<br/>

5) Binding (바인딩)
- Exchange와 Queue를 연결하여 메세지가 큐로 전달되도록 설정한다.<br/>

6) Producer (프로듀서)
- 메세지를 생성하고 Exchange로 전송하는 Application 또는 코드<br/>

7) Consumer (컨슈머)
- Queue에서 메세지를 받아 처리하는 Application 또는 코드<br/>


## Pika 설치
```
pip install pika
```
위의 명령어를 실행하여 설치하면 된다.<br/>

## Pika 예제
1) 메세지 보내기 (Producer)

```python
import pika

creds = pika.PlainCredentials(username=RABBITMQ_USER, password=RABBITMQ_PASSWORD)
params = pika.ConnectionParameters(host=RABBITMQ_HOST, credentials=creds)
connection = pika.BlockingConnection(params)
channel = connection.channel()

channel.queue_declare(queue='test_queue')

channel.basic_publish(exchange='amq.direct',
                      routing_key='test_key',
                      body='Hello, World!')
connection.close()
```

2) 메세지 받기 (Consumer)
```python
import pika

creds = pika.PlainCredentials(username=RABBITMQ_USER, password=RABBITMQ_PASSWORD)
params = pika.ConnectionParameters(host=RABBITMQ_HOST, credentials=creds)
connection = pika.BlockingConnection(params)
channel = connection.channel()

channel.queue_declare(queue='test_queue')
channel.queue_bind(queue='test_queue', exchange="amq.direct", routing_key='test_key')

def callback(ch, method, properties, body):
    print(f" Received {body}")

channel.basic_consume(queue='test_queue', on_message_callback=callback, auto_ack=True)
channel.start_consuming()
connection.close()
```