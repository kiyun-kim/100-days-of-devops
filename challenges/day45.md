## 📅 Day 45: Dockerfile 빌드 에러 트러블슈팅 및 Apache SSL 웹 서버 이미지 최적화

### 1. Task

- **요구사항**: Nautilus DevOps 팀원이 작성 중 빌드가 실패하던 기존 Dockerfile의 구조적 결함(상대 경로 누락 및 빌드 콘텍스트 파일 복사 가이드 위반)을 정밀 진단(RCA)하고, 베이스 이미지를 유지한 상태에서 무결하게 빌드가 완료되도록 수정합니다.

- **목표**: Stratos DC 내부의 Application Server 3 (`stapp03`) 환경에 접근하여 에러를 유발하는 Dockerfile을 교정하고 최종 빌드 테스트를 통과시킵니다.
  1. 대상 서버인 App Server 3 (`stapp03`)에 SSH 원격 접속을 수행합니다.
  2. `/opt/docker/Dockerfile` 내부의 유효한 설정(Apache 베이스 이미지, 포트 8080 변경 등)을 보존하면서 문법 오류를 수정합니다.
  3. 호스트 시스템의 인증서(`certs/`) 및 정적 파일(`html/`)이 이미지 레이어 내부로 정확히 이식되도록 파일 복사 메커니즘을 전면 수정합니다.
  4. 수정 완료 후 `docker build`를 실행하여 14개의 빌드 단계(`14/14 FINISHED`)가 에러 없이 완결되는지 검증합니다.

---

### 2. Workflow

```text
  [ Host: App Server 3 (/opt/docker) ]
        ├── certs/ (server.crt, server.key)
        ├── html/ (index.html)
        └── Dockerfile (Broken -> Fixed)
                 │
                 │ (sudo docker build)
                 ▼
  [ Docker Build Engine (Context) ]
        ├── Step 1/9 : FROM httpd:2.4.43
        ├── Step 2/9 : WORKDIR /usr/local/apache2 (경로 고정)
        ├── Step 3-6/9 : RUN sed (포트 및 SSL 모듈 활성화)
        └── Step 7-9/9 : COPY (호스트 파일 안전 이식)
                 │
                 ▼
  [ Successful Image: test-httpd:1.0 ]

```

---

### 3. 해결 과정 (Action)

#### 3-1. App Server 3 원격 접속 및 원본 Dockerfile 진단

작업 대상 서버에 원격 접속한 후, 기존에 실패하던 Dockerfile의 구문을 cat 명령어로 열람하여 문제 원인을 파악했습니다.

```bash
# Jump Host에서 App Server 3 서버로 SSH 원격 접속 실행
ssh banner@stapp03

# 작업 대상 디렉터리로 이동 및 파일 구조 확인
cd /opt/docker
ls -l

# 에러를 유발하는 기존 Dockerfile 내용 확인
cat Dockerfile

```

> **[기존 문제 코드 및 한계점]**
>
> - `RUN sed -i ... conf/httpd.conf`와 같이 명확한 작업 디렉터리(`WORKDIR`) 지정 없이 상대 경로를 호출하여 파일 탐색 실패 유발.
> - 호스트 내부 빌드 컨텍스트에 위치한 외부 파일(`certs`, `html`)을 가져오기 위해 도커 지시어인 `COPY`를 쓰지 않고 컨테이너 내부 명령인 `RUN cp`를 오용하여 파일 부재 에러 유발.

#### 3-2. Dockerfile 아키텍처 재설계 및 교정

베이스 이미지 사양을 완벽히 유지하면서, 경로 탐색 안정성과 도커 빌드 표준 규격을 준수한 형태로 Dockerfile을 전면 재작성했습니다.

```bash
# Dockerfile 편집기 오픈 후 올바른 템플릿으로 전면 수정
sudo vi Dockerfile

```

```dockerfile
FROM httpd:2.4.43

# 1. 작업 디렉터리를 Apache 홈 경로로 고정하여 상대 경로 에러 원천 차단
WORKDIR /usr/local/apache2

# 2. Apache 수신 포트를 기본 80에서 8080으로 치환
RUN sed -i "s/Listen 80/Listen 8080/g" conf/httpd.conf

# 3. SSL 통신에 필요한 핵심 모듈(mod_ssl, mod_socache_shmcb) 및 설정 파일 주석 해제
RUN sed -i '/LoadModule\ ssl_module modules\/mod_ssl.so/s/^#//g' conf/httpd.conf
RUN sed -i '/LoadModule\ socache_shmcb_module modules\/mod_socache_shmcb.so/s/^#//g' conf/httpd.conf
RUN sed -i '/Include\ conf\/extra\/httpd-ssl.conf/s/^#//g' conf/httpd.conf

# 4. RUN cp 방식의 치명적 오류를 극복하고, 호스트 자산을 안전하게 레이어로 삽입하는 COPY 지시어 적용
COPY certs/server.crt conf/server.crt
COPY certs/server.key conf/server.key
COPY html/index.html htdocs/index.html

```

