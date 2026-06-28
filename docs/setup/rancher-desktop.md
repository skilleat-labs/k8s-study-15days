# Rancher Desktop 설치·설정

모든 실습은 **Rancher Desktop** 하나로 진행합니다. 컨테이너 런타임과 내장 쿠버네티스(k3s)를 함께 제공합니다.

!!! info "이 가이드의 기준 환경"
    - **컨테이너 엔진: Docker(dockerd / moby) 모드** → CLI 명령은 `docker`
    - **터미널: Windows PowerShell** 기준으로 작성했습니다. (macOS/Linux는 동일 명령이 대부분 그대로 동작하며, 다른 부분은 별도 탭으로 표기)

## 1. 설치

1. [rancherdesktop.io](https://rancherdesktop.io/) 에서 OS(Windows / macOS / Linux)에 맞는 설치 파일을 받습니다.
2. 설치 후 실행하면 첫 기동 시 쿠버네티스 구성 요소를 내려받습니다. (수 분 소요, 네트워크 필요)

!!! note "시스템 권장 사양"
    메모리 8GB 이상(권장 16GB), 여유 디스크 10GB 이상. Windows는 **WSL2** 가 활성화되어 있어야 합니다.

## 2. 핵심 설정

Rancher Desktop을 실행하고 **Preferences(설정)** 에서 아래를 확인합니다.

=== "컨테이너 엔진 (이 가이드 기준)"

    **Container Engine** 탭에서 **dockerd (moby)** 를 선택합니다.

    - CLI 명령은 `docker` 를 사용합니다. (이 가이드 전체가 `docker` 기준)
    - dockerd 모드에서는 `docker build` 로 만든 **로컬 이미지를 내장 쿠버네티스가 그대로 사용**할 수 있습니다.

=== "Kubernetes 활성화"

    - **Kubernetes** 탭에서 **Enable Kubernetes** 체크
    - 버전은 기본 안정 버전 사용
    - 적용 후 상태가 **녹색(Running)** 이 될 때까지 대기

!!! tip "containerd 모드를 쓰는 경우"
    엔진을 containerd로 골랐다면 CLI는 `nerdctl` 입니다. 이 가이드의 `docker` 를 `nerdctl` 로 바꿔 읽고,
    이미지 빌드 시 `nerdctl --namespace k8s.io build ...` 처럼 k8s 네임스페이스로 빌드해야 합니다.
    **헷갈리지 않으려면 dockerd 모드를 권장합니다.**

## 3. 설치 확인 (PowerShell)

PowerShell을 열고 아래로 정상 동작을 확인합니다.

```powershell
# Docker(dockerd) 동작 확인
docker version
docker run --rm hello-world

# 쿠버네티스 컨텍스트 확인 (Rancher Desktop이 kubectl도 설치)
kubectl config get-contexts
kubectl config use-context rancher-desktop

# 노드가 Ready 인지 확인 (k3s 단일 노드)
kubectl get nodes -o wide
```

기대 결과: `hello-world` 메시지가 보이고, 노드 1개가 `Ready` 상태면 성공입니다.

```text
NAME                   STATUS   ROLES                  AGE   VERSION
lima-rancher-desktop   Ready    control-plane,master   1m    v1.3x.x+k3s1
```

!!! tip "docker / kubectl 명령이 안 잡힐 때"
    Rancher Desktop이 PATH에 자동 등록합니다. **PowerShell 창을 새로 열거나** Rancher Desktop을 재시작한 뒤 다시 시도하세요.

## 4. 파일(YAML) 만드는 법 — 셸별 차이

실습에서는 `pod.yaml` 같은 파일을 자주 만듭니다. 본문에서는 **"아래 내용을 `pod.yaml` 로 저장하세요"** 로 안내하니
VS Code 등 편집기로 저장하면 됩니다. 터미널에서 바로 만들고 싶다면 아래 방식을 쓰세요.

=== "PowerShell (Windows)"

    PowerShell의 **here-string** `@' ... '@` 를 사용합니다. (닫는 `'@` 는 줄 맨 앞에 와야 합니다)

    ```powershell
    @'
    apiVersion: v1
    kind: Pod
    metadata:
      name: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
    '@ | Set-Content -Encoding utf8 pod.yaml
    ```

=== "bash (macOS/Linux)"

    ```bash
    cat > pod.yaml <<'EOF'
    apiVersion: v1
    kind: Pod
    metadata:
      name: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
    EOF
    ```

!!! warning "PowerShell 주의사항 (자주 막히는 부분)"
    - **`&&` 연결 금지**: Windows PowerShell 5.1은 `cmd1 && cmd2` 를 지원하지 않습니다. 명령을 **줄을 나눠** 실행하세요.
    - **`curl` 은 별칭**: PowerShell에서 `curl` 은 `Invoke-WebRequest` 별칭입니다. 진짜 curl은 **`curl.exe`** 로 부르거나, 그냥 **브라우저**로 확인하세요.
    - **`grep`/`head`/`seq` 없음**: 리눅스 전용입니다. 본문은 이를 피하도록 작성했고, 꼭 필요한 곳은 PowerShell 버전을 별도 탭으로 제공합니다.

## 5. 내장 쿠버네티스의 특징 (실습에 중요)

Rancher Desktop의 쿠버네티스는 **k3s** 기반이라, 일반 클러스터와 약간 다른 편의 기능이 있습니다.

| 기능 | 내용 | 실습 영향 |
| --- | --- | --- |
| 기본 StorageClass | `local-path` (로컬 경로 프로비저너) | PVC가 자동으로 Bound (Lab 4) |
| LoadBalancer | 내장 ServiceLB로 동작, 외부 IP = `localhost` | `type: LoadBalancer` 가 바로 노출됨 (Lab 3) |
| Ingress | Traefik 기본 설치 | 추가 설치 없이 인그레스 사용 가능 |
| 로컬 이미지 | **dockerd 모드면 `docker build` 이미지를 k8s가 공유** | 레지스트리 없이 로컬 이미지 사용 (Lab 1·2) |

!!! warning "로컬에서 빌드한 이미지를 쿠버네티스에서 쓰려면"
    dockerd 모드에서는 `docker build` 로 만든 이미지를 내장 쿠버네티스가 그대로 볼 수 있습니다.
    매니페스트에서 **`imagePullPolicy: Never`** (또는 `IfNotPresent`) 로 두면, 레지스트리에 push하지 않고 로컬 이미지를 사용합니다.
    ```powershell
    docker build -t myapp:1.0 .
    ```

## 다음 단계

환경이 준비되었다면 [Lab 1 · 컨테이너 & 이미지 기초](../labs/lab1-container-basics.md)로 이동하세요.
