---
layout: post
title: AWS 보안 사고 대응 및 개선기
author: 'Juho'
date: 2025-11-26 09:00:00 +0900
categories: [AWS]
tags: [AWS, IAM, Security, MFA, AccessKey, 보안사고]
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
1. [사건 개요](#사건-개요)
2. [취약점 분석](#취약점-분석)
3. [즉각적인 대응 조치](#즉각적인-대응-조치)
4. [보안 강화 조치](#보안-강화-조치)
5. [IAM 보안 베스트 프랙티스](#iam-보안-베스트-프랙티스)
6. [EC2 보안 강화](#ec2-보안-강화)
7. [모니터링 및 알림 설정](#모니터링-및-알림-설정)
8. [체크리스트](#체크리스트)
9. [교훈과 회고](#교훈과-회고)


## 사건 개요
최근 회사에서 AWS 계정이 해킹당하는 보안 사고가 발생했다.  
부끄럽지만 이 경험을 공유해서 다른 분들이 같은 실수를 반복하지 않기를 바라며 글을 작성한다.  

**사고 원인 요약**
- 사용하지 않는 IAM 사용자의 Access Key가 유출됨  
- 해당 사용자에게 2단계 MFA 인증이 설정되어 있지 않았음  
- 해당 Access Key가 모든 권한(AdministratorAccess)을 보유하고 있었음  
  
공격자는 유출된 Access Key를 이용해 AWS 리소스에 접근할 수 있었고  
다행히 빠른 탐지로 큰 피해는 막을 수 있었지만 이 사건을 계기로 전체 보안 체계를 재검토하게 되었다.  


## 취약점 분석

### 1. 사용하지 않는 IAM 사용자 방치
```
문제: 퇴사자 또는 프로젝트 종료 후 IAM 사용자를 삭제하지 않음
위험: 방치된 계정은 공격자의 진입점이 될 수 있음
```

### 2. MFA 미적용
```
문제: IAM 사용자에게 MFA(Multi-Factor Authentication) 미설정
위험: Access Key 유출 시 추가 인증 없이 즉시 접근 가능
```

### 3. 과도한 권한 부여
```
문제: 개발/테스트 용도로 AdministratorAccess 정책 부여
위험: 최소 권한 원칙 위반, 전체 AWS 리소스 접근 가능
```

### 4. Access Key 관리 부실
```
문제: Access Key 로테이션 미실시, 장기간 동일 키 사용
위험: 키 유출 가능성 증가, 유출 시 탐지 어려움
```


## 즉각적인 대응 조치

사고 인지 후 다음과 같은 순서로 즉각 대응했다.

### 1단계: 유출된 Access Key 즉시 비활성화
```bash
# Access Key 비활성화
aws iam update-access-key \
  --user-name compromised-user \
  --access-key-id AKIAXXXXXXXXXXXXXXXX \
  --status Inactive

# Access Key 삭제
aws iam delete-access-key \
  --user-name compromised-user \
  --access-key-id AKIAXXXXXXXXXXXXXXXX
```

### 2단계: 의심스러운 리소스 확인 및 삭제
```bash
# 새로 생성된 EC2 인스턴스 확인
aws ec2 describe-instances \
  --filters "Name=launch-time,Values=2025-11-*" \
  --query 'Reservations[*].Instances[*].[InstanceId,LaunchTime,InstanceType]'

# 새로 생성된 IAM 사용자 확인
aws iam list-users --query 'Users[?CreateDate>=`2025-11-01`]'
```

### 3단계: 사용하지 않는 IAM 사용자 삭제
```bash
# 사용자의 모든 Access Key 삭제
aws iam list-access-keys --user-name unused-user
aws iam delete-access-key --user-name unused-user --access-key-id AKIAXXXXXXXXXXXXXXXX

# 사용자의 정책 분리
aws iam list-attached-user-policies --user-name unused-user
aws iam detach-user-policy --user-name unused-user --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# 사용자 삭제
aws iam delete-user --user-name unused-user
```

### 4단계: 피해 범위 확인
```bash
# CloudTrail에서 해당 Access Key의 활동 로그 확인
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAXXXXXXXXXXXXXXXX \
  --start-time 2025-11-01T00:00:00Z \
  --end-time 2025-11-24T23:59:59Z
```


## 보안 강화 조치

### 1. MFA 강제 적용

모든 IAM 사용자에게 MFA를 강제하는 정책을 적용했다.


### 2. 최소 권한 원칙 적용

각 사용자/역할에 필요한 최소한의 권한만 부여하도록 정책을 재설계했다.

**Before (위험)**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

**After (개선)**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2ReadOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:Get*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3SpecificBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-app-bucket",
        "arn:aws:s3:::my-app-bucket/*"
      ]
    }
  ]
}
```


### 3. Access Key 관리 정책 수립

```
1) Access Key 생성 최소화
   - 가능하면 IAM Role 사용
   - CLI 작업이 필요한 경우에만 Access Key 발급

2) 정기적인 Key 로테이션
   - 90일마다 Access Key 로테이션
   - 오래된 Key 자동 비활성화

3) 미사용 Key 정리
   - 30일 이상 미사용 Key 알림
   - 90일 이상 미사용 Key 자동 비활성화
```

Access Key 사용 현황 확인 스크립트
```bash
# 모든 사용자의 Access Key 마지막 사용 시간 확인
aws iam generate-credential-report
aws iam get-credential-report --query 'Content' --output text | base64 -d | cut -d',' -f1,9,11,14,16
```


### 4. AWS Secrets Manager 관리

Access Key나 DB 비밀번호를 코드에 하드코딩하거나 환경변수로 관리하는 대신 AWS Secrets Manager를 사용하여 안전하게 관리한다.  
사용하지 않는 Secrets Manager 제거, 노출되었을 것 같은 Secrets Manager 수정했다.  

```bash
# Secret 생성
aws secretsmanager create-secret \
  --name prod/myapp/db \
  --description "Production database credentials" \
  --secret-string '{"username":"admin","password":"StrongPassword123!"}'

# Secret 값 변경 (로테이션)
aws secretsmanager update-secret \
  --secret-id prod/myapp/db \
  --secret-string '{"username":"admin","password":"NewStrongPassword456!"}'

# Secret 값 조회
aws secretsmanager get-secret-value \
  --secret-id prod/myapp/db \
  --query SecretString \
  --output text
```

사용하지 않는 Secret 정리
```bash
# 모든 Secret 목록 확인
aws secretsmanager list-secrets

# 사용하지 않는 Secret 삭제 예약 (복구 가능 기간 7일)
aws secretsmanager delete-secret \
  --secret-id unused/secret/name \
  --recovery-window-in-days 7

# 즉시 삭제 (복구 불가능)
aws secretsmanager delete-secret \
  --secret-id unused/secret/name \
  --force-delete-without-recovery

# 삭제 예약된 Secret 목록 확인
aws secretsmanager list-secrets \
  --filters Key=all,Values=deleted
```

## IAM 보안 베스트 프랙티스

| 항목 | 권장사항 | 구현 방법 |
|------|----------|-----------|
| Root 계정 | 일상 작업에 사용 금지 | MFA 설정, Access Key 삭제 |
| MFA | 모든 사용자 필수 | 강제 MFA 정책 적용 |
| Access Key | 최소화 | IAM Role 우선 사용 |
| 권한 | 최소 권한 원칙 | 필요한 권한만 부여 |
| 비밀번호 | 강력한 정책 | 복잡도, 만료 기간 설정 |
| 감사 | 정기 검토 | IAM Access Analyzer 활용 |


### IAM Access Analyzer 활용

AWS IAM Access Analyzer를 사용하면 외부에 공개된 리소스나 과도한 권한을 쉽게 찾을 수 있다.

```bash
# Access Analyzer 생성
aws accessanalyzer create-analyzer \
  --analyzer-name my-analyzer \
  --type ACCOUNT

# 분석 결과 확인
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:ap-northeast-2:123456789012:analyzer/my-analyzer
```


## EC2 보안 강화

IAM 보안과 함께 EC2 인프라 보안도 강화했다.

### 1. IAM Role을 EC2에 부여

EC2 인스턴스에서 AWS 서비스에 접근할 때는 Access Key 대신 IAM Role을 사용한다.

```bash
# Instance Profile 생성 및 EC2에 연결
aws iam create-instance-profile --instance-profile-name MyAppProfile
aws iam add-role-to-instance-profile \
  --instance-profile-name MyAppProfile \
  --role-name MyAppRole

# EC2 인스턴스에 연결
aws ec2 associate-iam-instance-profile \
  --instance-id i-1234567890abcdef0 \
  --iam-instance-profile Name=MyAppProfile
```


### 2. 보안 그룹 재검토

```
Before (위험)
- Inbound: 0.0.0.0/0 → 22 (SSH)
- Inbound: 0.0.0.0/0 → 3389 (RDP)

After (개선)
- Inbound: 회사 VPN IP/32 → 22 (SSH)
- Inbound: Bastion SG → 22 (SSH)
- 또는 SSH 포트 완전 폐쇄 후 SSM Session Manager 사용
```

보안 그룹 점검 스크립트
```bash
# 0.0.0.0/0으로 열린 포트 확인
aws ec2 describe-security-groups \
  --query 'SecurityGroups[*].{Name:GroupName,Rules:IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`]]}' \
  --output table
```


### 3. Private Subnet 활용

```
Internet Gateway
       │
[Public Subnet]
├── NAT Gateway
├── ALB (Application Load Balancer)
└── Bastion Host (필요시)
       │
[Private Subnet]
├── Application Servers (EC2)
├── Database (RDS)
└── Internal Services
```

애플리케이션 서버는 Private Subnet에 배치하고 외부 접근은 ALB를 통해서만 허용한다.


### 4. VPC Endpoint 활용

Private Subnet의 EC2가 AWS 서비스에 접근할 때 NAT Gateway 대신 VPC Endpoint를 사용하면 더 안전하고 비용도 절감할 수 있다.

```bash
# S3 Gateway Endpoint 생성
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.ap-northeast-2.s3 \
  --route-table-ids rtb-12345678

# SSM Interface Endpoint 생성
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.ap-northeast-2.ssm \
  --subnet-ids subnet-12345678 \
  --security-group-ids sg-12345678
```


## 모니터링 및 알림 설정

### 1. CloudTrail 설정

모든 API 호출을 기록하도록 CloudTrail을 설정한다.

```bash
aws cloudtrail create-trail \
  --name my-security-trail \
  --s3-bucket-name my-cloudtrail-logs \
  --is-multi-region-trail \
  --enable-log-file-validation
```


### 2. CloudWatch 알림 설정

의심스러운 활동에 대한 알림을 설정한다.

```json
{
  "source": ["aws.iam"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["iam.amazonaws.com"],
    "eventName": [
      "CreateUser",
      "CreateAccessKey",
      "AttachUserPolicy",
      "AttachRolePolicy"
    ]
  }
}
```

## 체크리스트

### IAM 보안 체크리스트
- [ ] Root 계정 MFA 설정 완료
- [ ] Root 계정 Access Key 삭제
- [ ] 모든 IAM 사용자 MFA 필수 정책 적용
- [ ] 불필요한 IAM 사용자 삭제
- [ ] 최소 권한 원칙에 따른 정책 재설계
- [ ] Access Key 로테이션 정책 수립
- [ ] IAM Access Analyzer 활성화

### EC2 보안 체크리스트
- [ ] EC2에 IAM Role 사용 (Access Key 대신)
- [ ] 보안 그룹 최소 포트만 허용
- [ ] SSH/RDP 포트 제한 (특정 IP만 허용)
- [ ] Private Subnet에 애플리케이션 배치
- [ ] VPC Endpoint 활용

### 모니터링 체크리스트
- [ ] CloudTrail 전 리전 활성화
- [ ] CloudWatch 알림 설정
- [ ] AWS Config 규칙 활성화
- [ ] GuardDuty 활성화


## 교훈과 회고

이번 사고를 통해 얻은 교훈들이다.

### 1. "나중에"는 결국 오지 않는다
```
"나중에 정리해야지"라고 미뤘던 미사용 IAM 계정이 결국 보안 사고의 원인이 되었다.
보안 점검은 미루지 말고 정기적으로 수행해야 한다.
```

### 2. 편의성 vs 보안
```
"개발하기 편하게" AdministratorAccess를 부여했던 것이 큰 위험이었다.
초기 설정이 조금 불편하더라도 최소 권한 원칙을 지켜야 한다.
```

### 3. MFA는 선택이 아닌 필수
```
MFA 하나만 설정되어 있었어도 이번 사고는 막을 수 있었다.
모든 IAM 사용자에게 MFA를 강제하는 것이 기본이다.
```

### 4. 자동화의 중요성
```
수동으로 보안 점검을 하면 누락이 발생하기 쉽다.
AWS Config, Access Analyzer 등 자동화 도구를 적극 활용해야 한다.
```

### 5. 사고 대응 체계 구축
```
사고가 발생했을 때 당황하지 않고 대응할 수 있는 체계가 필요하다.
사전에 대응 절차를 문서화하고 훈련해야 한다.
```

---

AWS 보안 사고는 누구에게나 일어날 수 있다.  
중요한 것은 사고를 통해 배우고 더 강력한 보안 체계를 구축하는 것이다.  
이 글이 비슷한 상황을 겪고 있거나 예방하고 싶은 분들에게 도움이 되었으면 한다.  

보안은 한 번 설정하고 끝나는 것이 아니라 지속적으로 관리하고 개선해야 하는 영역이다.  
정기적인 보안 점검과 최신 보안 모범 사례 학습을 꾸준히 해야겠다.  

