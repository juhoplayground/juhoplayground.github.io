---
layout: post
title: GCP HTTP/HTTPS 로드 밸런서 구성 가이드
author: 'Juho'
date: 2026-01-04 00:00:00 +0900
categories: [DevOps]
tags: [GCP, Load-Balancer, cloud, Nginx]
toc: true
---

VM 인스턴스 IP를 숨기고 Google Cloud Load Balancer를 통해 서비스를 운영하는 방법을 단계별로 안내합니다.

## 1. 사전 준비

```bash
# GCP 프로젝트 설정 확인
gcloud config set project {PROJECT_ID}

# 사용할 리전/존 설정
gcloud config set compute/region asia-northeast3
gcloud config set compute/zone asia-northeast3-a
```

## 2. 인스턴스 그룹 생성

기존 VM을 비관리형 인스턴스 그룹에 추가:

```bash
# 비관리형 인스턴스 그룹 생성
 gcloud compute instance-groups unmanaged create {INSTANCE_GROUP_NAME} \
      --project={PROJECT_ID} \
      --description="Instance group for load balancer" \
      --zone=asia-northeast3-a \
  && \
  gcloud compute instance-groups unmanaged set-named-ports {INSTANCE_GROUP_NAME} \
      --project={PROJECT_ID} \
      --zone=asia-northeast3-a \
      --named-ports=http:80,https:443

# 기존 VM 인스턴스를 그룹에 추가
gcloud compute instance-groups unmanaged add-instances {INSTANCE_GROUP_NAME} \
    --zone=asia-northeast3-a \
    --instances={VM_INSTANCE_NAME}
```

## 3. 헬스 체크 생성

### HTTP 헬스 체크 (Nginx 기준)

```bash
# HTTP 헬스 체크 생성
gcloud compute health-checks create http {HEALTH_CHECK_NAME} \
    --port=80 \
    --request-path=/ \
    --check-interval=10s \
    --timeout=5s \
    --healthy-threshold=2 \
    --unhealthy-threshold=3
```

## 4. 백엔드 서비스 생성

```bash
# HTTP 백엔드 서비스 생성
gcloud compute backend-services create {BACKEND_SERVICE_NAME} \
    --protocol=HTTP \
    --health-checks={HEALTH_CHECK_NAME} \
    --global

# 인스턴스 그룹을 백엔드에 추가
gcloud compute backend-services add-backend {BACKEND_SERVICE_NAME} \
    --instance-group={INSTANCE_GROUP_NAME} \
    --instance-group-zone=asia-northeast3-a \
    --balancing-mode=UTILIZATION \
    --max-utilization=0.8 \
    --global
```

## 5. URL 맵 생성

```bash
# URL 맵 생성 (기본 백엔드 서비스 지정)
gcloud compute url-maps create {URL_MAP_NAME} \
    --default-service={BACKEND_SERVICE_NAME}
```

## 6.  Google 관리형 SSL 인증서 구성 (HTTPS용)
```bash
# 관리형 SSL 인증서 생성 (도메인 필요)
gcloud compute ssl-certificates create {SSL_CERT_NAME} \
    --domains=yourdomain.com,www.yourdomain.com \
    --global
```

## 7. 타겟 프록시 생성

### HTTPS 프록시

```bash
# HTTPS 타겟 프록시 생성
gcloud compute target-https-proxies create {HTTPS_PROXY_NAME} \
    --url-map={URL_MAP_NAME} \
    --ssl-certificates={SSL_CERT_NAME}
```

### HTTP 프록시 (HTTP→HTTPS 리다이렉트용)

```bash
# HTTP 타겟 프록시 생성
gcloud compute target-http-proxies create {HTTP_PROXY_NAME} \
    --url-map={URL_MAP_NAME}
```

## 8. 전역 IP 주소 예약

```bash
# 고정 외부 IP 주소 예약
gcloud compute addresses create {LB_IP_NAME} \
    --ip-version=IPV4 \
    --global

# 예약된 IP 주소 확인
gcloud compute addresses describe {LB_IP_NAME} \
    --format="get(address)" \
    --global
```

## 9. 전달 규칙 생성

### HTTPS 전달 규칙

```bash
# HTTPS 전달 규칙 생성
gcloud compute forwarding-rules create {HTTPS_FORWARDING_RULE_NAME} \
    --address={LB_IP_NAME} \
    --global \
    --target-https-proxy={HTTPS_PROXY_NAME} \
    --ports=443
```

### HTTP 전달 규칙 (선택 사항)

