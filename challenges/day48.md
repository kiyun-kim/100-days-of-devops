## 📅 Day 48: Kubernetes Pod 선언적 배포 및 메타데이터 구성

### 1. Task

- **요구사항**: Nautilus DevOps 팀의 Kubernetes 가상화 플랫폼 도입 계획에 따라, 애플리케이션의 최소 실행 단위인 Pod를 선언적 매니페스트(YAML) 파일 형태로 정의하고, 지정된 라벨 메타데이터와 컨테이너 식별명을 정확히 매핑하여 클러스터에 배포합니다.

- **목표**: Jump Host(`thor`) 환경의 `kubectl` CLI 유틸리티를 활용하여 클러스터 제어권을 확보하고 요구사항에 상응하는 Pod 객체를 생성 및 검증합니다.
  1. Pod 식별 명칭을 `pod-nginx`로 지정합니다.
  2. 도커 베이스 이미지를 `nginx:latest`로 적용합니다.
  3. Pod의 메타데이터 라벨 영역에 `app=nginx_app` 키-값(Key-Value) 쌍을 주입합니다.
  4. Pod 내부 단일 컨테이너의 이름을 `nginx-container`로 명시하여 생성합니다.

---

### 2. Workflow

```text
  [ Engineer / Jump Host (thor) ]
                 │
                 │ (kubectl apply -f pod.yaml)
                 ▼
  [ Kubernetes Control Plane (API Server) ]
                 │
                 │ (Pod Scheduling)
                 ▼
  [ Kubelet / Worker Node ]
                 │
                 └── [ Pod: pod-nginx ]
                         ├── Labels: app=nginx_app
                         └── [ Container: nginx-container ]
                                 └── Image: nginx:latest

```

---

### 3. 해결 과정 (Action)

#### 3-1. Kubernetes 선언적 매니페스트 작성 (`pod.yaml`)

명령형 방식(`kubectl run`)에서 발생할 수 있는 컨테이너 이름 자동 지정 오류를 방지하고, 정확한 명세를 적용하기 위해 YAML 표준 파일 형태로 매니페스트 스펙을 정의합니다.

```bash
# pod.yaml 작성 (선언적 스펙 정의)
cat <<EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nginx
  labels:
    app: nginx_app
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
EOF

```

- **설정 변경 의도**:
  - `metadata.labels` 항목에 `app: nginx_app`을 명시하여 향후 Service 객체나 ReplicaSet에서 라벨 셀렉터(Label Selector)로 해당 파드를 오케스트레이션할 수 있도록 제어 기반을 마련했습니다.
  - `spec.containers[0].name` 필드를 `nginx-container`로 명확하게 고정하여 파드명과 컨테이너명의 명시적 분리를 달성했습니다.

#### 3-2. Kubernetes 클러스터 상의 선언적 배포 적용

작성된 YAML 형상 파일을 Kubernetes API Server로 전송하여 파드를 생성합니다.

```bash
# 작성된 매니페스트 파일 기반의 선언적 파드 배포
kubectl apply -f pod.yaml

```

#### 3-3. 파드 상태 및 내장 메타데이터 정밀 검증

클러스터 내부에서 배포된 파드의 구동 상태(Running) 및 컨테이너 명칭 속성이 명세서와 완벽히 일치하는지 최종 검토합니다.

```bash
# 파드의 구동 상태 및 인스턴스 할당 확인
kubectl get pod pod-nginx

# Containers 영역 하위의 식별명이 nginx-container로 정상 매핑되었는지 검증
kubectl describe pod pod-nginx | grep -A 5 Containers:

```

---

### 4. 핵심 개념 정리

- **Pod (파드)**: Kubernetes에서 생성하고 관리할 수 있는 가장 작은 배포 가능 컴퓨팅 단위로, 하나 이상의 컨테이너(Docker 등) 그룹, 공유 스토리지(볼륨), 네트워크 IP 자원을 하나의 논리적 단위로 래핑(Wrapping)한 객체입니다.

  > 💡 비유하자면, 콩껍질(Pod) 안에 여러 개의 콩알(컨테이너)이 사이좋게 들어있어서, 콩껍질 하나만 들고 이동하면 그 안의 콩알들이 항상 같은 영양분(네트워크 IP 및 스토리지)을 공유하며 함께 움직이는 최소 단위 포장 상자와 같습니다.

- **labels & selectors**: Kubernetes의 모든 리소스 객체(Pod, Service, Deployment 등)에 임의의 키-값(Key-Value) 쌍으로 부여하는 메타데이터 식별 체계입니다. selector를 통해 특정 라벨을 가진 객체들만 유연하게 그룹핑하고 추적할 수 있습니다.

  > 💡 비유하자면, 물류 창고에 쌓인 수천 개의 택배 상자(Pod) 박스 겉면에 '영업부('app=nginx_app')'라는 색깔 스티커(Label)를 붙여놓고, 나중에 배송 기사(Service)가 와서 "영업부 스티커 붙은 상자들만 골라서 차에 실어라"라고 손쉽게 선별(Selector)하는 태그 시스템과 같습니다.

- **Declarative Management (선언적 관리 vs 명령형 관리)**: "어떻게 생성하라"는 일회성 절차(Imperative)를 일일이 명령하는 대신, "최종적으로 시스템이 이러한 상태(Desired State)여야 한다"라는 결과를 YAML 코드로 선언하여 API Server에 등록하고, 쿠버네티스의 컨트롤 러프(Control Loop)가 실시간으로 이를 추종하도록 만드는 현대적 인프라 관리 패러다임입니다.

  > 💡 비유하자면, 택시 기사에게 "100m 앞 좌회전 후 50m 가서 우회전해 주세요"라고 계속 지시하는 명령형 방식 대신, 자율주행 차에 "목적지 주소(YAML)"를 입력하면 차량 스스로 현재 위치와 목적지를 비교하며 최적의 핸들링으로 목표 상태에 도달해 유지하는 스마트 시스템과 같습니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **회고(Retrospective)**: Docker Compose 중심의 단일 호스트 가상화 단계를 넘어, 모던 컨테이너 오케스트레이션의 표준인 Kubernetes의 파드(Pod) 객체를 처음으로 다뤄보며 클러스터 아키텍처의 유연함과 정교함을 체감했습니다. CLI 명령형 생성 방식의 한계를 파악하고 선언적 YAML 매니페스트로 구조를 명확히 고정하는 과정의 중요성을 실감했습니다.

- **Pain Point**: 실무 쿠버네티스 환경에서 파드를 띄울 때 매니페스트의 메타데이터 라벨이나 컨테이너 명칭을 대충 작성하거나 오타를 내면, 차후 Deployment나 Service 연동 시 라벨 셀렉터 불일치로 인해 트래픽이 파드로 도달하지 못하는 '침묵의 배포 장애'가 발생합니다. 명세서를 작성할 때 스펙 필드 하나하나의 명칭 정확도를 높이는 습관이 현장 엔지니어에게 얼마나 치명적인 안전장치가 되는지 직관했습니다.

- **성장 포인트**: 단순히 단일 컨테이너를 구동하는 것을 넘어, 쿠버네티스 API가 요구하는 오브젝트 스펙 구조를 이해하고 YAML 매니페스트를 다룰 수 있는 기초 'Kubernetes Practitioner'로의 중요한 첫 걸음을 내딛게 되었습니다.
