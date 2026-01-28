---
layout: post
title: vLLM 메모리 누수 디버깅 - Heaps do lie
author: 'Juho'
date: 2026-01-28 00:00:00 +0900
categories: [vLLM]
tags: [Python, LLM]
pin: True
toc: True
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
1. [개요](#개요)
2. [문제 상황](#문제-상황)
3. [디버깅 과정](#디버깅-과정)
4. [메모리 누수의 근본 원인](#메모리-누수의-근본-원인)
5. [해결 방법](#해결-방법)
6. [핵심 교훈](#핵심-교훈)
7. [Reference](#reference)

---

## 개요

Mistral AI 팀이 vLLM에서 발견한 메모리 누수를 추적하고 해결한 과정을 다룬 Engineering Deep Dive 기사를 정리한 포스트입니다.
"Heaps do lie"라는 제목처럼, 힙 메모리 프로파일러로는 발견되지 않는 깊은 레이어의 문제였습니다.
이 사례는 현대 소프트웨어의 복잡한 의존성 계층에서 발생할 수 있는 문제를 잘 보여줍니다.

---

## 문제 상황

vLLM을 운영하면서 매분 400MB씩 선형적으로 메모리가 증가하는 현상이 관찰되었습니다.
일반적인 힙 메모리 프로파일러로는 누수를 발견할 수 없었습니다.

### 관찰된 증상

- RSS(Resident Set Size) 메모리가 지속적으로 증가
- 힙 프로파일러에서는 문제가 발견되지 않음
- 익명 메모리 맵핑 영역이 계속 확장됨

---

## 디버깅 과정

### 1단계: 표준 힙 프로파일러 시도 (실패)

처음에는 일반적인 Python 메모리 프로파일링 도구를 사용해 봤지만 모두 실패했습니다.

사용한 도구들:
- Memray
- Guppy 3
- Heaptrack

Heaptrack은 glibc의 malloc/free만 추적하므로, 직접적인 mmap 호출은 감지하지 못합니다.
힙 내부에서는 누수가 발견되지 않았습니다.

### 2단계: 메모리 구조 분석

`pmap` 명령어를 활용하여 프로세스의 메모리 맵을 분석했습니다.

```bash
watch -n 1 "pmap -X $pid | (head -n 2 && tail -n +3 | sort -k7 -nr)"
```

익명 메모리 맵핑이 지속적으로 증가하는 패턴을 발견했습니다.
이는 힙이 아닌 mmap을 통한 메모리 할당에서 문제가 있음을 시사합니다.

### 3단계: BPFtrace를 통한 syscall 추적

eBPF 가상머신을 활용한 커널 레벨 트레이싱으로 mmap/munmap 호출을 감시했습니다.
모든 호출이 glibc의 syscall wrapper를 통해 발생함을 확인했습니다.

### 4단계: GDB 자동화

조건부 breakpoint를 설정하여 문제의 원인을 특정했습니다.

```gdb
break syscall if $rdi == 9  # SYS_mmap 필터
```

호출 스택을 추적한 결과, UCX 라이브러리 경로를 발견했습니다.

```
ucm_orig_mmap_syscall → ucm_mmap → _PyMem_ArenaAlloc
```

---

## 메모리 누수의 근본 원인

### UCX의 mmap 훅 메커니즘

메모리 누수가 발생한 핵심 원인은 UCX(Unified Communication X)가 모든 mmap 호출을 가로채기 때문이었습니다.

UCX는 InfiniBand용 고성능 통신 라이브러리입니다.
Registration Cache(RCache)를 통해 메모리 등록을 최적화합니다.

### 문제의 메커니즘

1. munmap 호출 시 메모리가 즉시 해제되지 않음
2. 대신 invalidation queue에 이동
3. 큐가 동적으로 확장되면서 누수 발생
4. 매분 400MB 선형 증가가 관찰됨

### GOT 패칭 (Global Offset Table)

UCX는 동적 링커의 GOT 엔트리를 런타임에 수정하여 함수 호출을 가로챕니다.
이는 일반적으로 위험한 관행이지만, InfiniBand 성능 최적화를 위해 필요합니다.

### 호출 스택 예시

```
_PyThreadState_PopFrame (Python)
  → ucm_munmap (UCX 훅)
    → ucm_event_dispatch
      → ucs_mpool_grow (메모리 풀 확장)
        → ucm_orig_mmap_syscall (실제 mmap 호출)
```

munmap 호출 중에도 mmap이 발생하는 이상한 동작이 관찰되었습니다.
이것이 바로 메모리가 계속 증가하는 원인이었습니다.

---

## 해결 방법

### 방법 1: mmap 훅 완전 비활성화 (권장)

```bash
export UCX_MEM_MMAP_HOOK_MODE=none
```

mmap 훅 메커니즘을 완전히 비활성화합니다.
성능 영향 없이 누수를 해결할 수 있습니다.

### 방법 2: RCache 제한 설정

```bash
export UCX_RCACHE_MAX_UNRELEASED=1024
```

기본값(무제한)에서 제한값으로 변경하여 정기적인 메모리 정리를 강제합니다.

### 적용 방법

vLLM 실행 전에 환경 변수를 설정하면 됩니다.

```bash
# 방법 1: 훅 비활성화
export UCX_MEM_MMAP_HOOK_MODE=none
vllm serve your-model

# 방법 2: RCache 제한
export UCX_RCACHE_MAX_UNRELEASED=1024
vllm serve your-model
```

---

## 핵심 교훈

### Linux 메모리 구조 이해의 중요성

RSS(Resident Set Size)는 다음을 포함합니다:
- 힙 (sbrk/brk로 관리)
- 스택
- 익명 메모리 맵핑 (mmap으로 할당된 영역)

힙 프로파일러만으로는 mmap 기반 할당을 추적할 수 없습니다.

### 의존성 계층의 복잡성

현대 소프트웨어는 의존성의 계층으로 구축되어 있습니다.
성능 최적화가 숨겨진 위험을 야기할 수 있습니다.
깊은 스택의 라이브러리가 낮은 수준의 조작(GOT 패칭)을 수행할 때, 디버깅 난이도가 극도로 상승합니다.

### 디버깅 도구 선택

| 도구 | 용도 | 한계 |
|------|------|------|
| Memray | Python 메모리 프로파일링 | 힙 외부 할당 미감지 |
| Heaptrack | C/C++ 힙 추적 | mmap 직접 호출 미감지 |
| pmap | 메모리 맵 확인 | 원인 특정 불가 |
| BPFtrace | syscall 추적 | 커널 레벨 설정 필요 |
| GDB | 상세 디버깅 | 자동화 필요 |

### 협업의 결과

vLLM, NIXL, UCX 팀과의 협력으로:
- NIXL은 향후 릴리스에서 UCX_RCACHE_MAX_UNRELEASED에 기본값 설정 예정
- vLLM 저장소에 패치 병합됨 (PR #32181)

---

## Reference

- [Debugging memory leak in vLLM - Mistral AI](https://mistral.ai/news/debugging-memory-leak-in-vllm)
- [vLLM GitHub Repository](https://github.com/vllm-project/vllm)
- [UCX Documentation](https://openucx.org/)
