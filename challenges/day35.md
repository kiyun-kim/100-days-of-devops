## 📅 Day 35: Install Docker Packages and Start Docker Service

### 1. Task

- **요구사항**: Nautilus 애플리케이션 개발 팀의 요구에 따라 기존 레거시 환경을 컨테이너 환경으로 마이그레이션하기 위한 사전 테스트로, 지정된 애플리케이션 서버에 Docker 엔진과 Compose 플러그인을 설치하고 백그라운드 데몬을 활성화하여 컨테이너 런타임 인프라를 구축합니다.

- **목표**:
  1. App Server 3(`stapp03`, 계정: `banner`)에 안전하게 접속합니다.
  2. 서버에 `docker-ce` 및 `docker-compose-plugin` 등 필수 패키지들을 설치합니다.
  3. 설치된 `docker` 서비스를 시작(Start)하고 시스템 재부팅 시에도 데몬이 자동 실행되도록 서비스 활성화(Enable) 처리를 완료합니다.

---

### 2. Workflow

```text
[Client (jump-host)] ──(SSH 접속)──> [App Server 3 (stapp03)]
                                            │
                              (yum install docker-ce docker-compose-plugin)
                                            │
                                            ▼
                              [Docker Engine & Plugin 설치 완료]
                                            │
                              (systemctl start & enable docker)
                                            │
                                            ▼
                              [Docker Daemon Active (running)]

```

---

### 3. 해결 과정 (Action)

#### 3-1. 서버 접속 및 런타임 환경 진입

점프 호스트를 거쳐 인프라 명세서에 정의된 App Server 3으로 접속합니다.

```bash
# App Server 3 접속 (계정: banner)
ssh banner@stapp03
# 비밀번호 입력: BigGr33n

```

#### 3-2. Docker 공식 패키지 및 플러그인 설치

운영 체제의 패키지 매니저를 사용하여 Docker 커뮤니티 에디션(CE) 엔진 및 컨테이너 오케스트레이션을 돕는 Compose 플러그인을 설치합니다.

```bash
# Docker CE, CLI, Containerd 및 Compose 플러그인 등 필수 의존성 일괄 설치
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

```

#### 3-3. Docker 데몬 시작 및 영구 서비스 등록

컨테이너 런타임을 관장하는 백그라운드 프로세스를 가동하고, 장애나 패치 작업으로 인한 서버 재부팅 시 데몬이 누락되지 않도록 시스템 서비스에 영구 등록합니다.

```bash
# Docker 서비스 즉시 시작
sudo systemctl start docker

# 부팅 시 서비스 자동 실행 등록 (중요)
sudo systemctl enable docker

# 서비스 정상 동작 상태 점검
sudo systemctl status docker

```

---

### 4. 핵심 개념 정리

- **Docker Daemon (dockerd)**: 컨테이너의 생성, 실행, 배포 등 Docker의 전반적인 라이프사이클을 백그라운드에서 상시 관리하는 핵심 프로세스입니다.

  > 💡 항구(서버)에서 수출입 컨테이너(애플리케이션)를 언제 싣고 내릴지, 어디에 보관할지를 총괄 통제하는 '항만 관리소장'과 같습니다.

- **systemctl enable**: 시스템이 재부팅될 때 특정 서비스(데몬)가 자동으로 시작되도록 리눅스의 `systemd` 초기화 프로세스에 심볼릭 링크를 생성하여 등록하는 명령어입니다.
  > 💡 매일 아침 출근할 때마다 수동으로 켜야 했던 사무실 메인 전원 스위치를, 정해진 시간에 자동으로 켜지도록 '타이머 콘센트'에 연결해 두는 것과 같습니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **실무적 페인 포인트(Pain Point)**: 단순히 패키지만 덜렁 설치해 두고 `systemctl enable` 처리를 누락하는 것은 주니어들이 실무에서 흔히 하는 치명적인 실수입니다. 만약 야간에 정기 OS 패치 작업이나 예기치 못한 하드웨어 장애로 인스턴스가 재부팅되었을 때, 데몬이 자동으로 올라오지 않으면 그 위에 띄워져 있던 수십 개의 프로덕션 컨테이너가 모두 다운된 상태로 방치됩니다. "서버 재부팅 = 서비스 영구 장애"라는 끔찍한 공식을 만들지 않으려면 데몬 서비스 등록을 숨 쉬듯 당연하게 처리하는 습관이 필수적임을 실감했습니다.

- **회고(Retrospective)**: 그저 `yum install`을 치는 단순한 과정 같지만, 이 작업은 기존의 레거시(LAMP 스택 등)를 격리된 컨테이너 환경으로 탈바꿈하기 위한 가장 중요한 첫 단추입니다. 운영 체제의 레이어와 컨테이너 런타임 간의 의존성을 직접 세팅해 보며, 향후 애플리케이션 개발 팀이 "제 로컬에서는 되는데 서버 환경이 달라서 안 돌아가요"라고 말할 일 없는 일관된 인프라를 제공할 수 있다는 인프라 아키텍트로서의 기초 역량을 체득했습니다.
