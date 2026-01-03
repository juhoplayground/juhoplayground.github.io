---
layout: post
title: Ubuntu에서 Docker 설치 및 설정
author: 'Juho'
date: 2026-01-03 00:00:00 +0900
categories: [DevOps]
tags: [Docker, DevOps]
toc: true
---

## 목차
1. [개요](#개요)
2. [사전 준비](#사전-준비)
3. [Docker 공식 저장소 설정](#docker-공식-저장소-설정)
4. [Docker 설치](#docker-설치)
5. [Docker Data Root 설정](#docker-data-root-설정)
6. [Containerd 설정](#containerd-설정)
7. [Docker 서비스 시작](#docker-서비스-시작)
8. [사용자 권한 설정](#사용자-권한-설정)
9. [설정 확인](#설정-확인)
10. [트러블슈팅](#트러블슈팅)
11. [마무리](#마무리)
12. [참고 자료](#참고-자료)

## 개요

Docker는 컨테이너 기반 가상화 플랫폼으로, 애플리케이션을 신속하게 배포하고 관리할 수 있게 해줍니다.  
이 가이드에서는 Ubuntu 시스템에 Docker를 설치하고, 데이터 저장 경로를 커스터마이징하며, 사용자 권한을 설정하는 전체 과정을 다룹니다.  

## 사전 준비

Docker 설치를 시작하기 전에 필요한 패키지들을 설치합니다.  

```bash
sudo apt update
sudo apt install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release
```

## Docker 공식 저장소 설정

### GPG 키 등록

Docker 공식 GPG 키를 등록하여 패키지의 무결성을 검증합니다.  

```bash
# GPG 키링 디렉토리 생성
sudo install -m 0755 -d /etc/apt/keyrings

# Docker 공식 GPG 키 다운로드 및 등록
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# GPG 키 파일 권한 설정
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### APT 저장소 추가

Docker 공식 APT 저장소를 시스템에 추가합니다.  

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
```

## Docker 설치

Docker Engine과 관련 플러그인들을 설치합니다.  

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

설치되는 주요 컴포넌트:
- **docker-ce**: Docker Engine 커뮤니티 에디션  
- **docker-ce-cli**: Docker 명령줄 도구  
- **containerd.io**: 컨테이너 런타임  
- **docker-buildx-plugin**: 확장된 빌드 기능  
- **docker-compose-plugin**: Docker Compose V2  

## Docker Data Root 설정

기본적으로 Docker는 `/var/lib/docker`에 데이터를 저장합니다.  
디스크 공간 관리를 위해 별도 경로로 변경할 수 있습니다.  

```bash
# 새로운 데이터 디렉토리 생성
sudo mkdir -p /data/docker

# Docker 설정 디렉토리 생성
sudo mkdir -p /etc/docker

# daemon.json 파일 생성
sudo vi /etc/docker/daemon.json
```

`/etc/docker/daemon.json` 파일 내용:

```json
{
  "data-root": "/data/docker"
}
```

## Containerd 설정

Containerd의 데이터 경로도 별도로 설정할 수 있습니다.  

```bash
# Containerd 데이터 디렉토리 생성
sudo mkdir -p /data/containerd

# fstab에 바인드 마운트 추가
echo '/data/containerd  /var/lib/containerd  none  bind  0  0' | sudo tee -a /etc/fstab

# 마운트 적용
sudo mount -a

# systemd 데몬 리로드
sudo systemctl daemon-reload

# 마운트 확인
mount | grep containerd
```

## Docker 서비스 시작

```bash
# 부팅 시 자동 시작 설정
sudo systemctl enable containerd docker

# 서비스 시작
sudo systemctl start containerd
sudo systemctl start docker

# Docker 상태 확인
sudo systemctl status docker --no-pager
```

## 사용자 권한 설정

일반 사용자가 `sudo` 없이 Docker 명령을 실행할 수 있도록 설정합니다.  

```bash
# docker 그룹 확인 및 생성
getent group docker || sudo groupadd docker

# 현재 사용자를 docker 그룹에 추가
sudo usermod -aG docker $USER

# 그룹 변경 적용 (재로그인 대신)
newgrp docker
```

> 완전한 적용을 위해서는 로그아웃 후 다시 로그인하는 것이 권장됩니다.  

## 설정 확인

### Data Root 확인

```bash
{% raw %}docker info --format '{{.DockerRootDir}}'{% endraw %}
```

출력이 `/data/docker`로 나오면 정상적으로 설정된 것입니다.  

### Docker 버전 확인

```bash
docker --version
docker compose version
```

### 테스트 컨테이너 실행

```bash
docker run hello-world
```

## 트러블슈팅

### Permission Denied 에러

사용자가 docker 그룹에 추가되지 않은 경우 발생합니다.  

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Docker 데몬이 시작되지 않는 경우

`daemon.json` 파일의 JSON 문법을 확인하고, 로그를 확인합니다.  

```bash
sudo journalctl -xeu docker.service
```

### 디스크 공간 부족

Docker 이미지와 컨테이너가 많은 공간을 차지할 수 있습니다.  
정기적으로 정리합니다.  

```bash
# 사용하지 않는 리소스 정리
docker system prune -a

# 볼륨까지 포함하여 정리
docker system prune -a --volumes
```

## 마무리

이제 Ubuntu 시스템에 Docker가 성공적으로 설치되었고, 데이터 저장 경로도 커스터마이징되었습니다.  
Docker를 활용하여 다양한 컨테이너 기반 애플리케이션을 배포하고 관리할 수 있습니다.  

## 참고 자료

- [Docker 공식 문서](https://docs.docker.com/)
- [Docker Engine 설치 가이드](https://docs.docker.com/engine/install/ubuntu/)
- [Docker Post-installation 가이드](https://docs.docker.com/engine/install/linux-postinstall/)
