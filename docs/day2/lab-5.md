# 실습 5. Deployment와 자가복구

!!! abstract "이번 실습에서 할 것"
    - Deployment로 Pod 3개를 띄우고, 하나를 지워도 다시 생기는 것(자가복구)을 봅니다.
    - 포트 충돌 없이 숫자 하나로 Pod를 늘리고 줄입니다(스케일링).

**소요 시간** 약 12분 (STEP 1~3 7분, STEP 4~6 5분), **교안** DAY 2 섹션 5

## 준비

작업 폴더로 이동합니다.

=== "Windows (PowerShell)"

    ```powershell
    cd $HOME\k8s-labs\day2
    ```

=== "Mac (터미널)"

    ```bash
    cd ~/k8s-labs/day2
    ```

VS Code에서 새 파일 `deployment.yaml`을 만들고 아래 내용을 붙여 넣어 저장합니다. 또는 내려받아 작업 폴더에 넣습니다.

```yaml title="deployment.yaml"
--8<-- "files/day2/deployment.yaml"
```
[deployment.yaml 내려받기](../files/day2/deployment.yaml){ download }

| 필드 | 뜻 |
|---|---|
| `apiVersion: apps/v1`, `kind: Deployment` | Pod(`v1`)와 달리 `apps/v1`입니다 |
| `replicas: 3` | Pod를 몇 개 유지할지 |
| `selector.matchLabels` | 어떤 라벨의 Pod가 내 것인지 |
| `template` | 새 Pod를 찍어낼 틀. 그 아래는 Pod YAML과 같습니다 |

`selector`의 `app: web`과 `template.metadata.labels`의 `app: web`은 **반드시 같아야 합니다.**

!!! tip "다른 터미널로 지켜보기"
    터미널을 하나 더 열고 아래 명령을 켜 두면, Pod가 지워지고 생기는 순간을 실시간으로 볼 수 있습니다. 다 보면 ++ctrl+c++로 끝냅니다.

    ```bash
    kubectl get pods -w
    ```

## STEP 1. Deployment 만들기

```bash
kubectl apply -f deployment.yaml
```

```text
deployment.apps/web created
```

Deployment, ReplicaSet, Pod를 한 번에 봅니다.

```bash
kubectl get deploy,rs,pods
```

```text
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/web   3/3     3            3           20s

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/web-7c5b8d9f6d   3         3         3       20s

NAME                       READY   STATUS    RESTARTS   AGE
pod/web-7c5b8d9f6d-4xk2p   1/1     Running   0          20s
pod/web-7c5b8d9f6d-9qz7m   1/1     Running   0          20s
pod/web-7c5b8d9f6d-wq8rt   1/1     Running   0          20s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

Deployment 1개, ReplicaSet 1개, Pod 3개입니다. 이름이 `web` → `web-7c5b8d9f6d` → `web-7c5b8d9f6d-4xk2p`처럼 이어집니다. Deployment가 ReplicaSet을, ReplicaSet이 Pod를 만들었습니다.

## STEP 2. Pod 하나를 지운다

1일차에 `docker kill`로 컨테이너를 죽였을 때는 아무도 살리지 않았습니다. 오늘은 Pod 하나를 지워 봅니다. STEP 1에서 본 Pod 이름 하나를 복사해 넣습니다.

```bash
kubectl delete pod <web-로 시작하는 Pod 이름>
```

예를 들면 `kubectl delete pod web-7c5b8d9f6d-4xk2p`입니다. 이름은 Tab 키 자동완성이 안 되면 복사해서 붙입니다. 바로 목록을 봅니다.

```bash
kubectl get pods
```

```text
NAME                   READY   STATUS    RESTARTS   AGE
web-7c5b8d9f6d-9qz7m   1/1     Running   0          2m
web-7c5b8d9f6d-h2lwn   1/1     Running   0          3s
web-7c5b8d9f6d-wq8rt   1/1     Running   0          2m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

AGE가 몇 초짜리인 **새 이름의 Pod**가 생겨 다시 3개입니다. 아무도 손대지 않았는데 돌아왔습니다. 1일차 도커의 한계 2번이 풀렸습니다.

## STEP 3. 기록 보기

ReplicaSet의 Events에 누가 Pod를 만들었는지 기록이 남습니다.

```bash
kubectl describe rs
```

```text
...
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  3m    replicaset-controller  Created pod: web-7c5b8d9f6d-4xk2p
  Normal  SuccessfulCreate  3m    replicaset-controller  Created pod: web-7c5b8d9f6d-9qz7m
  Normal  SuccessfulCreate  3m    replicaset-controller  Created pod: web-7c5b8d9f6d-wq8rt
  Normal  SuccessfulCreate  40s   replicaset-controller  Created pod: web-7c5b8d9f6d-h2lwn
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

맨 아래 줄이 STEP 2에서 지운 자리를 채운 새 Pod입니다. ReplicaSet이 "원하는 3, 지금 2"를 보고 하나를 더 만들었습니다.

## STEP 4. 5개로

1일차에는 frontend를 하나 더 띄우려다 `port is already allocated`가 나서 8081, 8082로 포트를 바꿔 하나씩 실행했습니다. 오늘은 숫자만 바꿉니다.

```bash
kubectl scale deployment web --replicas=5
```

```text
deployment.apps/web scaled
```

```bash
kubectl get pods
```

```text
NAME                   READY   STATUS    RESTARTS   AGE
web-7c5b8d9f6d-9qz7m   1/1     Running   0          5m
web-7c5b8d9f6d-h2lwn   1/1     Running   0          2m
web-7c5b8d9f6d-kx5vd   1/1     Running   0          4s
web-7c5b8d9f6d-p7bnc   1/1     Running   0          4s
web-7c5b8d9f6d-wq8rt   1/1     Running   0          5m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

