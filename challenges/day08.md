## 📅 Day 08: Ansible 설치 및 환경 구성

### 1. Task

- **요구사항**: 향후 다중 분산 서버 노드들의 구성 관리(Configuration Management) 및 자동화 배포를 원격에서 일괄 제어하기 위해, Control Node 역할을 수행할 Jump Host에 Ansible 패키지를 설치하고 구동 환경을 검증

- **대상**: 인프라 제어 노드 (`Jump Host`)

- **목표**: 엔터프라이즈 리눅스용 확장 패키지 저장소(EPEL)를 활성화하고, 패키지 매니저를 통해 Ansible을 안정적으로 빌드한 뒤 시스템 런타임 버전을 확인하여 자동화 아키텍처의 기반을 마련

### 2. Workflow

---

```text
[Jump Host (Control Node)]
  1. 외부 확장 저장소 메타데이터 활성화 (EPEL Repository 설치)
        │
        ▼
  2. 최신 의존성 및 패키지 인덱스 업데이트 (yum check-update)
        │
        ▼
  3. 앤서블 코어 패키지 빌드 (yum install ansible)
        │
        ▼
  4. 앤서블 엔진 버전 및 파이썬 런타임 환경 검증 (ansible --version)
```

---

### 3. 해결 과정 (Troubleshooting & Action)

#### 3-1. CentOS/RHEL 환경에서 표준 저장소에 없는 패키지를 가져오기 위해 EPEL(Extra Packages for Enterprise Linux) 저장소 활성화하기

```bash
sudo yum install -y epel-release
```

#### 3-2. 새로 추가된 저장소의 최신 패키지 리스트 인덱스를 로컬 시스템에 동기화하기

```bash
sudo yum makecache
```

#### 3-3. 패키지 매니저를 사용하여 의존성 파일(Python 라이브러리 등)과 함께 ansible 인프라 코어 엔진 설치하기

```bash
sudo yum install -y ansible
```

#### 3-4. 설치가 완료된 ansible 바이너리 버전을 체크하여 정상 구동 상태 및 실행 파일 경로 검증하기

```bash
ansible --version

# Output:
ansible [core 2.x.x]
   executable location = /usr/bin/ansible
   python version = 3.x.x ...
```

---

### 4. 무엇을 배웠는가 (Takeaway)

- **오픈소스 구성 관리 도구(Configuration Management)의 도입**: Ansible은 Agentless(대상 서버에 별도의 감시 프로그램을 심지 않는 방식) 아키텍처를 채택하여, 오직 SSH 통신망만 확보되면 수백 대의 서버를 단 한 곳의 Control Node(Jump Host)에서 동시에 제어할 수 있는 효율적인 인프라 자동화의 표준임을 인지했습니다.

- **EPEL 저장소의 역할과 필요성**: 리눅스 배포판의 기본 저장소(Repository)는 순수 안정성 위주의 패키지만 보관하므로, Ansible 같은 최신 인프라 엔지니어링 도구를 빌드하려면 기업용 검증 확장 저장소인 `epel-release`를 선행적으로 마운트하고 인덱스를 갱신해야 호환성 충돌 없이 안전하게 패키지를 확보할 수 있다는 점을 체득했습니다.
