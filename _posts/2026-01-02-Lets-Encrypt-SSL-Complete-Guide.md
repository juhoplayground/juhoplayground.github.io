---
layout: post
title: Let's Encrypt - 무료 SSL/TLS 인증서 발급부터 자동화까지
author: 'Juho'
date: 2026-01-02 12:00:00 +0900
categories: [DevOps]
tags: [Let's Encrypt, SSL, TLS, HTTPS, Certbot, Nginx, Apache, Docker]
toc: True
---

Let's Encrypt는 무료로 SSL/TLS 인증서를 발급해주는 비영리 인증 기관(CA)입니다.  
이 글에서는 Let's Encrypt의 작동 원리부터 실전 설정, 자동 갱신까지 모든 과정을 단계별로 알아보겠습니다.  

## 목차
1. [Let's Encrypt란?](#lets-encrypt란)
2. [작동 원리 - ACME 프로토콜](#작동-원리---acme-프로토콜)
3. [Certbot 설치](#certbot-설치)
4. [Nginx에서 SSL 인증서 발급 및 설정](#nginx에서-ssl-인증서-발급-및-설정)
5. [Apache에서 SSL 인증서 발급 및 설정](#apache에서-ssl-인증서-발급-및-설정)
6. [자동 갱신 설정](#자동-갱신-설정)
7. [Docker 환경에서 Let's Encrypt 사용](#docker-환경에서-lets-encrypt-사용)
8. [문제 해결](#문제-해결)
9. [보안 모범 사례](#보안-모범-사례)
10. [마무리](#마무리)

## Let's Encrypt란?

Let's Encrypt는 2016년에 시작된 무료, 자동화된, 개방형 인증 기관입니다.  

### 주요 특징

- **무료**: 도메인 검증(DV) SSL/TLS 인증서를 무료로 발급  
- **자동화**: ACME 프로토콜을 통한 자동 발급 및 갱신  
- **보안**: 최신 암호화 기술 지원  
- **투명성**: 발급된 모든 인증서가 공개 로그에 기록  
- **개방형**: 모든 프로토콜과 소프트웨어가 오픈 소스  

### 제한사항

- 인증서 유효기간: 90일 (자동 갱신 권장)  
- Rate Limit:  
  - 등록된 도메인당 주당 50개 인증서  
  - 계정당 주당 300개의 신규 주문  
  - 중복 인증서: 주당 5개  

## 작동 원리 - ACME 프로토콜

Let's Encrypt는 ACME(Automatic Certificate Management Environment) 프로토콜을 사용합니다.  

### 인증 과정

1. **도메인 소유권 검증 요청**  
   - 클라이언트(Certbot)가 Let's Encrypt CA에 인증서 발급 요청  

2. **챌린지 제공**  
   - CA가 도메인 소유권을 증명할 챌린지 제공  
   - HTTP-01, DNS-01, TLS-ALPN-01 챌린지 중 선택  

3. **챌린지 응답**  
   - HTTP-01: 웹 서버의 특정 경로에 파일 배치  
   - DNS-01: 특정 DNS TXT 레코드 생성  
   - TLS-ALPN-01: TLS 확장을 사용한 검증  

4. **검증 및 발급**  
   - CA가 챌린지 검증 성공 시 인증서 발급  

### HTTP-01 챌린지 예시  

```
http://example.com/.well-known/acme-challenge/{token}
```

위 경로에 CA가 제공한 특정 내용의 파일을 배치하여 도메인 소유권을 증명합니다.  

## Certbot 설치

Certbot은 Let's Encrypt의 공식 권장 클라이언트입니다.  

### Ubuntu/Debian

```bash
# 시스템 업데이트
sudo apt update

# Certbot 설치
sudo apt install certbot

# Nginx용 플러그인 설치 (Nginx 사용 시)
sudo apt install python3-certbot-nginx

# Apache용 플러그인 설치 (Apache 사용 시)
sudo apt install python3-certbot-apache
```

### CentOS/RHEL

```bash
# EPEL 저장소 활성화
sudo yum install epel-release

# Certbot 설치
sudo yum install certbot

# Nginx용 플러그인
sudo yum install python3-certbot-nginx

# Apache용 플러그인
sudo yum install python3-certbot-apache
```

### macOS

```bash
# Homebrew 사용
brew install certbot
```

### 설치 확인

```bash
certbot --version
```

## Nginx에서 SSL 인증서 발급 및 설정

### 1. Nginx 설정 파일 준비

먼저 HTTP(80 포트)로 서비스가 가능한 상태여야 합니다.  

```nginx
# /etc/nginx/sites-available/example.com
server {
    listen 80;
    server_name example.com www.example.com;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

```bash
# 설정 활성화
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/

# Nginx 설정 테스트
sudo nginx -t

# Nginx 재시작
sudo systemctl restart nginx
```

### 2. 자동 설정 방식 (권장)

Certbot이 자동으로 Nginx 설정을 수정합니다.  

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

### 3. 수동 설정 방식

인증서만 발급받고 직접 설정합니다.  

```bash
# 인증서 발급
sudo certbot certonly --nginx -d example.com -d www.example.com
```

발급된 인증서로 Nginx 설정 수정:  

```nginx
# /etc/nginx/sites-available/example.com
server {
    listen 80;
    server_name example.com www.example.com;

    # HTTP를 HTTPS로 리다이렉트
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # SSL 인증서 경로
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # SSL 설정
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # HSTS (옵션)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

```bash
# 설정 테스트 및 재시작
sudo nginx -t
sudo systemctl reload nginx
```

## Apache에서 SSL 인증서 발급 및 설정

### 1. Apache 설정 파일 준비

```apache
# /etc/apache2/sites-available/example.com.conf
<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com
    DocumentRoot /var/www/html

    <Directory /var/www/html>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

```bash
# 사이트 활성화
sudo a2ensite example.com.conf

# SSL 모듈 활성화
sudo a2enmod ssl

# Apache 재시작
sudo systemctl restart apache2
```

### 2. 자동 설정 방식 (권장)

```bash
sudo certbot --apache -d example.com -d www.example.com
```

### 3. 수동 설정 방식

```bash
# 인증서 발급
sudo certbot certonly --apache -d example.com -d www.example.com
```

Apache 설정 수정:

```apache
# /etc/apache2/sites-available/example.com-ssl.conf
<VirtualHost *:443>
    ServerName example.com
    ServerAlias www.example.com
    DocumentRoot /var/www/html

    # SSL 인증서
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem

    # SSL 프로토콜 설정
    SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite HIGH:!aNULL:!MD5

    # HSTS
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"

    <Directory /var/www/html>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>

# HTTP를 HTTPS로 리다이렉트
<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com
    Redirect permanent / https://example.com/
</VirtualHost>
```

```bash
# SSL 사이트 활성화
sudo a2ensite example.com-ssl.conf

# headers 모듈 활성화 (HSTS용)
sudo a2enmod headers

# Apache 재시작
sudo systemctl reload apache2
```

## 자동 갱신 설정

Let's Encrypt 인증서는 90일마다 갱신해야 합니다.

### Certbot 자동 갱신 확인

Certbot 설치 시 자동으로 갱신 타이머가 설정됩니다.

```bash
# systemd 타이머 확인
sudo systemctl status certbot.timer

# 타이머 활성화
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer
```

### 수동 갱신 테스트

```bash
# 갱신 테스트 (실제로 갱신하지 않음)
sudo certbot renew --dry-run

# 실제 갱신
sudo certbot renew
```

### Cron을 사용한 자동 갱신

systemd 타이머를 사용하지 않는 경우:

```bash
# crontab 편집
sudo crontab -e

# 매일 자정에 갱신 시도
0 0 * * * certbot renew --quiet --post-hook "systemctl reload nginx"
```

### 갱신 후 훅(Hook) 설정

갱신 후 웹 서버를 자동으로 재시작:

```bash
# Nginx
sudo certbot renew --post-hook "systemctl reload nginx"

# Apache
sudo certbot renew --post-hook "systemctl reload apache2"
```

### 갱신 로그 확인

```bash
# 갱신 로그 위치
sudo tail -f /var/log/letsencrypt/letsencrypt.log
```

## Docker 환경에서 Let's Encrypt 사용

### 방법 1: docker-compose + certbot

```yaml
# docker-compose.yml
version: '3'

services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./html:/usr/share/nginx/html
      - certbot-etc:/etc/letsencrypt
      - certbot-var:/var/lib/letsencrypt
    depends_on:
      - certbot

  certbot:
    image: certbot/certbot
    volumes:
      - certbot-etc:/etc/letsencrypt
      - certbot-var:/var/lib/letsencrypt
      - ./html:/var/www/html
    command: certonly --webroot --webroot-path=/var/www/html --email your-email@example.com --agree-tos --no-eff-email -d example.com -d www.example.com

volumes:
  certbot-etc:
  certbot-var:
```

### 방법 2: nginx-proxy + letsencrypt-companion

자동으로 SSL 인증서를 발급하고 갱신하는 솔루션:

```yaml
# docker-compose.yml
version: '3'

services:
  nginx-proxy:
    image: nginxproxy/nginx-proxy
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - certs:/etc/nginx/certs
      - vhost:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html

  letsencrypt-companion:
    image: nginxproxy/acme-companion
    container_name: nginx-proxy-le
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - certs:/etc/nginx/certs
      - vhost:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html
      - acme:/etc/acme.sh
    environment:
      - DEFAULT_EMAIL=your-email@example.com

  # 실제 애플리케이션 예시
  myapp:
    image: myapp:latest
    expose:
      - "8000"
    environment:
      - VIRTUAL_HOST=example.com
      - LETSENCRYPT_HOST=example.com
      - LETSENCRYPT_EMAIL=your-email@example.com

volumes:
  certs:
  vhost:
  html:
  acme:
```

### 갱신 자동화 (Cron in Docker)

```bash
# 인증서 갱신을 위한 cron 작성
# renew.sh
#!/bin/bash
docker-compose run --rm certbot renew
docker-compose exec nginx nginx -s reload
```

```bash
# crontab에 등록
0 0 * * * /path/to/renew.sh
```

## 문제 해결

### 1. Port 80이 열려있지 않음

**증상**: HTTP-01 챌린지 실패

**해결**:
```bash
# 방화벽 확인
sudo ufw status

# Port 80, 443 열기
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

### 2. DNS 레코드 문제

**증상**: 도메인을 찾을 수 없음

**해결**:
```bash
# DNS 확인
nslookup example.com
dig example.com

# A 레코드가 서버 IP를 가리키는지 확인
```

### 3. Rate Limit 초과

**증상**: "too many certificates already issued"

**해결**:
- 테스트 시 `--staging` 플래그 사용
- Rate Limit 리셋 대기 (주간 제한)

```bash
# 스테이징 환경에서 테스트
sudo certbot --staging --nginx -d example.com
```

### 4. Webroot 경로 문제

**증상**: 404 Not Found during challenge

**해결**:
```bash
# 올바른 webroot 경로 지정
sudo certbot certonly --webroot -w /var/www/html -d example.com
```

### 5. 인증서 갱신 실패

```bash
# 상세 로그로 갱신 시도
sudo certbot renew --verbose

# 로그 확인
sudo less /var/log/letsencrypt/letsencrypt.log
```

## 보안 모범 사례

### 1. 강력한 SSL 설정

```nginx
# Nginx SSL 강화
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
ssl_prefer_server_ciphers off;

# DH 파라미터 생성
# sudo openssl dhparam -out /etc/nginx/dhparam.pem 2048
ssl_dhparam /etc/nginx/dhparam.pem;

# OCSP Stapling
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
```

### 2. 보안 헤더 추가

```nginx
# HSTS
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

# XSS 보호
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;

# CSP (Content Security Policy)
add_header Content-Security-Policy "default-src 'self' https: data: 'unsafe-inline' 'unsafe-eval';" always;
```

### 3. SSL Labs 테스트

인증서 설정 후 보안 등급 확인:

```
https://www.ssllabs.com/ssltest/analyze.html?d=example.com
```

목표: A+ 등급

### 4. 인증서 파일 권한 관리

```bash
# 인증서 디렉토리 권한 확인
sudo ls -la /etc/letsencrypt/live/example.com/

# 권한은 자동으로 적절히 설정됨 (root:root, 0600)
# 수동 변경 불필요
```

### 5. 백업

```bash
# 인증서 백업
sudo tar czf letsencrypt-backup-$(date +%Y%m%d).tar.gz /etc/letsencrypt/
```

## 마무리

Let's Encrypt는 무료로 SSL/TLS 인증서를 발급받아 웹사이트의 보안을 강화할 수 있는 훌륭한 솔루션입니다.

**핵심 요약**:
- Certbot을 사용하여 간단하게 인증서 발급 가능
- 자동 갱신 설정으로 90일 갱신 주기 관리
- Nginx, Apache 모두 자동 설정 지원
- Docker 환경에서도 쉽게 적용 가능
- 적절한 SSL 설정과 보안 헤더로 A+ 등급 달성 가능

HTTPS는 이제 웹사이트의 필수 요소입니다.  
Let's Encrypt를 활용하여 안전한 웹 서비스를 제공하시기 바랍니다.  
