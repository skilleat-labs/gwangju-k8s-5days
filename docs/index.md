# 쿠버네티스 5일 실습 가이드

광주전남 데이터센터 IT 인프라 전문과정의 실습 가이드입니다. 강의 화면의 실습 장표와 STEP 번호가 같습니다. 강의를 들으며 이 페이지에서 명령을 복사해 따라 하면 됩니다.

!!! tip "Windows와 Mac을 한 번만 고르세요"
    명령이 OS마다 다를 때는 아래처럼 탭으로 나눠 두었습니다. 한 번 고르면 모든 페이지의 탭이 같이 바뀝니다.

    === "Windows (PowerShell)"

        ```powershell
        curl.exe http://localhost:8080
        ```

    === "Mac (터미널)"

        ```bash
        curl http://localhost:8080
        ```

    탭이 없는 명령은 Windows와 Mac에서 똑같이 입력합니다.

## 5일 동안 하는 것

| 일차 | 주제 | 실습 |
|---|---|---|
| [1일차](day1/index.md) | 컨테이너 완전 정복 | 컨테이너 띄우기, 프로세스와 메모리 상한, 내 이미지 만들기, 도전 과제, 미니 블로그, 도커의 한계 |
| [2일차](day2/index.md) | 쿠버네티스 구조, Pod, Deployment | 클러스터 확인, 구성요소 찾기, Pod 만들기, 라벨, 자가복구와 스케일링, 롤링 업데이트와 롤백 |
| [3일차](day3/index.md) | Service와 Gateway | Service, CoreDNS, NodePort와 LoadBalancer, EndpointSlice, Envoy Gateway, HTTPRoute |
| [4일차](day4/index.md) | 설정, 상태 점검, 데이터 | ConfigMap, Secret, readinessProbe, OOMKilled, emptyDir, PVC |
| [5일차](day5/index.md) | 보안과 종합 | securityContext와 PSA, RBAC, NetworkPolicy, 미니 블로그 종합 배포와 장애 찾기 |

## 실습 방식

- 하루 3시간, 섹션 6개입니다. 섹션마다 **개념 설명 → 바로 실습** 순서로 갑니다.
- 1일차는 도커만 씁니다. 쿠버네티스는 2일차부터 씁니다.
- 1일차에 도커로 띄운 **미니 블로그**를 5일차에 쿠버네티스로 그대로 옮깁니다.
- 모든 실습은 `k8s-labs` 폴더 아래 일차별 폴더에서 합니다. 각 일차의 "오늘의 실습" 페이지에 폴더 만드는 명령이 있습니다.

## 시작 전에

첫 수업 전에 [시작 전 준비](setup/1-docker-desktop.md)의 1, 2번을 끝내 주세요. 3번 Kubernetes 켜기는 **2일차 수업 전까지** 하면 됩니다.

| 준비 | 언제까지 |
|---|---|
| [Docker Desktop 설치](setup/1-docker-desktop.md) | 1일차 전 |
| [터미널과 편집기](setup/2-terminal-editor.md) | 1일차 전 |
| [Kubernetes 켜기](setup/3-kubernetes.md) | 2일차 전 |

막히면 [막혔을 때](setup/4-troubleshooting.md)를 먼저 보고, 해결이 안 되면 오류 문구를 **그대로 복사해서** 채팅에 올려 주세요.

## 실습 파일

실습에 쓰는 YAML과 Dockerfile은 [실습 파일](files.md)에서 한꺼번에 내려받을 수 있습니다. 각 실습 페이지에도 내려받기 링크가 있습니다.
