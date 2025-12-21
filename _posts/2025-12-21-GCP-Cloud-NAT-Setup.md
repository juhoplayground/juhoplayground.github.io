---
layout: post
title: GCP Cloud NAT 구축 가이드
author: 'Juho'
date: 2025-12-21 00:00:00 +0900
categories: [GCP]
tags: [GCP, Cloud NAT, DevOps, 네트워크, Cloud Router]
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
1. [Cloud NAT 개요](#cloud-nat-개요)
2. [Cloud NAT 구축 단계](#cloud-nat-구축-단계)
3. [모니터링 및 검증](#모니터링-및-검증)
4. [비용 최적화](#비용-최적화)
5. [주의사항](#주의사항)



## Cloud NAT 개요

### Cloud NAT란?

Cloud NAT(Network Address Translation)는 GCP의 완전 관리형 서비스로 외부 IP 주소가 없는 VM 인스턴스가 인터넷에 액세스할 수 있도록 합니다.  
중요: Cloud NAT는 아웃바운드(Outbound) 전용 서비스입니다.  
VM에서 인터넷으로 나가는 트래픽만 처리하며 인터넷에서 VM으로 들어오는 인바운드 트래픽은 처리하지 않습니다.  

### Cloud NAT가 필요한 이유

#### 1. IP 화이트리스팅 관리  
- 외부 API(결제 게이트웨이, 인증 서비스 등)는 보안을 위해 IP 화이트리스팅 요구  
- VM이 동적으로 생성/삭제될 때마다 IP가 변경되면 관리 불가능  
- Cloud NAT로 하나의 고정 IP로 모든 VM의 외부 통신 관리  

#### 2. 보안 강화  
- VM에 외부 IP를 할당하면 인터넷에서 직접 접근 가능 (보안 위험)  
- Cloud NAT 사용 시 VM은 Private IP만 가지며, 외부에서 접근 불가  
- 필요한 아웃바운드 통신만 허용하여 공격 표면 축소  

#### 3. 비용 절감  
- 여러 VM이 하나의 NAT 게이트웨이와 고정 IP 공유  
- VM마다 개별 외부 IP를 할당하는 것보다 비용 효율적  

### 주요 특징

1. 관리형 서비스: 별도 인스턴스 없이 GCP가 완전 관리  
2. 고가용성: 자동으로 여러 가용 영역에 분산  
3. 확장성: 트래픽 증가에 따라 자동으로 확장  
4. 아웃바운드 전용: VM → 인터넷 방향 트래픽만 처리  

### 아키텍처 구성  

```
┌─────────────────────────────────────────┐
│  VPC Network                            │
│                                         │
│  ┌──────────────────────┐               │
│  │   Cloud Router       │               │
│  │  (Regional)          │               │
│  └──────────┬───────────┘               │
│             │                           │
│  ┌──────────▼───────────┐               │      ┌─────────────┐
│  │   Cloud NAT          │◄──────────────┼──────┤ 고정 IP     │
│  │   Gateway            │               │      │ (Static)    │
│  └──────────┬───────────┘               │      └─────────────┘
│             │                           │              │
│    ┌────────┴────────┐                  │              │
│    ▼                 ▼                  │              ▼
│  ┌────┐  ┌────┐  ┌────┐                 │        Internet
│  │VM1 │  │VM2 │  │VM3 │                 │
│  └────┘  └────┘  └────┘                 │
│  (No External IP)                       │
└─────────────────────────────────────────┘
```

## Cloud NAT 구축 단계

### 1단계: Cloud Router 생성

Cloud NAT는 Cloud Router를 통해 작동하므로, 먼저 Cloud Router를 생성해야 합니다.

#### gcloud CLI 사용
```bash
gcloud compute routers create nat-router \
    --network=default \
    --region=asia-northeast3
```

#### 설정 확인
```bash
gcloud compute routers describe nat-router --region=asia-northeast3
```

### 2단계: 고정 IP 주소 예약 (권장)

외부 API가 IP 화이트리스팅(allowlist)을 요구하는 경우 고정 IP 주소를 예약하는 것이 필수입니다.  
일반적인 외부 호출의 경우 고정 IP가 필수는 아니지만 안정적인 관리를 위해 권장됩니다.  

```bash
# 고정 IP 주소 예약
gcloud compute addresses create nat-external-ip \
    --region=asia-northeast3

# 예약된 IP 주소 확인
gcloud compute addresses describe nat-external-ip --region=asia-northeast3
```

출력 예시:
```yaml
address: 34.64.123.456
addressType: EXTERNAL
creationTimestamp: '2025-12-20T10:00:00.000-08:00'
name: nat-external-ip
region: asia-northeast3
status: RESERVED
```

### 3단계: Cloud NAT 게이트웨이 생성

이제 Cloud Router에 NAT 게이트웨이를 추가합니다.  

#### gcloud CLI 사용
```bash
gcloud compute routers nats create nat-config \
    --router=nat-router \
    --region=asia-northeast3 \
    --nat-external-ip-pool=nat-external-ip \
    --nat-all-subnet-ip-ranges \
    --enable-logging
```

#### 주요 옵션 설명

| 옵션 | 설명 |
|------|------|
| `--router` | 사용할 Cloud Router 이름 |
| `--nat-external-ip-pool` | 사용할 고정 IP 주소 (2단계에서 생성) |
| `--nat-all-subnet-ip-ranges` | 모든 서브넷에 NAT 적용 |
| `--enable-logging` | NAT 로깅 활성화 (모니터링용) |

### 4단계: GCP Console을 통한 설정 (대안)

CLI 대신 웹 콘솔에서도 설정 가능합니다.  
Network Services → Cloud NAT → Create Cloud NAT Gateway  

#### 설정 항목

1. Gateway name: `nat-gateway` (원하는 이름 입력)  
2. VPC network: 사용 중인 VPC 선택  
3. Region: 인스턴스가 있는 리전 선택 (예: `asia-northeast3`)  
4. Cloud Router:  
   - 기존 라우터 선택 또는  
   - Create new router 클릭하여 새로 생성  
5. NAT mapping:  
   - All subnets (권장) - 모든 서브넷에 NAT 적용  
   - Custom - 특정 서브넷만 선택  
6. NAT IP addresses:  
   - Manual 선택 (권장)  
   - Create IP address 클릭하여 고정 IP 생성  


## 모니터링 및 검증

### Cloud Logging에서 NAT 로그 확인

Cloud Console → Logging → Logs Explorer에서 다음 쿼리 실행:  

```sql
resource.type="nat_gateway"
logName="projects/[PROJECT_ID]/logs/compute.googleapis.com%2Fnat_flows"
```

#### 로그 예시
```json
{
  "insertId": "...",
  "jsonPayload": {
    "connection": {
      "src_ip": "10.128.0.5",
      "src_port": 45678,
      "dest_ip": "203.245.1.10",
      "dest_port": 443,
      "nat_ip": "34.64.123.456",
      "nat_port": 12345
    },
    "vpc": {
      "vpc_name": "default",
      "subnetwork_name": "default"
    }
  },
  "logName": "projects/my-project/logs/compute.googleapis.com%2Fnat_flows"
}
```

### 주요 메트릭 모니터링

Cloud Monitoring → Metrics Explorer에서 다음 메트릭을 추가:  

| 메트릭 | 설명 | 임계값 권장 |
|--------|------|-------------|
| `router.googleapis.com/nat/allocated_ports` | 할당된 NAT 포트 수 | < 60,000 (최대 64,512) |
| `router.googleapis.com/nat/dropped_sent_packets_count` | 드롭된 송신 패킷 수 | < 100/min |
| `router.googleapis.com/nat/nat_allocation_failed` | NAT 할당 실패 횟수 | 0 |
| `router.googleapis.com/nat/sent_packets_count` | 총 송신 패킷 수 | 모니터링 |
| `router.googleapis.com/nat/received_packets_count` | 총 수신 패킷 수 | 모니터링 |

참고: 메트릭은 GCP Console의 Metrics Explorer에서 "NAT"로 검색하면 쉽게 찾을 수 있습니다.  

### 알림 설정

포트 고갈 방지를 위한 알림 설정 예시:  

```bash
# gcloud CLI로 알림 정책 생성
gcloud alpha monitoring policies create \
    --notification-channels=CHANNEL_ID \
    --display-name="Cloud NAT Port Exhaustion" \
    --condition-display-name="Allocated ports > 60000" \
    --condition-threshold-value=60000 \
    --condition-threshold-duration=300s
```

### 설정 검증 및 테스트

Cloud NAT 설정이 올바르게 작동하는지 확인하는 방법입니다.  

#### 1. VM 인스턴스 생성 (외부 IP 없이)

```bash
# 외부 IP 없는 VM 생성
gcloud compute instances create test-vm \
    --zone=asia-northeast3-a \
    --machine-type=e2-micro \
    --network-interface=network=default,no-address \
    --metadata=startup-script='#!/bin/bash
apt-get update
apt-get install -y curl'
```

#### 2. IAP를 통한 SSH 접속

```bash
# IAP 터널을 통해 VM 접속
gcloud compute ssh test-vm \
    --zone=asia-northeast3-a \
    --tunnel-through-iap
```

#### 3. Cloud NAT 작동 확인

VM에 접속한 후 다음 명령어로 외부 IP 확인:  

```bash
# Cloud NAT를 통해 나가는 외부 IP 확인
curl ifconfig.me
# 또는
curl https://api.ipify.org

# 출력 예시: 34.64.123.456 (Cloud NAT의 고정 IP)
```

예상 결과: Cloud NAT에 할당한 고정 IP가 출력되어야 합니다.  

#### 4. 외부 API 호출 테스트

```bash
# 외부 API 호출 테스트
curl -I https://www.google.com
# 정상 응답: HTTP/2 200

# DNS 조회 테스트
nslookup google.com
# 정상 응답: DNS 서버 정보 출력
```

#### 5. 인바운드 트래픽 차단 확인

외부에서 VM으로의 직접 접근이 차단되었는지 확인:  

```bash
# 로컬 PC에서 실행 (VM의 Private IP 확인 후)
ping 10.128.0.5  # 타임아웃 발생 (정상)
curl http://10.128.0.5  # 연결 실패 (정상)
```

예상 결과: 외부에서 Private IP로 직접 접근 불가능 (보안 강화 확인)  

#### 6. Cloud NAT 로그 확인

```bash
# Cloud NAT를 통한 연결 로그 확인
gcloud logging read \
    "resource.type=nat_gateway AND
     jsonPayload.connection.nat_ip=\"34.64.123.456\"" \
    --limit=10 \
    --format=json
```

## 비용 최적화

### Cloud NAT 비용 구조

주의: 아래 가격은 2025년 12월 기준이며 변동될 수 있습니다.  
최신 가격은 [GCP 공식 가격 정책](https://cloud.google.com/nat/pricing){:target="_blank"}을 확인하세요.

| 항목 | 비용 (asia-northeast3 기준) |
|------|----------------------------|
| NAT 게이트웨이 | 시간당 $0.045 (~₩58) |
| 데이터 처리 | GB당 $0.045 (~₩58) |
| 고정 IP (사용 중) | 시간당 $0.004 (~₩5) |
| 고정 IP (미사용) | 시간당 $0.01 (~₩13) |

월간 예상 비용 (1개 NAT Gateway + 1TB 전송)  
```
NAT Gateway: $0.045 × 24시간 × 30일 = $32.40
데이터 처리: $0.045 × 1,000GB = $45.00
고정 IP (사용 중): $0.004 × 24시간 × 30일 = $2.88
인터넷 Egress 비용: 별도 부과 (리전에 따라 GB당 $0.08~$0.23)
──────────────────────────────────────
총 월간 비용 (NAT만): ~$80.28 (~₩104,000)
총 월간 비용 (Egress 포함): ~$150~$280 (~₩195,000~₩364,000)
```

참고: NAT 데이터 처리 요금과 별도로 인터넷 egress(송신) 비용이 추가로 부과됩니다.  

### 비용 절감 팁

#### 1. 필요한 리전에만 설정
```bash
# 나쁜 예: 모든 리전에 NAT 설정
# asia-northeast3, us-central1, europe-west1... (비용 3배)

# 좋은 예: 실제 사용하는 리전에만 설정
# asia-northeast3만 사용 → 비용 1/3
```

#### 2. 로깅 최적화
```bash
# 전체 로깅 (비용 높음)
gcloud compute routers nats update nat-config \
    --router=nat-router \
    --region=asia-northeast3 \
    --enable-logging \
    --log-filter=ALL  # 모든 연결 로깅

# 오류만 로깅 (권장)
gcloud compute routers nats update nat-config \
    --router=nat-router \
    --region=asia-northeast3 \
    --enable-logging \
    --log-filter=ERRORS_ONLY  # 오류만 로깅 → 비용 절감
```

#### 3. 최소 포트 설정
```bash
# 기본값:
#   - Static 포트 할당: 64 포트/VM
#   - Dynamic 포트 할당: 32~65,536 포트/VM (필요 시 자동 증가)
# 실제 필요량에 맞게 조정

gcloud compute routers nats update nat-config \
    --router=nat-router \
    --region=asia-northeast3 \
    --min-ports-per-vm=128  # 인스턴스당 최소 128 포트

# Dynamic 포트 할당 활성화 (권장)
gcloud compute routers nats update nat-config \
    --router=nat-router \
    --region=asia-northeast3 \
    --enable-dynamic-port-allocation \
    --min-ports-per-vm=32 \
    --max-ports-per-vm=65536
```

## 주의사항

### 1. 포트 고갈 (Port Exhaustion) 방지

하나의 NAT IP는 최대 64,512개의 포트를 제공합니다.  

#### 포트 할당 방식

Cloud NAT는 두 가지 포트 할당 방식을 제공합니다:  
1. Static 포트 할당: 각 VM에 고정된 수의 포트 할당 (기본값: 64 포트)  
2. Dynamic 포트 할당: 필요에 따라 포트 자동 증가 (기본값: 32 포트, 최대 65,536 포트)  

#### 포트 계산 예시 (Static 할당)
```
총 사용 가능 포트 = (NAT IP 개수) × 64,512
VM당 최소 포트 = min-ports-per-vm 설정값 (기본 64)

예시:
- NAT IP: 1개 (64,512 포트)
- VM: 10개
- min-ports-per-vm: 64 (기본값)
- VM당 할당 포트: 64개

만약 각 VM이 평균 50개 연결 유지 → OK
만약 각 VM이 평균 100개 연결 유지 → 포트 고갈 위험!
```

권장 사항: 트래픽이 많은 환경에서는 Dynamic 포트 할당을 사용하여 자동으로 포트를 증가시키는 것이 좋습니다.  

#### 해결 방법
```bash
# 추가 고정 IP 예약
gcloud compute addresses create nat-external-ip-2 --region=asia-northeast3

# NAT에 추가 IP 연결
gcloud compute routers nats update nat-config \
    --router=nat-router \
    --region=asia-northeast3 \
    --nat-external-ip-pool=nat-external-ip,nat-external-ip-2
```

### 2. 리전별 리소스

Cloud NAT는 리전별(Regional) 리소스입니다.  

```bash
# 나쁜 예: asia-northeast3에서 생성한 NAT를 us-central1에서 사용 불가
gcloud compute routers nats create nat-config \
    --router=nat-router \
    --region=asia-northeast3  # asia-northeast3에만 적용

# 다른 리전에서 사용하려면 별도 생성 필요
gcloud compute routers create nat-router-us \
    --network=default \
    --region=us-central1

gcloud compute routers nats create nat-config-us \
    --router=nat-router-us \
    --region=us-central1 \
    --nat-external-ip-pool=nat-external-ip-us \
    --nat-all-subnet-ip-ranges
```

### 3. IAP 터널 필수

외부 IP가 없는 인스턴스에 SSH 접속하려면 Identity-Aware Proxy (IAP) 사용이 필수입니다.  

#### IAP 활성화
```bash
# 방화벽 규칙 추가 (IAP 접속 허용)
gcloud compute firewall-rules create allow-ssh-ingress-from-iap \
    --direction=INGRESS \
    --action=allow \
    --rules=tcp:22 \
    --source-ranges=35.235.240.0/20 \
    --network=default

# IAP를 통한 SSH 접속
gcloud compute ssh my-instance \
    --zone=asia-northeast3-a \
    --tunnel-through-iap
```

### 4. API 화이트리스팅 등록

외부 API가 IP 화이트리스팅을 요구하는 경우, Cloud NAT 설정 후 해당 API에 고정 IP를 등록해야 합니다.  

#### API 화이트리스팅 절차
```bash
# 1. 할당된 고정 IP 확인
gcloud compute addresses describe nat-external-ip --region=asia-northeast3  

# 2. 외부 API 관리자 포털에서 IP 등록
# - 해당 서비스 관리 포털 접속
# - 설정 → IP 화이트리스트 → 추가
# - IP 입력: 34.64.123.456
# - 저장

# 3. 테스트
curl https://api.example.com/check
```

### 5. 방화벽 규칙 설정

Cloud NAT를 사용해도 방화벽 규칙은 여전히 중요합니다.  

```bash
# 송신 트래픽 제한 예시 (특정 IP만 허용)
gcloud compute firewall-rules create allow-egress-to-external-api \
    --direction=EGRESS \
    --action=allow \
    --rules=tcp:443 \
    --destination-ranges=203.245.1.0/24 \
    --priority=1000 \
    --network=default

# 기본 송신 트래픽 차단 (보안 강화)
gcloud compute firewall-rules create deny-all-egress \
    --direction=EGRESS \
    --action=deny \
    --rules=all \
    --priority=65534 \
    --network=default
```

## 마무리

Cloud NAT는 동적 인프라 환경에서 필수적인 서비스입니다.  

### 핵심 요약

1. 문제: 동적으로 생성되는 인스턴스의 IP 관리 어려움  
2. 해결: Cloud NAT로 고정 IP 제공  
3. 구성: Cloud Router + NAT Gateway + 고정 IP  
4. 보안: 인스턴스는 외부 IP 없이 안전하게 통신  
5. 비용: 리전, 로깅, 포트 설정 최적화로 절감 가능  

### 적용 시나리오

- 외부 API IP 화이트리스팅 필요 시 (결제, 인증, SMS 등)
- Auto Scaling 환경에서 일관된 송신 IP 필요
- 보안을 위해 VM을 Private Subnet에 배치
- 컴플라이언스 요구사항으로 VM 직접 노출 불가

### 다음 단계

1. Multi-Region 설정: 여러 리전에 Cloud NAT 구축  
2. Private Google Access: Google API를 Private IP로 호출  
3. Cloud Armor: DDoS 방어와 결합  

참고: Cloud NAT는 VPC Peering된 네트워크의 트래픽에는 적용되지 않습니다.  
각 VPC마다 별도의 Cloud NAT를 설정해야 합니다.  

Cloud NAT를 통해 안정적이고 보안성 높은 클라우드 인프라를 구축하시기 바랍니다.  

---

## 참고 자료
- [Cloud NAT 개요 (cloud.google.com)](https://cloud.google.com/nat/docs/overview){:target="_blank"}  
- [Cloud NAT 가격 정책 (cloud.google.com)](https://cloud.google.com/nat/pricing){:target="_blank"}  
- [Cloud NAT 할당량 및 한도 (cloud.google.com)](https://cloud.google.com/nat/quotas){:target="_blank"}  
