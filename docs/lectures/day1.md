# Day 1 · 쿠버네티스 구조 완전 정복

!!! abstract "오늘의 한 줄"
    컨테이너의 **본질**(격리된 프로세스)부터 쿠버네티스 **클러스터 구조**와 **Pod** 까지,
    "무엇이 어떻게 동작하는가"를 그림으로 이해하는 하루입니다.

[:material-file-powerpoint: Day 1 슬라이드 내려받기 (.pptx)](../assets/day1-kubernetes-architecture.pptx){ .md-button .md-button--primary }

!!! tip "함께 보면 좋은 실습"
    이 강의 내용은 [Lab 1 · 컨테이너 & 이미지 기초](../labs/lab1-container-basics.md) 와
    [Lab 2 · 쿠버네티스 첫걸음](../labs/lab2-kubernetes-start.md) 으로 직접 따라 할 수 있습니다.

## 오늘의 목표

- 컨테이너가 **왜** 필요한지, VM과 무엇이 다른지 한 문장으로 설명한다
- Namespace·cgroups가 격리와 자원 제한을 어떻게 만드는지 안다
- 이미지 레이어 구조와 런타임 계층(Docker→containerd→runc)을 그린다
- 단일 호스트 Docker의 한계를 이해한다
- 쿠버네티스 클러스터 구조(Control Plane / Worker)를 구성요소 단위로 이해한다
- 선언형 YAML로 Pod를 이해한다

> 오늘의 키워드: **격리**, **이미지(불변)**, **선언형**.

---

## 1. 컨테이너 기술이란?

### 문제: "제 PC에선 됐는데요?"

개발 PC·테스트·운영 서버의 OS·라이브러리·설정이 조금씩 다릅니다. 같은 코드라도 환경이 다르면 결과가 달라집니다.
근본 원인은 **애플리케이션과 '실행 환경'이 분리**되어 있다는 점입니다.

### 정의

> **컨테이너** = 애플리케이션과 실행에 필요한 모든 것(라이브러리·런타임·설정)을 하나로 패키징해 **격리 실행**하는 단위.

호스트 OS 커널은 공유하되, 프로세스/파일시스템/네트워크는 격리됩니다. **이미지(설계도)** 로 배포하고 **컨테이너(실행체)** 로 동작합니다.

### CA(Container Artifact): 배포의 단위는 '이미지'

- Artifact = 빌드로 만들어진 실행 가능한 패키지. 컨테이너 환경의 CA는 **이미지 그 자체**.
- 코드 + 런타임 + 라이브러리 + OS설정 + 환경변수를 통째로 박제한 **불변 패키지**.
- 핵심: "실행하기 **전의 정적 상태**"를 규격화한 것.

!!! info "Immutable Infrastructure (불변 인프라)"
    배포 단위(이미지)는 **절대 수정하지 않습니다**. 고쳐야 하면 실행 중 컨테이너를 만지는 게 아니라
    **새 이미지를 만들어 교체**합니다. 이를 통해 환경 **Drift(설정 뒤틀림)** 를 원천 차단합니다.

### Pets vs Cattle — 관리 대상의 변화

| 과거 — 서버 (Pets) | 현재 — 이미지 (Cattle) |
| --- | --- |
| 한 대 한 대 정성껏 수리 | 문제 생기면 버리고 새로 교체 |
| 죽으면 큰일 → 살려내야 함 | 이미지 버전(v1→v2)만 관리 |
| 상태가 제각각 → 재현 불가 | 어디서나 100% 동일·재현 가능 |

> 패러다임 전환: '서버의 개별 상태를 관리' → '이미지 버전을 관리'. **이제 서버가 아니라 이미지를 배포합니다.**

---

## 2. 컨테이너 vs 가상머신(VM)

가장 큰 차이는 **'어느 계층까지 가상화하느냐'** 입니다.

| 비교 항목 | VM (가상 머신) | 컨테이너 |
| --- | --- | --- |
| 가상화 수준 | 하드웨어 수준 | 운영체제(커널) 수준 |
| 운영체제 | VM마다 Guest OS 포함 | 호스트 OS 커널 공유 |
| 용량 | GB 단위 | MB 단위 |
| 기동 속도 | 수십 초~분 | 수십 ms~초 |
| 격리/보안 | 매우 강함(HW 격리) | 보통(프로세스 격리) |
| 이식성 | 낮음 | 매우 높음 |

!!! note "둘은 경쟁이 아니라 계층"
    하드웨어 → VM → 컨테이너. 실무에선 보통 **VM 위에 컨테이너**를 올립니다.
    (쿠버네티스 노드 자체가 대개 VM입니다.)

