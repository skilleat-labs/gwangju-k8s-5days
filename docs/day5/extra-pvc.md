# 추가 실습. PVC 더 다뤄 보기

!!! abstract "이번 실습에서 할 것"
    - PVC 하나를 만들면 PV가 저절로 생기는 것(동적 프로비저닝)을 봅니다.
    - Pod를 지워도, 다른 Pod로 바꿔 붙여도 데이터가 남는 것을 확인합니다.
    - ReadWriteOnce가 "Pod 하나"가 아니라 "노드 하나"라는 것을 직접 봅니다.
    - 쓰고 있는 PVC는 바로 지워지지 않고, PVC를 지우면 데이터도 사라지는 것을 봅니다.

**소요 시간** 약 20분, **교안** 4일차 섹션 6, 5일차 종합 배포의 연장

!!! tip "YAML 파일 없이 따라 합니다"
    이 실습은 파일을 만들지 않습니다. 명령 블록을 **통째로 복사해서 붙여 넣으면** YAML이 바로 적용됩니다. Windows는 PowerShell의 `@' ... '@` 문법을, Mac은 `<<'EOF'` 문법을 씁니다. 블록 첫 줄부터 마지막 줄까지 한 번에 복사하세요.

## STEP 1. 준비: 실습 공간과 StorageClass 확인

실습용 네임스페이스를 따로 만듭니다. 다른 실습과 섞이지 않고, 마지막에 한 번에 지울 수 있습니다.

```bash
kubectl create namespace pvc-lab
```

Docker Desktop이 준비해 둔 StorageClass를 봅니다.

```bash
kubectl get storageclass
```

```text
NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
hostpath (default)   docker.io/hostpath   Delete          Immediate           false                  5d
```
<small>출력은 예시입니다. Docker Desktop 버전에 따라 다를 수 있습니다.</small>

| 칸 | 뜻 |
|---|---|
| `(default)` | PVC에 storageClassName을 안 적으면 이것을 씁니다 |
| `RECLAIMPOLICY  Delete` | PVC를 지우면 PV와 데이터도 같이 지웁니다 (STEP 6에서 확인) |
| `VOLUMEBINDINGMODE` | `Immediate`면 PVC를 만들자마자 PV가 생기고, `WaitForFirstConsumer`면 Pod가 붙을 때 생깁니다 |

## STEP 2. PVC를 만들면 PV가 저절로 생긴다

PVC는 "디스크 1Gi 주세요"라는 요청서입니다.

=== "Windows (PowerShell)"

    ```powershell
    @'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: lab-pvc
      namespace: pvc-lab
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
    '@ | kubectl apply -f -
    ```

=== "Mac (터미널)"

    ```bash
    cat <<'EOF' | kubectl apply -f -
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: lab-pvc
      namespace: pvc-lab
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
    EOF
    ```

```bash
kubectl get pvc -n pvc-lab
```

```bash
kubectl get pv
```

```text
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
lab-pvc   Bound    pvc-3f2a9c1e-...                           1Gi        RWO            hostpath

NAME               CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS
pvc-3f2a9c1e-...   1Gi        RWO            Delete           Bound    pvc-lab/lab-pvc   hostpath
```
<small>출력은 예시입니다. 이름, 시간은 사람마다 다릅니다.</small>

**PV는 우리가 만든 적이 없는데 생겼습니다.** PVC를 보고 StorageClass(hostpath)가 PV를 만들어 묶어 준 것입니다. 이것이 동적 프로비저닝입니다.

!!! note "환경에 따라 다를 수 있음"
    PVC가 `Pending`이면 `VOLUMEBINDINGMODE`가 `WaitForFirstConsumer`인 경우입니다. 정상입니다. STEP 3에서 Pod가 붙으면 `Bound`로 바뀝니다.

## STEP 3. 데이터를 쓰는 Pod

5초마다 현재 시각을 `/data/log.txt`에 한 줄씩 쓰는 Pod입니다. `/data`가 PVC입니다.

