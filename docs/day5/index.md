# 5일차 오늘의 실습

마지막 날입니다. 앞의 절반은 클러스터를 지키는 법(root 금지, 최소 권한, Pod 사이 통신 막기)을 실습하고, 뒤의 절반은 1일차에 도커로 띄웠던 미니 블로그를 쿠버네티스 YAML로 옮겨 배포한 뒤 일부러 고장 낸 블로그를 스스로 고칩니다.

!!! abstract "오늘의 목표"
    - 컨테이너를 root로 돌리지 않게 하고, 네임스페이스 라벨 하나로 기준을 강제합니다.
    - Pod에 꽂히는 토큰의 권한을 확인하고 최소로 줄입니다.
    - NetworkPolicy로 필요한 통신만 열어 둡니다.
    - 미니 블로그를 YAML 두 장으로 배포하고, 장애를 get, describe, logs, events 순서로 찾습니다.

## 실습 목록

| 실습 | 내용 | 시간 | 교안 |
|---|---|---|---|
| [실습 1](lab-1.md) | securityContext와 PSA | 12분 | 섹션 1 |
| [실습 2](lab-2.md) | RBAC | 12분 | 섹션 2 |
| [실습 3](lab-3.md) | NetworkPolicy | 12분 | 섹션 3 |
| [실습 4](lab-4.md) | 미니 블로그 YAML 빈칸 채우기 | 10분 | 섹션 4 |
| [실습 5](lab-5.md) | 미니 블로그 종합 배포 | 15분 | 섹션 5 |
| [실습 6](lab-6.md) | 고장 난 미니 블로그 고치기, 과정 마무리 | 12분 | 섹션 6 |

STEP 번호는 강의 장표와 같습니다. 실습 1의 STEP 1부터 실습 6의 STEP 27까지 하루 동안 이어집니다.

## 시작 전 확인

Docker Desktop과 Kubernetes가 켜져 있는지 봅니다.

```bash
kubectl get nodes
```

```text
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   4d    v1.3x.x
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

### Envoy Gateway가 설치되어 있어야 합니다 { #envoy-check }

실습 4, 5의 미니 블로그는 3일차에 설치한 **Envoy Gateway**로 입구를 만듭니다. 설치가 남아 있는지 확인합니다.

```bash
kubectl get pods -n envoy-gateway-system
```

```text
NAME                             READY   STATUS    RESTARTS   AGE
envoy-gateway-6b8f7d9c4-abcde    1/1     Running   0          2d
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`envoy-gateway`로 시작하는 Pod가 `Running`이면 됩니다. `No resources found` 또는 `namespaces "envoy-gateway-system" not found`가 나오면 [3일차 실습 5 Envoy Gateway 설치](../day3/lab-5.md)를 먼저 다시 합니다.

### 1일차 미니 블로그 이미지

1일차 실습 5에서 쓴 `skilleat/backend:v3-kb5`, `skilleat/frontend:v3-kb5` 이미지를 오늘 다시 씁니다. 지웠어도 쿠버네티스가 Docker Hub에서 다시 받으니 괜찮습니다. 남아 있는지만 봅니다.

=== "Windows (PowerShell)"

    ```powershell
    docker images | Select-String skilleat
    ```

=== "Mac (터미널)"

    ```bash
    docker images | grep skilleat
    ```

!!! warning "Apple Silicon Mac"
    1일차에 미니 블로그가 `exec format error`나 `no match for platform`으로 안 떴다면 오늘도 같은 문제가 납니다. 수업 시작 전에 채팅으로 알려 주세요.

## 작업 폴더 만들기

오늘 실습은 모두 `k8s-labs/day5` 폴더에서 합니다.

=== "Windows (PowerShell)"

    ```powershell
    mkdir $HOME\k8s-labs\day5; cd $HOME\k8s-labs\day5
    ```

=== "Mac (터미널)"

    ```bash
    mkdir -p ~/k8s-labs/day5 && cd ~/k8s-labs/day5
    ```

`code .`을 입력하면 이 폴더가 VS Code로 열립니다. 각 실습 페이지의 **내려받기** 링크로 받은 YAML 파일은 이 폴더에 넣습니다.

!!! info "터미널을 새로 열었다면"
    새 터미널은 홈 폴더에서 시작합니다. `kubectl apply -f 파일` 전에는 위의 `cd` 명령으로 작업 폴더로 다시 이동합니다.

## 오늘 쓰는 파일

| 파일 | 실습 |
|---|---|
| [bad-pod.yaml](../files/day5/bad-pod.yaml){ download }, [good-pod.yaml](../files/day5/good-pod.yaml){ download } | 실습 1 |
| [no-token.yaml](../files/day5/no-token.yaml){ download }, [pod-reader-role.yaml](../files/day5/pod-reader-role.yaml){ download } (선택) | 실습 2 |
| [deny-all.yaml](../files/day5/deny-all.yaml){ download }, [allow-backend-db.yaml](../files/day5/allow-backend-db.yaml){ download }, [allow-dns.yaml](../files/day5/allow-dns.yaml){ download } | 실습 3 |
| [miniblog-backend-blank.yaml](../files/day5/miniblog-backend-blank.yaml){ download }, [miniblog-frontend-blank.yaml](../files/day5/miniblog-frontend-blank.yaml){ download } | 실습 4 |
| [miniblog-backend-answer.yaml](../files/day5/miniblog-backend-answer.yaml){ download }, [miniblog-frontend-answer.yaml](../files/day5/miniblog-frontend-answer.yaml){ download } | 실습 4, 5 (정답) |
| [broken-blog.yaml](../files/day5/broken-blog.yaml){ download } | 실습 6 |

## 정리

실습마다 끝에 정리 명령이 있습니다. 과정 전체를 정리하는 방법은 [실습 6의 과정 마무리](lab-6.md#wrap-up)에 있습니다.
