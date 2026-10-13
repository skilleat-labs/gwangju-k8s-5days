# 3. Kubernetes 켜기 (2일차 수업 전까지)

!!! abstract "이 페이지에서 할 것"
    - Docker Desktop에 들어 있는 쿠버네티스를 **kubeadm** 방식으로 켭니다.
    - `kubectl get nodes`에 `docker-desktop` 노드가 `Ready`로 보이게 합니다.

**소요 시간** 약 5~10분 (PC와 인터넷 속도에 따라 다릅니다)

!!! info "왜 미리 켜 오나요"
    처음 켤 때 쿠버네티스 구성요소 이미지를 내려받느라 몇 분이 걸립니다. 수업 중에 켜면 그동안 실습을 못 따라옵니다. 1일차가 끝난 뒤 집이나 회사에서 미리 켜 주세요.

## 켜는 순서

1. Docker Desktop을 열고 왼쪽 메뉴에서 **Kubernetes**를 누릅니다.
2. **Create cluster**를 누릅니다.
3. 클러스터 종류에서 **Kubeadm**을 고릅니다.
    - Kubeadm: 노드 1대짜리 클러스터. 이 과정은 이것을 씁니다.
    - kind: 노드 여러 대. 이 과정에서는 쓰지 않습니다.
4. **Show system containers (advanced)**는 체크하지 않아도 됩니다.
5. **Create**를 누르고, 상태가 초록색(Running)이 될 때까지 기다립니다.

<small>출처: [Docker Desktop Kubernetes 문서](https://docs.docker.com/desktop/use-desktop/kubernetes/). Docker Desktop 버전에 따라 메뉴 위치가 **Settings → Kubernetes**일 수 있습니다. 그 경우 **Enable Kubernetes**를 켜고 방식은 Kubeadm을 고른 뒤 **Apply**를 누릅니다.</small>

## 켜졌는지 확인

터미널을 **새로** 엽니다.

```bash
kubectl config get-contexts
```

```text
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
*         docker-desktop   docker-desktop   docker-desktop
```
<small>출력은 예시입니다. 다른 클러스터를 써 본 적이 있으면 줄이 더 있을 수 있습니다.</small>

`docker-desktop` 줄 앞에 `*`가 없으면 아래 명령으로 바꿉니다.

```bash
kubectl config use-context docker-desktop
```

노드를 봅니다.

```bash
kubectl get nodes
```

```text
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   2m    v1.3x.x
```
<small>출력은 예시입니다. VERSION은 Docker Desktop 버전에 따라 다릅니다.</small>

**`docker-desktop`이 `Ready`면 2일차 준비가 끝났습니다.** 버전 숫자를 채팅에 올려 주시면 수업에서 함께 확인합니다.

```bash
kubectl version
```

## kubectl은 어디에 설치되나

Docker Desktop이 `kubectl`을 같이 설치합니다. 따로 설치할 필요가 없습니다.

=== "Windows (PowerShell)"

    ```powershell
    Get-Command kubectl
    ```

    `C:\Program Files\Docker\Docker\resources\bin\kubectl.exe` 또는 `...\AppData\Local\Programs\DockerDesktop\resources\bin\kubectl.exe`가 나오면 Docker Desktop이 설치한 것입니다.

=== "Mac (터미널)"

    ```bash
    which kubectl
    ```

    `/usr/local/bin/kubectl` 또는 `~/.docker/bin/kubectl`이 나옵니다. Homebrew로 kubectl을 따로 설치한 적이 있어서 버전이 엉키면 강사에게 알려 주세요.

## 다시 시작하거나 처음부터 다시 만들기

- **잠시 끄기**: Kubernetes 화면의 **Stop**. 다시 켤 때 Create cluster부터 합니다.
- **고장 났을 때 초기화**: Kubernetes 설정의 **Reset cluster**. 클러스터 안의 모든 리소스가 지워집니다.

## 막히면

??? failure "Kubernetes가 계속 Starting 상태에서 멈춤"
    1. 10분 이상 기다려도 그대로면 **Reset cluster**를 누릅니다.
    2. 그래도 안 되면 Docker Desktop을 완전히 종료(트레이 또는 메뉴 막대의 고래 아이콘 → Quit)하고 다시 켭니다.
    3. 회사 네트워크에서 이미지 다운로드가 막혀 있을 수 있습니다. 휴대폰 핫스팟으로 바꿔 시도해 봅니다.

??? failure "Unable to connect to the server: dial tcp 127.0.0.1:6443: connectex: No connection could be made..."
    클러스터가 아직 켜지는 중이거나 꺼져 있습니다. Docker Desktop의 Kubernetes 상태가 Running인지 확인합니다.

??? failure "The connection to the server localhost:8080 was refused"
    kubectl이 접속할 클러스터 설정(kubeconfig)을 못 찾은 것입니다. 터미널을 새로 열고 `kubectl config use-context docker-desktop`을 실행합니다.

??? failure "error: no context exists with the name: \"docker-desktop\""
    클러스터가 아직 만들어지지 않았습니다. 위 켜는 순서의 Create까지 했는지 확인합니다.
