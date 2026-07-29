## 📅 Day 52: Kubernetes Deployment 롤백(Rollback)을 통한 이전 안정 버전 복구

### 1. Task

- **요구사항**: Nautilus 애플리케이션 개발 팀이 최신 배포한 신규 버전에서 고객 버그 리포트가 수집됨에 따라, 시스템 장애 확산을 방지하기 위해 구동 중인 `nginx-deployment` 디플로이먼트를 즉시 이전 리비전(Previous Revision)으로 복구(Rollback) 조치합니다.

- **목표**: Jump Host(`thor`) 환경의 `kubectl` CLI 유틸리티를 활용하여 디플로이먼트의 롤아웃 이력을 조회하고, 롤백 명령어를 통해 서비스 가용성을 보장하며 복구를 완료합니다.
  1. `nginx-deployment` 디플로이먼트의 변경 이력(Revision History)을 조회합니다.
  2. `kubectl rollout undo` 명령으로 이전 버전으로의 복구 작업을 가동합니다.
  3. 롤아웃 상태 모니터링을 통해 정상 롤백 완료 여부를 확인합니다.
  4. 복구된 3개의 파드(Pod)가 `1/1 READY` 및 `Running` 상태에 도달했는지, 복구된 이미지(`nginx:1.16`) 스펙을 최종 검증합니다.

---

### 2. Workflow

```text
  [ Engineer / Jump Host (thor) ]
                 │
                 │ (kubectl rollout undo deployment/nginx-deployment)
                 ▼
  [ Kubernetes Control Plane (API Server) ]
                 │
                 ├── [ Rollback Triggered ]
                 │
                 ├── Activates: [ Previous ReplicaSet (Revision 1: nginx:1.16) ] ───> Scale Up (3 Pods)
                 │                                                                         ▲
                 │                                                                         │ Fast Revert
                 │                                                                         ▼
                 └── Deactivates: [ Faulty ReplicaSet (Revision 2: nginx:alpine-perl) ] ──> Scale Down (0 Pods)
                                    │
                                    ▼
                 [ Final State: 3 x Pods running nginx:1.16 ]

```

---

### 3. 해결 과정 (Action)

#### 3-1. 디플로이먼트 롤아웃 변경 이력(History) 조회

작업 시작 전 `rollout history` 커맨드를 수행하여 복구 가능한 리비전 버전 목록과 변경 내역을 확인합니다.

```bash
# 디플로이먼트 변경 이력 및 리비전 번호 확인
kubectl rollout history deployment/nginx-deployment

```

- **결과 확인**:
- `REVISION 1`: 초기 버전 (`<none>`)
- `REVISION 2`: 버그가 포함된 최신 버전 (`nginx:alpine-perl`)

#### 3-2. 이전 리비전으로 롤백(Rollback) 수행

`kubectl rollout undo` 명령어를 실행하여 디플로이먼트를 이전 리비전(Revision 1) 상태로 즉시 복구합니다.

```bash
# 이전 리비전(Previous Revision)으로 복구 실행
kubectl rollout undo deployment/nginx-deployment

```

#### 3-3. 롤아웃 진행 상태 모니터링 및 복구 완결 검증

파드 스케일링 체인지가 완료되는 과정을 모니터링하고, 성공 메시지(`successfully rolled out`)를 수신합니다.

```bash
# 롤백 완료 상태 트래킹
kubectl rollout status deployment/nginx-deployment

```

#### 3-4. 파드 정상 동작 및 적용 이미지 최종 검증

신규 교체된 3개의 파드가 모두 `Running` 상태로 전환되었는지 확인하고, 이미지 버전이 이전의 안정 버전(`nginx:1.16`)으로 원상 복구되었는지 검증합니다.

```bash
# 신규 생성된 파드들의 구동 상태(1/1 READY, STATUS Running) 확인
kubectl get pods

# 디플로이먼트의 이미지가 nginx:1.16으로 원상 복구되었는지 최종 검증
kubectl describe deployment nginx-deployment | grep Image:

```

---

### 4. 핵심 개념 정리

- **Rollback (`kubectl rollout undo`)**: 배포된 신규 버전에서 장애나 버그가 발견되었을 때, 이전의 정상 작동하던 ReplicaSet 이력(Revision)을 재활용하여 시스템을 빠르게 이전 상태로 복원하는 기능입니다.

  > 💡 비유하자면, 스마트폰 OS 업데이트 후 앱들이 계속 튕기는 현상이 생겼을 때, 서비스 센터에 가지 않고도 상단 알림창의 '이전 OS 버전으로 되돌리기' 버튼을 눌러 바로 어제 쓰던 안정적인 상태로 즉시 복구하는 안전장치와 같습니다.

- **Revision History Limit (리비전 이력 보존)**: Kubernetes Deployment가 과거에 실행했던 매니페스트 변경 이력을 ReplicaSet 형태로 클러스터에 저장해 두는 개수 제한 설정(`spec.revisionHistoryLimit`)입니다.

  > 💡 비유하자면, 타임머신을 탈 때 언제든 돌아갈 수 있는 과거 저장점(Save Point)을 최근 10개까지 자동으로 보존해 주는 게임의 자동 저장 슬롯 시스템과 같습니다.

- **Fast Recovery / MTTR (최소 평균 복구 시간)**: 장애 발생 시 원인을 파악하고 코드를 새로 수정하여 재빌드/재배포하는 긴 과정(Fix-Forward) 대신, 이미 검증된 이전 컨테이너 이미지를 즉시 재가동하여 서비스 평균 복구 시간(MTTR, Mean Time To Repair)을 극단적으로 단축시키는 운영 전략입니다.

  > 💡 비유하자면, 운전 중 타이어에 펑크가 났을 때 길가에서 펑크 난 타이어를 때우려고 시간을 허비하는 대신, 트렁크에 미리 준비해 둔 멀쩡한 스페어타이어로 즉시 교체하여 바로 출발하는 것과 같습니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **회고(Retrospective)**: 실무 배포 현장에서 발생할 수 있는 긴급 버그 상황에 대처하여 `kubectl rollout undo` 명령 하나로 수초 만에 인프라 전체를 이전 안정 버전으로 완전 복구하는 경험을 다졌습니다. 빌드 파이프라인을 처음부터 다시 돌리지 않고도 쿠버네티스의 ReplicaSet 이력을 활용해 즉각 복구하는 가용성 중심 오케스트레이션의 강력함을 체감했습니다.

- **Pain Point**: 실무에서 디플로이먼트 업데이트 시 `--record` 옵션이나 적절한 롤아웃 어노테이션(Change-Cause)을 남기지 않으면, `rollout history` 조회 시 각 리비전이 어떤 변경점이었는지 구분하기 힘들어 올바른 리비전 번호로 롤백하는 데 차질이 생깁니다. 변경 이력을 투명하게 관리하는 문서화 습관이 긴급 장애 대응 시 중요함을 체감했습니다.

- **성장 포인트**: 단순 배포 체계를 넘어, 장애 발생 시 서비스 중단 시간을 최소화(Zero/Minimal Downtime)하고 시스템을 원상 복구할 수 있는 무중단 운영 및 장애 대처(Disaster Recovery) 능력을 직접 체감하며, 쿠버네티스의 선언적 오케스트레이션이 제공하는 안정성과 유연성을 깊이 이해했습니다.