```bash
# HTTP 전달 규칙 생성
gcloud compute forwarding-rules create {HTTP_FORWARDING_RULE_NAME} \
    --address={LB_IP_NAME} \
    --global \
    --target-http-proxy={HTTP_PROXY_NAME} \
    --ports=80
```

## 10. HTTP→HTTPS 리다이렉트 설정

```bash
# HTTP→HTTPS 리다이렉트용 URL 맵 생성
gcloud compute url-maps import {HTTP_REDIRECT_MAP_NAME} \
    --global \
    --source=/dev/stdin <<EOF
name: {HTTP_REDIRECT_MAP_NAME}
defaultUrlRedirect:
  redirectResponseCode: MOVED_PERMANENTLY_DEFAULT
  httpsRedirect: true
EOF

# HTTP 프록시를 리다이렉트 URL 맵으로 업데이트
gcloud compute target-http-proxies update {HTTP_PROXY_NAME} \
    --url-map={HTTP_REDIRECT_MAP_NAME}
```

## 11. 방화벽 규칙 구성

### 헬스 체크 허용

```bash
# 로드 밸런서 헬스 체크 허용 (GCP의 헬스 체크 IP 범위)
gcloud compute firewall-rules create allow-health-check \
    --network=default \
    --action=allow \
    --direction=ingress \
    --source-ranges=130.211.0.0/22,35.191.0.0/16 \
    --rules=tcp:80,tcp:443
```

### 로드 밸런서만 접근 허용 (선택 사항)

```bash
# 로드 밸런서만 접근 허용
gcloud compute firewall-rules create allow-lb-only \
    --network=default \
    --action=allow \
    --direction=ingress \
    --source-ranges=130.211.0.0/22,35.191.0.0/16 \
    --target-tags={BACKEND_TAG} \
    --rules=tcp:80,tcp:443

# VM 인스턴스에 네트워크 태그 추가
gcloud compute instances add-tags {VM_INSTANCE_NAME} \
    --zone=asia-northeast3-a \
    --tags={BACKEND_TAG}
```

## 12. VM 직접 접근 차단 (선택 사항)

```bash
# 외부 직접 접근 차단 (로드 밸런서만 허용)
gcloud compute firewall-rules create deny-direct-access \
    --network=default \
    --action=deny \
    --direction=ingress \
    --priority=900 \
    --source-ranges=0.0.0.0/0 \
    --target-tags={BACKEND_TAG} \
    --rules=tcp:80,tcp:443

# SSH 접근은 유지 (필요시)
gcloud compute firewall-rules create allow-ssh \
    --network=default \
    --action=allow \
    --direction=ingress \
    --priority=800 \
    --source-ranges=0.0.0.0/0 \
    --target-tags={BACKEND_TAG} \
    --rules=tcp:22
```

## 13. DNS 설정

### 로드 밸런서 IP 확인

```bash
# 예약된 IP 확인
gcloud compute addresses list --global
```

### Cloud DNS 사용 시

```bash
# DNS A 레코드에 로드 밸런서 IP 추가
gcloud dns record-sets create yourdomain.com. \
    --zone=YOUR_DNS_ZONE \
    --type=A \
    --ttl=300 \
    --rrdatas=LOAD_BALANCER_IP

# www 서브도메인 추가
gcloud dns record-sets create www.yourdomain.com. \
    --zone=YOUR_DNS_ZONE \
    --type=A \
    --ttl=300 \
    --rrdatas=LOAD_BALANCER_IP
```

### 외부 DNS 사용 시

도메인 관리 콘솔에서 A 레코드를 로드 밸런서 IP로 설정합니다.

## 14. 설정 확인

```bash
# 로드 밸런서 구성 확인
gcloud compute forwarding-rules list
gcloud compute backend-services list
gcloud compute health-checks list
gcloud compute url-maps list

# 백엔드 서비스 상태 확인
gcloud compute backend-services get-health {BACKEND_SERVICE_NAME} \
    --global

# SSL 인증서 상태 확인
gcloud compute ssl-certificates list
gcloud compute ssl-certificates describe {SSL_CERT_NAME} \
    --global \
    --format="get(managed.status)"
```

## 15. VM 외부 IP 제거 (완전히 숨기기)

```bash
# VM 인스턴스의 외부 IP 제거
gcloud compute instances delete-access-config {VM_INSTANCE_NAME} \
    --zone=asia-northeast3-a \
    --access-config-name="External NAT"
```

**주의**: 외부 IP 제거 후 SSH 접근은 다음 방법으로 가능합니다:
- Cloud Console의 SSH 버튼 사용
- IAP(Identity-Aware Proxy) 터널 사용
- Bastion 호스트를 통한 접근

