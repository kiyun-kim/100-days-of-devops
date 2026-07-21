## 📅 Day 49: Kubernetes Deployment 선언적 배포 및 Pod 오케스트레이션

### 1. Task

- **요구사항**: Nautilus DevOps 팀의 애플리케이션 수명 주기 관리 및 자율 복구(Self-Healing) 인프라 구축의 일환으로, 단일 Pod 배포 방식을 넘어 상위 오케스트레이션 객체인 Deployment를 선언적 매니페스트(YAML) 스펙으로 정의하여 클러스터에 배포합니다.

- **목표**: Jump Host(`thor`) 환경의 `kubectl` CLI 유틸리티를 활용하여 클러스터 상에 지정된 사양의 Deployment를 선언적 방식으로 구성하고 검증합니다.
  1. Deployment 식별 명칭을 `nginx`로 지정합니다.
  2. 베이스 이미지를 `nginx:latest`로 명시적 태그와 함께 적용합니다.
  3. 명령형(Imperative) 단발성 명령이 아닌, 선언적(Declarative) 매니페스트 파일을 작성하여 배포 및 형상을 관리합니다.

---

### 2. Workflow

```text
  [ Engineer / Jump Host (thor) ]
                 │
                 │ (kubectl apply -f deployment.yaml)
                 ▼
  [ Kubernetes Control Plane (API Server) ]
                 │
                 ├── Creates: [ Deployment: nginx ]
                 │                  │
                 │                  ▼
                 └── Controls: [ ReplicaSet: nginx-xxxx ]
                                    │
                                    ▼
                             [ Pod: nginx-xxxx-yyyy ]
                                    └── [ Container: nginx ]
                                            └── Image: nginx:latest

```

---

### 3. 해결 과정 (Action)

#### 3-1. 선언적 Deployment 매니페스트 설계 (`deployment.yaml`)

명령형 구문(`kubectl create deployment`) 대신, 애플리케이션의 목표 상태(Desired State)를 보존하고 버전 형상 관리가 가능하도록 YAML 매니페스트 스펙을 작성합니다.

```bash
# deployment.yaml 작성 (선언적 스펙 정의)
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
EOF

```

- **설정 변경 의도**:
- `spec.selector.matchLabels`와 `spec.template.metadata.labels` 간의 라벨 바인딩(`app: nginx`)을 명확히 정의하여, Deployment가 추적 및 관리할 Pod 그룹의 범위를 선언했습니다.
- `spec.template.spec.containers[0].image` 필드에 `:latest` 태그를 명시하여 이미지 버전 스펙 요구사항을 완벽히 충족시켰습니다.

#### 3-2. Kubernetes 클러스터 상의 선언적 배포 적용

작성된 YAML 형상 파일을 Kubernetes API Server로 송출하여 원하는 아키텍처 상태를 반영합니다.

```bash
# 선언적 매니페스트 기반 배포 수행
kubectl apply -f deployment.yaml

```

#### 3-3. Deployment 및 상속 객체(ReplicaSet, Pod) 최종 검증

클러스터 내부에서 Deployment가 정상 구동(`READY 1/1`) 상태에 도달했는지 확인하고, 하위 세부 메타데이터의 이미지 스펙이 일치하는지 정밀 검증합니다.

```bash
# Deployment 및 생성된 Pod 구동 상태 조회
kubectl get deployment nginx
kubectl get pods

# Deployment 명세 내 적용된 이미지 태그 정밀 검증
kubectl describe deployment nginx | grep Image:

```

---

### 4. 핵심 개념 정리

- **Deployment (디플로이먼트)**: Kubernetes에서 무중단 배포(Rolling Update), 롤백(Rollback), 파드 개수 조절(Scaling) 및 상태 회복(Self-Healing)을 관리하기 위해 ReplicaSet을 상위에서 제어하는 상위 수준의 오케스트레이션 컨트롤러 객체입니다.

  > 💡 비유하자면, 개별 현장 작업자(Pod)들에게 일일이 지시하는 대신, 현장 총괄 현장소장(Deployment)을 고용하여 "항상 정상 가동되는 1명의 정회원 작업자 스태프(`replicas: 1`)를 현장에 유지시키고, 문제 발생 시 즉시 새 작업자로 교체하라"고 관리 책임을 위임하는 것과 같습니다.

- **ReplicaSet (레프리카셋)**: Deployment 하위에서 작동하며, 지정된 라벨 셀렉터(Label Selector) 조건에 들어맞는 Pod의 개수가 항상 설정된 수치(Replicas)만큼 유지되도록 클러스터 상태를 지속적으로 감시하고 조정하는 컨트롤러입니다.

  > 💡 비유하자면, 매장 내 교대 근무 인원이 부족해지면 즉시 알바생을 추가 채용(Pod 생성)하고, 인원이 넘치면 퇴근시키는 교대 근무 정원 관리 매니저와 같습니다.

- **Desired State & Control Loop (목표 상태와 컨트롤 루프)**: 사용자가 선언한 YAML 스펙(Desired State)과 클러스터의 실제 상태(Current State)를 지속적으로 비교(Reconciliation Loop)하여, 차이가 발생할 경우 실제 상태를 목표 상태에 맞추어 자동으로 수렴시키는 Kubernetes의 핵심 동작 메커니즘입니다.

  > 💡 비유하자면, 희망 온도(`24°C`)를 설정해 두면 에어컨 내장 센서가 실내 온도를 계속 측정하며 온도가 올라갈 때 알아서 냉각기를 작동시켜 목표 온도를 사수하는 자동 온도 조절 시스템과 같습니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **엔지니어의 회고(Retrospective)**: 단일 Pod 중심의 실습에서 한 단계 나아가, 실제 엔터프라이즈 운영 환경의 표준 배포 객체인 Deployment를 직접 다뤄보았습니다. 일회성 CLI 명령어 방식이 아닌 선언적 YAML 파일 중심의 배포 체계를 구성함으로써, 코드 형태의 인프라(GitOps / IaC) 관리가 선사하는 이점을 명확히 이해하게 되었습니다.

- **Pain Point**: 실무 환경에서 Deployment 스펙 작성 시 `matchLabels`와 Pod 템플릿의 `labels` 설정이 불일치하면, Deployment가 자신이 관리해야 할 Pod를 찾지 못해 무한 재creaton 루프에 빠지거나 배포가 중단되는 에러가 빈번히 발생합니다. 선언적 매니페스트 내부의 메타데이터 관계 구성을 치밀하게 검증하는 습관이 실무 장애 방지에 직결된다는 점을 느꼈습니다.

- **성장 포인트**: 단순 컨테이너 실행 수준을 넘어서서, 장애 발생 시 자동으로 시스템을 복구하고 무중단 업데이트를 가능케 하는 쿠버네티스 오케스트레이션의 핵심 계층 구조(Deployment ➡️ ReplicaSet ➡️ Pod)를 완전히 파악하고 제어할 수 있게 되었습니다.
