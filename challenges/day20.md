## 📅 Day 20: Configure Nginx + PHP-FPM Using Unix Sock

### 1. Task

- **요구사항**: xFusionCorp Industries의 동적 웹 서비스 호스팅 환경을 고도화하기 위해, 웹 서버(Nginx)와 동적 애플리케이션 프로세스 매니저(PHP-FPM)를 동일 호스트 내부에서 가장 속도가 빠른 유닉스 소켓(Unix Socket) 구조로 결합하고 가용성을 확보합니다.

- **목표**:
  1. `app server 3 (stapp03)` 호스트에 Nginx 웹 서버를 설치하고 가동 포트를 커스텀 포트인 `8096`으로, 웹 도큐먼트 루트는 `/var/www/html`로 지정합니다.
  2. 동일 호스트에 **PHP-FPM 버전 8.3**을 설치하고, 통신 방식을 **`unix socket /var/run/php-fpm/default.sock`** 경로로 강제 매핑합니다. (해당 부모 디렉토리가 없을 시 수동 생성)
  3. 두 인프라 레이어를 유기적으로 결합한 후, 점프 호스트 환경에서 `curl http://stapp03:8096/index.php` 명령어를 날려 정적/동적 호스팅의 최종 무결성을 검증합니다.

---

### 2. Workflow

```text
[ 유저 브라우저 ] ➔ (8096 포트) ➔ [ Nginx 웹 서버 ]
                                         │
                                 (Unix Domain Socket 통신)
                                 /var/run/php-fpm/default.sock
                                         │
                                         ▼
                                 [ PHP-FPM 8.3 WAS ] ➔ [.php 동적 연산]

```

---

### 3. 해결 과정 (Action)

#### 3-1. 패키지 저장소 확장 및 클린 설치

```bash
# 1. App Server 3 (stapp03) 보안 쉘 접속
ssh banner@stapp03

# 2. Rocky Linux 9 환경에 대응하는 PHP 8.3 엔터프라이즈 리포지토리 활성화
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm
sudo dnf module reset php -y
sudo dnf module enable php:remi-8.3 -y

# 3. Nginx 웹 서버 및 PHP-FPM 핵심 데몬 일괄 설치
sudo dnf install -y nginx php-fpm php-cli php-common

```

#### 3-2. WAS 및 웹 서버 설정 제어 (vi 구성)

```bash
# 1. PHP-FPM 프로세스 풀 및 소켓 생성 규칙 정의
sudo vi /etc/php-fpm.d/www.conf
# (내부 listen 주소 오타 수정 및 권한 설정 반영)

# 2. 명세서 지시대로 소켓 파일이 안착할 부모 디렉토리 생성 및 소유권 주입
sudo mkdir -p /var/run/php-fpm
sudo chown -R nginx:nginx /var/run/php-fpm
sudo chmod 755 /var/run/php-fpm

# 3. Nginx 포트 포워딩 및 패스트CGI 소켓 라우팅 연동
sudo vi /etc/nginx/nginx.conf
# (8096 포트 변경 및 location 소켓 패스 매핑)

```

#### 3-3. 서비스 동기화 및 런타임 엔드포인트 검증

```bash
# 1. Nginx 아키텍처 환경 설정 문법 검사
sudo nginx -t

# 2. 동적 스크립트 엔진(PHP-FPM) 기동 후 웹 서버 엔진 순차 가동
sudo systemctl restart php-fpm
sudo systemctl restart nginx
sudo systemctl enable php-fpm nginx

# 3. 인프라 세션 탈출 후 점프 호스트에서 최종 원격 라우팅 가용성 검증
exit
curl http://stapp03:8096/index.php

```

---

### 4. 실무 설정 분석 (vi 편집 상세 내역)

이번 미션의 핵심 성공 요인은 `vi` 편집기를 통해 웹 서버와 WAS 간의 창구를 단 1토큰의 오차도 없이 일치시킨 점입니다.

#### ① PHP-FPM 핵심 바인딩 (`/etc/php-fpm.d/www.conf`)

PHP-FPM이 마스터 프로세스를 띄울 때 권한 충돌을 막기 위해 실행 유저를 `nginx`로 통일하고, 주석(`;`)을 제거하여 소켓 파일 제어권을 명시했습니다.

```ini
user = nginx
group = nginx

; [★오타 주의] 채점 규격에 맞춰 www. 을 제거한 순수 절대 경로 선언
listen = /var/run/php-fpm/default.sock

; 앞에 붙은 세미콜론(;) 주석을 제거하여 Nginx가 소켓을 읽을 수 있도록 허용
listen.owner = nginx
listen.group = nginx
listen.mode = 0660

```

#### ② Nginx 프록시 라우팅 구성 (`/etc/nginx/nginx.conf`)