#### 3-3. 이미지 빌드 실행 및 파이프라인 무결성 검증

재작성된 Dockerfile을 빌드 컨텍스트에 로드하여 최종 이미지가 정상적으로 패키징되는지 테스트를 수행했습니다.

```bash
# 수정된 명세서를 기반으로 로컬 태그 버전을 지정하여 빌드 프로세스 가동
sudo docker build -t test-httpd:1.0 .

```

---

### 4. 핵심 개념 정리

- **Build Context (빌드 컨텍스트)**: `docker build` 명령을 실행하는 시점에 도커 클라이언트가 도커 데몬(Engine) 측으로 전송하는 지정 경로 내의 파일 및 디렉터리 자산의 집합체입니다. Dockerfile 내부의 `COPY`나 `ADD` 지시어는 오직 이 빌드 컨텍스트 내부에 들어온 파일들만 참조할 수 있습니다.

  > 💡 비유하자면, 밀폐된 클린룸(도커 엔진) 안에서 조립 로봇이 반도체를 조립할 때, 클린룸 외부 보관소(호스트 전체 파일 시스템)에 있는 부품을 직접 꺼내올 수 없으므로, 조립 시작 전에 미리 작업용 쟁반(빌드 컨텍스트) 위에 필요한 부품들(`certs`, `html`)을 담아서 방 안으로 넣어주어야만 로봇이 집어 들 수 있는 것과 같습니다.

- **WORKDIR 지시어**: Dockerfile 내부에서 이후에 선언되는 `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, `ADD` 등의 지시어들이 실행될 컨테이너 내부 파일 시스템의 절대 경로 기준점을 고정하는 명령입니다. 지정한 디렉터리가 존재하지 않을 경우 자동으로 생성합니다.

  > 💡 비유하자면, 낯선 대도시(컨테이너 내부)의 분산된 사무실들을 찾아다니며 서류를 수정할 때, 매번 "지하 2층 탕비실 옆 문서고로 가라"고 모호하게 지시하는 대신, 아예 본사 7층 기획실(`WORKDIR /usr/local/apache2`)에 베이스캠프를 차려놓고 "거기 책상 위에 있는 서류 파일(`conf/httpd.conf`) 고쳐라"고 명확한 가이드를 주는 기준점과 같습니다.

- **COPY vs RUN cp**: `COPY`는 호스트 운영체제의 파일(빌드 컨텍스트 자산)을 빌드 시점에 이미지 레이어 내부로 주입하는 도커 고유 명령어인 반면, `RUN cp`는 이미 이미지 내부에 존재하는 파일 시스템의 자원을 컨테이너 내의 다른 경로로 복사할 때 사용하는 리눅스 셸 커맨드입니다.

  > 💡 비유하자면, `COPY`는 외부 대형 마트(호스트)에서 신선한 식재료를 구매해 식당 주방(이미지) 안으로 배달해 놓는 과정이고, `RUN cp`는 주방 냉장고에 이미 들어있는 양파를 꺼내 조리대 앞으로 옮기는 행위입니다. 마트에서 재료를 안 사 왔는데(`COPY` 누락) 주방 안에서 꺼내 쓰려고만 하면(`RUN cp`) 에러가 나는 이치입니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **회고(Retrospective)**: 타 엔지니어가 작성하다 실패한 트러블슈팅 세션에 투입되어 Docker의 빌드 메커니즘과 지시어 최적화의 중요성을 몸소 다져보는 좋은 계기가 되었습니다. 특히 컨테이너 내부 셸 환경의 컨텍스트와 호스트 운영체제의 파일 시스템 간 경계면을 명확히 이해하지 못하면, 아무리 훌륭한 애플리케이션 코드를 짜더라도 컨테이너라이징 단계에서 인프라 전체가 마비될 수 있다는 교훈을 얻었습니다.

- **Pain Point**: 실무 배포 파이프라인(CI/CD)에서 `RUN cp`와 같은 잘못된 명령어 매핑이나 잘못된 상대 경로 지정이 포함된 코드가 원격 레포지토리에 푸시되면, 빌드 에러로 인해 전체 파이프라인이 멈추고 릴리즈가 지연되는 병목 현상이 발생합니다. 특히 에러 메시지가 모호하게 출력될 경우, 원인을 찾기 위해 도커 레이어를 역추적해야 하는 심각한 리소스 낭비가 초래되므로, 선언적 가이드북인 Dockerfile을 작성할 때는 항상 `WORKDIR`을 통한 경로 절대화와 적격한 지시어 채택이 필수적입니다.

- **성장 포인트**: 에러 로그의 표면적인 메시지만 보고 임기응변식 패치를 적용하는 수준을 넘어, 도커 엔진이 레이어를 적층하는 논리적 메커니즘을 추론하여 아키텍처의 근본적인 취약점(RCA)을 즉각 교정해 내는 한층 더 숙련된 트러블슈팅 감각을 보유하게 되었습니다.
