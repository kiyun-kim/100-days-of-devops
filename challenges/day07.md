## 📅 Day 07: SSH Key 기반 Passwordless 인증 체계 구축

### 1. Task

---

- **요구사항**: 보안 강화 및 배포 자동화 스크립트 운영을 위해 Jump Host에서 App Server 1(`stapp01`)로 접속 시, 매번 비밀번호를 입력하지 않고 안전하게 다이렉트로 로그인할 수 있는 SSH Key 인증 환경 구현

- **대상**: `Jump Host` 및 Stratos DC 내 `App Server 1 (stapp01 - tony 유저 계정)`

- **목표**: Jump Host에서 비대칭 암호화 방식의 SSH 키 쌍(Key Pair)을 생성하고, 이를 App Server 1에 안전하게 복사/등록하여 패스워드 입력 절차 없이 단 한 번의 명령어로 원격 세션 진입을 완수

### 2. Workflow

---

```text
[Jump Host (thor)]                                  [App Server 1 (stapp01: tony)]
  1. SSH 키 쌍 생성 (ssh-keygen)
        │ (id_rsa.pub /공개키 생성)
        ▼
  2. 원격 노드로 공개키 전송 (ssh-copy-id) ── 비밀번호 인증 ──> 3. 비밀 금고방에 공개키 저장
                                                                  (.ssh/authorized_keys)
                                                                            │
                                                                            ▼
  4. 최종 패스워드리스 원격 접속 검증 (ssh tony@stapp01) <───────────────── 인증 통과!
```

### 3. 해결 과정 (Troubleshooting & Action)

---

#### 3-1. Jump Host 터미널에서 인증에 사용할 SSH 키 쌍(비밀키 & 공개키) 생성하기

```bash
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
```

#### 3-2. 생성된 공개키(`id_rsa.pub`)를 App Server 1의 `tony` 유저 계정으로 안전하게 전송 및 등록하기

```bash
ssh-copy-id tony@stapp01
```

#### 3-3. Jump Host에서 비밀번호 요구 팝업 없이 App Server 1에 즉시 로그인이 가능한지 최종 검증하기

```bash
ssh tony@stapp01
```

#### 3-4. 원격 접속 성공 후 호스트네임 및 세션 유저 정보 크로스 체크하기

```bash
whoami && hostname

# Output:
# tony
# stapp01
```

### 4. 무엇을 배웠는가 (Takeaway)

---

- **비대칭 암호화(Asymmetric Encryption) 기반 인증**: SSH Key 인증은 자물쇠 역할을 하는 '공개키(Public Key)'와 열쇠 역할을 하는 '비밀키(Private Key)'의 쌍으로 이루어집니다. 비밀번호 방식보다 크래킹 위협에 훨씬 안전하며, 사람이 개입할 수 없는 백그라운드 자동화 스크립트(Ansible, Jenkins 등)를 설계할 때 반드시 선행되어야 하는 필수 인프라 인프라 아키텍처임을 인지했습니다.

- **`.ssh/authorized_keys` 파일의 무결성**: `ssh-copy-id` 명령어를 실행하면 원격 서버의 홈 디렉토리 내에 `.ssh` 폴더와 `authorized_keys` 파일이 자동으로 생성되거나 누적됩니다. 이 파일의 권한 규칙(`700`, `600`)이 엄격하게 지켜지지 않으면 리눅스 커널이 보안상 접속을 거부하므로, 표준 유틸리티를 통한 자동 등록이 휴먼 에러를 줄이는 가장 안전한 경로임을 배웠습니다.
