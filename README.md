# 🚀 100 Days of DevOps Challenge

이 저장소는 KodeKloud에서 제공하는 [**100 Days of DevOps Challenge**](https://kodekloud.com/100-days-of-devops)를 수행하며 학습한 내용을 기록합니다.

가상 인프라 운영 시나리오를 바탕으로 Linux 시스템 관리, 보안, 자동화, 데이터베이스, 웹 서비스, Git 등의 과제를 해결합니다. 각 기록에는 요구사항, 작업 과정, 검증 방법과 회고를 담아 단순한 명령어 모음이 아닌 문제 해결 과정 중심으로 정리하고 있습니다.

## 📊 챌린지 진행 현황판

> **진행률: 33 / 100일 (33%)** · ✅ 완료 · ⏳ 예정

### 사용자 및 보안 설정

| Day | Topic                                                          | Tech Stack | Status |
| :-: | :------------------------------------------------------------- | :--------- | :----: |
| 01  | [Linux 사용자 생성 및 그룹 권한 관리](./challenges/day01.md)   | `Linux`    |   ✅   |
| 02  | [Linux 임시 사용자 계정 및 만료일 설정](./challenges/day02.md) | `Linux`    |   ✅   |
| 03  | [Root SSH 직접 로그인 차단](./challenges/day03.md)             | `Linux`    |   ✅   |

### 리눅스 권한 및 기본 보안

| Day | Topic                                                        | Tech Stack | Status |
| :-: | :----------------------------------------------------------- | :--------- | :----: |
| 04  | [Linux 권한 관리 및 스크립트 배포](./challenges/day04.md)    | `Linux`    |   ✅   |
| 05  | [SELinux 설치 및 영구 비활성화 설정](./challenges/day05.md)  | `Linux`    |   ✅   |
| 06  | [분산 서버 Crontab 스케줄러 등록](./challenges/day06.md)     | `Linux`    |   ✅   |
| 07  | [SSH Key 기반 Passwordless 인증 구축](./challenges/day07.md) | `Linux`    |   ✅   |

### 자동화 및 서버 구축

| Day | Topic                                                         | Tech Stack        | Status |
| :-: | :------------------------------------------------------------ | :---------------- | :----: |
| 08  | [Ansible 설치 및 환경 구성](./challenges/day08.md)            | `Linux` `Ansible` |   ✅   |
| 09  | [MariaDB 트러블슈팅 및 서비스 복구](./challenges/day09.md)    | `Linux`           |   ✅   |
| 10  | [쉘 스크립트 기반 원격 백업 자동화](./challenges/day10.md)    | `Linux` `Bash`    |   ✅   |
| 11  | [Tomcat 설치 및 WAR 애플리케이션 배포](./challenges/day11.md) | `Linux`           |   ✅   |

### 네트워크 및 프로세스

| Day | Topic                                                     | Tech Stack     | Status |
| :-: | :-------------------------------------------------------- | :------------- | :----: |
| 12  | [Apache 포트 충돌 프로세스 해결](./challenges/day12.md)   | `Linux`        |   ✅   |
| 13  | [IPtables 설치 및 구성 (EL9 환경)](./challenges/day13.md) | `Linux`        |   ✅   |
| 14  | [Linux Process 트러블슈팅](./challenges/day14.md)         | `Linux` `Bash` |   ✅   |

### 웹 서버 및 애플리케이션 환경 구축

| Day | Topic                                                                            | Tech Stack       | Status |
| :-: | :------------------------------------------------------------------------------- | :--------------- | :----: |
| 15  | [Nginx 웹 서버 설치 및 자체 서명 SSL 인증서 HTTPS 배포](./challenges/day15.md)   | `TLS/SSL`        |   ✅   |
| 16  | [Nginx 로드밸런서(LBR) 구축 및 트래픽 부하분산](./challenges/day16.md)           | `Load Balancing` |   ✅   |
| 17  | [PostgreSQL 데이터베이스 유저 생성 및 권한 배포](./challenges/day17.md)          | `Database`       |   ✅   |
| 18  | [MariaDB 패키지 설치 및 신규 데이터베이스·유저 권한 배포](./challenges/day18.md) | `Database`       |   ✅   |
| 19  | [Apache 정적 웹사이트 배포 및 원격 데이터 마이그레이션](./challenges/day19.md)   | `Deployment`     |   ✅   |
| 20  | [Configure Nginx + PHP-FPM Using Unix Sock](./challenges/day20.md)               | `Web Stack`      |   ✅   |

### Git 버전 관리 및 협업

| Day | Topic                                                                                  | Tech Stack | Status |
| :-: | :------------------------------------------------------------------------------------- | :--------- | :----: |
| 21  | [Storage 서버 내 중앙 집중형 Git Bare 저장소 구축 및 환경 설정](./challenges/day21.md) | `Git`      |   ✅   |
| 22  | [Storage Server에서 Git 저장소 복제](./challenges/day22.md)                            | `Git`      |   ✅   |
| 23  | [Git 저장소 Fork](./challenges/day23.md)                                               | `Git`      |   ✅   |
| 24  | [Storage 서버 내 Git 저장소 신규 브랜치 생성 및 소유권 보안](./challenges/day24.md)    | `Git`      |   ✅   |
| 25  | [Git Merge Branches](./challenges/day25.md)                                            | `Git`      |   ✅   |
| 26  | [Git Manage Remotes](./challenges/day26.md)                                            | `Git`      |   ✅   |
| 27  | [Git Revert Some Changes](./challenges/day27.md)                                       | `Git`      |   ✅   |
| 28  | [Git Cherry Pick](./challenges/day28.md)                                               | `Git`      |   ✅   |
| 29  | [Manage Git Pull Requests](./challenges/day29.md)                                      | `Git`      |   ✅   |
| 30  | [Git hard reset](./challenges/day30.md)                                                | `Git`      |   ✅   |
| 31  | [Git Stash](./challenges/day31.md)                                                     | `Git`      |   ✅   |
| 32  | [Git Rebase](./challenges/day32.md)                                                    | `Git`      |   ✅   |
| 33  | [Resolve Git Merge Conflicts ](./challenges/day33.md)                                  | `Git`      |   ✅   |

### Docker Containerization

### Kubernetes Orchestration

### Jenkins CI/CD 파이프라인 자동화

### Ansible 구성 관리 (IaC)

### Terraform & AWS 인프라 자동화
