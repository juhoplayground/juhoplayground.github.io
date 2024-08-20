---
layout: post
title: Python 병렬 프로그래밍 - (1)
author: 'Juho'
date: 2024-08-20 09:00:00 +0900
categories: [Python]
tags: [Python, thread, threading, threadpool, threadpoolexcutor]
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
1. [thread란?](#thread란)
2. [thread 장,단점](#thread-장단점)
 - 1) [장점](#1-장점)
 - 2) [단점](#2-단점)
3. [thread 종류](#thread-종류)
 - 1) [커널 수준 Thread(Kernel-Level Thread)](#1-커널-수준-threadkernel-level-thread)
 - 2) [사용자 수준 Thread(User-Level Thtread)](#2-사용자-수준-threaduser-level-thtread)
 - 3) [Hybrid Thread](#3-hybrid-thread)
4. [thread 상태 정의](#thread-상태-정의)
5. [threading 모듈 사용하는 방법](#threading-모듈-사용하는-방법)
6. [ThreadPoolExecutor 사용하는 방법](#threadpoolexecutor-사용하는-방법)


## thread란?
thread란 무엇일까? <br/>
thread는 프로그램의 실행 흐름을 나누어 병렬로 작업을 수행할 수 있도록 하는 기능이다.<br/>
thread를 사용하면 하나의 프로세스 내에서 여러 작업을 동시에 실행할 수 있다.<br/>

## thread 장,단점
#### 1) 장점
- 동일한 프로세스에서 thread간 통신 속도, 데이터 위치, 정보 공유가 빠름<br/>
- 스레드 생성은 프로세스 생성에 비해 생성 비용이 적음 <br/>
- 프로세서의 캐시 메모리를 통해 메모리 접근을 최적화<br/>

#### 2) 단점
- CPU bound thread를 사용할 때 GIL로 인해 multi threading의 성능이 제한될 수 있음 <br/>
- GIL(Global Interpreter Lock)은 한 번에 하나의 thread만이 python 바이트코드를 실행할 수 있도록 하는 잠금<br/>


## thread 종류
#### 1) 커널 수준 Thread(Kernel-Level Thread)
커널 수준 Thread는 운영 체제에 의해 관리된다.<br/>
운영 체제의 스케줄러에 의해 스케줄링되며, 각 Thread는 독립적으로 스케줄링되고 관리된다.<br/>
장점<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- 하나의 커널 수준 Thread는 하나의 프로세스를 참조하기 때문에<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 특정 커널 수준 Thread가 Block 되더라도 다른 커널 수준 Thread들은 정상적으로 실행될 수 있다.<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- 여러 CPU에서 병렬로 실행될 수 있다.<br/>

단점<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- 생성과 동기화 루틴, 컨텍스트 스위칭 비용이 비싸다.<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- thread 구현이 플랫폼(OS)에 종속된다. <br/>

#### 2) 사용자 수준 Thread(User-Level Thtread)
사용자 수준 Thread는 라이브러리에 의해 관리된다.<br/>
운영 체제는 사용자 수준 Thread를 인식하지 못하며, 하나의 커널 수준 Thread에 여러 사용자 수준 Thread가 매핑될 수 있다.<br/>
장점<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- 생성과 동기화 루틴, 컨텍스트 비용이 저렴하다.<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- thread가 플랫폼(OS)에 종속되지 않는다.<br/>

단점<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- 하나의 사용자 수준 Thred가 Block될 경우 전체 프로세스가 Block될 수 있다.<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- 하나의 CPU에서만 실행될 수 있다.<br/>

#### 3) Hybrid Thread
커널 수준 Thread와 사용자 수준 Thread의 조합으로 사용자 수준 Thread가 커널 수준 Thread로 매핑된다.<br/>
커널 수준 Thread의 장점과 사용자 수준 Thread의 장점을 결합한 것 이다.<br/>


## thread 상태 정의
thread의 생명주기는 5단계로 나타낼 수 있다.<br/>
1) 생성<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- 메인 프로세스가 thread를 생성한다.<br/>
2) 실행<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- 이 단계에서 thread는 CPU를 사용한다.<br/>
3) 준비<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- 이 단계에서 thread는 실행될 의무가 있다.<br/>
4) 블록<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- 이 단계에서 thread는 I/O 연산을 기다리기 위해 Block 된다.<br/>
5) 종료<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;- 이 단계에서 thread는 실행에 사용했던 자원을 해제한 후, 수명이 끝난다<br/>


## threading 모듈 사용하는 방법
threading 모듈은 _thread 모듈의 추상화 계층을 제공해서 thread 기반의 병렬 시스템을 개발할 수 있는 함수들을 제공한다. <br/>
threading 모듈은 thread를 개발자가 직접 관리할 수 있게 해준다.<br/>
그래서 thread의 생명주기, 동기화 문제, 예외 처리 등을 개발자가 직접 관리해야 한다.<br/>
장점은 thread를 세밀하게 제어할 수 있어 복잡한 thread 관리가 필요한 경우에 유리하다.<br/>
단점은 thread 수가 많아지면 관리가 복잡해질 수 있고, thread pool 과 같은 추상화가 없기 때문에 더 많은 코딩을 해야 한다.<br/>
threading 모듈에 대한 자세한 내용은 [threading](https://docs.python.org/3/library/threading.html){:target="_blank"} 에서 확인해볼 수 있다. <br/>
```python
from queue import Queue
import threading
import requests
import re


shared_queue = Queue()
input_list = ['https://www.samsung.com', 'https://www.samsung.com/sec/event/samsung-monitor/', 'https://www.samsung.com/sec/event/2024-tv-launching/',
              'https://www.samsung.com/sec/event/best_items/', 'https://www.samsung.com/sec/event/bespoke-refrigerator/', 'https://www.samsung.com/sec/event/kimchi-refrigerator/',
              'https://www.samsung.com/sec/event/bespoke-grande-ai/', 'https://www.samsung.com/sec/event/bespoke-jet-air-cleaner/', 'https://www.samsung.com/sec/event/air-conditioners-inhome/',
              'https://www.samsung.com/sec/event/kitchen-appliance/']
input_list = input_list * 100

results = {}
queue_condition = threading.Condition()
button_id_regex = re.compile(r'<button\s(?:.*?\s)*?id=[\'"](.*?)[\'"].*?>', re.IGNORECASE)
link_regex = re.compile(r'<a\s(?:.*?\s)*?href=[\'"](.*?)[\'"].*?>')


def crawl_website(condition):
    with condition:
        while shared_queue.empty():
            condition.wait()
        else:
            url = shared_queue.get()
            request_data = requests.get(url)
            button = button_id_regex.findall(request_data.text)
            
            link = link_regex.findall(request_data.text)
            results[url] = {'button': button, 'link': link}
            shared_queue.task_done()


def setup_queue(condition):
    with condition:
        for item in input_list:
            shared_queue.put(item)
        condition.notify_all()


if __name__  == '__main__':
    threads_1 = [threading.Thread(name=f'crawl_task_{i}',daemon=True, target=crawl_website, args=(queue_condition,)) for i in range(len(input_list))]
    [thread.start() for thread in threads_1]

    threads_2 = threading.Thread(name='setup_queue', daemon=True, target=setup_queue, args=(queue_condition,))
    threads_2.start()

    [thread.join() for thread in threads_1]
```

## ThreadPoolExecutor 사용하는 방법
concurrent.futures 모듈에서 제공하는 클래스 중 하나로, thread 기반의 병렬 처리를 손쉽게 구현할 수 있게 해주는 함수를 제공한다.<br/>
thread pool을 제공하여 thread 관리를 더 간편하게 해준다.<br/>
thread pool은 일정 수의 thread를 미리 생성하고, 작업을 queue에 넣어 관리를 한다.<br/>
그렇기 때문에 thread 수를 개발자가 직접 제어할 필요가 없고, thread 관리의 복잡함을 줄일 수 있다.<br/>
장점은 thread pool을 이용해 thread 수를 조절하고 관리하는 것이 간편하다.<br/>
단점은 thread를 직접 제어하거나 thread 관리가 필요한 경우에는 유연성이 부족하다.<br/>
또한 ThreadPoolExecutor에 대한 자세한 내용은 [ThreadPoolExecutor](https://docs.python.org/3/library/concurrent.futures.html){:target="_blank"} 에서 확인해볼 수 있다. <br/>
```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from queue import Queue
import requests
import re
import os


shared_queue = Queue()
input_list = ['https://www.samsung.com', 'https://www.samsung.com/sec/event/samsung-monitor/', 'https://www.samsung.com/sec/event/2024-tv-launching/',
              'https://www.samsung.com/sec/event/best_items/', 'https://www.samsung.com/sec/event/bespoke-refrigerator/', 'https://www.samsung.com/sec/event/kimchi-refrigerator/',
              'https://www.samsung.com/sec/event/bespoke-grande-ai/', 'https://www.samsung.com/sec/event/bespoke-jet-air-cleaner/', 'https://www.samsung.com/sec/event/air-conditioners-inhome/',
              'https://www.samsung.com/sec/event/kitchen-appliance/']
input_list = input_list * 100

results = {}
button_id_regex = re.compile(r'<button\s(?:.*?\s)*?id=[\'"](.*?)[\'"].*?>', re.IGNORECASE)
link_regex = re.compile(r'<a\s(?:.*?\s)*?href=[\'"](.*?)[\'"].*?>')


def crawl_website(url):
    request_data = requests.get(url)
    button = button_id_regex.findall(request_data.text)
            
    link = link_regex.findall(request_data.text)
    return {url: {'button': button, 'link': link}}
        

def setup_queue(url):
    shared_queue.put(url)


if __name__  == '__main__':
    max_workers = os.cpu_count()
    with ThreadPoolExecutor(max_workers=max_workers) as setup_queue_threads:
        future_tasks = [setup_queue_threads.submit(setup_queue, item) for item in input_list]
        for future in as_completed(future_tasks):
            pass
        
    with ThreadPoolExecutor(max_workers=max_workers) as crawl_website_threads:
        future_tasks = []
        while not shared_queue.empty():
            url = shared_queue.get()
            future_tasks.append(crawl_website_threads.submit(crawl_website, url))
            
        for future in as_completed(future_tasks):
            results.update(future.result())
```