## 추가 최적화 옵션

### CDN 활성화

```bash
gcloud compute backend-services update {BACKEND_SERVICE_NAME} \
    --enable-cdn \
    --global
```

### 로깅 활성화

```bash
gcloud compute backend-services update {BACKEND_SERVICE_NAME} \
    --enable-logging \
    --logging-sample-rate=1.0 \
    --global
```

### Cloud Armor 보안 정책 (DDoS 방어)

```bash
# 보안 정책 생성
gcloud compute security-policies create {SECURITY_POLICY_NAME} \
    --description="Security policy for load balancer"

# 기본 규칙: 모든 트래픽 허용
gcloud compute security-policies rules create 1000 \
    --security-policy={SECURITY_POLICY_NAME} \
    --action=allow

# 특정 IP 차단 예시
gcloud compute security-policies rules create 500 \
    --security-policy={SECURITY_POLICY_NAME} \
    --expression="origin.ip == '123.123.123.123'" \
    --action=deny-403

# 백엔드 서비스에 적용
gcloud compute backend-services update {BACKEND_SERVICE_NAME} \
    --security-policy={SECURITY_POLICY_NAME} \
    --global
```

### 세션 어피니티 설정

```bash
# 세션 어피니티 활성화 (쿠키 기반)
gcloud compute backend-services update {BACKEND_SERVICE_NAME} \
    --session-affinity=GENERATED_COOKIE \
    --affinity-cookie-ttl=3600 \
    --global
```

## 트러블슈팅

### Billing 활성화 오류 시

**오류 메시지**:
```
인스턴스 그룹 생성 실패: This API method requires billing to be enabled.
Please enable billing on project #{PROJECT_ID} by visiting
https://console.developers.google.com/billing/enable?project={PROJECT_ID} then retry.
If you enabled billing for this project recently, wait a few minutes for the action to propagate to our systems and retry.
```

**원인**: GCP 프로젝트에 결제(Billing) 계정이 연결되지 않았거나 비활성화된 상태

**해결 방법**:

```bash
# 1. 브라우저에서 Billing 활성화 페이지 접속
# https://console.developers.google.com/billing/enable?project={PROJECT_ID}

# 2. 또는 GCP Console에서 수동 설정
# - GCP Console > Billing > 프로젝트에 연결
# - 기존 Billing 계정 선택 또는 새로 생성

# 3. Billing 계정 연결 확인
gcloud beta billing projects describe {PROJECT_ID}

# 4. 활성화 후 5-10분 대기 후 재시도
gcloud compute instance-groups unmanaged create {INSTANCE_GROUP_NAME} \
    --project={PROJECT_ID} \
    --description="Instance group for load balancer" \
    --zone=asia-northeast3-a \
    --named-ports=http:80,https:443
```

**참고**:
- Billing 활성화 후 즉시 반영되지 않을 수 있음 (최대 10분 소요)
- Free Tier 크레딧이 있어도 Billing 계정 연결은 필수
- 결제 정보 등록 필요 (신용카드/체크카드)

### 헬스 체크 실패 시

```bash
# 헬스 체크 상태 확인
gcloud compute backend-services get-health {BACKEND_SERVICE_NAME} --global

# 방화벽 규칙 확인
gcloud compute firewall-rules list --filter="name~allow-health-check"

# VM에서 직접 테스트
curl -I http://localhost:80
```

### SSL 인증서 프로비저닝 실패 시

```bash
# 인증서 상태 확인
gcloud compute ssl-certificates describe {SSL_CERT_NAME} --global

# DNS 설정 확인
dig yourdomain.com
nslookup yourdomain.com
```

### 로드 밸런서 연결 실패 시

```bash
# 전달 규칙 확인
gcloud compute forwarding-rules describe {HTTPS_FORWARDING_RULE_NAME} --global

# 백엔드 서비스 확인
gcloud compute backend-services describe {BACKEND_SERVICE_NAME} --global
```

## 리소스 정리 (삭제)

설정을 제거하려면 역순으로 삭제:

