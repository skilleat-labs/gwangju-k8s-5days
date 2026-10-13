# 실습 4. 도전 과제 guide 컨테이너

!!! abstract "이번 실습에서 할 것"
    - 명령 없이 조건만 보고 컨테이너를 띄웁니다.
    - 실행 중인 컨테이너 안으로 들어가 파일에서 답을 찾습니다.

**소요 시간** 약 10분, **교안** DAY 1 섹션 4

## 준비

지금까지 배운 `docker run` 옵션(`-d`, `-p`, `--name`)과 `docker exec`로 풀 수 있습니다. 막히면 교안의 **도커 기본 문법** 장을 다시 봅니다.

## 조건

| 항목 | 조건 |
|---|---|
| 이미지 | `docker/getting-started` (태그는 쓰지 않는다) |
| 컨테이너 이름 | `guide` |
| 포트 | 내 PC `1234` → 컨테이너 `80` |
| 실행 방식 | 백그라운드 |

**찾을 것**: 컨테이너 안 `/usr/share/nginx/html/50x.html` 파일의 `<title>` 문구

## 힌트

??? tip "힌트 1. 실행"
    실습 1 STEP 3의 명령과 모양이 같습니다. 이미지, 이름, 포트만 바꿉니다. 포트는 항상 **내 PC 포트:컨테이너 포트** 순서입니다.

??? tip "힌트 2. 안으로 들어가기"
    `docker exec`에 `-it`를 붙이면 컨테이너 안에서 대화형 셸을 열 수 있습니다. 이 이미지에는 `bash`가 없으니 `sh`를 씁니다.

??? tip "힌트 3. 파일 내용 보기"
    컨테이너 안은 리눅스입니다. 파일 내용은 `cat 파일경로`로 봅니다. 다 봤으면 `exit`로 나옵니다.

## 성공하면

- 브라우저 `http://localhost:1234`에 Docker Getting Started 페이지가 열립니다.
- `50x.html`의 `<title>` 문구를 찾아 채팅에 올립니다.

## 정답

??? success "정답 보기"
    1. 조건대로 실행합니다.

        ```bash
        docker run -d -p 1234:80 --name guide docker/getting-started
        ```

    2. 안으로 들어가 파일을 엽니다.

        ```bash
        docker exec -it guide sh
        ```

        프롬프트가 `/ #`으로 바뀌면 컨테이너 안입니다. 이어서 입력합니다.

        ```bash
        cat /usr/share/nginx/html/50x.html
        ```

        ```text
        <!DOCTYPE html>
        <html>
        <head>
        <title>Error</title>
        ...
        ```
        <small>출력은 예시입니다.</small>

        ```bash
        exit
        ```

    | 옵션 | 뜻 |
    |---|---|
    | `-d` | 백그라운드 |
    | `-p 1234:80` | 내 PC 포트 : 컨테이너 포트 |
    | `exec -it` | 안에서 셸 열기. `-i`는 입력을 받고, `-t`는 터미널을 붙입니다. |

    !!! note "환경에 따라 다를 수 있음"
        기본 nginx의 `50x.html` 기준으로 title은 `Error`입니다. 이미지가 바뀌어 다른 문구가 나오면 그 문구가 정답입니다.

## 막히면

??? failure "OCI runtime exec failed: exec failed: unable to start container process: exec: \"bash\": executable file not found in $PATH"
    이 이미지에는 `bash`가 없습니다. `docker exec -it guide sh`로 들어갑니다.

??? failure "브라우저 localhost:1234 가 안 열림"
    포트 순서를 거꾸로(`-p 80:1234`) 썼을 가능성이 큽니다. `docker ps`의 `PORTS`가 `0.0.0.0:1234->80/tcp`인지 봅니다. 다르면 지우고 다시 실행합니다.

    ```bash
    docker rm -f guide
    ```

??? failure "Bind for 0.0.0.0:1234 failed: port is already allocated"
    1234 포트를 다른 프로그램이 쓰고 있습니다. `docker rm -f guide` 후 `1235`처럼 다른 포트로 실행합니다.

??? failure "Conflict. The container name \"/guide\" is already in use"
    `guide`가 이미 있습니다. `docker rm -f guide`로 지우고 다시 실행합니다.

??? failure "cat: can't open '/usr/share/nginx/html/50x.html': No such file or directory"
    경로 오타입니다. `ls /usr/share/nginx/html`로 파일 목록을 먼저 봅니다.

??? failure "프롬프트가 / # 에서 안 돌아옴"
    아직 컨테이너 안입니다. `exit`를 입력하면 내 PC 터미널로 돌아옵니다.

## 정리

```bash
docker rm -f guide
```
