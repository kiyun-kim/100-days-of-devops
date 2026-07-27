## 📅 Day 51: Kubernetes Deployment 롤링 업데이트(Rolling Update)를 통한 애플리케이션 무중단 배포

### 1. Task

- **요구사항**: Nautilus 애플리케이션 개발 팀의 최신 웹 기능 수정 사항을 반영하기 위해, 쿠버네티스 클러스터 상에서 구동 중인 `nginx-deployment` 디플로이먼트의 이미지를 무중단 방식인 롤링 업데이트(Rolling Update) 기법을 활용하여 `nginx:1.19` 버전으로 교체 배포합니다.

- **목표**: Jump Host(`thor`) 환경의 `kubectl` CLI 유틸리티를 활용하여 기존 구동 중인 디플로이먼트의 상태를 점검하고, 새로운 이미지 버전으로 무중단 교체를 수행한 후 전체 파드의 가용성을 최종 검증합니다.
  1. 대상 디플로이먼트인 `nginx-deployment` 상태 및 현재 적용된 이미지를 확인합니다.
  2. 디플로이먼트 내 컨테이너 이미지를 `nginx:1.19` 버전으로 업데이트 명령을 실행합니다.
  3. `kubectl rollout status` 명령을 통해 교체 작업의 진행 상태를 모니터링합니다.
  4. 롤아웃 완료 후, 모든 신규 파드(Pod)가 `1/1 READY` 및 `Running` 상태로 무결하게 전환되었는지 확인합니다.

---

### 2. Workflow

```text
  [ Engineer / Jump Host (thor) ]
                 │
                 │ (kubectl set image deployment/nginx-deployment nginx-container=nginx:1.19)
                 ▼
  [ Kubernetes Control Plane (API Server) ]
                 │
                 ├── [ Rolling Update Triggered ]
                 │
                 ├── Creates: [ New ReplicaSet (nginx:1.19) ] ───> Scale Up (1 -> 2 -> 3 Pods)
                 │                                                          ▲
                 │                                                          │ Zero Downtime
                 │                                                          ▼
                 └── Terminates: [ Old ReplicaSet (Old Image) ] ──> Scale Down (3 -> 2 -> 0 Pods)
                                    │
                                    ▼
                 [ Final State: 3 x Pods running nginx:1.19 ]

```

---

### 3. 해결 과정 (Action)

#### 3-1. 기존 디플로이먼트 상태 및 이미지 확인

작업 시작 전 대상 디플로이먼트의 구동 상태 및 지정된 컨테이너 명칭과 기존 베이스 이미지를 조회합니다.

```bash
# 디플로이먼트 구동 상태 및 속성 확인
kubectl get deployment nginx-deployment -o wide

# 디플로이먼트 내 컨테이너 명칭 및 현재 이미지 버전 조회
kubectl describe deployment nginx-deployment | grep -E "Containers:|Image:"

```

#### 3-2. 롤링 업데이트 명령 실행

`kubectl set image` 커맨드를 활용하여 `nginx-deployment` 내부의 `ng*` 이미지를 신규 사양인 `nginx:1.19`로 업데이트합니다.

```bash
# nginx-deployment 디플로이먼트의 컨테이너 이미지를 nginx:1.19로 업데이트
kubectl set image deployment/nginx-deployment nginx-container=nginx:1.19

```

#### 3-3. 롤아웃 진행 상태 모니터링

새로운 인스턴스가 생성되고 기존 파드가 점진적으로 제거되는 롤아웃 파이프라인의 진행 상태를 실시간 확인합니다.

```bash
# 롤아웃 진행 상태 및 완료 여부 모니터링
kubectl rollout status deployment/nginx-deployment

```

#### 3-4. 파드 정상 동작 및 이미지 변경 최종 검증

신규 파드들이 모두 정상 기동 상태(`Running`)에 도달했는지, 그리고 변경된 명세가 정확히 반영되었는지 정밀 대조합니다.