포트를 신경 쓰지 않았습니다. Pod마다 자기 IP를 따로 가지기 때문입니다. 1일차 도커의 한계 3번이 풀렸습니다.

## STEP 5. 2개로

```bash
kubectl scale deployment web --replicas=2
```

```bash
kubectl get pods
```

```text
NAME                   READY   STATUS        RESTARTS   AGE
web-7c5b8d9f6d-9qz7m   1/1     Running       0          6m
web-7c5b8d9f6d-h2lwn   1/1     Terminating   0          3m
web-7c5b8d9f6d-kx5vd   1/1     Terminating   0          1m
web-7c5b8d9f6d-p7bnc   1/1     Terminating   0          1m
web-7c5b8d9f6d-wq8rt   1/1     Running       0          6m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

3개가 `Terminating`이 되며 사라집니다. 몇 초 뒤 다시 보면 2개만 남습니다.

## STEP 6. 다시 3개로 (선언형: 파일 기준)

`kubectl scale`로 바꾼 숫자는 `deployment.yaml` 파일에 남지 않습니다. 파일을 다시 apply하면 파일에 적힌 `replicas: 3`으로 돌아갑니다.

```bash
kubectl apply -f deployment.yaml
```

```text
deployment.apps/web configured
```

```bash
kubectl get pods
```

```text
NAME                   READY   STATUS    RESTARTS   AGE
web-7c5b8d9f6d-9qz7m   1/1     Running   0          7m
web-7c5b8d9f6d-t4mzc   1/1     Running   0          3s
web-7c5b8d9f6d-wq8rt   1/1     Running   0          7m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

운영에서는 `kubectl scale` 대신 YAML의 `replicas` 값을 고쳐 apply하는 방식을 권합니다. 파일이 곧 기록이 됩니다.

## 성공하면

- STEP 1: Deployment 1개, ReplicaSet 1개, Pod 3개
- STEP 2: 새 이름 Pod가 생겨 다시 3개
- STEP 3: `Created pod` 기록
- STEP 4: Pod 5개 Running
- STEP 5: 3개가 Terminating
- STEP 6: 파일대로 3개

## 막히면

??? failure "The Deployment \"web\" is invalid: spec.template.metadata.labels: Invalid value: ... selector does not match template labels"
    `selector.matchLabels`와 `template.metadata.labels`의 라벨이 다릅니다. 두 곳 모두 `app: web`으로 맞춥니다. 위 [deployment.yaml](../files/day2/deployment.yaml)과 한 줄씩 비교합니다.

??? failure "spec.selector: Invalid value: ... field is immutable"
    이미 만든 Deployment의 `selector`를 바꿔서 apply했습니다. selector는 만든 뒤에 바꿀 수 없습니다. `kubectl delete deployment web`으로 지운 뒤 다시 apply합니다.

??? failure "no matches for kind \"Deployment\" in version \"v1\""
    `apiVersion: v1`으로 썼습니다. Deployment는 `apiVersion: apps/v1`입니다.

??? failure "error converting YAML to JSON: yaml: line N: found character that cannot start any token"
    들여쓰기에 탭이 들어갔습니다. 오류에 나온 줄 번호의 들여쓰기를 공백 2칸으로 바꿉니다. VS Code 오른쪽 아래 상태 표시줄이 `Spaces: 2`인지 확인합니다.

??? failure "error: the path \"deployment.yaml\" does not exist"
    작업 폴더가 아니거나 파일 이름이 다릅니다. Windows 메모장으로 저장하면 `deployment.yaml.txt`가 될 수 있습니다. VS Code로 만들거나, 탐색기에서 **보기 → 파일 확장명**을 켜고 이름을 고칩니다.

??? failure "Error from server (NotFound): deployments.apps \"web-7c5b8d9f6d-9qz7m\" not found"
    `kubectl scale`에 Pod 이름을 넣었습니다. scale 대상은 Deployment 이름 `web`입니다: `kubectl scale deployment web --replicas=5`

??? failure "STEP 2에서 Pod 이름을 넣었는데 Error from server (NotFound): pods \"...\" not found"
    이름을 잘못 옮겨 적었거나 꺾쇠(`< >`)까지 같이 넣었습니다. `kubectl get pods`로 이름을 다시 보고 꺾쇠 없이 이름만 넣습니다.

## 정리

`web` Deployment와 `deployment.yaml`은 **실습 6에서 계속 씁니다. 지우지 마세요.**
