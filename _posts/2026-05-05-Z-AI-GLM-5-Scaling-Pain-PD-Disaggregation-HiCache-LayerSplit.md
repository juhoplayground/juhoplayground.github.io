---
layout: post
title: "GLM-5 Scaling Pain: PD 분리 KV 캐시 race, HiCache 동기화 누락, LayerSplit가 풀어낸 코딩 에이전트 서빙의 진짜 병목"
author: 'Juho'
date: 2026-05-05 00:00:00 +0900
categories: [LLM]
tags: [LLM, Caching, GPU]
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
2. [오프라인 재현과 이상 탐지](#오프라인-재현과-이상-탐지)
   - [재현 조건](#재현-조건)
   - [Speculative Decoding 지표를 이상 탐지 신호로](#speculative-decoding-지표를-이상-탐지-신호로)
3. [Bug Fix #1: PD 분리에서의 KV 캐시 Race](#bug-fix-1-pd-분리에서의-kv-캐시-race)
   - [근본 원인](#근본-원인)
   - [수정 방법](#수정-방법)
4. [Bug Fix #2: HiCache의 Load-Use Ordering 누락](#bug-fix-2-hicache의-load-use-ordering-누락)
5. [LayerSplit: Layer 단위 KV 캐시 분할 최적화](#layersplit-layer-단위-kv-캐시-분할-최적화)
6. [의미와 시사점](#의미와-시사점)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

z.ai가 [Scaling Pain of Coding Agent Serving](https://z.ai/blog/scaling-pain){:target="_blank"} 글에서 GLM-5 시리즈를 하루 수억 건의 코딩 에이전트 요청에 서빙하면서 마주친 시스템 레벨 버그와 그 수정 과정을 공개했다.
3월 이후 사용자들이 보고한 garbled output, repetition, rare-character generation은 표준 추론 환경에서는 나타나지 않고 고동시성 + 장문 컨텍스트 코딩 에이전트 워크로드에서만 등장하는 종류였다.
저자들은 이를 "Scaling Pain"이라고 부르며, 모델 자체의 품질 저하가 아닌 추론 인프라의 race condition 두 건을 식별하고 LayerSplit이라는 KV 캐시 분할 기법을 함께 도입했다.

## 오프라인 재현과 이상 탐지

### 재현 조건

처음 보고된 이상 출력의 원인을 가르는 1차 질문은 단순했다.
"모델인가, 인프라인가."
모델이 원인이라면 같은 입력에 대해 안정적으로 재현되어야 하고, 시스템 압력에 따라 달라진다면 인프라 쪽 문제다.
사용자 bad case를 로컬에서 수백 번 재현해도 이상이 나오지 않았다는 사실 자체가 이미 모델 원인 가설을 약화시켰다.

본격 재현은 production 로그를 익명화하여 동시성 분포와 요청 타이밍을 그대로 살린 풀스케일 리플레이로 이어졌다.

| 조건 | 재현 결과 |
|------|------|
| 단일 bad case 반복 재현 | 이상 출력 없음 |
| Production 로그 풀스케일 리플레이 (기본) | 여전히 이상 없음 |
| Prefill-Decode 비율 조정 + 부하 증가 | 1만 건당 약 3~5건 이상 출력 |

이상 출력이 요청 내용과 무관하면서도 시스템 압력과 상관관계를 가졌다는 것이 결정적 단서였다.
근본 원인은 고부하 상태의 추론 상태 관리에 있었다.

### Speculative Decoding 지표를 이상 탐지 신호로

이상 출력을 자동으로 잡아내는 것은 또 다른 문제였다.
Repetition은 비교적 쉽게 식별되지만 garbled output과 rare-character generation은 정규식이나 문자셋 매칭에 false positive와 false negative가 동시에 많고, 모델 기반 분류기는 대규모 ablation에 비용이 너무 컸다.

z.ai 팀이 발견한 우회 신호는 speculative decoding 지표였다.
원래 성능 최적화용으로 도입된 이 메커니즘이 의외로 KV 캐시 정합성의 실시간 시그널이 되었다.

| 이상 유형 | speculative decoding 시그널 | 해석 |
|------|------|------|
| Garbled / 희귀문자 | spec_accept_length 매우 낮음 | target 모델과 draft 모델 간 KV 캐시 상태 불일치 |
| Repetition | spec_accept_rate 매우 높음 | KV 캐시 손상 → attention 패턴 퇴화 → 고확신 반복 루프 |

이 관찰을 바탕으로 온라인 모니터링 정책이 도입되었다.
생성 길이가 128 토큰을 넘은 시점부터 spec_accept_length가 1.4 미만이거나 spec_accept_rate가 0.96 초과이면 현재 생성을 즉시 종료하고 로드 밸런서로 재시도시킨다.
성능 최적화 도구가 출력 품질의 실시간 게이트로 확장된 사례다.

## Bug Fix #1: PD 분리에서의 KV 캐시 Race

### 근본 원인

z.ai의 추론 엔진은 tail latency를 묶기 위해 timeout 기반 요청 종료를 사용한다.
Prefill이 정해진 budget 안에 끝나지 않으면 Decode가 요청을 abort하고 KV 캐시 슬롯을 회수한다.

문제는 abort 신호가 Prefill 쪽에 적절히 전파되지 않는다는 점이었고, Decode는 KV 캐시를 안전하게 회수·재사용해도 되는지 판단할 정보가 부족했다.
결과적으로 Decode가 abort 후 같은 KV 캐시 주소를 새 요청에 재할당해도, 이전 요청의 RDMA 쓰기와 Prefill 연산은 동기적으로 취소되지 않은 채 계속 진행되었다.

| 단계 | 내용 |
|------|------|
| Req1 | P1로 디스패치, queueing 지연 발생 |
| Decode timeout | Req1 abort, 동일 KV 캐시 슬롯 회수 |
| Req2 도착 | P2와 Decode에 할당, 같은 KV 캐시 주소 재사용 |
| 잔류 RDMA write | 이전 Req1의 P1 측 쓰기가 Req2의 KV 캐시 영역을 덮어씀 |
| 결과 | Req2가 손상된 KV 캐시를 읽고 이상 출력 발생 |

### 수정 방법

해결책은 요청 종료와 KV 캐시 쓰기 완료 사이에 명시적 동기화를 두는 것이다.

| 절차 | 동작 |
|------|------|
| Decode → Prefill 알림 | Decode가 abort 후 Prefill에 통지 |
| Prefill 안전 신호 | RDMA 쓰기가 시작되지 않았거나 모두 완료된 경우에만 safe-to-reclaim 반환 |
| Decode 회수 | 안전 신호 수신 후에만 KV 캐시 슬롯 재사용 |

이 메커니즘으로 KV 캐시 쓰기가 메모리 재사용 경계를 넘지 못하게 막아 cross-request KV 캐시 손상이 차단된다.
적용 후 이상 출력 비율은 약 0.1%에서 0.03% 미만으로 떨어졌다.
저자들은 PD 분리 아키텍처에서는 cross-node KV 캐시 전송과 메모리 재사용 사이의 명시적 정합성 보장이 필수라고 결론짓는다.

## Bug Fix #2: HiCache의 Load-Use Ordering 누락

코딩 에이전트 워크로드는 평균 입력 길이가 70K 토큰을 넘고 prefix 재사용률도 매우 높다.
이런 조건에서는 HiCache(Hierarchical KV Caching)가 핵심 최적화가 되지만, KV 캐시 swap-in이 연산과 오버랩될 때 데이터가 완전히 로드되기 전에 사용되는 read-before-ready 패턴이 발생할 수 있었다.

DSA 기반 HiCache 구현에서 시스템은 CPU 메모리에서 historical prefix cache를 비동기로 swap-in하면서 Load Stream과 Forward Stream을 분리해 연산과 오버랩시킨다.

| Stream | 역할 |
|------|------|
| Load Stream | KV 캐시와 Indexer 캐시 로딩 |
| Forward Stream | Index 연산 후 sparse attention 실행 |

문제는 Indexer 커널이 Indexer 캐시 로드 완료와 동기화 제약을 명시적으로 걸지 않았다는 점이다.
Forward Stream이 Load Stream의 데이터 로딩이 끝나기 전에 시작될 수 있고, 이로 인해 Indexer 연산이 불완전·미초기화 상태의 KV 캐시 위에서 수행되어 후속 sparse attention까지 오염시킨다.

수정은 단순하지만 결정적이었다.

| 변경 | 내용 |
|------|------|
| 명시적 동기화 포인트 | Indexer 커널 launch 직전에 Load Stream과의 sync 삽입 |
| 효과 | 해당 레벨의 Indexer 캐시 로드 완료 후에만 연산 진행 |

이 수정 이후 동일 워크로드에서 실행 순서 불일치로 인한 이상이 사라지고 시스템이 안정화되었다.
이 수정은 SGLang 커뮤니티에 PR #22811로 제출되었다.

## LayerSplit: Layer 단위 KV 캐시 분할 최적화

두 race condition이 공통적으로 드러낸 시스템 병목은 Prefill 단계가 long-context 코딩 에이전트 서빙에서 지배적인 성능 요인이라는 점이었다.
Timeout 기반 abort로 TTFT를 제어하고 HiCache로 KV 용량 압력을 줄인 뒤에도, 결국 Prefill 처리량과 GPU 메모리 압력 자체를 손봐야 했다.
이 자리에 도입된 것이 LayerSplit이다.

기존 SGLang 오픈소스 구현의 Context Parallelism은 모든 GPU가 모든 layer의 KV 캐시를 들고 있는 redundant storage 문제를 가졌다.
LayerSplit은 각 GPU가 일부 layer의 KV 캐시만 보유하도록 분할한다.

| 항목 | 기존 CP | LayerSplit |
|------|------|------|
| GPU별 KV 캐시 | 모든 layer 보유 | 일부 layer만 보유 |
| Prefill 시 통신 | 없음 | 해당 layer 소유 rank가 broadcast |
| 통신 오버헤드 | 0 | KV broadcast vs indexer 연산 오버랩으로 은폐 |
| 추가 비용 | - | Indexer 캐시 broadcast (KV 약 1/8 크기) |

각 attention 연산 직전, 해당 layer의 KV 캐시를 가진 rank가 다른 참여 rank로 broadcast하고, 이 통신을 indexer 연산과 오버랩시켜 latency를 가린다.
유일하게 남는 통신 비용인 indexer 캐시 broadcast는 KV 캐시 대비 1/8 수준이라 전체적으로 무시할 수 있는 규모다.

| 컨텍스트 길이 | 처리량 개선 (90% cache hit 기준) |
|------|------|
| 40K~120K 토큰 | 10% ~ 132% |

컨텍스트가 길수록 개선폭이 커진다는 결과는 long-context 코딩 에이전트 워크로드에서 KV 캐시 용량이 곧 GPU 활용률을 좌우한다는 점을 다시 확인시킨다.

## 의미와 시사점

이 글이 정리하는 교훈은 명확하다.
모델 능력이 확장될수록 추론 인프라에 잠겨 있던 가정들이 모델 품질 실패의 형태로 표면에 떠오른다.
사용자에게 보이는 garbled output이 실제로는 PD 분리 구조의 KV 캐시 race이거나 HiCache의 동기화 누락이었다는 사실은 추론 시스템 엔지니어링이 더 이상 throughput·latency·availability만으로 측정될 수 없다는 신호다.

| 측정 차원 | 전통적 SLA | 코딩 에이전트 서빙 |
|------|------|------|
| Throughput | 핵심 지표 | 여전히 중요 |
| Latency | 핵심 지표 | 여전히 중요 |
| Availability | 핵심 지표 | 여전히 중요 |
| Model state correctness | - | 새로 추가된 1급 지표 |

또 한 가지 인상적인 점은 이상 탐지 도구로 speculative decoding 지표를 재활용했다는 것이다.
원래는 디코딩 가속을 위한 메커니즘이었지만, draft-target 모델 간 일치율이 KV 캐시 정합성의 간접 신호임을 발견하면서 출력 품질의 실시간 게이트가 되었다.
시스템에 이미 존재하는 신호를 새로운 목적으로 재해석하는 패턴은 다른 추론 인프라에도 적용 가능한 발상이다.

## 결론

GLM-5 Scaling Pain은 코딩 에이전트 서빙이라는 새 워크로드가 만든 새로운 시스템 요구사항을 잘 보여준다.
두 개의 race condition(PD 분리 KV 캐시 정합성, HiCache load-use ordering)을 잡고 LayerSplit으로 Prefill 병목을 줄이는 일련의 작업은 모두 같은 결론을 가리킨다.
**확장은 모델만의 문제가 아니다.**
하루 수억 건 단위의 코딩 에이전트 추론에서는 cross-node 동기화, kernel pipeline의 atomicity, layer-wise 분할 같은 시스템 엔지니어링이 모델 품질 그 자체를 좌우한다.
SGLang PR #22811로 커뮤니티에 환원된 수정과 10~132%의 LayerSplit 처리량 개선은 그 결론을 뒷받침하는 구체적 증거다.

## Reference

- [Scaling Pain of Coding Agent Serving — z.ai](https://z.ai/blog/scaling-pain)
- [SGLang Pull Request #22811](https://github.com/sgl-project/sglang/pull/22811)
