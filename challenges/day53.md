## 📅 Day 53: Kubernetes Pod Multi-Container VolumeMounts 경로 불일치 및 Nginx FastCGI 트러블슈팅

### 1. Task

- **요구사항**: `nginx-phpfpm` Pod 내부의 `nginx-container`와 `php-fpm-container` 간 공유 볼륨(`shared-files`) 마운트 경로 불일치 문제를 해결하고, ConfigMap의 Nginx root 설정과 FastCGI 스크립트 경로를 정상화하여 웹 요청 시 PHP 애플리케이션이 올바르게 처리되도록 인프라를 복구합니다.

- **목표**:
  1. `nginx-config` ConfigMap의 웹 루트(`root`) 경로를 `/usr/share/nginx/html`로, FastCGI 스크립트 경로를 `/var/www/html/$fastcgi_script_name`으로 정정합니다.
  2. `nginx-phpfpm` Pod 매니페스트 추출 후, `nginx-container`의 볼륨 마운트 경로(`mountPath`)를 `/usr/share/nginx/html`로 교체하여 강제 재배포합니다.
  3. 로컬 호스트의 `/home/thor/index.php` 파일을 `nginx-container` 공유 디렉터리로 복사하여 웹 서버 200 OK 응답을 검증합니다.

---

### 2. Workflow

```text
[HTTP Client / Browser]
       │ (Port 80)
       ▼
┌────────────────────────────────────────────────────────────────────────┐
│ Pod: nginx-phpfpm                                                      │
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │ nginx-container                                                │   │
│   │  - ConfigMap: root /usr/share/nginx/html                       │   │
│   │  - fastcgi_param SCRIPT_FILENAME /var/www/html/...             │   │
│   └───────────────────────────────┬────────────────────────────────┘   │
│                                   │                                    │
│                 ┌─────────────────┴─────────────────┐                  │
│                 │ emptyDir Volume: shared-files     │                  │
│                 └─────────────────┬─────────────────┘                  │
│                                   │                                    │
│   ┌───────────────────────────────┴────────────────────────────────┐   │
│   │ php-fpm-container                                              │   │
│   │  - FastCGI Pass: 127.0.0.1:9000                                │   │
│   │  - mountPath: /var/www/html                                    │   │
│   └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘

```

---

### 3. 해결 과정 (Action)

#### 3-1. ConfigMap 웹 루트 및 FastCGI 경로 수정

`nginx-config` ConfigMap을 편집하여 Nginx가 바라보는 로컬 루트 경로와 PHP-FPM 프로세스로 전달할 프로세스 해석 경로를 각각의 마운트 디렉터리에 맞게 수정합니다.

```bash
# ConfigMap 편집
kubectl edit configmap nginx-config

```

```nginx
events {
}
http {
  server {
    listen 80 default_server;
    listen [::]:80 default_server;

    # Nginx 컨테이너의 shared-files 마운트 경로 지정
    root /usr/share/nginx/html;
    index index.html index.htm index.php;

    server_name _;

    location / {
      try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
      include fastcgi_params;
      fastcgi_param REQUEST_METHOD $request_method;
      # PHP-FPM 컨테이너가 바라보는 마운트 경로(/var/www/html)로 SCRIPT_FILENAME 전달
      fastcgi_param SCRIPT_FILENAME /var/www/html/$fastcgi_script_name;
      fastcgi_pass 127.0.0.1:9000;
    }
  }
}

```

#### 3-2. Pod 매니페스트 추출 및 VolumeMount 경로 수정

쿠버네티스의 Pod `volumeMounts` 스펙은 생성 후 직접 수정(In-place update)이 불가능한 불변(Immutable) 객체이므로, 기존 YAML을 추출하여 수정합니다.

```bash
# 기존 Pod 매니페스트 추출
kubectl get pod nginx-phpfpm -o yaml > pod.yaml

# pod.yaml 파일 편집
vi pod.yaml

```

