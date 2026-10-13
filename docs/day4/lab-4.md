# 실습 4. 메모리 한도 넘겨 OOMKilled 보기, requests가 크면 Pending

!!! abstract "이번 실습에서 할 것"
    - 메모리 limits를 넘으면 커널이 컨테이너를 죽이는 것을 직접 봅니다 (1일차 cgroups).
    - OOMKilled와 CrashLoopBackOff가 반복되는 것을 보고, describe에서 죽은 이유를 읽습니다.
    - requests가 노드보다 크면 Pod가 배치되지 않고 Pending에 머무는 것을 봅니다.

**소요 시간** 약 12분, **교안** DAY 4 섹션 4

## 준비

`day4` 폴더에서 합니다. 이번 실습에서 쓸 파일은 `memory-hog.yaml`입니다. Windows, Intel Mac, Apple Silicon Mac 모두 같은 파일을 씁니다.

```yaml title="memory-hog.yaml"
--8<-- "files/day4/memory-hog.yaml"
```
[memory-hog.yaml 내려받기](../files/day4/memory-hog.yaml){ download }

| 부분 | 뜻 |
|---|---|
| `limits: { memory: "100Mi" }` | 이 이상 쓰면 커널이 죽입니다. |
| `requests: { memory: "50Mi" }` | 스케줄러가 노드를 고를 때 쓰는 요청량입니다. 메모리를 미리 떼어 두는 값은 아닙니다. |
| `busybox:1.36` | amd64, arm64 모두 지원하는 작은 이미지입니다. |
| `tail /dev/zero` | 줄바꿈 없이 0만 나오는 입력을 끝없이 메모리에 쌓습니다. 반드시 100Mi를 넘습니다. |

??? note "참고: 공식 문서 예제(polinux/stress) 버전"
    kubernetes.io 문서 "Assign Memory Resources to Containers"는 아래처럼 `polinux/stress` 이미지를 씁니다. 이 이미지는 **amd64(인텔, AMD) 전용**이라 Apple Silicon Mac에서는 `exec format error` 등으로 실패할 수 있어, 수업에서는 위 busybox 파일을 씁니다. 강사 시연이나 참고용입니다.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: memory-hog
    spec:
      containers:
      - name: stress
        image: polinux/stress
        resources:
          requests: { memory: "50Mi" }
          limits: { memory: "100Mi" }
        command: ["stress"]
        args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
    ```

## STEP 19. 메모리 먹는 Pod 올리기

```bash
kubectl apply -f memory-hog.yaml
```

```text
pod/memory-hog created
```

## STEP 20. 상태 지켜보기 (Ctrl+C로 멈춤)

`-w`는 상태가 바뀔 때마다 한 줄씩 새로 보여 줍니다. 1분쯤 지켜본 뒤 `Ctrl + C`로 멈춥니다.

```bash
kubectl get pod memory-hog -w
```

```text
NAME         READY   STATUS              RESTARTS     AGE
memory-hog   0/1     ContainerCreating   0            1s
memory-hog   0/1     OOMKilled           0            5s
memory-hog   0/1     OOMKilled           1 (3s ago)   9s
memory-hog   0/1     CrashLoopBackOff    1 (2s ago)   10s
memory-hog   0/1     OOMKilled           2 (20s ago)  25s
memory-hog   0/1     CrashLoopBackOff    2 (2s ago)   26s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

OOMKilled와 CrashLoopBackOff가 번갈아 나오고 RESTARTS가 늘어납니다. 죽을 때마다 다시 살리기까지 기다리는 시간이 10초, 20초, 40초로 두 배씩 늘어납니다 (최대 5분).

!!! note "환경에 따라 다를 수 있음"
    Docker Desktop과 PC 속도에 따라 OOMKilled 줄이 너무 빨리 지나가 CrashLoopBackOff만 보이거나, 순서와 시간이 다를 수 있습니다. 어느 쪽이든 STEP 21의 describe에서 `Last State`의 Reason이 `OOMKilled`면 성공입니다.

## STEP 21. 죽은 이유 읽기

```bash
kubectl describe pod memory-hog
```

출력이 깁니다. 중간의 `Containers:` 부분에서 아래 줄들을 찾습니다.