---

## 3. 핵심 기술: 격리는 어떻게 만들어지나

### 컨테이너의 실체 = 격리된 '프로세스' 하나

- 새 컴퓨터를 부팅하는 게 아니라, 켜져 있는 호스트에서 '특수한 벽'을 가진 프로세스를 하나 더 실행.
- **생존 조건**: 메인 프로세스(PID 1)가 살아있는 동안만 '실행 중(Up)'.
- **사멸 조건**: PID 1이 종료되면 즉시 '종료(Exited)'. (`docker run ubuntu echo hi` 는 출력 후 바로 종료)

### 격리의 기술적 근거: chroot → Namespace → cgroups

- **chroot**: 프로세스가 보는 루트(`/`)를 특정 폴더로 바꿔 파일시스템을 가둠 (격리의 시작, 파일시스템만 격리).
- **Namespace**: '무엇을 볼 수 있는가'를 격리 — **시야 격리** (PID/NET/MNT/UTS/IPC/USER).
- **cgroups**: '얼마나 쓸 수 있는가'를 제한 — **자원 격리** (CPU/메모리/IO). 초과 시 OOM Kill / throttle.

> 컨테이너 = chroot(뿌리) + Namespace(시야) + cgroups(자원)의 조합. 특별한 가상화가 아니라 **커널 기능의 조합**입니다.

---

## 4. 컨테이너 이미지 구조

- **이미지 vs 컨테이너**: 이미지=읽기전용 템플릿(설계도), 컨테이너=이미지를 실행한 인스턴스(실행체).
- **레이어 구조**: 이미지는 여러 읽기전용 레이어가 쌓인 것. Dockerfile의 각 명령이 하나의 레이어를 만들고,
  맨 위에 컨테이너 쓰기 레이어가 얹힙니다(Copy-on-Write).
- **레이어의 이점**: 공유(같은 base 재사용), 캐시(바뀐 레이어부터만 재빌드).

```dockerfile
FROM python:3.11-slim            # 베이스 이미지
WORKDIR /app
COPY requirements.txt .          # 의존성 먼저 (캐시 활용)
RUN pip install -r requirements.txt
COPY . .                         # 내 소스 (자주 바뀜 → 뒤로)
CMD ["python", "app.py"]         # 컨테이너 시작 명령
```

> 자주 바뀌는 것은 뒤로, 거의 안 바뀌는 것은 앞으로 → 캐시 적중률↑

---

## 5. 컨테이너 런타임

런타임에도 계층이 있습니다.

- **고수준 런타임**: 이미지 관리·API·네트워크 — `containerd`, `CRI-O`
- **저수준 런타임**: 실제로 namespace·cgroups를 만들어 프로세스 실행 — `runc`

호출 사슬: **Docker(도구) → containerd(관리) → runc(실행)**.
쿠버네티스는 Docker가 아니라 **CRI** 표준을 통해 런타임과 대화합니다(Dockershim 제거 이후 기본은 containerd).

---

## 6. 이미지 공유: 레지스트리

- **Registry** = 이미지를 모아둔 중앙 저장소(도서관). 예: Docker Hub.
- **Repository** = 한 앱의 여러 버전(태그)을 담는 책꽂이 (예: `nginx:1.27`, `nginx:alpine`).
- 주소 체계: `[Registry]/[Repository]:[Tag]` (Registry 생략 시 Docker Hub).
- 흐름: **build → push → pull → run**.
- 운영: 클라우드/사설 레지스트리(ECR·ACR·GCR) + 버전 고정 태그. OCI 표준이라 어디서나 동일하게 사용.

---

## 7. Docker 네트워크 (요약)

- 드라이버: **bridge(기본)** · host · none · overlay.
- `-p 호스트포트:컨테이너포트` 로 외부 노출(포트 매핑).
- 사용자 정의 bridge 네트워크에선 **컨테이너 이름이 곧 DNS** → IP를 몰라도 통신.

---

## 8. Docker 한 대의 한계 (오늘의 전환점)

단일 호스트 Docker로는 다음을 **사람이 손으로** 해야 합니다.

- 자가 치유(죽은 컨테이너 자동 재시작) ✕
- 스케일링(부하에 따른 자동 증감) ✕
- 로드밸런싱(복제본에 트래픽 분배) ✕
- 무중단 배포·롤백 ✕
- 여러 호스트 통합 관리 ✕

> 이 다섯 가지를 자동으로 해주는 것이 **오케스트레이션 = 쿠버네티스** 입니다.

---

## 9. 오케스트레이션과 쿠버네티스의 등장

