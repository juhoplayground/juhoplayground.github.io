---
layout: post
title: AWS 보안 서비스 우선순위별 도입 가이드
author: 'Juho'
date: 2025-11-29 10:00:00 +0900
categories: [AWS]
tags: [AWS, Security, GuardDuty, Config, SecurityHub, IAM, 보안]
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
1. [AWS 보안 서비스 도입이 필요한 이유](#aws-보안-서비스-도입이-필요한-이유)
2. [우선순위 결정 기준](#우선순위-결정-기준)
3. [우선순위별 보안 서비스](#우선순위별-보안-서비스)
4. [1순위: GuardDuty - 위협 탐지의 핵심](#1순위-guardduty---위협-탐지의-핵심)
5. [2순위: IAM Access Analyzer - 권한 분석](#2순위-iam-access-analyzer---권한-분석)
6. [3순위: AWS Config - 변경 추적과 규정 준수](#3순위-aws-config---변경-추적과-규정-준수)
7. [4순위: VPC Flow Logs - 네트워크 분석](#4순위-vpc-flow-logs---네트워크-분석)
8. [5순위: Security Hub - 통합 보안 관리](#5순위-security-hub---통합-보안-관리)
9. [6순위: Inspector - 취약점 스캔](#6순위-inspector---취약점-스캔)
10. [7순위: AWS Backup - 데이터 복구](#7순위-aws-backup---데이터-복구)
11. [8순위: AWS WAF - 웹 애플리케이션 방화벽](#8순위-aws-waf---웹-애플리케이션-방화벽)
12. [9순위: Macie - 민감 정보 탐지](#9순위-macie---민감-정보-탐지)
13. [단계별 도입 로드맵](#단계별-도입-로드맵)
14. [비용 최적화 팁](#비용-최적화-팁)
15. [마무리](#마무리)


## AWS 보안 서비스 도입이 필요한 이유
AWS에는 수십 가지의 보안 관련 서비스가 있다.  
하지만 모든 서비스를 한 번에 도입하는 것은 비용과 운영 부담이 크기 때문에
우선순위를 정해서 단계적으로 도입하는 것이 중요하다.  

이 글에서는 실무 경험을 바탕으로 스타트업과 중소기업 관점에서 어떤 보안 서비스를 먼저 도입해야 하는지  
그리고 각 서비스의 실질적인 도입 방법과 비용을 정리했다.  


## 우선순위 결정 기준
보안 서비스의 우선순위는 다음 기준으로 결정했다:  

1. 위협 탐지 및 예방 효과: 보안 사고를 사전에 방지하거나 빠르게 탐지할 수 있는가?  
2. 비용 대비 효과: 투자 대비 실질적인 보안 개선 효과가 큰가?  
3. 운영 부담: 설정과 운영이 간단한가?  
4. 규정 준수: 컴플라이언스와 감사 요구사항을 충족하는가?  
5. 즉시 적용 가능성: 기존 인프라 변경 없이 바로 도입할 수 있는가?  


## 우선순위별 보안 서비스

| 우선순위 | 서비스                 | 이유                                    |
|------|---------------------|---------------------------------------|
| 1    | GuardDuty           | 이상 행동 탐지 · 사고 예방 효과 매우 큼              |
| 2    | IAM Access Analyzer | Public 노출 · 권한 오남용 방지                 |
| 3    | AWS Config          | 변경 추적 · 오류 대응 · 규정 준수                 |
| 4    | VPC Flow Logs       | 네트워크 분석 기본 데이터                        |
| 5    | Security Hub        | 여러 보안 결과를 통합 관리                       |
| 6    | Inspector           | EC2/ECR 취약점 스캔                        |
| 7    | AWS Backup          | 백업 자동화로 운영 안정성 확보                     |
| 8    | AWS WAF             | 웹 서비스 공격 방어                           |
| 9    | Macie               | S3 민감 정보 탐지 · 개인정보 처리 시 필수           |


## 1순위: GuardDuty - 위협 탐지의 핵심

### 왜 1순위인가?
GuardDuty는 AWS 계정 전체의 이상 행동을 자동으로 탐지하는 서비스다.
- 설정이 매우 간단 (클릭 몇 번으로 활성화)
- 기존 인프라 변경 불필요
- 실시간 위협 탐지 (예: 비정상적인 API 호출, 악성 IP 접근 등)
- 비용 대비 효과가 가장 큼

### 주요 탐지 항목
- 비정상적인 API 호출: 알 수 없는 지역에서의 API 호출, 비정상적인 권한 변경
- 암호화폐 채굴 공격: EC2 인스턴스가 악의적인 채굴 풀에 연결
- 데이터 유출 시도: 대량 데이터 전송, 비정상적인 DNS 쿼리
- 루트 계정 사용: 루트 계정 사용 시 알림

### 도입 방법
```bash
# AWS CLI로 활성화
aws guardduty create-detector --enable

# 특정 리전에서 활성화
aws guardduty create-detector --enable --region ap-northeast-2
```

콘솔 설정 방법:
1. AWS Console > GuardDuty > "Get Started" 클릭
2. "Enable GuardDuty" 클릭
3. 모든 리전에서 활성화 권장

### 비용
- CloudTrail 로그 분석: $4.64/1백만 이벤트
- VPC Flow Logs 분석: $1.18/GB
- DNS 로그 분석: $0.43/1백만 DNS 쿼리

예상 비용 예시:
- 소규모 스타트업 (EC2 10대): 월 $20~50
- 중소기업 (EC2 50대): 월 $100~200

### 알림 설정 (EventBridge + SNS)
```python
# GuardDuty 결과를 Slack으로 전송하는 Lambda 함수 예시
import json
import urllib3

http = urllib3.PoolManager()

def lambda_handler(event, context):
    detail = event['detail']
    severity = detail['severity']
    finding_type = detail['type']

    message = {
        'text': f"⚠️ GuardDuty Alert - Severity: {severity}\n{finding_type}"
    }

    webhook_url = "YOUR_SLACK_WEBHOOK_URL"
    http.request('POST', webhook_url,
                 body=json.dumps(message),
                 headers={'Content-Type': 'application/json'})

    return {'statusCode': 200}
```

### 실전 팁
- 높은 심각도만 알림 받기: Severity 7 이상만 알림 설정
- False Positive 처리: Suppression Rules로 오탐 필터링
- Multi-Region: 모든 리전에서 활성화 필수


## 2순위: IAM Access Analyzer - 권한 분석

### 왜 2순위인가?
IAM Access Analyzer는 외부에 공개된 리소스를 자동으로 탐지한다.
- 무료 서비스
- S3, IAM Role, Lambda 등 Public 노출 탐지
- 최소 권한 원칙 준수 지원

### 주요 탐지 항목
- Public S3 버킷: 외부에서 접근 가능한 S3 버킷
- 외부 계정 접근 가능 IAM Role: Cross-Account 접근 설정
- Public Lambda 함수: 외부에서 호출 가능한 Lambda
- Public Secrets Manager: 외부 노출된 Secrets

### 도입 방법
```bash
# AWS CLI로 활성화
aws accessanalyzer create-analyzer \
    --analyzer-name my-account-analyzer \
    --type ACCOUNT
```

콘솔 설정 방법:
1. AWS Console > IAM > Access Analyzer
2. "Create analyzer" 클릭
3. Analyzer type: "Account" 선택

### 최소 권한 정책 생성
IAM Access Analyzer는 실제 사용한 권한만 포함된 정책을 자동 생성해준다.

```bash
# CloudTrail 로그 기반으로 정책 생성
aws accessanalyzer create-access-preview \
    --analyzer-arn arn:aws:access-analyzer:ap-northeast-2:123456789012:analyzer/my-analyzer \
    --configurations file://config.json
```

### 비용
- 무료

### 실전 팁
- 정기적인 리뷰: 주 1회 External Access Findings 확인
- CI/CD 통합: 배포 전에 Public 노출 여부 자동 검증
- 조직 레벨 분석: AWS Organizations와 통합하여 전체 계정 분석


## 3순위: AWS Config - 변경 추적과 규정 준수

### 왜 3순위인가?
AWS Config는 리소스 변경 이력을 기록하고 규정 준수 여부를 자동 검증한다.
- 누가, 언제, 무엇을 변경했는지 추적
- 보안 사고 원인 분석에 필수
- 컴플라이언스 감사 대응

### 주요 기능
- Configuration History: 리소스 변경 이력 기록
- Config Rules: 보안 정책 자동 검증 (예: S3 암호화 필수)
- Conformance Packs: 규정 준수 템플릿 (CIS, PCI-DSS 등)

### 도입 방법
```bash
# AWS CLI로 활성화
aws configservice put-configuration-recorder \
    --configuration-recorder name=default,roleARN=arn:aws:iam::123456789012:role/config-role

aws configservice put-delivery-channel \
    --delivery-channel name=default,s3BucketName=my-config-bucket
```

필수 Config Rules:
```yaml
# S3 버킷 암호화 필수
- rule_name: s3-bucket-server-side-encryption-enabled
  description: S3 버킷이 암호화되어 있는지 확인

# RDS 퍼블릭 액세스 차단
- rule_name: rds-instance-public-access-check
  description: RDS가 Public 서브넷에 있지 않은지 확인

# IAM 루트 계정 MFA 활성화
- rule_name: root-account-mfa-enabled
  description: 루트 계정에 MFA가 활성화되어 있는지 확인
```

### 자동 복구 (Remediation)
```python
# Config Rule 위반 시 자동으로 S3 암호화 활성화
import boto3

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    config = boto3.client('config')

    # Config에서 위반된 리소스 확인
    non_compliant_resources = event['configRuleNames']

    for resource in non_compliant_resources:
        bucket_name = resource['resourceId']

        # S3 암호화 활성화
        s3.put_bucket_encryption(
            Bucket=bucket_name,
            ServerSideEncryptionConfiguration={
                'Rules': [{'ApplyServerSideEncryptionByDefault': {'SSEAlgorithm': 'AES256'}}]
            }
        )
```

### 비용
- Configuration Items: $0.003/항목
- Rule Evaluations: $0.001/평가

예상 비용 예시:
- 소규모 (100개 리소스, 10개 규칙): 월 $30~50
- 중소기업 (500개 리소스, 20개 규칙): 월 $150~250

### 실전 팁
- 중요 리소스만 기록: 모든 리소스를 기록하면 비용 증가
- 자동 복구 활용: Lambda와 연동하여 위반 사항 자동 수정
- Conformance Packs 사용: CIS AWS Foundations Benchmark 템플릿 추천


## 4순위: VPC Flow Logs - 네트워크 분석

### 왜 4순위인가?
VPC Flow Logs는 네트워크 트래픽을 기록하여 보안 사고 분석에 필수적이다.
- 비정상적인 네트워크 접근 탐지
- DDoS 공격 분석
- 네트워크 병목 지점 파악

### 도입 방법
```bash
# VPC Flow Logs 활성화 (S3 저장)
aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids vpc-12345678 \
    --traffic-type ALL \
    --log-destination-type s3 \
    --log-destination arn:aws:s3:::my-flow-logs-bucket
```

### 비용 최적화
```bash
# CloudWatch Logs 대신 S3 저장 (비용 절감)
# CloudWatch: $0.50/GB
# S3: $0.023/GB (약 20배 저렴)
```

### Athena로 분석
```sql
-- 특정 IP에서의 접속 시도 분석
SELECT srcaddr, dstaddr, dstport, action, COUNT(*) as count
FROM vpc_flow_logs
WHERE dstport = 22 AND action = 'REJECT'
GROUP BY srcaddr, dstaddr, dstport, action
ORDER BY count DESC
LIMIT 20;
```

### 비용
- CloudWatch Logs: $0.50/GB
- S3: $0.023/GB (권장)

예상 비용 예시:
- 소규모 (50GB/월): 월 $1~2 (S3 저장 시)
- 중소기업 (200GB/월): 월 $5~10 (S3 저장 시)

### 실전 팁
- S3 저장 필수: CloudWatch보다 20배 저렴
- Athena 쿼리 활용: 로그 분석 자동화
- Lifecycle Policy: 90일 이후 S3 Glacier로 이동


## 5순위: Security Hub - 통합 보안 관리

### 왜 5순위인가?
Security Hub는 여러 보안 서비스의 결과를 한곳에 통합한다.
- GuardDuty, Config, Inspector 등 결과 통합
- AWS Security Best Practices 자동 검증
- 대시보드로 전체 보안 상태 확인

### 도입 방법
```bash
# Security Hub 활성화
aws securityhub enable-security-hub

# CIS AWS Foundations Benchmark 활성화
aws securityhub batch-enable-standards \
    --standards-subscription-requests StandardsArn="arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.2.0"
```

### 주요 표준
- CIS AWS Foundations Benchmark: AWS 보안 모범 사례
- PCI DSS: 결제 카드 산업 보안 표준
- AWS Foundational Security Best Practices: AWS 권장 설정

### 비용
- Security Checks: $0.0010/체크
- Finding Ingestion: $0.00003/건

예상 비용 예시:
- 소규모 (1,000개 체크/일): 월 $30~50
- 중소기업 (5,000개 체크/일): 월 $150~250

### 실전 팁
- 중요도 필터링: CRITICAL, HIGH만 우선 처리
- 자동화: EventBridge로 Slack/Email 알림
- Organization 통합: 멀티 계정 환경에서 중앙 관리


## 6순위: Inspector - 취약점 스캔

### 왜 6순위인가?
Inspector는 EC2/ECR의 취약점을 자동 스캔한다.
- CVE 취약점 자동 탐지
- 패키지 버전 관리 부담 감소
- 컨테이너 이미지 보안 검증

### 도입 방법
```bash
# Inspector 활성화
aws inspector2 enable \
    --resource-types EC2 ECR
```

### 취약점 우선순위 처리
```python
# 높은 심각도 취약점만 Slack 알림
import boto3
import json
import urllib3

def lambda_handler(event, context):
    detail = event['detail']
    severity = detail['severity']

    if severity in ['CRITICAL', 'HIGH']:
        # Slack 알림
        message = {
            'text': f"⚠️ Critical CVE Found: {detail['title']}"
        }
        # Webhook 전송 로직
```

### 비용
- EC2 스캔: $0.09/인스턴스/월
- ECR 스캔: $0.09/이미지 재스캔

예상 비용 예시:
- 소규모 (EC2 10대): 월 $0.90
- 중소기업 (EC2 50대, ECR 100개): 월 $13.50

### 실전 팁
- 자동 스캔 주기: 매일 자동 스캔 설정
- CI/CD 통합: ECR 푸시 시 자동 스캔
- 억제 규칙: False Positive 필터링


## 7순위: AWS Backup - 데이터 복구

### 왜 7순위인가?
AWS Backup은 백업 자동화와 복구 전략을 제공한다.
- 랜섬웨어 공격 대응
- 실수로 인한 데이터 삭제 복구
- 재해 복구 (DR)

### 도입 방법
```bash
# Backup Plan 생성
aws backup create-backup-plan \
    --backup-plan file://backup-plan.json

# 백업 정책 예시 (backup-plan.json)
{
  "BackupPlanName": "DailyBackup",
  "Rules": [{
    "RuleName": "DailyBackups",
    "TargetBackupVaultName": "Default",
    "ScheduleExpression": "cron(0 2 * * ? *)",
    "Lifecycle": {
      "DeleteAfterDays": 30
    }
  }]
}
```

### 비용
- 백업 스토리지: $0.05/GB/월
- 복원: $0.02/GB

예상 비용 예시:
- 소규모 (100GB 백업): 월 $5
- 중소기업 (1TB 백업): 월 $50

### 실전 팁
- 태그 기반 백업: 중요 리소스만 백업
- Cross-Region 백업: DR 대비
- Backup Vault Lock: 랜섬웨어 방어


## 8순위: AWS WAF - 웹 애플리케이션 방화벽

### 왜 8순위인가?
AWS WAF는 웹 애플리케이션 공격을 방어한다.
- SQL Injection, XSS 차단
- DDoS 방어
- Rate Limiting

### 도입 방법
```bash
# WAF Web ACL 생성
aws wafv2 create-web-acl \
    --name MyWebACL \
    --scope REGIONAL \
    --default-action Block={} \
    --rules file://waf-rules.json
```

### Managed Rules (추천)
```json
{
  "Name": "AWSManagedRulesCommonRuleSet",
  "Priority": 1,
  "Statement": {
    "ManagedRuleGroupStatement": {
      "VendorName": "AWS",
      "Name": "AWSManagedRulesCommonRuleSet"
    }
  }
}
```

### 비용
- Web ACL: $5/월
- Rule: $1/규칙/월
- Request: $0.60/100만 요청

예상 비용 예시:
- 소규모 (10M 요청/월): 월 $10~15
- 중소기업 (100M 요청/월): 월 $65~75

### 실전 팁
- Managed Rules 우선 사용: AWS 관리형 규칙이 효율적
- Rate Limiting: 1분당 2,000 요청 제한 추천
- Geo Blocking: 불필요한 국가 차단


## 9순위: Macie - 민감 정보 탐지

### 왜 9순위인가?
Macie는 S3의 민감 정보를 자동 탐지한다.
- 개인정보 (주민등록번호, 신용카드 등)
- 규정 준수 (GDPR, PIPA 등)
- 데이터 유출 방지

### 도입 방법
```bash
# Macie 활성화
aws macie2 enable-macie

# S3 버킷 스캔 작업 생성
aws macie2 create-classification-job \
    --s3-job-definition '{
      "bucketDefinitions": [{
        "accountId": "123456789012",
        "buckets": ["my-sensitive-bucket"]
      }]
    }' \
    --job-type ONE_TIME \
    --name MySensitiveDataScan
```

### 비용
- 계정 평가: $0.10/월
- 객체 스캔: $1.00/GB

예상 비용 예시:
- 소규모 (100GB): 월 $100
- 중소기업 (1TB): 월 $1,000

### 실전 팁
- 선별적 스캔: 민감 데이터가 있는 버킷만 스캔
- 자동화: 새 파일 업로드 시 자동 스캔
- 알림 설정: 민감 정보 탐지 시 즉시 알림


## 단계별 도입 로드맵

### 1단계: 즉시 도입 (0~1개월)
```
✅ GuardDuty 활성화 (모든 리전)
✅ IAM Access Analyzer 활성화
✅ VPC Flow Logs 활성화 (S3 저장)
```
예상 비용: 월 $30~80

### 2단계: 기본 보안 강화 (1~3개월)
```
✅ AWS Config 활성화 + 필수 Config Rules 설정
✅ Security Hub 활성화 + CIS Benchmark 적용
✅ Inspector 활성화 (EC2/ECR)
```
예상 비용: 월 $150~350 (1단계 포함)

### 3단계: 고급 보안 (3~6개월)
```
✅ AWS Backup 자동화
✅ AWS WAF 적용 (웹 서비스)
✅ Macie 스캔 (민감 정보 처리 시)
```
예상 비용: 월 $250~600 (전단계 포함)


## 비용 최적화 팁

### 1. GuardDuty
- 멀티 계정: Organization Master 계정에서 통합 관리로 비용 절감
- S3 Protection: 필요한 버킷만 활성화

### 2. Config
- 선별적 리소스 기록: 중요 리소스만 기록 (EC2, RDS, S3)
- Rule 최적화: 꼭 필요한 Rule만 활성화

### 3. VPC Flow Logs
- S3 저장 필수: CloudWatch 대비 20배 저렴
- Lifecycle Policy: 90일 후 Glacier로 이동

### 4. Security Hub
- 통합 비활성화: 사용하지 않는 통합 서비스 비활성화
- Organization 통합: 멀티 계정 환경에서 중앙 관리

### 5. Inspector
- 스캔 주기 조정: 매일 대신 주 1회로 조정
- 불필요한 이미지 삭제: ECR 오래된 이미지 자동 삭제

### 6. Macie
- 선별적 스캔: 전체 S3 대신 민감 데이터 버킷만 스캔
- 스캔 주기: 월 1회로 제한


## 마무리
AWS 보안 서비스는 많지만 단계적으로 도입하는 것이 핵심이다.

즉시 시작할 수 있는 3가지:
1. GuardDuty 활성화 (클릭 몇 번)
2. IAM Access Analyzer 활성화 (무료)
3. VPC Flow Logs 활성화 (S3 저장)

이 3가지만 활성화해도 월 $30~50 정도의 비용으로 핵심 보안을 크게 강화할 수 있다.  

보안은 한 번에 완벽하게 하는 것이 아니라 지속적으로 개선하는 것이다.  
이 글이 AWS 보안 서비스 도입에 도움이 되길 바란다.  
