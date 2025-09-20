---
layout: post
title: Python 프로파일링
author: 'Juho'
date: 2025-09-20 09:00:00 +0900
categories: [Python]
tags: [Python, Profile]
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
1. [Profile이란?](#profile이란)  
2. [CPU Profile](#cpu-profile)
  - [cProfile (표준 라이브러리)](#cprofile-표준-라이브러리)
  - [line_profiler (라인 단위 CPU 분석)](#line_profiler-라인-단위-cpu-분석)
3. [Memory Profile](#memory-profile)
 - [tracemalloc (표준 라이브러리)](#tracemalloc-표준-라이브러리)
 - [memory_profiler (라인 단위 메모리 분석)](#memory_profiler-라인-단위-메모리-분석)
4. [I/O Profile](#io-profile)
   - [strace](#strace)
   - [ltrace](#ltrace)
5. [종합 Profile](#종합-profile)
   - [scalene](#scalene)
   - [py-spy](#py-spy)
6. [GPU Profile](#gpu-profile)
   - [torch.profiler](#torchprofiler)
7. [병렬 처리 및 스레드 Profile](#병렬-처리-및-스레드-profile)
   - [yappi](#yappi)
8. [이벤트 루프 & 비동기 성능](#이벤트-루프--비동기-성능)
   - [aiomonitor](#aiomonitor)

## Profile이란?
**Profiling**이란 프로그램 성능을 분석하기 위한 기법이다.  
- **CPU 프로파일링**: 어떤 함수(혹은 코드 라인)가 CPU 시간을 많이 쓰는지 분석  
- **메모리 프로파일링**: 어떤 부분에서 메모리를 얼마나 사용하는지, 누수가 있는지 분석  
- **I/O 프로파일링**: 디스크/네트워크 같은 입출력 병목 구간 추적  
- **GPU 프로파일링**: 딥러닝/머신러닝 연산의 GPU 사용량 측정  
- **스레드/병렬 처리 프로파일링**: 동시성 코드에서 락(lock)이나 대기 시간 분석  

코드의 병목 지점이 어디인지 파악하여 성능 최적화에 활용한다.  

## CPU Profile
  
### cProfile (표준 라이브러리)
함수 단위 실행 시간을 측정  
```python
import cProfile
import pstats

def slow_function():
    total = 0
    for i in range(10**6):
        total += i
    return total

if __name__ == "__main__":
    profiler = cProfile.Profile()
    profiler.enable()

    slow_function()

    profiler.disable()
    stats = pstats.Stats(profiler).sort_stats("cumtime")
    stats.print_stats(10)
```

실행 방법 `python script.py`  
출력 예시  
```bash
ncalls  tottime  percall  cumtime  percall filename:lineno(function)
1       0.08     0.08     0.19     0.19    script.py:4(slow_function)
```
- tottime: 해당 함수 자체 실행 시간  
- cumtime: 해당 함수 + 내부 함수 실행까지 누적 시간  

  
### line_profiler (라인 단위 CPU 분석)
함수 단위가 아닌 코드 라인별 CPU 시간을 분석  

설치 방법 `pip install line-profiler`  
```python
@profile
def slow_function():
    total = 0
    for i in range(10**6):
        total += i
    return total
```

실행 방법 `kernprof -l -v script.py`  
출력 예시  
```bash
Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
    2                                           @profile
    3                                           def slow_function():
    4         1          2.0      2.0      0.0      total = 0
    5   1000000     120000.0      0.1     95.0      total += i
```

## Memory Profile  

### tracemalloc (표준 라이브러리)
현재 메모리와 최대 피크 메모리를 추적  
```python
import tracemalloc

tracemalloc.start()

def make_list():
    return [i for i in range(10**6)]

data = make_list()

current, peak = tracemalloc.get_traced_memory()
print(f"현재 메모리: {current / 1024**2:.2f} MB")
print(f"최대 메모리: {peak / 1024**2:.2f} MB")

tracemalloc.stop()
```
출력 예시  
```bash
현재 메모리: 8.32 MB
최대 메모리: 9.45 MB
```

### memory_profiler (라인 단위 메모리 분석)
라인 단위로 메모리 변화를 추적  
설치 방법 `pip install memory-profiler`  
```python
from memory_profiler import profile

@profile
def allocate_list():
    a = [i for i in range(10**6)]
    return a

if __name__ == "__main__":
    allocate_list()
```

실행 방법 `python -m memory_profiler script.py`  
출력 예시  
```bash
Line #    Mem usage    Increment  Occurrences   Line Contents
=============================================================
     4     12.0 MiB     12.0 MiB           1   a = [i for i in range(10**6)]
```


## I/O Profile
### strace  
리눅스에서 시스템 호출을 추적하는 도구  
실행 방법 `strace -c -o trace.log python script.py`  
> -c: 호출 통계 요약  
> -o trace.log: 로그 파일 저장  
  
출력 예시 
```bash
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 50.00    0.002000         200        10           read
 30.00    0.001200         150         8           write
```

### ltrace
동적 라이브러리 호출을 추적하는 도구  
실행 방법 `ltrace -c python script.py`
> 외부 C 라이브러리 함수 호출 횟수, 소요 시간 분석 가능  
  

## 종합 Profile
### scalene  
CPU, 메모리, I/O까지 종합적으로 분석  
설치 방법 `pip install scalene`  
실행 방법 `scalene script.py`  

### py-spy
실행 중인 Python 프로세스를 attach하여 분석  
설치 방법 `pip install py-spy`  
실행 방법 `py-spy top -- python script.py`
> top처럼 실시간 CPU 사용률 확인  
> flame graph 생성 가능

  

## GPU Profile
### torch.profiler
GPU 사용량을 분석하는 PyTorch 내장 프로파일러  
```python
import torch
import torch.profiler as profiler

def train_step():
    x = torch.randn(1000, 1000, device="cuda")
    y = torch.matmul(x, x)
    return y

with profiler.profile(
    activities=[profiler.ProfilerActivity.CPU, profiler.ProfilerActivity.CUDA],
    on_trace_ready=profiler.tensorboard_trace_handler("./log")
) as prof:
    train_step()

print(prof.key_averages().table(sort_by="cuda_time_total"))
```
CPU/GPU 시간 모두 측정  
TensorBoard에서 시각화 가능  


## 병렬 처리 및 스레드 Profile  
### yappi
멀티스레드/멀티프로세스 환경에서 CPU 사용량과 lock 대기 시간을 분석
설치 방법 `pip install yappi`  
```python
import yappi
import threading

def worker():
    sum(i for i in range(10**6))

yappi.start()
t1 = threading.Thread(target=worker)
t2 = threading.Thread(target=worker)
t1.start(); t2.start()
t1.join(); t2.join()
yappi.stop()

yappi.get_func_stats().print_all()
```
> 스레드/코루틴 단위 CPU 시간 확인 가능  


## 이벤트 루프 & 비동기 성능
### aiomonitor
비동기(asyncio) 코드에서 이벤트 루프 지연을 추적  
설치 방법 `pip install aiomonitor`  
```
import asyncio
import aiomonitor

async def main():
    await asyncio.sleep(2)

loop = asyncio.get_event_loop()
with aiomonitor.start_monitor(loop=loop):
    loop.run_until_complete(main())
```
> `telnet localhost 50101` 접속 후 이벤트 루프 상태 모니터링 가능  


---  
  
최적화의 첫걸음은 "측정" 이라고 생각을 한다.  
목적에 맞는 프로파일링 도구를 선택, 활용하여 불필요한 최적화 대신 실제 병목 지점을 정확히 개선할 수 있다.  
