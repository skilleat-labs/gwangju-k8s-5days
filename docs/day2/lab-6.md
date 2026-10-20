# 실습 6. 롤링 업데이트와 롤백

!!! abstract "이번 실습에서 할 것"
    - nginx 1.27을 1.28로 Pod 하나씩 바꾸고(롤링 업데이트), 남는 이력을 봅니다.
    - 명령으로 바꾼 상태와 YAML 파일이 어긋나는 것을 알아 둡니다.
    - 명령 한 줄로 옛 버전으로 되돌립니다(롤백).

**소요 시간** 약 12분 (STEP 1~3 7분, STEP 4~6 5분), **교안** DAY 2 섹션 6

## 준비

[실습 5](lab-5.md)에서 만든 `web` Deployment가 Pod 3개로 떠 있어야 합니다.

=== "Windows (PowerShell)"

    ```powershell
    cd $HOME\k8s-labs\day2
    ```

=== "Mac (터미널)"

    ```bash
    cd ~/k8s-labs/day2
    ```

```bash
kubectl get deploy web
```

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    3/3     3            3           10m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`NotFound`가 나오면 실습 5의 [deployment.yaml](../files/day2/deployment.yaml)로 `kubectl apply -f deployment.yaml`을 먼저 실행합니다.

!!! tip "다른 터미널로 지켜보기"
    터미널을 하나 더 열고 `kubectl get pods -w`를 켜 두면, 새 Pod가 하나 뜨고 옛 Pod가 하나 내려가는 롤링 업데이트 과정이 그대로 보입니다. 새 Pod의 READY가 `1/1`이 된 것을 확인한 뒤 ++ctrl+c++로 끝냅니다.

## STEP 1. 이미지 교체

`set image`로 이미지를 바꿉니다. 등호 왼쪽 `nginx`는 YAML의 **컨테이너 이름**(`containers.name`), 오른쪽이 새 이미지입니다.

```bash
kubectl set image deployment/web nginx=nginx:1.28
```

```text
deployment.apps/web image updated
```

교체가 끝날 때까지 진행 상황을 봅니다.

```bash
kubectl rollout status deployment/web
```

```text
Waiting for deployment "web" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "web" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "web" rollout to finish: 1 old replicas are pending termination...
deployment "web" successfully rolled out
```
<small>출력은 예시입니다. 이미지를 받는 시간에 따라 줄 수가 다릅니다.</small>

## STEP 2. ReplicaSet이 2개가 됐다

```bash
kubectl get rs
```

```text
NAME             DESIRED   CURRENT   READY   AGE
web-5d8f7c6b9d   3         3         3       40s
web-7c5b8d9f6d   0         0         0       12m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

새 ReplicaSet(1.28)이 3개, 옛 ReplicaSet(1.27)이 0개입니다. 옛 것을 지우지 않고 0개로 남겨 두는 이유가 STEP 4의 롤백입니다.

!!! info "오늘 확인하는 것은 순차 교체 원리입니다"
    오늘은 Pod가 한꺼번에 내려가지 않고 하나씩 교체되는 것을 확인합니다. 사용자 접속이 실제로 끊기지 않는지(무중단)는 요청을 준비된 Pod로만 보내는 Service(3일차)와 준비 상태를 판단하는 readinessProbe(4일차)를 배운 뒤에 확인합니다.

## STEP 3. 바꾼 이유를 기록하고 이력 보기

`change-cause` 주석을 달면 이력에 바꾼 이유가 남습니다.

```bash
kubectl annotate deployment/web kubernetes.io/change-cause="nginx 1.28 업데이트"
```

```text
deployment.apps/web annotated
```

```bash
kubectl rollout history deployment/web
```

```text
deployment.apps/web
REVISION  CHANGE-CAUSE
1         <none>
2         nginx 1.28 업데이트
```
<small>출력은 예시입니다.</small>

리비전 1은 주석을 달기 전이라 `<none>`입니다.

!!! note "환경에 따라 다를 수 있음"
    Windows PowerShell에서 CHANGE-CAUSE의 한글이 `??`나 깨진 글자로 보일 수 있습니다. 터미널 글꼴, 코드 페이지 문제라 실습에는 지장이 없습니다. 신경 쓰이면 영어로 다시 답니다. 같은 키에 값을 바꿀 때는 `--overwrite`가 필요합니다.

    ```bash
    kubectl annotate deployment/web kubernetes.io/change-cause="nginx 1.28 update" --overwrite
    ```

!!! warning "명령으로 바꾼 상태와 YAML 파일"
    `set image`로 클러스터는 1.28이 됐지만 `deployment.yaml` 파일에는 아직 `nginx:1.27`이 적혀 있습니다. 이 상태에서 `kubectl apply -f deployment.yaml`을 하면 파일 값으로 맞춰져 이미지가 1.27로 되돌아갑니다. 실습 5 STEP 6에서 `replicas`가 파일대로 3으로 돌아간 것과 같은 현상입니다. 운영에서는 파일의 `image`를 고쳐 apply하고, 급히 명령으로 바꿨다면 파일에도 반영합니다. 오늘은 STEP 4의 롤백으로 1.27로 돌아가므로 파일과 클러스터가 다시 같아집니다.

## STEP 4. 문제가 생겼다고 치고 되돌린다

`rollout undo`는 바로 직전 리비전으로 되돌립니다. 이것도 Pod를 하나씩 바꿔 끼우는 방식입니다.

```bash
kubectl rollout undo deployment/web
```

```text
deployment.apps/web rolled back
```

```bash
kubectl rollout status deployment/web
```

```text
deployment "web" successfully rolled out
```

## STEP 5. 정말 1.27로 돌아왔나

=== "Windows (PowerShell)"

    ```powershell
    kubectl describe deployment web | Select-String Image
    ```

=== "Mac (터미널)"

    ```bash
    kubectl describe deployment web | grep Image
    ```

```text
    Image:         nginx:1.27
