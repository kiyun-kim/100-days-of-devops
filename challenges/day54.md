## 📅 Day 54: Kubernetes emptyDir 볼륨을 통한 다중 컨테이너 데이터 공유 환경 구축

### 1. Task

- **요구사항**: 동일한 파드(Pod) 내에서 실행되는 다중 컨테이너 간 임시 데이터를 원활하게 공유하기 위해 `emptyDir` 볼륨을 구성하고, 컨테이너 간 데이터 실시간 연동 및 볼륨 마운트 정상 동작 여부를 검증합니다.

- **목표**:
  1. `volume-share-nautilus` 이름의 Pod 생성
  2. 첫 번째 컨테이너(`volume-container-nautilus-1`) 설정: `ubuntu:latest` 이미지 사용, `sleep` 명령 실행, `volume-share` 볼륨을 `/tmp/news` 경로에 마운트
  3. 두 번째 컨테이너(`volume-container-nautilus-2`) 설정: `ubuntu:latest` 이미지 사용, `sleep` 명령 실행, `volume-share` 볼륨을 `/tmp/apps` 경로에 마운트
  4. `emptyDir` 유형의 볼륨 `volume-share` 정의 및 적용
  5. `volume-container-nautilus-1` 컨테이너의 마운트 경로(`/tmp/news`)에 `news.txt` 파일 생성 및 내용(`Welcome to xFusionCorp Industries`) 작성
  6. `volume-container-nautilus-2` 컨테이너의 마운트 경로(`/tmp/apps`)에서 동일한 `news.txt` 파일 및 내용 공유 확인

---

### 2. Workflow

```text
+---------------------------------------------------------------------------------+
| Pod: volume-share-nautilus                                                      |
|                                                                                 |
|  +-----------------------------------+     +---------------------------------+  |
|  | Container 1                       |     | Container 2                     |  |
|  | volume-container-nautilus-1       |     | volume-container-nautilus-2     |  |
|  | Image: ubuntu:latest              |     | Image: ubuntu:latest            |  |
|  | Mount: /tmp/news                  |     | Mount: /tmp/apps                |  |
|  +-----------------+-----------------+     +----------------+----------------+  |
|                    |                                        |                   |
|                    +------------------+  +------------------+                   |
|                                       |  |                                      |
|                                       v  v                                      |
|                         +------------------------------+                        |
|                         | Volume: volume-share         |                        |
|                         | Type: emptyDir               |                        |
|                         +------------------------------+                        |
+---------------------------------------------------------------------------------+

```

---

### 3. 해결 과정 (Action)

#### 3-1. Pod 정의 매니페스트 작성 (`pod.yaml`)

공유 볼륨 `emptyDir` 구조와 각 컨테이너별 서로 다른 마운트 경로를 명시한 Pod YAML 파일을 생성합니다.

```bash
# pod.yaml 매니페스트 작성
cat <<EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-share-nautilus
spec:
  volumes:
  - name: volume-share
    emptyDir: {}
  containers:
  - name: volume-container-nautilus-1
    image: ubuntu:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - name: volume-share
      mountPath: /tmp/news
  - name: volume-container-nautilus-2
    image: ubuntu:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - name: volume-share
      mountPath: /tmp/apps
EOF

```

#### 3-2. Pod 생성 및 상태 확인

작성한 매니페스트 파일을 적용하여 Pod를 생성하고 정상 실행 상태(`Running`)를 확인합니다.

```bash
# Pod 배포
thor@jump-host ~$ kubectl apply -f pod.yaml
pod/volume-share-nautilus created

# Pod 상태 및 컨테이너 정상 생성 확인 (2/2 READY)
thor@jump-host ~$ kubectl get pod volume-share-nautilus
NAME                   READY   STATUS    RESTARTS   AGE
volume-share-nautilus   2/2     Running   0          11s

```

#### 3-3. 데이터 생성 및 컨테이너 간 볼륨 공유 검증

첫 번째 컨테이너에서 테스트 파일을 생성하고, 두 번째 컨테이너에서 동일한 볼륨 경로를 통해 파일이 실시간으로 공유되는지 검증합니다.

