## 📅 Day 13: IPtables 설치 및 구성 (EL9 환경)

### 1. Task

---

- **요구사항**: Stratos DC의 Nautilus 인프라 내 모든 App 호스트에 보안 레이어를 추가하기 위해 방화벽 설정 작업을 수행한다. Apache 웹 서버 포트(`6100`)로 인입되는 비인가 트래픽을 차단하되, 부하분산기(LBR) 호스트로부터의 접근만 허용해야 하며 시스템 재부팅 후에도 해당 방화벽 규칙이 영구적으로 유지되도록 설정한다.

- **대상**: Nautilus App 호스트 전수 (`stapp01`, `stapp02`, `stapp03`), LBR(Load Balancer) 호스트

- **목표**:
  1. 각 App 호스트에 `iptables` 및 `iptables-services` 패키지 설치 및 서비스 활성화
  2. LBR 호스트의 IP(`10.244.189.207`)로부터 유입되는 `6100` 포트 트래픽 허용(ACCEPT) 규칙 정의
  3. 그 외 모든 호스트로부터 `6100` 포트로 들어오는 트래픽 차단(DROP) 규칙 정의
  4. Enterprise Linux 9(EL9) 환경에 맞춘 방화벽 규칙 영구 저장 및 무결성 검증

### 2. Workflow

---

```text
[ 외부 / 타 호스트 트래픽 ] ----> ( Port: 6100 ) ----> [ iptables: DROP ] (차단)
                                                        |
[ LBR Host (10.244.189.207) ] -> ( Port: 6100 ) ----> [ iptables: ACCEPT ] (허용)
                                                        |
                                                        v
                                             [ Apache Web Server ]
```

### 3. 해결 과정 (Troubleshooting & Action)

---

#### 3-1. OS 버전 확인 및 의존성 패키지 설치

시스템의 OS 버전을 확인하고, 누락된 iptables 커널 도구와 서비스 패키지를 동시 설치한다.

```bash
# 1. 현재 OS 배포판 및 버전 확인 (Enterprise Linux 9 계열 확인)
cat /etc/os-release

# 2. 서비스 레이어 패키지 통합 설치
sudo yum install -y iptables iptables-services

# 3. 방화벽 서비스 시작 및 부팅 시 자동 시작 등록
sudo systemctl start iptables
sudo systemctl enable iptables
```

#### 3-2. 방화벽 규칙 구성 및 EL9 표준 영구 저장

LBR IP를 기반으로 규칙 순서(Rule Priority)가 뒤바뀌지 않도록 최상단 주입(`-I INPUT 1`) 방식을 사용하고, 구형 `service iptables save` 명령 대신 파일 리다이렉션을 통해 설정을 영구 박제한다.

```bash
# 1. 기존에 잘못 들어간 규칙이 있다면 깨끗하게 초기화
sudo iptables -F

# 2. LBR 호스트(10.244.189.207)의 6100 포트 접근을 1순위로 허용
sudo iptables -I INPUT 1 -p tcp -s 10.244.189.207 --dport 6100 -j ACCEPT

# 3. 그 외 모든 대역에서 6100 포트로 오는 트래픽은 DROP 정책으로 차단
sudo iptables -A INPUT -p tcp --dport 6100 -j DROP

# 4. EL9 환경의 표준 방식인 파일 리다이렉션으로 영구 저장
sudo iptables-save | sudo tee /etc/sysconfig/iptables

# 5. 방화벽 서비스 재시작을 통한 규칙 적용 상태 최종 검증
sudo systemctl restart iptables
sudo iptables -L INPUT -n --line-numbers
```

### 4. 무엇을 배웠는가 (Takeaway)

---

- **리눅스 배포판 확인 (`/etc/os-release`)**: 서버 세팅 전에 배포판 버전을 확인하는 것은 '스마트폰의 OS 버전을 체크하는 것'과 같다. 안드로이드 버전에 따라 지원하는 앱이나 설정 메뉴가 다르듯, 리눅스도 EL7, EL8, EL9 등 메이저 버전에 따라 관리 명령어가 완전히 달라진다. 실무 인프라 관점에서는 OS 버전별 호환성을 사전에 체크해야 명령어 미지원으로 인한 서비스 장애나 작업 지연을 방지할 수 있다.
- **iptables (방화벽 시스템)**: iptables는 서버로 들어오는 트래픽을 검사하는 **'공항의 보안 검색대'**와 같다. 여권(출발지 IP)과 목적지(포트 번호)를 확인해서 통과시킬지(ACCEPT) 쫓아낼지(DROP) 결정한다. 특히 규칙의 순서가 중요하므로, 허용 규칙을 차단 규칙보다 무조건 위에 두어야 하는 '우선순위 메커니즘'을 철저히 계산하여 방화벽을 설계해야 한다.
- **영구 저장 메커니즘 (iptables-save)**: 메모리상에 띄워둔 방화벽 규칙은 서버가 꺼지면 사라지는 '칠판에 쓴 낙서'와 같다. 구형 OS에서는 `service iptables save`라는 편리한 도구를 썼지만, 최신 인프라(EL9 등)에서는 이 명령어가 제거되었기 때문에 `iptables-save` 도구를 이용해 설정 파일에 **'직접 인쇄하여 박제'**하는 표준 방식을 숙지하는 것이 인프라의 연속성 측면에서 매우 중요하다.