=== "Windows (PowerShell)"

    ```powershell
    @'
    apiVersion: v1
    kind: Pod
    metadata:
      name: writer
      namespace: pvc-lab
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "while true; do echo writer $(date) >> /data/log.txt; sleep 5; done"]
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: lab-pvc
    '@ | kubectl apply -f -
    ```

    `@'` 와 `'@` 사이는 PowerShell이 글자 그대로 넘깁니다. 그래서 `$(date)`도 PowerShell이 아니라 컨테이너 안에서 계산됩니다.

=== "Mac (터미널)"

    ```bash
    cat <<'EOF' | kubectl apply -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: writer
      namespace: pvc-lab
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "while true; do echo writer $(date) >> /data/log.txt; sleep 5; done"]
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: lab-pvc
    EOF
    ```

준비될 때까지 기다린 뒤 15초쯤 지나서 파일을 봅니다.

```bash
kubectl wait --for=condition=Ready pod/writer -n pvc-lab --timeout=120s
```

```bash
kubectl exec -n pvc-lab writer -- cat /data/log.txt
```

```text
writer Tue Oct 21 10:00:01 UTC 2026
writer Tue Oct 21 10:00:06 UTC 2026
writer Tue Oct 21 10:00:11 UTC 2026
```
<small>출력은 예시입니다. 시각은 사람마다 다릅니다.</small>

## STEP 4. Pod를 지우고 다른 Pod로 붙여도 데이터는 남는다

writer를 지웁니다. PVC는 그대로입니다.

```bash
kubectl delete pod writer -n pvc-lab
```

```bash
kubectl get pvc -n pvc-lab
```

`lab-pvc`가 여전히 `Bound`입니다. 이제 **전혀 다른 Pod**를 같은 PVC에 붙여 읽어 봅니다.

=== "Windows (PowerShell)"

    ```powershell
    @'
    apiVersion: v1
    kind: Pod
    metadata:
      name: reader
      namespace: pvc-lab
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: lab-pvc
    '@ | kubectl apply -f -
    ```

=== "Mac (터미널)"

    ```bash
    cat <<'EOF' | kubectl apply -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: reader
      namespace: pvc-lab
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: lab-pvc
    EOF
    ```

```bash
kubectl wait --for=condition=Ready pod/reader -n pvc-lab --timeout=120s
```

```bash
kubectl exec -n pvc-lab reader -- cat /data/log.txt
```

**writer가 쓴 줄이 그대로 있습니다.** 데이터는 Pod가 아니라 PVC(뒤의 PV)에 있기 때문입니다. 1일차에 컨테이너를 지우면 파일이 사라졌던 것과 비교해 보세요.

## STEP 5. ReadWriteOnce는 "노드 하나"다

reader를 둔 채로 writer를 다시 띄웁니다. 같은 RWO PVC에 Pod 두 개가 동시에 붙습니다.

STEP 3의 writer 명령 블록을 **그대로 다시** 붙여 넣습니다. 그다음:

```bash
kubectl get pods -n pvc-lab -o wide
```

```text
NAME     READY   STATUS    RESTARTS   AGE   IP          NODE
reader   1/1     Running   0          2m    10.1.0.40   docker-desktop
writer   1/1     Running   0          10s   10.1.0.41   docker-desktop
```
<small>출력은 예시입니다.</small>

**둘 다 Running입니다.** 이번에는 reader 쪽에서 writer가 방금 쓴 줄을 봅니다.

```bash
kubectl exec -n pvc-lab reader -- tail -n 3 /data/log.txt
```

몇 초 뒤 다시 실행하면 줄이 늘어나 있습니다. 두 Pod가 같은 디스크를 함께 보고 있습니다.

!!! info "왜 둘 다 붙을까"
    `ReadWriteOnce`는 "Pod 하나만"이 아니라 **"노드 하나에만 붙는다"** 는 뜻입니다. Docker Desktop은 노드가 1대라서 두 Pod가 같은 노드에 있고, 그래서 둘 다 붙습니다. 노드가 여러 대인 클러스터라면 다른 노드에 놓인 Pod는 붙지 못하고 `ContainerCreating`에 머뭅니다. 정말 Pod 하나로 제한하려면 `ReadWriteOncePod`를 씁니다.

    그래서 5일차 미니 블로그 backend는 `replicas: 1`과 `strategy: Recreate`로, 같은 파일에 두 Pod가 동시에 쓰지 않게 했습니다.

