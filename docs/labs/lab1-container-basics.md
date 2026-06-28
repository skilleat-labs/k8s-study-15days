# Lab 1 · 컨테이너 & 이미지 기초

!!! abstract "학습 목표"
    - 컨테이너가 "환경째 패키징된 격리 프로세스"임을 직접 확인한다
    - 이미지를 받아 컨테이너로 실행하고, 직접 이미지를 **빌드**한다
    - 컨테이너 = **프로세스(PID 1)** 임을 체험한다
    - **Docker(단일 호스트) 한 대의 한계**를 몸으로 느낀다 → Lab 2(쿠버네티스) 동기

    PPT 연계: *컨테이너 개념 · 이미지 · Docker 한계*

!!! note "준비"
    [Rancher Desktop 설치·설정](../setup/rancher-desktop.md)을 마쳤고, `nerdctl version` 이 정상 출력되어야 합니다.
    (dockerd 모드라면 `nerdctl` 을 `docker` 로 바꿔 실행)

---

## 1-1. 이미지 받아 컨테이너 실행하기

이미지(설계도)를 받아 컨테이너(실행체)로 띄웁니다.

```bash
# 1) 공식 nginx 이미지 받기
nerdctl pull nginx:1.27

# 2) 컨테이너로 실행 (백그라운드 -d, 호스트 8080 → 컨테이너 80)
nerdctl run -d --name web -p 8080:80 nginx:1.27

# 3) 동작 확인
nerdctl ps
curl http://localhost:8080
```

브라우저에서 `http://localhost:8080` 을 열어 nginx 기본 페이지가 보이면 성공입니다.

!!! question "확인"
    `nerdctl images` 로 받은 이미지를, `nerdctl ps` 로 실행 중인 컨테이너를 봅니다.
    **하나의 이미지로 여러 컨테이너**를 띄울 수 있습니다.
    ```bash
    nerdctl run -d --name web2 -p 8081:80 nginx:1.27
    nerdctl ps   # web, web2 두 개가 같은 이미지로 동작
    ```

---

## 1-2. 컨테이너는 "격리된 프로세스"다

컨테이너 안에서 보이는 세상이 호스트와 다른 것을 확인합니다. (Namespace 격리)

```bash
# 컨테이너 안으로 들어가기
nerdctl exec -it web sh

# (컨테이너 안에서) 자기 자신이 PID 1 임을 확인
ps -ef
hostname
ip addr        # 자기만의 IP (없으면 'apk add iproute2' 후 재시도)
exit
```

!!! info "방금 본 것"
    컨테이너 안에서는 **자기 프로세스만** 보이고, **자기만의 호스트명·IP**를 가집니다.
    호스트의 다른 프로세스는 보이지 않습니다 — 이것이 **Namespace(시야 격리)** 입니다.

### 컨테이너 = 프로세스 (PID 1 수명)

컨테이너는 메인 프로세스가 살아있는 동안만 살아 있습니다.

```bash
# 할 일(echo)만 하고 즉시 끝나는 컨테이너
nerdctl run --name once ubuntu echo "Hello Container"

# 상태 확인 → Exited (메인 프로세스가 끝났으니 컨테이너도 종료)
nerdctl ps -a | grep once
```

!!! success "결론"
    `docker run ubuntu echo "..."` 는 출력 후 **즉시 종료**됩니다.
    컨테이너는 "계속 켜진 서버"가 아니라 **PID 1이 살아있는 동안만 사는 프로세스**입니다.

---

## 1-3. 내 이미지 직접 빌드하기

`Dockerfile`(설계도) → 이미지 → 컨테이너 흐름을 직접 만듭니다.

```bash
# 작업 폴더 생성
mkdir -p ~/lab1-app && cd ~/lab1-app

# 간단한 앱
cat > app.py <<'PY'
print("hello from my image")
PY

# Dockerfile 작성
cat > Dockerfile <<'DOCKER'
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
DOCKER
```

이미지를 빌드합니다. **쿠버네티스에서도 쓰려면** containerd의 `k8s.io` 네임스페이스로 빌드합니다(Lab 2에서 사용).

