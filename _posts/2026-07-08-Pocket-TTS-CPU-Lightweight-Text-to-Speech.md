---
layout: post
title: "Pocket TTS: 주머니 속 CPU에서 돌아가는 경량 음성 합성"
author: 'Juho'
date: 2026-07-08 00:00:00 +0900
categories: [AI]
tags: [AI, Python]
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
2. [핵심 특징](#핵심-특징)
   - [성능](#성능)
   - [언어 지원](#언어-지원)
3. [사용 방법](#사용-방법)
   - [CLI](#cli)
   - [Python 라이브러리](#python-라이브러리)
4. [한계와 생태계](#한계와-생태계)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Kyutai Labs의 Pocket TTS는 GPU나 웹 API 없이 CPU만으로 동작하는 경량 텍스트음성변환(TTS) 애플리케이션이다.
"A TTS that fits in your CPU (and pocket)"라는 슬로건처럼, 주머니에 들어갈 만큼 작은 모델로 빠르게 음성을 생성하는 것이 목표다.
1억 개(100M) 파라미터 규모의 소형 모델로 설계되어, 별도의 고성능 하드웨어 없이도 실시간 스트리밍 음성 합성을 지원한다.

## 핵심 특징

Pocket TTS는 소형 모델의 이점을 극대화하기 위해 배치 크기 1을 전제로 설계되었다.
PyTorch 2.5 이상 위에서 동작하며, GPU 빌드가 필요하지 않다.

### 성능

CPU 실행에 최적화되어 GPU 없이도 낮은 지연시간을 달성한다.

| 항목 | 값 |
|------|------|
| 모델 크기 | 1억 파라미터 (100M) |
| 첫 청크 지연시간 | 약 200ms |
| 처리 속도 | 실시간 대비 6배 (MacBook Air M4, 2코어) |
| 스트리밍 | 청크 단위 실시간 생성 |
| Python 지원 | 3.10 ~ 3.14 |

임의로 긴 텍스트도 청크 단위 스트리밍으로 처리하므로 입력 길이에 제약이 없다.

### 언어 지원

영어, 프랑스어, 독일어, 포르투갈어, 이탈리아어, 스페인어를 지원한다.
비영어 언어의 경우 더 높은 품질을 위한 24층 대형 변형 모델도 제공된다.
약 20개의 사전학습된 음성(alba, giovanni, lola, anna, charles 등)이 Hugging Face에서 호스팅되며, 각 음성은 언어 및 라이선스 정보와 함께 제공된다.

## 사용 방법

설치는 pip 또는 uv로 간단하게 진행할 수 있다.

```bash
pip install pocket-tts
# 또는 uv 사용
uvx pocket-tts generate
```

### CLI

CLI는 generate, serve, export-voice 세 가지 명령을 제공한다.

```bash
# 사전정의 음성으로 생성
pocket-tts generate --voice alba --text "Hello world"

# 메모리에 모델을 유지하는 HTTP 서버 실행
pocket-tts serve
# http://localhost:8000 접속
```

serve 명령은 모델을 메모리에 상주시켜 다중 요청을 빠르게 처리한다.
export-voice 명령으로 .wav 파일을 safetensors 형식의 음성 임베딩으로 변환하면, 이후 계산 없이 빠르게 로딩할 수 있다.

### Python 라이브러리

`pocket_tts` 패키지를 직접 임포트해 코드에서 음성을 생성할 수 있다.

```python
from pocket_tts import TTSModel

model = TTSModel.load_model()
voice_state = model.get_state_for_audio_prompt("alba")
audio = model.generate_audio(voice_state, "Hello world")
```

사전정의 음성 외에도 커스텀 .wav 파일을 넣어 음성 복제가 가능하다.

## 한계와 생태계

명확한 한계도 존재한다.
텍스트 내 명시적 휴지(silence) 생성은 지원하지 않는다.
또한 배치 크기 1의 소형 모델 특성상 GPU에서 실행해도 CPU 대비 속도 향상이 없다.

라이선스는 MIT이며, 명확한 동의 없는 음성 복제, 거짓 정보 생성, 개인정보 침해 콘텐츠 생성 등은 사용 제약으로 금지된다.
공식 Python 구현 외에도 MLX(Apple Silicon), Rust(Candle), C++, C#/.NET, Deno 등 다양한 언어 포팅과 WebAssembly/JavaScript 커뮤니티 구현체가 존재한다.
Discord 봇, ComfyUI 플러그인, Home Assistant 통합, macOS 네이티브 앱 등으로 활발한 생태계가 형성되고 있다.

## 결론

Pocket TTS는 "작은 모델로 충분히 빠르고 실용적인 음성 합성이 가능하다"는 것을 보여주는 프로젝트다.
GPU와 클라우드 API에 의존하지 않고 로컬 CPU에서 200ms 지연으로 실시간 음성을 만들어내는 접근은, 온디바이스 음성 애플리케이션의 진입 장벽을 크게 낮춘다.
경량 TTS를 로컬 환경에 통합하려는 개발자에게 실질적인 선택지가 될 만하다.

## Reference

- [kyutai-labs/pocket-tts](https://github.com/kyutai-labs/pocket-tts/)
