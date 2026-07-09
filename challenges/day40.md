## 📅 Day 40: Docker EXEC Operations

### 1. Task

- **요구사항**: 휴가를 떠난 DevOps 팀원의 업무 공백을 메우기 위해, App Server 2에서 구동 중인 `kkloud` 컨테이너 내부의 서비스 구성을 긴급히 마무리해야 합니다.

- **목표**:
  1. App Server 2(`stapp02`, 계정: `steve`)에 접속하여 실행 중인 `kkloud` 컨테이너 내부로 진입합니다.
  2. 컨테이너 내부의 `apt` 패키지 매니저를 사용하여 `apache2` 웹 서버를 설치합니다.
  3. Apache가 기본 80 포트 대신 **5004 포트**에서 모든 인터페이스의 요청을 수신(Listen)하도록 설정합니다.
  4. 컨테이너 내부에서 Apache 서비스를 구동하고, 작업 완료 후에도 컨테이너가 종료되지 않도록 유지합니다.

---

### 2. Workflow

```text
[App Server 2 (stapp02)]
          │
  (docker exec -it bash)
          │
          ▼
[Container: kkloud]
 1. apt update & install apache2
 2. sed -i 's/80/5004/g' /etc/apache2/ports.conf
 3. sed -i 's/80/5004/g' /etc/apache2/sites-enabled/000-default.conf
 4. service apache2 start

```

---

### 3. 해결 과정 (Action)

#### 3-1. 대상 서버 접속 및 컨테이너 내부 진입

작업 대상인 App Server 2로 원격 접속한 뒤, 실행 중인 컨테이너에 대화형 셸(bash)을 열어 진입합니다.

```bash
# App Server 2로 SSH 접속 (계정: steve)
ssh steve@stapp02
# 비밀번호 입력: Am3ric@

# kkloud 컨테이너 내부에 root 권한으로 bash 셸 진입
sudo docker exec -it kkloud bash

```

#### 3-2. 웹 서버 패키지 설치

컨테이너 내부 운영체제의 패키지 목록을 최신화하고 요구된 웹 서버 패키지를 설치합니다.

```bash
# [컨테이너 내부] 패키지 인덱스 업데이트
apt update -y

# [컨테이너 내부] apache2 설치
apt install -y apache2

```

#### 3-3. Apache 수신 포트(Listen Port) 변경

기본 HTTP 포트(80)를 요구사항(5004)에 맞게 변경합니다. 수신 대기 포트 설정 파일과 기본 가상 호스트(Virtual Host) 설정 파일을 모두 수정해야 정상적으로 라우팅됩니다.

```bash
# [컨테이너 내부] ports.conf 파일의 80 포트를 5004로 일괄 치환
sed -i 's/80/5004/g' /etc/apache2/ports.conf

# [컨테이너 내부] 가상 호스트 설정 파일의 80 포트를 5004로 일괄 치환
sed -i 's/80/5004/g' /etc/apache2/sites-enabled/000-default.conf

```

#### 3-4. 서비스 가동 및 상태 검증

설정을 적용하기 위해 Apache 데몬을 시작하고, 프로세스가 정상적으로 백그라운드에 안착했는지 확인합니다.

```bash
# [컨테이너 내부] Apache 서비스 시작
service apache2 start

# [컨테이너 내부] 서비스가 정상 구동(running) 중인지 상태 점검
service apache2 status

# 작업 완료 후 컨테이너 셸 종료 (컨테이너는 백그라운드에서 계속 실행됨)
exit

```

---

### 4. 핵심 개념 정리

- **`docker exec -it [컨테이너명] bash`**: 실행 중인 컨테이너에 새로운 프로세스(이 경우 bash 셸)를 실행하여 터미널 제어권을 가져오는 명령어입니다. `-i`(Interactive)와 `-t`(TTY) 옵션을 결합하여 마치 일반 리눅스 서버에 SSH로 접속한 것과 같은 환경을 제공합니다.
- **Apache Listen Port와 Virtual Host**: `ports.conf` 파일의 `Listen` 지시어는 웹 서버가 네트워크 인터페이스에서 어떤 포트를 열고 기다릴지 결정합니다. 그리고 `sites-enabled` 하위의 가상 호스트 설정은 "해당 포트로 들어온 요청을 어느 디렉터리의 웹 페이지로 연결할지"를 결정하므로, 포트 변경 시 두 곳의 아라비아 숫자를 모두 맞춰주어야 합니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **'동료의 똥(?) 치우기'에서 배우는 핫픽스(Hotfix)의 기술**: 휴가 간 동료가 마무리하지 못한 컨테이너 내부 작업을 넘겨받는 상황은 실무에서 꽤나 빈번하게 일어납니다. 원칙적으로라면 도커 이미지를 새로 빌드(Dockerfile 수정)하거나 외부 볼륨을 마운트해서 설정 파일을 덮어씌우는 것이 맞습니다. 하지만 당장 오늘 안에 급하게 테스트 환경을 띄워야 하는 상황에서는 `docker exec`로 침투하여 리눅스 원격 서버 다루듯 셸 스크립트(`sed`)로 설정을 때려 맞추는 유연함도 필요합니다. 정석과 응급처치 사이에서 줄타기하는 요령을 익힐 수 있었습니다.

- **포트 변경 시 흔히 겪는 삽질 포인트 방지**: 주니어 시절 아파치 포트를 바꿀 때 `ports.conf`만 수정했다가, 데몬은 떴는데 웹 브라우저 접속은 안 되는 황당한 에러(가상 호스트 불일치)를 겪곤 합니다. 오늘 실습에서 `sites-enabled/000-default.conf`까지 세트로 수정하는 프로세스를 명확히 하면서, "포트 번호를 바꿀 때는 문지기(Listen)와 안내원(Virtual Host)의 지시서를 동시에 업데이트해야 한다"는 웹 서버 구성의 불문율을 손끝에 새길 수 있었습니다.