```yaml
spec:
  containers:
    - image: php:7.2-fpm-alpine
      name: php-fpm-container
      volumeMounts:
        - mountPath: /var/www/html
          name: shared-files
    - image: nginx:latest
      name: nginx-container
      volumeMounts:
        # nginx-container의 마운트 경로를 /usr/share/nginx/html로 변경
        - mountPath: /usr/share/nginx/html
          name: shared-files
        - mountPath: /etc/nginx/nginx.conf
          name: nginx-config-volume
          subPath: nginx.conf
```

#### 3-3. Pod 강제 재배포 (Replace)

수정된 매니페스트를 적용하기 위해 기존 Pod를 강제 삭제 및 재생성합니다.

```bash
# 수정된 YAML 기반 강제 교체
kubectl replace --force -f pod.yaml

```

#### 3-4. 소스 파일 복사 및 검증

Pod가 Running 상태로 전환된 후 웹 소스 파일을 공유 디렉터리로 복사하고 응답을 확인합니다.

```bash
# index.php 파일 복사
kubectl cp /home/thor/index.php nginx-phpfpm:/usr/share/nginx/html/index.php -c nginx-container

# 컨테이너 내 로컬 HTTP 요청 검증
kubectl exec -it nginx-phpfpm -c nginx-container -- curl http://localhost/

```

---

### 4. 핵심 개념 정리

#### Multi-Container Pod & emptyDir Volume

하나의 Pod 안에 2개 이상의 컨테이너가 존재하는 구조를 다중 컨테이너(Multi-Container) Pod라고 합니다. 이들이 로컬 파일 시스템을 통해 데이터를 주고받기 위해 쿠버네티스의 `emptyDir` 볼륨을 사용합니다. `emptyDir`은 Pod가 노드에 할당될 때 임시로 생성되어 Pod가 생존하는 동안 유지되는 비지속성 볼륨입니다.

> 💡 비유하자면, 한 집(Pod)에 사는 두 명의 룸메이트(Nginx, PHP-FPM)가 거실에 놓인 '공용 서류함(`emptyDir`)'을 함께 사용하는 것과 같습니다. Nginx는 서류함 윗칸(`/usr/share/nginx/html`)을 열고, PHP-FPM은 서류함 아래칸(`/var/www/html`)을 열지만, 실제로는 동일한 서류함 내부 공간을 함께 공유하고 있습니다.

#### Pod Spec Immutability (불변성)

쿠버네티스의 Pod 명세 중 이미지, 셀렉터, 볼륨 마운트 등 대부분의 인프라 스펙은 한 번 생성되면 가동 중에 직접 수정할 수 없습니다.

> 💡 비유하자면, 완성되어 출고된 자동차의 엔진이나 차체를 운전 중에 고칠 수 없는 것과 같습니다. 설정을 바꾸려면 설계도(YAML)를 수정하여 새 차로 완전히 교체(`replace --force`)해야 합니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **회고(Retrospective)**:
  컨테이너 간 업무 분담 구조에서 단순 파일 존재 여부보다 각 컨테이너 프로세스가 바라보는 디렉터리 시점(Context)이 얼마나 중요한지 체감할 수 있었습니다. 로컬 디렉터리에 분명히 파일이 존재하더라도 Nginx와 PHP-FPM이 마운트한 절대 경로가 다르면 서비스는 정상 작동하지 않는다는 점을 배웠습니다.

- **Pain Point**:
  Pod 내부 프로세스나 Nginx 설정만 부분적으로 변경하려고 시도할 경우, 포트 바인딩 소켓 문제나 프로세스 라이프사이클 한계로 인해 더 큰 장애를 초래할 수 있습니다. 불변 객체인 Pod의 구조적 문제를 다룰 때는 `kubectl edit`으로 미봉책을 찾기보다 YAML 추출 후 Clean Replace를 진행하는 것이 명확한 해결책임을 경험했습니다.

- **성장 포인트**:
  앞으로 다중 컨테이너 아키텍처를 설계하거나 트러블슈팅할 때, 각 컨테이너의 볼륨 마운트 지점(`mountPath`)과 애플리케이션 프레임워크의 내부 경로 인지 방식을 제일 먼저 교차 검증하는 표준 체크리스트를 갖추게 되었습니다.
