---
layout: post
title: "DSpark: 반자기회귀 생성과 신뢰도 기반 검증으로 speculative decoding 가속하기"
author: 'Juho'
date: 2026-07-01 00:00:00 +0900
categories: [LLM]
tags: [LLM, vLLM, GPU]
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
2. [배경](#배경)
   - [speculative decoding의 기본 구조](#speculative-decoding의-기본-구조)
   - [두 가지 drafter 아키텍처와 병목](#두-가지-drafter-아키텍처와-병목)
3. [방법론](#방법론)
   - [반자기회귀 생성](#반자기회귀-생성)
   - [신뢰도 기반 검증](#신뢰도-기반-검증)
   - [학습 목적함수](#학습-목적함수)
4. [주요 결과](#주요-결과)
   - [오프라인 벤치마크](#오프라인-벤치마크)
   - [설계 공간 분석](#설계-공간-분석)
   - [프로덕션 배포 결과](#프로덕션-배포-결과)
5. [한계와 주의사항](#한계와-주의사항)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

DSpark는 DeepSeek-AI와 Peking University가 공개한 speculative decoding 프레임워크입니다.
speculative decoding은 경량 draft 모델이 여러 후보 토큰을 제안하고, 대형 target 모델이 이를 한 번의 forward pass로 검증하여 LLM 추론을 가속하는 기법입니다.
검증이 병렬로 이뤄지고 rejection sampling 규칙이 target 분포를 정확히 보존하기 때문에, 품질 손실 없이 생성 속도를 높일 수 있습니다.

최근의 parallel drafter는 단일 forward pass로 긴 토큰 블록을 제안할 수 있지만, 토큰 간 의존성을 모델링하지 못해 뒤쪽 위치로 갈수록 acceptance가 급격히 떨어지는 문제가 있습니다.
또한 이렇게 만들어진 긴 블록을 무차별적으로 검증하면 거부될 가능성이 높은 토큰에 배치 용량을 낭비하게 되어, 높은 동시성 서빙 환경에서 처리량이 심하게 저하됩니다.

DSpark는 이 두 문제를 각각 반자기회귀(semi-autoregressive) 구조와 신뢰도 기반(confidence-scheduled) 검증으로 해결합니다.
DeepSeek-V4 서빙 시스템에 배포했을 때, 기존 프로덕션 베이스라인인 MTP-1 대비 동일 처리량에서 사용자당 생성 속도를 60%에서 85%까지 가속하고 서빙 시스템의 Pareto frontier를 바깥으로 이동시켰습니다.
저자들은 DSpark 체크포인트와 함께 speculative decoding 학습 저장소인 DeepSpec을 오픈소스로 공개했습니다.

## 배경

### speculative decoding의 기본 구조

자기회귀 언어 모델은 forward pass 한 번에 토큰 하나만 생성하므로, 추론 지연이 출력 길이에 비례합니다.
speculative decoding은 target 모델의 추론을 경량 draft 모델로 가속합니다.
각 디코딩 사이클에서 draft 모델이 여러 개의 후보 토큰을 제안하면, target 모델이 이를 단일 forward pass로 검증하고 자신의 분포와 일치하는 가장 긴 prefix를 수락합니다.

검증은 왼쪽에서 오른쪽으로 진행되며, 특정 위치에서 첫 거부가 발생하면 그 이후 토큰은 품질과 무관하게 모두 폐기됩니다.
사이클당 수락 토큰 수를 alpha, drafting과 verification 패스의 실제 소요 시간을 각각 T_draft, T_verify라고 하면, 생성 토큰당 평균 지연은 drafting 시간과 verification 시간을 더한 값을 수락 토큰 수로 나눈 값이 됩니다.
따라서 speedup을 높이는 방법은 세 가지로 요약됩니다.
더 빠르게 draft하거나(T_draft 감소), 더 잘 draft하거나(alpha 증가), 더 똑똑하게 검증하는(유효 T_verify 감소) 것입니다.

### 두 가지 drafter 아키텍처와 병목

기존 drafter는 크게 두 부류로 나뉩니다.

자기회귀 drafter는 각 위치를 이전에 샘플링된 토큰에 조건화하여 순차적으로 생성합니다.
명시적 의존성 덕분에 모델링 능력은 강하지만, drafting 비용이 블록 크기에 선형으로 증가하기 때문에 작은 블록과 얕은 아키텍처를 쓸 수밖에 없습니다.

parallel drafter는 모든 draft 토큰을 단일 forward pass로 생성하여, drafting 지연이 블록 크기와 거의 무관합니다.
이 논문에서 기준으로 삼는 DFlash가 대표적인 state-of-the-art parallel drafter이며, target 모델의 여러 레이어 hidden state를 draft 공간으로 투영해 주입(KV injection)하는 방식으로 풍부한 문맥을 활용합니다.
DFlash의 draft 블록 내 모든 위치는 서로, 그리고 주입된 target 문맥에 양방향으로 attention합니다.

긴 병렬 블록의 잠재력을 끌어내려 할 때 두 가지 병목이 나타납니다.
첫째는 생성 품질 문제입니다.
parallel drafter는 각 위치를 독립적으로 예측하므로 블록 내 토큰 간 의존성을 모델링하지 못합니다.
예를 들어 "of course"와 "no problem"이 모두 그럴듯한 문맥에서, 각 위치가 실제로 샘플링된 선행 토큰에 조건화하지 못하고 모든 가능성을 주변화하기 때문에 "of problem" 같은 부적절한 조합을 만들어냅니다.
이런 multi-modal collision으로 인해 뒤쪽 위치에서 acceptance가 급격히 감소합니다.

둘째는 시스템 효율 문제입니다.
코드처럼 구조화된 요청은 자연스럽게 높은 acceptance를 유지하지만, 개방형 채팅은 훨씬 낮습니다.
또한 시스템 부하가 낮을 때는 추가 검증 비용이 거의 없지만, 부하가 높을 때는 거부 위험이 큰 토큰을 검증하는 것이 다른 활성 요청에 쓰일 배치 용량을 잠식합니다.

## 방법론

DSpark는 두 가지 상호보완적 구성요소로 위 병목을 해결합니다.

### 반자기회귀 생성

DSpark는 draft 생성을 두 단계로 분리합니다.

병렬 단계에서는 병렬 백본(이 논문의 구현에서는 DFlash)이 전체 블록에 대해 단일 forward pass를 수행하여 hidden state와 base logit을 만듭니다.
원본 DFlash와의 차이는 anchor 토큰 자체를 첫 예측 위치로 취급한다는 점입니다.
anchor에 이어 mask 토큰들을 입력해 블록 크기만큼의 draft logit을 얻으며, 이는 draft 계산량을 줄이면서 품질은 비슷하게 유지합니다.

순차 단계에서는 base logit에 prefix 의존적인 전이 bias를 더해, 각 draft 위치가 블록 내에서 이전에 샘플링된 토큰에 조건화되도록 합니다.
전역 정규화된 에너지 모델을 정의하는 대신, 자기회귀 인수분해를 통해 인과적 블록 분포를 유도합니다.
이 샘플링 과정은 본질적으로 순차적이므로, 순차 블록은 병렬 백본보다 훨씬 가벼워야 전체 draft 지연이 병렬 단계에 의해 지배됩니다.
논문은 두 가지 순차 블록 구현을 제시합니다.

첫째, Markov head는 가장 단순한 형태로, 전이 bias가 바로 앞 토큰에만 의존하는 1차 전이로 제한됩니다.
원칙적으로는 전체 어휘 크기의 정방 행렬이지만, 저랭크 인수분해(기본값 r=256)로 근사하여 임베딩 조회 테이블과 logit 투영으로 분해합니다.
이 방식은 저장 공간과 스텝당 연산을 작게 유지해, 큰 어휘에서도 순차 루프를 효율적으로 만듭니다.
앞의 예시에서 위치 1이 "of"를 샘플링하면, Markov head가 위치 2에서 "course"를 부양하고 "problem"을 억제하여 cross-mode collision을 완화합니다.

둘째, RNN head는 Markov head가 한 스텝 이전만 볼 수 있다는 한계를 완화합니다.
블록 내 전체 prefix 이력을 누적하는 recurrent state를 유지하며, 각 스텝에서 현재 state, 이전 토큰 임베딩, 백본 hidden을 결합해 게이트 업데이트를 적용합니다.

### 신뢰도 기반 검증

큰 draft 블록을 생성한다고 해서 자동으로 종단 간 speedup이 높아지지는 않습니다.
전체 블록을 무차별 검증하면 특히 고동시성 환경에서 시스템 처리량이 오히려 저하됩니다.
DSpark는 prefix 생존 확률을 예측하는 confidence head와, 시스템 부하에 따라 최적 검증 길이를 정하는 hardware-aware prefix scheduler를 결합해 이 문제를 해결합니다.

confidence head는 각 draft 위치에 대해 0과 1 사이의 스칼라 값을 출력합니다.
이 값은 선행 토큰이 모두 수락되었다는 조건 아래 해당 위치의 draft 토큰이 target 검증을 통과할 조건부 확률을 모델링합니다.
백본 hidden state와 직전 draft 토큰의 Markov 임베딩을 입력받아 경량 선형 투영과 sigmoid로 계산됩니다.
지도 신호로는 draft 분포와 target 분포 사이의 total variation distance로 결정되는 해석적 스텝별 acceptance rate를 사용합니다.

여기서 중요한 보정 단계가 Sequential Temperature Scaling(STS)입니다.
scheduler는 예상 수락 길이를 계산하기 위해 누적 acceptance 확률의 절대적 크기를 정확히 필요로 하는데, 신경망 신뢰도 추정치는 대체로 과신되는 경향이 있습니다.
STS는 각 신뢰도가 조건부 확률을 모델링한다는 점을 이용해, 연쇄 법칙에 따라 누적곱 형태로 표현되는 prefix 수락 결합 확률을 왼쪽에서 오른쪽으로 순차 보정합니다.
각 위치에서 앞선 위치의 보정값을 고정한 채 1차원 grid search로 Expected Calibration Error(ECE)를 최소화하는 온도 스칼라를 찾습니다.
온도 스케일링은 순서를 보존하는 변환이므로, confidence head가 학습한 상대적 순위를 흐트러뜨리지 않으면서 예측 확률을 경험적 수락률에 맞춥니다.

hardware-aware prefix scheduler는 검증 길이 선택을 전역 처리량 최대화 문제로 정식화합니다.
speculative decoding은 draft 토큰을 연속된 prefix로만 수락하므로, 어떤 위치의 생존 확률은 그 위치까지의 스텝별 신뢰도의 누적곱이 됩니다.
scheduler는 엔진 초기화 시 한 번 프로파일링해 경량 비용 테이블로 저장한 처리량 곡선 SPS(B)를 활용해, 예상 시스템 전체 토큰 처리량을 극대화하는 검증 길이를 선택합니다.
누적 생존 확률이 위치에 대해 단조 비증가하기 때문에, 후보 토큰을 생존 확률 기준으로 전역 정렬하는 greedy 방식이 블록 내 prefix 의존성을 자연스럽게 존중합니다.

다만 lossless speculative decoding은 non-anticipating 속성을 요구합니다.
즉 수락 결정이 아직 샘플링되지 않은 미래 후보 토큰에 의존해서는 안 됩니다.
confidence head가 직전 샘플링 토큰의 Markov feature에 의존하므로, 소급적인 전역 탐색은 현재 토큰 정보를 수락 결정에 누출시켜 selection bias를 유발합니다.
논문은 부록에서 이 위반을 보이는 구체적 반례를 제시합니다.
이를 막기 위해 scheduler는 처리량이 감소하는 순간 greedy 탐색을 즉시 중단하는 early-stopping을 적용해, truncation 결정이 해당 스텝까지의 prefix에만 의존하도록 인과성을 강제합니다.

### 학습 목적함수

학습 시 target 모델은 전 과정 동결되며, draft 모델은 임베딩 레이어와 language modeling head를 공유하되 동결하고, 백본 drafter, 순차 블록, confidence head만 업데이트합니다.
목적함수는 세 항으로 구성됩니다.

| 손실 항 | 역할 |
|------|------|
| cross-entropy 손실 | drafter가 올바른 다음 토큰을 예측하도록 학습 |
| distribution-matching 손실 | draft와 target 분포 간 total variation distance를 최소화하여 수락률 극대화 |
| confidence 손실 | soft acceptance 레이블을 예측하도록 confidence head를 학습하는 binary cross-entropy |

세 항은 모두 초반 블록 위치를 강조하는 위치 가중치로 조정됩니다.
prefix 기반 검증에서 초반 위치가 예상 수락 길이에 더 크게 기여하기 때문입니다.
기본 가중치는 cross-entropy 0.1, distribution-matching 0.9, confidence 1.0입니다.

## 주요 결과

### 오프라인 벤치마크

오프라인 평가에서는 순수 draft 품질을 분리하기 위해 confidence scheduler를 비활성화하고 모든 drafter가 고정 길이 블록을 제안하도록 했습니다.
평가 지표는 디코딩 라운드당 평균 수락 길이(accepted length)이며, 값이 클수록 좋습니다.
아래는 Qwen3-4B target 모델에서의 라운드당 수락 길이입니다.

| Drafter | GSM8K | MATH | AIME25 | MBPP | HumanEval | LCB | MT-Bench | Alpaca | Arena-Hard |
|------|------|------|------|------|------|------|------|------|------|
| Eagle3 | 5.14 | 4.62 | 3.92 | 3.69 | 4.16 | 3.77 | 2.39 | 2.26 | 2.55 |
| DFlash | 5.40 | 4.85 | 4.15 | 4.40 | 4.74 | 4.18 | 3.07 | 2.96 | 2.83 |
| DSpark | 6.11 | 5.70 | 4.89 | 5.13 | 5.38 | 4.86 | 3.64 | 3.54 | 3.29 |

DSpark는 모든 target 모델과 도메인에서 자기회귀 baseline(Eagle3)과 병렬 baseline(DFlash)을 일관되게 능가했습니다.
Qwen3-4B, 8B, 14B에서 매크로 평균 수락 길이 기준으로 Eagle3 대비 각각 30.9%, 26.7%, 30.0% 향상했습니다.
DFlash 대비로는 각각 16.3%, 18.4%, 18.3% 향상했으며, Gemma4-12B target에서도 일관된 성능 향상을 보여 모델 계열에 걸쳐 일반화됨을 확인했습니다.

또한 수락 길이는 수학이나 코드 같은 구조화된 작업에서 자연히 높고 개방형 채팅에서 낮은 강한 도메인 효과를 보였습니다.
이 데이터 예측 가능성의 편차가 정적 검증 길이가 뒤쪽 토큰에서 연산을 낭비하게 만드는 원인이며, 신뢰도 기반 검증의 직접적 동기가 됩니다.

### 설계 공간 분석

논문은 왜 병렬 생성이 자기회귀를 능가하는지를 위치별 조건부 acceptance로 분석했습니다.
이 지표는 앞선 모든 토큰이 수락된 경우만 분모로 세어, 이전 거부의 페널티를 제거하고 각 위치 고유의 예측 품질을 드러냅니다.

위치 1에서는 두 아키텍처 모두 target 문맥만으로 다음 토큰을 예측합니다.
Eagle3는 선형 지연 제약으로 얕은 네트워크에 묶이는 반면 parallel drafter는 훨씬 깊은 네트워크를 쓸 수 있어, DFlash가 Eagle3보다 위치 1에서 눈에 띄게 높게 시작합니다(예: 수학에서 0.88 대 0.81, 채팅에서 0.72 대 0.53).
speculative decoding은 엄격한 prefix 매칭 생존 과정이라 첫 토큰의 레버리지가 가장 크므로, 이 초기 우위가 최종 수락 길이를 크게 끌어올립니다.

반면 위치 2에서 7까지의 뒤쪽 구간에서는 독립적 병렬 생성의 한계가 드러납니다.
Eagle3는 조건부 확실성을 활용해 블록 깊이가 늘어도 조건부 acceptance를 유지하거나 오히려 높이지만(예: 채팅에서 0.53에서 0.74로 상승), DFlash는 급격한 acceptance 감소를 겪습니다(예: 코드에서 0.87에서 0.78, 채팅에서 0.72에서 0.63).
DSpark는 깊은 병렬 백본의 높은 초기 acceptance(예: 수학에서 0.93으로 시작)를 물려받으면서, 경량 순차 head로 뒤쪽 감소를 완화하여 블록 전체에서 높고 안정적인 조건부 acceptance를 유지합니다.

drafter 깊이 실험에서는 블록 크기를 7로 고정하고 DSpark 레이어를 1에서 5까지 변화시켰습니다.
성능은 깊이에 따라 단조 증가했고 1개에서 2개 레이어로 늘릴 때 한계 이득이 가장 컸으며, 놀랍게도 2레이어 DSpark가 5레이어 DFlash baseline을 모든 도메인에서 능가했습니다.
이는 경량 순차 head를 통한 국소 자기회귀 주입이 병렬 레이어를 단순히 쌓는 것보다 정확도 대비 파라미터 효율이 훨씬 좋음을 보여줍니다.

제안 길이 실험에서는 draft 길이를 4, 8, 12, 16으로 확장했습니다.
DSpark는 모든 길이에서 DFlash를 능가했고, 길이가 커질수록 격차가 벌어졌습니다.
블록 크기 7 근처에서 DSpark의 수락 길이 향상은 수학 16%, 코드 15%, 채팅 18%였으나, 길이 15에서는 각각 30%, 26%, 22%로 확대되었습니다.
RNN head는 긴 제안 길이에서 Markov head 대비 소폭의 추가 이득만 제공했고, 구현 복잡도와 배포 특성을 고려해 Markov head를 기본값으로 채택했습니다.

지연 오버헤드 측면에서는 배치 크기 128에서 순차 생성 루프의 비용을 측정했습니다.
이 배치 규모에서는 target 모델이 검증 연산 시간을 지배하므로 순차 블록의 오버헤드는 무시할 수준이며, draft 길이를 4에서 16으로 늘려도 DFlash baseline 대비 전체 라운드 지연에 0.2%에서 1.3%만 추가되면서 수락 길이는 최대 30% 개선되었습니다.

confidence head의 효과는 정적 임계값 스윕으로 검증했습니다.
임계값을 높일수록 궁극적으로 거부될 토큰을 걸러내어 전체 acceptance rate가 꾸준히 상승했습니다.
이 pruning은 고엔트로피 분포를 가진 채팅에서 가장 두드러져, 채팅의 acceptance rate가 45.7%에서 95.7%로 상승했습니다.
반면 수학과 코드는 더 완만하게 상승하여 각각 76.9%에서 92.5%, 67.6%에서 92.0%였습니다.

또한 raw confidence 모델은 강한 판별력(ROC-AUC 0.81에서 0.90)을 보였으나 과신되어 ECE가 3%에서 8%였고, STS 사후 보정을 적용하자 평균 ECE가 1%로 낮아져 신뢰할 수 있는 생존 확률 추정을 제공했습니다.

### 프로덕션 배포 결과

DSpark draft 모델은 DeepSeek-V4-Flash와 DeepSeek-V4-Pro 프리뷰 버전과 함께 배포되었습니다.
병렬 백본은 mHC와 128 슬라이딩 윈도우 attention을 가진 세 개의 MoE 레이어로 구성되며, 최대 블록 크기는 5, 순차 모델링은 Markov head를 사용합니다.
비교 대상인 MTP-1은 DeepSeek-V4-preview 출시 2주 후 DSpark로 대체된 이전 프로덕션 셋업으로, 정적 다중 토큰 drafter가 고동시성에서 과도한 검증 오버헤드로 처리량을 저하시키기 때문에 단일 토큰 방식으로 유지되어 왔습니다.

프로덕션에서는 고정 KV-cache 용량과 사용자 트래픽 풀 제약으로 유효 배치 크기가 GPU의 연산 포화 임계값보다 지속적으로 낮게 유지됩니다.
이 영역에서는 고정 동시성 한도 아래 GPU당 총 토큰 처리량 극대화와 사용자당 생성 속도 극대화가 경쟁이 아니라 강하게 상관된 목표가 됩니다.

라이브 사용자 트래픽 평가에서 DSpark는 SLA 앵커별로 성능을 측정했습니다.
SLA는 시스템이 보장해야 하는 사용자당 최소 생성 속도(초당 토큰)를 뜻합니다.
아래는 두 엔진에서 관측된 주요 수치입니다.

| 엔진 | SLA 앵커 | 처리량 개선 | 매칭 처리량에서 사용자당 속도 가속 |
|------|------|------|------|
| V4-Flash | 80 tok/s/user | 51% | 60%에서 85% |
| V4-Flash | 120 tok/s/user | 명목 661% | 엄격 구간에서 프론티어 확장 |
| V4-Pro | 35 tok/s/user | 52% | 57%에서 78% |
| V4-Pro | 50 tok/s/user | 명목 406% | 엄격 구간에서 프론티어 확장 |

120 tok/s/user와 50 tok/s/user 같은 엄격한 SLA 구간에서는 단일 토큰 MTP-1이 운영 한계에 근접해 매우 작은 동시 배치만 유지할 수 있습니다.
이 지점의 명목 661%, 406% 처리량 비율은 잘 활용된 baseline 대비 배수 speedup이라기보다, DSpark가 baseline이 효율적으로 지원할 수 없는 상호작용성 목표에서도 유의미한 서빙 용량을 유지함을 보여주는 증거로 해석됩니다.
매칭된 실제 처리량 수준이라는 더 안정적인 비교에서 DSpark는 V4-Flash에서 60%에서 85%, V4-Pro에서 57%에서 78% 더 빠른 사용자당 생성을 제공했습니다.

부하에 따른 동작도 분석되었습니다.
V4-Flash 200개 미만, V4-Pro 150개 미만의 중간 동시성 구간에서는 scheduler가 여유 target 연산을 활용해 검증 예산을 MTP-1의 정적 2 토큰에서 대략 4에서 6 토큰으로 확장했습니다.
동시성이 커지고 target 용량이 포화되면 scheduler가 이 예산을 동적으로 줄여, 저신뢰 draft 토큰이 배치 용량을 잠식하기 전에 제거합니다.
이 부하 적응 동작 덕분에 DSpark는 가벼운 트래픽에서 유휴 연산의 효용을 극대화하고 무거운 트래픽에서 핵심 배치 용량을 보존합니다.

프로덕션 배포에는 두 가지 시스템 적응이 필요했습니다.
실제 하드웨어의 SPS 곡선은 매끄럽지 않고 계단식으로 저하되며, 동적 draft 토큰 스케줄링이 CUDA graph replay와 Zero-Overhead Scheduling(ZOS)과 충돌합니다.
DSpark는 scheduler를 비동기로 운용하여, 두 스텝 전 confidence head 출력으로 다가올 검증 용량을 근사하되 현재 스텝 후보 토큰은 최신 누적 신뢰도로 정렬하는 dynamic top-k 방식으로 처리합니다.
이 비동기 설계는 미래 토큰 실현으로부터 수락 결정을 격리하는 인과적 장벽을 형성하여, ZOS와 매끄럽게 통합되면서 정확한 target 분포를 보존합니다.

## 한계와 주의사항

prefix scheduler가 낭비되는 target 검증을 최소화하더라도, DSpark는 병렬 백본으로 초기 블록을 생성하는 고정된 draft 측 비용을 여전히 부담합니다.
본질적으로 acceptance rate가 낮은 복잡한 질의에서는 이 사전 drafting 연산을 회수할 수 없습니다.
저자들은 향후 최적화로 draft 모델 내부에 난이도 인식 early exiting을 도입해, 이런 요청이 전체 블록 생성을 우회하도록 하는 방향을 제시합니다.

또한 early-stopping을 제거한 무제약 전역 탐색은 원래 lossless 보장을 위반하지만, 두 스텝 전 예측만 평가하는 ZOS 기반 비동기 적응이 인과적 장벽을 형성하여 이를 방지합니다.
이 단계별 early-stopping이 전역 최대 처리량을 산출하려면 목적함수가 단봉(unimodal)이어야 하며, 이는 매끄럽게 감소하는 하드웨어 용량 곡선을 암묵적으로 가정합니다.

## 결론

DSpark는 고동시성 프로덕션 환경에서 LLM 추론의 구조적, 시스템적 병목을 함께 해결하는 speculative decoding 프레임워크입니다.
알고리즘 측면에서는 무거운 병렬 백본에 경량 순차 head를 결합한 반자기회귀 생성으로 독립적 병렬 drafter의 급격한 뒤쪽 감소를 완화합니다.
시스템 측면에서는 검증 길이 선택을 전역 처리량 최대화 문제로 정식화하고, 보정된 생존 확률과 실시간 엔진 부하에 기반해 target 검증 예산을 동적으로 조절하는 hardware-aware prefix scheduler를 사용합니다.
DeepSeek-V4 프로덕션 배포에서 DSpark는 무거운 부하에서도 견고한 동시성을 유지하고 사용자당 생성 속도를 일관되게 가속하며, LLM 서빙의 Pareto frontier를 바깥으로 이동시켰습니다.

## Reference

- [DeepSpec: DSpark speculative decoding 학습 저장소](https://github.com/deepseek-ai/DeepSpec)
