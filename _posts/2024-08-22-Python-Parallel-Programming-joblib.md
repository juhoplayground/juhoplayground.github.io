---
layout: post
title: Python 병렬 프로그래밍 - (3) joblib
author: 'Juho'
date: 2024-08-22 09:00:00 +0900
categories: [Python]
tags: [Python, joblib]
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
1. [joblib이란?](#joblib이란)
2. [backend 옵션](#backend-옵션)
 - 1) [loky](#1-loky)
 - 2) [threading](#2-threading)
 - 3) [multiprocessing](#3-multiprocessing)
 - 4) [dask](#4-dask)
 - 5) [ray](#5-ray)

## joblib이란?
joblib은 python에서 데이터 처리 및 모델 저장과 로드를 위해 주로 사용되는 라이브러리로 대용량 데이터의 직렬화와 병렬 처리에 유용하다.<br/>
오늘 이야기할 내용은 병렬 처리에 대한 부분으로 joblib의 Parallel과 delayed 기능이다.<br/>
프로세스나 thread로 나누어 병렬적으로 처리를 할 수 있게 한다.<br/>

`Parallel`은 병렬 실행을 위한 엔진이다.<br/>
Parallel은 반복 작업을 여러 개의 프로세스로 나누어 병렬로 처리할 수 있다.<br/>
몇 개의 작업을 동시에 실행할지(n_jobs로 지정)와 같은 설정을 제어할 수 있다.<br/>

`delayed`는 함수를 래핑하여 병렬 실행을 위해 Parallel에 전달할 수 있도록 도와준다.<br/>
delayed를 사용하면 함수 호출을 지연시키고, 나중에 Parallel을 통해 이 함수들을 병렬로 실행할 수 있다.<br/>


## backend 옵션
#### 1) loky
default값으로 python 프로세스를 여러 개 생성하여 작업을 병렬로 수행한다.<br/>
이는 메모리를 독립적으로 관리할 수 있어, 메모리 누수가 발생할 가능성을 줄일 수 있다.<br/>
프로세스 간의 완전한 격리를 제공하며, 주로 CPU 집약적인 작업에 적합하다.<br/>
```python
from joblib import Parallel, delayed, parallel_backend
import requests
import re
import os

input_list = ['https://www.samsung.com', 'https://www.samsung.com/sec/event/samsung-monitor/', 'https://www.samsung.com/sec/event/2024-tv-launching/',
              'https://www.samsung.com/sec/event/best_items/', 'https://www.samsung.com/sec/event/bespoke-refrigerator/', 'https://www.samsung.com/sec/event/kimchi-refrigerator/',
              'https://www.samsung.com/sec/event/bespoke-grande-ai/', 'https://www.samsung.com/sec/event/bespoke-jet-air-cleaner/', 'https://www.samsung.com/sec/event/air-conditioners-inhome/',
              'https://www.samsung.com/sec/event/kitchen-appliance/']
input_list = input_list * 100

button_id_regex = re.compile(r'<button\s(?:.*?\s)*?id=[\'"](.*?)[\'"].*?>', re.IGNORECASE)
link_regex = re.compile(r'<a\s(?:.*?\s)*?href=[\'"](.*?)[\'"].*?>')

def crawl_website(url):
    request_data = requests.get(url)
    button = button_id_regex.findall(request_data.text)   
    link = link_regex.findall(request_data.text)
    return {url: {'button': button, 'link': link}}

if __name__  == '__main__':
    max_workers = os.cpu_count()
    with parallel_backend(backend="loky", n_jobs=max_workers):
        results = Parallel()(delayed(crawl_website)(url) for url in input_list)

```
<br/>

#### 2) threading
python의 thread를 사용하여 병렬처리를 수행한다.<br/>
I/O 바운드 작업이나 GIL이 해제된 경우에 유리하다.<br/>
CPU 바운드 작업의 경우 GIL 때문에 성능 향상이 제한적일 수 있다.<br/>
```python
from joblib import Parallel, delayed, parallel_backend
import requests
import re
import os

input_list = ['https://www.samsung.com', 'https://www.samsung.com/sec/event/samsung-monitor/', 'https://www.samsung.com/sec/event/2024-tv-launching/',
              'https://www.samsung.com/sec/event/best_items/', 'https://www.samsung.com/sec/event/bespoke-refrigerator/', 'https://www.samsung.com/sec/event/kimchi-refrigerator/',
              'https://www.samsung.com/sec/event/bespoke-grande-ai/', 'https://www.samsung.com/sec/event/bespoke-jet-air-cleaner/', 'https://www.samsung.com/sec/event/air-conditioners-inhome/',
              'https://www.samsung.com/sec/event/kitchen-appliance/']
input_list = input_list * 100

button_id_regex = re.compile(r'<button\s(?:.*?\s)*?id=[\'"](.*?)[\'"].*?>', re.IGNORECASE)
link_regex = re.compile(r'<a\s(?:.*?\s)*?href=[\'"](.*?)[\'"].*?>')

def crawl_website(url):
    request_data = requests.get(url)
    button = button_id_regex.findall(request_data.text)   
    link = link_regex.findall(request_data.text)
    return {url: {'button': button, 'link': link}}

if __name__  == '__main__':
    max_workers = os.cpu_count()
    with parallel_backend(backend="threading", n_jobs=max_workers):
        results = Parallel()(delayed(crawl_website)(url) for url in input_list)

```
<br/>

#### 3) multiprocessing
python의 multiprocessing를 사용하여 여러 프로세스를 생성하여 작업을 병렬로 수행한다.<br/>
메모리 복사와 관련된 이슈가 있을 수 있다.<br/>
```python
from joblib import Parallel, delayed, parallel_backend
import requests
import re
import os

input_list = ['https://www.samsung.com', 'https://www.samsung.com/sec/event/samsung-monitor/', 'https://www.samsung.com/sec/event/2024-tv-launching/',
              'https://www.samsung.com/sec/event/best_items/', 'https://www.samsung.com/sec/event/bespoke-refrigerator/', 'https://www.samsung.com/sec/event/kimchi-refrigerator/',
              'https://www.samsung.com/sec/event/bespoke-grande-ai/', 'https://www.samsung.com/sec/event/bespoke-jet-air-cleaner/', 'https://www.samsung.com/sec/event/air-conditioners-inhome/',
              'https://www.samsung.com/sec/event/kitchen-appliance/']
input_list = input_list * 100

button_id_regex = re.compile(r'<button\s(?:.*?\s)*?id=[\'"](.*?)[\'"].*?>', re.IGNORECASE)
link_regex = re.compile(r'<a\s(?:.*?\s)*?href=[\'"](.*?)[\'"].*?>')

def crawl_website(url):
    request_data = requests.get(url)
    button = button_id_regex.findall(request_data.text)   
    link = link_regex.findall(request_data.text)
    return {url: {'button': button, 'link': link}}

if __name__  == '__main__':
    max_workers = os.cpu_count()
    with parallel_backend(backend="multiprocessing", n_jobs=max_workers):
        results = Parallel()(delayed(crawl_website)(url) for url in input_list)

```
<br/>

#### 4) dask
Dask는 대규모 병렬 계산을 위한 라이브러리로, 로컬 및 분산 환경에서 모두 사용할 수 있다.<br/>
Dask를 사용하면 더 복잡한 작업 흐름과 분산 환경에서의 병렬 처리가 가능하다.<br/>
큰 데이터셋을 처리하거나 클러스터에서 작업할 때 유리하다.<br/>
Dask는 데이터 프레임, 배열, 그래프 등 다양한 데이터 구조를 지원한다.<br/>
```python
from joblib import Parallel, delayed, parallel_backend
from dask.distributed import Client, LocalCluster
import requests
import re
import os

input_list = ['https://www.samsung.com', 'https://www.samsung.com/sec/event/samsung-monitor/', 'https://www.samsung.com/sec/event/2024-tv-launching/',
              'https://www.samsung.com/sec/event/best_items/', 'https://www.samsung.com/sec/event/bespoke-refrigerator/', 'https://www.samsung.com/sec/event/kimchi-refrigerator/',
              'https://www.samsung.com/sec/event/bespoke-grande-ai/', 'https://www.samsung.com/sec/event/bespoke-jet-air-cleaner/', 'https://www.samsung.com/sec/event/air-conditioners-inhome/',
              'https://www.samsung.com/sec/event/kitchen-appliance/']
input_list = input_list * 100

button_id_regex = re.compile(r'<button\s(?:.*?\s)*?id=[\'"](.*?)[\'"].*?>', re.IGNORECASE)
link_regex = re.compile(r'<a\s(?:.*?\s)*?href=[\'"](.*?)[\'"].*?>')

def crawl_website(url):
    request_data = requests.get(url)
    button = button_id_regex.findall(request_data.text)   
    link = link_regex.findall(request_data.text)
    return {url: {'button': button, 'link': link}}

if __name__  == '__main__':
    max_workers = os.cpu_count()
    cluster = LocalCluster()
    client = Client(cluster)  
    with parallel_backend(backend="dask", n_jobs=max_workers):
        results = Parallel()(delayed(crawl_website)(url) for url in input_list)

```
<br/>

#### 5) ray
Ray는 분산 컴퓨팅 프레임워크로, 클러스터 환경에서의 병렬 처리를 지원한다.<br/>
대규모 클러스터에서 효율적으로 작업을 병렬화하고, 머신러닝 작업에 자주 사용된다.<br/>
```python
from joblib import Parallel, delayed, parallel_backend
from ray.util.joblib import register_ray
import requests
import re
import os

input_list = ['https://www.samsung.com', 'https://www.samsung.com/sec/event/samsung-monitor/', 'https://www.samsung.com/sec/event/2024-tv-launching/',
              'https://www.samsung.com/sec/event/best_items/', 'https://www.samsung.com/sec/event/bespoke-refrigerator/', 'https://www.samsung.com/sec/event/kimchi-refrigerator/',
              'https://www.samsung.com/sec/event/bespoke-grande-ai/', 'https://www.samsung.com/sec/event/bespoke-jet-air-cleaner/', 'https://www.samsung.com/sec/event/air-conditioners-inhome/',
              'https://www.samsung.com/sec/event/kitchen-appliance/']
input_list = input_list * 100

button_id_regex = re.compile(r'<button\s(?:.*?\s)*?id=[\'"](.*?)[\'"].*?>', re.IGNORECASE)
link_regex = re.compile(r'<a\s(?:.*?\s)*?href=[\'"](.*?)[\'"].*?>')

def crawl_website(url):
    request_data = requests.get(url)
    button = button_id_regex.findall(request_data.text)   
    link = link_regex.findall(request_data.text)
    return {url: {'button': button, 'link': link}}

if __name__  == '__main__':
    max_workers = os.cpu_count()
    register_ray()
    with parallel_backend(backend="ray", n_jobs=max_workers):
        results = Parallel()(delayed(crawl_website)(url) for url in input_list)

```
<br/>
ray를 사용하면 `WARNING pool.py:589 -- The 'context' argument is not supported using ray. Please refer to the documentation for how to control ray initialization.`<br/>
이런 오류가 발생하는데 [ray](https://github.com/ray-project/ray/issues/13855){:target="_blank"}를 보면 WARNING이 발생하지만 무시해도 될 것 같다.<br/>


---

<br/>

`Parallel()`이 부분에 `verbose='100'` 이런식으로 입력하게 되면 작업 진행 상황을 출력한다.<br/>
기본 값은 0으로 아무것도 출력하지 않는다. <br/>