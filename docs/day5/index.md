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
| [실습 5](lab-5.md) | 미니 블로그 종합 배포, (선택) 운영 보안 설정 붙이기 | 23분 (선택 5분 포함) | 섹션 5 |
| [실습 6](lab-6.md) | 고장 난 미니 블로그 고치기, 과정 마무리 | 12분 | 섹션 6 |

STEP 번호는 강의 장표와 같습니다. 실습 1의 STEP 1부터 실습 6의 STEP 33까지 하루 동안 이어집니다.

!!! tip "추가 실습"
    시간이 남거나 복습할 때 [추가 실습. PVC 더 다뤄 보기](extra-pvc.md)를 해 봅니다. YAML 파일 없이 명령만 붙여 넣어 따라 할 수 있습니다 (약 20분).

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

3일차 끝에 `eg` Gateway와 GatewayClass를 지웠습니다. 남아 있는지도 봅니다.

```bash
kubectl get gateway -A
```

`eg`가 보이면 지웁니다. 실습 5 STEP 20에서도 한 번 더 확인합니다.

```bash
kubectl delete gateway eg
```

```bash
kubectl delete gatewayclass eg
```

### 미니 블로그 이미지

오늘은 `skilleat/backend:v3-kb5`, `skilleat/frontend:v3-kb5`를 씁니다. 1일차(`v4-kb5`)와 태그가 다릅니다. v3는 백엔드가 `/app/data` 폴더에 글을 저장해서 PVC와 맞고, frontend가 `webapp` 네임스페이스의 백엔드를 부르도록 만들어져 있습니다. 실습 6의 `broken` 블로그만 같은 네임스페이스의 백엔드를 부르는 `frontend:v4-kb5`를 씁니다.

처음 받을 때 시간이 걸리니 미리 받아 둡니다.

```bash
docker pull skilleat/backend:v3-kb5
```

```bash
docker pull skilleat/frontend:v3-kb5
```

!!! warning "Apple Silicon Mac"
    `v3-kb5`는 Intel(amd64)용만 있습니다. 실습 5에서 Pod가 `exec format error`나 `no match for platform`으로 안 뜨면 채팅으로 알려 주세요.

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

### git clone으로 한 번에 받기

파일을 하나씩 만들거나 내려받는 대신, 실습 저장소를 통째로 받아 오늘 폴더로 복사할 수 있습니다.

**① git이 있는지 확인** (`git version 2.x`처럼 나오면 됩니다)

```bash
git --version
```

??? info "git이 없다고 나오면"
    - Windows: PowerShell에서 `winget install --id Git.Git -e` 를 실행하거나 [git-scm.com](https://git-scm.com/download/win)에서 설치한 뒤, **터미널을 새로 엽니다.**
    - Mac: `git --version`을 입력하면 "명령어 라인 개발자 도구" 설치 창이 뜹니다. **설치**를 누르고 끝나면 다시 실행합니다.

**② 저장소 받기** (처음 한 번만. 이미 받았으면 ③으로)

=== "Windows (PowerShell)"

    ```powershell
    cd $HOME\k8s-labs
    git clone https://github.com/skilleat-labs/gwangju-k8s-5days.git
    ```

=== "Mac (터미널)"

    ```bash
    cd ~/k8s-labs
    git clone https://github.com/skilleat-labs/gwangju-k8s-5days.git
    ```

이미 받아 둔 저장소가 있으면 최신으로 맞춥니다.

=== "Windows (PowerShell)"

    ```powershell
    cd $HOME\k8s-labs\gwangju-k8s-5days; git pull
    ```

=== "Mac (터미널)"

    ```bash
    cd ~/k8s-labs/gwangju-k8s-5days && git pull
    ```

**③ 오늘 파일을 day5 폴더로 복사하고 이동**

=== "Windows (PowerShell)"

    ```powershell
    mkdir $HOME\k8s-labs\day5 -Force; Copy-Item $HOME\k8s-labs\gwangju-k8s-5days\docs\files\day5\* $HOME\k8s-labs\day5\ -Force; cd $HOME\k8s-labs\day5
    ```

=== "Mac (터미널)"

    ```bash
    mkdir -p ~/k8s-labs/day5 && cp ~/k8s-labs/gwangju-k8s-5days/docs/files/day5/* ~/k8s-labs/day5/ && cd ~/k8s-labs/day5
    ```

```bash
ls
```

아래 표의 파일이 모두 보이면 준비가 끝났습니다. 실습 중에 파일을 고쳐도 저장소 원본은 그대로라서, 다시 복사하면 처음 상태로 돌아갑니다.

### 하나씩 받기

| 파일 | 실습 |
|---|---|
| [bad-pod.yaml](../files/day5/bad-pod.yaml){ download }, [good-pod.yaml](../files/day5/good-pod.yaml){ download } | 실습 1 |
| [no-token.yaml](../files/day5/no-token.yaml){ download }, [pod-reader-role.yaml](../files/day5/pod-reader-role.yaml){ download } (선택) | 실습 2 |
| [deny-all.yaml](../files/day5/deny-all.yaml){ download }, [allow-backend-db.yaml](../files/day5/allow-backend-db.yaml){ download }, [allow-dns.yaml](../files/day5/allow-dns.yaml){ download } | 실습 3 |
| [miniblog-backend-blank.yaml](../files/day5/miniblog-backend-blank.yaml){ download }, [miniblog-frontend-blank.yaml](../files/day5/miniblog-frontend-blank.yaml){ download } | 실습 4 |
| [miniblog-backend-answer.yaml](../files/day5/miniblog-backend-answer.yaml){ download }, [miniblog-frontend-answer.yaml](../files/day5/miniblog-frontend-answer.yaml){ download } | 실습 4, 5 (정답) |
| [miniblog-backend-v2.yaml](../files/day5/miniblog-backend-v2.yaml){ download }, [miniblog-frontend-v2.yaml](../files/day5/miniblog-frontend-v2.yaml){ download } | 실습 5 (선택, 운영 보안 설정) |
| [broken-blog.yaml](../files/day5/broken-blog.yaml){ download } | 실습 6 |

## 정리

실습마다 끝에 정리 명령이 있습니다. 과정 전체를 정리하는 방법은 [실습 6의 과정 마무리](lab-6.md#wrap-up)에 있습니다.
