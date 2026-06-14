## 📅 Day 16: Nginx 로드밸런서(LBR) 구축 및 트래픽 부하분산

### 1. Task

- **요구사항**: xFusionCorp Industries의 웹 트래픽 증가로 인한 성능 저하를 해결하기 위해 고가용성 스택(High Availability Stack)의 핵심인 로드밸런서(LBR) 서버를 구성한다.

- **목표**:

1. `stlb01` 서버에 Nginx 설치 및 활성화
2. 메인 설정 파일 `/etc/nginx/nginx.conf`의 `http` 컨텍스트 내부에 `upstream` 풀을 정의하여 3대의 앱 서버(`포트 8087`) 연동
3. 모든 앱 서버의 Apache 가동 상태를 유지한 채 LBR 프록시 패스(Proxy Pass) 연결 및 최종 라운드로빈 통신 무결성 검증

---

### 2. Workflow

```text
[ 유저 트래픽 요청 (Port 80) ] ---> [ stlb01 (Nginx Load Balancer) ]
                                                |
                      +-------------------------+-------------------------+
                      |                         |                         |
                      v                         v                         v
           [ stapp01 (Port 8087) ]   [ stapp02 (Port 8087) ]   [ stapp03 (Port 8087) ]

```

---

### 3. 해결 과정 (Action)

#### 3-1. 로드밸런서 패키지 설치 및 환경 설정

```bash
# 1. LBR 서버 접속 및 Nginx 가동
ssh loki@stlb01
sudo yum install -y nginx
sudo systemctl start nginx && sudo systemctl enable nginx

# 2. 부하분산 구조 수동 편집
sudo vi /etc/nginx/nginx.conf

```

**[nginx.conf 수정 내역]**

```nginx
http {
    # 3대의 백엔드 앱 서버 그룹 정의
    upstream backend_servers {
        server stapp01:8087;
        server stapp02:8087;
        server stapp03:8087;
    }

    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;

        # 들어오는 모든 라우팅 설정을 백엔드 업스트림 풀로 전달
        location / {
            proxy_pass http://backend_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}

```

#### 3-2. 문법 예외 처리 및 최종 서비스 반영

```bash
# 1. 중괄호 매칭 오류 및 구문 오류 수정 후 문법 무결성 검사
sudo nginx -t
# 출력: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok

# 2. 엔진 재시작 및 세션 탈출
sudo systemctl restart nginx
exit

# 3. [최종 검증] 점프 호스트에서 LBR 대표 IP로 curl 요청 연속 송신
curl http://stlb01:80
# 출력: Welcome to xFusionCorp Industries! (정상 프록시 통신 완료)

```

---

### 4. 무엇을 배웠는가 (Takeaway)

#### 💡 주니어를 위한 Nginx 부하분산 3단계 원리

실무에서 맨땅에 `nginx.conf`를 고칠 때 머릿속으로 떠올려야 하는 핵심 메커니즘 3단계는 다음과 같다.

1. **Upstream (뒤에서 일할 요리사 팀 등록)**: 맛집 매니저(Nginx)가 주방에서 실제 요리를 할 정직원 3명의 이름표와 자리 번호(`server stapp01:8087;`)를 노트에 적어두는 과정이다. `backend_servers`라는 팀 이름을 임의로 정의한다.

2. **Listen (매장 정문 열기)**: 사용자들이 치고 들어올 대표 통로인 `80번 포트`를 활짝 열고 대기하는 단계다. 사용자가 주소창에 도메인을 치면 무조건 이 정문으로 들어오게 된다.

3. **Proxy_pass (손님을 주방장에게 토스)**: 정문으로 손님이 들어오자마자(`location /`), 안내원(Nginx)이 직접 일하지 않고 아까 등록한 주방장 팀에게 손님을 밀어주는 연결 고리(`proxy_pass http://backend_servers;`) 장치다.

#### ⚠️ 주니어가 실무에서 가장 많이 터트리는 디버깅 포인트

- **중괄호(`{ }`) 러시아 인형 구조**: Nginx 설정은 큰 상자 안에 작은 상자가 계속 들어가는 계층형 구조다. `http` 상자 안에 `upstream`과 `server` 상자가 들어가고, `server` 상자 안에 `location` 상자가 들어간다.
- `vi` 편집기로 수동 수정을 할 때는 항상 열린 괄호(`{`)와 닫힌 괄호(`}`)의 짝이 완벽하게 맞는지 눈으로 라인을 정렬하며 확인해야 한다. 편집이 끝나면 수동으로 서비스를 올리지 말고, 무조건 `sudo nginx -t` (문법 검사기)를 돌려 시스템의 오케이 신호(`syntax is ok`)를 확인하는 습관이 휴먼 에러를 방지하는 가장 확실한 방어벽이다.
