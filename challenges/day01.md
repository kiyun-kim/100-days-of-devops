## 📅 Day 01: Linux 사용자(User) 생성 및 그룹(Group) 권한 관리

### 1. Task

- **요구사항**: Nautilus 인프라 보안 규정 및 접근 제어 정책에 따라 새로운 엔지니어 계정을 생성하고 시스템 최고 관리자 권한을 안전하게 부여할 수 있도록 설정

- **대상**: Stratos DC 내 지정된 애플리케이션 서버 노드

- **목표**: 일반 사용자 계정 생성 및 암호 설정을 완료하고, `wheel` 관리자 그룹 매핑을 통해 개별 계정 기반의 안전한 `sudo` 권한 승격 아키텍처를 검증

---

### 2. Workflow

```text
  [새로운 엔지니어 입사] ──>  1. 유저 계정 생성 (useradd)
                                     │
                                     ▼
                          2. 비밀번호 부여 (passwd)
                                     │
                                     ▼
                          3. 관리자 그룹(`wheel`)에 임명 (usermod)
                                     │
                                     ▼
  [검증 완료] <──  4. 로그인 및 sudo 권한 확인 (su - / sudo)
```

---

### 3. 해결 과정 (Troubleshooting & Action)

#### 3-1. 새로운 엔지니어 유저(예: nisha) 생성하기

```bash
sudo useradd nisha
```

#### 3-2. 새로 만든 유저의 비밀번호 설정하기

```bash
sudo passwd nisha
```

#### 3-3. ⚠️ 리눅스에서 대장 권한(sudo)을 쓸 수 있는 'wheel' 그룹에 유저 집어넣기

```bash
sudo usermod -aG wheel nisha
```

(-a: append/기존 그룹 유지하면서 추가, -G: Group/타겟 그룹 지정)

#### 3-4. 새로 만든 유저로 변신해서 관리자 권한이 잘 작동하는지 테스트하기

```bash
su - nisha
sudo whoami

# (출력 결과로 'root'가 나오면 성공!)
```

---

### 4. 무엇을 배웠는가 (Takeaway)

- **wheel의 정체**: 레드햇/CentOS 계열의 리눅스 시스템에는 `wheel`이라는 특별한 내장 그룹이 있습니다. 이 그룹에 소속된 유저들은 명령어 앞에 `sudo`(SuperUser Do - 대장 권한으로 실행해라)를 붙여서 시스템 설정 파일이나 프로그램을 제어할 수 있게 됩니다.

- **`sudo whoami` 명령어의 의미**: 그냥 `whoami`를 치면 내 본래 이름인 `nisha`가 나오지만, `sudo whoami`를 쳤을 때 `root`라고 대답한다면 컴퓨터가 나를 최고 관리자로 인정하고 권한을 정상적으로 빌려주고 있다는 뜻입니다.