```bash
# 전달 규칙 삭제
gcloud compute forwarding-rules delete {HTTPS_FORWARDING_RULE_NAME} --global
gcloud compute forwarding-rules delete {HTTP_FORWARDING_RULE_NAME} --global

# 타겟 프록시 삭제
gcloud compute target-https-proxies delete {HTTPS_PROXY_NAME}
gcloud compute target-http-proxies delete {HTTP_PROXY_NAME}

# URL 맵 삭제
gcloud compute url-maps delete {URL_MAP_NAME}
gcloud compute url-maps delete {HTTP_REDIRECT_MAP_NAME}

# SSL 인증서 삭제
gcloud compute ssl-certificates delete {SSL_CERT_NAME} --global

# 백엔드 서비스 삭제
gcloud compute backend-services delete {BACKEND_SERVICE_NAME} --global

# 헬스 체크 삭제
gcloud compute health-checks delete {HEALTH_CHECK_NAME}

# 인스턴스 그룹 삭제
gcloud compute instance-groups unmanaged delete {INSTANCE_GROUP_NAME} --zone=asia-northeast3-a

# IP 주소 해제
gcloud compute addresses delete {LB_IP_NAME} --global

# 방화벽 규칙 삭제
gcloud compute firewall-rules delete allow-health-check
gcloud compute firewall-rules delete allow-lb-only
gcloud compute firewall-rules delete deny-direct-access
gcloud compute firewall-rules delete allow-ssh
```

## 16. 환경 설정 업데이트 (애플리케이션 설정)

로드 밸런서 설정 완료 후, 애플리케이션 환경 변수를 업데이트해야 합니다.

### 로드 밸런서 IP 확인

```bash
# 예약된 로드 밸런서 IP 주소 확인
LOAD_BALANCER_IP=$(gcloud compute addresses describe {LB_IP_NAME} \
    --format="get(address)" \
    --global)

echo "로드 밸런서 IP: $LOAD_BALANCER_IP"
```

### 애플리케이션 설정 파일 수정

**변경 전** (기존 VM IP):
```bash
SERVER_IP={OLD_VM_IP}
SERVER_HOST={OLD_VM_IP}
SERVER_NAME={OLD_VM_IP}
```

**변경 후** (로드 밸런서 IP 또는 도메인):

#### Option A: IP 주소 사용
```bash
SERVER_IP=<LOAD_BALANCER_IP>
SERVER_HOST=<LOAD_BALANCER_IP>
SERVER_NAME=<LOAD_BALANCER_IP>
```

#### Option B: 도메인 사용
```bash
SERVER_IP=yourdomain.com
SERVER_HOST=yourdomain.com
SERVER_NAME=yourdomain.com
```

### 애플리케이션 Allowed Hosts 업데이트

로드 밸런서 IP/도메인을 애플리케이션의 허용된 호스트 목록에 추가:

```bash
# 예시: Django의 경우
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1,<LOAD_BALANCER_IP>,yourdomain.com
```

### 변경 사항 적용

```bash
# 애플리케이션 재시작 (예시: Docker 사용 시)
docker compose down
docker compose up -d

# 또는 웹 서버 재시작
systemctl restart nginx
```

### 변경 사항 확인

```bash
# 로드 밸런서 IP로 접속 테스트
curl -I https://<LOAD_BALANCER_IP>

# 도메인 사용 시
curl -I https://yourdomain.com
```

**참고**:
- VM 직접 IP는 더 이상 사용하지 않으므로, 애플리케이션이 로드 밸런서 IP/도메인을 참조하도록 설정
- CORS 설정, API 엔드포인트, 프론트엔드 설정도 함께 업데이트 필요
- SSL 인증서 사용 시 HTTPS 프로토콜로 설정

## 시나리오별 구성 가이드

### 시나리오 A: HTTP만 사용 (테스트/개발 환경)

**적용 대상**: IP 주소 접근, 내부 네트워크, 비용 절감 우선

**진행할 단계**:
1. ✅ 사전 준비 (1번)
2. ✅ 인스턴스 그룹 생성 (2번)
3. ✅ HTTP 헬스 체크 생성 (3번)
4. ✅ 백엔드 서비스 생성 (4번)
5. ✅ URL 맵 생성 (5번)
6. ❌ **건너뛰기**: SSL 인증서 구성 (6번)
7. ✅ HTTP 타겟 프록시만 생성 (7번 - HTTP 프록시만)
8. ✅ 전역 IP 주소 예약 (8번)
9. ✅ HTTP 전달 규칙만 생성 (9번 - HTTP 전달 규칙만)
10. ❌ **건너뛰기**: HTTP→HTTPS 리다이렉트 (10번)
11. ✅ 방화벽 규칙 구성 (11번)
12. ✅ VM 직접 접근 차단 (12번 - 선택)
13. ❌ **건너뛰기**: DNS 설정 (13번 - 도메인 없으면)
14. ✅ 설정 확인 (14번 - SSL 관련 제외)
15. ✅ VM 외부 IP 제거 (15번 - 선택)
16. ✅ 환경 설정 업데이트 (16번 - IP 주소 사용)

