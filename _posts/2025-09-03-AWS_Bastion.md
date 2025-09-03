---
layout: post
title: AWS Bastion
author: 'Juho'
date: 2025-09-03 09:00:00 +0900
categories: [AWS]
tags: [AWS, Bastion, SSM, EIC]
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
1. [AWS Bastion을 공부하게 된 이유](#blue-ocean)  
2. [Pipeline: Stage View](#pipeline-stage-view)
3. [Pipeline Graph View](#pipeline-graph-view)



## AWS Bastion을 공부하게 된 이유
새로 이직한 회사에서 AWS Bastion을 사용하고 있었다.  
처음 들어보는 AWS 서비스 이름이라 퇴근 이후에 확인해본 내용을 정리하려고 한다.  


## AWS Bastion이란?
AWS Bastion은 프라이빗 서브넷에 있는 서버에 안전하게 접속하기 위한 점프 호스트(점프 서버, 점프 박스) 개념이다.  
일종의 보안 게이트웨이 역할을 하며 외부에서 내부 리소스로의 직접 접근을 차단하면서도  
관리자가 필요할 때 안전하게 접근할 수 있는 통로를 제공한다.  
  
## AWS Bastion 주요 목적과 기능, 장점
1) 보안 강화 : 프라이빗 인스턴스에 퍼블릭 IP를 할당하지 않아도 관리 접속 가능  
2) 접근 제어 : 운영자가 프라이빗 서버를 관리할 수 있는 유일한 진입점 생성  
3) 감사 추적 : 모든 관리 접속을 중앙 집중화하여 로그를 한곳에서 수집, 분석  
  
주의 사항 : 바스티온 자체가 공격 표면이 되어 패치, 키 관리, 로깅 등의 운영 부담이 발생할 수 있음  


## AWS Bastion 아키텍처  
```test
Internet
│
[Public Subnet]
└─ Bastion (EC2 / EIC Endpoint)
│ (SG: 회사 고정 IP만 22/3389)
▼
[Private Subnet]
└─ App/DB EC2 (SG: Bastion SG만 허용)
│
(NAT Gateway: 프라이빗 아웃바운드)
```
퍼블릭 서브넷: 바스티온 호스트 위치  
프라이빗 서브넷: 업무용 애플리케이션/DB 서버  
보안 그룹: 바스티온은 신뢰 IP만 SSH/RDP 허용, 내부 서버는 바스티온 SG만 허용  
NAT 게이트웨이: 프라이빗의 아웃바운드 인터넷 업데이트/패키지 설치 등  

## AWS Batrion 보안 그룹 및 접근 제어 예시
Bastion SG (inbound)  
- TCP 22: 회사 고정 IP/CIDR만 허용  
- TCP 3389 ← 회사 고정 IP만 허용 (Windows RDP용)  
  
App/DB SG (inbound)  
- TCP 22: Bastion SG에서만 허용(소스: Bastion SG)  
- TCP 3389 ← Bastion 보안 그룹만 허용
  
공통 권장  
- outbound는 기본 허용 대신 필요 포트만 세분화  
- IAM은 최소권한(+MFA), 역할(Instance Profile) 사용

## AWS Batrion  EC2 구축 체크리스트
1) 인프라 설정  
 [ ] 퍼블릭 서브넷에 소형 EC2 인스턴스 생성  
 [ ] 탄력적 IP 설정 (필요시)  
 [ ] 보안 그룹 제한적 설정  
  
2) 보안 설정  
 [ ] SSM 에이전트 설치/활성화  
 [] VPC 엔드포인트 구성 (com.amazonaws.*.ssm, ec2messages, ssm)  
 [ ] CloudWatch Logs로 접속 로그 수집  
 [ ] IAM 역할 최소권한 설정  
 [ ] 포트 22는 회사 고정 IP만 허용  
  
3) 운영 설정  
[ ] 프라이빗 인스턴스 SSH inbound를 Bastion SG만 허용  
[ ] 정기 패치 스케줄 설정  
[ ] 키 로테이션 정책 수립  
[ ] 취약점 스캔 체계 구축  


## AWS Bastion 대안  
  
| 항목          | 전통 바스티온 EC2         | SSM Session Manager | EC2 Instance Connect Endpoint(EICE) |  
| ----------- | ------------------- | ------------------- | ----------------------------------- |  
| 접속 방식       | SSH/RDP(고정키)        | 브라우저/CLI 세션(에이전트)   | 임시 SSH 키 + 사설 터널                    |  
| 인바운드 포트     | 필요(22/3389)         | 불필요                 | 불필요(엔드포인트 내부)                       |  
| 퍼블릭 IP      | 보통 필요               | 불필요                 | 불필요                                 |  
| 키 관리        | 필요(배포·로테이션)         | 불필요                 | 임시 키 자동                             |  
| 로깅          | 직접 구성(CloudWatch 등) | 자동 세션 로깅/감사 용이      | CloudWatch/CloudTrail 연계            |  
| 네트워크 제어     | SG/ACL로 제어          | IAM+KMS+로그로 세밀      | SG+IAM로 세밀                          |  
| 러닝코스트/운영 부담 | 패치·하드닝·키 관리 필요      | 상대적 낮음              | 낮음(엔드포인트 운영만)                       |  
| 적합 사례       | 레거시 SSH/RDP 필수      | 보안·감사 엄격, 포트리스 선호   | SSH 워크플로우 유지 + 포트리스                 |  

