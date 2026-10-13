# 실습 4. 메모리 한도 넘겨 OOMKilled 보기

!!! abstract "이번 실습에서 할 것"
    - 메모리 limits를 넘으면 커널이 컨테이너를 죽이는 것을 직접 봅니다 (1일차 cgroups).
    - OOMKilled와 CrashLoopBackOff가 반복되는 것을 보고, describe에서 죽은 이유를 읽습니다.

**소요 시간** 약 8분, **교안** DAY 4 섹션 4

## 준비

`day4` 폴더에서 합니다. 이번 실습에서 쓸 파일은 `memory-hog.yaml`입니다. kubernetes.io 공식 문서 "Assign Memory Resources to Containers"의 예제를 이름만 바꾼 것입니다.

```yaml title="memory-hog.yaml"
--8<-- "files/day4/memory-hog.yaml"
```
[memory-hog.yaml 내려받기](../files/day4/memory-hog.yaml){ download }

| 부분 | 뜻 |
|---|---|
| `limits: { memory: "100Mi" }` | 이 이상 쓰면 커널이 죽입니다. |
| `requests: { memory: "50Mi" }` | 스케줄러가 노드를 고를 때 보는 최소 보장 값입니다. |
| `--vm-bytes 250M` | `stress` 도구가 메모리 250MB를 잡으려 합니다. 반드시 100Mi를 넘습니다. |

!!! warning "Apple Silicon Mac"
    `polinux/stress` 이미지는 **amd64(인텔, AMD) 전용**입니다. Windows와 Intel Mac은 그대로 하면 됩니다. Apple Silicon(M1~M4) Mac에서는 Pod가 `exec format error`로 죽거나, Reason이 OOMKilled가 아닌 다른 이유로 끝날 수 있습니다.

    Apple Silicon Mac은 아래 `memory-hog-arm.yaml`을 씁니다. 여러 아키텍처를 지원하는 `busybox:1.36` 이미지에서 `tail /dev/zero`가 메모리를 계속 늘리다가 같은 100Mi 한도에 걸려 OOMKilled 됩니다. Pod 이름이 `memory-hog`로 같아서 STEP 17~19 명령은 그대로입니다.

    ```yaml title="memory-hog-arm.yaml"
    --8<-- "files/day4/memory-hog-arm.yaml"
    ```
    [memory-hog-arm.yaml 내려받기](../files/day4/memory-hog-arm.yaml){ download }

    STEP 16에서 파일 이름만 바꿔 올립니다.

    ```bash
    kubectl apply -f memory-hog-arm.yaml
    ```

    `polinux/stress`로 이미 올렸다면 먼저 `kubectl delete pod memory-hog`로 지우고 올립니다.

## STEP 16. 메모리 먹는 Pod 올리기

```bash
kubectl apply -f memory-hog.yaml
```

```text
pod/memory-hog created
```

## STEP 17. 상태 지켜보기 (Ctrl+C로 멈춤)

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
    Docker Desktop과 PC 속도에 따라 OOMKilled 줄이 너무 빨리 지나가 CrashLoopBackOff만 보이거나, 순서와 시간이 다를 수 있습니다. 어느 쪽이든 STEP 18의 describe에서 `Last State`의 Reason이 `OOMKilled`면 성공입니다.

## STEP 18. 죽은 이유 읽기

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
| `Exit Code: 137` | 128 + 9. SIGKILL로 강제 종료됐다는 뜻입니다. |
| `Limits: memory: 100Mi` | 이 상한을 넘어서 죽었습니다. |

CrashLoopBackOff는 원인이 아니라 결과입니다. 장애 때 `State`만 보고 멈추지 말고 `Last State`까지 내려갑니다.

!!! tip "직전 컨테이너의 로그 보기"
    죽은 이유를 찾을 때 함께 쓰는 명령입니다. 5일차 장애 찾기에서 다시 씁니다.

    ```bash
    kubectl logs memory-hog --previous
    ```

## STEP 19. 정리

```bash
kubectl delete pod memory-hog
```

```text
pod "memory-hog" deleted
```

## 성공하면

- STEP 17: OOMKilled와 CrashLoopBackOff가 반복되고 RESTARTS가 늘어납니다.
- STEP 18: `Last State`의 `Reason: OOMKilled`, `Exit Code: 137`이 보입니다.

## 막히면

??? failure "Pod가 ErrImagePull / ImagePullBackOff"
    이미지를 받지 못했습니다. 이미지 이름 오타나 인터넷 문제입니다. `kubectl describe pod memory-hog` 맨 아래 Events를 봅니다. 해결이 안 되면 강사 화면으로 봅니다.

??? failure "exec format error / no match for platform in manifest (Apple Silicon Mac)"
    `polinux/stress`가 amd64 전용이라 생기는 문제입니다. `kubectl delete pod memory-hog`로 지우고 위 **Apple Silicon Mac** 상자의 `memory-hog-arm.yaml`로 다시 올립니다.

??? failure "Reason이 OOMKilled가 아니라 Error"
    이미지 아키텍처가 맞지 않거나 명령이 잘못됐습니다. `kubectl logs memory-hog --previous`로 로그를 봅니다. Apple Silicon Mac이면 `memory-hog-arm.yaml`을 씁니다.

??? failure "Error from server (AlreadyExists) / pod memory-hog unchanged"
    같은 이름의 Pod가 이미 있습니다. `kubectl delete pod memory-hog`로 지우고 다시 올립니다.

??? failure "-w가 끝나지 않음"
    `-w`는 계속 지켜보는 옵션입니다. `Ctrl + C`로 멈춥니다.

## 정리

STEP 19에서 지웠습니다. 남아 있으면 지웁니다.

```bash
kubectl delete pod memory-hog --ignore-not-found
```
