# 실습 2. 컨테이너는 프로세스다

!!! abstract "이번 실습에서 할 것"
    - 밖에서는 프로세스로 보이고, 안에서는 자기 혼자인 것을 확인합니다 (Namespace).
    - cgroups가 거는 메모리 상한을 숫자로 확인합니다.

**소요 시간** 약 13분 (STEP 1~3 8분, STEP 4~6 5분), **교안** DAY 1 섹션 2

## 준비

실습 1의 `web` 컨테이너가 떠 있어야 합니다.

```bash
docker ps
```

`web`이 없으면 다시 띄웁니다.

```bash
docker run -d -p 8080:80 --name web nginx
```

## STEP 1. 호스트에서 web 컨테이너의 프로세스 보기

`docker top`은 컨테이너 **밖**, 즉 Docker가 보는 쪽에서 컨테이너의 프로세스를 보여 줍니다.

```bash
docker top web
```

```text
UID      PID     PPID    C   STIME   TTY   TIME       CMD
root     2481    2459    0   01:23   ?     00:00:00   nginx: master process nginx -g daemon off;
101      2532    2481    0   01:23   ?     00:00:00   nginx: worker process
101      2533    2481    0   01:23   ?     00:00:00   nginx: worker process
```
<small>출력은 예시입니다. PID와 worker 개수는 사람마다 다릅니다.</small>

nginx master와 worker 프로세스가 보이고, PID 번호가 큽니다. 밖에서 보면 컨테이너는 그냥 **프로세스 몇 개**입니다.

!!! info "왜 작업 관리자나 ps로 안 보고 docker top으로 보나요"
    Windows 작업 관리자나 Mac의 `ps`, 활동 모니터에서는 nginx가 **보이지 않습니다**. Docker Desktop은 PC 안에 작은 리눅스 가상머신을 만들고, 컨테이너를 그 **가상머신 안에서** 돌리기 때문입니다. 우리 PC에서는 그 가상머신 프로세스 하나만 보입니다 (Windows는 `vmmem` 또는 `VmmemWSL`, Mac은 `com.docker.virtualization` 등).

    리눅스 서버에 도커를 직접 설치했다면 서버의 `ps`에서 nginx가 그대로 보입니다. 우리는 그 가상머신 안을 대신 보여 주는 `docker top`을 씁니다.

## STEP 2. 컨테이너 안에서 보이는 프로세스

이번에는 컨테이너 **안**에서 `ps`를 실행합니다. `--rm`은 실행이 끝나면 컨테이너를 바로 지우는 옵션입니다.

```bash
docker run --rm alpine ps
```

```text
PID   USER     TIME  COMMAND
    1 root      0:00 ps
```
<small>출력은 예시입니다.</small>

안에서는 `ps` 자기 자신이 **PID 1**이고, 다른 프로세스가 하나도 안 보입니다. 같은 가상머신 안에 nginx가 떠 있는데도 보이지 않습니다. 이것이 **PID 네임스페이스**입니다.

## STEP 3. 컨테이너의 hostname

```bash
docker exec web hostname
```

```text
3f1c2a9b8d7e
```

```bash
docker ps
```

```text
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                  NAMES
3f1c2a9b8d7e   nginx     "/docker-entrypoint.…"   5 minutes ago    Up 5 minutes    0.0.0.0:8080->80/tcp   web
```
<small>출력은 예시입니다. ID는 사람마다 다릅니다.</small>

hostname이 `CONTAINER ID` 앞자리와 같습니다. 컨테이너마다 자기만의 hostname을 가지는 것은 **UTS 네임스페이스**입니다.

## STEP 4. 메모리를 100MB로 제한해서 실행

`--memory` 옵션을 주면 Docker가 cgroups에 메모리 상한을 설정합니다. 이름은 `small`입니다.

```bash
docker run -d --name small --memory 100m nginx
```

## STEP 5. 사용량과 상한 보기

`--no-stream`은 한 번만 출력하고 끝내는 옵션입니다.

```bash
docker stats --no-stream small web
```

```text
CONTAINER ID   NAME    CPU %   MEM USAGE / LIMIT     MEM %   NET I/O         BLOCK I/O   PIDS
8c7d6e5f4a3b   small   0.00%   3.1MiB / 100MiB       3.10%   796B / 0B       0B / 0B     5
3f1c2a9b8d7e   web     0.00%   3.4MiB / 7.654GiB     0.04%   1.2kB / 1.1kB   0B / 0B     5
```
<small>출력은 예시입니다. 사용량과 web의 LIMIT 숫자는 사람마다 다릅니다.</small>

`MEM USAGE / LIMIT` 칸만 비교합니다.

- `small`: LIMIT가 **100MiB**입니다. 옵션 하나가 커널의 cgroups 설정이 되었습니다.
- `web`: 옵션을 안 줬으므로 LIMIT가 **Docker Desktop 가상머신 전체 메모리**입니다.

!!! note "환경에 따라 다를 수 있음"
    `web`의 LIMIT는 PC 전체 메모리가 아니라 Docker Desktop이 가상머신에 나눠 준 메모리입니다. Mac은 Docker Desktop **Settings → Resources**의 Memory 값과 거의 같습니다. Windows(WSL 2 방식)는 WSL이 쓰는 메모리 한도로 나오며, 기본값은 PC 메모리의 절반 정도입니다. 숫자는 사람마다 다릅니다.

## STEP 6. 정리

```bash
docker rm -f small
```

`web`은 지우지 않습니다.

## 성공하면

- STEP 1: nginx master, worker 프로세스가 보입니다.
- STEP 2: PID 1 하나만 보입니다.
- STEP 3: hostname이 컨테이너 ID와 같습니다.
- STEP 5: `small`의 LIMIT가 100MiB, `web`의 LIMIT는 가상머신 전체 메모리입니다.

## 막히면

??? failure "OCI runtime exec failed: exec failed: unable to start container process: exec: \"ps\": executable file not found in $PATH"
    `docker exec web ps`처럼 nginx 컨테이너에서 `ps`를 실행하면 나옵니다. nginx 이미지에는 `ps` 명령이 들어 있지 않습니다. 그래서 STEP 2는 `ps`가 들어 있는 `alpine` 이미지로 합니다.

??? failure "Error response from daemon: No such container: web"
    실습 1의 `web` 컨테이너가 없습니다. 위 준비의 `docker run` 명령으로 다시 띄웁니다.

??? failure "작업 관리자(활동 모니터)에서 nginx를 찾을 수 없음"
    정상입니다. 컨테이너는 Docker Desktop의 리눅스 가상머신 안에서 돕니다. STEP 1의 설명을 봅니다.

??? failure "Conflict. The container name \"/small\" is already in use"
    STEP 4를 두 번 실행했습니다. `docker rm -f small`로 지우고 다시 실행합니다.

## 정리

STEP 6에서 `small`을 지웠으면 끝입니다. `web`은 실습 5 전까지 그대로 둡니다.
