---
layout: post
title: Python 병렬 프로그래밍 - (2) multiprocessing
author: 'Juho'
date: 2024-08-21 09:00:00 +0900
categories: [Python]
tags: [Python, multiprocessing]
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
1. [process란?](#process란)
2. [process 장,단점](#process-장단점)
 - 1) [장점](#1-장점)
 - 2) [단점](#2-단점)
3. [process 상태 정의](#process-상태-정의)
4. [multiprocessing 모듈 사용하는 방법](#multiprocessing-모듈-사용하는-방법)
5. [ProcessPoolExecutor 사용하는 방법](#processpoolexecutor-사용하는-방법)


## process란?
프로세스는 실행 중인 프로그램의 인스턴스로, 운영 체제에서 프로그램이 실행될 때 독립된 주소 공간을 가지며 실행되는 단위다.<br/>
프로세스는 프로그램과 관련된 데이터 영역, 자식 프로세스, 주소 공간, 그리고 다른 프로세스와의 통신 등을 관리할 수 있다.<br/>
프로세스는 조작과 제어가 가능한 정보 및 자원과 관련이 있다.<br/> 
운영체제는  프로세스와 관련된 정보를 저장한 프로세스 제어 블록이라고 하는 구조체를 갖는다.<br/> 
PCB는 아래와 같은 정보 등을 저장한다.<br/> 
- 프로세스 상태 : 프로세스가 준비, 대기, 실행 중인지의 여부<br/> 
- 프로세스 식별자: 운영체제에서 프로세스를 식별할 수 있는 고유 정수 값<br/> 
- 프로그램 카운터 : 실행할 다음 프로그램 명령의 주소, 프로그램 카운터는 컨텍스트 스위칭에 중요한 역할을 함<br/> 
- CPU 레지스터 : 프로세스가 실행될 때 사용되는 레지스터의 값(일반 목적 레지스터, 인덱스 레지스터, 스택 포인터 등)<br/> 
- 메모리 할당 : 프로세스가 사용하는 메모리 공간과 프로세스와 페이지 테이블, 세그먼트 테이블, 베이스와 리미트 레지스터 정보 등<br/> 
- 계정 정보 : 프로세스의 CPU 사용 시간, 실제 사용 시간, 프로세스 생성 시간 등<br/> 
- I/O 정보 : 프로세스와 관련된 파일 디스크립터, 열린 파일 목록, I/O 장치의 상태 등<br/> 
- 프로세스 우선순위 : 프로세스 우선순위에 관한 정보와 큐의 해당 프로세스 관련 정보<br/> 
- 프로세스 연관 정보 : 부모 프로세스와 자식 프로세스에 대한 정보, 프로세스 간의 관계를 관리하는데 사용됨<br/> 
- 신호 처리 정보 : 프로세스가 받는 신호와 그 신호에 대한 처리 방법<br/> 


## process 장,단점
#### 1) 장점
- 여러 프로세스를 사용하면 여러 CPU 코어에서 병렬로 작업을 수행할 수 있는데 Python의 GIL으로 인한 제한 없이 CPU 집약적인 작업을 병렬로 수행할 수 있다.<br/>
- 각 프로세스는 독립적인 메모리 공간을 가지고 있어, 하나의 프로세스에서 문제가 발생(메모리 손상, 비정상적인 동작이)하더라도 다른 프로세스로 전파될 위험이 적어 안정성이 높아진다.<br/>
- 프로세스 간에는 메모리를 공유하지 않기 때문에, thread에서 발생할 수 있는 자원 경쟁이나 데드락(deadlock) 문제를 피할 수 있다.<br/>

#### 2) 단점
- 각 프로세스가 독립된 메모리 공간을 가지므로, 여러 프로세스를 실행할 때 메모리 사용량이 크게 증가할 수 있다.<br/>
- 프로세스 간에 데이터를 주고받으려면 IPC(Inter-Process Communication) 메커니즘을 사용해야 하는데 파이프, 큐, 공유 메모리 등 다양한 방법이 있지만, 스레드 간의 메모리 공유보다 복잡하고 느릴 수 있다. <br/>
- 새로운 프로세스를 생성하는 것은 thread 생성보다 비용이 많이 든다. 그래서 많은 수의 프로세스를 생성하는 것은 비효율적일 수 있다.<br/>


## process 상태 정의
프로세스 상태는 운영체제에서 프로세스가 현재 어떤 단계에 있는지를 나타내는 것으로, 프로세스의 생명 주기 동안 다양한 상태로 전환된다.<br/>
- 생성<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;프로세스가 생성되고 있는 상태로 이 단계에서 운영체제는 필요한 자원을 할당<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;PCB를 생성하는 등의 작업을 수행<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;아직 실행 준비가 되지 않았기 때문에 CPU에 할당되지 않은 상태<br/>

- 준비<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;프로세스가 실행될 준비가 된 상태<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;프로세스는 실행되기 위해 CPU 할당을 기다리고 있음<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;준비 큐에 들어가서 CPU가 이 프로세스를 선택할 때까지 대기<br/>

- 실행 중<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;프로세스가 CPU를 점유하고 실제로 명령어를 실행 중인 상태<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CPU가 프로그램 카운터에 의해 지정된 명령어를 실행<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;시스템에서는 동시에 여러 프로세스가 실행될 수 있으나, CPU 코어당 하나의 프로세스만이 실행 상태<br/>

- 대기<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;프로세스가 어떤 이벤트(I/O 작업의 완료, 특정 자원의 사용 가능)가 발생하기를 기다리는 상태<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;이 상태에서는 CPU를 사용하지 않으며, 이벤트가 발생하면 다시 준비 상태로 돌아감<br/>

- 종료<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;프로세스가 모든 작업을 완료하고 종료된 상태<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;이 상태에서는 더 이상 CPU를 사용할 수 없음<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;프로세스의 자원은 운영체제에 의해 회수<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;종료된 프로세스의 PCB도 삭제됨<br/>

- 보류 준비<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;생성된 프로세스가 바로 메모리를 받지 못할때, 준비 또는 실행 상태에서 메모리를 잃게 될 때<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;또는 충분한 메모리 공간의 확보를 위해 준비 상태의 프로세스를 보류시키는 경우<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Running -> Suspended Ready : 높은 우선순위의 보류 대기 상태 프로세스가 준비 상태가 되어, 실행 상태의 프로세스로부터 CPU를 뺏는 경우<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Suspended Blocked -> Suspended Ready : 보류 대기 상태에 있던 프로세스가 기다리던 입출력이 완료된 경우<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Suspended Ready -> Ready : 다시 메모리를 받은 경우<br/>

- 보류 대기<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;대기 상태일 때 메모리 공간을 잃은 상태<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Blocked -> Suspended Blocked : 메모리 공간 확보를 위해 메모리를 잃은 상태<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Suspended Blocked -> Blocked : 메모리가 확보되어 대기가 된 경우<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Suspended Blocked -> Suspended Ready : 입출력이나 기다리던 이벤트가 종료된 경우<br/>


## multiprocessing 모듈 사용하는 방법
```python
from multiprocessing import Process, Queue, cpu_count, current_process, Manager
import requests
import re
import os


input_list = ['https://www.samsung.com', 'https://www.samsung.com/sec/event/samsung-monitor/', 'https://www.samsung.com/sec/event/2024-tv-launching/',
              'https://www.samsung.com/sec/event/best_items/', 'https://www.samsung.com/sec/event/bespoke-refrigerator/', 'https://www.samsung.com/sec/event/kimchi-refrigerator/',
              'https://www.samsung.com/sec/event/bespoke-grande-ai/', 'https://www.samsung.com/sec/event/bespoke-jet-air-cleaner/', 'https://www.samsung.com/sec/event/air-conditioners-inhome/',
              'https://www.samsung.com/sec/event/kitchen-appliance/']
input_list = input_list

button_id_regex = re.compile(r'<button\s(?:.*?\s)*?id=[\'"](.*?)[\'"].*?>', re.IGNORECASE)
link_regex = re.compile(r'<a\s(?:.*?\s)*?href=[\'"](.*?)[\'"].*?>')


def crawl_website(q, results):
    print('PID [%d] [%s]' % (os.getpid(), current_process().name))
    while not q.empty():
        value = q.get(True, 0.05)
        request_data = requests.get(value)
        button = button_id_regex.findall(request_data.text)
        link = link_regex.findall(request_data.text)
        results[value] = {'button': button, 'link': link}

def setup_queue(input_list, queue):
    print('PID [%d] [%s]' % (os.getpid(), current_process().name))
    for item in input_list:
        queue.put(item)
         

if __name__ == '__main__':
    shared_queue = Queue()
    max_workers = cpu_count()
    results = Manager().dict()

    producer = Process(target=setup_queue, args=(input_list, shared_queue))
    producer.start()
    producer.join()
    
    consumer_list = []
    for i in range(max_workers):
        consumer = Process(target=crawl_website, args=(shared_queue, results))
        consumer.start()
        consumer_list.append(consumer)
    
    [consumer.join() for consumer in consumer_list]
```



## ProcessPoolExecutor 사용하는 방법
```python
from concurrent.futures import as_completed, ProcessPoolExecutor
from multiprocessing import cpu_count, current_process, Manager
import requests
import re
import os


button_id_regex = re.compile(r'<button\s(?:.*?\s)*?id=[\'"](.*?)[\'"].*?>', re.IGNORECASE)
link_regex = re.compile(r'<a\s(?:.*?\s)*?href=[\'"](.*?)[\'"].*?>')
    
input_list = ['https://www.samsung.com', 'https://www.samsung.com/sec/event/samsung-monitor/', 'https://www.samsung.com/sec/event/2024-tv-launching/',
              'https://www.samsung.com/sec/event/best_items/', 'https://www.samsung.com/sec/event/bespoke-refrigerator/', 'https://www.samsung.com/sec/event/kimchi-refrigerator/',
              'https://www.samsung.com/sec/event/bespoke-grande-ai/', 'https://www.samsung.com/sec/event/bespoke-jet-air-cleaner/', 'https://www.samsung.com/sec/event/air-conditioners-inhome/',
              'https://www.samsung.com/sec/event/kitchen-appliance/']


def setup_queue(input_list, queue):
    print('PID [%d] [%s]' % (os.getpid(), current_process().name))
    for item in input_list:
        queue.put(item)


def crawl_website(url):
    print('PID [%d] [%s]' % (os.getpid(), current_process().name))
    request_data = requests.get(url)
    button = button_id_regex.findall(request_data.text)
    link = link_regex.findall(request_data.text)
    return {url: {'button': button, 'link': link}}


if __name__ == '__main__':
    manager  = Manager()
    shared_queue = manager.Queue()
    results = manager.dict()
    max_workers = cpu_count()
    
    with ProcessPoolExecutor(max_workers=max_workers) as setup_queue_processes:
        setup_queue_processes.submit(setup_queue, input_list, shared_queue)
    
    with ProcessPoolExecutor(max_workers=max_workers) as crawl_website_processes:
        future_tasks = []
        while not shared_queue.empty():
            url = shared_queue.get()
            future_tasks.append(crawl_website_processes.submit(crawl_website, url))

        for future in as_completed(future_tasks):
            results.update(future.result())
```