```bash
# containerd 모드: k8s가 보는 네임스페이스로 빌드
nerdctl --namespace k8s.io build -t myapp:1.0 .

# 실행해 보기
nerdctl --namespace k8s.io run --rm myapp:1.0
```

기대 결과: `hello from my image` 출력.

!!! tip "레이어 캐시 관찰"
    `app.py` 만 바꾸고 다시 빌드하면, 위쪽 `FROM`/패키지 레이어는 **CACHED** 로 재사용됩니다.
    ```bash
    nerdctl --namespace k8s.io build -t myapp:1.1 .   # 일부 레이어 CACHED
    ```
    자주 바뀌는 `COPY` 는 Dockerfile 뒤쪽에 두면 캐시 적중률이 올라갑니다.

---

## 1-4. Docker 한 대의 한계 체험 (중요)

여기서 느끼는 불편함이 **쿠버네티스가 필요한 이유**입니다.

### 자동 복구가 없다

```bash
# web 컨테이너를 강제 종료
nerdctl kill web
nerdctl ps          # web 이 사라짐 — 아무도 다시 살려주지 않는다
```

### 수동 확장 / 포트 충돌

```bash
# 같은 80포트로 또 띄우려 하면 포트 충돌
nerdctl run -d --name web3 -p 8080:80 nginx:1.27   # 8080 이미 사용 중 → 실패
```

복제본을 늘리려면 포트를 일일이 다르게 잡고, 그 앞에 로드밸런서도 직접 세워야 합니다.

!!! danger "단일 호스트 Docker가 못 하는 것"
    - **자가 치유**: 죽은 컨테이너 자동 재시작/재배치 ✕
    - **스케일링**: 부하에 따른 자동 증감 ✕
    - **로드밸런싱**: 복제본에 트래픽 분배 ✕
    - **무중단 배포·롤백** ✕
    - **여러 호스트 통합 관리** ✕

    → 이 다섯 가지를 자동으로 해주는 것이 **오케스트레이션(쿠버네티스)** 입니다. (Lab 2)

---

## 도전과제 🔥

??? question "도전 1 (★☆☆) · 한 이미지, 두 컨테이너"
    `nginx:1.27` 하나로 컨테이너 3개를 서로 다른 포트(8080/8081/8082)로 띄우고
    `nerdctl ps` 로 확인하세요. "하나의 이미지 → 여러 컨테이너"를 체감합니다.

    ??? note "힌트"
        ```bash
        nerdctl run -d --name w1 -p 8080:80 nginx:1.27
        nerdctl run -d --name w2 -p 8081:80 nginx:1.27
        nerdctl run -d --name w3 -p 8082:80 nginx:1.27
        ```

??? question "도전 2 (★★☆) · 캐시 최적화 Dockerfile"
    의존성 설치를 캐시하도록 `requirements.txt` 를 먼저 `COPY` → `pip install` → 그다음 소스 `COPY` 순서로
    Dockerfile을 재구성하고, 두 번째 빌드에서 의존성 레이어가 **CACHED** 되는지 확인하세요.

    ??? note "힌트"
        ```dockerfile
        FROM python:3.12-slim
        WORKDIR /app
        COPY requirements.txt .
        RUN pip install -r requirements.txt
        COPY . .
        CMD ["python", "app.py"]
        ```

??? question "도전 3 (★★☆) · 한계를 요구사항으로"
    방금 겪은 불편(자동복구 없음·수동확장·포트충돌)을 "원하는 자동 동작"으로 한 줄씩 바꿔 적어보세요.
    예) *컨테이너가 죽음 → '항상 N개가 떠 있도록 자동 유지'*. 이 요구사항이 Lab 2의 어떤 오브젝트로 충족될지 예상해 보세요.

---

## 정리 & 다음 단계

- 컨테이너 = **이미지로 패키징된 격리 프로세스**, PID 1이 살아있는 동안만 동작
- 이미지는 **레이어**로 빌드·캐시되며, 한 이미지로 여러 컨테이너 실행
- 단일 호스트 Docker는 복구·확장·LB를 **사람이 손으로** 해야 함

➡️ 다음: [Lab 2 · 쿠버네티스 첫걸음](lab2-kubernetes-start.md) — 위 한계를 쿠버네티스가 어떻게 자동으로 해결하는지 봅니다.
