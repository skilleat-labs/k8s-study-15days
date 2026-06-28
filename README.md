# 쿠버네티스 15일 실습 가이드

Rancher Desktop 기반 컨테이너·쿠버네티스 핸즈온 실습 가이드입니다.
강의 PPT(컨테이너 개념 → 이미지 → Docker 한계 → 오케스트레이션 → 쿠버네티스 → Pod)의 흐름에 맞춰
직접 따라 할 수 있도록 구성했습니다.

## 구성

- `docs/setup/rancher-desktop.md` — 실습 환경(Rancher Desktop) 설치·설정
- `docs/labs/lab1-container-basics.md` — 컨테이너 & 이미지 기초
- `docs/labs/lab2-kubernetes-start.md` — 쿠버네티스 첫걸음 (Pod·Deployment)
- `docs/labs/lab3-workloads-service.md` — 워크로드와 서비스 노출
- `docs/labs/lab4-operations.md` — 운영 기반과 자동화 (Namespace·ConfigMap·Secret·PV/PVC·RBAC)

## 로컬에서 미리보기

```bash
pip install -r requirements.txt
mkdocs serve         # http://127.0.0.1:8000
```

## 배포 (GitHub Pages)

`main` 브랜치에 push하면 `.github/workflows/deploy.yml` 이 자동으로
`mkdocs gh-deploy` 를 실행해 `gh-pages` 브랜치로 배포합니다.

1. 저장소 **Settings → Pages → Build and deployment → Source: Deploy from a branch**
2. **Branch: `gh-pages` / `(root)`** 선택 후 저장
3. 잠시 후 `https://skilleat-labs.github.io/k8s-study-15days/` 에서 확인

> 수동 배포: `mkdocs gh-deploy --force`