```bash
# 첫 번째 컨테이너(/tmp/news)에서 news.txt 파일 생성
thor@jump-host ~$ kubectl exec -it volume-share-nautilus -c volume-container-nautilus-1 -- bash -c "echo 'Welcome to xFusionCorp Industries' > /tmp/news/news.txt"

# 두 번째 컨테이너(/tmp/apps)에서 news.txt 파일 조회 및 내용 검증
thor@jump-host ~$ kubectl exec -it volume-share-nautilus -c volume-container-nautilus-2 -- cat /tmp/apps/news.txt
Welcome to xFusionCorp Industries

```

---

### 4. 핵심 개념 정리

#### emptyDir 볼륨 (emptyDir Volume)

`emptyDir`은 Pod가 노드에 할당될 때 처음 생성되는 임시 볼륨으로, Pod 내의 모든 컨테이너가 해당 디렉터리를 읽고 쓸 수 있도록 공유 메모리/디스크 공간을 제공하는 스토리지 유형입니다. Pod가 노드에서 삭제되면 `emptyDir` 내의 모든 데이터도 영구적으로 삭제됩니다.

> 💡 비유하자면, 하나의 아파트(Pod) 안에 들어선 여러 개의 방(Container)들이 공용으로 사용하는 '중앙 거실 테이블'과 같습니다. 첫 번째 방에 있는 사람이 거실 테이블 위에 메모를 적어두면, 두 번째 방에 있는 사람도 거실로 나와 그 메모를 즉시 확인하고 수정할 수 있는 구조입니다. 다만 아파트 자체가 재건축을 위해 철거(Pod 삭제)되면 거실 테이블 위의 메모도 함께 사라집니다.

#### 사이드카 패턴 (Sidecar Pattern)

단일 Pod 내부에서 메인 애플리케이션 컨테이너 외에 보조 역할을 수행하는 컨테이너를 함께 배포하는 디자인 패턴입니다. 예를 들어 메인 컨테이너가 로그나 데이터를 생성하면, 사이드카 컨테이너가 `emptyDir`을 통해 해당 데이터를 공유받아 외부로 수집·전송하거나 수집된 데이터를 가공하는 역할을 담당합니다.

> 💡 비유하자면, 인기 가수의 무대(Main Container) 바로 옆에서 실시간으로 음향을 조정하거나 통역을 제공하는 '전담 스태프(Sidecar Container)'와 같습니다. 두 사람이 같은 무대 뒤 전용 통로(emptyDir)를 통해 필요한 소품을 빠르게 주고받으며 공연을 완성하는 것과 동일합니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **회고(Retrospective)**: 단일 Pod 내 다중 컨테이너 구조에서 컨테이너 간 독립성을 유지하면서도 데이터를 효율적으로 주고받기 위해 `emptyDir` 볼륨이 얼마나 유용하게 쓰이는지 직접 체감했습니다. 각각의 컨테이너가 서로 다른 로컬 마운트 경로(`/tmp/news`, `/tmp/apps`)를 바라보고 있더라도, 하위 스토리지 레이어가 동일한 `emptyDir` 볼륨에 바인딩되어 있다면 완전한 데이터 일관성이 보장된다는 점을 실습을 배울 수 있었습니다.

- **Pain Point**: 만약 이 개념을 모른 채 마이크로서비스 간 임시 데이터를 주고받기 위해 외부 영구 스토리지(EBS, NFS 등)를 무분별하게 붙이거나, HTTP REST API 기반의 네트워크 통신을 거치도록 설계했다면 불필요한 네트워크 지연(Latency)과 오버헤드, 비용 상승을 초래했을 것입니다. Pod 라이프사이클과 궤를 같이 하는 단순 임시 데이터 처리는 `emptyDir`을 활용하는 것이 인프라 복잡도를 줄이는 길임을 깨달았습니다.

- **성장 포인트**: 로깅 agent 배포나 로컬 캐시 공유와 같은 사이드카 패턴 아키텍처를 설계할 때 컨테이너 간 볼륨 마운트를 어떻게 격리하고 연결해야 하는지 기준이 생겼습니다. 앞으로 K8s 기반 인프라를 구축할 때 스토리지의 생명주기(Ephemeral vs Persistent)에 맞춰 적절한 볼륨 유형을 고르는 안목을 한 단계 더 높일 수 있게 되었습니다.
