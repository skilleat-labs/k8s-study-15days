# Lab 4 · 운영 기반과 자동화

!!! abstract "학습 목표"
    - **Namespace** 로 운영 구역을 나누고 자원 한도를 건다
    - **ConfigMap / Secret** 으로 설정·비밀을 이미지에서 분리한다
    - **PV/PVC** 로 데이터를 영속화한다 (Pod가 죽어도 데이터 보존)
    - **ServiceAccount / RBAC** 로 권한을 최소화한다

    PPT 연계: *운영 기반 · 자동화 · 보안*

!!! note "준비 / 기준 환경"
    Lab 2~3을 마친 상태. Windows PowerShell 기준이며, YAML은 편집기로 저장해서 사용합니다.
    Rancher Desktop은 기본 StorageClass `local-path` 를 제공하므로 PVC 실습이 추가 설정 없이 됩니다.

---

## 4-1. Namespace로 구역 나누기

```powershell
kubectl create namespace dev
kubectl get ns

# dev 안에 배포
kubectl create deployment web --image=nginx:1.27 -n dev
kubectl get pods -n dev

# 기본 네임스페이스를 dev로 전환(이후 -n 생략 가능)
kubectl config set-context --current --namespace=dev
kubectl get pods
```

### 자원 총량 제한 (ResourceQuota)

아래를 **`quota.yaml`** 로 저장합니다.

```yaml title="quota.yaml"
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 2Gi
    pods: "10"
```

```powershell
kubectl apply -f quota.yaml
kubectl describe quota dev-quota -n dev
```

!!! info "구역 + 한도"
    Namespace(구역) + ResourceQuota(한도)로 "한 팀이 클러스터를 독식"하는 사고를 예방합니다.

---

## 4-2. ConfigMap: 설정 분리

같은 이미지를 **환경별로 다르게** 동작시킵니다. (12-factor)

```powershell
kubectl create configmap app-config --from-literal=APP_MODE=production --from-literal=PAGE_SIZE=20 -n dev
```

아래를 **`cm-pod.yaml`** 로 저장합니다.

```yaml title="cm-pod.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: cm-demo
  namespace: dev
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","echo MODE=$APP_MODE SIZE=$PAGE_SIZE; sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-config
```

```powershell
kubectl apply -f cm-pod.yaml
kubectl logs cm-demo -n dev      # MODE=production SIZE=20
```

!!! tip "env vs 볼륨"
    환경변수(env)로 주입한 값은 바꿔도 **재시작해야** 반영됩니다.
    설정 '파일'은 볼륨으로 마운트하면 자동 갱신됩니다.

---

## 4-3. Secret: 비밀 분리

```powershell
kubectl create secret generic db-secret --from-literal=DB_PASS='S3cr3t!' -n dev
```

저장된 값은 base64로 인코딩되어 있습니다(암호화가 아님). 디코드해서 확인해 봅니다.

=== "PowerShell (Windows)"

    ```powershell
    $b64 = kubectl get secret db-secret -n dev -o jsonpath="{.data.DB_PASS}"
    [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($b64))
    ```

=== "bash (macOS/Linux)"

    ```bash
    kubectl get secret db-secret -n dev -o jsonpath='{.data.DB_PASS}' | base64 -d; echo
    ```

아래를 **`secret-pod.yaml`** 로 저장합니다.

```yaml title="secret-pod.yaml"
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
  namespace: dev
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","echo pass=$DB_PASS; sleep 3600"]
      env:
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASS
```

```powershell
kubectl apply -f secret-pod.yaml
kubectl logs secret-demo -n dev
```

!!! warning "base64 ≠ 암호화"
    Secret 값은 누구나 디코드할 수 있습니다. 진짜 보호는 **RBAC**(아래)·etcd 암호화·외부 비밀관리로 합니다.

---

## 4-4. PV/PVC: 데이터 영속화

컨테이너 저장은 휘발성입니다. **PVC** 로 Pod 수명과 분리된 저장소를 붙입니다.
Rancher Desktop은 기본 StorageClass `local-path` 로 PV를 자동 생성합니다.

```powershell
kubectl get storageclass         # local-path (default) 확인
```

