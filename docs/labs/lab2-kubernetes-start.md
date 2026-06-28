# Lab 2 · 쿠버네티스 첫걸음 (Pod·Deployment)

!!! abstract "학습 목표"
    - Rancher Desktop 내장 쿠버네티스(k3s)에 접속해 클러스터를 확인한다
    - **Pod** 를 만들고 구조를 들여다본다 (describe·logs·exec)
    - Pod가 **일회용**임을, 그리고 **Deployment** 가 어떻게 자가 치유하는지 확인한다

    PPT 연계: *오케스트레이션 필요성 · 쿠버네티스 · 아키텍처 · Pod*

!!! note "준비"
    Lab 1을 마쳤고, `kubectl get nodes` 가 `Ready` 를 보여야 합니다.
    ```bash
    kubectl config use-context rancher-desktop
    kubectl get nodes
    ```

---

## 2-1. 클러스터 둘러보기 (아키텍처 확인)

PPT의 Control Plane / Worker 구조를 실제로 확인합니다.

```bash
# 클러스터 정보
kubectl cluster-info

# 컨트롤플레인 구성요소(시스템 네임스페이스의 Pod들)
kubectl get pods -n kube-system

# 노드 상세 (kubelet, 컨테이너 런타임 정보)
kubectl describe node | head -40
```

!!! info "방금 본 것"
    `kube-system` 네임스페이스에 코어 구성요소들이 Pod로 떠 있습니다.
    (k3s는 경량이라 단일 프로세스로 합쳐진 부분도 있지만 개념은 동일합니다)
    사용자의 모든 요청은 **API 서버**를 거칩니다.

---

## 2-2. Pod 만들기 (명령형 → 선언형)

### 명령형으로 빠르게

```bash
kubectl run web --image=nginx:1.27 --port=80
kubectl get pods -o wide
```

### 선언형(YAML)으로

운영 표준은 선언형입니다. 파일로 "원하는 상태"를 적어 `apply` 합니다.

```bash
cat > pod.yaml <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: web2
  labels:
    app: web
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
YAML

kubectl apply -f pod.yaml
kubectl get pods
```

!!! tip "YAML 4필드"
    어떤 오브젝트든 뼈대는 같습니다: `apiVersion` · `kind` · `metadata` · `spec`.

---

## 2-3. Pod 구조 들여다보기

```bash
# 상태·이벤트·노드 배치 등 상세
kubectl describe pod web

# 컨테이너 로그
kubectl logs web

# Pod 안으로 들어가 IP·프로세스 확인
kubectl exec -it web -- sh -c "hostname; ip addr 2>/dev/null; ps -ef | head"
```

!!! info "describe의 Events"
    `describe` 하단의 **Events** 로 *스케줄 → 이미지 풀 → 컨테이너 시작* 흐름을 읽을 수 있습니다.
    문제가 생기면 여기부터 봅니다.

---

## 2-4. Pod는 일회용이다

Lab 1에서 "Docker는 죽은 컨테이너를 안 살려준다"고 했습니다. **단독 Pod도 마찬가지**입니다.

```bash
kubectl delete pod web
kubectl get pods         # web 은 사라지고, 아무도 되살리지 않는다
```

!!! warning "그래서"
    Pod를 직접 만들면 죽었을 때 끝입니다. 그래서 실무에서는 Pod를 직접 만들지 않고,
    **상위 컨트롤러(Deployment)** 로 관리합니다.

---

## 2-5. Deployment: 자가 치유 (Docker 한계의 해답)

이제 Lab 1에서 겪은 "자동 복구 없음" 문제를 쿠버네티스가 어떻게 푸는지 봅니다.

```bash
cat > deploy.yaml <<'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
YAML

kubectl apply -f deploy.yaml
kubectl get deploy,rs,pods
```

### 자가 치유 확인

```bash
# Pod 하나를 강제로 삭제
kubectl delete pod -l app=web --field-selector status.phase=Running --grace-period=0 --force 2>/dev/null | head -1
# 또는 이름을 직접 골라 삭제: kubectl delete pod <pod이름>

# 즉시 새 Pod가 생겨 항상 3개를 유지하는지 관찰
kubectl get pods -w
```

(관찰이 끝나면 `Ctrl+C`)

!!! success "방금 본 것 = 오케스트레이션의 핵심"
    Deployment는 "**replicas: 3**" 이라는 *원하는 상태*를 끝까지 지킵니다.
    하나가 죽으면 **자동으로** 새 Pod로 채웁니다. Lab 1의 `nerdctl kill` 과 비교해 보세요.

### 스케일링

```bash
kubectl scale deploy web --replicas=5
kubectl get pods         # 즉시 5개로
kubectl scale deploy web --replicas=2
```

---

## 도전과제 🔥

??? question "도전 1 (★☆☆) · 소유 관계 추적"
    Pod가 누구 소유인지 확인하세요. Pod → ReplicaSet → Deployment 로 이어집니다.
    ```bash
    kubectl get pod <pod이름> -o jsonpath='{.metadata.ownerReferences[0].kind}{"\n"}'
    ```
    `Deployment` 를 지우면 왜 Pod가 전부 사라지는지 한 줄로 설명해 보세요.

??? question "도전 2 (★★☆) · 내가 만든 이미지로 배포"
    Lab 1-3에서 만든 `myapp:1.0` (k8s.io 네임스페이스로 빌드) 으로 Deployment를 띄워 보세요.
    로컬 이미지이므로 `imagePullPolicy: Never` 를 넣습니다.

    ??? note "힌트"
        ```yaml
        spec:
          containers:
            - name: myapp
              image: myapp:1.0
              imagePullPolicy: Never
        ```
        ```bash
        kubectl logs deploy/<이름>   # "hello from my image" 확인
        ```

??? question "도전 3 (★★☆) · 항상 N개 증명"
    `replicas: 3` 상태에서 Pod 2개를 빠르게 연달아 삭제하고, 다시 3개로 수렴하는 데 걸리는 시간을 관찰하세요.
    이것이 PPT의 **Reconcile Loop**(desired=current 유지)입니다.

---

## 정리 & 다음 단계

- 쿠버네티스는 **API 서버**를 중심으로 동작하고, 최소 단위는 **Pod**
- 단독 Pod는 일회용 → **Deployment** 가 replicas를 선언해 **자가 치유**
- Lab 1의 "자동 복구 없음" 한계가 여기서 해결됨

➡️ 다음: [Lab 3 · 워크로드와 서비스 노출](lab3-workloads-service.md) — 무중단 배포/롤백과 Service로 외부 노출까지.
