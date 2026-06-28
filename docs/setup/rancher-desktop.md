# Rancher Desktop 설치·설정

모든 실습은 **Rancher Desktop** 하나로 진행합니다. 컨테이너 런타임과 내장 쿠버네티스(k3s)를 함께 제공합니다.

## 1. 설치

1. [rancherdesktop.io](https://rancherdesktop.io/) 에서 OS(Windows / macOS / Linux)에 맞는 설치 파일을 받습니다.
2. 설치 후 실행하면 첫 기동 시 쿠버네티스 구성 요소를 내려받습니다. (수 분 소요, 네트워크 필요)

!!! info "시스템 권장 사양"
    메모리 8GB 이상(권장 16GB), 여유 디스크 10GB 이상. WSL2(Windows) 또는 가상화가 활성화되어 있어야 합니다.

## 2. 핵심 설정

Rancher Desktop을 실행하고 **Preferences(설정)** 에서 아래를 확인합니다.

=== "Kubernetes 활성화"

    - **Kubernetes** 탭에서 **Enable Kubernetes** 체크
    - 버전은 기본 안정 버전(예: v1.3x) 사용
    - 적용 후 우측 하단/상단 상태가 **녹색(Running)** 이 될 때까지 대기

=== "컨테이너 엔진 선택"

    **Container Engine** 탭에서 둘 중 하나를 고릅니다.

    - **containerd (권장 기본값)** → CLI 명령은 `nerdctl`
    - **dockerd (moby)** → CLI 명령은 `docker`

    이 가이드는 `nerdctl` 기준으로 작성했습니다. dockerd를 골랐다면 `nerdctl` → `docker` 로 바꿔 읽으세요.

## 3. 설치 확인

터미널을 열고 아래 명령으로 정상 동작을 확인합니다.

```bash
# 컨테이너 CLI 확인 (containerd 모드)
nerdctl version

# 쿠버네티스 컨텍스트 확인 (Rancher Desktop이 kubectl도 설치)
kubectl config get-contexts
kubectl config use-context rancher-desktop

# 노드가 Ready 인지 확인 (k3s 단일 노드)
kubectl get nodes -o wide
```

기대 결과: 노드 1개가 `Ready` 상태로 보이면 성공입니다.

```text
NAME                   STATUS   ROLES                  AGE   VERSION
lima-rancher-desktop   Ready    control-plane,master   1m    v1.3x.x+k3s1
```

!!! tip "kubectl이 안 잡힐 때"
    Rancher Desktop은 자체 `kubectl` 을 PATH에 추가합니다. 터미널을 새로 열거나,
    Rancher Desktop을 재시작한 뒤 다시 시도하세요. macOS/Linux는 `~/.rd/bin` 이 PATH에 있는지 확인합니다.

## 4. 내장 쿠버네티스의 특징 (실습에 중요)

Rancher Desktop의 쿠버네티스는 **k3s** 기반이라, 일반 클러스터와 약간 다른 편의 기능이 있습니다.

| 기능 | 내용 | 실습 영향 |
| --- | --- | --- |
| 기본 StorageClass | `local-path` (로컬 경로 프로비저너) | PVC가 자동으로 Bound (Lab 4) |
| LoadBalancer | 내장 ServiceLB로 동작, 외부 IP = `localhost` | `type: LoadBalancer` 가 바로 노출됨 (Lab 3) |
| Ingress | Traefik 기본 설치 | 추가 설치 없이 인그레스 사용 가능 |
| 로컬 이미지 | containerd `k8s.io` 네임스페이스 공유 | 빌드한 이미지를 레지스트리 없이 사용 (Lab 1·2) |

!!! warning "로컬에서 빌드한 이미지를 쿠버네티스에서 쓰려면"
    containerd 모드에서는 이미지를 **k8s가 보는 네임스페이스**로 빌드해야 합니다.
    ```bash
    nerdctl --namespace k8s.io build -t myapp:1.0 .
    ```
    그리고 매니페스트에서 `imagePullPolicy: Never` (또는 `IfNotPresent`) 로 두면
    레지스트리에 push하지 않고도 로컬 이미지를 그대로 사용할 수 있습니다.

## 다음 단계

환경이 준비되었다면 [Lab 1 · 컨테이너 & 이미지 기초](../labs/lab1-container-basics.md)로 이동하세요.
