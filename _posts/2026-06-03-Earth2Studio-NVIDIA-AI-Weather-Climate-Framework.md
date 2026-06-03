---
layout: post
title: "Earth2Studio: NVIDIA의 AI 기반 날씨·기후 모델링 통합 프레임워크"
author: 'Juho'
date: 2026-06-03 00:00:00 +0900
categories: [AI]
tags: [AI, Python, GPU]
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
2. [프레임워크 구조](#프레임워크-구조)
   - [모델 카탈로그](#모델-카탈로그)
   - [데이터 소스와 IO 백엔드](#데이터-소스와-io-백엔드)
3. [사용 예시](#사용-예시)
   - [FourCastNet3 실행](#fourcastnet3-실행)
   - [ECMWF AIFS와 GraphCast](#ecmwf-aifs와-graphcast)
4. [Perturbation과 통계 평가](#perturbation과-통계-평가)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Earth2Studio는 NVIDIA가 공개한 오픈소스 Python 프레임워크다.
AI 기반 날씨와 기후 워크플로우의 구축, 배포, 탐색을 빠르게 시작할 수 있도록 설계되었다.
"누구나 AI 기반 날씨와 기후 과학을 만들고 연구하고 탐색할 수 있게 한다"는 미션을 내걸고 있다.

다양한 AI 모델, 데이터 소스, 처리 컴포넌트를 통일된 인터페이스로 제공하여 사전 학습된 모델을 즉시 활용할 수 있다.
저장소는 Apache 2.0 라이선스이며 코드의 99.1%가 Python으로 작성되어 있다.
의존성은 `uv` 패키지 매니저로 관리하며 mypy 타입 체크, Black 포맷터, Ruff 린터가 적용되어 있다.

## 프레임워크 구조

Earth2Studio는 모델, 데이터 소스, IO 백엔드, Perturbation, 통계라는 다섯 가지 모듈을 컴포저블하게 결합한다.
사용자는 각 구성 요소를 자유롭게 교체할 수 있으며, 이는 다중 데이터 소스와 다중 모델을 결합한 복잡한 파이프라인 개발을 가능하게 한다.

### 모델 카탈로그

Earth2Studio는 사전 학습된 날씨와 기후 모델의 가장 큰 컬렉션을 포함한다.
크게 시간 적분 예측을 수행하는 Prognostic 모델과 즉시 변환을 수행하는 Diagnostic 모델로 나뉜다.

Prognostic 모델 목록은 다음과 같다.

| 모델 | 해상도 | 타임스텝 | 아키텍처 |
|------|--------|----------|----------|
| GraphCast Small | 1.0° | 6시간 | Graph Neural Network |
| GraphCast Operational | 0.25° | 6시간 | Graph Neural Network |
| Pangu-Weather | 0.25° | 3, 6, 24시간 | Transformer |
| Aurora | 0.25° | 6시간 | Transformer |
| FuXi | 0.25° | - | Transformer |
| AIFS (ECMWF) | - | - | Transformer + Ensemble |
| FourCastNet (FCN3) | - | - | AFNO |
| StormCast | 3km 지역 (US) | 1시간 | Diffusion |
| SFNO, DLESyM | - | - | Spectral / 기타 |

Diagnostic 모델은 즉시 변환 형태로 작동한다.
PrecipitationAFNO는 강수 예측을, SolarRadiationAFNO는 표면 태양 복사량을, WindgustAFNO는 최대 돌풍을 추정한다.
TCTrackerVitart는 열대성 사이클론을 추적하며, CBottle 계열은 기후 데이터 생성과 초해상도를, CorrDiff는 세밀한 다운스케일링을 담당한다.

### 데이터 소스와 IO 백엔드

데이터 소스 API는 다양한 운영 모델과 재분석, 관측, 위성 데이터를 통일된 인터페이스로 노출한다.
운영 모델로는 GFS, HRRR, IFS가 지원되며, 재분석으로는 ERA5(ARCO), CDS, NCAR ERA5가 포함된다.
관측 자료로는 ISD 기상 관측소, MRMS, GHCN-Daily가, 위성 자료로는 Himawari AHI가 지원된다.
WeatherBench2와 같이 Zarr 포맷으로 제공되는 클라우드 최적화 데이터 셋도 사용할 수 있다.
모든 소스는 Xarray를 통해 공유 어휘와 좌표계로 정규화된다.

IO 백엔드는 파이프라인 출력을 위한 다양한 저장 옵션을 제공한다.

| 백엔드 | 특징 |
|--------|------|
| ZarrBackend | 압축과 청크 지원, 로컬 또는 인메모리 저장 |
| AsyncZarrBackend | 원격 저장을 위한 병렬 IO |
| NetCDF4Backend | CF 호환 과학 데이터 포맷 |
| XarrayBackend | 분석 준비 완료된 인메모리 데이터셋 |
| KVBackend | 빠른 임시 키값 저장 |

## 사용 예시

Earth2Studio의 모든 모델은 동일한 패턴으로 호출된다.
모델 클래스를 로드하고, 데이터 소스를 지정하고, IO 백엔드를 연결한 뒤 `run` 함수에 시간 범위와 스텝 수를 넘기면 된다.

### FourCastNet3 실행

FourCastNet3는 AFNO 기반의 글로벌 예측 모델이다.
GFS 운영 데이터를 입력으로 받아 10 스텝의 결정론적 예측을 Zarr에 저장하는 예시는 다음과 같다.

```python
from earth2studio.models.px import FCN3
from earth2studio.data import GFS
from earth2studio.io import ZarrBackend
from earth2studio.run import deterministic as run

model = FCN3.load_model(FCN3.load_default_package())
data = GFS()
io = ZarrBackend("outputs/fcn3_forecast.zarr")
run(["2025-01-01T00:00:00"], 10, model, data, io)
```

### ECMWF AIFS와 GraphCast

ECMWF AIFS는 Transformer 기반 글로벌 모델로, IFS 데이터를 입력으로 사용한다.

```python
from earth2studio.models.px import AIFS
from earth2studio.data import IFS
from earth2studio.io import ZarrBackend
from earth2studio.run import deterministic as run

model = AIFS.load_model(AIFS.load_default_package())
data = IFS()
io = ZarrBackend("outputs/aifs_forecast.zarr")
run(["2025-01-01T00:00:00"], 10, model, data, io)
```

DeepMind의 GraphCast Operational 모델은 그래프 신경망으로 0.25° 해상도에서 6시간 단위로 예측한다.

```python
from earth2studio.models.px import GraphCastOperational
from earth2studio.data import GFS
from earth2studio.io import ZarrBackend
from earth2studio.run import deterministic as run

package = GraphCastOperational.load_default_package()
model = GraphCastOperational.load_model(package)
data = GFS()
io = ZarrBackend("outputs/graphcast_operational_forecast.zarr")
run(["2025-01-01T00:00:00"], 4, model, data, io)
```

세 예시 모두 동일한 패턴을 따른다.
이로써 사용자가 모델을 비교 평가하거나 앙상블을 구성할 때 코드 수정 부담이 최소화된다.

## Perturbation과 통계 평가

Perturbation 모듈은 불확실성 정량화와 앙상블 예측을 위한 노이즈 주입 방법을 제공한다.
다양한 공간 상관 구조를 갖는 가우시안 노이즈, 구면(Spherical) 및 Matérn 상관 구조, 동적 Bred Vector 방법, AR(1) 시간 상관 등이 포함된다.

통계와 메트릭 모듈은 평가 연산을 표준화한다.
RMSE, ACC(패턴 상관), CRPS, 랭크 히스토그램, Spread-Skill 비율과 같은 지표를 제공하며, 시공간 검증 기능까지 포함한다.
이는 앙상블 신뢰도 분석과 운영 검증을 한 프레임워크 안에서 수행할 수 있게 해 준다.

버전 0.14.0 이후로는 기본적으로 CUDA 13을 타겟팅하며, Himawari-8/9 위성 데이터 통합, GHCN-Daily 관측, Orbit-2 강수 다운스케일링 모델, NNJA와 GDAS 같은 전통적 관측 소스가 추가되었다.

## 결론

Earth2Studio는 사전 학습된 다수의 AI 날씨·기후 모델을 단일 API로 통합해 진입 장벽을 크게 낮춘다.
GraphCast, Pangu-Weather, Aurora, FuXi, AIFS, FourCastNet, StormCast 등 서로 다른 아키텍처를 같은 패턴으로 호출할 수 있으며, GFS·ERA5·HRRR 등 다양한 데이터 소스를 자유롭게 결합할 수 있다.
Perturbation과 통계 모듈을 통해 단일 결정론 예측에서 앙상블 확률 예측까지 단일 파이프라인으로 확장 가능하다.
저장소는 952개 별과 210개 포크를 보유하며 552개 커밋으로 활발히 유지되고 있어, AI 기반 지구 시스템 모델링 연구의 사실상 표준 도구로 자리잡고 있다.

## Reference

- [NVIDIA/earth2studio GitHub Repository](https://github.com/NVIDIA/earth2studio)
