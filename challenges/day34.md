## 📅 Day 34: Git Hook

### 1. Task

- **요구사항**: Stratos 데이터 센터의 Storage 서버 내 원격 저장소(`/opt/apps.git`)에 푸시(Push) 이벤트 발생 시 자동으로 날짜 기반 릴리스 태그를 생성하는 `post-update` 훅(Hook)을 구성하고, 로컬 저장소에서 브랜치를 병합하여 훅의 정상 동작을 검증해야 합니다.

- **목표**:
  1. Storage 서버(`ststor01`)에 `natasha` 계정으로 접속하여 원격 저장소(`/opt/apps.git/hooks`)에 `post-update` 훅 스크립트를 작성하고 실행(Execute) 권한을 부여합니다.
  2. 로컬에 복제된 프로젝트 저장소 경로(`/usr/src/kodekloudrepos/apps`)로 이동하여 `master` 브랜치로 전환합니다.
  3. `feature` 브랜치의 변경 사항을 `master` 브랜치로 병합(Merge)합니다.
  4. 원격 저장소로 병합 내역을 푸시하여, 훅이 트리거되고 오늘 날짜의 릴리스 태그(예: `release-2026-07-03`)가 자동 생성되는지 확인합니다.

---

### 2. Workflow

```text
[Storage Server (ststor01)]
/opt/apps.git/hooks/post-update (Server-side Hook 구성)
       │
[Local Workspace: /usr/src/kodekloudrepos/apps]
feature ──(git merge)──> master
       │
       ▼
(git push origin master)
       │
       ▼
[Remote Repository: /opt/apps.git]
Push 감지 ──> post-update 훅 트리거 ──> release-YYYY-MM-DD 태그 자동 생성

```

---

### 3. 해결 과정 (Action)

#### 3-1. Server-side Hook 스크립트 작성 및 권한 부여

클라이언트(로컬)가 아닌 원격 저장소(Bare Repository)의 훅 디렉터리에 접근하여, 푸시 이벤트 직후 동작할 스크립트를 작성합니다.

```bash
# Storage 서버로 natasha 계정을 사용하여 SSH 접속
ssh natasha@ststor01

# 원격 저장소의 hooks 디렉터리로 이동
cd /opt/apps.git/hooks/

# post-update 훅 파일 생성 및 vi 편집기 진입
vi post-update

# [vi 편집기 내부 스크립트 작성 - i를 눌러 입력 모드 진입 후 아래 내용 작성]
# #!/bin/bash
#
# # 푸시된 레퍼런스($@) 중 master 브랜치가 포함되어 있는지 검증
# if echo "$@" | grep -q "refs/heads/master"; then
#     # 오늘 날짜를 YYYY-MM-DD 형식으로 변수 할당
#     TAG_NAME="release-$(date +'%Y-%m-%d')"
#
#     # 해당 변수명으로 Git 태그 생성
#     git tag $TAG_NAME
# fi
# [작성 완료 후 ESC -> :wq 로 저장 후 종료]

# 스크립트가 정상적으로 동작할 수 있도록 실행(x) 권한 부여
chmod +x post-update

```

#### 3-2. 로컬 저장소 브랜치 병합 (Merge)

실제 프로젝트 파일들이 클론되어 있는 올바른 하위 디렉터리로 이동하여 명세서에서 요구한 브랜치 병합 작업을 수행합니다.

```bash
# 실제 Git 저장소가 위치한 apps 하위 디렉터리로 이동
cd /usr/src/kodekloudrepos/apps

# 병합의 기준이 될 master 브랜치로 전환
git checkout master

# feature 브랜치의 변경 사항을 master 브랜치로 병합 (Fast-forward)
git merge feature

```

#### 3-3. 변경 사항 푸시 및 자동화 동작 검증

병합이 완료된 메인 라인을 원격 저장소로 밀어 넣고, 앞서 작성한 `post-update` 훅이 정상적으로 발동했는지 확인합니다.

```bash
# 원격 저장소(origin)의 master 브랜치로 최종 푸시
git push origin master

# 원격 저장소에 오늘 날짜의 릴리스 태그(release-202X-XX-XX)가 자동 생성되었는지 검증
git ls-remote --tags origin

```

---

### 4. 핵심 개념 정리

- **Git Hooks (Server-side)**: Git 이벤트(커밋, 푸시 등)가 발생할 때 특정 스크립트를 자동으로 실행하도록 구성하는 기능입니다. 이번 실습에서 사용한 `post-update`는 서버 측(Remote)에서 클라이언트의 푸시를 성공적으로 받은 직후에 백그라운드에서 실행되는 훅입니다.

  > 💡 우체국(원격 저장소)에 새로운 택배(Push)가 도착했을 때, 시스템이 이를 감지하자마자 자동으로 도착 날짜 도장(Tag)을 찍어 분류하는 '자동 날인 시스템'과 같습니다.

- **Fast-forward Merge**: 병합 대상 브랜치(`feature`)가 기준 브랜치(`master`) 이후의 커밋만 가지고 있을 때, 새로운 병합 커밋(Merge Commit)을 만들지 않고 기준 브랜치의 포인터를 최신 커밋으로 단순히 이동시키는 깔끔한 병합 방식입니다.
  > 💡 이어달리기에서 다음 주자가 바통을 건네받고 기존 트랙을 그대로 이어서 달리는 것과 같습니다. 동선이 갈라지지 않고 하나의 직선을 유지합니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **배포 날마다 반복되는 '태그 지옥' 탈출기**: 사실 매번 릴리스할 때마다 날짜 확인하고 수동으로 태그를 따는 건 무척 번거로운 일입니다. 바쁜 배포 당일에 누군가 오타라도 내면 버전 관리가 꼬이기 십상이죠. 이번 Git Hook 설정을 통해 "사람의 기억력에 의존하지 않는 시스템"의 중요성을 다시금 깨달았습니다.

- **서버가 나 대신 일하게 만드는 법**: 로컬에서 아무리 조심해도 원격 서버(Bare Repo) 레벨에서 자동화가 되어 있지 않으면 결국 구멍이 생깁니다. post-update 훅처럼 서버 측 도구를 활용해 팀 전체의 작업 표준을 강제하는 것이 진정한 의미의 'DevOps 엔지니어링'임을 배웠습니다. 앞으로는 "내가 없어도 시스템이 알아서 굴러가게" 만드는 아키텍처를 설계하는 데 더 집중하게 될 것 같습니다.