```text
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
    Restart Count:  2
    Limits:
      memory:  100Mi
    Requests:
      memory:  50Mi
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

| 줄 | 뜻 |
|---|---|
| `State: Waiting`, `CrashLoopBackOff` | 지금 상태. 다시 살리기를 기다리는 중입니다 (결과). |
| `Last State: Terminated`, `OOMKilled` | 직전에 왜 죽었나 (원인). |
| `Exit Code: 137` | 128 + 9. SIGKILL로 강제 종료됐다는 뜻입니다. 다른 이유로도 나오니 137만 보고 OOM이라 단정하지 않고, 위 줄의 `Reason: OOMKilled`와 함께 봅니다. |
| `Limits: memory: 100Mi` | 이 상한을 넘어서 죽었습니다. |

CrashLoopBackOff는 원인이 아니라 결과입니다. 장애 때 `State`만 보고 멈추지 말고 `Last State`까지 내려갑니다.

!!! tip "직전 컨테이너의 로그 보기"
    죽은 이유를 찾을 때 함께 쓰는 명령입니다. 5일차 장애 찾기에서 다시 씁니다.

    ```bash
    kubectl logs memory-hog --previous
    ```

## STEP 22. 정리

```bash
kubectl delete pod memory-hog
```

```text
pod "memory-hog" deleted
```

## STEP 23. 노드보다 큰 requests로 Pod 올리기

이번에는 반대로, 메모리를 쓰지도 않았는데 **requests가 너무 커서** 배치되지 않는 Pod를 봅니다. 파일은 `big-request.yaml`입니다.

```yaml title="big-request.yaml"
--8<-- "files/day4/big-request.yaml"
```
[big-request.yaml 내려받기](../files/day4/big-request.yaml){ download }

노트북 한 대의 Docker Desktop에는 100Gi 메모리가 없습니다.

```bash
kubectl apply -f big-request.yaml
```

```bash
kubectl get pod big-request
```

```text
NAME          READY   STATUS    RESTARTS   AGE
big-request   0/1     Pending   0          10s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

시간이 지나도 `Pending`에서 바뀌지 않습니다.

## STEP 24. 왜 못 뜨는지 읽기

```bash
kubectl describe pod big-request
```

맨 아래 `Events:`를 봅니다.

```text
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  15s   default-scheduler  0/1 nodes are available: 1 Insufficient memory. ...
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

스케줄러가 노드 1개 중 메모리 요청량을 받아 줄 노드를 찾지 못했다는 뜻입니다. requests는 **스케줄링에 쓰는 요청량**이라, 실제 사용량과 관계없이 이 값만으로 배치가 막힙니다.

| 증상 | 원인 | 어디서 보나 |
|---|---|---|
| `OOMKilled` (STEP 20~21) | 실제 사용량이 **limits**를 넘어 커널이 죽임 | describe의 `Last State` |
| `Pending` (STEP 23~24) | **requests**를 받아 줄 노드가 없음 | describe의 `Events` |

## STEP 25. 정리

```bash
kubectl delete pod big-request
```

## 성공하면

- STEP 20: OOMKilled와 CrashLoopBackOff가 반복되고 RESTARTS가 늘어납니다.
- STEP 21: `Last State`의 `Reason: OOMKilled`, `Exit Code: 137`이 함께 보입니다.
- STEP 23~24: `big-request`가 `Pending`이고, Events에 `Insufficient memory`가 보입니다.

## 막히면

??? failure "Pod가 ErrImagePull / ImagePullBackOff"
    이미지를 받지 못했습니다. 이미지 이름 오타나 인터넷 문제입니다. `kubectl describe pod memory-hog` 맨 아래 Events를 봅니다. 해결이 안 되면 강사 화면으로 봅니다.

??? failure "Reason이 OOMKilled가 아니라 Error / Completed"
    OOM이 아닌 다른 이유로 끝났습니다. `kubectl logs memory-hog --previous`로 직전 로그를 보고, 파일의 `command` 줄이 `["sh", "-c", "tail /dev/zero"]`인지 확인합니다.

??? failure "big-request가 Pending이 아니라 Running"
    Docker Desktop에 100Gi보다 큰 메모리를 준 경우입니다 (거의 없습니다). 파일의 `100Gi`를 `1000Gi`로 바꿔 다시 올립니다.

??? failure "Error from server (AlreadyExists) / pod memory-hog unchanged"
    같은 이름의 Pod가 이미 있습니다. `kubectl delete pod memory-hog`로 지우고 다시 올립니다.

??? failure "-w가 끝나지 않음"
    `-w`는 계속 지켜보는 옵션입니다. `Ctrl + C`로 멈춥니다.

## 정리

STEP 22, STEP 25에서 지웠습니다. 남아 있으면 지웁니다.

```bash
kubectl delete pod memory-hog big-request --ignore-not-found
```
