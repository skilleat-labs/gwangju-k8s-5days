# 실습 파일

실습에 쓰는 파일을 모았습니다. 각 실습 페이지에도 같은 파일의 내려받기 링크가 있습니다.

!!! tip "한 번에 받기"
    [전체 실습 파일 zip 내려받기](assets/k8s-labs-files.zip){ download }

    압축을 풀면 `day1` ~ `day5` 폴더가 나옵니다. 홈 폴더의 `k8s-labs` 안에 그대로 넣으면 실습 폴더와 이름이 맞습니다.

## 1일차 컨테이너

| 파일 | 쓰는 곳 |
|---|---|
| [Dockerfile](files/day1/Dockerfile){ download } | 실습 3 내 이미지 만들기 |
| [index.html](files/day1/index.html){ download } | 실습 3 내 이미지 만들기 |

## 2일차 Pod와 Deployment

| 파일 | 쓰는 곳 |
|---|---|
| [deployment.yaml](files/day2/deployment.yaml){ download } | 실습 5, 6 Deployment |
| [pod.yaml](files/day2/pod.yaml){ download } | 실습 3 Pod 만들기 (막혔을 때) |

## 3일차 Service와 Gateway

| 파일 | 쓰는 곳 |
|---|---|
| [app1.yaml](files/day3/app1.yaml){ download } | 실습 6 테스트 앱 1 |
| [app2.yaml](files/day3/app2.yaml){ download } | 실습 6 테스트 앱 2 |
| [gateway.yaml](files/day3/gateway.yaml){ download } | 실습 5 GatewayClass, Gateway |
| [route.yaml](files/day3/route.yaml){ download } | 실습 6 HTTPRoute |
| [web-deploy.yaml](files/day3/web-deploy.yaml){ download } | 2일차 web Deployment가 없을 때 |
| [web-np.yaml](files/day3/web-np.yaml){ download } | 실습 3 NodePort |
| [web-svc-broken.yaml](files/day3/web-svc-broken.yaml){ download } | 실습 4 고장 낸 Service |
| [web-svc.yaml](files/day3/web-svc.yaml){ download } | 실습 1 ClusterIP Service |

## 4일차 설정과 데이터

| 파일 | 쓰는 곳 |
|---|---|
| [cm-pod.yaml](files/day4/cm-pod.yaml){ download } | 실습 1 ConfigMap |
| [memory-hog-arm.yaml](files/day4/memory-hog-arm.yaml){ download } | 실습 4 OOMKilled (Apple Silicon Mac) |
| [memory-hog.yaml](files/day4/memory-hog.yaml){ download } | 실습 4 OOMKilled (Windows, Intel Mac) |
| [mysql-emptydir.yaml](files/day4/mysql-emptydir.yaml){ download } | 실습 5 MySQL emptyDir |
| [mysql-pvc-deploy.yaml](files/day4/mysql-pvc-deploy.yaml){ download } | 실습 6 MySQL + PVC |
| [mysql-pvc.yaml](files/day4/mysql-pvc.yaml){ download } | 실습 6 PVC |
| [probe-web.yaml](files/day4/probe-web.yaml){ download } | 실습 3 readinessProbe |
| [secret-demo.yaml](files/day4/secret-demo.yaml){ download } | 실습 2 Secret |

## 5일차 보안과 종합

| 파일 | 쓰는 곳 |
|---|---|
| [allow-backend-db.yaml](files/day5/allow-backend-db.yaml){ download } | 실습 3 backend → DB 허용 |
| [allow-dns.yaml](files/day5/allow-dns.yaml){ download } | 실습 3 DNS 허용 |
| [bad-pod.yaml](files/day5/bad-pod.yaml){ download } | 실습 1 거부될 Pod |
| [broken-blog.yaml](files/day5/broken-blog.yaml){ download } | 실습 6 고장 난 미니 블로그 |
| [deny-all.yaml](files/day5/deny-all.yaml){ download } | 실습 3 deny-all |
| [good-pod.yaml](files/day5/good-pod.yaml){ download } | 실습 1 통과할 Pod |
| [miniblog-backend-answer.yaml](files/day5/miniblog-backend-answer.yaml){ download } | 실습 4 정답 (backend) |
| [miniblog-backend-blank.yaml](files/day5/miniblog-backend-blank.yaml){ download } | 실습 4 빈칸 (backend) |
| [miniblog-frontend-answer.yaml](files/day5/miniblog-frontend-answer.yaml){ download } | 실습 4 정답 (frontend) |
| [miniblog-frontend-blank.yaml](files/day5/miniblog-frontend-blank.yaml){ download } | 실습 4 빈칸 (frontend) |
| [no-token.yaml](files/day5/no-token.yaml){ download } | 실습 2 토큰 자동 마운트 끄기 |
| [pod-reader-role.yaml](files/day5/pod-reader-role.yaml){ download } | 실습 2 Role, RoleBinding |

!!! warning "빈칸 파일과 정답 파일"
    5일차 `*-blank.yaml`은 빈칸을 채워 쓰는 파일입니다. `*-answer.yaml`은 막혔을 때만 열어 보세요.
