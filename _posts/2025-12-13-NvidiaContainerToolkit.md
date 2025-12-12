---
layout: post
title: NVIDIA Driver 및 Container Toolkit 설치 가이드
author: 'Juho'
date: 2025-12-13 00:00:00 +0900
categories: [NVIDIA]
tags: [NVIDIA, Docker, GPU, Container, Driver]
pin: false
toc: true
---

## 목차
1. [NVIDIA Driver 설치](#nvidia-driver-설치)
2. [NVIDIA 드라이버와 CUDA 버전 호환성](#nvidia-드라이버와-cuda-버전-호환성)
3. [NVIDIA Container Toolkit 설치](#nvidia-container-toolkit-설치)
4. [Docker Compose에서 GPU 사용](#docker-compose에서-gpu-사용)
5. [다중 GPU 환경 관리](#다중-gpu-환경-관리)
6. [성능 모니터링](#성능-모니터링)
7. [문제 해결](#문제-해결)

## NVIDIA Driver 설치

Ubuntu 환경에서 NVIDIA 드라이버를 업데이트하는 방법입니다.

### STEP 1: 사용 가능한 드라이버 확인

다음 명령으로 시스템에 호환되는 드라이버 목록을 확인합니다.

```bash
ubuntu-drivers devices
```

출력 예시:
```
== /sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0 ==
modalias : pci:v000010DEd00001C03sv00001458sd0000372Fbc03sc00i00
vendor   : NVIDIA Corporation
model    : GP106 [GeForce GTX 1060 6GB]
driver   : nvidia-driver-550 - distro non-free recommended
driver   : nvidia-driver-535 - distro non-free
driver   : nvidia-driver-470 - distro non-free
driver   : xserver-xorg-video-nouveau - distro free builtin
```

`recommended` 표시가 있는 드라이버가 권장 버전입니다. (2025년 현재 일반적으로 535 또는 550 계열이 권장됩니다)

### STEP 2: 드라이버 설치

#### 자동 설치 (권장)

시스템에 가장 적합한 드라이버를 자동으로 설치합니다. 이 방법이 가장 안전합니다.

```bash
sudo ubuntu-drivers autoinstall
```

#### 수동 설치

특정 버전의 드라이버를 지정하여 설치할 수도 있습니다. STEP 1에서 확인한 `recommended` 버전을 사용하세요.

```bash
# 예시: 550 버전 설치 (실제로는 STEP 1에서 확인한 권장 버전 사용)
sudo apt install nvidia-driver-550
```

**주의**: 하드코딩된 버전 번호 대신 `ubuntu-drivers autoinstall`을 사용하거나, STEP 1에서 확인한 권장 버전을 설치하는 것을 권장합니다.

### STEP 3: 시스템 재시작

드라이버 설치를 완료하려면 시스템을 재시작해야 합니다.

```bash
sudo reboot
```

### STEP 4: 설치 확인

재부팅 후 다음 명령으로 드라이버가 정상적으로 로드되었는지 확인합니다.

```bash
nvidia-smi
```

정상적으로 설치되었다면 GPU 정보와 드라이버 버전이 출력됩니다.

---

## NVIDIA 드라이버와 CUDA 버전 호환성

### 호환성 이해하기

NVIDIA 드라이버와 CUDA 버전 간의 호환성은 GPU 컨테이너 환경에서 가장 중요한 개념 중 하나입니다.

**핵심 원칙:**
- **호스트 드라이버**는 컨테이너의 **CUDA 런타임 버전**을 지원할 수 있어야 합니다
- 드라이버 버전이 높으면 낮은 CUDA 버전을 지원하지만, 그 반대는 불가능합니다
- 컨테이너 내부의 CUDA 버전은 호스트 드라이버가 지원하는 범위 내에서 자유롭게 선택 가능합니다

### nvidia-smi로 확인하기

`nvidia-smi` 명령을 실행하면 우측 상단에 `CUDA Version`이 표시됩니다.

```bash
nvidia-smi
```

출력 예시:
```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 550.54.15    Driver Version: 550.54.15    CUDA Version: 12.4   |
|-------------------------------+----------------------+----------------------+
```

여기서 표시되는 `CUDA Version: 12.4`는 **호스트 드라이버가 지원하는 최대 CUDA 버전**을 의미합니다.
이는 컨테이너에서 **CUDA 12.4 이하의 모든 버전**을 사용할 수 있다는 뜻입니다.

### 주요 호환성 매핑

| 드라이버 버전 | 지원하는 최대 CUDA 버전 | 권장 사용 예시 |
|------------|-------------------|------------|
| 550.x | CUDA 12.4 | 최신 GPU 워크로드 |
| 535.x | CUDA 12.2 | 안정적인 프로덕션 환경 |
| 520.x | CUDA 11.8 | 레거시 CUDA 11 지원 |
| 470.x | CUDA 11.4 | 구형 시스템 |

### 실전 예시

#### 예시 1: 호스트 드라이버 550 사용 시

호스트에 드라이버 550이 설치되어 있다면 (CUDA 12.4 지원):

```bash
# ✅ 가능: CUDA 12.2 사용
docker run --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

# ✅ 가능: CUDA 11.8 사용 (하위 호환)
docker run --gpus all nvidia/cuda:11.8.0-base-ubuntu22.04 nvidia-smi

# ✅ 가능: CUDA 12.4 사용
docker run --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi

# ❌ 불가능: CUDA 12.5는 지원하지 않음
docker run --gpus all nvidia/cuda:12.5.0-base-ubuntu22.04 nvidia-smi
```

#### 예시 2: 호스트 드라이버 470 사용 시

호스트에 드라이버 470이 설치되어 있다면 (CUDA 11.4 지원):

```bash
# 가능: CUDA 11.0 사용
docker run --gpus all nvidia/cuda:11.0-base nvidia-smi

# 불가능: CUDA 12.x는 지원하지 않음
docker run --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
# 에러: CUDA driver version is insufficient for CUDA runtime version
```

### 호환성 확인 방법

1. **호스트 드라이버 버전 확인:**
   ```bash
   nvidia-smi | grep "Driver Version"
   ```

2. **지원 가능한 CUDA 버전 확인:**
   ```bash
   nvidia-smi | grep "CUDA Version"
   ```

3. **공식 호환성 표 확인:**
   - [NVIDIA CUDA Compatibility Guide](https://docs.nvidia.com/deploy/cuda-compatibility/)
   - [CUDA Toolkit Release Notes](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/)

### 권장 사항

1. **최신 드라이버 사용**: 가능하면 535 또는 550 계열의 최신 드라이버를 사용하세요. 하위 호환성이 뛰어나 대부분의 CUDA 버전을 지원합니다.

2. **컨테이너 이미지 선택**: 프로젝트에서 사용하는 프레임워크(PyTorch, TensorFlow 등)의 공식 이미지를 사용하세요. 이미 호환성이 검증된 CUDA 버전을 포함하고 있습니다.

3. **프로덕션 환경**: 안정성이 중요한 경우 드라이버 535 + CUDA 12.2 조합을 권장합니다.

### 자주 발생하는 오류

#### "CUDA driver version is insufficient"

```
CUDA driver version is insufficient for CUDA runtime version
```

**원인**: 호스트 드라이버가 컨테이너의 CUDA 버전을 지원하지 않습니다.

**해결 방법**:
- 호스트 드라이버를 업그레이드하거나
- 컨테이너에서 더 낮은 CUDA 버전을 사용하세요

```bash
# 현재 지원 가능한 CUDA 버전 확인
nvidia-smi

# 지원되는 범위 내의 이미지 사용
docker run --gpus all nvidia/cuda:11.8.0-base-ubuntu22.04 nvidia-smi
```

---

## NVIDIA Container Toolkit 설치

NVIDIA Container Toolkit은 Docker 컨테이너 환경에서 NVIDIA GPU를 사용할 수 있도록 해주는 도구입니다.  
GPU 디바이스와 CUDA 라이브러리를 컨테이너에 자동으로 마운트하여 Docker 내에서 GPU 기반 연산을 가능하게 합니다.  

### 주요 기능
- Docker 컨테이너 내에서 GPU 리소스 접근  
- CUDA 라이브러리 자동 마운트  
- GPU 기반 머신러닝/딥러닝 워크로드 지원  
- 여러 컨테이너 간 GPU 공유 관리  

### 사전 확인 사항

#### Ubuntu 버전 확인
```bash
lsb_release -a
```

#### Docker 설치 확인
```bash
docker --version
```

#### GPU 정보 확인
```bash
nvidia-smi
```

### APT를 통한 설치

#### STEP 1: 저장소 설정

먼저 배포판 정보를 확인하고 저장소를 추가합니다.

```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/$distribution/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

#### STEP 2: 패키지 저장소 업데이트

```bash
sudo apt-get update
```

#### STEP 3: NVIDIA Container Toolkit 설치

```bash
sudo apt-get install nvidia-container-toolkit
```

#### STEP 4: Docker 런타임 설정

NVIDIA Container Toolkit을 Docker 런타임으로 설정합니다.

```bash
sudo nvidia-ctk runtime configure --runtime=docker
```

#### STEP 5: Docker 데몬 재시작

```bash
sudo systemctl restart docker
```

#### STEP 6: 설치 확인

```bash
nvidia-ctk --version
```

또는 다음 명령으로 설치된 패키지를 확인할 수 있습니다.

```bash
dpkg -l | grep nvidia-container-toolkit
```

#### STEP 7: GPU 접근 테스트

```bash
docker run --rm -it --ipc=host --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
```

정상적으로 설치되었다면 컨테이너 내에서 `nvidia-smi` 명령이 실행되어 GPU 정보가 출력됩니다.

---

## Docker Compose에서 GPU 사용

실무에서는 Docker Compose를 사용하여 컨테이너를 관리하는 경우가 많습니다.   
Docker Compose에서 GPU를 사용하는 방법을 소개합니다.  

### 기본 설정

Docker Compose v2.3 이상에서는 `deploy` 섹션을 사용하여 GPU를 설정합니다.

```yaml
version: '3.8'

services:
  gpu-app:
    image: nvidia/cuda:12.2.0-base-ubuntu22.04
    command: nvidia-smi
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

### 모든 GPU 사용

모든 GPU를 컨테이너에 할당하는 설정입니다.

```yaml
services:
  my-service:
    image: my-gpu-app:latest
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

### 특정 개수의 GPU만 사용

특정 개수의 GPU만 할당할 수 있습니다.

```yaml
services:
  my-service:
    image: my-gpu-app:latest
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 2  # 2개의 GPU만 사용
              capabilities: [gpu]
```

### 특정 GPU 지정

특정 GPU를 인덱스로 지정하여 사용할 수 있습니다.

```yaml
services:
  my-service:
    image: my-gpu-app:latest
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['0', '2']  # GPU 0번과 2번만 사용
              capabilities: [gpu]
```

### 실전 예시: PyTorch 컨테이너

실제 머신러닝 워크로드에서 사용하는 예시입니다.

```yaml
version: '3.8'

services:
  pytorch-training:
    image: pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime
    container_name: pytorch-gpu
    volumes:
      - ./code:/workspace
      - ./data:/data
    working_dir: /workspace
    command: python train.py
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    environment:
      - CUDA_VISIBLE_DEVICES=0,1  # 추가 GPU 제어
    ipc: host  # 공유 메모리 설정 (대용량 데이터 로딩 시 필요)
```

### 실전 예시: Jupyter Notebook with GPU

GPU를 사용하는 Jupyter Notebook 환경 설정입니다.

```yaml
version: '3.8'

services:
  jupyter-gpu:
    image: jupyter/tensorflow-notebook:latest
    container_name: jupyter-gpu
    ports:
      - "8888:8888"
    volumes:
      - ./notebooks:/home/jovyan/work
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    environment:
      - JUPYTER_ENABLE_LAB=yes
      - GRANT_SUDO=yes
    command: start-notebook.sh --NotebookApp.token='' --NotebookApp.password=''
```

### Docker Compose 실행

GPU 설정이 포함된 Docker Compose를 실행할 때는 다음 명령을 사용합니다.

```bash
docker compose up -d
```

### GPU 사용 확인

컨테이너가 GPU를 정상적으로 사용하는지 확인합니다.

```bash
# 컨테이너 내에서 nvidia-smi 실행
docker compose exec gpu-app nvidia-smi

# 또는 로그 확인
docker compose logs gpu-app
```

### 주의사항

- Docker Compose v2.3 이상 버전을 사용해야 합니다
- `docker-compose` 명령 대신 `docker compose` (공백 포함) 명령을 사용하세요
- GPU 설정은 `deploy` 섹션 내에 있어야 하며, 루트 레벨에 직접 작성하면 안 됩니다
- `runtime: nvidia` 설정은 구버전 방식이므로 사용하지 않는 것이 좋습니다

---

## 다중 GPU 환경 관리

여러 개의 GPU를 사용하는 시스템에서 특정 GPU만 선택하거나, GPU 리소스를 효율적으로 관리하는 방법을 소개합니다.

### GPU 목록 확인

먼저 시스템에 설치된 GPU 목록을 확인합니다.

```bash
nvidia-smi -L
```

출력 예시:
```
GPU 0: NVIDIA GeForce RTX 3090 (UUID: GPU-xxx)
GPU 1: NVIDIA GeForce RTX 3090 (UUID: GPU-xxx)
GPU 2: NVIDIA GeForce RTX 3080 (UUID: GPU-xxx)
GPU 3: NVIDIA GeForce RTX 3080 (UUID: GPU-xxx)
```

### Docker에서 특정 GPU 선택

#### 방법 1: --gpus 옵션으로 선택

특정 GPU의 인덱스를 지정하여 사용할 수 있습니다.

```bash
# GPU 0번만 사용
docker run --gpus '"device=0"' nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

# GPU 1번과 2번만 사용
docker run --gpus '"device=1,2"' nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

# GPU 0번과 3번만 사용
docker run --gpus '"device=0,3"' nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
```

#### 방법 2: CUDA_VISIBLE_DEVICES 환경 변수

`CUDA_VISIBLE_DEVICES` 환경 변수를 사용하여 컨테이너에서 보이는 GPU를 제어할 수 있습니다.

```bash
# GPU 0번만 사용 (컨테이너 내에서는 GPU 0으로 인식됨)
docker run --gpus all -e CUDA_VISIBLE_DEVICES=0 \
  nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

# GPU 1,2번 사용 (컨테이너 내에서는 GPU 0,1로 인식됨)
docker run --gpus all -e CUDA_VISIBLE_DEVICES=1,2 \
  nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

# GPU를 숨기기 (CPU만 사용)
docker run --gpus all -e CUDA_VISIBLE_DEVICES=-1 \
  nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
```

**중요**: `CUDA_VISIBLE_DEVICES`를 사용하면 컨테이너 내에서 GPU 인덱스가 재매핑됩니다.
예를 들어, `CUDA_VISIBLE_DEVICES=2,3`으로 설정하면 컨테이너 내에서는 GPU 0, 1로 보입니다.

### Docker Compose에서 특정 GPU 선택

```yaml
version: '3.8'

services:
  training-gpu-0:
    image: pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['0']  # GPU 0번만 사용
              capabilities: [gpu]
    environment:
      - CUDA_VISIBLE_DEVICES=0

  training-gpu-1-2:
    image: pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['1', '2']  # GPU 1번과 2번 사용
              capabilities: [gpu]
    environment:
      - CUDA_VISIBLE_DEVICES=0,1  # 컨테이너 내에서 0,1로 재매핑
```

### GPU 개수 제한

특정 개수의 GPU만 할당할 수 있습니다.

```bash
# 2개의 GPU만 사용 (시스템이 자동으로 선택)
docker run --gpus 2 nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
```

### GPU 메모리 제한

GPU 메모리를 직접 제한하는 Docker 옵션은 없지만, CUDA 프로그램 레벨에서 제어할 수 있습니다.

#### 방법 1: TensorFlow에서 메모리 제한

```python
import tensorflow as tf

# GPU 메모리 성장 허용 (필요한 만큼만 할당)
gpus = tf.config.experimental.list_physical_devices('GPU')
if gpus:
    try:
        for gpu in gpus:
            tf.config.experimental.set_memory_growth(gpu, True)
    except RuntimeError as e:
        print(e)

# 또는 특정 메모리 크기 제한 (예: 4GB)
tf.config.experimental.set_virtual_device_configuration(
    gpus[0],
    [tf.config.experimental.VirtualDeviceConfiguration(memory_limit=4096)]
)
```

#### 방법 2: PyTorch에서 메모리 제한

```python
import torch

# GPU 메모리 캐시 비우기
torch.cuda.empty_cache()

# 메모리 할당 전략 설정
torch.cuda.set_per_process_memory_fraction(0.5, device=0)  # GPU 0의 50%만 사용
```

#### 방법 3: 환경 변수로 메모리 제한

```bash
# PyTorch - 메모리 할당 전략 변경
docker run --gpus all \
  -e PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:512 \
  pytorch/pytorch:latest python train.py

# TensorFlow - GPU 메모리 성장 허용
docker run --gpus all \
  -e TF_FORCE_GPU_ALLOW_GROWTH=true \
  tensorflow/tensorflow:latest-gpu python train.py
```

### 실전 예시: 다중 GPU 학습 환경

여러 실험을 동시에 실행하면서 각각 다른 GPU를 사용하는 설정입니다.

```yaml
version: '3.8'

services:
  experiment-1:
    image: pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime
    container_name: exp1-gpu0
    volumes:
      - ./exp1:/workspace
    working_dir: /workspace
    command: python train.py --epochs 100
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['0']
              capabilities: [gpu]
    environment:
      - CUDA_VISIBLE_DEVICES=0

  experiment-2:
    image: pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime
    container_name: exp2-gpu1
    volumes:
      - ./exp2:/workspace
    working_dir: /workspace
    command: python train.py --epochs 100
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['1']
              capabilities: [gpu]
    environment:
      - CUDA_VISIBLE_DEVICES=0

  experiment-3:
    image: pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime
    container_name: exp3-gpu2-3
    volumes:
      - ./exp3:/workspace
    working_dir: /workspace
    command: python train.py --epochs 100 --multi-gpu
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['2', '3']
              capabilities: [gpu]
    environment:
      - CUDA_VISIBLE_DEVICES=0,1
```

### GPU 사용 확인

특정 GPU의 사용 상태를 확인합니다.

```bash
# 특정 GPU만 모니터링
nvidia-smi -i 0  # GPU 0번만 표시

# 여러 GPU 모니터링
nvidia-smi -i 0,2  # GPU 0번과 2번만 표시
```

---

## 성능 모니터링

GPU 사용률과 성능을 모니터링하는 다양한 방법을 소개합니다.

### 기본 GPU 상태 확인

가장 기본적인 GPU 상태 확인 방법입니다.

```bash
nvidia-smi
```

출력 정보:
- GPU 사용률 (GPU-Util)
- 메모리 사용량 (Memory-Usage)
- 온도 (Temp)
- 전력 소비 (Power)
- 실행 중인 프로세스 목록

### 실시간 모니터링

#### watch 명령으로 주기적 갱신

```bash
# 1초마다 nvidia-smi 갱신
watch -n 1 nvidia-smi

# 0.5초마다 갱신 (더 빠른 갱신)
watch -n 0.5 nvidia-smi

# 특정 GPU만 모니터링
watch -n 1 'nvidia-smi -i 0,1'
```

**종료**: `Ctrl + C`

#### nvidia-smi dmon - 실시간 통계

`dmon` (device monitoring) 모드는 간결한 형태로 실시간 통계를 제공합니다.

```bash
# 기본 실시간 모니터링
nvidia-smi dmon

# 1초 간격으로 모니터링
nvidia-smi dmon -s pucvmet -d 1
```

출력 예시:
```
# gpu   pwr gtemp mtemp    sm   mem   enc   dec  mclk  pclk
# Idx     W     C     C     %     %     %     %   MHz   MHz
    0    45    42     -    12     8     0     0  5001  1410
    1   120    65     -    98    85     0     0  5001  1875
```

**컬럼 설명**:
- `pwr`: 전력 소비 (W)
- `gtemp`: GPU 온도 (°C)
- `sm`: GPU 사용률 (%)
- `mem`: 메모리 사용률 (%)
- `enc/dec`: 인코더/디코더 사용률
- `mclk/pclk`: 메모리/프로세서 클럭

**옵션**:
```bash
# 특정 GPU만 모니터링
nvidia-smi dmon -i 0,1

# 10회만 샘플링 후 종료
nvidia-smi dmon -c 10

# 모니터링 항목 선택
nvidia-smi dmon -s pucvmet
# p: 전력, u: 사용률, c: 프로세서 클럭, v: 전압, m: 메모리, e: 인코더, t: 온도
```

### 컨테이너별 GPU 사용률 확인

실행 중인 Docker 컨테이너의 GPU 사용을 확인합니다.

#### 방법 1: nvidia-smi로 프로세스 확인

```bash
# GPU를 사용하는 프로세스 목록 표시
nvidia-smi pmon

# 또는 nvidia-smi의 프로세스 섹션 확인
nvidia-smi
```

프로세스 섹션에서 각 프로세스의 PID와 메모리 사용량을 확인할 수 있습니다.

#### 방법 2: 컨테이너 내부에서 확인

```bash
# 실행 중인 컨테이너에서 nvidia-smi 실행
docker exec -it <container_name> nvidia-smi

# Docker Compose 사용 시
docker compose exec <service_name> nvidia-smi
```

#### 방법 3: GPU 메트릭 쿼리

특정 메트릭만 쿼리하여 확인합니다.

```bash
# GPU 메모리 사용량만 확인
nvidia-smi --query-gpu=index,name,memory.used,memory.total --format=csv

# GPU 사용률과 온도 확인
nvidia-smi --query-gpu=index,name,utilization.gpu,temperature.gpu --format=csv

# 1초마다 갱신하면서 모니터링
watch -n 1 'nvidia-smi --query-gpu=index,name,utilization.gpu,memory.used,memory.total,temperature.gpu --format=csv'
```

출력 예시:
```
index, name, utilization.gpu [%], memory.used [MiB], memory.total [MiB], temperature.gpu
0, NVIDIA GeForce RTX 3090, 98 %, 22000 MiB, 24576 MiB, 65
1, NVIDIA GeForce RTX 3090, 45 %, 8000 MiB, 24576 MiB, 52
```

### 프로세스 모니터링

#### nvidia-smi pmon - 프로세스별 모니터링

```bash
# 프로세스별 GPU/메모리 사용률 실시간 모니터링
nvidia-smi pmon

# 1초 간격으로 갱신
nvidia-smi pmon -d 1
```

출력 예시:
```
# gpu        pid  type    sm   mem   enc   dec   command
# Idx          #   C/G     %     %     %     %   name
    0      12345     C    98    85     0     0   python
    1      12346     C    95    80     0     0   python
```

### 로그 파일로 저장

장기간 모니터링 데이터를 파일로 저장합니다.

```bash
# 10초마다 GPU 상태를 파일에 기록
nvidia-smi dmon -d 10 -o TD > gpu_monitoring.log

# CSV 형식으로 저장
nvidia-smi --query-gpu=timestamp,name,utilization.gpu,memory.used,temperature.gpu \
  --format=csv -l 10 > gpu_metrics.csv
```

### Docker 컨테이너 GPU 통계

Docker stats 명령으로는 GPU 정보를 직접 볼 수 없지만, nvidia-smi와 조합하여 확인할 수 있습니다.

```bash
# 컨테이너 CPU/메모리 사용량
docker stats

# GPU 사용 중인 컨테이너의 PID 확인 후 nvidia-smi와 매칭
docker inspect <container_name> | grep Pid
nvidia-smi | grep <PID>
```

### 모니터링 스크립트 예시

여러 정보를 한 번에 모니터링하는 간단한 스크립트입니다.

{% raw %}
```bash
#!/bin/bash
# gpu_monitor.sh

while true; do
    clear
    echo "=== GPU Status ==="
    nvidia-smi --query-gpu=index,name,utilization.gpu,memory.used,memory.total,temperature.gpu \
      --format=csv,noheader,nounits
    echo ""
    echo "=== Docker Containers ==="
    docker ps --format "table {{.Names}}\t{{.Status}}"
    sleep 2
done
```
{% endraw %}

사용법:
```bash
chmod +x gpu_monitor.sh
./gpu_monitor.sh
```

### 웹 기반 모니터링 (선택사항)

더 고급 모니터링이 필요한 경우 다음 도구들을 고려할 수 있습니다:

- **nvtop**: TUI 기반 GPU 모니터링 도구
  ```bash
  sudo apt install nvtop
  nvtop
  ```

- **Grafana + Prometheus**: 프로덕션 환경 모니터링
- **TensorBoard**: 딥러닝 학습 과정 모니터링

---

## 문제 해결

### 1. "could not select device driver with capabilities: [[gpu]]" 에러

이 에러는 Docker가 GPU를 인식하지 못할 때 발생합니다.

**해결 방법:**
- NVIDIA Container Toolkit이 설치되어 있는지 확인
- Docker 데몬 재시작: `sudo systemctl restart docker`
- Docker 실행 시 `--gpus all` 옵션 사용 확인

### 2. nvidia-smi 명령 실패

**해결 방법:**
- NVIDIA 드라이버가 설치되어 있는지 확인: `dpkg -l | grep nvidia-driver`
- 설치되지 않았다면 위의 드라이버 설치 과정 진행
- 커널 모듈 확인: `lsmod | grep nvidia`

### 3. 드라이버 설치 후 화면이 나오지 않는 경우

**해결 방법:**
- 복구 모드로 부팅 (부팅 시 Shift 키 누름)
- 드라이버 제거: `sudo apt remove nvidia-*`
- 오픈소스 드라이버로 전환: `sudo apt install xserver-xorg-video-nouveau`
- 재부팅 후 다시 드라이버 설치 시도

### 4. 여러 버전의 드라이버가 충돌하는 경우

**해결 방법:**
```bash
# 기존 드라이버 완전 제거
sudo apt purge nvidia-*
sudo apt autoremove

# 저장소 업데이트
sudo apt update

# 드라이버 재설치
sudo ubuntu-drivers autoinstall
```

---

NVIDIA Container Toolkit과 드라이버를 올바르게 설정하면 Docker 컨테이너 환경에서 강력한 GPU 가속을 활용할 수 있습니다.  
특히 머신러닝, 딥러닝, 과학 컴퓨팅 등 GPU를 필요로 하는 워크로드를 컨테이너로 쉽게 배포하고 관리할 수 있습니다.  
