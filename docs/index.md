# 쿠버네티스 15일 실습 가이드

이 사이트는 강의 PPT의 흐름을 **직접 손으로 따라 하며** 익히도록 만든 실습 가이드입니다.
모든 실습은 **Rancher Desktop** 하나로 진행합니다. (컨테이너 런타임 + 내장 쿠버네티스 제공)

## 학습 흐름

PPT와 동일한 스토리라인을 실습으로 재현합니다.

``` mermaid
graph LR
  A[컨테이너 개념·원리] --> B[이미지]
  B --> C[Docker 단일 호스트 한계]
  C --> D[오케스트레이션 필요]
  D --> E[쿠버네티스 표준]
  E --> F[아키텍처 · Pod]
```

| 실습 | 내용 | PPT 연계 |
| --- | --- | --- |
| [Lab 1](labs/lab1-container-basics.md) | 컨테이너 실행, 이미지 빌드, Docker 한계 체험 | 컨테이너 개념·이미지·Docker 한계 |
| [Lab 2](labs/lab2-kubernetes-start.md) | Pod·Deployment, 자가 치유 | 오케스트레이션·쿠버네티스·Pod |
| [Lab 3](labs/lab3-workloads-service.md) | 롤링 업데이트·롤백, Service 노출 | 워크로드 안정화·노출 |
| [Lab 4](labs/lab4-operations.md) | Namespace·ConfigMap·Secret·PV/PVC·RBAC | 운영 기반·자동화·보안 |

## 시작하기 전에

1. 먼저 [Rancher Desktop 설치·설정](setup/rancher-desktop.md)을 완료하세요.
2. 각 Lab은 **순서대로** 진행하도록 설계되어 있습니다. (앞 실습 결과를 뒤에서 사용)
3. 각 Lab 끝의 **도전과제**는 배운 내용을 응용합니다. 막히면 힌트를 펼쳐 보세요.

!!! tip "표기 약속 (기준 환경)"
    - **컨테이너 엔진: Docker(dockerd) 모드** → CLI는 `docker` 를 사용합니다. (containerd 모드라면 `docker` 를 `nerdctl` 로 바꿔 읽으세요)
    - **터미널: Windows PowerShell** 기준으로 작성했습니다. `docker`·`kubectl` 명령은 macOS/Linux에서도 동일하게 동작하며, 셸이 다른 부분(파일 생성·반복문 등)은 **PowerShell / bash 탭**으로 함께 제공합니다.
    - PowerShell 주의: `&&` 연결은 쓰지 않고 줄을 나눠 실행하며, HTTP 확인은 브라우저나 `curl.exe` 를 씁니다. (자세한 내용은 [환경 준비](setup/rancher-desktop.md) 4번 참고)
    - `kubectl` 은 Rancher Desktop이 함께 설치합니다. 별도 설치가 필요 없습니다.

!!! note "왜 Rancher Desktop인가"
    Minikube/kubeadm 대신 Rancher Desktop을 씁니다. **컨테이너 런타임(containerd/dockerd)** 과
    **내장 쿠버네티스(k3s)** 를 한 번에 제공하므로, 컨테이너 기초부터 쿠버네티스까지
    추가 설치 없이 한 도구로 이어서 실습할 수 있습니다.
