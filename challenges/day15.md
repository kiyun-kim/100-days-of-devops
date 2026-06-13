## 📅 Day 15: Nginx 웹 서버 설치 및 자체 서명 SSL 인증서 HTTPS 배포

### 1. Task

---

- **요구사항**: xFusionCorp Industries의 애플리케이션 배포 전 작업을 위해 Stratos 데이터 센터의 **`App Server 3`** 환경을 준비한다. Nginx 웹 서버를 안전하게 가동하고, 임시 경로에 존재하는 자체 서명 SSL 인증서와 키를 표준 보안 경로로 이동시킨 뒤 HTTPS 보안 프로토콜을 활성화한다. 또한 웹 서비스의 정상 작동 확인을 위한 인덱스 콘텐츠를 생성한다.

- **대상**: Nautilus App 호스트 3 (`stapp03`)

- **목표**:
  1. `stapp03` 호스트 내 Nginx 패키지 설치 및 서비스 자동 활성화
  2. 임시 경로(`/tmp`)의 `nautilus.crt` 및 `nautilus.key` 파일을 적절한 위치로 이동 및 유실 복구
  3. Nginx 설정 파일(`/etc/nginx/nginx.conf`)을 수동 편집(vi)하여 SSL 443 포트 및 인증서 경로 매핑
  4. 웹 루트 디렉토리에 `Welcome!` 문구를 포함한 `index.html` 생성 및 `curl`을 통한 최종 HTTPS 통신 무결성 검증

### 2. Workflow

---

```text
[ 임시 경로 파일 확인 ] ----> [ 오타 경로 유실 트러블슈팅 ] ----> [ 보안 표준 경로 배치 완료 ]
                                                                      |
                                                                      v
[ 점프 호스트 최종 검증 ] <--- [ Nginx 구동 및 인덱스 생성 ] <--- [ nginx.conf 수동 편집 ]
  ( HTTP/1.1 200 OK )

```

### 3. 해결 과정 (Troubleshooting & Action)

---

#### 3-1. Nginx 설치 및 오타로 인한 경로 유실 복구

Nginx를 설치한 후 인증서 이동 과정에서 발생한 오타(파일명 오인식) 문제를 감지하고 정상 경로로 원복 배치를 완료한다.

```bash
# 1. App Server 3 원격 접속 및 Nginx 설치
ssh banner@stapp03
sudo yum install -y nginx

# 2. 인증서 이동
sudo mkdir -p /etc/nginx/ssl
sudo mv /etc/nginx/s /etc/nginx/ssl/nautilus.key
sudo mv /tmp/nautilus.crt /etc/nginx/ssl/

# 3. 파일 유실 여부 리스트 최종 검증 (무결성 확보)
sudo ls -l /etc/nginx/ssl/

```

#### 3-2. Nginx 웹 서버 HTTPS 바인딩 및 파일 수동 편집

기본 설정 파일에서 주석 처리되어 꼬여있던 TLS 블록을 정리하고, 기존 서버 컨텍스트 안에 IPv4/IPv6 사설 대역을 위한 포트 정의와 인증서 위치를 명시한다.

```bash
# 1. vi 편집기를 활용하여 Nginx 주 설정 파일 진입
sudo vi /etc/nginx/nginx.conf

# 2. [vi 내부 편집 파일 수정 사항] 기존 server 블록 내부에 HTTPS 포트 및 인증서 경로 강제 병합
server {
    listen       80 default_server;
    listen       [::]:80 default_server;

    # HTTPS 보안 포트 추가 지정
    listen       443 ssl default_server;
    listen       [::]:443 ssl default_server;

    server_name  _;
    root         /usr/share/nginx/html;

    # SSL 자체 서명 인증서 및 개인키 체인 경로 연동
    ssl_certificate "/etc/nginx/ssl/nautilus.crt";
    ssl_certificate_key "/etc/nginx/ssl/nautilus.key";

    # ... 기존 내부 구문 유지 ...
}

```

#### 3-3. 콘텐츠 생성 및 최종 통신 검증

```bash
# 1. Nginx 문서 루트에 요구사항 콘텐츠 생성
echo "Welcome!" | sudo tee /usr/share/nginx/html/index.html

# 2. Nginx 환경설정 문법 무결성 검사 및 엔진 재시작
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl enable nginx
exit

# 3. [★최종 검증] 점프 호스트 상태에서 대상 서버 암호화 통신 확인
curl -Ik https://stapp03/

```

_터미널 확인 결과 `HTTP/1.1 200 OK` 및 SSL 웹 서버 정보 정상 표출 완료._

### 4. 무엇을 배웠는가 (Takeaway)

---

- **SSL (Secure Sockets Layer)의 본질**: SSL은 **인터넷상에서 데이터를 주고받을 때 내용을 아무도 훔쳐보지 못하게 포장**하는 '암호화 금고 상자'와 같다. 일반적인 HTTP 통신은 편지봉투 없이 엽서에 글을 써서 보내는 것과 같아서 중간에 누구나 읽을 수 있지만, SSL을 적용하면 암호화된 특수 상자에 데이터를 넣어 보내기 때문에 해킹 위험을 원천 차단한다. 실무 인프라 관점에서는 고객의 개인정보와 결제 데이터를 안전하게 보호하기 위해 웹 서비스 배포 시 무조건 적용해야 하는 표준 보안 기술이다.

- **`listen 80` 및 `[::]:80` 설정의 이유**: Nginx 설정 파일에 기재된 `listen 80`은 옛날 통신 규격인 IPv4 주소 환경에서 대기하라는 명령이고, `[::]:80`은 최신 통신 규격인 IPv6 주소 환경에서도 트래픽을 받을 수 있도록 문을 열어두는 것이다. 이는 '정문에 일반 도로(IPv4) 연결 통로와 지하철(IPv6) 연결 통로를 모두 개방하는 것'과 같다. 둘 중 하나라도 누락되면 특정 네트워크 환경을 사용하는 사용자가 서버에 접속하지 못하는 장애가 발생하므로 인프라의 확장성 측면에서 두 프로토콜 문을 모두 열어두는 것이 핵심이다.

- **인증서(`crt`)와 개인키(`key`) 경로 매핑의 필수성**: 웹 서버에 SSL 암호화를 켜는 것은 '문에 번호키 도어락을 설치하는 것'과 같다. 이때 `ssl_certificate` (인증서)는 외부에 우리 집이 맞다고 보여주는 '도어락 기계 본체'이고, `ssl_certificate_key` (개인키)는 그 도어락을 열 수 있는 전세계에 단 하나뿐인 '비밀번호/마스터키'이다. Nginx 엔진이 실행될 때 이 두 파일의 위치를 정확하게 찾지 못하면 도어락 시스템 자체가 정상 작동하지 않아 문을 잠글 수 없으므로(웹 서버 구동 불가), 서버 내부의 안전한 절대 경로를 설정 파일에 한 치의 오차도 없이 연동해 주어야 한다.
