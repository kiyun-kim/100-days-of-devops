## 📅 Day 55: 쿠버네티스 네이티브 사이드카(Native Sidecar) 패턴을 활용한 Nginx 로그 공유 파드 구축

### 1. Task

- **요구사항**: 단일 Pod 내에서 실행되는 Nginx 웹 서버의 액세스 및 에러 로그를 수집하기 위해 `emptyDir` 공유 볼륨을 구성하고, 쿠버네티스의 네이티브 사이드카(Native Sidecar) 패턴을 적용하여 메인 애플리케이션과 로그 수집 프로세스를 효과적으로 분리 및 연동합니다.

- **목표**:
  1. `webserver` 이름의 Pod 생성 및 `emptyDir` 유형의 `shared-logs` 볼륨 정의
  2. 메인 컨테이너(`nginx-container`): `nginx:latest` 이미지 사용, `/var/log/nginx` 경로에 볼륨 마운트
  3. 사이드카 컨테이너(`sidecar-container`): `ubuntu:latest` 이미지 사용, `initContainers` 영역에 배치 및 `restartPolicy: Always` 적용
  4. 사이드카 실행 명령: `["sh", "-c", "while true; do cat /var/log/nginx/access.log /var/log/nginx/error.log; sleep 30; done"]` 지정하여 지속적인 로그 수집
  5. 두 컨테이너 모두 정상적으로 `Running` 상태를 유지하고 볼륨을 통한 데이터 공유 검증

---

### 2. Workflow

```text
+---------------------------------------------------------------------------------------+
| Pod: webserver                                                                        |
|                                                                                       |
|  [ 1. Init Phase / Background ]           [ 2. Main Phase ]                           |
|  +-----------------------------------+    +----------------------------------------+  |
|  | InitContainer (Sidecar)           |    | Regular Container (Main)               |  |
|  | sidecar-container                 |    | nginx-container                        |  |
|  | Image: ubuntu:latest              |    | Image: nginx:latest                    |  |
|  | restartPolicy: Always             |    |                                        |  |
|  | Mount: /var/log/nginx             |    | Mount: /var/log/nginx                  |  |
|  +-----------------+-----------------+    +-------------------+--------------------+  |
|                    |                                          |                       |
|   (Reads Logs)     +------------------+    +------------------+ (Writes Logs)         |
|                                       |    |                                          |
|                                       v    v                                          |
|                         +------------------------------+                              |
|                         | Volume: shared-logs          |                              |
|                         | Type: emptyDir               |                              |
|                         +------------------------------+                              |
+---------------------------------------------------------------------------------------+

```

---

### 3. 해결 과정 (Action)

#### 3-1. 네이티브 사이드카 패턴이 적용된 매니페스트 작성 (`pod.yaml`)

메인 Nginx 컨테이너와 로그 수집용 사이드카 컨테이너가 동일한 생명주기 하에서 안전하게 작동하도록 `initContainers` 배열 내에 `restartPolicy: Always`를 부여하여 작성합니다.

```bash
# pod.yaml 매니페스트 작성
cat <<EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: webserver
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}
  initContainers:
  - name: sidecar-container
    image: ubuntu:latest
    restartPolicy: Always
    command: ["sh", "-c", "while true; do cat /var/log/nginx/access.log /var/log/nginx/error.log; sleep 30; done"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  containers:
  - name: nginx-container
    image: nginx:latest
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
EOF

```

#### 3-2. Pod 배포 및 구동 상태 확인

매니페스트를 클러스터에 배포하고, 초기화 컨테이너가 백그라운드에서 실행을 유지하며 메인 컨테이너와 함께 정상적으로 `2/2 Running` 상태에 도달하는지 확인합니다.

```bash
# Pod 생성
kubectl apply -f pod.yaml

# 상태 확인 (Init 단계 후 Running 전환 확인)
kubectl get pod webserver

```

#### 3-3. 볼륨 마운트 및 로그 공유 검증

사이드카 컨테이너가 메인 Nginx 컨테이너가 기록하는 로그 파일을 공유 볼륨을 통해 정상적으로 수집하여 출력하는지 검증합니다.

```bash
# 사이드카 컨테이너 로그 수집 확인
kubectl logs webserver -c sidecar-container

# (선택) 메인 컨테이너 내부 파일 시스템 마운트 확인
kubectl exec webserver -c nginx-container -- df -h /var/log/nginx

```

---

### 4. 핵심 개념 정리

#### 네이티브 사이드카 (Native Sidecar - K8s v1.28+)

쿠버네티스의 `initContainers` 속성을 활용하되 `restartPolicy: Always`를 명시하여, 파드 시작 시 메인 컨테이너보다 먼저 실행을 시작하고 파드가 종료될 때까지 백그라운드에서 지속 실행되도록 보장하는 컨테이너 라이프사이클 관리 기법입니다.

> "💡 비유하자면, 식당(Pod) 영업 전 미리 출근하여 영업이 끝날 때까지 주방장(메인 컨테이너) 옆에 상주하며 보조 업무를 수행하는 '전담 보조 셰프'와 같습니다. 단순 준비 작업만 하고 퇴근하는 일반적인 초기화(Init) 인력과는 달리, 영업 내내 주방장과 보조를 맞추며 독립적인 공간에서 지속적으로 협업합니다."

---

### 5. 무엇을 배웠는가 (Takeaway)

- **회고(Retrospective)**: 파드 내에 여러 컨테이너를 배치할 때 단순히 `containers` 배열에 나열하는 것을 넘어, 실행 순서와 생명 주기를 엄격히 제어해야 할 때 네이티브 사이드카 패턴이 얼마나 강력하고 명확한 기준을 제시하는지 경험했습니다.

- **Pain Point**: 기존 방식처럼 일반 멀티 컨테이너 아키텍처로 구성할 경우, 메인 애플리케이션과 로그 수집기 간의 구동 순서가 보장되지 않아 시작 시점의 레이스 컨디션(Race Condition)을 겪거나 종료 시 로그가 유실되는 운영상의 리스크를 떠안아야 했을 것입니다. K8s 공식 명세에 맞춘 명확한 분리 선언이 이러한 사이드 이펙트를 완벽히 차단한다는 점을 확인했습니다.

- **성장 포인트**: 애플리케이션 코드의 수정 없이 로깅, 모니터링, 프록시 등의 공통 기능을 외부에서 안전하게 주입하고 관리하는 마이크로서비스 설계 패턴의 기초를 다졌습니다.
