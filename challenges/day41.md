## 📅 Day 41: Write a Docker File

### 1. Task

- **요구사항**: Nautilus 애플리케이션 개발 팀에서 신규 프로젝트를 위한 커스텀 Docker 이미지 제작을 요청했습니다. 수동으로 패키지를 설치하는 대신, 향후 CI/CD 파이프라인에서 자동 빌드가 가능하도록 인프라 명세가 담긴 `Dockerfile`을 작성해야 합니다.

- **목표**:
  1. App Server 1(`stapp01`, 계정: `tony`)에 접속합니다.
  2. 서버 내부에 지정된 작업 경로(`/opt/docker/`)를 생성합니다.
  3. `ubuntu:24.04` 베이스 이미지 위에 `apache2` 패키지를 설치하고, 웹 서버가 **5001 포트**에서 동작하도록 지시하는 `Dockerfile`을 올바르게 작성합니다.

---

### 2. Workflow

```text
[App Server 1 (stapp01)]
          │
  (mkdir /opt/docker/)
          │
          ▼
    [vi Dockerfile]
 1. FROM ubuntu:24.04
 2. ENV DEBIAN_FRONTEND=noninteractive
 3. RUN apt-get update && install apache2
 4. RUN sed (80 -> 5001 port change)
 5. CMD ["apache2ctl", "-D", "FOREGROUND"]

```

---

### 3. 해결 과정 (Action)

#### 3-1. 대상 서버 접속 및 작업 디렉터리 준비

인프라 명세서에 정의된 자격 증명으로 서버에 진입한 뒤, 빌드 컨텍스트(Build Context)로 사용할 독립된 디렉터리를 생성합니다.

```bash
# App Server 1로 SSH 접속 (계정: tony)
ssh tony@stapp01
# 비밀번호 입력: Ir0nM@n

# Dockerfile을 위치시킬 작업 디렉터리 생성 (권한 필요)
sudo mkdir -p /opt/docker/

```

#### 3-2. Dockerfile 명세서 작성

`vi` 편집기를 사용하여 요구사항을 코드로 번역합니다.

```bash
# Dockerfile 생성 및 편집 모드 진입
sudo vi /opt/docker/Dockerfile

```

**[Dockerfile 전체 코드]**

```dockerfile
# 1. 베이스 이미지 지정
FROM ubuntu:24.04

# 2. Ubuntu 패키지 설치 시 대화형 프롬프트(Timezone 등)에 의한 빌드 멈춤 방지
ENV DEBIAN_FRONTEND=noninteractive

# 3. 패키지 인덱스 갱신 및 apache2 웹 서버 설치
RUN apt-get update && apt-get install -y apache2

# 4. 아파치 기본 포트(80)를 요구사항(5001)으로 일괄 변경
RUN sed -i 's/80/5001/g' /etc/apache2/ports.conf
RUN sed -i 's/80/5001/g' /etc/apache2/sites-available/000-default.conf

# 5. 컨테이너 실행 시 아파치 데몬을 포그라운드로 유지하여 컨테이너 종료 방지
CMD ["apache2ctl", "-D", "FOREGROUND"]

```

_(작성 완료 후 `ESC` ➔ `:wq` ➔ `Enter` 로 저장 후 종료)_

#### 3-3. 파일 무결성 검증

오타나 누락된 라인이 없는지 최종적으로 확인합니다.

```bash
# 작성된 파일 내용 검증
cat /opt/docker/Dockerfile

```

---

### 4. 핵심 개념 정리

- **`Dockerfile` (Infrastructure as Code)**: 서버에 접속해서 수동으로 치던 명령어들을 텍스트 파일 하나로 정의하여, 언제 어디서든 동일한 환경을 찍어낼 수 있게 만드는 도커의 핵심 레시피입니다.
- **`ENV DEBIAN_FRONTEND=noninteractive`**: 우분투(Debian 계열) 패키지 중 일부는 설치 과정에서 사용자에게 국가나 타임존을 묻는 팝업을 띄웁니다. 도커 빌드 과정은 무인(Unattended) 상태로 진행되므로, 이 환경 변수 없이 빌드를 돌리면 프롬프트 입력을 무한정 기다리며 빌드가 멈춰버리는(Hang) 현상이 발생합니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **CI/CD 파이프라인 타임아웃의 주범 잡기 (Pain Point)**: 실무에서 CI/CD 파이프라인을 구축해 놓고 퇴근했는데, 다음날 아침에 빌드 서버가 타임아웃으로 뻗어있는 끔찍한 경험을 흔히 겪습니다. 십중팔구는 누군가 `Dockerfile`에 `DEBIAN_FRONTEND=noninteractive`나 `apt install -y`의 `-y`(자동 Yes) 옵션을 빼먹어서, 보이지도 않는 터미널 프롬프트가 사용자 입력을 밤새 기다리고 있었던 탓입니다. 이번 실습을 통해 "자동화된 빌드 환경에서는 인간의 개입을 원천 차단하는 방어적 코딩이 필수적"임을 뼈저리게 상기하게 되었습니다.

- **'블랙박스'에서 '투명한 유리상자'로 (Retrospective)**: 바로 며칠 전 미션(Day 39)에서 `docker commit`으로 실행 중인 컨테이너를 구워내는 임시방편을 배웠습니다. 하지만 오늘 `Dockerfile`을 직접 작성해 보니, 컨테이너 인프라 관점에서는 결국 코드로 형상 관리(Git)가 가능한 `Dockerfile`이 정답임을 확신하게 되었습니다. 누가 언제 어떤 포트를 바꿨고(Commit History), 어떤 베이스 이미지를 썼는지 투명하게 추적할 수 있는 인프라. 이것이 바로 주니어 티를 벗고 프로페셔널한 DevOps 엔지니어로 넘어가는 가장 중요한 마인드셋의 전환점이 아닐까 싶습니다.
