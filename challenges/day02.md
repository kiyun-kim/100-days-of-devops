## 📅 Day 02: Linux 임시 사용자 계정 생성 및 만료일 설정

### 1. Task

- **요구사항**: Nautilus 프로젝트의 임시 배정 개발자(`kareem`)를 위한 제한된 기간의 액세스 계정 생성
- **대상 서버**: Stratos Datacenter의 **App Sever 3**
- **조건**:
  - 사용자명은 소문자 `kareem`으로 생성
  - 계정 만료일을 `2027-04-15`로 설정

---

### 2. 해결 과정 (Troubleshooting & Action)

#### 2-1. App Server 3 접속

먼저 jump-host에서 제공된 인프라 정보를 바탕으로 App Server 3에 SSH 접속을 수행함.

```bash
ssh banner@app-server-3
```

#### 2-2. 만료일 지정 유저 생성

리눅스의 `useradd` 명령어와 `-e (expire)` 옵션을 사용하여 계정 생성과 동시에 만료일을 지정함. (보안 및 관리 효율성을 위해 실무에서 자주 쓰이는 방식)

```bash
sudo useradd -e 2027-04-15 kareem
```

#### 2-3. 계정 생성 및 만료일 상태 검증

`chage -l` 명령어를 통해 유저의 패스워드 및 계정 만료 속성이 정상적으로 적용되었는지 확인함.

```bash
sudo chage -l kareem
```

[검증 출력 결과]

```bash
Last password change					: May 30, 2026
Password expires					: never
Password inactive					: never
Account expires						: Apr 15, 2027
Minimum number of days between password change		: 0
Maximum number of days between password change		: 99999
Number of days of warning before password expires	: 7
```

### 3. 무엇을 배웠는가 (Takeaway)

#### 최소 권한 원칙과 계정 수명 주기 관리 (Identity Lifecycle Management)

- **보안 컴플라이언스**: 실무 인프라 환경에서 외부 협력업체, 인턴, 혹은 임시 프로젝트 인력에게 계정을 발급할 때 만료일을 지정하지 않는 것은 잠재적인 보안 취약점(Backdoor)이 됩니다. 프로젝트 종료 후 방치된 휴면 계정은 권한 탈취의 표적이 되기 때문입니다.
- **실무 best practice**: 인프라 엔지니어는 계정 생성 시점에 명확한 수명 주기(Lifecycle)를 부여하여 자동 격리되도록 설계해야 하며, 이번 과제는 이를 위한 리눅스 핵심 명령어(useradd -e, chage)의 활용 능력을 검증하는 좋은 엔지니어링 연습이었습니다.