- **오케스트레이션** = 다수 컨테이너를 여러 호스트에 걸쳐 자동으로 배치·확장·복구·연결하는 것.
- 패러다임 전환: **명령형(어떻게)** → **선언형(무엇을)**. 원하는 상태를 선언하면 시스템이 유지.
- **쿠버네티스**는 구글의 Borg 경험에서 출발(2014 오픈소스, CNCF) → 사실상의 표준.

---

## 10. 쿠버네티스 클러스터 구조

클러스터 = **Control Plane(두뇌)** + **Worker Node(일꾼)**. 모든 대화는 **API 서버**를 거칩니다.

``` mermaid
graph LR
  subgraph CP[Control Plane 두뇌]
    API[kube-apiserver]
    ETCD[(etcd)]
    SCH[scheduler]
    CM[controller-manager]
  end
  subgraph W[Worker Node 일꾼]
    KUBELET[kubelet]
    KPROXY[kube-proxy]
    POD[Pods + 런타임]
  end
  API --- ETCD
  API --- SCH
  API --- CM
  CP -->|지시| KUBELET
  KUBELET --> POD
```

**Control Plane**

- `kube-apiserver`: 모든 요청의 관문(REST API). 인증·검증·etcd 기록.
- `etcd`: 클러스터의 모든 상태를 담는 분산 저장소(유일한 진실의 원천 — 백업 중요).
- `kube-scheduler`: 새 Pod를 어느 노드에 둘지 결정.
- `kube-controller-manager`: 현재 상태를 desired 상태로 맞추는 컨트롤러 묶음.

**Worker Node**

- `kubelet`: 노드의 대리인. apiserver 지시대로 Pod를 띄우고 상태 보고.
- `kube-proxy`: 노드의 네트워크 규칙 관리(Service 트래픽 전달).
- 컨테이너 런타임: containerd/CRI-O — 실제 컨테이너 실행.

### 'Pod 하나 만들어줘'가 처리되는 길

``` mermaid
graph LR
  U[kubectl] -->|1. 요청| API[apiserver]
  API -->|2. 기록| E[(etcd)]
  API -->|3. 스케줄| S[scheduler]
  API -->|4. 할당| K[kubelet]
  K -->|5. 실행| P[Pod]
```

> 사용자는 apiserver에만 말합니다. 나머지는 컨트롤러들이 watch하며 자동으로 협업합니다.

---

## 11. 선언형 vs 명령형 · YAML · Pod

### 선언형이 운영 표준

- **명령형**: '어떻게'를 한 단계씩 지시(`kubectl run/create/scale`). 빠르고 직관적, 학습에 좋음.
- **선언형**: '무엇을' 원하는지 파일로 선언(`kubectl apply -f`). 시스템이 desired 상태로 **수렴(Reconcile)**.
  Git으로 버전관리(GitOps)에 적합 → **운영 표준**.

### YAML 4필드

모든 오브젝트의 뼈대는 같습니다.

```yaml
apiVersion: v1            # 어떤 API 버전
kind: Pod                 # 무슨 오브젝트
metadata:                 # 이름·라벨 등 메타
  name: web
  labels: { app: web }
spec:                     # 원하는 상태(핵심)
  containers:
    - name: nginx
      image: nginx:1.27
      ports: [ { containerPort: 80 } ]
```

### Pod란

- 쿠버네티스가 배포하는 **가장 작은 단위**. 1개 이상의 컨테이너 묶음.
- 같은 Pod의 컨테이너는 네트워크(IP·포트)와 일부 볼륨을 공유 → `localhost` 로 통신.
- Pod는 **일회용(ephemeral)** — 죽으면 새 Pod로 '대체'됩니다(되살아나는 게 아님).
- 그래서 직접 만들기보다 상위 컨트롤러(**Deployment**)로 관리합니다. → Day 2

---

## 12. 오늘 정리

- 컨테이너 = 환경째 패키징된 **격리 프로세스**(chroot+Namespace+cgroups), 이미지는 **불변 레이어 패키지**.
- 런타임은 Docker→containerd→runc 계층. 이미지는 레지스트리로 공유.
- 단일 호스트 Docker의 한계 → **오케스트레이션(쿠버네티스)** 의 필요.
- 클러스터 = Control Plane(결정) + Worker(실행), **모든 길은 apiserver**.
- 선언형 YAML로 원하는 상태를 선언, 최소 단위는 **Pod**.

➡️ 이어지는 실습: [Lab 1](../labs/lab1-container-basics.md) → [Lab 2](../labs/lab2-kubernetes-start.md)
