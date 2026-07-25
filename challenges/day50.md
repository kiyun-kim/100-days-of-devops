## 📅 Day 50: Kubernetes Pod 리소스 제한(Requests & Limits) 선언적 설정

### 1. Task

- **요구사항**: Nautilus DevOps 팀의 쿠버네티스 클러스터 자원 관리 최적화 계획에 따라, 특정 컨테이너가 워커 노드의 CPU 및 메모리 자원을 과도하게 점유하여 타 애플리케이션의 성능 저하를 일으키는 현상을 방지하고자 컨테이너 수준의 리소스 요청량(Requests) 및 제한량(Limits)을 선언적으로 설정하여 배포합니다.

- **목표**: Jump Host(`thor`) 환경의 `kubectl` CLI 유틸리티를 활용하여 요구사항에 맞는 리소스 제약 조건이 포함된 매니페스트(YAML)를 작성하고 클러스터 상에 정상 배포합니다.
  1. Pod 명칭을 `httpd-pod`로 지정합니다.
  2. Pod 내부 컨테이너 명칭을 `httpd-container`로 지정하고, 베이스 이미지를 `httpd:latest`로 적용합니다.
  3. 컨테이너의 리소스 요청량(Requests)을 Memory `15Mi`, CPU `100m`로 설정합니다.
  4. 컨테이너의 리소스 제한량(Limits)을 Memory `20Mi`, CPU `100m`로 설정합니다.

---

### 2. Workflow

```text
  [ Engineer / Jump Host (thor) ]
                 │
                 │ (kubectl apply -f pod.yaml)
                 ▼
  [ Kubernetes Control Plane (API Server) ]
                 │
                 │ (Scheduler: Resource Requests Check)
                 ▼
  [ Kubelet / Worker Node ]
                 │
                 └── [ Pod: httpd-pod ]
                         └── [ Container: httpd-container ]
                                 ├── Image: httpd:latest
                                 ├── Requests: CPU 100m / Memory 15Mi
                                 └── Limits:   CPU 100m / Memory 20Mi

```

---

### 3. 해결 과정 (Action)

#### 3-1. 리소스 제한 스펙이 포함된 선언적 매니페스트 작성 (`pod.yaml`)

요구사항인 컨테이너 수준의 `resources.requests` 및 `resources.limits` 구성을 포함하는 표준 YAML 스펙을 작성합니다.

```bash
# pod.yaml 작성 (리소스 제약 조건 정의)
cat <<EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: httpd-pod
spec:
  containers:
  - name: httpd-container
    image: httpd:latest
    resources:
      requests:
        memory: "15Mi"
        cpu: "100m"
      limits:
        memory: "20Mi"
        cpu: "100m"
EOF

```

- **설정 변경 의도**:
- `spec.containers[0].resources.requests`에 CPU 100m (0.1 코어) 및 메모리 15MiB를 할당하여 스케줄러가 이 최소 자원을 확보할 수 있는 워커 노드에 파드를배치하도록 유도했습니다.
- `spec.containers[0].resources.limits`에 CPU 100m 및 메모리 20MiB의 임계값을 설정하여 노드의 물리적 스토리지 및 메모리 자원을 보존하도록 격리했습니다.

#### 3-2. Kubernetes 클러스터 상의 선언적 배포 적용

작성된 YAML 형상 파일을 Kubernetes API Server로 전송하여 파드를 생성합니다.

```bash
# 선언적 매니페스트 기반 파드 배포
kubectl apply -f pod.yaml

```

#### 3-3. 파드 구동 및 리소스 메타데이터 정밀 검증

파드가 정상적으로 `Running` 상태에 도달했는지 확인하고, `describe` 커맨드를 활용하여 반영된 Limits 및 Requests 수치를 대조 검증합니다.

```bash
# 파드 구동 상태 조회
kubectl get pod httpd-pod

# Limits 및 Requests 설정값 정밀 검증
kubectl describe pod httpd-pod | grep -A 10 Limits:

```

---

### 4. 핵심 개념 정리

- **Resource Requests (리소스 요청량)**: 쿠버네티스 스케줄러(kube-scheduler)가 파드를 배포할 노드를 선택할 때 고려하는 최소한의 보장된 컴퓨팅 자원(CPU, Memory) 크기입니다. 노드에 요청량 이상의 여유 자원이 있을 때만 파드가 해당 노드에 스케줄링됩니다.

  > 💡 비유하자면, 출장을 갈 호텔(워커 노드)을 예약할 때 "최소한 싱글 침대 하나와 와이파이가 제공되는 방(Requests)"을 요청하여 조건에 맞는 호텔로만 예약 입실하는 것과 같습니다.

- **Resource Limits (리소스 제한량)**: 컨테이너가 런타임 중에 최대로 사용할 수 있는 자원의 상한선입니다. CPU 사용량이 Limit을 초과하면 Throttling(제어)이 발생하며, 메모리 사용량이 Limit을 초과하면 OOMKilled(Out Of Memory) 프로세스에 의해 컨테이너가 즉시 강제 종료 및 재시작됩니다.

  > 💡 비유하자면, 무제한 뷔페에 들어갔을 때 식사 시간이 최대 2시간(Limits)으로 정해져 있어서, 시간을 초과하는 순간 퇴장 조치(OOMKilled) 당하는 안전장치 제한선과 같습니다.

- **m (milliCPU / 밀리코어)**: 쿠버네티스에서 CPU 자원을 측정하는 단위로, `1000m`은 vCPU/물리 코어 1개와 동일합니다. 따라서 `1000m` 단위 하위의 `100m`은 코어 1개의 10%(0.1 CPU) 자원을 의미합니다.

  > 💡 비유하자면, 1L짜리 대용량 우유(1 CPU)를 1,000ml 단위로 나누어 그중 정확히 100ml(100m) 만큼만 컨테이너 전용 컵에 따라주는 분량 개념과 같습니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **회고(Retrospective)**: 단순히 파드를 띄우는 가상화 단계를 지나, 워커 노드의 물리 자원을 지키고 애플리케이션 간 자원 간섭을 차단하는 자원 관리 기법(Resource Management)의 핵심을 학습했습니다. Requests와 Limits의 차이를 직접 매니페스트에 선언하고 검증함으로써 안정적인 클러스터 운영을 위한 필수 요소임을 체감했습니다.

- **Pain Point**: 실무 환경에서 리소스 Limit을 설정하지 않은 파드가 메모리 누수(Memory Leak)를 일으키면, 동일한 노드에서 구동 중이던 인접 파드들까지 OOM으로 함께 다운되는 '지옥의 쇄도 장애(Cascading Failure)'가 발생합니다. 반대로 Requests를 과도하게 크게 설정하면 노드 자원이 실제로는 남아돌아도 스케줄러가 파드를 배치하지 못하는 '자원 파편화' 현상이 초래되므로, 정밀한 리소스 산정이 엔지니어의 핵심 역량임을 깨달았습니다.

- **성장 포인트**: 애플리케이션의 워크로드를 무작정 배포하는 수준을 넘어, 클러스터의 전체 성능과 가용성을 보장하는 다각적 관점의 프로페셔널 Infrastructure Architect로서의 역량을 한층 강화할 수 있었습니다.
