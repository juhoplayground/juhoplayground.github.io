 [---
layout: post
title: LangSmith tracer 에러 및 해결 방법
author: 'Juho'
date: 2025-04-25 09:00:00 +0900
categories: [LangSmith]
tags: [LangSmith, Langgraph, LangChain, Agent, Chatbot, Python]
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
1. [발생 오류 및 원인](#발생-오류-및-원인)
2. [오류 해결 방법](#오류-해결-방법)

## 발생 오류 및 원인
```bash
Exception in thread Thread-1 (tracing_control_thread_func):
Traceback (most recent call last):
  File "/home/ubuntu/.local/share/uv/python/cpython-3.12.0-linux-x86_64-gnu/lib/python3.12/threading.py", line 1052, in _bootstrap_inner
    self.run()
  File "/home/ubuntu/.local/share/uv/python/cpython-3.12.0-linux-x86_64-gnu/lib/python3.12/threading.py", line 989, in run
    self._target(*self._args, **self._kwargs)
  File "/home/ubuntu/artience_llm/.venv/lib/python3.12/site-packages/langsmith/_internal/_background_thread.py", line 298, in tracing_control_thread_func
    ).start()
      ^^^^^^^
  File "/home/ubuntu/.local/share/uv/python/cpython-3.12.0-linux-x86_64-gnu/lib/python3.12/threading.py", line 971, in start
    _start_new_thread(self._bootstrap, ())
RuntimeError: can't create new thread at interpreter shutdown
```

처음 보는 오류가 발생해서 확인을 해보았음  
Python 3.12에서 도입된 인터프리터 종료 단계 변경으로 인해 `atexit` 콜백이나 백그라운드 타이머 스레드가 새 non-daemon 스레드를 생성하려 할 때 발생하는 것  
CPython 3.12부터는 인터프리터가 종료 중일 때 새로운 스레드 생성을 금지하며 이로 인해 `RuntimeError: can't create new thread at interpreter shutdown` 에러가 발생한 것  
이는 3.12 릴리즈 이후 [보고된 버그(#113964)](https://github.com/python/cpython/issues/113964?utm_source=chatgpt.com){:target="_blank"}이며, 관련 GitHub 이슈에서 자세히 다루고 있음  

## 오류 해결 방법
### 1. Python 버전 변경
3.12 버전에서 발생한 이슈이기 때문에 3.11로 Python을 다운그레이드하면 간단히 해결할 수 있음  
하지만 현재 사용하는 특정 라이브러리가 3.12를 요구하기 때문에 이 방법은 사용할 수 없었음  

### 2.LangSmith/LangChain tracing 비활성화
LangSmith의 tracing을 사용하지 않으면 백그라운드 스레드가 생성되지 않음  
`LANGCHAIN_TRACING_V2` 값을 `False`로 설정하면 됨  
하지만 LangSmith를 이용해서 Graph의 실행 내역을 모니터링하는것이 목적이기 때문에 이 방법도 사용할 수 없었음  


### 3. tracer 종료 대기
Python 스크립트가 끝나기전에 LangChain tracer들이 모두 작업을 마치도록 명시적으로 대기할 수도 있음  
```python
from langchain.callbacks.tracers.langchain import wait_for_all_tracers
try:
    do_something()
finally:
    wait_for_all_tracers()
``` 
이렇게 하면 인터프리터 종료 시점에 새 스레드를 생성할 일이 없어져서 오류가 해결됨  



---  

