## 📅 Day 09: MariaDB 트러블슈팅 및 서비스 복구

### 1. Task

---

- **요구사항**: Stratos DC 내 가동 중인 관계형 데이터베이스(MariaDB)의 커넥션 및 런타임 크래시 장애 원인을 분석하고 정상 서비스 상태로 즉시 복구

- **대상**: Stratos DC 내 데이터베이스 서버 노드 (`stdb01`)

- **목표**: 시스템 로그 파일(`journalctl`, `systemctl`)을 역추적하여 프로세스 중단의 근본 원인을 식별하고, 비정상 세션을 정리하거나 유실된 설정 무결성을 복구하여 데이터베이스 엔진의 안정적인 재구동을 완수

### 2. Workflow

---

```text
[Database Server (stdb01)]
  1. 서비스 상태 모니터링 (systemctl status mariadb) ──> Critical Crash 발견!
        │
        ▼
  2. 시스템 로그 역추적 및 에러 원인 분석 (journalctl -xe / /var/log/mariadb/)
        │
        ▼
  3. 장애 원인 제거 (잘못된 설정 파일 수정 또는 불법 프로세스 종료)
        │
        ▼
  4. 데이터베이스 엔진 가동 및 포트 소켓 바인딩 검증 (systemctl start / netstat)
```

### 3. 해결 과정 (Troubleshooting & Action)

---

#### 3-1. 현재 MariaDB 데이터베이스 서비스의 활성화 상태와 에러 코드 1차 조회하기

```bash
sudo systemctl status mariadb
# Output:
Active: failed (Result: exit-code) 이후 상태 확인
```

#### 3-2. `journalctl` 및 내부 에러 로그 덤프 파일을 분석하여 구동 실패를 유발한 원인(오타, 파일 권한, 포트 충돌 등) 정확히 진단하기

```bash
sudo journalctl -n 50 -u mariadb
# 또는 에러 로그 파일 직접 확인
sudo tail -n 50 /var/log/mariadb/mariadb.log
```

#### 3-3. 진단된 장애 원인(예: 설정 파일 `/etc/my.cnf` 내부의 잘못된 문법 또는 스토리지 권한 꼬임)을 수정하고 정상 데이터 구조로 복구하기

```bash
sudo vi /etc/my.cnf
# (분석된 문제 항목 수정 후 저장 및 종료)
```

#### 3-4. 장애 요인이 제거된 MariaDB 서비스를 재기동하고 시스템 포트(3306)가 정상적으로 수신(Listening) 상태인지 교차 검증하기

```bash
sudo systemctl start mariadb
sudo ss -lntp | grep 3306
```

### 4. 무엇을 배웠는가 (Takeaway)

---

- **로그 중심의 트러블슈팅(Log-Driven Troubleshooting)**: 인프라에 장애가 발생했을 때 임의로 서비스를 재시작하는 행위는 상태를 악화시킬 수 있습니다. `systemctl status`와 `journalctl`을 통해 시스템이 남긴 에러 로그를 정밀 분석하는 방법을 배웠습니다.

- **스토리지 및 설정 파일의 무결성 중요성**: 데이터베이스 엔진은 구동될 때 설정 파일(`my.cnf`)의 문법 하나, 데이터 디렉토리(`/var/lib/mysql`)의 소유권 권한 하나라도 맞지 않으면 보안 및 데이터 보호를 위해 스스로 프로세스를 차단합니다. 인프라의 모든 코드는 완벽한 무결성을 유지해야 안정성이 보장된다는 점을 체득했습니다.
