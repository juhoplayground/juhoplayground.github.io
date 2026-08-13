---
layout: post
title: "Qwen 샘플링·디코딩 파라미터 presence_penalty, repetition_penalty, temperature"
author: 'Juho'
date: 2026-08-13 00:00:00 +0900
categories: [LLM]
tags: [LLM, vLLM, Ollama]
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
2. [세 가지 페널티의 정의와 차이](#세-가지-페널티의-정의와-차이)
   - [presence_penalty와 frequency_penalty (가산형)](#presence_penalty와-frequency_penalty-가산형)
   - [repetition_penalty (곱셈형, CTRL 방식)](#repetition_penalty-곱셈형-ctrl-방식)
3. [Qwen이 감산형을 선호하는 이유](#qwen이-감산형을-선호하는-이유)
4. [Qwen 공식 권장 generation 설정](#qwen-공식-권장-generation-설정)
5. [무한 반복 문제와 완화 전략](#무한-반복-문제와-완화-전략)
6. [커뮤니티의 비판적 관점](#커뮤니티의-비판적-관점)
7. [서빙 엔진별 파라미터 노출 차이](#서빙-엔진별-파라미터-노출-차이)
8. [실전 권장 설정 예시](#실전-권장-설정-예시)
9. [결론](#결론)
10. [Reference](#reference)

## 개요

Qwen 계열 LLM을 서빙하다 보면 같은 문장을 끝없이 반복하는 endless repetition 문제를 자주 만난다.
이 문제를 다루는 핵심 도구가 샘플링·디코딩 파라미터, 그중에서도 세 가지 페널티다.
`presence_penalty`, `frequency_penalty`, `repetition_penalty`는 모두 "이미 등장한 토큰의 재등장을 억제"하지만, 적용하는 수학적 연산과 적용 대상이 서로 다르다.

특히 Qwen은 다른 오픈소스 모델과 다른 선택을 한다.
곱셈형 `repetition_penalty`를 사실상 비활성(1.0)으로 두고, 감산형 `presence_penalty`로 반복을 제어하는 방향으로 튜닝되어 있다.
이 글은 세 페널티의 수식과 차이, Qwen 공식 권장값, 무한 반복 완화 전략, 그리고 커뮤니티에서 제기되는 비판까지 1차 소스 기준으로 정리한다.

이 글이 다루는 파라미터의 동작 원리는 특정 버전에 종속되지 않는다.
Qwen 계열은 Qwen2.5, Qwen3, Qwen3-2507을 거쳐 Qwen3.5, 그리고 현재 최신인 Qwen3.6까지 공개되었는데, 세대가 바뀌어도 반복 제어 철학(곱셈형 off + 감산형 조절)은 그대로 유지된다.
버전별 권장 샘플링 값 차이는 뒤의 공식 권장 설정 섹션에서 별도로 다룬다.

## 세 가지 페널티의 정의와 차이

세 파라미터는 크게 두 계열로 나뉜다.
`presence_penalty`와 `frequency_penalty`는 OpenAI API 계열의 가산형(로짓 뺄셈)이고, `repetition_penalty`는 HuggingFace·로컬 배포 계열의 곱셈형이다.

### presence_penalty와 frequency_penalty (가산형)

두 파라미터는 이미 등장한 토큰의 logit(raw score)에서 값을 빼는 방식이다.
차이는 "등장 여부"를 보느냐 "등장 횟수"를 보느냐에 있다.

- presence_penalty는 토큰이 지금까지 생성 텍스트에 한 번이라도 등장했는가만 본다(빈도 무관, 이진적).
- frequency_penalty는 토큰이 몇 번 등장했는가에 비례하여 더 강하게 페널티를 준다.

OpenAI가 정의하고 vLLM이 준거로 삼는 canonical 로짓 조정 공식은 다음과 같다.

```
mu[j] -> mu[j] - c[j] * frequency_penalty - float(c[j] > 0) * presence_penalty
```

여기서 `mu[j]`는 토큰 j의 로짓, `c[j]`는 지금까지 생성 텍스트에서 토큰 j가 등장한 횟수다.
`frequency_penalty`는 등장 횟수 `c[j]`에 비례해 로짓을 감소시키고, `presence_penalty`는 한 번이라도 등장했으면(`c[j] > 0`) 고정값만큼 로짓을 감소시킨다.
두 파라미터 모두 중립값은 0.0이고 유효 범위는 -2.0 ~ 2.0이다.
양수는 새 토큰 사용을 장려(반복 억제)하고, 음수는 반복을 장려한다.

### repetition_penalty (곱셈형, CTRL 방식)

`repetition_penalty`는 오픈소스 진영(HuggingFace, vLLM, llama.cpp 등)의 표준 파라미터로, Keskar et al. (2019)의 CTRL 논문(arXiv:1909.05858)에서 도입되었다.
가산형과 달리 곱셈(multiplier) 방식이며, 중립값은 1.0이다.

- penalty가 1.0이면 페널티 없음.
- penalty가 1.0보다 크면 이미 등장한 토큰의 확률 감소(반복 억제).
- penalty가 0보다 크고 1.0보다 작으면 이미 등장한 토큰의 확률 증가(반복 유도).

CTRL 논문 저자는 진실성과 반복 억제의 균형점으로 θ 약 1.2를 권장했다.
구현에는 주의할 부분이 있다.
초기 구현은 logit을 단순히 페널티로 나눴는데, logit이 음수일 때는 나누면 오히려 확률이 올라가는 버그가 있었다.
그래서 HuggingFace(PR #2303) 등은 양수 logit은 페널티로 나누고 음수 logit은 페널티로 곱하는 sign-branching 방식으로 수정했고, 이 방식이 vLLM·llama.cpp 등 생태계 전반에 퍼졌다.

세 페널티를 한눈에 비교하면 다음과 같다.

| 파라미터 | 연산 방식 | 적용 기준 | 중립값 | 범위 |
|------|------|------|------|------|
| presence_penalty | 로짓 뺄셈(가산형) | 등장 여부(이진) | 0.0 | -2.0 ~ 2.0 |
| frequency_penalty | 로짓 뺄셈(가산형) | 등장 횟수(비례) | 0.0 | -2.0 ~ 2.0 |
| repetition_penalty | 로짓 곱셈(CTRL 방식) | 프롬프트+생성 텍스트 등장 | 1.0 | 0.0 초과 |

참고로 HuggingFace Transformers의 `GenerationConfig`에는 `presence_penalty`와 `frequency_penalty` 파라미터 자체가 존재하지 않는다.
HF에서는 반복 억제를 `repetition_penalty`(곱셈형)와 `no_repeat_ngram_size`(해당 크기 n-gram을 1회만 허용하는 하드 제약)로 처리한다.

## Qwen이 감산형을 선호하는 이유

`presence_penalty=1.5`, `repetition_penalty=1.0`은 Qwen 서빙에서 자주 보이는 조합이다.
이 설정은 곱셈형을 끄고(1.0 = 무페널티) 감산형으로만 반복을 제어하겠다는 의미다.

Qwen 진영이 감산형을 선호하는 핵심 이유는 조절의 부드러움이다.
Qwen 메인테이너(jklj077)의 설명에 따르면 `presence_penalty`는 입력 컨텍스트에 이미 등장한 토큰의 logit에서 값을 빼는 방식이라 조절이 상대적으로 부드럽다.
1.5 정도면 반복 문장 문제를 완화하고, 반면 50.0 같은 극단값은 해당 토큰 확률을 0에 수렴시켜 반복을 거의 확실히 피하지만 부자연스럽거나 비일관적인 텍스트를 낳는다.

반대로 `repetition_penalty`를 1.0으로 두는 것이 관행인 이유는 Qwen이 이 곱셈형 페널티에 유독 민감하기 때문이다.
커뮤니티의 반복된 관찰에 따르면 Qwen 모델은 다른 모델처럼 1.1~1.2만 걸어도 출력 품질이 붕괴한다.
그래서 기본 1.0(off)을 유지하고, 반복이 심할 때만 `presence_penalty`를 켜는 것이 사실상 합의된 접근이다.

이 방향은 Qwen3-VL 계열의 튜닝에서도 드러난다.
Qwen3-VL의 generation_config는 `repetition_penalty=1.0`(비활성)을 유지하면서, 대신 `presence_penalty`(4B의 경우 VL 태스크 1.5, 텍스트 태스크 2.0)로 반복을 제어하는 방향으로 조정되었다.

다만 Qwen이 곱셈형을 완전히 배제하는 것은 아니다.
Qwen2.5 계열은 `generation_config.json`에 `repetition_penalty=1.05`라는 약한 값을 기본 탑재하며, Qwen3-Coder도 예외적으로 `repetition_penalty=1.05` 수준을 권장한다.
즉 Qwen2.5는 약한 곱셈형 기본값을 내장하는 반면, Qwen3의 권장 전략은 `presence_penalty` 기반으로 이동했다고 정리할 수 있다.

## Qwen 공식 권장 generation 설정

Qwen3는 thinking / non-thinking 하이브리드 모델이며, 공식 모델 카드와 문서가 모드별 권장 샘플링 파라미터를 명시한다.

| 파라미터 | Thinking 모드 | Non-Thinking 모드 |
|------|------|------|
| temperature | 0.6 | 0.7 |
| top_p | 0.95 | 0.8 |
| top_k | 20 | 20 |
| min_p | 0 | 0 |
| presence_penalty | 0~2 조정(반복 완화용) | 0~2 조정, vLLM 예시는 1.5 사용 |

Qwen 공식은 특정 최적 `presence_penalty` 값을 못박지 않고 범위(0~2)와 트레이드오프만 안내한다.
Qwen의 공식 vLLM 배포 예시에서 non-thinking 코드에 `presence_penalty=1.5`, `max_tokens=8192`가 사용되고, thinking 예시는 `temperature=0.6`, `top_p=0.95`, `top_k=20`, `max_tokens=32768`을 쓴다.

최신 세대인 Qwen3.6은 Qwen3.6-27B(Dense)와 Qwen3.6-35B-A3B(MoE) 두 오픈웨이트 모델로 공개되었고, 마찬가지로 thinking을 기본으로 하며 `enable_thinking=False`로 non-thinking으로 전환한다.
공식 모델 카드가 제시하는 Qwen3.6 권장 샘플링은 다음과 같다.

| 파라미터 | Thinking(일반) | Thinking(정밀 코딩) | Non-Thinking |
|------|------|------|------|
| temperature | 1.0 | 0.6 | 0.7 |
| top_p | 0.95 | 0.95 | 0.80 |
| top_k | 20 | 20 | 20 |
| min_p | 0 | 0 | 0 |
| presence_penalty | 0 (반복 시 0~2 상향) | 0 | 1.5 |
| repetition_penalty | 1.0 | 1.0 | 1.0 |

이전 세대와 비교했을 때 가장 뚜렷한 변화는 thinking 모드 권장 temperature가 0.6에서 1.0으로 올라간 점이다.
정밀 코딩 작업은 여전히 0.6을 권장하고, non-thinking(0.7/0.80/20)과 top_p·top_k·min_p·repetition_penalty 체계는 사실상 그대로다.
`repetition_penalty=1.0`으로 곱셈형을 끄고 감산형 `presence_penalty`(0~2)로 반복을 제어하는 Qwen의 기조도 유지된다.
Qwen3.6-35B-A3B의 generation_config.json 실측 기본값도 `temperature=1.0`, `top_p=0.95`, `top_k=20`, `do_sample=true`로, greedy가 아닌 샘플링이 기본임을 보여준다.

한 가지 주의할 점은 Qwen3.6-35B-A3B(MoE)의 thinking 일반 프리셋 `presence_penalty` 값이 모델 카드 리비전 사이에서 0.0과 1.5로 엇갈려 기재된 이력이 있다는 것이다(Dense 27B 카드는 0.0).
실사용 시에는 자신이 내려받은 정확한 모델 카드 리비전의 값을 확인하는 편이 안전하다.

Qwen2.5의 `generation_config.json` 실제 기본값은 다음과 같다.

| 파라미터 | 값 |
|------|------|
| temperature | 0.7 |
| top_p | 0.8 |
| top_k | 20 |
| repetition_penalty | 1.05 |
| do_sample | true |

가장 강조되는 공식 경고는 greedy decoding 금지다.
Qwen 문서는 "DO NOT use greedy decoding, as it can lead to performance degradation and endless repetitions"라고 명시한다.
즉 `temperature=0`(greedy) 대신 반드시 temperature/top_p 샘플링을 써야 반복을 회피할 수 있다.
또한 공식은 "always pass the sampling parameters to the API", 즉 기본값에 의존하지 말고 파라미터를 명시적으로 전달하라고 권한다.

temperature/top_p/top_k/min_p는 후보 토큰 분포를 좁히거나 넓히는 필터이고, 페널티 3종은 이미 등장한 토큰의 점수를 깎는 억제 장치다.
temperature를 너무 낮추거나 greedy로 가면 반복 위험이 커지므로, temperature를 적정 수준(0.6~0.7)으로 유지하는 것이 Qwen 권장 접근이다.

## 무한 반복 문제와 완화 전략

Qwen 계열의 endless repetition은 여러 실사용 사례로 보고되었다.
Qwen2 72B Chat AWQ가 vLLM OpenAI 호환 API에서 무한 루프에 빠져 GPU를 계속 점유하는 버그, Qwen3-VL의 무한 반복 이슈, KV-cache 활성 시 Qwen3-VL의 반복 루프와 성능 저하 등이다.

공식 문서가 안내하는 1차 완화책은 `presence_penalty`를 0~2 사이로 올리는 것이다.
심각한 무한 반복이 발생하면 `presence_penalty=1.5`로 설정하라는 것이 대표적 처방이다.
실사용 후기에서도 Qwen3-4B-Instruct 사용자가 여러 값을 실험한 끝에 약 1.9로 수렴해 안정화한 사례가 공유된다.

그러나 여기에는 명확한 트레이드오프가 있다.
Qwen 공식 문서가 직접 경고하듯, `presence_penalty`를 높게 주면 간혹 language mixing(언어 혼용)과 약간의 성능 저하가 발생할 수 있다.
반복을 잡으려다 한국어 출력에 영어나 중국어가 섞이는 부작용이 나타날 수 있다는 뜻이다.
그래서 한국어 등 다국어 사용자는 값을 낮게(1.0 근처) 두거나 temperature/top_p로 우회하라는 조언이 많다.

또한 long-context와 YaRN이 함께 활성화된 상황에서는 어떤 페널티로도 반복이 잡히지 않는 경우가 보고된다.
이 경우는 파라미터가 아니라 모델·서빙 스택 레벨의 문제로 보아야 한다는 시각이 있다.

## 커뮤니티의 비판적 관점

커뮤니티는 페널티 상향을 근본 해결책으로 보지 않는 시각이 강하다.
가장 핵심적인 비판은 `presence_penalty`나 `repetition_penalty`를 올리는 것이 구조적으로 반복이 필요한 출력을 망가뜨린다는 점이다.

Qwen3-VL의 무한 반복 이슈(#1611)에서 사용자는 이미지를 마크다운으로 변환할 때 표 구분선(`|`) 등이 수십 번 반복되는 무한 루프를 보고했다.
그러면서 "repetition penalty를 올리는 것은 받아들일 수 없는 해법 — 표처럼 자연스럽게 반복되는 텍스트의 전사(transcription)를 망가뜨린다"고 못박았다.
즉 페널티는 OCR, 표, 코드 전사 같은 작업에서 오히려 정확도를 해치는 증상 완화책일 뿐이라는 것이다.

`presence_penalty=1.5`를 기본값으로 박아둔 것에 대한 반발도 크다.
Qwen3-VL 이슈 #1825는 큰 모델의 generation_config에는 `presence_penalty`가 아예 없는데 Qwen3-VL-4B와 평가 재현 코드에는 `presence_penalty=1.5`가 고정되어 있는 모순을 지적한다.
사용자는 이 설정에서도 토큰 반복을 겪었다고 보고했고, 값이 모델과 문서마다 제각각(0 / 1.0 / 1.5)이라 근거가 불투명하다는 혼란이 반복적으로 제기된다.

실사용 수렴값은 상황에 따라 다르게 공유된다.
문장 단위 반복에는 `presence_penalty` 약 1.9가 안정적이었다는 보고가 있고, 롤플레이 반복 대응 경험값으로는 temperature 0.6~0.8, top_p 0.9~1.0, frequency_penalty 약 0.1, repetition_penalty 1~1.x 범위에 프롬프트 엔지니어링을 병행하라는 조언이 나온다.
학술적으로도 기존 repetition/frequency penalty가 반복을 확실히 못 잡거나, 너무 높으면 공백·마침표 같은 필수 토큰까지 억눌러 품질을 떨어뜨린다는 지적(LZ Penalty 논문)이 있다.

## 서빙 엔진별 파라미터 노출 차이

같은 개념의 파라미터라도 서빙 엔진에 따라 이름과 노출 방식이 다르다.
이 차이를 모르면 툴 연동에서 마찰이 생긴다.

- vLLM: OpenAI text completion API를 준거로 삼아 `presence_penalty`(기본 0.0), `frequency_penalty`(기본 0.0), `repetition_penalty`(기본 1.0), `min_p`(기본 0.0)를 모두 노출한다. presence/frequency는 "생성 텍스트 등장 기반", repetition은 "프롬프트+생성 텍스트 등장 기반"으로 문서상 구분한다.
- SGLang: vLLM과 사실상 동일한 4종(presence 0.0, frequency 0.0, repetition 1.0, min_p 0.0)을 노출하며 설명 문구도 거의 같다.
- Ollama: 명칭이 다르다. `repeat_penalty`(기본 1.1, 1.0=페널티 없음)와 `repeat_last_n`(기본 64, 페널티를 적용할 직전 토큰 개수)을 Modelfile의 PARAMETER나 `/set parameter`로 설정한다. Ollama는 OpenAI식 presence/frequency_penalty를 네이티브로 완전히 노출하지 않아 서드파티 툴(Continue 등) 연동에서 설정 불가 이슈가 있었다.

주의할 실무 함정도 있다.
vLLM의 OpenAI 서버 경로(`vllm serve`)에서 요청에 `repetition_penalty`나 `presence_penalty`를 넣으면 CUDA scatter gather kernel index out of bounds 또는 device-side assert로 워커 프로세스가 죽는 크래시 버그가 여러 건 보고되었다(vLLM #28307, Qwen3-VL #1812).
같은 파라미터가 `AsyncLLMEngine` 직접 호출로는 정상 동작해 서버 구현부에 국한된 버그로 추정된다.
즉 설정값을 논하기 전에 그 파라미터를 켜면 서버가 크래시하는 함정을 먼저 확인해야 한다.

각 엔진의 페널티 파라미터 명칭을 정리하면 다음과 같다.

| 엔진 | 등장 여부 페널티 | 곱셈형 반복 페널티 | 곱셈형 기본값 |
|------|------|------|------|
| vLLM | presence_penalty | repetition_penalty | 1.0 |
| SGLang | presence_penalty | repetition_penalty | 1.0 |
| Ollama | 미노출 | repeat_penalty | 1.1 |

## 실전 권장 설정 예시

아래는 리서치에서 수집된 실사용·공식 프리셋을 상황별로 정리한 예시다.
공통 원칙은 greedy decoding을 쓰지 않고, 페널티는 기본을 끈 상태에서 반복이 있을 때만 `presence_penalty`를 켜는 것이다.

```python
# 0) Qwen3.6 thinking 모드 (일반) — thinking temperature가 1.0으로 상향됨
sampling = dict(
    temperature=1.0, top_p=0.95, top_k=20, min_p=0.0,
    repetition_penalty=1.0,   # 곱셈형은 off, 반복 시 presence_penalty를 0~2에서 상향
)

# 1) Qwen3 thinking 모드 (일반)
sampling = dict(
    temperature=0.6, top_p=0.95, top_k=20, min_p=0.0,
    # 반복 발생 시에만 presence_penalty를 0~2 범위에서 상향
)

# 2) Qwen3 thinking 모드 (정밀 코딩) — 페널티 완전 off
sampling = dict(
    temperature=0.6, top_p=0.95, top_k=20, min_p=0.0,
    presence_penalty=0.0, repetition_penalty=1.0,
)

# 3) Qwen3 instruct / non-thinking
sampling = dict(
    temperature=0.7, top_p=0.80, top_k=20, min_p=0.0,
    repetition_penalty=1.0,   # 곱셈형은 끄고
    # 심각한 무한 반복 시 presence_penalty=1.5 (vLLM 공식 예시)
)

# 4) Qwen3-Coder
sampling = dict(
    temperature=0.7, top_p=0.8, top_k=20,
    repetition_penalty=1.05,
)
```

vLLM에서 non-thinking 반복을 억제하는 구체적 예시는 다음과 같다.

```python
from vllm import SamplingParams

sampling_params = SamplingParams(
    temperature=0.7,
    top_p=0.8,
    top_k=20,
    presence_penalty=1.5,   # 0~2 범위에서 반복 완화 (높으면 language mixing 주의)
    max_tokens=8192,
)
```

OpenAI 호환 엔드포인트로 Qwen을 서빙할 때는 `top_k`가 OpenAI 표준 밖이므로 `extra_body`로 전달해야 한다.

```python
resp = client.chat.completions.create(
    model="Qwen3-8B",
    messages=[...],
    temperature=0.7, top_p=0.8,
    presence_penalty=1.5,   # 기본 0.0, 범위 -2.0 ~ 2.0
    extra_body={"top_k": 20},
)
```

한국어 등 다국어 출력을 다룰 때는 `presence_penalty`를 무작정 올리기보다 1.0 근처에서 시작하고, 표·코드·OCR 전사 작업이라면 페널티를 아예 끄는 편이 안전하다.

## 결론

Qwen의 반복 제어 철학은 명확하다.
곱셈형 `repetition_penalty`는 민감하므로 1.0(off)으로 두고, 필요할 때 감산형 `presence_penalty`를 0~2 범위에서 부드럽게 조절한다.
공식 권장의 뼈대는 greedy decoding 금지와 모드별 샘플링 프리셋(thinking 0.6/0.95/20, non-thinking 0.7/0.8/20)이다.
최신 세대인 Qwen3.6에서는 thinking 권장 temperature가 1.0으로 상향됐지만(정밀 코딩은 여전히 0.6), 곱셈형 off + 감산형 조절이라는 반복 제어 기조 자체는 그대로다.

다만 페널티 상향은 만능이 아니다.
높은 `presence_penalty`는 language mixing과 성능 저하를 부르고, 표·코드·OCR처럼 반복이 정상인 출력을 망가뜨리며, 근본 원인이 long-context·서빙 스택에 있는 경우엔 효과가 없다.
따라서 기본값을 명시적으로 전달하되, 자신의 태스크(창작 vs 전사, 단일 언어 vs 다국어)에 맞춰 페널티를 신중히 조율하는 것이 실전의 핵심이다.

## Reference

- [Qwen3 Discussion #1744 — presence_penalty 동작 원리·실사용 수렴값](https://github.com/QwenLM/Qwen3/discussions/1744/)
- [Qwen3-VL Issue #1825 — presence_penalty=1.5 기본값 논쟁](https://github.com/QwenLM/Qwen3-VL/issues/1825/)
- [Qwen3-VL Issue #1611 — 무한 반복 버그와 표/OCR 전사 손상 비판](https://github.com/QwenLM/Qwen3-VL/issues/1611/)
- [Qwen3-VL Issue #1812 — vllm serve에서 penalty 사용 시 CUDA assert 크래시](https://github.com/QwenLM/Qwen3-VL/issues/1812/)
- [vLLM Issue #28307 — repetition_penalty 사용 시 엔진 실패](https://github.com/vllm-project/vllm/issues/28307/)
- [Qwen2.5 Issue #513 — Qwen2 + vLLM 무한 루프](https://github.com/QwenLM/Qwen2.5/issues/513/)
- [Qwen3-30B-A3B Discussion #23 — 롤플레이 반복과 YaRN long-context](https://huggingface.co/Qwen/Qwen3-30B-A3B/discussions/23/)
- [Hugging Face — Qwen3-8B 모델 카드(Best Practices)](https://huggingface.co/Qwen/Qwen3-8B/)
- [Hugging Face — Qwen3-32B 모델 카드](https://huggingface.co/Qwen/Qwen3-32B/)
- [Hugging Face — Qwen3.6-35B-A3B 모델 카드(Best Practices)](https://huggingface.co/Qwen/Qwen3.6-35B-A3B/)
- [Hugging Face — Qwen3.6-27B 모델 카드](https://huggingface.co/Qwen/Qwen3.6-27B/)
- [GitHub — QwenLM/Qwen3.6](https://github.com/QwenLM/Qwen3.6/)
- [Hugging Face — Qwen2.5-7B-Instruct 모델 카드](https://huggingface.co/Qwen/Qwen2.5-7B-Instruct/)
- [Qwen 공식 문서 — Quickstart](https://qwen.readthedocs.io/en/latest/getting_started/quickstart.html/)
- [Qwen 공식 문서 — vLLM 배포](https://qwen.readthedocs.io/en/latest/deployment/vllm.html/)
- [HuggingFace Transformers — Text Generation(GenerationConfig)](https://huggingface.co/docs/transformers/main_classes/text_generation/)
- [HuggingFace Transformers PR #2303 — repetition penalty 음수 logit 버그 수정](https://github.com/huggingface/transformers/pull/2303/)
- [HuggingFace Transformers Issue #2302 — 음수 logit repetition penalty 오작동](https://github.com/huggingface/transformers/issues/2302/)
- [vLLM 문서 — SamplingParams](https://docs.vllm.ai/en/latest/api/vllm/sampling_params.html/)
- [SGLang — 페널티·제약 파라미터](https://deepwiki.com/sgl-project/sglang/17.4-penalties-and-constraints/)
- [Ollama 파라미터 정리 — repeat_penalty / repeat_last_n](https://technovangelist.com/notes/ollama-parameters/)
- [Ollama Modelfile로 Qwen3 코드 튜닝](https://akitaonrails.com/en/2025/04/29/dissecting-an-ollama-modelfile-tuning-qwen3-for-code/)
- [Continue Issue #3053 — Ollama provider penalty 설정 불가](https://github.com/continuedev/continue/issues/3053/)
- [Stop the LLM from Rambling — 세 페널티 차이 해설](https://dev.to/superorange0707/stop-the-llm-from-rambling-using-penalties-to-control-repetition-5h8/)
- [Repetition Penalties in Language Model Generation](https://mbrenndoerfer.com/writing/repetition-penalties-language-model-generation/)
- [Muxup — 벤더 권장 LLM 파라미터 퀵 레퍼런스](https://muxup.com/2025q2/recommended-llm-parameter-quick-reference/)
- [Unsloth — Qwen3-2507 실행·파인튜닝 가이드](https://unsloth.ai/docs/models/tutorials/qwen3-how-to-run-and-fine-tune/qwen3-2507/)
- [vLLM 포럼 — Qwen2.5-VL 반복 루프와 penalty 상향의 부작용](https://discuss.vllm.ai/t/speeding-up-vllm-inference-for-qwen2-5-vl/615/10/)
- [CTRL 논문(Keskar et al. 2019) arXiv:1909.05858](https://arxiv.org/abs/1909.05858/)
- [LZ Penalty 논문 arXiv:2504.20131](https://arxiv.org/html/2504.20131v2/)
