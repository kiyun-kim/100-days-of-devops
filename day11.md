## 📅 Day 11: [Java Web] Tomcat 서버 설치 및 WAR 웹 애플리케이션 배포

### 1. Task

- **요구사항**: Nautilus 앱 개발 팀이 자바(Java) 기반으로 개발한 신규 베타 버전 웹 애플리케이션을 구동하기 위해 WAS(Web Application Server) 환경을 구축하고 웹 패키지를 배포
- **대상**: `Jump Host` 및 Stratos DC 내 App Server 1 (`stapp01`)
- **목표**: App Server 1에 Apache Tomcat을 설치하고 서비스 포트를 `8084`로 변경한 뒤, Jump Host의 `/tmp/ROOT.war` 파일을 원격 배포 경로로 이관 및 기동하여 최종 베이스 URL 접속 상태를 검증

### 2. Workflow

```text
[Jump Host (Control Node)]                  [App Server 1 (stapp01)]
   /tmp/ROOT.war (웹 패키지)                  (1) 톰캣 패키지 빌드 (yum install tomcat)
        │                                     (2) 리스너 포트 수정 (8080 -> 8084)
        ▼ (3) 원격 파일 전송 (scp)                  │
   stapp01 서버의 /tmp/ 경로 ───────────────────────┘
        │
        ▼ (4) 배포 디렉토리 이관 (/var/lib/tomcat/webapps/)
   [Tomcat 서비스 엔진 가동] ── (5) 로컬 런타임 통신 검증 (curl http://localhost:8084)
```

### 3. 해결 과정 (Troubleshooting & Action)

#### 3-1. App Server 1(`stapp01`)에 접속하여 자바 웹 애플리케이션 구동을 위한 표준 `tomcat` 패키지 설치하기

```bash
# App Server 1 접속 후 톰캣 빌드 (의존성 자바 패키지 자동 포함)
sudo yum install -y tomcat
```

#### 3-2. 톰캣 메인 환경 설정 파일(`server.xml`)을 수정하여 기본 서비스 포트를 8080에서 8084로 변경하기

```bash
# 설정 파일 편집기 진입
sudo vi /etc/tomcat/server.xml

# [포트 수정 내역]
# 기존:
<Connector port="8080" protocol="HTTP/1.1" ... />
# 변경:
<Connector port="8084" protocol="HTTP/1.1" ... />
```

#### 3-3. Jump Host에 보관 중인 애플리케이션 압축 팩(`ROOT.war`)을 App Server 1의 임시 경로로 원격 전송하기

```bash
scp /tmp/ROOT.war natasha@stapp01:/tmp/
```

#### 3-4. 전송된 `ROOT.war` 파일을 톰캣의 공식 웹 애플리케이션 자동 전개(Deployment) 디렉토리로 이관하기

```bash
# stapp01 서버에서 실행
sudo mv /tmp/ROOT.war /var/lib/tomcat/webapps/
```

#### 3-5. 톰캣 서비스 데몬을 기동하고, 시스템 리부팅 시에도 웹 서버가 자동 활성화되도록 스케줄러 등록하기

```bash
sudo systemctl start tomcat
sudo systemctl enable tomcat
```

#### 3-6. 변경된 포트(8084) 및 베이스 URL을 통해 자바 웹앱이 에러 없이 정상 렌더링되는지 터미널에서 교차 검증하기

```bash
curl http://localhost:8084

# Output:
# <!DOCTYPE html><html><head><title>SampleWebApp</title></head>
# <body><h2>Welcome to xFusionCorp Industries!</h2></body></html>
```

### 4. 무엇을 배웠는가 (Takeaway)

- **WAR(Web Application Archive) 패키징과 자동 배포 원리**: 자바 웹 애플리케이션을 구성하는 수많은 소스 코드, 클래스 파일, 이미지 자원들을 단 하나의 압축 팩(`.war`)으로 묶어 배포하는 표준 아키텍처를 이해했습니다. 톰캣 엔진은 구동 시 `webapps` 폴더에 위치한 `.war` 파일을 탐지하여 스스로 압축을 해제하고 주소창을 통해 접근 가능한 라이브 웹사이트로 전개하는 자동화 메커니즘을 가졌음을 배웠습니다.
- **네트워크 소켓 포트(Port)의 커스텀 매핑**: 모든 네트워크 기반 프로그램은 고유한 방 번호(Port)를 할당받아 통신합니다. 톰캣의 기본값인 `8080` 포트를 인프라 설계 지침에 맞춰 `8284` 또는 `8084` 등으로 유연하게 변경하려면 `server.xml` 파싱 능력이 필수적이며, 이를 통해 하나의 IP 서버 안에서도 여러 개의 독립된 웹 서비스를 격리하여 안전하게 구동할 수 있다는 점을 체득했습니다.
- **Headless 환경에서의 웹 서비스 검증(`curl`)**: 모니터나 크롬 브라우저 같은 그래픽 UI(GUI)가 존재하지 않는 검은색 텍스트 터미널 환경에서도, `curl` 명령어를 활용하면 웹 서버가 뿜어내는 HTML 소스코드를 직접 긁어와 런타임 크래시 여부를 즉시 판단할 수 있습니다.
