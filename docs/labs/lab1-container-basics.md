# Lab 1 · 컨테이너 & 이미지 기초

!!! abstract "학습 목표"
    - 컨테이너가 "환경째 패키징된 격리 프로세스"임을 직접 확인한다
    - 이미지를 받아 컨테이너로 실행하고, 직접 이미지를 **빌드**한다
    - 컨테이너 = **프로세스(PID 1)** 임을 체험한다
    - **Docker(단일 호스트) 한 대의 한계**를 몸으로 느낀다 → Lab 2(쿠버네티스) 동기

    PPT 연계: *컨테이너 개념 · 이미지 · Docker 한계*

!!! note "준비 / 기준 환경"
    [Rancher Desktop 설치·설정](../setup/rancher-desktop.md)을 마쳤고, **Docker(dockerd) 모드**에서 `docker version` 이 정상 출력되어야 합니다.
    명령은 **Windows PowerShell** 기준입니다. (`&&` 미사용, 파일은 편집기로 저장)

---

## 1-1. 이미지 받아 컨테이너 실행하기

이미지(설계도)를 받아 컨테이너(실행체)로 띄웁니다.

```powershell
# 1) 공식 nginx 이미지 받기
docker pull nginx:1.27

# 2) 컨테이너로 실행 (백그라운드 -d, 호스트 8080 → 컨테이너 80)
docker run -d --name web -p 8080:80 nginx:1.27

# 3) 동작 확인
docker ps
```

브라우저에서 **http://localhost:8080** 을 열어 nginx 기본 페이지가 보이면 성공입니다.

!!! tip "터미널로 HTTP 확인"
    PowerShell에서 `curl` 은 `Invoke-WebRequest` 별칭이라 동작이 다릅니다. 진짜 curl은 `curl.exe` 로 부르세요.
    ```powershell
    curl.exe http://localhost:8080
    ```

!!! question "확인 · 하나의 이미지 → 여러 컨테이너"
    같은 이미지로 컨테이너를 하나 더 띄워봅니다.
    ```powershell
    docker run -d --name web2 -p 8081:80 nginx:1.27
    docker ps
    ```

---

## 1-2. 컨테이너는 "격리된 프로세스"다

컨테이너 안에서 보이는 세상이 호스트와 다른 것을 확인합니다. (Namespace 격리)

```powershell
# 컨테이너 안으로 들어가기 (셸 진입)
docker exec -it web sh
```

컨테이너 안(리눅스 셸)에서 아래를 실행합니다.

```sh
ps -ef        # 자기 자신이 PID 1 임을 확인
hostname      # 자기만의 호스트명
ip addr       # 자기만의 IP (명령이 없으면 'apk add iproute2' 후 재시도)
exit          # 컨테이너에서 빠져나오기
```

!!! info "방금 본 것"
    컨테이너 안에서는 **자기 프로세스만** 보이고, **자기만의 호스트명·IP**를 가집니다.
    호스트의 다른 프로세스는 보이지 않습니다 — 이것이 **Namespace(시야 격리)** 입니다.

### 컨테이너 = 프로세스 (PID 1 수명)

컨테이너는 메인 프로세스가 살아있는 동안만 살아 있습니다.

```powershell
# 할 일(echo)만 하고 즉시 끝나는 컨테이너
docker run --name once ubuntu echo "Hello Container"

# 전체 컨테이너 목록에서 once 의 STATUS 확인 → Exited
docker ps -a
```

!!! success "결론"
    `docker run ubuntu echo "..."` 는 출력 후 **즉시 종료(Exited)** 됩니다.
    컨테이너는 "계속 켜진 서버"가 아니라 **PID 1이 살아있는 동안만 사는 프로세스**입니다.

---

## 1-3. 내 이미지 직접 빌드하기

`Dockerfile`(설계도) → 이미지 → 컨테이너 흐름을 직접 만듭니다.

먼저 작업 폴더로 이동합니다.

```powershell
mkdir lab1-app
cd lab1-app
```

아래 두 파일을 편집기로 저장합니다. (같은 `lab1-app` 폴더 안)

```python title="app.py"
print("hello from my image")
```

```dockerfile title="Dockerfile"
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
```

??? tip "터미널에서 바로 파일 만들기 (선택)"
    === "PowerShell (Windows)"
        ```powershell
        'print("hello from my image")' | Set-Content -Encoding utf8 app.py
        @'
        FROM python:3.12-slim
        WORKDIR /app
        COPY app.py .
        CMD ["python", "app.py"]
        '@ | Set-Content -Encoding utf8 Dockerfile
        ```
    === "bash (macOS/Linux)"
        ```bash
        echo 'print("hello from my image")' > app.py
        cat > Dockerfile <<'EOF'
        FROM python:3.12-slim
        WORKDIR /app
        COPY app.py .
        CMD ["python", "app.py"]
        EOF
        ```

이미지를 빌드하고 실행합니다. (dockerd 모드라 별도 네임스페이스 불필요)

```powershell
# 이미지 빌드
docker build -t myapp:1.0 .

# 실행해 보기
docker run --rm myapp:1.0
```

기대 결과: `hello from my image` 출력.

!!! tip "레이어 캐시 관찰"
    `app.py` 만 바꾸고 다시 빌드하면, 위쪽 `FROM`/패키지 레이어는 **CACHED** 로 재사용됩니다.
    ```powershell
    docker build -t myapp:1.1 .
    ```
    자주 바뀌는 `COPY` 는 Dockerfile 뒤쪽에 두면 캐시 적중률이 올라갑니다.

---

## 1-4. Docker 한 대의 한계 체험 (중요)

여기서 느끼는 불편함이 **쿠버네티스가 필요한 이유**입니다.

### 자동 복구가 없다

```powershell
# web 컨테이너를 강제 종료
docker kill web

# 목록 확인 — web 이 사라짐. 아무도 다시 살려주지 않는다
docker ps
```

### 수동 확장 / 포트 충돌

```powershell
# 같은 8080 포트로 또 띄우려 하면 포트 충돌로 실패
docker run -d --name web3 -p 8080:80 nginx:1.27
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

## 정리(실습 청소)

```powershell
docker rm -f web web2 web3 once
```

---

## 도전과제 🔥

??? question "도전 1 (★☆☆) · 한 이미지, 두 컨테이너"
    `nginx:1.27` 하나로 컨테이너 3개를 서로 다른 포트(8080/8081/8082)로 띄우고
    `docker ps` 로 확인하세요. "하나의 이미지 → 여러 컨테이너"를 체감합니다.

    ??? note "힌트"
        ```powershell
        docker run -d --name w1 -p 8080:80 nginx:1.27
        docker run -d --name w2 -p 8081:80 nginx:1.27
        docker run -d --name w3 -p 8082:80 nginx:1.27
        ```

??? question "도전 2 (★★☆) · 캐시 최적화 Dockerfile"
    의존성 설치를 캐시하도록 `requirements.txt` 를 먼저 `COPY` → `pip install` → 그다음 소스 `COPY` 순서로
    Dockerfile을 재구성하고, 두 번째 빌드에서 의존성 레이어가 **CACHED** 되는지 확인하세요.

    ??? note "힌트"
        ```dockerfile title="Dockerfile"
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