## STEP 6. 쓰고 있는 PVC는 바로 지워지지 않는다

Pod가 쓰고 있는 PVC를 지워 봅니다. 명령이 끝나지 않고 기다리면 `Ctrl+C`로 빠져나옵니다.

```bash
kubectl delete pvc lab-pvc -n pvc-lab --wait=false
```

```bash
kubectl get pvc -n pvc-lab
```

```text
NAME      STATUS        VOLUME             CAPACITY   ACCESS MODES   STORAGECLASS
lab-pvc   Terminating   pvc-3f2a9c1e-...   1Gi        RWO            hostpath
```
<small>출력은 예시입니다.</small>

`Terminating`에서 멈춰 있습니다. 쓰고 있는 Pod가 있으면 쿠버네티스가 PVC 삭제를 미룹니다(PVC 보호). 이유를 확인합니다.

=== "Windows (PowerShell)"

    ```powershell
    kubectl describe pvc lab-pvc -n pvc-lab | Select-String "Finalizers|Used By" -Context 0,2
    ```

=== "Mac (터미널)"

    ```bash
    kubectl describe pvc lab-pvc -n pvc-lab | grep -A2 -E "Finalizers|Used By"
    ```

```text
Finalizers:    [kubernetes.io/pvc-protection]
Used By:       reader
               writer
```
<small>출력은 예시입니다.</small>

Pod 두 개를 지우면 PVC가 마저 지워집니다.

```bash
kubectl delete pod writer reader -n pvc-lab
```

```bash
kubectl get pvc -n pvc-lab
```

```bash
kubectl get pv
```

**PVC도, PV도 사라졌습니다.** STEP 1에서 본 `RECLAIMPOLICY Delete` 때문입니다. PVC를 지우면 데이터도 함께 지워집니다.

!!! warning "PVC는 백업이 아니다"
    PVC는 Pod가 바뀌어도 데이터를 남겨 주지만, PVC를 지우거나 노드의 디스크가 망가지면 데이터도 사라집니다. 운영에서는 백업을 따로 하고, 지우면 안 되는 데이터는 `reclaimPolicy: Retain`인 StorageClass를 씁니다. Docker Desktop의 hostpath는 노드(내 PC 안의 VM) 디스크에 저장하므로 다른 노드로 옮겨 가지도 않습니다.

## 성공하면

| STEP | 확인한 것 |
|---|---|
| 2 | PVC만 만들었는데 PV가 저절로 생겼다 (동적 프로비저닝) |
| 4 | writer를 지우고 reader로 붙여도 log.txt가 남아 있다 |
| 5 | RWO PVC에 두 Pod가 동시에 붙었다 (노드가 1대라서) |
| 6 | 쓰는 중인 PVC는 Terminating에서 기다리고, Pod를 지우자 PVC와 PV가 함께 사라졌다 |

## 막히면

??? failure "PowerShell에서 error: error parsing STDIN / did not find expected key"
    블록을 일부만 복사했거나, `@'` 다음 줄부터 `'@` 앞 줄까지의 들여쓰기가 깨졌습니다. 명령 블록 오른쪽 위의 **복사 버튼**으로 통째로 복사해 다시 붙여 넣습니다. `'@`는 반드시 줄 맨 앞에 있어야 합니다.

??? failure "Mac에서 > 표시만 나오고 끝나지 않음"
    마지막 줄 `EOF`까지 복사되지 않았습니다. `EOF`를 입력하고 Enter를 누르거나, `Ctrl+C`로 빠져나와 블록 전체를 다시 복사합니다.

??? failure "PVC가 계속 Pending"
    `kubectl describe pvc lab-pvc -n pvc-lab`의 Events를 봅니다. StorageClass가 없다는 메시지면 `kubectl get storageclass`에 `(default)`가 있는지 확인합니다. Docker Desktop의 Kubernetes가 켜져 있는지도 봅니다.

## 정리

실습 공간을 통째로 지웁니다. 남아 있는 PVC, Pod가 함께 지워집니다.

```bash
kubectl delete namespace pvc-lab
```