기본 `80`번 포트를 차단하고, 들어오는 정적/동적 트래픽 중 `.php` 확장자를 가진 요청만 유닉스 소켓으로 토스하도록 분기했습니다.

```nginx
server {
    listen       8096 default_server; # 요구사항 (a) 포트 지정
    listen       [::]:8096 default_server;
    server_name  _;
    root         /var/www/html;       # 도큐먼트 루트 고정
    index        index.php index.html;

    # 동적 PHP 스크립트 연산 요청 처리 블록
    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_intercept_errors on;

        # [★결합점] www.conf에 정의된 소켓 경로와 백분의 일 밀리초의 오차도 없이 일치시킴
        fastcgi_pass unix:/var/run/php-fpm/default.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}

```

---

### 5. 핵심 인프라 개념 정리

#### 📌 데몬 (Daemon) 이란?

사용자가 터미널 창을 닫거나 로그아웃하더라도 시스템 **백그라운드(Background)에서 상시 대기하며 독립적으로 실행되는 프로세스**입니다. 리눅스 서비스 명칭 뒤에 붙는 `d`(`nginx`, `php-fpm`, `sshd`)가 바로 데몬의 약자이며, 클라이언트의 요청이 언제 올지 모르기 때문에 메모리에 상주하며 수신 대기 상태를 유지합니다.

💡 이는 우리가 스마트폰 화면을 닫아도 백그라운드에서 실시간 카카오톡 알림을 받기 위해 대기하고 있는 상시 대기조 앱들과 유사한 개념입니다.

#### 📌 PHP-FPM 이란?

**FastCGI Process Manager**의 약자로, PHP 스크립트 언어를 해석하고 연산하는 동적 어플리케이션 서버(WAS) 엔진입니다. 정적 리소스(HTML/이미지)만 서빙할 수 있는 Nginx를 대신해, 백엔드 비즈니스 로직을 동적으로 연산하여 결과물만 웹 서버에 다시 돌려주는 핵심 백엔드 컴퓨팅 유닛입니다.

💡 웹 서버인 Nginx는 단순 주문 수령과 인테리어를 담당하는 '카운터 직원'이고, PHP-FPM은 "로그인 처리 및 데이터 연산"이라는 복잡한 주문서를 넘겨받아 음식을 조리하는 '주방 요리사'의 포지션을 가집니다.

#### 📌 수신 소켓 (Listen Socket) 이란?

특정 서버 프로세스가 외부 네트워크나 내부의 또 다른 프로세스로부터 **연결 요청(Connection)을 전달받기 위해 열어놓는 대기 창구**입니다.

- **TCP/IP 소켓**: 포트 주소(`127.0.0.1:9000`)를 기반으로 네트워크 프로토콜 스택을 거쳐 통신합니다.

  > 💡 멀리 떨어진 컴퓨터끼리 유선 전화를 연결하는 방식과 같습니다.

- **유닉스 소켓 (Unix Domain Socket)**: 파일 시스템 경로(`/var/run/php-fpm/default.sock`)를 창구로 삼아, 같은 OS 커널 내부에서 프로세스 간 메모리 직접 복사(IPC)로 통신하므로 네트워크 오버헤드가 전무하여 속도가 훨씬 빠릅니다.

  > 💡 같은 건물 내부에서 일하는 카운터 직원(Nginx)과 주방 요리사(PHP-FPM)가 굳이 전화를 쓰지 않고, 벽에 작은 전용 구멍(소켓 파일)을 뚫어 주문서를 즉시 주고받는 밀폐형 초고속 통로 구조와 동일합니다.

---

### 6. 무엇을 배웠는가

- **원점(명세서) 절대 신뢰의 법칙**: 트러블슈팅 과정에서 에러가 발생하면 새로운 우회책이나 임시방편(포트 변경 등)을 구글링하기 전에, **설정 파일 내부에 명세서와 일치하지 않는 미세한 오타(`www.`)가 유실되어 있는지 원천 데이터부터 검증**해야 함을 배웠습니다.

- **레이어 격리(Isolate) 분석**: `curl` 테스트 결과 `404 Not Found`가 떴다는 것은 Nginx 웹 서버 레이어는 정상 작동하나, 소켓 파일 바인딩 미스로 인해 PHP-FPM 레이어와의 커뮤니케이션에 실패했다는 신호임을 인지하는 아키텍처 관점의 분기 판단력을 배양했습니다.

- **리눅스 권한과 프로세스의 상관관계**: 프로세스를 기동하는 주체 유저와 소켓 디렉토리의 소유권(`chown`, `chmod`)이 완벽하게 결합되어야만 데몬이 런타임 오류 없이 안전하게 파일 시스템에 소켓을 밀어 넣을 수 있다는 인프라의 기본 정렬 메커니즘을 마스터했습니다.
