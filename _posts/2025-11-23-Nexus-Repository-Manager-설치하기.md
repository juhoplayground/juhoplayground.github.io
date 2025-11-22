---
layout: post
title: AWS EC2에 Nexus Repository Manager 설치하기
author: 'Juho'
date: 2025-11-23 00:00:00 +0900
categories: [DevOps]
tags: [DevOps, Nexus, Maven, Repository, AWS, EC2]
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
1. [Nexus Repository Manager란?](#nexus-repository-manager란)
2. [EC2 인스턴스 생성](#1단계-ec2-인스턴스-생성)
3. [시스템 업데이트 및 Java 설치](#2단계-시스템-업데이트-및-java-설치)
4. [Nexus Repository Manager 설치](#3단계-nexus-repository-manager-설치)
5. [시스템 서비스 등록](#4단계-시스템-서비스-등록)
6. [웹 UI 접속 및 초기 설정](#5단계-웹-ui-접속-및-초기-설정)
7. [Maven Repository 생성](#6단계-maven-repository-생성)
8. [프로젝트에서 Nexus 사용](#7단계-프로젝트에서-nexus-사용)
9. [보안 설정](#8단계-보안-설정)
10. [백업 및 복구](#9단계-백업-및-복구)
11. [트러블슈팅](#트러블슈팅)

## Nexus Repository Manager란?  
Nexus Repository Manager는 Sonatype에서 개발한 저장소 관리자입니다.  
Maven, npm, Docker, PyPI 등 다양한 패키지 형식을 지원하며, 사내 프라이빗 저장소로 활용할 수 있습니다.  

주요 사용 목적:  
- 사내 라이브러리 관리 및 배포  
- 외부 저장소 프록시를 통한 캐싱으로 빌드 속도 향상  
- 의존성 관리 및 보안 통제  

## 1단계: EC2 인스턴스 생성  

### AWS 콘솔에서 EC2 생성  
1. AWS Console → EC2 → 인스턴스 시작  
2. AMI 선택: Amazon Linux 2023 또는 Ubuntu 22.04 LTS 권장  
3. 인스턴스 타입: 최소 t3.medium (2 vCPU, 4GB RAM) 권장  
   - Nexus는 메모리를 많이 사용하므로 t3.large (8GB)가 더 안정적  
4. 스토리지: 최소 50GB (EBS gp3) - 패키지가 쌓이면 늘어나므로 넉넉하게  
5. 키 페어: 새로 생성하거나 기존 키 사용  

### 보안 그룹 설정  
인바운드 규칙:  

| 포트 | 프로토콜 | 소스 | 용도 |
|------|----------|------|------|
| 22 | TCP | 내 IP | SSH 접속 |
| 8081 | TCP | 필요한 IP 대역 | Nexus 웹 UI |
| 443 | TCP | 필요한 IP 대역 | HTTPS (선택사항) |

## 2단계: 시스템 업데이트 및 Java 설치  

### 시스템 업데이트  
```bash
# Amazon Linux
sudo yum update -y
```

### Java 설치 (Nexus 필수 요구사항)  
```bash
sudo yum install java-17-amazon-corretto -y

# 설치 확인
java -version
```

## 3단계: Nexus Repository Manager 설치  

### 3-1. Nexus 다운로드  
```bash
cd /opt
sudo wget https://download.sonatype.com/nexus/3/latest-unix.tar.gz
sudo tar -xvzf latest-unix.tar.gz
sudo mv nexus-3* nexus
sudo mv sonatype-work sonatype-work
```

### 3-2. Nexus 전용 사용자 생성  
```bash
sudo useradd -r -m -U -d /opt/nexus -s /bin/bash nexus
sudo chown -R nexus:nexus /opt/nexus
sudo chown -R nexus:nexus /opt/sonatype-work
```

### 3-3. Nexus 실행 사용자 설정  
```bash
echo 'run_as_user="nexus"' | sudo tee /opt/nexus/bin/nexus.rc
```

### 3-4. 메모리 설정 (선택사항)  
인스턴스 사양에 맞게 메모리를 조정합니다.  
```bash
sudo vi /opt/nexus/bin/nexus.vmoptions

-Xms1024m
-Xmx1024m
```

## 4단계: 시스템 서비스 등록  

### systemd 서비스 파일 생성  
```bash
sudo vi /etc/systemd/system/nexus.service
```

아래 내용을 입력합니다:  
```ini
[Unit]
Description=Nexus Repository Manager
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
ExecStart=/opt/nexus/bin/nexus start
ExecStop=/opt/nexus/bin/nexus stop
User=nexus
Restart=on-abort

[Install]
WantedBy=multi-user.target
```

### 서비스 시작  
```bash
sudo systemctl daemon-reload
sudo systemctl enable nexus
sudo systemctl start nexus

# 상태 확인 (시작까지 1-2분 소요)
sudo systemctl status nexus
```

### 로그 확인  
```bash
# Nexus 로그 위치
tail -f /opt/sonatype-work/nexus3/log/nexus.log
```

## 5단계: 웹 UI 접속 및 초기 설정  

### 브라우저에서 접속  
```
http://<EC2-퍼블릭-IP>:8081
```

### 초기 관리자 비밀번호 확인  
```bash
sudo cat /opt/sonatype-work/nexus3/admin.password
```

### 초기 설정 진행  
1. 위 비밀번호로 admin 계정 로그인  
2. 새 비밀번호 설정  
3. Anonymous Access 설정 (보안상 Disable 권장)  

## 6단계: Maven Repository 생성  

Nexus 웹 UI에서 설정(톱니바퀴) → Repositories → Create repository를 선택합니다.  

### 6-1. Release 저장소 생성  
Recipe 선택: **maven2 (hosted)**  

| 설정 항목 | 값 |
|-----------|-----|
| Name | maven-releases |
| Version policy | Release |
| Layout policy | Strict |
| Deployment policy | Allow redeploy (개발용) 또는 Disable (운영용) |

### 6-2. Snapshot 저장소 생성  
Recipe 선택: **maven2 (hosted)**  

| 설정 항목 | 값 |
|-----------|-----|
| Name | maven-snapshots |
| Version policy | Snapshot |
| Layout policy | Strict |
| Deployment policy | Allow redeploy |

### 6-3. Maven Central 프록시 생성  
Recipe 선택: **maven2 (proxy)**  

| 설정 항목 | 값 |
|-----------|-----|
| Name | maven-central |
| Remote storage | https://repo1.maven.org/maven2/ |
| Version policy | Release |

### 6-4. 그룹 저장소 생성  
Recipe 선택: **maven2 (group)**  

| 설정 항목 | 값 |
|-----------|-----|
| Name | maven-public |
| Member repositories | maven-releases, maven-snapshots, maven-central (순서대로) |

### 6-5. 생성된 저장소 확인  

| Name | Type | URL |
|------|------|-----|
| maven-releases | hosted | http://\<IP\>:8081/repository/maven-releases/ |
| maven-snapshots | hosted | http://\<IP\>:8081/repository/maven-snapshots/ |
| maven-public | group | http://\<IP\>:8081/repository/maven-public/ |

## 7단계: 프로젝트에서 Nexus 사용  

### Gradle 설정 (build.gradle)  

#### gradle.properties 설정 (보안)  
credentials를 build.gradle에 직접 작성하면 보안상 위험합니다.  
`~/.gradle/gradle.properties` 또는 프로젝트 루트의 `gradle.properties`에 작성하고, `.gitignore`에 추가합니다.  

```properties
# gradle.properties
nexusUrl=http://<Nexus-IP>:8081
nexusUsername=deployer
nexusPassword=your-secure-password
```

#### repositories 블록  
```groovy
repositories {
    maven {
        url "${nexusUrl}/repository/maven-public/"
        allowInsecureProtocol true
        credentials {
            username nexusUsername
            password nexusPassword
        }
    }
}
```

#### publishing 블록  
```groovy
publishing {
    repositories {
        maven {
            def snapshotsRepoUrl = "${nexusUrl}/repository/maven-snapshots/"
            def releasesRepoUrl = "${nexusUrl}/repository/maven-releases/"
            url = version.endsWith('SNAPSHOT') ? snapshotsRepoUrl : releasesRepoUrl
            allowInsecureProtocol true
            credentials {
                username nexusUsername
                password nexusPassword
            }
        }
    }
}
```

### 배포 테스트  
```bash
./gradlew publish
```

성공하면 Nexus 웹 UI → Browse → maven-releases에서 배포된 아티팩트를 확인할 수 있습니다.  

### 다른 프로젝트에서 의존성으로 사용  
```groovy
repositories {
    maven {
        url = uri("${nexusUrl}/repository/maven-public/")
        allowInsecureProtocol true
        credentials {
            username = nexusUsername
            password = nexusPassword
        }
    }
}

dependencies {
    implementation("com.example:my-lib:1.0.0")
}
```

## 8단계: 보안 설정  

### 8-1. 배포 전용 사용자 생성  
admin 계정을 직접 사용하는 것은 보안상 위험합니다.  
배포 전용 사용자를 생성합니다.  

Nexus 웹 UI → 설정(톱니바퀴) → Security → Users → Create local user  

| 설정 항목 | 값 |
|-----------|-----|
| ID | deployer |
| First name | Deploy |
| Last name | User |
| Email | deployer@example.com |
| Status | Active |
| Roles | nx-deploy (아래에서 생성) |

### 8-2. 배포 전용 Role 생성  
설정 → Security → Roles → Create Role → Nexus Role  

| 설정 항목 | 값 |
|-----------|-----|
| Role ID | nx-deploy |
| Role name | Deployment Role |
| Privileges | nx-repository-view-maven2-*-add, nx-repository-view-maven2-*-edit, nx-repository-view-maven2-*-read |

권한 설명:  
- **nx-repository-view-maven2-\*-read**: 저장소 조회  
- **nx-repository-view-maven2-\*-add**: 아티팩트 업로드  
- **nx-repository-view-maven2-\*-edit**: 아티팩트 수정  

### 8-3. Anonymous Access 비활성화  
설정 → Security → Anonymous Access → 체크 해제  

이렇게 하면 인증 없이는 저장소에 접근할 수 없습니다.  

## 9단계: 백업 및 복구  

### 9-1. 백업 대상  
Nexus의 모든 데이터는 `sonatype-work` 디렉토리에 저장됩니다.  

```
/opt/sonatype-work/nexus3/
├── blobs/          # 실제 아티팩트 파일
├── db/             # OrientDB 데이터베이스
├── etc/            # 설정 파일
├── log/            # 로그 파일
└── tmp/            # 임시 파일
```

### 9-2. 백업 스크립트  
```bash
#!/bin/bash
# nexus-backup.sh

BACKUP_DIR="/backup/nexus"
NEXUS_DATA="/opt/sonatype-work/nexus3"
DATE=$(date +%Y%m%d_%H%M%S)

# 백업 디렉토리 생성
mkdir -p ${BACKUP_DIR}

# Nexus 서비스 중지 (일관성 있는 백업을 위해)
sudo systemctl stop nexus

# 백업 수행
tar -czvf ${BACKUP_DIR}/nexus-backup-${DATE}.tar.gz -C /opt/sonatype-work nexus3

# Nexus 서비스 시작
sudo systemctl start nexus

# 7일 이상 된 백업 삭제
find ${BACKUP_DIR} -name "nexus-backup-*.tar.gz" -mtime +7 -delete

echo "Backup completed: nexus-backup-${DATE}.tar.gz"
```

### 9-3. 복구 방법  
```bash
# Nexus 서비스 중지
sudo systemctl stop nexus

# 기존 데이터 백업 (안전을 위해)
sudo mv /opt/sonatype-work/nexus3 /opt/sonatype-work/nexus3.old

# 백업 복구
sudo tar -xzvf /backup/nexus/nexus-backup-YYYYMMDD_HHMMSS.tar.gz -C /opt/sonatype-work

# 권한 설정
sudo chown -R nexus:nexus /opt/sonatype-work/nexus3

# Nexus 서비스 시작
sudo systemctl start nexus
```

### 9-4. cron으로 자동 백업 설정  
```bash
# crontab 편집
sudo crontab -e

# 매일 새벽 3시에 백업 실행
0 3 * * * /path/to/nexus-backup.sh >> /var/log/nexus-backup.log 2>&1
```

## 트러블슈팅  

### Nexus가 시작되지 않을 때  

#### 1. 로그 확인  
```bash
# 서비스 로그
sudo journalctl -u nexus -f

# Nexus 애플리케이션 로그
tail -f /opt/sonatype-work/nexus3/log/nexus.log
```

#### 2. 권한 문제  
```bash
# 권한 재설정
sudo chown -R nexus:nexus /opt/nexus
sudo chown -R nexus:nexus /opt/sonatype-work
```

#### 3. Java 버전 확인  
```bash
java -version
# Nexus 3.x는 Java 8 또는 Java 11/17 필요
```

### 메모리 부족 시 증상 및 해결  

#### 증상  
- Nexus 웹 UI 응답 지연  
- OutOfMemoryError 로그 발생  
- 서비스가 갑자기 종료됨  

#### 해결 방법  
```bash
sudo vi /opt/nexus/bin/nexus.vmoptions
```

인스턴스 사양에 맞게 조정:  

| 인스턴스 타입 | RAM | 권장 설정 |
|---------------|-----|-----------|
| t3.medium | 4GB | -Xms1024m -Xmx1024m |
| t3.large | 8GB | -Xms2048m -Xmx2048m |
| t3.xlarge | 16GB | -Xms4096m -Xmx4096m |

설정 변경 후 서비스 재시작:  
```bash
sudo systemctl restart nexus
```

### 포트 충돌 시 변경 방법  

#### 1. 현재 포트 사용 확인  
```bash
sudo netstat -tlnp | grep 8081
# 또는
sudo ss -tlnp | grep 8081
```

#### 2. Nexus 포트 변경  
```bash
sudo vi /opt/sonatype-work/nexus3/etc/nexus.properties
```

```properties
# 포트 변경 (예: 8082)
application-port=8082
```

#### 3. 서비스 재시작 및 보안 그룹 업데이트  
```bash
sudo systemctl restart nexus
```

AWS 보안 그룹에서 새 포트(8082)를 허용하도록 인바운드 규칙을 수정합니다.  

### 디스크 공간 부족  

#### 1. 사용량 확인  
```bash
df -h /opt/sonatype-work
du -sh /opt/sonatype-work/nexus3/blobs/*
```

#### 2. Blob Store Compact (정리 작업)  
Nexus 웹 UI → 설정 → System → Tasks → Create task

| 설정 | 값 |
|------|-----|
| Type | Admin - Compact blob store |
| Blob store | default |
| Schedule | Manual 또는 Weekly |

#### 3. 오래된 스냅샷 삭제 Task  

| 설정 | 값 |
|------|-----|
| Type | Maven - Delete SNAPSHOT |
| Repository | maven-snapshots |
| Minimum snapshot count | 3 |
| Snapshot retention days | 30 |

---

Nexus Repository Manager를 통해 사내 라이브러리를 효율적으로 관리할 수 있습니다.  
