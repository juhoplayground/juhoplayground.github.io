---
layout: post
title: PM2로 Python 애플리케이션 관리하기
author: 'Juho'
date: 2025-12-06 00:00:00 +0900
categories: [DevOps]
tags: [PM2, Python, Process Manager, Node.js]
pin: True
toc: True
---

## 목차
1. [PM2란 무엇인가?](#pm2란-무엇인가)
2. [PM2 설치하기](#pm2-설치하기)
3. [Python 애플리케이션 관리하기](#python-애플리케이션-관리하기)
4. [고급 설정 및 운영](#고급-설정-및-운영)

---

## PM2란 무엇인가?

PM2(Process Manager 2)는 Node.js 애플리케이션을 위한 프로덕션 레벨의 프로세스 매니저다.   
하지만 Python, Ruby, PHP 등 다른 언어로 작성된 애플리케이션도 관리할 수 있다.  

### 핵심 기능

#### 1. 프로세스 관리
- 애플리케이션 시작, 중지, 재시작, 삭제  
- 자동 재시작  
- 클러스터 모드 지원 (CPU 코어별 프로세스 분산)  
- 메모리 제한 설정 및 자동 재시작  

#### 2. 모니터링
- 실시간 CPU, 메모리 사용량 모니터링  
- 로그 관리 (stdout, stderr)  
- 프로세스 상태 확인  
- 성능 메트릭 수집  

#### 3. 배포 및 운영
- 무중단 재시작  
- 서버 재부팅 시 자동 실행  
- 환경 변수 관리  
- 설정 파일 기반 관리  

### 왜 PM2를 사용하는가?

**단순 실행 vs PM2 관리**

```bash
# 일반적인 실행 방식
python app.py

# 문제점:
# - 터미널을 닫으면 프로세스도 종료
# - 자동 재시작 불가
# - 로그 관리 어려움
# - 모니터링 불가
```

```bash
# PM2로 실행
pm2 start app.py --name my-app

# 장점:
# - 백그라운드에서 안정적으로 실행
# - 통합 로그 관리
# - 실시간 모니터링
# - 서버 재부팅 시 자동 실행
```

---

## PM2 설치하기

PM2는 Node.js 기반이므로 Node.js와 npm이 필요  

### STEP 1: Node.js 설치

PM2를 사용하기 위해서는 Node.js가 설치되어 있어야 함  

#### Windows
```bash
# Node.js 공식 사이트에서 설치 프로그램 다운로드
# https://nodejs.org/

# 또는 Chocolatey 사용
choco install nodejs
```

#### macOS
```bash
# Homebrew 사용
brew install node
```

#### Linux (Ubuntu/Debian)
```bash
# Node.js 20.x LTS 설치
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
```

#### 설치 확인
```bash
node --version
npm --version
```

### STEP 2: PM2 설치

Node.js가 설치되었다면 npm을 통해 PM2를 전역으로 설치  

```bash
# npm을 사용한 전역 설치
npm install -g pm2

# 또는 yarn 사용
yarn global add pm2
```

### STEP 3: 설치 확인

```bash
# PM2 버전 확인
pm2 --version

# PM2 명령어 도움말
pm2 --help
```

출력 예시:
```
Usage: pm2 [cmd] app

Commands:
  start [options] <file|json>        start and daemonize an app
  stop <id|name|all|json>           stop a process
  restart <id|name|all|json>        restart a process
  ...
```

---

## Python 애플리케이션 관리하기

PM2로 Python 애플리케이션을 관리하는 방법  

### 기본 사용법

#### 1. Python 스크립트 실행

```bash
# 기본 실행
pm2 start app.py

# 애플리케이션 이름 지정
pm2 start app.py --name my-python-app

# Python 인터프리터 명시적으로 지정
pm2 start app.py --interpreter python3

# 특정 Python 가상환경 사용
pm2 start app.py --interpreter /path/to/venv/bin/python
```

#### 2. 명령어 인자 전달

```bash
# Python 스크립트에 인자 전달
pm2 start app.py --name api-server -- --port 8000 --host 0.0.0.0

# 이는 다음과 같이 실행됨:
# python app.py --port 8000 --host 0.0.0.0
```

#### 3. 환경 변수 설정

```bash
# 환경 변수와 함께 실행
pm2 start app.py --name my-app --env production

# 또는 직접 환경 변수 전달
pm2 start app.py --name my-app -- --env DATABASE_URL=postgresql://localhost/db
```

### 프로세스 관리 명령어

#### 프로세스 상태 확인
```bash
# 모든 프로세스 목록 보기
pm2 list
# 또는
pm2 ls

# 특정 프로세스 상세 정보
pm2 show my-app
# 또는 프로세스 ID 사용
pm2 show 0
```

출력 예시:
```
┌─────┬───────────────┬─────────┬─────────┬───────────┬────────┬─────────┐
│ id  │ name          │ mode    │ ↺      │ status    │ cpu    │ memory  │
├─────┼───────────────┼─────────┼─────────┼───────────┼────────┼─────────┤
│ 0   │ my-python-app │ fork    │ 15      │ online    │ 0%     │ 50.2mb  │
└─────┴───────────────┴─────────┴─────────┴───────────┴────────┴─────────┘
```

#### 프로세스 제어
```bash
# 프로세스 중지
pm2 stop my-app

# 프로세스 재시작
pm2 restart my-app

# 프로세스 삭제 (중지 후 목록에서 제거)
pm2 delete my-app

# 모든 프로세스 재시작
pm2 restart all

# 무중단 재시작 (0-downtime reload)
pm2 reload my-app
```

#### 로그 관리
```bash
# 실시간 로그 보기
pm2 logs

# 특정 앱의 로그만 보기
pm2 logs my-app

# 로그 줄 수 제한
pm2 logs --lines 100

# 로그 파일 경로 확인
pm2 show my-app | grep log

# 로그 파일 비우기
pm2 flush

# 로그 파일 완전 삭제
pm2 flush my-app
```

#### 모니터링
```bash
# 실시간 모니터링 대시보드
pm2 monit

# CPU/메모리 사용량 확인
pm2 list
```

### 설정 파일 기반 관리 (ecosystem.config.js)

복잡한 설정은 설정 파일로 관리하는 것이 편리함

#### ecosystem.config.js 생성

```bash
# 기본 설정 파일 생성
pm2 ecosystem
```

#### Python 애플리케이션을 위한 설정 예시

**ecosystem.config.js**
```javascript
module.exports = {
  apps: [
    {
      name: "flask-api",
      script: "app.py",
      interpreter: "python3",
      args: "--host 0.0.0.0 --port 5000",
      instances: 1,
      autorestart: true,
      watch: false,
      max_memory_restart: "500M",
      env: {
        NODE_ENV: "development",
        FLASK_ENV: "development",
        DATABASE_URL: "postgresql://localhost/dev_db"
      },
      env_production: {
        NODE_ENV: "production",
        FLASK_ENV: "production",
        DATABASE_URL: "postgresql://localhost/prod_db"
      },
      error_file: "./logs/err.log",
      out_file: "./logs/out.log",
      log_date_format: "YYYY-MM-DD HH:mm:ss Z",
      merge_logs: true,
      min_uptime: "10s",
      max_restarts: 10,
      restart_delay: 4000
    },
    {
      name: "celery-worker",
      script: "celery_worker.py",
      interpreter: "/path/to/venv/bin/python",
      args: "-A tasks worker --loglevel=info",
      instances: 2,
      exec_mode: "fork",
      autorestart: true,
      watch: false,
      max_memory_restart: "1G",
      env: {
        CELERY_BROKER_URL: "redis://localhost:6379/0"
      }
    },
    {
      name: "data-processor",
      script: "process_data.py",
      interpreter: "python3",
      cron_restart: "0 0 * * *",
      autorestart: false,
      watch: false
    }
  ]
};
```

#### 주요 설정 옵션

| 옵션 | 설명 |
|------|------|
| `name` | 애플리케이션 이름 |
| `script` | 실행할 스크립트 파일 |
| `interpreter` | 사용할 인터프리터 (python, python3, venv 경로 등) |
| `args` | 스크립트에 전달할 인자 |
| `instances` | 실행할 인스턴스 수 (클러스터 모드) |
| `exec_mode` | 실행 모드 (`fork` 또는 `cluster`) |
| `autorestart` | 크래시 시 자동 재시작 여부 |
| `watch` | 파일 변경 감지 및 자동 재시작 |
| `max_memory_restart` | 메모리 제한 (초과 시 재시작) |
| `env` | 환경 변수 |
| `error_file` | 에러 로그 파일 경로 |
| `out_file` | 출력 로그 파일 경로 |
| `cron_restart` | Cron 패턴으로 주기적 재시작 |
| `min_uptime` | 최소 실행 시간 (이보다 빨리 종료되면 에러로 간주) |
| `max_restarts` | 최대 재시작 횟수 |

#### 설정 파일로 실행

```bash
# 개발 환경으로 실행
pm2 start ecosystem.config.js

# 프로덕션 환경으로 실행
pm2 start ecosystem.config.js --env production

# 특정 앱만 실행
pm2 start ecosystem.config.js --only flask-api

# 설정 파일 재적용
pm2 reload ecosystem.config.js
```

### 실전 예제

#### 1. Flask API 서버

**app.py**
```python
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route('/health')
def health():
    return jsonify({"status": "healthy"})

@app.route('/')
def hello():
    env = os.getenv('FLASK_ENV', 'development')
    return jsonify({"message": f"Hello from {env}!"})

if __name__ == '__main__':
    port = int(os.getenv('PORT', 5000))
    app.run(host='0.0.0.0', port=port)
```

**PM2로 실행**
```bash
# 기본 실행
pm2 start app.py --name flask-api --interpreter python3

# 포트 지정
pm2 start app.py --name flask-api --interpreter python3 -- --port 8000

# 설정 파일 사용
pm2 start ecosystem.config.js
```

#### 2. FastAPI 애플리케이션

**main.py**
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}

@app.get("/health")
async def health():
    return {"status": "ok"}
```

**PM2로 실행 (Uvicorn 사용)**
```bash
# Uvicorn으로 FastAPI 실행
pm2 start "uvicorn main:app --host 0.0.0.0 --port 8000" --name fastapi-app

# 또는 ecosystem.config.js 사용
```

**ecosystem.config.js for FastAPI**
```javascript
module.exports = {
  apps: [{
    name: "fastapi-app",
    script: "uvicorn",
    args: "main:app --host 0.0.0.0 --port 8000",
    interpreter: "python3",
    exec_mode: "fork",
    instances: 1,
    autorestart: true,
    watch: false,
    max_memory_restart: "500M",
    env: {
      PORT: 8000
    }
  }]
};
```

#### 3. Django 애플리케이션

```bash
# Gunicorn으로 Django 실행
pm2 start "gunicorn myproject.wsgi:application --bind 0.0.0.0:8000" --name django-app
```

**ecosystem.config.js for Django**
```javascript
module.exports = {
  apps: [{
    name: "django-app",
    script: "gunicorn",
    args: "myproject.wsgi:application --bind 0.0.0.0:8000 --workers 4",
    interpreter: "/path/to/venv/bin/python",
    autorestart: true,
    max_memory_restart: "1G"
  }]
};
```

---

## 고급 설정 및 운영

### 서버 재부팅 시 자동 실행

PM2로 관리하는 프로세스를 서버 재부팅 후에도 자동으로 실행되도록 설정할 수 있음  

```bash
# 현재 실행 중인 프로세스 저장
pm2 save

# 부팅 시 자동 실행 설정
pm2 startup

# 출력되는 명령어를 복사해서 실행 (sudo 권한 필요)
# 예: sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u user --hp /home/user
```

**설정 해제**
```bash
pm2 unstartup
```

### 클러스터 모드

여러 인스턴스를 실행하여 부하를 분산할 수 있음  

```bash
# CPU 코어 수만큼 인스턴스 실행
pm2 start app.py -i max

# 특정 개수의 인스턴스 실행
pm2 start app.py -i 4

# 스케일링 (인스턴스 추가/제거)
pm2 scale my-app +3
pm2 scale my-app 2
```

**ecosystem.config.js에서 설정**
```javascript
module.exports = {
  apps: [{
    name: "api-cluster",
    script: "app.py",
    interpreter: "python3",
    instances: "max",
    exec_mode: "cluster"
  }]
};
```

### 메모리 관리

```bash
# 메모리 제한 설정 (초과 시 자동 재시작)
pm2 start app.py --max-memory-restart 500M
```

**설정 파일에서**
```javascript
{
  max_memory_restart: "500M"
}
```

### 로그 로테이션

PM2는 기본적으로 로그 로테이션을 제공하지 않으므로 `pm2-logrotate` 모듈을 설치  

```bash
# pm2-logrotate 설치
pm2 install pm2-logrotate

# 설정 확인
pm2 conf pm2-logrotate

# 로그 로테이션 설정
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 7
pm2 set pm2-logrotate:compress true
```

### 유용한 팁

#### 1. 프로세스 ID를 이용한 관리

```bash
# ID로 재시작
pm2 restart 0

# 여러 프로세스 동시 제어
pm2 restart 0 2 4
```

#### 2. JSON 출력으로 스크립트 연동

```bash
# JSON 형식으로 프로세스 목록 출력
pm2 jlist

# 프로그래밍 방식으로 활용
pm2 jlist | jq '.[0].pm2_env.status'
```

#### 3. 환경별 설정 관리

```bash
# 개발 환경
pm2 start ecosystem.config.js --env development

# 스테이징 환경
pm2 start ecosystem.config.js --env staging

# 프로덕션 환경
pm2 start ecosystem.config.js --env production
```

### 트러블슈팅

#### 문제: PM2가 Python을 찾지 못함
```bash
# Python 경로 확인
which python3

# 명시적으로 경로 지정
pm2 start app.py --interpreter /usr/bin/python3
```

#### 문제: 가상환경의 Python 사용
```bash
# 가상환경 활성화 후 Python 경로 확인
source venv/bin/activate
which python

# 해당 경로로 실행
pm2 start app.py --interpreter /path/to/venv/bin/python
```

#### 문제: 프로세스가 계속 재시작됨
```bash
# 로그 확인
pm2 logs my-app

# 최소 실행 시간 설정 (빠른 재시작 방지)
pm2 start app.py --min-uptime 10000
```

#### 문제: 메모리 부족
```bash
# 메모리 사용량 확인
pm2 monit

# 메모리 제한 설정
pm2 start app.py --max-memory-restart 500M
```

---

PM2를 활용해서도 Python 애플리케이션을 안정적이고 효율적으로 운영할 수 있다.  
