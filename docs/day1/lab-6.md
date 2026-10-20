# 실습 6. 도커의 한계 직접 겪기

!!! abstract "이번 실습에서 할 것"
    - 죽은 컨테이너를 아무도 살려 주지 않는 것을 확인합니다 (한계 2).
    - 같은 포트로 늘릴 수 없어 사람이 포트를 바꿔야 하는 것을 겪습니다 (한계 3).
    - 복구와 확장을 사람이 손으로 하는 불편함을 직접 느낍니다.

**소요 시간** 약 10분, **교안** DAY 1 섹션 6

## 준비

실습 5의 미니 블로그가 떠 있어야 합니다.

```bash
docker ps
```

`frontend`와 `backend-service`가 `Up`이고, 브라우저 `http://localhost:8080`이 열리면 됩니다. 없으면 [실습 5](lab-5.md)를 다시 합니다.

## STEP 1. 컨테이너를 강제로 죽인다 (한계 2)

```bash
docker kill frontend
```

```bash
docker ps
```

```text
CONTAINER ID   IMAGE                     COMMAND           CREATED          STATUS          PORTS      NAMES
1f2e3d4c5b6a   skilleat/backend:v4-kb5   "python app.py"   10 minutes ago   Up 10 minutes   5000/tcp   backend-service
```
<small>출력은 예시입니다.</small>

`frontend`가 목록에서 사라졌고, 브라우저 `localhost:8080`도 열리지 않습니다. 기다려도 **아무도 다시 살려 주지 않습니다.** `docker ps -a`로 보면 `Exited` 상태로 남아 있습니다.

## STEP 2. 같은 포트로 하나 더 (한계 3)

사람이 직접 살립니다. 브라우저가 다시 열리는지 확인합니다.

```bash
docker start frontend
```

트래픽이 늘었다고 생각하고, 같은 8080 포트로 frontend를 하나 더 띄워 봅니다.

```bash
docker run -d --name frontend2 --network blog-net -p 8080:80 skilleat/frontend:v4-kb5
```

```text
docker: Error response from daemon: driver failed programming external connectivity on endpoint frontend2 (...): Bind for 0.0.0.0:8080 failed: port is already allocated.
```
<small>출력은 예시입니다. Docker Desktop 버전에 따라 앞부분 문구가 다를 수 있습니다.</small>

PC의 8080 포트는 컨테이너 하나만 쓸 수 있습니다. **포트 충돌 오류가 나는 것이 이 STEP의 정답입니다.**

## STEP 3. 포트를 바꿔 늘리면

STEP 2에서 오류가 나도 `frontend2` 컨테이너는 `Created` 상태로 남아 있습니다. 먼저 지웁니다.

```bash
docker rm frontend2
```

포트를 8081로 바꿔 다시 띄웁니다.

```bash
docker run -d --name frontend2 --network blog-net -p 8081:80 skilleat/frontend:v4-kb5
```

브라우저에서 `http://localhost:8080`과 `http://localhost:8081`을 둘 다 열어 봅니다. 둘 다 같은 블로그가 열립니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost:8081
    ```

=== "Mac (터미널)"

    ```bash
    curl http://localhost:8081
    ```

뜨기는 하지만 **주소가 2개**가 되었습니다. 사용자는 8080과 8081 중 어디로 가야 할지 모릅니다. 앞에서 요청을 나눠 줄 로드밸런서를 사람이 따로 만들어야 합니다. 300개로 늘린다면 어떻게 할지 생각해 봅니다.

!!! tip "내일 풀어 봅니다"
    오늘 겪은 한계 2(자동 복구), 3(확장)를 2일차에 쿠버네티스 Deployment로 풀고, 4(배포 중 끊김)는 하나씩 바꾸는 원리를 2일차에 확인한 뒤 실제 요청 연속성은 3~4일차에 확인하며, 한계 5(바뀌는 주소)를 3일차에 Service로 풉니다.

## 성공하면

- STEP 1: `frontend`가 자동으로 살아나지 않습니다.
- STEP 2: `port is already allocated` 포트 충돌 오류가 납니다.
- STEP 3: 8080과 8081, 주소가 2개가 됩니다.

## 막히면

??? failure "Conflict. The container name \"/frontend2\" is already in use by container ..."
    STEP 2에서 실패한 `frontend2`가 `Created` 상태로 남아 있습니다. STEP 3 첫 줄의 `docker rm frontend2`를 실행했는지 봅니다.

??? failure "STEP 3에서도 Bind for 0.0.0.0:8081 failed: port is already allocated"
    8081을 다른 프로그램이나 실습 3의 `myweb` 컨테이너가 쓰고 있습니다. `docker rm -f frontend2` 후 `-p 8082:80`처럼 다른 포트로 실행합니다.

??? failure "STEP 2의 docker run이 오류 없이 실행됨"
    `frontend`가 8080을 쓰고 있지 않은 상태입니다. `docker start frontend`를 먼저 실행했는지, `docker ps`에서 `frontend`의 PORTS가 `0.0.0.0:8080->80/tcp`인지 봅니다. 실습 5에서 8081 등 다른 포트를 썼다면 STEP 2도 그 포트로 바꿔야 충돌이 납니다.

??? failure "docker start frontend 후에도 브라우저가 안 열림"
    몇 초 기다린 뒤 새로 고침합니다. `docker ps`에서 `frontend`가 `Up`인지 봅니다.

## 정리

오늘 실습이 모두 끝났습니다. [오늘의 실습](index.md) 페이지 맨 아래 **정리** 명령으로 컨테이너와 네트워크를 지웁니다. **이미지는 지우지 않습니다.** 5일차에 미니 블로그 이미지를 다시 씁니다.
