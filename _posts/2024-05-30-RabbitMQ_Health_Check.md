---
layout: post
title: RabbitMQ Health Check
author: 'Juho'
date: 2024-05-30 09:00:00 +0900
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
1. [RabbitMQ Health Check](#rabbitmq-health-check)

## RabbitMQ Health Check
```python
import pika

def rabbitmq_health_check():
  try:
    creds = pika.PlainCredentials(username=RABBITMQ_USER, password=RABBITMQ_PASSWORD)
    params = pika.ConnectionParameters(host=RABBITMQ_HOST, credentials=creds)
    connection = pika.BlockingConnection(params)
    channel = connection.channel()

    if connection.is_open and channel.is_open:
      channel.close()
      connection.close()
      print('Success to connect to RabbitMQ Connection & Channel')
    elif not connection.is_open:
      print('Success to connect to RabbitMQ Connection')
    elif not channel.is_open:
      connection.close()
      print('Success to connect to RabbitMQ Channel')

  except pika.exceptions.AMQPConnectionError as e:
    print(f"Failed to connect to RabbitMQ : {e}")
```