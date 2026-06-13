## 📅 Day 03: Linux 인프라 보안 강화를 위한 Root SSH 직접 로그인 차단 (Server Hardening)

### 1. Task

- **요구사항**: xFusionCorp Industries 보안 감사(Security Audit) 결과 반영 및 전체 인프라 보안 표준(Security Baseline) 준수

- **대상 인프라**: Startos Datacenter의 모든 App Server (App Server 1, 2, 3)
  - `stapp01` (App Server 1)
  - `stapp02` (App Server 2)
  - `stapp03` (App Server 3)

- **목표**: 무차별 대입 공격(Brute Force Attack) 및 권한 탈취 리스크를 원천 차단하기 위해 모든 앱 서버의 **Root 계정 직접 SSH 로그인 비활성화**

---

### 2. 해결 과정 (Troubleshooting & Action)

Stratos Datacenter의 모든 앱서버(`stapp01~03`)에 순차적으로 접근하여 아래의 인프라 요새화(Hardening) 작업을 동일하게 적용함.

#### 2-1. SSH Daemon 설정 파일(`sshd_config`) 수정

각 서버의 개별 관리자 계정으로 접속 후, 최고 권한으로 SSH 설정 파일에 접근함.

```bash
sudo vi /etc/ssh/sshd_config
```

#### 2-2. 보안 파라미터 변경

기본 설저으로 허용되어 있거나 주석 처리되어 있던 `PermitRootLogin` 설정을 명시적으로 안전한 값인 `no`로 변경하여 외부로부터의 Root 직접 접근을 차단함.

```bash
# 변경 전
#PermitRootLogin yes (또는 PermitRootLogin yes)

# 변경 후
PermitRootLogin no
```

#### 2-3. 구성 변경 사항 반영 및 서비스 재시작

수정된 런타임 구성을 프로세스에 안전하게 반영하기 위해 SSH Daemon 프로세스를 재시작함.

```bash
sudo systemctl restart sshd
```

#### 2-4. Multi-Server 반복 적용 및 무결성 검증

stapp01, stapp02, stapp03 모든 인프라 노드에 누락 없이 동일한 형상 관리 작업을 반복 수행하여 전체 클러스터의 보안 수준을 균일하게 맞춤.

---

### 3. 무엇을 배웠는가 (Takeaway)

#### 왜 Root SSH 로그인을 차단해야 하는가? (Server Hardening)

- **공격 표면(Attack Surface) 최소화**: 리눅스 서버에서 `root`라는 사용자 이름은 전 세계 모든 해커와 자동화된 공격 봇(Bot)이 알고 있는 가장 대중적인 타겟이다. Root 접속을 열어두는 것은 공격자에게 패스워드만 맞추면 서버 전체를 장악할 수 있는 빌미를 제공한다.

- **책임 추적성(Accountability) 확보**: 실무에서는 엔지니어 개인 계정으로 먼저 로그인(Identification)한 뒤, 필요한 경우에만 `sudo` 권한을 획득(Autorization)하여 작업해야 한다. 그래야 `/var/log/secure`나 Audit 로그에 "누가, 언제, 어떤 관리자 명령을 내렸는지" 명확한 감사 추적이 가능해진다.

- **향후 인프라 확장성**: 현재는 3대의 서버에 수동으로 반영했지만, 서버가 수십~수백 대 수준으로 늘어날 경우 이러한 Baseline 관리는 Ansible, SaltStack 같은 형상 관리(CM) 도구를 통해 코드로 자동화(IaC)해야 함을 인지.