```

이력을 다시 봅니다.

```bash
kubectl rollout history deployment/web
```

```text
deployment.apps/web
REVISION  CHANGE-CAUSE
2         nginx 1.28 업데이트
3         <none>
```
<small>출력은 예시입니다.</small>

리비전 1이 사라지고 2와 3이 남았습니다. 리비전 1의 내용(1.27)이 3번으로 다시 올라왔기 때문입니다. 특정 리비전으로 가려면 `kubectl rollout undo deployment/web --to-revision=2`처럼 번호를 줍니다.

## STEP 6. 남은 것 확인

`web` Deployment는 **지우지 않습니다.** 3일차 첫 실습에서 이 Deployment에 Service를 붙이기 때문입니다. 남은 것만 확인합니다.

```bash
kubectl get all
```

```text
NAME                       READY   STATUS    RESTARTS   AGE
pod/web-7c5b8d9f6d-4fz8k   1/1     Running   0          1m
pod/web-7c5b8d9f6d-b9xqp   1/1     Running   0          1m
pod/web-7c5b8d9f6d-n6t2c   1/1     Running   0          1m

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   1d

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/web   3/3     3            3           15m

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/web-5d8f7c6b9d   0         0         0       4m
replicaset.apps/web-7c5b8d9f6d   3         3         3       15m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`web` Deployment가 nginx 1.27, Pod 3개로 남아 있으면 오늘 실습은 끝입니다.

!!! info "지우면 어떻게 되나"
    `kubectl delete -f deployment.yaml`을 실행하면 Deployment, ReplicaSet, Pod가 한꺼번에 지워집니다. Deployment가 ReplicaSet을, ReplicaSet이 Pod를 소유하기 때문입니다. 실수로 지웠으면 `kubectl apply -f deployment.yaml`로 다시 만들면 됩니다.

## 성공하면

- STEP 1: `successfully rolled out`
- STEP 2: 새 rs 3, 옛 rs 0
- STEP 3: REVISION 1, 2
- STEP 5: `Image: nginx:1.27`, 이력은 REVISION 2, 3

## 막히면

??? failure "error: unable to find container named \"web\""
    `kubectl set image deployment/web web=nginx:1.28`처럼 등호 왼쪽에 Deployment 이름을 썼습니다. 등호 왼쪽은 **컨테이너 이름** `nginx`입니다. `kubectl set image deployment/web nginx=nginx:1.28`

??? failure "rollout status가 Waiting ... 에서 끝나지 않음"
    새 이미지를 받지 못하고 있을 가능성이 큽니다. ++ctrl+c++로 빠져나와 `kubectl get pods`를 봅니다. 새 Pod가 `ImagePullBackOff`면 이미지 태그 오타입니다. 바로 STEP 4의 `kubectl rollout undo deployment/web`으로 되돌린 뒤 올바른 태그로 다시 합니다. 롤링 업데이트라서 그동안에도 옛 Pod는 Running으로 남아 있습니다.

??? failure "grep : The term 'grep' is not recognized as the name of a cmdlet, function, script file, or operable program."
    Windows PowerShell에는 grep이 없습니다. Windows 탭의 `Select-String`을 씁니다. `findstr Image`도 됩니다.

??? failure "error: --overwrite is false but found the following declared annotation(s): 'kubernetes.io/change-cause' already has a value"
    같은 주석을 두 번 달았습니다. 값을 바꾸려면 명령 끝에 `--overwrite`를 붙입니다.

??? failure "error: no rollout history found for deployment \"web\""
    Deployment 이름 오타이거나 아직 만들지 않았습니다. `kubectl get deploy`로 이름을 확인합니다.

## 정리

**`web` Deployment와 `deployment.yaml` 파일은 3일차에 씁니다. 지우지 마세요.** 하루를 마칠 때 확인할 것은 [오늘의 실습](index.md) 맨 아래 "하루를 마칠 때 정리"를 봅니다.
