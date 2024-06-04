---
layout: post
title: RabbitMQ 오류 발생 및 해결
author: 'Juho'
date: 2024-06-04 09:00:00 +0900
categories: [RabbitMQ]
tags: [RabbitMQ, Celery, Python]
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
1. [RabbitMQ 오류 발생 Case1](#rabbitmq-오류-발생-case1)
2. [RabbitMQ 오류 발생 Case2](#rabbitmq-오류-발생-case2)

## RabbitMQ 오류 발생 Case1
첫번째 오류는 `precondition_failed - delivery acknowledgement on channel 1 timed out`였다.<br/>
해당 문제가 발생하는 이유는 RabbitMQ Server가 Producer가 보낸 데이터를 가지고 있는데 <br/>
해당 데이터를 수신할 Consumer가 하나도 없을때 발생한다고 한다.<br/>

Consumer가 수행되지 않아 RabbitMQ Server가 계속 데이터를 가지고 있으면 <br/>
Delivery Acknowledgement Timeout(Default : 30초)동안 보관하고 이후 데이터를 ACK 받은 것으로 처리한다.<br/>
아마 이 떄 저 에러가 발생한 것 같다.<br/>


내가 생각한 해결 방법은 2가지 정도가 있었다.<br/>
첫번째 방법은 <br>
Delivery Acknowledgement Timeout 값을 RabbitMQ Server의 rabbitmq.conf파일에서 30s 보다 큰 값으로 설정하는 것 이다.<br/>
`consumer_timeout = 3600000` <br/>

두번째 방법은 <br/>
Timeout 자체를 Disable 하는 것으로 advanced.config에서 <br/>
```
%% advanced.config
[
  {rabbit, [
    {consumer_timeout, undefined}
  ]}
].
```
와 같이 수정을 하면 된다.<br/>



## RabbitMQ 오류 발생 Case2
두번째 오류는 `precondition_failed - message size 180760424 is larger than configured max size 134217728`였다.<br/>

RabbitMQ의 최대 메세지 용량은  이전에는 2GB였는데, 현재는 128mb라고 한다.<br/>

Celery를 통해서 메세지에 json을 element로 가지는 list를 메세지에 넣어야했는데 <br/>
1일씩 처리할때는 오류가 발생하지 않았는데 1달 기간으로 처리하려니 해당 오류가 발생했다.<br/>
그래서 처음 해결 방법으로 생각한 방식은 1달 기간의 작업을 1일씩 분할해서 처리하는 것이였다.<br/>

첫번째 해결 방법은 1주일 정도 유효했다.<br/>
근데 1일의 데이터가 128mb를 넘게 되어서 동일한 문제를 가지게 되었다.<br/>

그래서 찾아본 두번째 해결 방법은 메세지를 bytes로 압축하여 보내는 것 이다.<br/>

```
import json
import zlib


result = json.dumps(result)
result = zlib.compress(result.encode(encoding='utf-8'))
```

`json.dumps(result)`을 통해서 json 형식의 문자열로 직렬화한다.<br/>
이 후에 `zlib.compress(result.encode(encoding='utf-8'))`을 통해서 json 문자열을 UTF-8인코딩을 사용하여 바이트 문자열로 변환한 뒤 압축하였다.<br/>


이후에 result를 사용해야 할 때는 아래와 같은 방법을 사용했다.<br/>

```
import json
import zlib

report = zlib.decompress(report).decode('utf-8')
report = json.loads(report)
```

`zlib.decompress(report).decode('utf-8')`을 통해서 압축을 해제하고 압축 해제된 바티으 문자열을 UTF-8을 사용하여 문자열로 변환한다.<br/>
이 후에 `json.loads(report)`을 통해서 json 문자열을 list 객체로 역직렬화하였다.<br/>

