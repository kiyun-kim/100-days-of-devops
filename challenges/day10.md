## 📅 Day 10: 쉘 스크립트 기반 웹 소스 자동 아카이브 및 원격 백업

### 1. Task

---

- **요구사항**: App Server 3(`stapp03`) 내부의 특정 웹 서비스 소스 디렉토리를 주기적으로 백업하기 위해, 압축 및 원격 전송 프로세스를 수행하는 쉘 스크립트(`official_archive.sh`) 자동화 구현

- **대상**: App Server 3 (`stapp03`) 및 Storage Server (`ststor01`)

- **목표**: 오타 없는 정확한 스크립트 경로 지정, SSH 공개키 기반의 passwordless 인증 연동, 그리고 백업 대상 서버의 올바른 호스트네임 매핑을 통해 무인 자동 백업 파이프라인 구축을 완수

### 2. Workflow

---

```text
[App Server 3 (stapp03)]                       [Storage Server (ststor01)]
   (1) zip 패키지 설치                             (3) 패스워드리스 SSH 키 등록
   (2) 백업 스크립트 작성                              (.ssh/authorized_keys)
        │                                                   ▲
        ▼ (4) 스크립트 구동 (/scripts/official_archive.sh)  │
   [웹소스 압축] ─── (5) 원격 안전 복사 (scp) ──────────────┘
```

### 3. 해결 과정 (Troubleshooting & Action)

---

#### 3-1. 소스 디렉토리 압축에 필요한 `zip` 유틸리티 패키지 설치하기

```bash
sudo yum install -y zip
```

#### 3-2. 백업 스크립트 및 압축 파일 보관을 위한 전용 디렉토리 생성 및 권한 변경하기

```bash
sudo mkdir -p /scripts /archives
sudo chown banner:banner /scripts /archives
```

#### 3-3. Storage Server(`ststor01`)의 natasha 계정으로 패스워드 없이 scp 통신이 가능하도록 SSH 인증서 복사하기

```bash
ssh-copy-id natasha@ststor01
```

#### 3-4. 자동화 쉘 스크립트 파일 작성 및 실행 권한 부여하기

```bash
cat << 'EOF' > /scripts/official_archive.sh
#!/bin/bash
# 1. 대상 웹 소스 폴더를 지정된 파일명으로 압축
zip -r /archives/xfusioncorp_official.zip /var/www/html/official

# 2. 패스워드 입력 없이 Storage 서버의 백업 디렉토리로 원격 전송
scp /archives/xfusioncorp_official.zip natasha@ststor01:/archives/
EOF

# 스크립트 실행 권한 추가 및 강제 구동 테스트
chmod +x /scripts/official_archive.sh
/scripts/official_archive.sh
```

### 4. 무엇을 배웠는가 (Takeaway)

---

- **자동화의 핵심, 무인(Unattended) 인증**: 백업 스크립트처럼 사람이 없는 백그라운드에서 주기적으로 돌아가는 시스템은 비밀번호 입력 팝업창에서 프로세스가 멈추면 안 됩니다. `ssh-copy-id`를 통한 사전 키 교환으로 암호 입력 단계를 완전히 생략하는 것이 인프라 자동화 엔지니어링의 기본 원리임을 이해했습니다.