상황별 선택 가이드  
- 빠른 시작이 필요한 경우 : 간단 바스티온 EC2 → 즉시 SSM 로깅 연동  
- 보안/감사가 엄격한 환경 : SSM Session Manager 우선 (포트 22/3389 차단, 세션 로깅/KMS 암호화)  
- SSH 워크플로우를 유지하고 싶은 경우 : EIC Endpoint + EC2 Instance Connect (임시키 방식)
- 다수 사용자/광범위 접근이 필요한 경우 : Client VPN 또는 Verified Access  

## SSM Session Manager 빠른 시작
```
# 1) 인스턴스에 SSM 에이전트 및 IAM Role(SSMManagedInstanceCore) 연결
# 2) 사설 서브넷이라면 VPC 엔드포인트(ssm, ec2messages, ssmmessages) 생성
# 3) 세션 시작
aws ssm start-session --target i-xxxxxxxxxxxxxxxxx


# (옵션) DB 등에 포트포워딩
aws ssm start-session \
--target i-xxxxxxxxxxxxxxxxx \
--document-name AWS-StartPortForwardingSession \
--parameters 'portNumber=5432,localPortNumber=5432'
```

IAM 최소 권한(예시, 사용자용)  
```
{
  "Version": "2025-08-31",
  "Statement": [{"Effect": "Allow",
                "Action": ["ssm:StartSession", "ssm:TerminateSession", "ssm:DescribeSessions", "ssm:DescribeInstanceInformation"],
                "Resource": "*" }, 
              {"Effect": "Allow",
              "Action": ["ec2:DescribeInstances"],
              "Resource": "*" }]
}
```

## EC2 Instance Connect Endpoint(EICE) 빠른 시작  
```
# 1) EIC Endpoint를 프라이빗 서브넷에 생성
# 2) 보안그룹: 관리자가 접속할 사내망/프록시만 허용
# 3) 터널 오픈(예시)
aws ec2-instance-connect open-tunnel \
--instance-id i-xxxxxxxxxxxxxxxxx \
--endpoint-id eice-xxxxxxxxxxxxxxxxx \
--local-port 22 --remote-port 22
# 4) 로컬에서 SSH (임시 키는 CLI가 처리)
ssh ec2-user@localhost -p 22
```

## 마이그레이션 경로(바스티온 → SSM/EICE)  
1. 모든 인스턴스에 SSM 에이전트와 IAM Role 부여  
2. VPC 엔드포인트(ssm, ec2messages, ssmmessages) 구성  
3. 세션 로깅/KMS 활성화, 운영팀 교육  
4. 내부 SG에서 22/3389를 점진 축소(특정 팀만 잔존)  
5. EICE 도입(SSH가 필요한 팀 전용), 잔여 22/3389 폐쇄  
6. 바스티온 EC2 종료 및 IaC 정리  

## 모니터링·감사·로깅
- CloudTrail: API 호출 추적(SSM 세션 시작/종료 포함)  
- CloudWatch Logs: SSM 세션 기록, 바스티온 시스템 로그 수집  
- VPC Flow Logs: 네트워크 흐름 감사  
- S3 + KMS: 장기 보관·보안  
- 알림: CloudWatch Alarms, EventBridge로 이상 접속 알림  

## 운영 베스트 프랙티스  
- Portless: 가능하면 22/3389 닫고 SSM/EICE 사용  
- Keyless: 고정 SSH 키 대신 임시 자격/역할 기반  
- MFA & 세분화: IAM 정책·태그로 접속 범위 최소화, 중요 자원은 MFA 조건부  
- 정기 패치/하드닝: 자동 패치, CIS Benchmarks 적용, Fail2ban 등 침입 방지  
- 드리프트 방지: IaC로 표준화, 변경은 PR 기반 코드리뷰  

## 자동화 아이디어
- Auto Scaling + Launch Template로 바스티온 EC2 자동 배포/교체  
- SSM Document로 공통 점검·로그 수집·패치 작업 표준화  
- Session Manager Preferences로 전사 표준 로깅·KMS 일괄 적용  

---  
  
전통적인 바스티온 호스트는 여전히 유효한 보안 패턴이지만  
새로운 프로젝트라면 SSM Session Manager나 EIC Endpoint를 우선 검토하는 것이 좋겠다.  
이들은 운영 부담을 줄이면서도 더 강력한 보안과 감사 기능을 제공하는 것 같다.  
기존에 바스티온을 사용하고 있다면 점진적으로 현대적 방식으로 마이그레이션을 고려해보는게 좋겠다.  
특히 SSM Session Manager는 기존 바스티온과 병행 운영이 가능해 안전한 전환이 가능한 것 같다.  





