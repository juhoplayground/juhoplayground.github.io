---
layout: post
title: Python 병렬 프로그래밍 - (4) pp (Parallel Python)
author: 'Juho'
date: 2024-08-26 09:00:00 +0900
categories: [Python]
tags: [Python, pp, Parallel Python]
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
1. [pp란?](#pp란)
2. [pp 사용하는 방법](#)

## pp란?
pp는 pprint의 내용인 줄 알았는데 [Parallel Python](https://www.parallelpython.com/){:target="_blank"}에서 제공하는 라이브러리였다.<br/>
설치는 pip으로 할 수 없고 zip 파일을 풀어서 `python setup.py install`로 설치를 해야한다.<br/>
그렇게 설치를 하면 아래와 같은 경고가 발생한다.<br/>

```
DEPRECATION: Loading egg at /venv/lib/python3.11/site-packages/pp-1.6.4.4-py3.11.egg is deprecated. pip 24.3 will enforce this behaviour change. A possible replacement is to use pip for package installation.. Discussion can be found at https://github.com/pypa/pip/issues/12330
```

`.egg` 형식의 패키지는 오래된 Python 패키지 형식으로 패키지 로딩이 더 이상 지원이 되지 않을거라는 경고다.<br/>
현재는 `.whl` 방식이 일반적으로 사용된다. <br/>
그 만큼 업데이트가 안되는 패키지라는 사실을 알 수 있다.<br/>
사실 이런 라이브러리는 안쓰는게 맞다.<br/>
그럼에도 사용해보고 싶다는 분을 위한 내용 <br/>



## pp 사용하는 방법
```python
import requests
import pp
import os
import re


result_dict = {}


def aggregate_results(result):
    result_dict.update(result)


def crawl_task(url, button_id_regex, link_regex):
    request_data = requests.get(url)
    button = button_id_regex.findall(request_data.text)[:3]
    link = link_regex.findall(request_data.text)[:3]
    return {url : {'button': button, 'link': link}} 


if __name__ == '__main__':
    input_list = ['https://www.samsung.com', 'https://www.samsung.com/sec/event/samsung-monitor/', 'https://www.samsung.com/sec/event/2024-tv-launching/',
              'https://www.samsung.com/sec/event/best_items/', 'https://www.samsung.com/sec/event/bespoke-refrigerator/', 'https://www.samsung.com/sec/event/kimchi-refrigerator/',
              'https://www.samsung.com/sec/event/bespoke-grande-ai/', 'https://www.samsung.com/sec/event/bespoke-jet-air-cleaner/', 'https://www.samsung.com/sec/event/air-conditioners-inhome/',
              'https://www.samsung.com/sec/event/kitchen-appliance/']
    input_list = input_list * 100

    button_id_regex = re.compile(r'<button\s(?:.*?\s)*?id=[\'"](.*?)[\'"].*?>', re.IGNORECASE)
    link_regex = re.compile(r'<a\s(?:.*?\s)*?href=[\'"](.*?)[\'"].*?>')
    
    node_list  = ('localhost', ) # 노드 여러개를 입력하면 분산 처리를 할 수 있다
    job_dispatcher = pp.Server(ncpus=os.cpu_count(), ppservers=node_list, socket_timeout=60000)
    
    for url in input_list:
        job_dispatcher.submit(crawl_task, # 실행하려고 하는 함수
                              (url, button_id_regex, link_regex), # 함수에 필요한 파라미터, 튜플 형태로 보내줘야 함 
                              modules=('os', 're', 'requests',), # 해당 함수를 실행하는데 필요한 라이브러리
                              callback=aggregate_results) #실행하려고 하는 함수의 return 값을 처리하는 함수
        
    job_dispatcher.wait()
    
    print(job_dispatcher.print_stats())
    
    job_dispatcher.destroy()

```

---




<br/>

안쓰는걸 추천한다.<br/>