```bash
# 새로 교체된 신규 파드들의 구동 상태(READY 1/1, STATUS Running) 조회
kubectl get pods

# 디플로이먼트의 이미지 명세가 nginx:1.19로 변경되었는지 최종 확인
kubectl describe deployment nginx-deployment | grep Image:

```

---

### 4. 핵심 개념 정리

- **Rolling Update (롤링 업데이트)**: 기존 버전의 파드를 한 번에 전부 삭제하지 않고, 새 버전의 파드를 하나씩 순차적으로 생성하면서 기존 파드를 단계를 밟아 교체(Scale Up & Scale Down)해 나가는 무중단 서비스 배포 전략입니다.

  > 💡 비유하자면, 24시간 운행하는 대형 버스(서비스)의 타이어 4개를 한 번에 빼서 교체하느라 버스를 정차시키는 대신, 버스가 천천히 달리는 동안 바퀴를 하나씩 새것으로 교체하여 승객(사용자)이 서비스 중단을 전혀 느끼지 못하게 하는 가동 유지 방식과 같습니다.

- **Rollout History & Rollback (롤아웃 이력 및 롤백)**: 디플로이먼트를 통해 실행된 모든 이미지 및 스펙 변경 사항은 클러스터 내부의 ReplicaSet 이력(Revision)에 기록됩니다. 만약 신규 배포된 이미지에 치명적인 장애가 발생할 경우, `kubectl rollout undo` 명령을 통해 이전의 안정적인 버전으로 손쉽게 되돌릴 수 있는 안전장치를 제공합니다.

  > 💡 비유하자면, 문서 작업 중 오타나 에러가 났을 때 키보드의 `Ctrl + Z`를 눌러 에러가 발생하기 바로 직전의 가장 깨끗했던 작업 상태로 즉시 타임슬립시키는 되돌리기 버튼과 같습니다.

- **Zero Downtime Deployment (무중단 배포)**: 애플리케이션의 신규 버전 출시나 보안 패치 적용 시, 사용자의 접속을 끊거나 점검 페이지를 띄우지 않고 100% 가용성(Availability)을 유지하면서 새로운 시스템으로 유연하게 전환하는 현대적 배포 방법론입니다.

  > 💡 비유하자면, 24시간 운영하는 교차로에서 신호등 교체 공사를 할 때, 도로 전체를 통제하지 않고 가변 차선을 열어 차들을 우회시킨 뒤 공사가 끝난 차선부터 순차적으로 해제하여 교통 흐름을 멈추지 않는 공사 방식과 같습니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **회고(Retrospective)**: 전통적인 온프레미스(On-Premises) 환경에서 서비스를 업데이트할 때 필수적으로 요구되던 '새벽 작업'과 '점검 공지' 방식에서 벗어나, 쿠버네티스의 선언적 오케스트레이션을 통한 무중단 롤링 업데이트 체계를 직접 체감했습니다. 단 한 줄의 커맨드로 복잡한 레프리카셋 스케일링이 알아서 조율되는 과정에서 쿠버네티스 컨트롤러의 강력함을 깊이 깨달았습니다.

- **Pain Point**: 실무 배포 환경에서 잘못된 이미지 태그나 애플리케이션 런타임 오류가 포함된 신규 버전을 배포하게 되면 파드가 `CrashLoopBackOff` 상태에 빠지게 됩니다. 이때 적절한 준비성 검사(Readiness Probe) 설정이 없으면 신규 파드로 트래픽이 유입되어 대규모 서비스 장애로 이어집니다. 롤링 업데이트 수행 시 항상 `rollout status`로 상태를 추적하고, 이상 발생 시 즉시 `rollout undo`로 복구할 수 있는 대처 능력이 중요함을 체감했습니다.

- **성장 포인트**: 단순 컨테이너 생성과 배포 수준을 넘어, 엔터프라이즈 운영 환경의 핵심 요구사항인 '서비스 연속성'과 '무중단 라이프사이클 관리'를 경험했습니다.
