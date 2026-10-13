# 1일차 오늘의 실습

오늘은 쿠버네티스를 쓰지 않습니다. 도커만으로 컨테이너를 띄우고, 컨테이너가 무엇인지 손으로 확인합니다. 마지막에는 도커 하나로 서비스를 운영할 때의 불편함을 직접 겪어 봅니다.

!!! abstract "오늘의 목표"
    - 컨테이너를 띄우고, 컨테이너가 **격리된 프로세스**라는 것을 확인합니다.
    - Dockerfile로 내 이미지를 만들고 레이어를 봅니다.
    - 두 컨테이너가 이름으로 통신하는 미니 블로그를 띄웁니다.
    - 도커만으로는 복구와 확장을 사람이 해야 한다는 것을 겪어 봅니다.

## 실습 목록

| 실습 | 내용 | 시간 | 교안 |
|---|---|---|---|
| [실습 1](lab-1.md) | 컨테이너 처음 띄워 보기 | 10분 | 섹션 1 |
| [실습 2](lab-2.md) | 컨테이너는 프로세스다 (메모리 상한 포함) | 13분 | 섹션 2 |
| [실습 3](lab-3.md) | 내 이미지 만들기 | 10분 | 섹션 3 |
| [실습 4](lab-4.md) | 도전 과제 guide 컨테이너 | 10분 | 섹션 4 |
| [실습 5](lab-5.md) | 미니 블로그 띄우기 | 12분 | 섹션 5 |
| [실습 6](lab-6.md) | 도커의 한계 직접 겪기 | 10분 | 섹션 6 |

## 시작 전 확인

- Docker Desktop이 켜져 있고 왼쪽 아래가 **Engine running**입니다. 설치가 안 됐다면 [Docker Desktop 설치](../setup/1-docker-desktop.md)를 먼저 합니다.
- 터미널과 VS Code가 준비되어 있습니다. [터미널과 편집기](../setup/2-terminal-editor.md)

## 작업 폴더 만들기

오늘 실습은 모두 `k8s-labs/day1` 폴더에서 합니다.

=== "Windows (PowerShell)"

    ```powershell
    mkdir $HOME\k8s-labs\day1; cd $HOME\k8s-labs\day1
    ```

=== "Mac (터미널)"

    ```bash
    mkdir -p ~/k8s-labs/day1 && cd ~/k8s-labs/day1
    ```

`code .`을 입력하면 이 폴더가 VS Code로 열립니다. 실습 3에서 파일을 만들 때 씁니다.

!!! info "터미널을 새로 열었다면"
    새 터미널은 홈 폴더에서 시작합니다. 실습 3처럼 파일을 쓰는 실습 전에는 위의 `cd` 명령으로 작업 폴더로 다시 이동합니다.

## 오늘 끝나고 할 일

!!! warning "2일차 수업 전까지 Kubernetes를 켜 오세요"
    [Kubernetes 켜기](../setup/3-kubernetes.md)를 따라 Docker Desktop에서 **kubeadm** 방식으로 클러스터를 만들고, `kubectl get nodes`에 `docker-desktop`이 `Ready`로 나오는지 확인합니다. 처음 켤 때 이미지를 내려받느라 몇 분이 걸려서 수업 중에 하면 실습을 못 따라옵니다.

    PC 메모리가 8GB 이하면 Docker Desktop **Settings → Resources**에서 메모리를 4GB 이상으로 올립니다.

## 정리

오늘 만든 컨테이너와 네트워크를 모두 지웁니다. **이미지는 지우지 않습니다.** 미니 블로그 이미지는 5일차에 다시 씁니다.

```bash
docker rm -f web small myweb guide frontend frontend2 backend-service
```

```bash
docker network rm blog-net
```

이미 지운 컨테이너가 있으면 `No such container` 메시지가 나오는데, 나머지는 정상으로 지워지니 신경 쓰지 않아도 됩니다.

실습 1의 hello-world 컨테이너는 이름이 무작위라서 이미지 이름으로 골라 지웁니다.

```bash
docker rm $(docker ps -aq --filter ancestor=hello-world)
```

```bash
docker ps -a
```

목록이 비어 있으면 정리가 끝났습니다. `docker images`에는 `nginx`, `myweb`, `skilleat/backend`, `skilleat/frontend` 등이 남아 있어야 합니다.
