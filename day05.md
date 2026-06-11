## 📅 Day 05: Linux 보안 아키텍처 - SELinux 패키지 설치 및 영구 비활성화(Disabled) 설정

### 1. Task

- **요구사항**: xFusionCorp Industries 보안 감사에 따른 인프라 표준 보안 엔진(SELinux) 도입 및 사전 환경 검증
- **대상 인프라**: App Server 3 (`stapp03`)
- **목표**: 전사적인 SELinux 적용에 앞서, 애플리케이션 호환성 테스트를 위해 **패키지 설치 후 시스템을 안전하게 영구 비활성화(disabled)** 상태로 구성
- **특이사항**: 금일 야간 정기 점검 재부팅(Maintenance Reboot)이 예정되어 있으므로, 런타임 제어가 아닌 **영구 설정 파일 기반의 변경**이 필수적임.

### 2. 해결 과정 (Troubleshooting & Action)

#### 2-1. 패키지 매니저(`yum`)를 통한 SELinux 코어 요소 설치

SELinux 운영 및 검증에 필요한 핵심 정책(Policy) 패키지와 유틸리티 도구들을 설치함.

```bash
sudo yum install -y selinux-policy selinux-policy-targeted libselinux-utils
```

💡 Note: -y 옵션을 주어 설치 과정 중의 Prompt(확인창)를 자동 승인함으로써 배포 프로세스를 효율화

#### 2-2. 구성 파일 설정을 통한 영구 비활성화 (/etc/selinux/config)

시스템이 재부팅될 때 커널 레벨에서 SELinux를 끄도록 환경 설정 파일을 수정함. `vi` 편집기로 해당 경로의 파일을 열어 기존 설정을 `disabled`로 변경함.

```bash
sudo vi /etc/selinux/config
```

[수정 내역]

```bash
# 파일 내부의 SELINUX 파라미터를 아래와 같이 수정
SELINUX=disabled
```

#### 2-3. 설정 무결성 검증 (Cross Validation)

과제 요구사항에 따라 당장 서버를 리부팅할 필요는 없으므로, 파일에 설정 값이 정상적으로 주입되었는지 `cat`과 `grep` 파이프라인 명령어로 최종 교차 검증함.

```bash
cat /etc/selinux/config | grep SELINUX=
# Output: SELINUX=disabled
```

### 3. 무엇을 배웠는가 (Takeaway)

#### SELinux의 역할과 실무에서의 트러블슈팅 경험

- **SELinux란?**: 리눅스 커널 수준에서 파일, 프로세스, 포트에 대해 깐깐하게 접근 권한을 통제하는 강제 접근 제어(MAC) 시스템이다.

- **실무적인 관점**: 실제 인프라 환경에서 새로운 웹 서버나 애플리케이션을 배포했을 때 원인 모를 `403 Forbidden` 에러나 프로세스 기동 실패를 겪는다면, 높은 확률로 이 SELinux가 권한을 차단하고 있는 경우가 많다.

- **비활성화(Disabled)의 의미**: 보안상 항상 켜두는 것이 이상적이지만, 대규모 시스템 테스트나 특정 레거시 앱과의 호환성을 검증할 때는 이처럼 영구 비활성화(`disabled`) 혹은 경고만 남기는 `permissive` 모드로 전환하여 인프라를 격리 분석하는 트러블슈팅 절차가 빈번히 발생함을 배웠다.

- **설정 파일 관리의 중요성**: `setenforce 0` 명령어로 현재 켜져 있는 상태를 임시로 끌 수도 있지만, 이 방식은 서버가 리부팅되면 초기화된다. 인프라 엔지니어는 반드시 시스템의 지속성(Persistence)을 고려하여 `/etc/selinux/config` 같은 영구 설정 파일을 제어해야 한다는 리눅스 운영의 기본기를 다질 수 있었음.
