# Lab 3 · 워크로드와 서비스 노출

!!! abstract "학습 목표"
    - **롤링 업데이트**로 무중단 배포하고, 문제 시 **롤백**한다
    - **Service** 로 Pod에 고정 주소와 로드밸런싱을 부여한다
    - Rancher Desktop의 **LoadBalancer**(localhost)로 외부에서 접속한다
    - Pod → Deployment → Service 전체 스택을 완성한다

    PPT 연계: *워크로드 안정화 · 서비스 노출*

!!! note "준비"
    Lab 2의 `web` Deployment(replicas 2~3)가 떠 있는 상태에서 시작합니다.
    ```bash
    kubectl get deploy web || kubectl apply -f deploy.yaml
    ```

---

## 3-1. 무중단 롤링 업데이트

이미지 버전을 바꾸면 쿠버네티스가 Pod를 **조금씩 교체**합니다(항상 일부는 서비스 유지).

```bash
# 현재 이미지 확인
kubectl get deploy web -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

# 이미지 변경 → 롤링 업데이트 트리거
kubectl set image deploy/web nginx=nginx:1.27-alpine

# 진행 상황 관찰
kubectl rollout status deploy/web
kubectl get rs        # 새 ReplicaSet으로 점진 교체된 흔적
```

!!! info "방금 본 것"
    구 ReplicaSet은 줄고 새 ReplicaSet이 늘어납니다. 한 번에 갈아엎지 않으므로 **사용자는 끊김을 느끼지 않습니다.**

### 롤백

```bash
# 일부러 잘못된 이미지로 배포 (실패 유도)
kubectl set image deploy/web nginx=nginx:nope-not-exist
kubectl rollout status deploy/web --timeout=30s    # 멈춤/ImagePullBackOff

# 직전 정상 버전으로 한 줄 롤백
kubectl rollout undo deploy/web
kubectl rollout status deploy/web
kubectl get pods
```

!!! success "핵심"
    `rollout undo` 한 번으로 직전 버전 복구. 서버에 들어가 설정을 되돌릴 필요가 없습니다(불변 인프라).

---

## 3-2. Service: 고정 주소 + 로드밸런싱

Pod IP는 재생성 때마다 바뀝니다. **Service** 는 변하지 않는 주소와 로드밸런싱을 제공합니다.

### ClusterIP (내부 통신)

```bash
kubectl expose deploy web --port=80 --target-port=80 --name web
kubectl get svc,endpoints web
```

```bash
# 클러스터 내부에서 '이름'으로 접근 (임시 Pod)
kubectl run tmp --rm -it --image=busybox --restart=Never -- \
  sh -c "wget -qO- http://web | head -5"
```

!!! info "selector → Endpoints → Pod"
    Service는 `app=web` 라벨의 Pod IP를 자동으로 **Endpoints** 에 등록합니다.
    Pod가 바뀌어도 Service 이름(`web`)은 그대로입니다.

### LoadBalancer (외부 노출 — Rancher Desktop 편의)

Rancher Desktop은 내장 ServiceLB로 `type: LoadBalancer` 를 **localhost** 에 바로 노출합니다.

```bash
kubectl expose deploy web --port=80 --target-port=80 \
  --type=LoadBalancer --name web-lb
kubectl get svc web-lb        # EXTERNAL-IP 가 localhost/127.0.0.1 로 채워짐
```

```bash
# 외부(내 PC)에서 접속
curl http://localhost:80
# 포트 80이 막혀 있으면 다른 포트로:  --port=8088 로 expose 후 curl localhost:8088
```

!!! tip "포트가 충돌하면"
    호스트 80포트가 이미 쓰이면 `--port=8088` 처럼 다른 포트로 노출하세요.
    ```bash
    kubectl delete svc web-lb
    kubectl expose deploy web --port=8088 --target-port=80 --type=LoadBalancer --name web-lb
    curl http://localhost:8088
    ```

---

## 3-3. 통합 실습: Pod → Deployment → Service 한 파일로

운영처럼 하나의 매니페스트로 묶어 배포합니다.

```bash
cat > web-stack.yaml <<'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop-web
spec:
  replicas: 3
  selector:
    matchLabels: { app: shop-web }
  template:
    metadata:
      labels: { app: shop-web }
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet: { path: /, port: 80 }
            initialDelaySeconds: 2
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: shop-web
spec:
  type: LoadBalancer
  selector: { app: shop-web }
  ports:
    - port: 8090
      targetPort: 80
YAML

kubectl apply -f web-stack.yaml
kubectl get deploy,svc,pods -l app=shop-web
```

```bash
# 외부 접속
curl http://localhost:8090

# 무중단 업데이트 + 자가 치유 동시 확인
kubectl set image deploy/shop-web web=nginx:1.27-alpine
kubectl rollout status deploy/shop-web
kubectl delete pod -l app=shop-web | head -1     # 하나 죽여도
curl http://localhost:8090                        # 서비스는 계속 응답
```

!!! success "방금 완성한 것"
    **배포(Deployment) + 노출(Service) + 무중단 업데이트 + 자가 치유**가 한 흐름으로 동작합니다.
    `readinessProbe` 덕분에 준비 안 된 새 Pod로는 트래픽이 가지 않습니다.

---

## 도전과제 🔥

??? question "도전 1 (★★☆) · 분배 확인"
    `shop-web` 의 각 Pod에 서로 다른 응답을 넣고(예: 각 Pod의 hostname), 여러 번 호출했을 때
    여러 Pod로 분배되는지 확인하세요.

    ??? note "힌트"
        ```bash
        for i in $(seq 6); do kubectl exec deploy/shop-web -- hostname; done
        ```

??? question "도전 2 (★★☆) · ClusterIP vs LoadBalancer"
    같은 Deployment에 ClusterIP Service와 LoadBalancer Service를 둘 다 만들고,
    "내부 전용"과 "외부 노출"의 차이를 `kubectl get svc` 의 `EXTERNAL-IP` 로 설명하세요.

??? question "도전 3 (★★★) · 셀렉터로 트래픽 전환 (블루-그린 맛보기)"
    `version: blue` / `version: green` 두 Deployment를 띄우고, Service의 `selector` 를
    `blue` → `green` 으로 바꿔 트래픽이 순간 전환되는지 확인하세요.

    ??? note "힌트"
        Service는 selector에 매칭되는 Endpoints만 바라봅니다. `kubectl patch svc ...` 로 selector를 바꿔보세요.

---

## 정리 & 다음 단계

- **RollingUpdate** 로 무중단 배포, `rollout undo` 로 즉시 롤백
- **Service** = 고정 주소 + 로드밸런싱 + 디스커버리, Rancher Desktop은 `LoadBalancer` 를 localhost로 노출
- Pod → Deployment → Service 스택을 한 파일로 운영

➡️ 다음: [Lab 4 · 운영 기반과 자동화](lab4-operations.md) — 네임스페이스·설정/비밀 분리·스토리지·권한.
