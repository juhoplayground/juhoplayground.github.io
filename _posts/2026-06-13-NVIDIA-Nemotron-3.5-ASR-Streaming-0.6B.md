---
layout: post
title: "NVIDIA Nemotron 3.5 ASR Streaming 0.6B - 다국어 스트리밍 음성 인식 모델"
author: 'Juho'
date: 2026-06-13 00:00:00 +0900
categories: [AI]
tags: [AI, Benchmark, GPU]
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
   - [언어 ID 프롬프트 컨디셔닝](#언어-id-프롬프트-컨디셔닝)
3. [지원 언어](#지원-언어)
4. [학습 데이터](#학습-데이터)
5. [벤치마크 결과](#벤치마크-결과)
   - [FLEURS WER 결과](#fleurs-wer-결과)
   - [청크 크기별 성능](#청크-크기별-성능)
6. [사용법](#사용법)
   - [설치](#설치)
   - [기본 추론](#기본-추론)
   - [스트리밍 추론](#스트리밍-추론)
7. [한계와 주의사항](#한계와-주의사항)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

[NVIDIA Nemotron 3.5 ASR Streaming 0.6B](https://huggingface.co/nvidia/nemotron-3.5-asr-streaming-0.6b){:target="_blank"}은 NVIDIA가 2026년 6월 4일 공개한 다국어 스트리밍 자동 음성 인식(ASR) 모델이다.
40개 언어-로케일에 걸쳐 고품질 다국어 전사를 제공하도록 설계되었으며, 저지연 음성 에이전트 애플리케이션과 프로덕션 배포를 위한 캐시 인식 스트리밍 아키텍처를 특징으로 한다.
모델 파라미터는 6억 개(0.6B)이며, 라이선스는 OpenMDW-1.1이다.

주요 특징은 다음과 같다.

- 40개 언어-로케일 지원
- 네이티브 스트리밍 아키텍처(80ms, 160ms, 320ms, 560ms, 1120ms 청크 크기 지원)
- 내장 구두점 및 대소문자 복원
- `target_lang=auto` 파라미터를 통한 자동 언어 감지
- 혼합 언어 트래픽 처리를 위한 출력 언어 태깅
- Parakeet RNNT 1.1B 대비 80ms 지연에서 약 17배 높은 동시 스트림 처리량

## 아키텍처

### 모델 구조

Nemotron 3.5 ASR Streaming 0.6B는 FastConformer-CacheAware-RNNT with Language-ID Prompt Conditioning 아키텍처를 사용한다.
구성 요소는 다음과 같다.

| 구성 요소 | 세부 내용 |
|-----------|-----------|
| 인코더 타입 | Cache-Aware FastConformer |
| 인코더 레이어 수 | 24개 |
| 디코더 타입 | RNNT (Recurrent Neural Network Transducer) |
| 총 파라미터 수 | 600M (0.6B) |
| 입력 형식 | WAV 오디오(모노, 1D) + 언어 ID(문자열) |
| 출력 형식 | 구두점/대소문자 포함 전사 텍스트 |

캐시 인식 스트리밍의 핵심은 중복 겹침 연산을 제거하는 데 있다.
인코더는 캐시된 컨텍스트를 활용하여 새로운 오디오 청크만 처리함으로써 지연 시간을 최소화한다.

### 언어 ID 프롬프트 컨디셔닝

언어 ID 프롬프트는 연결(concatenation)과 프로젝션(projection) 방식으로 음향 표현과 융합된다.
이 방식은 단일 모델로 다국어를 지원할 수 있게 하는 핵심 메커니즘이다.
`target_lang=auto`로 설정하면 모델이 자동으로 언어를 감지하고, 출력에 언어 태그를 포함시킬 수 있다.

## 지원 언어

지원 언어는 세 가지 수준으로 분류된다.

### 전사 준비 완료 (19개 로케일)

영어, 스페인어, 프랑스어, 이탈리아어, 포르투갈어, 네덜란드어, 독일어, 터키어, 러시아어, 아랍어, 힌디어, 일본어, 한국어, 베트남어, 우크라이나어, 폴란드어, 스웨덴어, 체코어, 노르웨이 보크몰

### 광범위 커버리지 (13개 로케일)

덴마크어, 불가리아어, 핀란드어, 크로아티아어, 슬로바키아어, 루마니아어, 헝가리어, 에스토니아어, 중국어(만다린) 등이 포함된다.

### 적응 필요 (8개 로케일)

도메인 특화 데이터로의 파인튜닝이 필요한 언어들이다.

## 학습 데이터

모델은 공개 및 독점 데이터셋을 포함해 10,000시간 이상의 데이터로 학습되었다.
사용된 주요 데이터셋은 다음과 같다.

| 데이터셋 | 설명 |
|----------|------|
| NVIDIA Riva 다국어 ASR 학습 세트 | NVIDIA 독점 다국어 데이터 |
| NVIDIA Granary | NVIDIA 내부 데이터 |
| Multilingual LibriSpeech (MLS) | 다국어 공개 음성 데이터 |
| Mozilla Common Voice | 오픈소스 크라우드소싱 음성 |
| FLEURS | 다국어 평가용 공개 데이터셋 |
| VoxPopuli | 유럽의회 다국어 음성 데이터 |
| Europarl-ASR | 유럽의회 ASR 데이터 |

레이블은 NVIDIA Canary, Parakeet, Whisper, FunASR로 구성된 앙상블 모델에서 합성하였다.
구두점과 대소문자 복원은 Qwen3-32B를 활용하였다.

## 벤치마크 결과

### FLEURS WER 결과

일본어, 한국어, 중국어(만다린)는 WER 대신 CER(Character Error Rate)로 측정한다.
아래는 언어 입력 모드에서 1.12초 청크 기준 주요 언어별 결과이다.

| 언어 | WER/CER (%) |
|------|-------------|
| 스페인어 | 4.11 |
| 이탈리아어 | 4.25 |
| 포르투갈어 | 5.48 |
| 한국어 (CER) | 7.12 |
| 영어 | 7.91 |
| 폴란드어 | 15.15 |
| 헝가리어 | 28.68 |

자동 감지 모드(`target_lang=auto`)에서 전사 준비 완료 언어 평균은 1.12초 청크 기준 9.21% WER이다.
광범위 커버리지 언어와 적응 필요 언어는 상대적으로 높은 오류율을 보인다.

### 청크 크기별 성능

청크 크기가 클수록 성능이 향상되지만, 80ms 최저 지연 설정에서도 경쟁력 있는 성능을 유지한다.
지원 청크 크기는 80ms, 160ms, 320ms, 560ms, 1120ms이다.
텍스트 정규화 방식에 따라 보고되는 오류율에 차이가 발생할 수 있다.

## 사용법

### 설치

NeMo 프레임워크와 필수 시스템 패키지를 먼저 설치해야 한다.
지원 런타임은 NeMo 26.06이며, NVIDIA Ampere, Hopper, Blackwell, Volta, Turing, Lovelace, Jetson GPU에서 동작한다.

```bash
apt-get update && apt-get install -y libsndfile1 ffmpeg
pip install Cython packaging
pip install git+https://github.com/NVIDIA/NeMo.git@main#egg=nemo_toolkit[asr]
```

### 기본 추론

NeMo의 `ASRModel.from_pretrained`을 사용해 모델을 불러오고 오디오 파일을 전사한다.

```python
import nemo.collections.asr as nemo_asr

asr_model = nemo_asr.models.ASRModel.from_pretrained(
    model_name="nvidia/nemotron-3.5-asr-streaming-0.6b"
)

transcriptions = asr_model.transcribe(["file.wav"])
```

### 스트리밍 추론

캐시 인식 스트리밍 추론은 NeMo에서 제공하는 전용 스크립트를 통해 실행할 수 있다.
`target_lang` 파라미터로 언어를 지정하거나 `auto`로 자동 감지를 사용한다.
`strip_lang_tags=true`로 설정하면 출력에서 언어 태그를 제거한다.

```bash
python examples/asr/asr_cache_aware_streaming/speech_to_text_cache_aware_streaming_infer.py \
    model_path=<model_path> \
    dataset_manifest=<dataset_manifest> \
    batch_size=<batch_size> \
    target_lang=<lang_id> \
    att_context_size="[56,13]" \
    strip_lang_tags=true \
    output_path=<output_folder>
```

## 한계와 주의사항

다음 사항에 유의해야 한다.

- 적응 필요(Adaptation-ready) 언어는 도메인 특화 파인튜닝 없이는 높은 오류율이 예상된다.
- 광범위 커버리지 언어(예: 헝가리어 28.68% WER)는 전사 준비 완료 언어에 비해 성능 차이가 크다.
- 텍스트 정규화 방식에 따라 공개된 벤치마크 수치와 실제 측정값이 다를 수 있다.
- 자동 언어 감지 모드는 언어 명시 모드 대비 평균 오류율이 다소 높을 수 있다.
- 모델은 WAV 형식(모노 채널, 1D)만 지원한다.

## 결론

NVIDIA Nemotron 3.5 ASR Streaming 0.6B는 600M 파라미터 규모에서 40개 언어-로케일을 지원하는 캐시 인식 스트리밍 ASR 모델이다.
FastConformer-CacheAware-RNNT 아키텍처와 언어 ID 프롬프트 컨디셔닝 방식으로 단일 모델에서 다국어 스트리밍 전사를 실현하였다.
Parakeet RNNT 1.1B 대비 80ms 지연에서 약 17배 높은 처리량을 제공하며, 전사 준비 완료 언어에서는 낮은 WER를 달성한다.
한국어를 포함한 주요 언어를 지원하며, 내장 구두점 및 대소문자 복원 기능을 갖추고 있어 실시간 음성 에이전트 및 프로덕션 환경에 적합하다.

## Reference

- [nvidia/nemotron-3.5-asr-streaming-0.6b - Hugging Face](https://huggingface.co/nvidia/nemotron-3.5-asr-streaming-0.6b/)
