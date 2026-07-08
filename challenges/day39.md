## 📅 Day 39: Create a Docker Image From Container

### 1. Task

- **요구사항**: Nautilus 개발 팀이 테스트 용도로 사용하며 내부 설정을 변경해 둔 컨테이너가 있습니다. 해당 컨테이너가 삭제되어 변경 내역이 휘발되는 것을 방지하기 위해, 현재 상태 그대로 새로운 Docker 이미지로 떠서 백업해 달라는 요청이 인입되었습니다.

- **목표**:
  1. Application Server 1(`stapp01`, 계정: `tony`)에 접속합니다.
  2. 현재 실행 중인 `ubuntu_latest` 컨테이너의 상태를 점검합니다.
  3. 해당 컨테이너의 현재 스냅샷을 찍어 `ecommerce:devops`라는 이름의 새로운 커스텀 이미지로 저장(Commit)하고 검증합니다.

---

### 2. Workflow

```text
[App Server 1 (stapp01)]

   (Running Container)                          (Local Image Registry)
      ubuntu_latest                                ecommerce:devops
 [테스트 파일 및 설정 변경]                         [백업 이미지]
           │                                             ▲
           └────────── (docker commit) ──────────────────┘

```

---

### 3. 해결 과정 (Action)

#### 3-1. 대상 애플리케이션 서버 접속

인프라 명세서에 정의된 자격 증명을 사용하여 타겟 서버로 진입합니다.

```bash
# Application Server 1로 SSH 접속 (계정: tony)
ssh tony@stapp01
# 비밀번호 입력: Ir0nM@n

```

#### 3-2. 백업 대상 컨테이너 식별

이미지를 구워내기 전, 개발자가 작업 중인 대상 컨테이너가 서버에서 정상적으로 구동 중인지 상태를 점검합니다.

```bash
# 실행 중인 컨테이너 목록 중 ubuntu_latest 필터링 확인
sudo docker ps | grep ubuntu_latest

```

#### 3-3. 컨테이너를 새로운 이미지로 커밋(Commit)

현재 컨테이너의 레이어(변경된 파일 시스템)를 캡처하여 새로운 이미지로 영구 저장합니다.

```bash
# ubuntu_latest 컨테이너를 ecommerce:devops 이미지로 구워내기
sudo docker commit ubuntu_latest ecommerce:devops

```

_(명령어 실행 후 `sha256:c60bd90...` 형태의 해시값이 반환되면 성공적으로 캡처된 것입니다.)_

#### 3-4. 생성된 이미지 검증

로컬 저장소에 요구사항에 맞는 레포지토리와 태그명으로 이미지가 잘 적재되었는지 확인합니다.

```bash
# 생성된 ecommerce 이미지 확인
sudo docker images | grep ecommerce

```

---

### 4. 핵심 개념 정리

- **`docker commit`**: 현재 실행 중이거나 중지된 컨테이너의 변경된 최상단 레이어(Read-Write Layer)를 묶어서, 마치 사진을 찍듯 새로운 이미지(Read-Only Layer)로 저장하는 명령어입니다.
  > 💡 비유하자면, RPG 게임을 하다가 주인공의 레벨업과 아이템 파밍이 끝난 현재 상태를 그대로 '세이브(Save) 파일'로 만들어두는 것과 같습니다. 언제든 이 세이브 파일을 불러오면(Run) 똑같은 장비와 레벨로 게임을 시작할 수 있습니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **컨테이너의 '휘발성' 공포에서 벗어나기 (Pain Point)**: 도커 컨테이너는 기본적으로 무상태(Stateless)를 지향합니다. 즉, 개발자가 컨테이너 내부에 접속해서 이것저것 패키지를 깔고 설정 파일을 고쳐놔도, 누군가 무심코 `docker rm`으로 컨테이너를 지우거나 재생성해 버리면 그동안의 고생이 한순간에 날아갑니다. 이런 끔찍한 데이터 유실 사태를 막고, 개발자가 세팅해 둔 '임시 작업 환경'을 가장 빠르고 직관적으로 백업할 수 있는 응급 구조대가 바로 `docker commit`임을 실감했습니다.

- **회고(Retrospective)**: 실무의 정석(Best Practice) 관점에서 보자면, 인프라의 모든 변경 사항은 `Dockerfile`에 코드로 기록하여(IaC) 이미지를 빌드하는 것이 맞습니다. `docker commit`으로 만든 이미지는 내부에 정확히 어떤 명령어가 실행되어 패키지가 깔렸는지 추적할 수 없는 '블랙박스'가 되기 때문입니다. 하지만 이번 미션처럼 당장 코드로 정의하기 애매한 긴급 테스트 환경을 보존해야 할 때, 이 명령어는 가뭄의 단비와 같습니다. 앞으로는 상황의 급박함과 유지보수성을 저울질하여, **'빠른 응급처치(Commit)'와 '영구적인 표준화(Dockerfile)'를 유연하게 선택**하는 아키텍트의 시각을 가져야겠다고 다짐했습니다.
