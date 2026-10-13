# 실습 3. 컨테이너를 지우면 파일은? / 내 이미지 만들기

!!! abstract "이번 실습에서 할 것"
    - 컨테이너를 멈췄다 켤 때와 지울 때 파일이 어떻게 다른지 확인합니다.
    - Dockerfile로 내 이미지를 만들고 실행합니다.
    - `docker history`로 내 이미지가 nginx 레이어 위에 얇은 층 하나를 얹은 것임을 확인합니다.

**소요 시간** 약 15분, **교안** DAY 1 섹션 3

## 준비

터미널이 작업 폴더에 있어야 합니다. 새 터미널이면 먼저 이동합니다.

=== "Windows (PowerShell)"

    ```powershell
    cd $HOME\k8s-labs\day1
    ```

=== "Mac (터미널)"

    ```bash
    cd ~/k8s-labs/day1
    ```

## STEP 1. 컨테이너 안에 파일 만들기

```bash
docker run -d --name note nginx
```

```bash
docker exec note sh -c "echo hello > /tmp/memo.txt"
```

```bash
docker exec note cat /tmp/memo.txt
```

```text
hello
```

## STEP 2. 멈췄다 켜면?

```bash
docker stop note
```

```bash
docker start note
```

```bash
docker exec note cat /tmp/memo.txt
```

```text
hello
```

멈췄다 켠 것은 **같은 컨테이너**라서 쓰기 레이어가 그대로 남아 있습니다.

## STEP 3. 지우고 새로 만들면?

```bash
docker rm -f note
```

```bash
docker run -d --name note nginx
```

```bash
docker exec note cat /tmp/memo.txt
```

```text
cat: /tmp/memo.txt: No such file or directory
```

쓰기 레이어는 컨테이너와 함께 지워졌고, 이미지에는 그 파일이 없습니다. **같은 이미지로 다시 만들었다고 이전 컨테이너의 데이터가 돌아오지는 않습니다.** 컨테이너 밖에 데이터를 두는 방법(볼륨, PVC)은 4일차에 다룹니다.

```bash
docker rm -f note
```

## STEP 4. 작업 폴더 만들고 이동

`day1` 안에 `myweb` 폴더를 만들고 들어갑니다.

```bash
mkdir myweb
```

```bash
cd myweb
```

## STEP 5. 편집기로 파일 2개 만들기

VS Code에서 `day1/myweb` 폴더를 우클릭 → **새 파일**로 아래 두 파일을 만듭니다. 이름을 **정확히** 똑같이 씁니다. `Dockerfile`은 확장자가 없고 D가 대문자입니다.

```html title="index.html"
--8<-- "files/day1/index.html"
```
[index.html 내려받기](../files/day1/index.html){ download }

```dockerfile title="Dockerfile"
--8<-- "files/day1/Dockerfile"
```
[Dockerfile 내려받기](../files/day1/Dockerfile){ download }

- `FROM nginx:1.27`: nginx 1.27 이미지 위에 쌓습니다 (베이스).
- `COPY`: 내 PC의 `index.html`을 nginx 기본 페이지 자리에 복사합니다. 새 레이어가 하나 생깁니다.

내려받은 파일을 쓸 때는 두 파일을 `myweb` 폴더로 옮깁니다. 파일이 제대로 있는지 확인합니다.

```bash
ls
```

```text
Dockerfile  index.html
```
<small>출력은 예시입니다. Windows PowerShell은 Mode, LastWriteTime 칸이 함께 나옵니다.</small>

`Dockerfile.txt`나 `index.html.txt`가 보이면 아래 막히면을 봅니다.

## STEP 6. 빌드하고 실행

`-t`는 이미지 이름과 태그, **맨 끝의 점(`.`)은 "지금 폴더의 Dockerfile을 써라"** 는 뜻입니다. 점을 빠뜨리지 않습니다.

```bash
docker build -t myweb:v1 .
```

```text
[+] Building 3.2s (7/7) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load metadata for docker.io/library/nginx:1.27
 => [1/2] FROM docker.io/library/nginx:1.27
 => [2/2] COPY index.html /usr/share/nginx/html/index.html
 => exporting to image
 => => naming to docker.io/library/myweb:v1
```
<small>출력은 예시입니다.</small>

만든 이미지로 컨테이너를 띄웁니다. 실습 1의 `web`이 8080을 쓰고 있으므로 8081을 씁니다.

```bash
docker run -d -p 8081:80 --name myweb myweb:v1
```

