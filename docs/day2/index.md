# 2일차 오늘의 실습

!!! abstract "오늘 할 것"
    - 내 노트북의 쿠버네티스 클러스터에 접속하고, 구성요소를 직접 찾아봅니다.
    - Pod를 명령형과 선언형(YAML) 두 가지로 만들고, 안을 들여다봅니다.
    - Deployment로 Pod 개수를 지키고(자가복구), 늘리고 줄이고(스케일링), 새 버전으로 바꾸고 되돌립니다(롤링 업데이트, 롤백).

1일차에 도커로 겪은 한계 가운데 **2번 "죽으면 아무도 안 살린다"**와 **3번 "하나 더 띄우려면 포트를 바꿔야 한다"**가 오늘 풀립니다.

## 오늘의 실습

| 실습 | 제목 | 시간 | 교안 |
|---|---|---|---|
| [실습 1](lab-1.md) | 쿠버네티스가 켜졌는지 확인 | 5분 | 섹션 1 |
| [실습 2](lab-2.md) | 클러스터 구성요소 찾기 | 10분 | 섹션 2 |
| [실습 3](lab-3.md) | Pod 만들기 (명령형, 선언형) | 12분 | 섹션 3 |
| [실습 4](lab-4.md) | Pod 들여다보기와 라벨, IP 비교 | 12분 | 섹션 4 |
| [실습 5](lab-5.md) | Deployment와 자가복구, 스케일링 | 12분 | 섹션 5 |
| [실습 6](lab-6.md) | 롤링 업데이트와 롤백 | 12분 | 섹션 6 |

## 사전 과제 확인

2일차는 쿠버네티스가 켜져 있어야 시작할 수 있습니다. 아래 명령에서 `docker-desktop`이 `Ready`로 나오는지 먼저 확인합니다.

```bash
kubectl get nodes
```

```text
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   1d    v1.3x.x
```
<small>출력은 예시입니다. AGE와 VERSION은 사람마다 다릅니다.</small>

오류가 나거나 노드가 안 보이면 [Kubernetes 켜기](../setup/3-kubernetes.md)를 지금 따라 합니다. 처음 켤 때 몇 분 걸리니, 켜지는 동안 강의를 들으며 기다리면 됩니다.

## 작업 폴더

오늘 실습은 모두 `k8s-labs\day2` 폴더에서 합니다. 터미널을 열고 아래 명령으로 폴더를 만들고 이동합니다.

=== "Windows (PowerShell)"

    ```powershell
    mkdir $HOME\k8s-labs\day2; cd $HOME\k8s-labs\day2
    ```

=== "Mac (터미널)"

    ```bash
    mkdir -p ~/k8s-labs/day2 && cd ~/k8s-labs/day2
    ```

VS Code로 이 폴더를 열어 두면 YAML 파일을 만들고 고치기 편합니다.

```bash
code .
```

## 오늘 쓰는 파일

| 파일 | 쓰는 곳 | 비고 |
|---|---|---|
| [pod.yaml](../files/day2/pod.yaml){ download } | 실습 3 | 명령으로 직접 뽑아 만듭니다. 내려받기는 막혔을 때만 |
| [deployment.yaml](../files/day2/deployment.yaml){ download } | 실습 5, 6 | **3일차에도 씁니다. 지우지 마세요** |

## 하루를 마칠 때 정리

실습 4까지 만든 Pod `web`, `web2`는 실습 4 끝에서 지웠습니다. 혹시 남아 있으면 지웁니다. 이미 없으면 `--ignore-not-found` 덕분에 오류 없이 넘어갑니다.

```bash
kubectl delete pod web web2 --ignore-not-found
```

남은 것을 확인합니다.

```bash
kubectl get all
```

```text
NAME                       READY   STATUS    RESTARTS   AGE
pod/web-7c5b8d9f6d-4xk2p   1/1     Running   0          10m
pod/web-7c5b8d9f6d-9qz7m   1/1     Running   0          10m
pod/web-7c5b8d9f6d-h2lwn   1/1     Running   0          10m

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   1d

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/web   3/3     3            3           40m

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/web-5d8f7c6b9d   0         0         0       30m
replicaset.apps/web-7c5b8d9f6d   3         3         3       40m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

!!! warning "web Deployment와 deployment.yaml은 지우지 마세요"
    3일차 첫 실습에서 오늘 만든 **`web` Deployment**에 Service를 붙입니다. `kubectl delete -f deployment.yaml`은 실행하지 않습니다. `deployment.yaml` 파일도 그대로 둡니다.

    - `service/kubernetes`는 쿠버네티스가 원래 만들어 두는 것이라 지우지 않습니다.
    - 실수로 지웠어도 괜찮습니다. 3일차 실습 1에서 다시 만드는 방법을 안내합니다.
    - Docker Desktop을 꺼도 클러스터 안의 리소스는 남아 있습니다. 다음 날 켜면 `web` Pod 3개가 다시 Running이 됩니다.
