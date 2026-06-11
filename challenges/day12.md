## 📅 Day 12: Apache 웹 서버 포트 차단 및 불법 점거 프로세스 해결

### 1. Task

- **요구사항**: xFusionCorp Industries 내 특정 앱 서버의 Apache 웹 서비스(지정된 커스텀 포트)가 외부와 통신이 되지 않는 장애를 발견하여 시스템 정상화 조치 진행
- **대상**: Stratos DC 내 지정된 애플리케이션 서버 노드 (`stapp01~03` 중 시스템이 무작위 지정한 Target 노드)
- **목표**: 서비스 가동을 막고 있는 '포트 중복 사용(Address already in use)' 프로세스를 식별하여 강제 종료하고 웹 서비스 무결성을 완수

### 2. Workflow

```text
[외부 사용자 / Jump Host]
         │
         ▼  (X) 네트워크 요청 차단 (포트 락 또는 방화벽 장벽)
 [Target App Server] ── [진짜 주인: Apache (httpd)] -> 포트 점거 불청객 때문에 구동 실패
```

### 3. 해결 과정 (Troubleshooting & Action)

#### 3-1. 문제가 발생한 App 서버에 접속한 뒤 환경 변수 꼬임 방지를 위해 root 권한 획득하기

```bash
# 예시: stapp01 서버 접속 후 root 쉘 전환
ssh tony@stapp01
sudo su -
```

#### 3-2. 지정된 커스텀 포트(예: 6200 또는 6400)를 불법 점거하여 Apache 기동을 막고 있는 PID(유령 프로그램의 주민번호) 찾아내기

```bash
# netstat가 없는 미니멀 환경이므로 표준 내장 명령어 ss 활용
ss -lntp | grep [지정 포트]

# 예시 출력 결과: users:(("dummy_proc",pid=1452,fd=3)) -> 범인 PID는 1452!
```

#### 3-3. 6200/6400번에 먼저 있던 불청객 프로세스 강제 종료하기

```bash
# kill -9 [확인한 범인 PID]
kill -9 1452
```

#### 3-4. 소켓 방이 완전히 비워진 것을 확인한 후, 진짜 주인인 Apache(httpd) 엔진 가동하기

```bash
systemctl start httpd
systemctl status httpd
# (Active: active (running) 초록색 불 확인)
```

#### 3-5. `Jump Host`로 복귀하여 최종적으로 포트가 완전히 뚫렸는지 웹 소스 응답 교차 검증하기

```bash
# stapp 세션 탈출 후 Jump Host에서 실행
exit
exit

curl http://[대상 서버]:[지정 포트]
# (페이지 HTML 코드가 정상 출력되면 복구 성공)
```

### 4. 무엇을 배웠는가 (Takeaway)

- **소켓 주소 중복(Address already in use)의 실무적 대응**: 리눅스 시스템에서 하나의 포트(Port) 채널은 단 하나의 프로세스에만 독점 점유권이 주어집니다. 웹 서버가 가동 도중 포트 바인딩 에러를 내며 튕긴다면, `ss -lntp` 명령어로 소켓을 꽉 쥐고 놓지 않는 유령 프로세스(PID)를 정확히 저격하여 `kill -9`로 메모리 상에서 증발시켜야 엔진이 정상 입주할 수 있음을 배웠습니다.