**환경 변수 설정**:
```bash
# 애플리케이션 설정 예시
SERVER_IP=<LOAD_BALANCER_IP>
SERVER_HOST=<LOAD_BALANCER_IP>
SERVER_NAME=<LOAD_BALANCER_IP>
SERVER_PROTOCOL=http
ALLOWED_HOSTS=localhost,127.0.0.1,<LOAD_BALANCER_IP>
```

**접속 URL**: `http://<LOAD_BALANCER_IP>`

---

### 시나리오 B: HTTPS 사용 (프로덕션 권장)

**적용 대상**: 도메인 보유, 프로덕션 환경, 보안 필요 (의료/연구 데이터)

**진행할 단계**:
1. ✅ 사전 준비 (1번)
2. ✅ 인스턴스 그룹 생성 (2번)
3. ✅ HTTP 헬스 체크 생성 (3번)
4. ✅ 백엔드 서비스 생성 (4번)
5. ✅ URL 맵 생성 (5번)
6. ✅ **Google 관리형 SSL 인증서 구성** (6번)
7. ✅ HTTPS + HTTP 타겟 프록시 모두 생성 (7번)
8. ✅ 전역 IP 주소 예약 (8번)
9. ✅ HTTPS + HTTP 전달 규칙 모두 생성 (9번)
10. ✅ **HTTP→HTTPS 리다이렉트 설정** (10번)
11. ✅ 방화벽 규칙 구성 (11번)
12. ✅ VM 직접 접근 차단 (12번 - 선택)
13. ✅ **DNS 설정** (13번 - 필수)
14. ✅ 설정 확인 (14번 - SSL 상태 포함)
15. ✅ VM 외부 IP 제거 (15번 - 선택)
16. ✅ 환경 설정 업데이트 (16번 - 도메인 사용)

**환경 변수 설정**:
```bash
# 애플리케이션 설정 예시
SERVER_IP=yourdomain.com
SERVER_HOST=yourdomain.com
SERVER_NAME=yourdomain.com
SERVER_PROTOCOL=https
ALLOWED_HOSTS=localhost,127.0.0.1,yourdomain.com,www.yourdomain.com,<LOAD_BALANCER_IP>
```

**접속 URL**:
- `https://yourdomain.com` (권장)
- `http://yourdomain.com` (자동으로 HTTPS로 리다이렉트)

**중요 사항**:
- DNS 설정 후 SSL 인증서 프로비저닝까지 **최대 15분** 소요
- 인증서 상태는 `gcloud compute ssl-certificates describe` 명령으로 확인
- 상태가 `ACTIVE`가 되어야 HTTPS 접속 가능

---

### 두 시나리오 비교

| 항목 | HTTP만 사용 | HTTPS 사용 |
|------|------------|-----------|
| **SSL 인증서** | 불필요 | Google 관리형 (무료) |
| **도메인** | 선택 사항 | 필수 |
| **접속 URL** | `http://IP` | `https://domain.com` |
| **보안** | 낮음 (평문 전송) | 높음 (암호화) |
| **설정 복잡도** | 낮음 | 중간 (DNS 설정 필요) |
| **비용** | 저렴 | 동일 (SSL 무료) |
| **프로덕션 적합성** | ❌ 부적합 | ✅ 권장 |
| **사용 권장** | ⚠️ 개발/테스트용 | ✅ 프로덕션 권장 |

**환경별 추천**:
- **로컬 개발**: 로드 밸런서 불필요 (직접 접속)
- **테스트 서버**: HTTP만 사용 (시나리오 A)
- **프로덕션**: HTTPS 사용 (시나리오 B) - 데이터 보안을 위해 필수

## 비용 최적화 팁

1. **헬스 체크 간격 조정**: 프로덕션 환경이 아니면 check-interval을 늘려 비용 절감
2. **CDN 선택적 사용**: 정적 자산에만 CDN 적용
3. **로깅 샘플링**: 로깅 샘플 레이트를 0.1로 낮춰 비용 절감
4. **예약 용량**: 장기 사용 시 Committed Use Discounts 고려

## 참고 자료

- [GCP Load Balancing 공식 문서](https://cloud.google.com/load-balancing/docs)
- [SSL 인증서 관리](https://cloud.google.com/load-balancing/docs/ssl-certificates)
- [Cloud Armor 보안 정책](https://cloud.google.com/armor/docs/security-policy-overview)
- [헬스 체크 구성](https://cloud.google.com/load-balancing/docs/health-checks)