**브라우저에서 `http://localhost:8081`** 을 엽니다. 내가 쓴 **Hello, Gwangju!** 가 보이면 성공입니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost:8081
    ```

=== "Mac (터미널)"

    ```bash
    curl http://localhost:8081
    ```

```text
<h1>Hello, Gwangju!</h1>
```

## STEP 7. 레이어 보고 정리

내 이미지가 어떤 레이어로 되어 있는지 위에서부터 봅니다.

```bash
docker history myweb:v1
```

```text
IMAGE          CREATED         CREATED BY                                      SIZE
a1b2c3d4e5f6   1 minute ago    COPY index.html /usr/share/nginx/html/inde…   25B
<missing>      2 weeks ago     CMD ["nginx" "-g" "daemon off;"]                0B
<missing>      2 weeks ago     STOPSIGNAL SIGQUIT                              0B
<missing>      2 weeks ago     EXPOSE map[80/tcp:{}]                           0B
<missing>      2 weeks ago     RUN /bin/sh -c set -x && groupadd --sys…       60MB
<missing>      2 weeks ago     # debian.sh --arch 'amd64' …                    75MB
```
<small>출력은 예시입니다. 이미지 ID, 날짜, 크기, 줄 수는 사람마다 다릅니다.</small>

| 줄 | 뜻 |
|---|---|
| 맨 위 줄 | 내가 `COPY`로 만든 층. 크기가 수십 바이트로 아주 작습니다. |
| 아래 줄들 | nginx 이미지가 원래 가진 층입니다. |
| SIZE 0B | 파일은 안 바꾸고 설정만 기록한 단계입니다. |

내 이미지는 nginx 이미지 전체를 복사한 것이 아니라, **nginx 층 위에 얇은 층 하나를 얹은 것**입니다.

확인했으면 `myweb` 컨테이너를 지웁니다. **실습 6에서 8081 포트를 다시 쓰기 때문입니다.** 이미지 `myweb:v1`은 남겨 둡니다.

```bash
docker rm -f myweb
```

## 성공하면

- STEP 2에서는 `hello`가 남아 있고, STEP 3에서는 `No such file or directory`가 나옵니다.
- 브라우저 `localhost:8081`에 **Hello, Gwangju!** 가 보입니다.
- `docker history myweb:v1` 맨 위 줄이 `COPY index.html ...`입니다.

## 막히면

??? failure "failed to read dockerfile: open Dockerfile: no such file or directory"
    지금 폴더에 `Dockerfile`이라는 이름의 파일이 없습니다. 원인은 셋 중 하나입니다.

    1. **Windows 메모장으로 만들어서 `Dockerfile.txt`로 저장됨**: 파일 탐색기는 확장자를 숨겨서 `Dockerfile`처럼 보입니다. `ls`로 진짜 이름을 확인하고 이름을 바꿉니다.

        ```powershell
        Rename-Item Dockerfile.txt Dockerfile
        ```

        메모장으로 다시 저장할 때는 **파일 형식을 "모든 파일"** 로 바꾸고 이름을 `"Dockerfile"`처럼 큰따옴표로 감쌉니다. VS Code로 만들면 이 문제가 없습니다.

    2. **Mac 텍스트 편집기로 만들어서 `Dockerfile.rtf`나 `.txt`로 저장됨**: VS Code로 다시 만듭니다.
    3. **다른 폴더에서 build를 실행함**: `cd`로 `myweb` 폴더에 들어왔는지 확인합니다.

??? failure "ERROR: \"docker buildx build\" requires exactly 1 argument / \"docker build\" requires exactly 1 argument"
    build 명령 맨 끝의 점(`.`)을 빠뜨렸습니다. `docker build -t myweb:v1 .`처럼 띄어쓰기 한 칸 뒤에 점을 붙입니다.

??? failure "COPY failed: file not found / \"/index.html\": not found"
    `index.html`이 `Dockerfile`과 같은 폴더에 없거나 이름이 다릅니다 (`index.html.txt`, `Index.html` 등). `ls`로 확인합니다.

??? failure "브라우저에 Hello, Gwangju! 대신 Welcome to nginx! 가 나옴"
    8080(실습 1의 `web`)으로 접속했을 수 있습니다. 주소가 `localhost:8081`인지 봅니다. 그래도 같으면 `index.html`을 저장하지 않고 빌드한 것입니다. 저장한 뒤 컨테이너를 지우고 다시 빌드, 실행합니다.

    ```bash
    docker rm -f myweb
    ```

    그다음 STEP 6을 다시 합니다.

??? failure "Bind for 0.0.0.0:8081 failed: port is already allocated"
    8081을 다른 프로그램이나 컨테이너가 쓰고 있습니다. `docker rm -f myweb` 후 `-p 8082:80`처럼 다른 포트로 실행하고 `localhost:8082`로 접속합니다.

## 정리

`myweb` 컨테이너는 더 쓰지 않습니다. 이미지 `myweb:v1`은 남겨 둡니다.

```bash
docker rm -f myweb
```

다음 실습을 위해 작업 폴더로 돌아갑니다.

```bash
cd ..
```
