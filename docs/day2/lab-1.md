# 실습 1. 쿠버네티스가 켜졌는지 확인

!!! abstract "이번 실습에서 할 것"
    - 내 노트북의 쿠버네티스 클러스터에 `kubectl`로 접속되는지 확인합니다.

**소요 시간** 약 5분, **교안** DAY 2 섹션 1

## 준비

- Docker Desktop이 켜져 있고, Kubernetes 상태가 초록색(Running)이어야 합니다. 아직 안 켰으면 [Kubernetes 켜기](../setup/3-kubernetes.md)의 켜는 순서대로 지금 **Create cluster**를 누릅니다.
- 작업 폴더로 이동합니다.

=== "Windows (PowerShell)"

    ```powershell
    cd $HOME\k8s-labs\day2
    ```

=== "Mac (터미널)"

    ```bash
    cd ~/k8s-labs/day2
    ```

## STEP 1. kubectl과 서버 버전

`kubectl`(내 PC의 명령 도구)과 클러스터(서버)의 버전을 함께 봅니다. 두 줄이 다 나오면 클러스터에 접속된 것입니다.

```bash
kubectl version
```

```text
Client Version: v1.3x.x
Kustomize Version: v5.x.x
Server Version: v1.3x.x
```
<small>출력은 예시입니다. 버전 숫자는 Docker Desktop 버전에 따라 다릅니다.</small>

`Server Version` 줄이 있는지만 보면 됩니다.

## STEP 2. 지금 접속 대상

kubectl이 어느 클러스터를 보고 있는지 확인합니다. `docker-desktop` 줄 앞에 별표(`*`)가 있어야 합니다.

```bash
kubectl config get-contexts
```

```text
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
*         docker-desktop   docker-desktop   docker-desktop
```
<small>출력은 예시입니다. 다른 클러스터를 써 본 적이 있으면 줄이 더 있을 수 있습니다.</small>

별표가 다른 줄에 있으면 바꿉니다.

```bash
kubectl config use-context docker-desktop
```

!!! info "컨텍스트와 네임스페이스"
    - **컨텍스트**: 어느 클러스터에 접속할지. 내 PC의 kubeconfig 파일에 적혀 있고, `kubectl config get-contexts`의 별표가 지금 접속 대상입니다.
    - **네임스페이스**: 접속한 클러스터 안을 나눈 칸. `-n`을 안 쓰면 `default` 칸을 봅니다. 오늘 실습은 모두 `default`에 만들고, 클러스터 두뇌 부품은 `kube-system` 칸에 있습니다(실습 2).

    ```bash
    kubectl get ns
    ```

## STEP 3. 노드가 Ready인지

```bash
kubectl get nodes
```

```text
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   1d    v1.3x.x
```
<small>출력은 예시입니다. AGE와 VERSION은 사람마다 다릅니다.</small>

노드가 `docker-desktop` 한 대뿐입니다. kubeadm 방식은 노드 1대짜리 클러스터라서, 이 한 대가 두뇌(컨트롤플레인)와 일꾼(워커노드) 역할을 같이 합니다.

## 성공하면

- STEP 1: `Server Version`이 나옵니다.
- STEP 2: `* docker-desktop`
- STEP 3: `docker-desktop`이 `Ready`

## 막히면

??? failure "The connection to the server localhost:8080 was refused - did you specify the right host or port?"
    kubectl이 접속할 클러스터 설정(kubeconfig)을 못 찾았거나 클러스터가 꺼져 있습니다.

    1. Docker Desktop의 Kubernetes 상태가 Running인지 확인합니다. 아직 안 만들었으면 Create cluster를 누릅니다.
    2. 켠 뒤에는 터미널을 **새로** 열고 `kubectl config use-context docker-desktop`을 실행합니다.

??? failure "Unable to connect to the server: dial tcp 127.0.0.1:6443: ..."
    클러스터가 아직 켜지는 중이거나 꺼져 있습니다. Docker Desktop에서 Kubernetes가 초록색이 될 때까지 기다린 뒤 다시 실행합니다.

??? failure "error: no context exists with the name: \"docker-desktop\""
    클러스터가 아직 만들어지지 않았습니다. [Kubernetes 켜기](../setup/3-kubernetes.md)의 Create까지 했는지 확인합니다.

??? failure "STEP 2에서 별표가 다른 컨텍스트(minikube, kind-..., 회사 클러스터 등)에 있음"
    예전에 다른 도구를 썼던 경우입니다. `kubectl config use-context docker-desktop`으로 바꿉니다. 바꾸지 않으면 오늘 실습이 다른 클러스터에 만들어집니다.

??? failure "Windows에서 컨텍스트가 하나도 안 보임"
    명령 프롬프트(cmd)나 WSL 우분투 창에서 실행했을 수 있습니다. **PowerShell**을 새로 열고 다시 실행합니다.

## 정리

만든 리소스가 없으니 정리할 것이 없습니다.
