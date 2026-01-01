---
layout: post
title: GCP IAP를 이용한 안전한 SSH 접속 설정 가이드
author: 'Juho'
date: 2026-01-01 00:00:00 +0900
categories: [GCP]
tags: [GCP, IAP, SSH]
toc: True
---

Google Cloud Platform(GCP)에서 Identity-Aware Proxy(IAP)를 사용하면 공개 IP 없이도 안전하게 VM 인스턴스에 SSH 접속할 수 있습니다.
이 글에서는 IAP 터널링을 통한 SSH 접속 설정 방법을 단계별로 알아보겠습니다.

## 목차
1. [사전 준비사항](#사전-준비사항)
2. [IAP 설정 방법](#iap-설정-방법)
3. [로컬에서 IAP를 이용한 SSH 접속](#로컬에서-iap를-이용한-ssh-접속)
4. [IAP 사용의 장점](#iap-사용의-장점)
5. [주의사항](#주의사항)
6. [마무리](#마무리)

## 사전 준비사항

### OS Login 설정 확인

OS Login이 활성화되어 있으면 프로젝트/인스턴스 메타데이터의 SSH 키를 사용하지 않습니다.  
이 점을 염두에 두고 설정을 진행해야 합니다.

### gcloud CLI 설치 및 초기화  

```bash
gcloud init  

## 생성한 Public Key 등록
gcloud compute os-login ssh-keys add \
  --key-file ~/.ssh/pem_key.pub
```

## IAP 설정 방법

### 1. 사용자 권한 설정

IAP를 통한 SSH 접속을 위해서는 다음 IAM 역할이 필요합니다:

- `roles/iap.tunnelResourceAccessor` - IAP 터널 리소스에 대한 접근 권한  
- `roles/compute.osLogin` - OS Login 사용 권한  

### 2. 방화벽 규칙 생성

VPC 네트워크에서 IAP의 IP 범위로부터 SSH 트래픽을 허용하는 방화벽 규칙을 생성합니다.  

**설정 방법:**
1. GCP 콘솔에서 **VPC 네트워크 > 방화벽 > 방화벽 규칙 만들기** 선택  
2. 다음과 같이 설정:  
   - **이름**: `allow-iap-ssh`  
   - **설명**: `Allow SSH from IAP`  
   - **네트워크**: `default` (VM이 속한 VPC와 동일해야 함)  
   - **트래픽 방향**: 인그레스, 허용  
   - **대상**: 지정된 대상 태그  
   - **대상 태그**: `iap-ssh`  
   - **소스 필터**: `35.235.240.0/20` (IAP의 IP 범위)  
   - **프로토콜 및 포트**: `tcp:22`  

### 3. VM 인스턴스에 네트워크 태그 추가

VM 인스턴스 설정을 수정하여 네트워크 태그에 `iap-ssh`를 추가합니다.  

## 로컬에서 IAP를 이용한 SSH 접속

### 기본 접속 명령어

```bash
## IAP TCP 업로드 대역폭 증가를 위한 numpy 설치
$(gcloud info --format="value(basic.python_location)") -m pip install numpy  

gcloud compute ssh {gcp_vm_instance_name} \
  --zone asia-northeast3-a \
  --tunnel-through-iap
```

### VSCode SSH 설정

VSCode에서 원격 개발을 위해 SSH 설정 파일(`~/.ssh/config`)에 다음 내용을 추가합니다:

```bash
Host gcp_vm_instance_name
    HostName gcp_vm_instance_name
    User {gcp_user_name}
    IdentityFile ~/.ssh/gcp_ai_agent_server
    IdentitiesOnly yes
    ProxyCommand gcloud compute start-iap-tunnel {gcp_vm_instance_name} 22 --zone asia-northeast3-a --project project-{project_id_number}--listen-on-stdin
```

### 연결 테스트

SSH 연결을 테스트하려면 verbose 모드로 실행하여 상세 로그를 확인할 수 있습니다:  

```bash
ssh -vvv {gcp_vm_instance_name}
```

## IAP 사용의 장점

1. **보안 강화**: 공개 IP 없이도 안전하게 SSH 접속 가능  
2. **중앙 집중식 액세스 제어**: IAM을 통한 세밀한 권한 관리  
3. **감사 로그**: 모든 접속 기록이 Cloud Logging에 저장  
4. **방화벽 규칙 단순화**: 특정 IP 화이트리스트 관리 불필요  

## 주의사항

- IAP의 소스 IP 범위(`35.235.240.0/20`)는 변경될 수 있으므로 [공식 문서](https://cloud.google.com/iap/docs/using-tcp-forwarding){:target="_blank"}를 정기적으로 확인하세요  
- OS Login 사용 시 사용자 이름이 이메일 주소 기반으로 자동 생성됩니다  
- `ProxyCommand`에서 프로젝트 ID는 실제 프로젝트 ID로 변경해야 합니다  

## 마무리

IAP를 통한 SSH 접속은 GCP 환경에서 보안을 강화하면서도 편리하게 VM 인스턴스를 관리할 수 있는 방법입니다.  
특히 VSCode와 같은 IDE와 연동하면 로컬 개발 환경처럼 편리하게 원격 개발을 수행할 수 있습니다.  
