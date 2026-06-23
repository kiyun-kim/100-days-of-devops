## 📅 Day 25: Git Merge Branches

### 1. Task

- **요구사항**: Nautilus 애플리케이션 개발 팀의 신규 데이터센터 관리 기능을 소스 코드 라인에 안전하게 편입하기 위해, 로컬 저장소에서 독립 브랜치를 생성하고 파일 추가 및 머지(Merge)를 수행한 후 최종 형상 관리 본진인 원격 저장소(origin)로 동기화 배포합니다.

- **목표**:
  1. Storage 서버(`ststor01`)의 로컬 저장소 경로(`/usr/src/kodekloudrepos/official`)로 이동합니다.
  2. `master` 브랜치 기반으로 `datacenter`라는 신규 피처 브랜치를 생성합니다.
  3. 임시 경로(`/tmp/index.html`)의 파일을 저장소 루트로 가져와 `datacenter` 브랜치에 커밋합니다.
  4. 해당 변경 사항을 메인 줄기(`master`)에 병합(Merge)한 뒤, 두 브랜치 구조를 원격 원본 저장소(`origin`)에 모두 전송합니다.

---

### 2. Workflow

```text
[master 브랜치] ──> [datacenter 브랜치 분기] ──> [index.html 복사 & 커밋]
      │                                                     │
      └─────── [master 복귀 후 Fast-forward 머지] <─────────┘
                                │
                                v
             [git push origin master & datacenter] ──> [원격 저장소 반영 완료]

```

---

### 3. 해결 과정 (Action)

#### 3-1. 저장소 진입 및 브랜치 분기 (Branching)

다중 사용자 인프라 환경의 소유권 경고(`dubious ownership`)를 차단하기 위해 `sudo` 권한을 활용하여 안전하게 기준점을 잡고 분기합니다.

```bash
# 로컬 Git 저장소 디렉토리로 이동합니다.
cd /usr/src/kodekloudrepos/official/

# 원본 최신 형상 위치인 master 브랜치로 명시적 체크아웃을 수행합니다.
sudo git checkout master

# master 분기점으로부터 datacenter 피처 브랜치를 생성하고 즉시 작업 공간을 전환합니다.
sudo git checkout -b datacenter

```

#### 3-2. 소스 파일 이주 및 로컬 커밋 (Commit)

지정된 임시 소스 코드를 작업 디렉토리 내부로 가져와 인덱싱 처리 및 스냅샷을 생성합니다.

```bash
# Storage 서버 내 /tmp/index.html 파일을 현재 Git 저장소 루트(.) 경로로 복사합니다.
sudo cp /tmp/index.html .

# 스테이징 영역에 파일을 추가하여 Git 추적 대상으로 등록합니다.
sudo git add index.html

# 신규 피처 브랜치에 이력을 기록하기 위해 커밋 명세를 생성합니다.
sudo git commit -m "Add index.html to datacenter branch"

```

#### 3-3. 로컬 병합 및 원격 동기화 (Merge & Push)

개별 분기된 파이프라인 코드를 메인 줄기에 병합하고, 로컬의 물리적 이력을 원격 원본 저장소(origin)로 영구 전송합니다.

```bash
# 메인 통합 라인인 master 브랜치로 다시 복귀합니다.
sudo git checkout master

# datacenter 브랜치의 스냅샷 이력을 master 브랜치로 Fast-forward 병합합니다.
sudo git merge datacenter

# 통합 완료된 master 브랜치의 변경 사항을 원격(origin) 저장소로 푸시합니다.
sudo git push origin master

# 개발 추적성 유지를 위해 생성했던 datacenter 피처 브랜치의 이력도 원격 저장소로 함께 푸시합니다.
sudo git push origin datacenter

```

---

### 4. 핵심 개념 정리

- **Git Merge**: 분기된 서로 다른 브랜치의 작업 이력을 하나로 통합하는 작업입니다. 기준 브랜치에 다른 브랜치의 커밋 내역을 병합하여 개발 파이프라인을 최종 완성시키는 형상 관리의 핵심 기능입니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **원격 저장소(Upstream) 생명주기 제어 역량**: 단순 로컬 환경에서의 단방향 파일 관리를 넘어, 분기(Branch) ➡️ 변경(Commit) ➡️ 병합(Merge) ➡️ 배포(Push)로 이어지는 현대 인프라 형상 관리의 엔드투엔드(End-to-End) 파이프라인 주기 전체를 유기적이고 안전하게 제어하는 실무 아키텍처 능력을 함양했습니다.
