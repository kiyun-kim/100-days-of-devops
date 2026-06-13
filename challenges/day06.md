## 📅 Day 06: 분산 인프라 크론탭(Crontab) 스케줄러 등록

### 1. Task

---

- **요구사항**: xFusionCorp Industries 내 일상적인 반복 업무(Daily Tasks) 자동화 스크립트 배포를 위한 사전 스케줄러 환경 검증

- **대상 인프라**:Stratos DC 내 전체 분산 애플리케이션 서버 3대 (`stapp01`, `stapp02`, `stapp03`)

- **목표**: 전사 배포에 앞서, 모든 인프라 노드에 스케줄러 패키지를 표준화하여 설치하고 샘플 크론잡(Cron Job)을 등록하여 자동화 기능성 테스트를 완수

### 2. 해결 과정 (Troubleshooting & Action)

---

Stratos Datacenter의 모든 앱 서버 환경의 일관성(Consistency)을 유지하기 위해 아래 작업을 순차적으로 반복 수행함.

#### 2-1. 스케줄러 패키지(`cronie`) 설치 및 시스템 Daemon 활성화

일부 미니멀 리눅스 환경이나 컨테이너 기반 환경에서는 크론 데몬이 누락되어 있을 수 있으므로, 명시적으로 패키지를 설치하고 부팅 시 자동 시작되도록 활성화함.

```bash
# 크론 스케줄링 패키지 설치
sudo yum install -y cronie

# crond 서비스 시작 및 런타임 활성화 (부팅 시 자동 기동)
sudo systemctl start crond
sudo systemctl enable crond
```

#### 2-2. Root 권한의 크론잡(Cron Job) 구성

```bash
sudo crontab -u root -e
```

[크론 표현식 설정 내역]

```bash
# 5분마다 hello 문자열을 지정된 임시 디렉토리 파일에 Overwrite 하도록 스케줄링
*/5 * * * * echo hello > /tmp/cron_text
```

#### 2-3. 크론탭 무결성 및 리스트 검증 (Validation)

```bash
sudo crontab -u root -l

# Output:
*/5 * * * * echo hello > /tmp/cron_text
```

### 3. 무엇을 배웠는가 (Takeaway)

---

#### 크론 표현식(Cron Expression)과 자동화의 실무적 중요성

- **주기적 작업의 자동화**: 인프라 운영에서 주기적인 로그 로테이션(Log Rotation), 데이터베이스 백업(DB Backup), 시스템 리소스 모니터링 등은 수동으로 처리할 수 없습니다. 크론탭은 이러한 '반복 업무의 자동화'를 가능하게 하는 가장 기본적이고 강력한 도구임을 배웠습니다.

- **실무 환경에서의 주의점 (Redirection)**: 크론잡은 터미널이 없는 백그라운드 환경에서 실행되므로 표준 출력(`stdout`)과 표준 에러(`stderr`)가 유실되기 쉽습니다. 이번 과제처럼 `>` (리다이렉션)을 이용해 파일로 로그를 쌓거나, 실무에서는 `>> /var/log/cron_output.log 2>&1` 형태로 에러 로그까지 완벽히 트래킹하는 설계가 장애 대응력을 높인다는 점을 인지했습니다.

- **분산 인프라 구성 관리(IaC)의 필요성**: 서버 3대에 일일이 접속해서 크론탭을 설정하는 방식은 서버가 100대로 늘어나면 휴먼 에러율이 극도로 높아집니다. 주니어 단계 이후에는 이러한 다중 노드 스케줄링 설정을 Ansible의 `cron `모듈 등을 활용하여 코드로 일괄 관리(Infrastructure as Code)해야 대규모 인프라를 안정적으로 핸들링할 수 있다는 관점을 배웠습니다.
