# 실습 1. 컨테이너 처음 띄워 보기

!!! abstract "이번 실습에서 할 것"
    - 도커가 켜져 있는지 확인하고 첫 컨테이너를 실행합니다.
    - nginx 웹 서버를 띄워 몇 초 만에 서버 하나가 뜨는 것을 체감합니다.

**소요 시간** 약 10분, **교안** DAY 1 섹션 1

## 준비

- Docker Desktop을 실행하고 왼쪽 아래가 **Engine running**이 될 때까지 기다립니다.
- 터미널을 열고 [작업 폴더](index.md)로 이동합니다.

## STEP 1. 도커가 켜져 있는지 확인

```bash
docker version
```

```text
Client:
 Version:           27.x.x
 API version:       1.47
 OS/Arch:           windows/amd64
 ...

Server: Docker Desktop 4.x.x (xxxxxx)
 Engine:
  Version:          27.x.x
  OS/Arch:          linux/amd64
 ...
```
<small>출력은 예시입니다. 버전 숫자는 사람마다 다릅니다.</small>

**Client와 Server가 둘 다 나와야 정상입니다.** Server 부분이 없고 오류가 나오면 Docker Desktop이 꺼져 있는 것입니다. 아래 막히면을 봅니다.

Server의 `OS/Arch`가 Windows, Mac 상관없이 `linux/...`인 것을 봐 두세요. Docker Desktop은 PC 안에 작은 리눅스 가상머신을 만들고 그 안에서 컨테이너를 돌립니다. 실습 2에서 다시 나옵니다.

## STEP 2. 첫 컨테이너

```bash
docker run hello-world
```

```text
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
...
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```
<small>출력은 예시입니다.</small>

내 PC에 이미지가 없어서 Docker Hub에서 받아 온 뒤 실행합니다. 메시지를 출력하고 바로 끝나는 테스트용 컨테이너입니다.

## STEP 3. 웹 서버 띄우기

nginx 웹 서버를 백그라운드(`-d`)로 띄우고, 내 PC의 8080 포트를 컨테이너의 80 포트에 연결(`-p`)합니다. 컨테이너 이름은 `web`입니다.

```bash
docker run -d -p 8080:80 --name web nginx
```

```text
Unable to find image 'nginx:latest' locally
latest: Pulling from library/nginx
...: Pull complete
...: Pull complete
Status: Downloaded newer image for nginx:latest
3f1c2a9b8d7e6f5a4b3c2d1e0f9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a
```
<small>출력은 예시입니다. 마지막 긴 글자(컨테이너 ID)는 사람마다 다릅니다.</small>

`Pull complete`가 여러 줄 나오는 것은 이미지가 **레이어** 여러 개로 되어 있기 때문입니다. 실습 3에서 다룹니다.

## STEP 4. 떠 있는지 확인

```bash
docker ps
```

```text
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                  NAMES
3f1c2a9b8d7e   nginx     "/docker-entrypoint.…"   10 seconds ago   Up 9 seconds    0.0.0.0:8080->80/tcp   web
```
<small>출력은 예시입니다. ID와 시간은 사람마다 다릅니다.</small>

`STATUS`가 `Up`이면 실행 중입니다. 브라우저에서 접속해 봅니다.

**브라우저 주소창에 `http://localhost:8080`** 을 입력합니다. **Welcome to nginx!** 페이지가 보이면 됩니다.

터미널에서 확인하려면 아래처럼 합니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost:8080
    ```

=== "Mac (터미널)"

    ```bash
    curl http://localhost:8080
    ```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

!!! info "hello-world 는 어디 갔을까"
    `docker ps`에는 `web`만 보입니다. 멈춘 컨테이너까지 보려면 `-a`를 붙입니다.

    ```bash
    docker ps -a
    ```

    `hello-world` 컨테이너는 메시지를 찍고 할 일이 끝나 **Exited** 상태입니다. nginx는 요청을 기다리며 계속 **Up**입니다. 컨테이너는 주 프로세스가 끝나면 함께 멈춥니다.

    | 명령 | 뜻 |
    |---|---|
    | `docker run` | 이미지로 **새 컨테이너를 만들어** 실행 |
    | `docker stop` | 컨테이너 멈추기 |
    | `docker start` | 멈춘 **그 컨테이너를** 다시 실행 |
    | `docker rm` | 컨테이너 지우기 |
    | `docker ps -a` | 멈춘 컨테이너까지 보기 |

!!! note "web 컨테이너는 지우지 마세요"
    실습 2에서 이 `web` 컨테이너를 계속 씁니다.

## 성공하면

- STEP 2에서 `Hello from Docker!`가 나옵니다.
- STEP 4에서 `web`이 `Up` 상태이고, `docker ps -a`에서 `hello-world`는 `Exited`입니다.
- 브라우저 `localhost:8080`에 **Welcome to nginx!** 가 보입니다.

## 막히면

??? failure "Cannot connect to the Docker daemon / error during connect"
    Docker Desktop이 꺼져 있거나 아직 켜지는 중입니다. Windows에서는 `error during connect: ... open //./pipe/dockerDesktopLinuxEngine`처럼 나옵니다.

    Docker Desktop을 실행하고, 고래 아이콘의 움직임이 멈추고 왼쪽 아래가 **Engine running**이 될 때까지 기다린 뒤 다시 실행합니다.

??? failure "Bind for 0.0.0.0:8080 failed: port is already allocated"
    내 PC의 8080 포트를 다른 프로그램이 쓰고 있습니다. Windows에서는 다른 개발 도구나 회사 보안 프로그램이 8080을 쓰는 경우가 있습니다.

    포트를 8081로 바꿔 띄웁니다. 실패한 컨테이너가 `web`이라는 이름으로 남아 있으니 먼저 지웁니다.

    ```bash
    docker rm -f web
    ```

    ```bash
    docker run -d -p 8081:80 --name web nginx
    ```

    브라우저는 `http://localhost:8081`로 엽니다. 이후 실습에서 8080이 나오면 8081 대신 다른 빈 포트(예: 8082)를 씁니다.

??? failure "Conflict. The container name \"/web\" is already in use"
    `web`이라는 컨테이너가 이미 있습니다 (멈춘 것 포함). STEP 3을 두 번 실행하면 나옵니다. `docker ps -a`로 보고, 지운 뒤 다시 실행합니다.

    ```bash
    docker rm -f web
    ```

??? failure "toomanyrequests: You have reached your pull rate limit"
    Docker Hub는 로그인하지 않은 사용자의 내려받기 횟수를 제한합니다. 같은 네트워크에서 여러 명이 동시에 받으면 걸릴 수 있습니다. Docker Desktop에서 무료 계정으로 로그인하거나 잠시 뒤 다시 시도합니다.

??? failure "브라우저에서 사이트에 연결할 수 없음"
    - `docker ps`에서 `web`이 `Up`인지, `PORTS`에 `0.0.0.0:8080->80/tcp`가 있는지 봅니다.
    - 주소를 `https://`가 아니라 `http://localhost:8080`으로 입력했는지 봅니다.

## 정리

지울 것이 없습니다. `web` 컨테이너는 실습 2에서 씁니다.
