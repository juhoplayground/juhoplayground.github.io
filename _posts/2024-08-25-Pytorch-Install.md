---
layout: post
title: Pytorch GPU를 설정하는 방법
author: 'Juho'
date: 2024-08-25 09:00:00 +0900
categories: [Pytorch]
tags: [Pytorch, GPU]
pin: True
toc : True
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
0. [NVIDIA 그래픽 카드 확인](#nvidia-그래픽-카드-확인)
1. [Pytorch 버전 확인 및 설치](#pytorch-버전-확인-및-설치)
2. [CUDA 설치](#cuda-설치)
3. [cuDNN 설치](#cudnn-설치)
4. [환경 변수 등록](#환경-변수-등록)
5. [GPU 사용 확인](#gpu-사용-확인)
6. [오류 수정](#오류-수정)

## NVIDIA 그래픽 카드 확인
터미널에 `nvidia-smi`를 입력한다.
```
Sun Aug 25 23:10:10 2024       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 560.76                 Driver Version: 560.76         CUDA Version: 12.6     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                  Driver-Model | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 4050 ...  WDDM  |   00000000:01:00.0 Off |                  N/A |
| N/A   50C    P3             15W /   69W |    1185MiB /   6141MiB |      5%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
```
그럼 위와 같은 결과가 출력이 되는데 내 노트북의 GPU는 NVIDIA GeForce RTX 4050인것을 확인할 수 있다.<br/>
그리고 가능한 CUDA 버전은 12.6까지인것을 확인할 수 있다.<br/>

## Pytorch 버전 확인 및 설치
[Pytorch](https://pytorch.org/get-started/locally/){:target="_blank"}를 들어가보면 Stable한 버전의 Pytorch를 확인할 수 있다.<br/>
![image](https://github.com/user-attachments/assets/cf07f999-6d84-4286-90b4-4af9def3b757)
현재 Stable한 버전은 2.4.0인데 Python은 3.8버전 이상을 요구한다.<br/>
Windows환경에서 pip을 이용해 설치를 할 것이므로 선택해주고 <br/>
가장 최신 버전의 CUDA인 12.4를 선택해준다.<br/>
그럼 아래의 `Run this Command:` 부분에 있는 `pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124`를 실행해주면 Pytorch는 설치가 끝난다.<br/>

## CUDA 설치
[CUDA](https://developer.nvidia.com/cuda-toolkit-archive){:target="_blank"}에 들어간다.<br/>
그럼 아까 Pytorch에서 선택한 버전인 CUDA 12.4 버전을 찾는다.<br/>
![image](https://github.com/user-attachments/assets/06fdfa20-2bd4-45d1-9ebb-f4d891ced7aa)
해당 파일을 다운로드 해서 설치하면  CUDA는 설치가 완료된다.<br/>
터미널에서 `nvcc --version` 명령어를 실행해보면 <br/>
```
Copyright (c) 2005-2024 NVIDIA Corporation
Built on Thu_Mar_28_02:30:10_Pacific_Daylight_Time_2024
Cuda compilation tools, release 12.4, V12.4.131
Build cuda_12.4.r12.4/compiler.34097967_0
```
방금 설치한 CUDA 버전이 정상적으로 출력되는 것을 확인할 수 있다.<br/>


## cuDNN 설치
[CuDNN](https://developer.nvidia.com/rdp/cudnn-archive){:target="_blank"}에 들어간다.<br/>
설치한 CUDA 버전이 12.4버전이므로 12.X 버전의 cuDNN을 다운로드 받는다.<br/>
![image](https://github.com/user-attachments/assets/539d0539-b535-48fa-b4e7-265f7962f217)
zip 파일을 압축 해제하고 `bin`, `include`, `lib` 폴더와 `LICENSE` 파일을 CUDA 폴더에 덮어씌워준다.<br/>
CUDA 폴더는 기본 설정으로 설치했다면 `C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.4` 경로에 존재한다.<br/>

## 환경 변수 등록
![image](https://github.com/user-attachments/assets/fcc3e14f-c204-44d8-8341-83928329d97b)<br/>
환경 변수를 클릭한 다음 Path를 선택하고 편집을 선택한다.<br/>
![image](https://github.com/user-attachments/assets/1c34e852-6894-4be4-b2de-2577ebad8b56) <br/>
그리고 아까 붙여 넣었던 `bin`, `include`, `lib`에 대한 환경 변수를 등록한다.<br/>


## GPU 사용 확인
```python
import torch
print(torch.__version__)
print(torch.cuda.is_available())
```
위의 코드를 실행했을 때 
![image](https://github.com/user-attachments/assets/57911088-4326-4da9-80a9-faff30d21f70)
이렇게 출력이 된다면 정상적으로 Pytorch에서 GPU를 사용할 수 있게 된 것 이다.<br/>

근데 예전 노트북에 설정할때는 문제가 없었는데 이번 노트북에는 설치를 정상적으로 했는데<br/>
`OSError: [WinError 126] 지정된 모듈을 찾을 수 없습니다. Error loading ~~ venv\Lib\site-packages\torch\lib\fbgemm.dll" or one of its dependencies.` 오류가 발생하면서 실행이 되지 않았다.<br/>

## 오류 수정
fbgemm.dll과 같은 파일은 Microsoft의 Visual C++ Redistributable에 의존할 수 있기 때문에 <br/>
[Visual C++ Redistributable](https://learn.microsoft.com/en-us/cpp/windows/latest-supported-vc-redist?view=msvc-170){:target="_blank"}에
접속해서 최신 버전의 Visual C++ Redistributable을 다운로드 받아 보았다.<br/>
그리고 다시 GPU 사용 확인을 시도했는데 여전히 같은 문제가 발생했다.<br/>
아쉽게 이 방법으로는 해결이 되지 않아서 다른 방법을 찾아야 했다.<br/>
<br/>
그래서 `fbgemm.dll`이 로드하지 못하는 다른 DLL이 있는지 확인하려고 [Dependencies](https://github.com/lucasg/Dependencies){:target="_blank"}에서 `DependenciesGui`를 설치했다.<br/>
그리고 `venv\Lib\site-packages\torch\lib\fbgemm.dll` 파일을 확인 해보니<br/>
![351870942-065a323f-ce56-48b5-8454-d95b1f345009](https://github.com/user-attachments/assets/3e1c1d93-90c6-4a9e-8ba3-59a7026f3769)<br/>
위의 이미지 처럼 `libomp140.x86_64.dll` 파일이 없어서 발생한 문제 같았다.<br/>
그래서 `libomp140.x86_64.dll`을 다운로드 받아서 `C:\Windows\System32` 폴더에 붙여넣었다.<br/>

![image](https://github.com/user-attachments/assets/7acd5dbd-9110-433a-9fa9-de951ef5294b) <br/>
그러자 문제 없이 로드 되는 것을 확인할 수 있었고 다시 GPU 사용 확인을 했더니 이상없이 되었다.<br/


---

<br/>


예전에 대학생 때는 CUDA 설정하기가 꽤 어려웠었는데, 요즘은 많이 편해진 것 같다.<br/>