아래를 **`pvc.yaml`** 로 저장합니다.

```yaml title="pvc.yaml"
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  namespace: dev
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 100Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: writer
  namespace: dev
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh","-c","sleep 3600"]
      volumeMounts:
        - name: d
          mountPath: /data
  volumes:
    - name: d
      persistentVolumeClaim:
        claimName: data-pvc
```

```powershell
kubectl apply -f pvc.yaml
kubectl get pvc,pv -n dev        # STATUS: Bound

# 데이터 기록 (sh -c 안은 컨테이너 셸)
kubectl exec writer -n dev -- sh -c "echo hello-persist > /data/note.txt"
```

### Pod를 죽여도 데이터가 남는지 확인

```powershell
kubectl delete pod writer -n dev
kubectl apply -f pvc.yaml         # 같은 PVC를 다시 마운트하는 Pod 재생성
kubectl exec writer -n dev -- cat /data/note.txt   # hello-persist 그대로!
```

!!! success "영속성 증명"
    Pod는 새로 떴지만 데이터는 보존됩니다 — PV/PVC가 Pod와 **독립적인 수명**을 갖기 때문입니다.

---

## 4-5. RBAC: 최소 권한 (맛보기)

"누가 무엇을 할 수 있는가"를 설계합니다.

```powershell
kubectl create serviceaccount app-sa -n dev
```

아래를 **`rbac.yaml`** 로 저장합니다.

```yaml title="rbac.yaml"
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get","list","watch"]      # 읽기만
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: dev
  name: read-pods
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```powershell
kubectl apply -f rbac.yaml

# 권한 시뮬레이션 (can-i)
kubectl auth can-i list pods   -n dev --as=system:serviceaccount:dev:app-sa   # yes
kubectl auth can-i delete pods -n dev --as=system:serviceaccount:dev:app-sa   # no
```

!!! info "최소 권한 원칙"
    부여한 만큼만(list pods=yes, delete=no) 가능합니다. 와일드카드(`verbs: ["*"]`)는 피하고
    앱마다 전용 ServiceAccount로 권한을 좁힙니다.

---

## 정리(실습 청소)

```powershell
# 기본 네임스페이스 원복
kubectl config set-context --current --namespace=default

# dev 네임스페이스 통째로 정리 (안에 만든 리소스 모두 삭제)
kubectl delete namespace dev
```

---

## 도전과제 🔥

??? question "도전 1 (★★☆) · 한 이미지, 두 환경"
    `dev` 와 `prod` 네임스페이스에 서로 다른 ConfigMap(APP_MODE=debug / production)을 두고,
    동일한 Deployment를 양쪽에 배포해 환경변수가 다르게 주입되는지 확인하세요.

??? question "도전 2 (★★★) · 운영급 스택 캡스톤"
    Lab 3의 `web-stack.yaml` 을 `dev` 네임스페이스에 배포하되:
    (1) ConfigMap으로 설정 주입, (2) PVC로 콘텐츠 영속화, (3) ResourceQuota 적용,
    (4) 전용 ServiceAccount + 읽기 전용 RBAC 를 모두 결합해 보세요.

??? question "도전 3 (★★☆) · emptyDir vs PVC 수명 비교"
    `emptyDir` 볼륨에 파일을 쓰고 Pod 삭제→재생성 시 사라지는 것과,
    PVC에서는 유지되는 것을 비교 실험으로 보이세요.

---

## 정리 & 마무리

- **Namespace + Quota** 로 구역과 한도를, **ConfigMap/Secret** 으로 설정·비밀을 분리
- **PV/PVC** 로 데이터를 Pod와 독립적으로 영속화 (Rancher Desktop `local-path`)
- **RBAC** 로 최소 권한 설계

🎉 4개 실습을 모두 마쳤습니다. PPT의 *컨테이너 → 이미지 → Docker 한계 → 오케스트레이션 → 쿠버네티스 → Pod →
워크로드·노출 → 운영·보안* 흐름을 직접 손으로 재현했습니다.

다음 학습으로는 Helm/Kustomize(패키징), Ingress·Gateway API(L7 라우팅),
모니터링(Prometheus/Grafana), GitOps(ArgoCD)를 추천합니다.
