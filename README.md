# 쿠버네티스 5일 실습 가이드 (광주전남 데이터센터 IT 인프라 전문과정)

사이트: https://skilleat-labs.github.io/gwangju-k8s-5days/

ITSA 광주전남 데이터센터 인력양성 전문과정 (과정1) IT 인프라 15시간용 실습 가이드입니다.
하루 3시간, 온라인, 5일. 실습 환경은 Docker Desktop 내장 쿠버네티스 (kubeadm 단일 노드). Windows와 Mac을 모두 다룹니다.

## 다시 쓸 때 순서

1. 최신으로 받기: `git pull --rebase`
2. 고치기: `docs/` 아래 페이지, 실습 파일은 `docs/files/day1~5/`
    - 실습 파일을 바꿨으면 zip도 다시 만든다:
      `cd docs/files && rm -f ../assets/k8s-labs-files.zip && zip -qr ../assets/k8s-labs-files.zip day1 day2 day3 day4 day5`
3. 확인: `mkdocs serve` (http://127.0.0.1:8000), 마지막에 `mkdocs build --strict` 경고 0건
4. 배포: `main`에 push하면 GitHub Actions가 `gh-pages` 브랜치로 배포 (1~2분)

작성 규칙(Windows/Mac 탭, 한 줄 명령, 문체)은 [GUIDE_RULES.md](GUIDE_RULES.md)에 있습니다.

## 교안과의 관계

- 강의 교안(pptx)과 STEP 번호, 명령, 파일 이름이 같습니다. 교안 빌드 스크립트는 로컬
  `~/Documents/스킬잇/🔶🔶01. 강의 교안 자료 /광주전남 데이터센터 IT - 쿠버네티스/5일과정/ppt-template/build_dayN.js`
- 교안을 고치면 이 가이드의 같은 실습 페이지도 같이 고친다 (반대도 마찬가지).

## 수업 전 확인 (Docker Desktop 실측 필요)

가이드의 출력 예시는 실측 전 예시입니다. 수업 전에 실제 출력으로 바꿉니다.

- NetworkPolicy가 실제로 막는지 (5일차 실습 3). 안 막히면 강사 시연
- LoadBalancer, Envoy Gateway 주소가 localhost로 열리는지 (3일차 실습 3, 5)
- 기본 StorageClass `hostpath`로 PVC가 바로 묶이는지 (4일차 실습 6)
- 미니 블로그 태그: 1일차 도커는 `v4-kb5`(짧은 이름 backend-service, amd64+arm64), 5일차 webapp은 `v3-kb5`(webapp FQDN, /app/data 저장, amd64 전용), 5일차 broken은 frontend `v4-kb5` + backend `v3-kb5`. Apple Silicon에서 v3가 쿠버네티스로 뜨는지 확인
- `polinux/stress`는 amd64 전용 (Apple Silicon은 `memory-hog-arm.yaml`)
