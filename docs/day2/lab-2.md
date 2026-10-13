# 실습 2. 클러스터 구성요소 찾기

!!! abstract "이번 실습에서 할 것"
    - 강의 그림에서 본 컨트롤플레인 부품(etcd, apiserver, scheduler, controller-manager)을 내 클러스터에서 찾습니다.
    - 노드가 어떤 컨테이너 런타임을 쓰는지 확인합니다.

**소요 시간** 약 10분, **교안** DAY 2 섹션 2

## 준비

[실습 1](lab-1.md)에서 `docker-desktop` 노드가 `Ready`인 것을 확인했으면 됩니다. 작업 폴더에 있는지 확인합니다.

=== "Windows (PowerShell)"

    ```powershell
    cd $HOME\k8s-labs\day2
    ```

=== "Mac (터미널)"

    ```bash
    cd ~/k8s-labs/day2
    ```

## STEP 1. 두뇌 부품은 kube-system 네임스페이스의 Pod

쿠버네티스를 움직이는 부품은 `kube-system`이라는 네임스페이스에 Pod로 떠 있습니다.

```bash
kubectl get pods -n kube-system
```

```text
NAME                                     READY   STATUS    RESTARTS   AGE
coredns-xxxxxxxxxx-aaaaa                 1/1     Running   0          1d
coredns-xxxxxxxxxx-bbbbb                 1/1     Running   0          1d
etcd-docker-desktop                      1/1     Running   0          1d
kube-apiserver-docker-desktop            1/1     Running   0          1d
kube-controller-manager-docker-desktop   1/1     Running   0          1d
kube-proxy-ccccc                         1/1     Running   0          1d
kube-scheduler-docker-desktop            1/1     Running   0          1d
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

| 이름 | 하는 일 |
|---|---|
| `etcd-docker-desktop` | 클러스터의 모든 상태를 저장하는 데이터베이스 |
| `kube-apiserver-docker-desktop` | 모든 요청이 들어오는 입구. kubectl은 여기에만 말합니다 |
| `kube-scheduler-docker-desktop` | 새 Pod를 어느 노드에 둘지 고릅니다 |
| `kube-controller-manager-docker-desktop` | 원하는 상태와 지금 상태를 맞추는 컨트롤러 묶음 |
| `kube-proxy-...` | 노드마다 하나씩 뜨는 일꾼 부품. 네트워크 규칙을 만듭니다 |
| `coredns-...` | 이름을 IP로 바꾸는 DNS. 3일차에 다시 봅니다 |

이름 끝에 `-docker-desktop`이 붙은 네 개가 컨트롤플레인(두뇌) 부품입니다. 노드의 설정 파일을 보고 kubelet이 직접 띄우는 Pod(static Pod)라서 이름에 노드 이름이 붙습니다.

!!! note "환경에 따라 다를 수 있음"
    Docker Desktop 버전에 따라 `storage-provisioner`, `vpnkit-controller` 같은 Docker Desktop 전용 Pod가 더 보이거나 안 보일 수 있습니다. 위 표의 여섯 종류만 찾으면 됩니다.

## STEP 2. 노드 정보와 런타임

`-o wide`를 붙이면 칸이 더 나옵니다. 맨 오른쪽 `CONTAINER-RUNTIME` 칸을 봅니다.

```bash
kubectl get nodes -o wide
```

```text
NAME             STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION                       CONTAINER-RUNTIME
docker-desktop   Ready    control-plane   1d    v1.3x.x   192.168.65.3   <none>        Docker Desktop   6.10.x-linuxkit                      containerd://1.7.x
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`containerd://`가 보이면 1일차에 배운 컨테이너 런타임 containerd가 이 노드에서 컨테이너를 실행하고 있다는 뜻입니다.

!!! note "환경에 따라 다를 수 있음"
    Docker Desktop 버전과 설정에 따라 `CONTAINER-RUNTIME` 칸이 `docker://2x.x.x`로 나올 수 있습니다. 이 경우 Docker Engine이 런타임 역할을 하는 것입니다. 실습에는 차이가 없습니다.

## STEP 3. 노드 한 대가 무엇을 들고 있나

노드를 자세히 봅니다. 출력이 길지만 **Non-terminated Pods** 부분만 봅니다.

```bash
kubectl describe node docker-desktop
```

```text
...
Non-terminated Pods:          (9 in total)
  Namespace                   Name                                      CPU Requests  ...
  ---------                   ----                                      ------------
  kube-system                 coredns-xxxxxxxxxx-aaaaa                  100m (1%)     ...
  kube-system                 etcd-docker-desktop                       100m (1%)     ...
  kube-system                 kube-apiserver-docker-desktop             250m (3%)     ...
  ...
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

STEP 1에서 본 Pod들이 모두 이 노드 위에 있습니다. 노드가 한 대뿐이라 두뇌 부품과 앞으로 우리가 만들 Pod가 모두 같은 노드에서 돕니다.

## 성공하면

- STEP 1: etcd, apiserver, scheduler, controller-manager가 보입니다.
- STEP 2: `CONTAINER-RUNTIME` 칸에 `containerd://...`가 보입니다.

## 막히면

??? failure "No resources found in default namespace."
    `-n kube-system`을 빼고 실행했습니다. 네임스페이스를 안 주면 `default` 네임스페이스를 보는데, 아직 아무것도 만들지 않았으니 비어 있습니다. `kubectl get pods -n kube-system`으로 다시 실행합니다.

??? failure "Error from server (NotFound): nodes \"docker-desktop\" not found"
    노드 이름 오타입니다. `kubectl get nodes`로 이름을 확인하고 그대로 붙여 넣습니다.

??? failure "kube-system Pod 중 일부가 Running이 아님 (Pending, CrashLoopBackOff)"
    클러스터를 켠 직후라면 1~2분 기다린 뒤 다시 봅니다. 계속 그대로면 Docker Desktop의 Kubernetes 설정에서 **Reset cluster**를 누릅니다.

## 정리

만든 리소스가 없으니 정리할 것이 없습니다.
