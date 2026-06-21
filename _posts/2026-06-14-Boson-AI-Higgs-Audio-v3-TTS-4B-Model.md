---
layout: post
title: "Boson AI Higgs Audio v3 TTS 4B 모델"
author: 'Juho'
date: 2026-06-14 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, GPU]
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
2. [아키텍처](#아키텍처)
   - [모델 구조](#모델-구조)
   - [오디오 인코딩](#오디오-인코딩)
3. [주요 기능](#주요-기능)
   - [인라인 제어 토큰](#인라인-제어-토큰)
   - [다국어 지원](#다국어-지원)
4. [벤치마크 결과](#벤치마크-결과)
   - [다국어 음성 복제 WER/CER](#다국어-음성-복제-wercer)
   - [처리량 성능](#처리량-성능)
5. [사용법](#사용법)
   - [SGLang 서버 기반 음성 합성](#sglang-서버-기반-음성-합성)
   - [음성 복제](#음성-복제)
   - [인라인 제어 예시](#인라인-제어-예시)
   - [스트리밍](#스트리밍)
6. [라이선스 및 제한사항](#라이선스-및-제한사항)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

[Higgs Audio v3 TTS](https://huggingface.co/bosonai/higgs-audio-v3-tts-4b){:target="_blank"}는 Boson AI가 공개한 40억(4B) 파라미터 규모의 텍스트 음성 변환(TTS) 모델이다.
대화형 음성 애플리케이션을 위해 설계되었으며, 100개 이상의 언어에서 표현력 있는 대화 음성을 생성하는 것을 목표로 한다.
제로샷(Zero-shot) 음성 복제와 인라인 제어 기능을 지원하며, 이전 버전인 Higgs Audio v2 대비 전반적인 성능이 크게 향상되었다.

## 아키텍처

### 모델 구조

Higgs Audio v3는 약 4B 파라미터 규모의 자기회귀(Autoregressive) 디코더 아키텍처를 기반으로 한다.
주요 구조 사양은 다음과 같다.

| 항목 | 사양 |
|------|------|
| 레이어 수 | 36개 |
| 히든 유닛 수 | 2560 |
| 어텐션 방식 | GQA 32/8 |
| 컨텍스트 길이 | 8,192 토큰 |
| 샘플 레이트 | 24 kHz |
| 학습 시퀀스 길이 | 8,192 토큰 |
| 텐서 타입 | BF16 |
| 모델 포맷 | Safetensors |

### 오디오 인코딩

오디오 인코딩에는 Higgs Tokenizer를 사용하며, 딜레이 패턴(Delay Pattern)을 적용한 8개의 코드북(Codebook)으로 구성된다.
프레임 레이트는 25fps(초당 25프레임)이며, 프레임당 40ms에 해당한다.
코드북별 어휘 크기는 1026이다.

## 주요 기능

### 인라인 제어 토큰

Higgs Audio v3는 `<|category:value|>` 형식의 인라인 태그를 텍스트 내에 삽입하여 음성의 다양한 특성을 세밀하게 제어할 수 있다.
지원하는 제어 카테고리는 다음과 같다.

| 카테고리 | 옵션 |
|----------|------|
| 감정(Emotion) | elation, anger, sadness, confusion 등 21가지 |
| 스타일(Style) | singing, shouting, whispering |
| 운율(Prosody) | speed 4단계, pitch 2단계, expressive 2단계 |
| 음향 효과(Sound Effects) | cough, laughter, crying, screaming, burping, humming, sigh, sniff, sneeze |
| 일시 정지(Pause) | pause(약 400~700ms), long_pause(약 700~1500ms) |

### 다국어 지원

총 102개 언어를 지원하며, 품질 수준에 따라 두 단계로 분류된다.
WER/CER 5 미만인 85개 언어는 프로덕션 품질 수준이며, WER/CER 5~10 구간의 17개 언어는 사용 가능한 수준이지만 상대적으로 정확도가 낮다.
영어, 스페인어, 프랑스어, 중국어, 일본어, 독일어를 포함한 주요 언어가 모두 포함된다.

## 벤치마크 결과

### 다국어 음성 복제 WER/CER

Higgs Audio v3는 공개 벤치마크에서 이전 버전 대비 큰 폭의 개선을 보였다.
수치가 낮을수록 더 좋은 성능을 의미한다.

| 벤치마크 | v3 점수 | v2 점수 |
|----------|---------|---------|
| BABEL | 1.11 | 2.10 |
| CV3 | 4.41 | 21.19 |
| MiniMax-Multilingual | 2.74 | 49.86 |

CV3 및 MiniMax-Multilingual 벤치마크에서 v2 대비 약 5배에서 18배 가까운 오류율 감소를 달성했다.

Emergent TTS 항목에서는 경쟁 모델 대비 인간 평가 승률이 전체 53.65%이며, 특히 감정(53.75%)과 준언어적(Paralinguistics) 표현(68.57%)에서 강점을 보였다.

### 처리량 성능

단일 H100 GPU에서 Seed-TTS 영어 벤치마크(N=1088)를 기준으로 측정한 처리량 결과다.

| 동시 요청 수 | 처리량(req/s) | 평균 지연 | RTF | 오디오(초/초) |
|-------------|-------------|----------|-----|-------------|
| 1 | 1.62 | 617 ms | 0.147 | 6.89 |
| 16 | 14.74 | 1079 ms | 0.262 | 61.84 |

RTF(Real-Time Factor)가 1 미만이면 실시간보다 빠른 생성을 의미한다.
동시 요청 1개 기준 RTF 0.147은 실시간 대비 약 6.8배 빠른 속도를 나타낸다.

## 사용법

### SGLang 서버 기반 음성 합성

SGLang-Omni를 이용한 프로덕션 서빙 환경에서 기본 음성 합성을 수행하는 예시다.

```bash
curl -X POST http://localhost:8000/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{"input": "Hello, how are you?"}' \
  --output output.wav
```

### 음성 복제

참조 오디오 파일을 제공하여 특정 화자의 목소리를 복제하는 예시다.

```python
resp = requests.post(
    "http://localhost:8000/v1/audio/speech",
    json={
        "input": "Have a nice day.",
        "references": [{
            "audio_path": "ref.wav",
            "text": "Reference transcript here.",
        }],
        "temperature": 0.8,
    },
)
```

### 인라인 제어 예시

인라인 태그를 사용하여 감정과 음향 효과를 제어하는 예시다.

```bash
curl -X POST http://localhost:8000/v1/audio/speech \
  -d '{"input": "<|emotion:amusement|><|prosody:expressive_high|>That was hilarious. <|sfx:laughter|>Hehe"}'
```

### 스트리밍

`"stream": true` 옵션을 설정하면 Server-Sent Events(SSE) 방식으로 base64 인코딩된 WAV 청크를 수신한다.
첫 번째 오디오 청크까지의 지연(TTFA)이 1초 미만으로 실시간에 가까운 스트리밍이 가능하다.

Hugging Face Transformers 라이브러리를 통해서도 사용 가능하며, `pipeline("text-to-speech")` 인터페이스를 지원한다.
Boson AI API 및 Hugging Face Inference Endpoints를 통한 제로 운영(Zero-ops) 배포 옵션도 제공된다.

## 라이선스 및 제한사항

Higgs Audio v3는 "Boson Higgs Audio v3 Research and Non-Commercial License"에 따라 배포된다.
연구 및 비상업적 목적의 사용만 허용되며, 프로덕션 API 운영이나 수익 창출 목적의 애플리케이션에는 별도 상업 라이선스가 필요하다.
동의 없는 음성 복제, 개인 사칭, 사기, 선거 관련 기만 행위는 명시적으로 금지된다.

## 결론

Higgs Audio v3 TTS 4B는 4B 파라미터 규모의 자기회귀 모델로, 102개 언어 지원과 세밀한 인라인 감정·운율·음향 효과 제어 기능을 갖춘 텍스트 음성 변환 모델이다.
이전 버전 대비 다국어 음성 복제 품질이 크게 향상되었으며, 단일 H100 GPU에서 실시간 대비 최대 6.8배 빠른 생성 속도를 달성했다.
비상업적 연구 목적으로 공개되었으며, SGLang-Omni 기반 서빙, Transformers 라이브러리, Boson AI API 등 다양한 배포 옵션을 지원한다.

## Reference

- [Higgs Audio v3 TTS 4B - HuggingFace](https://huggingface.co/bosonai/higgs-audio-v3-tts-4b/)
