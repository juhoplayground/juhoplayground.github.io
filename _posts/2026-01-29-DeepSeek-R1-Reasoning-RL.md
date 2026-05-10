---
layout: post
title: DeepSeek-R1 - 강화학습을 통한 LLM 추론 능력 향상
author: 'Juho'
date: 2026-01-29 01:00:00 +0900
categories: [AI]
tags: [AI, LLM]
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
2. [배경과 선행 연구](#배경과-선행-연구)
   - [test-time scaling과 o1 흐름](#test-time-scaling과-o1-흐름)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [DeepSeek-R1-Zero와 GRPO](#deepseek-r1-zero와-grpo)
   - [DeepSeek-R1 4단계 파이프라인](#deepseek-r1-4단계-파이프라인)
   - [Distillation](#distillation)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [DeepSeek-R1-Zero 자기진화와 aha moment](#deepseek-r1-zero-자기진화와-aha-moment)
   - [DeepSeek-R1 본 모델 성능](#deepseek-r1-본-모델-성능)
   - [Distilled 모델 6종](#distilled-모델-6종)
   - [Distillation vs RL 비교](#distillation-vs-rl-비교)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

DeepSeek-AI 팀의 "DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning"는 supervised fine-tuning 없이 순수 강화학습만으로 LLM이 자발적으로 고급 추론 패턴을 학습할 수 있음을 입증한 연구다.
저자는 Daya Guo, Dejian Yang, Haowei Zhang, Junxiao Song, Ruoyu Zhang을 비롯한 30여 명의 DeepSeek-AI 연구진이며, 2025년 1월 22일 arXiv에 공개되었고 Nature 645권에 게재되었다.

논문 abstract는 다음과 같은 핵심을 제시한다.
DeepSeek-R1-Zero는 base 모델에 SFT 없이 RL만 적용한 모델이며, DeepSeek-R1은 cold-start 데이터와 다단계 학습을 결합한 모델이다.
저자들은 두 모델과 함께 Qwen, Llama 기반 1.5B에서 70B까지의 distilled 변형 6종을 오픈소스로 공개했다.

## 배경과 선행 연구

### test-time scaling과 o1 흐름

본 논문은 OpenAI o1이 제시한 "test-time scaling" 흐름에 대응한다.
o1은 추론 시점의 chain-of-thought 길이를 늘려 정확도를 끌어올렸지만, 그 학습 방식은 비공개였다.
저자들은 reasoning 능력을 RL만으로 끌어낼 수 있음을 검증한 첫 공개 연구라고 자신의 위치를 명시한다.

기존 reasoning 모델은 모두 인간이 작성한 long-CoT 데이터에 의존한 SFT를 거쳤다.
이 데이터는 수집 비용이 크고 인간 추론 패턴의 편향을 모델에 전달한다.
RL로 SFT 단계를 우회할 수 있다면 데이터 비용을 크게 줄이고 모델이 인간 편향에서 자유로운 추론 경로를 탐색하게 된다.

### Related Work 정리

| 카테고리 | 주요 흐름 | 본 연구와의 차이 |
|----------|-----------|------------------|
| Reasoning via SFT | OpenAI o1 (비공개), Qwen-QwQ, DeepSeek-Math | 인간 작성 CoT 의존 — 본 연구는 RL only |
| Process Reward Models | PRM800K, Math-Shepherd | 단계별 보상 — 본 연구는 결과 기반 rule reward |
| Self-improvement | Self-Refine, STaR | 단일 모델 반복 — 본 연구는 RL 자기진화 |
| MCTS reasoning | AlphaZero 계열 시도 | 탐색 트리 폭증 — 본 연구는 single-pass RL |

저자들은 PRM과 MCTS를 직접 시도했지만 두 방향 모두 실패로 보고했다(섹션 4.2).

## 방법론

### DeepSeek-R1-Zero와 GRPO

DeepSeek-R1-Zero는 DeepSeek-V3-Base에 SFT를 거치지 않고 곧장 GRPO(Group Relative Policy Optimization)를 적용한다.
GRPO는 critic 모델 없이 그룹 내 샘플의 상대 점수로 advantage를 추정해 PPO 대비 메모리 비용을 크게 줄인다.

```
J_GRPO(θ) = E[ 1/G * Σ ( min( π_θ/π_old * A_i,
              clip(π_θ/π_old, 1-ε, 1+ε) * A_i ) - β * D_KL(π_θ || π_ref) ) ]
```

advantage는 그룹 내 reward의 z-score로 계산된다.

```
A_i = (r_i - mean(rewards)) / std(rewards)
```

reward는 두 종류다.
첫째, accuracy reward — 수학 문제는 형식화된 답의 정답성, 코드는 컴파일러 기반 테스트 통과 여부로 룰 기반 검증된다.
둘째, format reward — 모델 출력이 `<think>` 태그와 `<answer>` 태그를 갖추는지 검사한다.
저자들은 neural reward model이 reward hacking에 취약함을 명시적으로 언급하며 사용을 배제한다.

학습 템플릿은 의도적으로 최소한의 구조만 제약한다.
`<think>...</think>` 안에 추론 과정을, 그 뒤에 `<answer>` 태그로 답을 출력하라는 지시 외에는 특정 추론 전략을 강요하지 않아 자연스러운 진화를 관찰할 수 있게 한다.

### DeepSeek-R1 4단계 파이프라인

DeepSeek-R1은 R1-Zero의 가독성과 언어 일관성 문제를 해결하기 위해 4단계 파이프라인을 거친다.

| 단계 | 내용 | 데이터 규모 |
|------|------|--------------|
| 1. Cold Start | few-shot prompting과 R1-Zero 출력 후처리로 high-quality CoT 수집 | 수천 건 |
| 2. Reasoning RL | GRPO + language consistency reward 추가 | — |
| 3. Rejection Sampling + SFT | RL 체크포인트로 reasoning 600k, non-reasoning 200k 샘플 생성 후 2-epoch SFT | 약 800k |
| 4. General RL | reasoning은 rule reward, 일반 도메인은 학습된 reward로 2차 RL | — |

cold start 출력 형식은 `|special_token|<reasoning_process>|special_token|<summary>` 구조로 마크다운 가독성과 언어 일관성을 갖추도록 설계되었다.
language consistency reward는 CoT에서 목표 언어 단어가 차지하는 비율로 계산되며 accuracy reward와 직접 합산된다.

### Distillation

DeepSeek-R1이 만든 800k 샘플로 더 작은 base 모델 6종을 직접 SFT한 distilled 변형이 함께 공개되었다.
RL 단계는 적용하지 않고 SFT만 수행했다.

| Base 모델 | Distilled 변형 |
|-----------|----------------|
| Qwen2.5-Math-1.5B | DeepSeek-R1-Distill-Qwen-1.5B |
| Qwen2.5-Math-7B | DeepSeek-R1-Distill-Qwen-7B |
| Qwen2.5-14B | DeepSeek-R1-Distill-Qwen-14B |
| Qwen2.5-32B | DeepSeek-R1-Distill-Qwen-32B |
| Llama-3.1-8B | DeepSeek-R1-Distill-Llama-8B |
| Llama-3.3-70B-Instruct | DeepSeek-R1-Distill-Llama-70B |

## 실험 셋업

| 항목 | 값 |
|------|-----|
| Base 모델 | DeepSeek-V3-Base (671B 총 / 37B activated MoE) |
| RL 알고리즘 | GRPO |
| 학습 reward | rule-based (accuracy + format + language consistency) |
| Reasoning SFT | 600k samples |
| Non-reasoning SFT | 200k samples |
| 총 SFT corpus | 약 800k, 2 epochs |
| Sampling temperature | 0.6 |
| Top-p | 0.95 |
| Max generation tokens | 32,768 |
| AIME 평가 | cons@64 (majority voting) |
| 평가 judge | GPT-4-Turbo-1106 (open-ended) |

평가 벤치마크는 수학(AIME 2024, MATH-500, CNMO 2024), 지식(MMLU, MMLU-Redux, MMLU-Pro, GPQA Diamond, C-Eval, CMMLU), 코드(LiveCodeBench, Codeforces, SWE-Bench Verified, HumanEval-Mul), 일반(AlpacaEval 2.0, Arena-Hard, SimpleQA, IFEval, FRAMES)을 포괄한다.

## 주요 결과

### DeepSeek-R1-Zero 자기진화와 aha moment

R1-Zero는 SFT 없이도 AIME 2024 pass@1을 학습 시작 시점 15.6%에서 71.0%까지 끌어올렸다.
cons@64 majority voting을 적용하면 86.7%로 OpenAI o1-0912와 비슷한 수준이 된다.

| 벤치마크 | DeepSeek-R1-Zero |
|----------|-------------------|
| AIME 2024 pass@1 | 71.0% |
| AIME 2024 cons@64 | 86.7% |
| MATH-500 pass@1 | 95.9% |
| GPQA Diamond pass@1 | 95.9% |
| LiveCodeBench pass@1 | 73.3% |
| Codeforces rating | 1444 |

학습이 진행되면서 평균 응답 길이가 약 1,000 토큰에서 5,000 토큰 이상으로 증가했고, 자기 점검(reflection)과 대안 탐색(exploration of alternatives) 같은 행동이 명시적 학습 없이 자발적으로 등장했다.
저자들은 학습 도중 모델이 "Wait, wait. Wait. That's an aha moment I can flag here" 같은 표현으로 스스로 추론 단계를 재고하는 순간을 aha moment로 보고한다.

### DeepSeek-R1 본 모델 성능

| 벤치마크 | DeepSeek-R1 | OpenAI o1-1217 | DeepSeek-V3 |
|----------|-------------|-----------------|-------------|
| AIME 2024 pass@1 | 79.8% | 79.2% | 39.2% |
| MATH-500 pass@1 | 97.3% | 96.4% | 90.2% |
| MMLU pass@1 | 90.8% | 91.8% | 88.5% |
| MMLU-Pro | 84.0% | — | 75.9% |
| GPQA Diamond | 71.5% | 75.7% | 59.1% |
| LiveCodeBench | 65.9% | 63.4% | 36.2% |
| Codeforces rating | 2029 | 2061 | 1134 |
| Codeforces percentile | 96.3% | 96.6% | 58.7% |
| AlpacaEval 2.0 LC-winrate | 87.6% | — | 70.0% |
| Arena-Hard winrate | 92.3% | — | 85.5% |

DeepSeek-R1은 AIME, MATH-500, LiveCodeBench, Arena-Hard에서 o1-1217을 상회하고 GPQA Diamond에서만 다소 뒤졌다.
모델 크기는 671B 전체 / 37B activated MoE 구조다.

### Distilled 모델 6종

| 모델 | AIME 2024 | MATH-500 | GPQA Diamond | LiveCodeBench | Codeforces |
|------|-----------|----------|---------------|----------------|-------------|
| R1-Distill-Qwen-1.5B | 28.9% | 83.9% | 33.8% | 16.9% | 954 |
| R1-Distill-Qwen-7B | 55.5% | 92.8% | 49.1% | 37.6% | 1189 |
| R1-Distill-Qwen-14B | 69.7% | 93.9% | 59.1% | 53.1% | 1481 |
| R1-Distill-Qwen-32B | 72.6% | 94.3% | 62.1% | 57.2% | 1691 |
| R1-Distill-Llama-70B | 70.0% | 94.5% | 65.2% | 57.5% | 1633 |
| QwQ-32B-Preview | 50.0% | 90.6% | 54.5% | 41.9% | 1316 |
| OpenAI o1-mini | 63.6% | 90.0% | 60.0% | 53.8% | 1820 |

R1-Distill-Qwen-7B는 32B QwQ-Preview를 상회했고, 14B 변형부터는 모든 지표에서 QwQ-32B-Preview를 능가한다.
32B와 70B 변형은 OpenAI o1-mini를 추론 벤치마크 전반에서 상회한다.

### Distillation vs RL 비교

저자는 distillation의 효율성을 직접 검증하기 위해 같은 Qwen-32B base에 두 가지 처리를 비교했다.

| 모델 | AIME 2024 pass@1 | AIME 2024 cons@64 |
|------|-------------------|---------------------|
| R1-Zero-Qwen-32B (RL only) | 47.0% | 60.0% |
| R1-Distill-Qwen-32B (distill from R1) | 72.6% | 83.3% |

결론은 명확하다.
큰 모델을 distillation 소스로 사용하는 편이 작은 모델에 직접 RL을 돌리는 것보다 훨씬 우수하며, 작은 모델의 단순 RL로는 distillation 수준에 도달하지 못한다.

## 한계와 디스커션

저자가 명시한 한계는 네 가지다.
첫째, function calling, multi-turn 대화, 복잡한 role-playing, JSON 출력에서 DeepSeek-R1이 DeepSeek-V3보다 떨어진다.
둘째, 영어와 중국어 외 언어에서 language mixing이 발생한다.
셋째, 프롬프트에 매우 민감하며 few-shot prompting이 일관되게 성능을 떨어뜨린다.
넷째, 코드 평가에 시간이 오래 걸려 SWE 류 소프트웨어 엔지니어링 RL 적용이 제한적이다.

저자가 보고한 unsuccessful attempts도 두 가지다.
첫째, Process Reward Model(PRM)은 단계 정의의 모호성, 정답 검증 어려움, reward hacking, 추가 학습 자원 부담 등의 이유로 실패했다.
둘째, MCTS는 token 단위 탐색 공간이 체스 대비 지수적으로 크고, 최대 확장 한계 설정이 local optima를 유발하며, value model 학습이 어려워 자기 탐색 기반 반복 향상에 도달하지 못했다.

future work으로 long-CoT를 활용한 일반 능력 강화, 다국어 mixing 해소, 프롬프트 견고성 개선, 비동기 평가를 통한 SWE RL 확장이 제시된다.

## 결론

DeepSeek-R1-Zero는 SFT 없이 RL만으로 reasoning을 끌어낼 수 있음을 AIME 71.0%(cons@64 86.7%)로 입증했다.
DeepSeek-R1은 cold start와 4단계 파이프라인을 통해 가독성과 언어 일관성을 확보하면서 o1-1217과 동급의 성능에 도달했다(AIME 79.8%, MATH-500 97.3%, Codeforces 2029).
800k 샘플 distillation으로 1.5B에서 70B까지 6종 변형을 오픈소스 공개했고, 14B부터 QwQ-32B-Preview를 모든 지표에서 능가하며 32B/70B는 OpenAI o1-mini를 상회한다.
distillation이 작은 모델에 대한 직접 RL보다 우월함을 정량적으로 보였다는 점이 LLM reasoning 연구 커뮤니티에 큰 시사점을 남긴 연구다.

## Reference

- [DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning (arXiv:2501.12948)](https://arxiv.org/abs/2501.12948/)
- [DeepSeek-R1 GitHub](https://github.com/deepseek-ai/DeepSeek-R1/)
