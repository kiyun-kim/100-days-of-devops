## 📅 Day 31: Git Stash

### 1. Task

- **요구사항**: Stratos 데이터 센터의 Storage 서버 내에서 개발자가 임시 저장해 둔 특정 작업 내역(`stash@{1}`)을 안전하게 복원하고, 이를 로컬 저장소의 활성화된 브랜치에 반영하여 원격 레포지토리로 안전하게 통합해야 합니다.

- **목표**:
  1. Storage 서버(`ststor01`)에 `natasha` 계정으로 접속 후 지정된 Git 저장소(`/usr/src/kodekloudrepos/official`)로 이동합니다.
  2. 현재 저장된 스태시 목록 중 `stash@{1}`의 변경 사항을 검증하고 로컬 워킹 디렉터리에 적용합니다.
  3. 로컬의 형상 관리 상태를 점검하여 소유권 분쟁(Dubious ownership) 문제를 해결하고 변경 사항을 스테이징합니다.
  4. 로컬 `master` 브랜치에 커밋을 생성한 후, 추적 중인 원격 저장소(`origin`)의 동일 브랜치로 데이터를 완벽히 푸시(`Push`)합니다.

---

### 2. Workflow

```text
[Local Workspace] ──(git stash apply)──> [Working Directory (welcome.txt)]
       │                                              │
(git add/commit)                                 (sudo 권한 적용)
       │                                              │
       ▼                                              ▼
[Local Repository (master)] ──(git push origin master)──> [Remote Repository (/opt/official.git)]

```

---

### 3. 해결 과정 (Action)

#### 3-1. 저장소 이동 및 Stash 내역 검증

Storage 서버에 접속한 후, 해당 Git 레포지토리로 이동하여 임시 저장된 파일의 세부 변경 사항을 사전에 확인합니다.

```bash
# 지정된 Git 저장소 디렉터리로 이동
cd /usr/src/kodekloudrepos/official/

# 안전한 실행을 위해 stash 리스트 및 stash@{1}의 상세 Diff 내역 확인
sudo git stash list
sudo git stash show -p stash@{1}

```

#### 3-2. 작업 내역 복원 및 디렉터리 예외 처리

`stash@{1}`에 들어있던 `welcome.txt` 파일의 변경 사항을 적용합니다. 이 과정에서 리눅스 파일 시스템 소유권 관련 Git 보안 경고가 발생할 수 있으므로 안전한 디렉터리(safe.directory) 설정을 추가한 후 상태를 재점검합니다.

```bash
# stash@{1}의 변경 항목을 워킹 디렉터리에 적용
sudo git stash apply stash@{1}

# Git Dubious Ownership 보안 경고 발생 시 해당 저장소를 안전한 디렉터리로 전역 등록
sudo git config --global --add safe.directory /usr/src/kodekloudrepos/official

# 저장소 상태 및 welcome.txt의 스테이징 대기 상태 확인
sudo git status

```

#### 3-3. 스테이징, 커밋 및 원격 저장소 최종 푸시

수정 사항을 로컬 저장소에 반영하고, 원격 브랜치 명세(`master`)를 명확히 타게팅하여 푸시를 완료합니다.

```bash
# 변경된 welcome.txt 파일을 스테이징 영역에 추가
sudo git add .

# 영문 요구사항에 부합하는 명확한 커밋 메시지 작성 및 커밋 실행
sudo git commit -m "Restore and apply in-progress changes from stash@{1}"

# 현재 로컬 브랜치(master)와 매핑되는 원격 저장소(origin master)로 최종 푸시 실행
sudo git push origin master

```

---

### 4. 핵심 개념 정리

- **Git Stash**: 현재 워킹 디렉터리에서 작업 중이던 변경 사항들을 커밋하지 않고, 내부 스택에 임시로 저장해 둘 수 있도록 지원하는 Git의 빌트인 하위 유틸리티입니다.

  > 💡 **비유하자면**, 요리사가 메인 요리(Commit)를 완성하기 전에 급하게 다른 주문을 처리해야 해서, 지금까지 다듬어 둔 재료들을 잠시 '임시 보관용 선반(Stash)'에 라벨을 붙여 얹어두는 것과 같습니다.

- **Safe Directory**: Git 2.35.2 이후 도입된 보안 기능으로, 현재 로그인한 사용자 계정과 타겟 디렉터리의 소유주가 다를 경우 발생할 수 있는 악의적인 코드 실행 취약점을 차단하기 위한 예외 방어 메커니즘입니다.

  > 💡 **비유하자면**, 건물의 보안 요원이 다른 사람의 이름으로 등록된 비밀 서재(디렉터리)에 접근하려는 관리자에게 "잠시만요, 출입증 명부에 이 방을 안전 구역으로 등록하셔야만 열어드릴 수 있습니다"라며 제지하는 출입 통제 절차와 같습니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **실무 관점의 의의**: 실제 기업의 공용 인프라 서버나 CI/CD 배포 에이전트 환경에서는 여러 엔지니어와 시스템 프로세스가 동일한 파일 디렉터리에 접근하곤 합니다. 이때 발생할 수 있는 `dubious ownership`과 같은 리눅스 권한 불일치 문제를 `git config` 레이어에서 우아하게 격리 및 통제하는 역량은 인프라 자동화의 안정성을 보장하는 데 매우 중요합니다.

- **엔지니어링 인사이트**: 무조건적인 표준 명령어(`git push origin main`) 맹신을 지양하고, 시스템 내부의 브랜치 명세(`master`)와 Refspec 매핑 에러 로그를 정확히 해석하여 동적으로 대응하는 근본 원인 분석(RCA)의 중요성을 다시 한번 학습할 수 있었습니다.
