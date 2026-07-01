## 📅 Day 28: Git Cherry Pick

### 1. Task

- **요구사항**: Nautilus 애플리케이션 개발 팀의 형상 관리 전략에 따라, 진행 중인 `feature` 브랜치 전체를 병합하지 않고 배포에 필요한 단 하나의 특정 커밋만 선별하여 `master` 브랜치에 안전하게 병합(Cherry-pick)한 뒤 원격 저장소에 동기화합니다.

- **목표**:
  1. Storage 서버(`ststor01`)의 정확한 로컬 워킹 트리 저장소 경로인 `/usr/src/kodekloudrepos/demo`로 이동합니다.
  2. `feature` 브랜치의 커밋 로그를 추적하여 메시지가 `Update info.txt`인 커밋의 고유 해시 ID를 식별합니다.
  3. `master` 브랜치로 전환한 후 해당 커밋만 선택적으로 병합합니다.
  4. 변경된 형상 이력을 원격 원본 저장소(`/opt/demo.git`)의 `master` 브랜치로 푸시(Push) 배포합니다.

---

### 2. Workflow

```text
[feature 브랜치] ── (9eae1db: Update info.txt) ── (다른 진행 중인 커밋들)
                         │
                         │ [git cherry-pick 9eae1db] -> 오직 이 커밋만 복사
                         ▼
[master 브랜치] ───► [선택 병합 완료] ───► [git push origin master]

```

---

### 3. 해결 과정 (Action)

#### 3-1. 저장소 진입 및 타겟 커밋 식별

상위 디렉토리 수준이 아닌, 실제 `.git` 메타데이터가 파싱되는 프로젝트 하위 워킹 트리 경로로 명확히 진입하여 작업을 시작합니다.

```bash
# 1. 실제 프로젝트 워킹 트리인 demo 폴더 내부로 진입합니다.
cd /usr/src/kodekloudrepos/demo

# 2. feature 브랜치의 히스토리를 한 줄씩 출력하여 'Update info.txt' 커밋의 해시값(9eae1db)을 추출합니다.
sudo git log feature --oneline

```

#### 3-2. master 브랜치 전환 및 체리 픽(Cherry-pick) 실행

```bash
# 1. 피처를 적용할 대상 선상인 master 브랜치로 명시적 체크아웃을 수행합니다.
sudo git checkout master

# 2. feature 브랜치에서 확보한 고유 커밋 해시(9eae1db)를 master 브랜치에 매핑 적용합니다.
sudo git cherry-pick 9eae1db

```

#### 3-3. 원격 저장소 푸시 및 무결성 검증

로컬에서 체리 픽 완료된 최신 형상을 원격 백본 저장소(origin)로 영구 전송하여 배포를 완수합니다.

```bash
# 1. 업스트림 master 브랜치로 변경된 로컬 커밋 이력을 푸시합니다.
sudo git push origin master

# 2. 이력이 정상적으로 정착했는지 히스토리 로그와 작업 트리 상태를 최종 교차 검증합니다.
sudo git log --oneline
sudo git status

```

---

### 4. 핵심 개념 정리

- **Git Cherry Pick**: 다른 브랜치에 있는 수많은 커밋들 중, 전체 병합(Merge)을 수행하지 않고 오직 원하는 특정 커밋 하나만을 선택하여 현재 브랜치에 마이그레이션 적용하는 형상 관리 기술입니다.

  > 💡 마트에서 파는 "종합 과일 바구니(feature 브랜치의 전체 커밋)"를 통째로 사 오는 것이 아니라, 내가 지금 바로 먹고 싶은 "체리 알맹이 하나(Update info.txt 커밋)"만 집게로 쏙 골라내어 "내 접시(master 브랜치)"에 담는 것과 같습니다.

- **Bare Repository vs Working Tree**: 베어 저장소는 실제 소스 파일 없이 오직 Git의 버전 관리 메타데이터와 히스토리(`.git` 폴더 내부 구성 요소)만 가지는 중앙 저장소 구조입니다. 반면 워킹 트리는 개발자가 직접 파일을 수정하고 눈으로 볼 수 있는 실제 소스 코드가 풀려 있는 로컬 작업 공간입니다.

  > 💡 베어 저장소는 서류의 이력과 도장만 찍어 보관하는 공장 내부의 "중앙 문서 보관 창고(.git)"이고, 워킹 트리는 직원이 직접 책상 위에 서류를 펼쳐놓고 글을 쓰는 "실제 집무실"과 같습니다. 글을 쓰거나 수정(Checkout, Cherry-pick)하는 작업은 반드시 창고 내부가 아닌 집무실(워킹 트리 경로)에서 수행해야 합니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **실무 관점의 의의**: 실제 프로덕션 환경이나 기업용 MLOps 배포 아키텍처에서는 특정 기능이 아직 완전히 검증되지 않아 `feature` 브랜치 전체를 메인에 합칠 수 없는 상황이 빈번합니다. 이때 긴급 패치나 특정 명세 파일만 운영 환경에 선반영해야 하는 경우 `git cherry-pick`은 파이프라인의 안전성을 담보하는 최고의 선택적 배포 수단이 됩니다.
