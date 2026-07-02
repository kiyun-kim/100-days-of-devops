## 📅 Day 33: Resolve Git Merge Conflicts

### 1. Task

- **요구사항**: 다수의 작업자(Sarah와 Max)가 동일한 파일에 변경 사항을 발생시켰을 때 발생하는 원격 저장소와의 병합 충돌(Merge Conflict) 현상을 로컬 환경에서 수동으로 정제하고, 요구된 텍스트 정합성(오타 교정 및 라인 수 유지)을 완벽하게 맞추어 원격 레포지토리에 동기화해야 합니다.

- **목표**:
  1. Storage 서버(`ststor01`)에 `max` 계정으로 SSH 접속하여 작업 디렉터리(`/home/max/story-blog`)로 이동합니다.
  2. 원격 저장소(`origin/master`)의 최신 이력을 로컬에 병합(Pull)하여 파일 충돌 상황을 유도 및 확인합니다.
  3. `story-index.txt` 내부의 Git 충돌 마커를 제거하고, 4개의 고유한 스토리 제목이 중복 없이 1줄씩 유지되도록 정렬합니다.
  4. 파일 내 `The Lion and the Mooose` 오타를 `The Lion and the Mouse`로 정확하게 교정합니다.
  5. 수정된 파일을 스테이징 및 커밋(`6d3015e`)한 후, 원격 저장소로 성공적으로 푸시합니다.

---

### 2. Workflow

```text
[Remote: origin/master (Sarah's Commit)]      [Local: master (Max's Commit)]
                        \                                 /
                         \                               /
                          +--- (git pull origin master)
                                        |
                          [CONFLICT: story-index.txt 발생]
                                        |
                          (vi 수동 편집: 중복 제거 및 오타 수정)
                                        |
                             (git add & git commit)
                                        |
[Remote Repository (Gitea)] <--- (git push origin master) --- [정상 병합 완료]

```

---

### 3. 해결 과정 (Action)

#### 3-1. 저장소 접속 및 병합 충돌 유도

Storage 서버에 접속한 후, 원격 저장소에 먼저 반영된 변경 사항을 로컬로 가져와 충돌 상황을 활성화합니다.

```bash
# Storage 서버로 max 계정을 사용하여 접속
ssh max@ststor01

# 작업 디렉터리로 이동
cd /home/max/story-blog/

# 원격 저장소의 최신 이력을 로컬로 병합 (이 과정에서 Merge conflict 발생)
git pull origin master

```

#### 3-2. 충돌 원인 분석 및 수동 편집 (vi 에디터)

충돌이 발생한 파일을 열어 Git이 삽입한 충돌 마커를 제거하고 명세서의 요구사항에 맞게 데이터를 정제합니다.

```bash
# 편집기를 통해 충돌이 발생한 파일 열기
vi story-index.txt

# [수정 의도 및 톺아보기]
# 1. <<<<<<< HEAD, =======, >>>>>>> 등의 Git 충돌 마커 라인 삭제
# 2. 동일한 제목이 2줄씩 중복 기재된 부분을 삭제하여, 총 4개의 스토리 제목이 1줄씩만 존재하도록 정렬
# 3. 'The Lion and the Mooose' 라인을 찾아 'The Lion and the Mouse'로 오타 수정 후 저장(:wq)

```

#### 3-3. 병합 커밋 생성 및 원격 저장소 최종 푸시

파일의 정합성이 확보된 것을 확인한 후, 스테이징을 거쳐 병합 커밋을 확정하고 원격 저장소로 밀어 넣습니다.

```bash
# 충돌 해결 및 정제가 완료된 파일을 스테이징 영역에 추가
git add story-index.txt

# 병합을 확정 짓는 커밋 생성
git commit -m "commit"

# 로컬의 정상 병합 내역을 원격 저장소(origin)의 master 브랜치로 최종 푸시
git push origin master

```

---

### 4. 핵심 개념 정리

- **Merge Conflict (병합 충돌)**: 두 개 이상의 브랜치에서 동일한 파일의 같은 라인을 동시에 수정하여 병합을 시도했을 때, Git 시스템이 어느 코드를 우선해야 할지 판단하지 못하고 개발자에게 수동 선택을 위임하는 상태입니다.
  > 💡 두 명의 건축가가 동일한 설계도의 같은 방 구조를 각자 다르게 고친 뒤 제출했을 때, 현장 소장이 어느 도면을 최종 반영할지, 혹은 두 의견을 어떻게 합칠지 직접 빨간펜을 들고 결정하여 승인해야 하는 상황과 같습니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **실무 관점의 의의**: 다수의 엔지니어가 협업하는 Git 환경에서 파일 충돌은 필연적으로 발생합니다. 단순히 당황하여 작업 디렉터리를 지우거나 `git push --force`로 타인의 코드를 덮어씌우는 대신, 충돌 마커의 구조를 분석하고 로컬/원격 코드를 융합하여 안전하게 정제(Refining)하는 과정은 소스코드 유실 방지와 서비스 무결성 유지에 필수적인 인프라 엔지니어링 역량입니다.
