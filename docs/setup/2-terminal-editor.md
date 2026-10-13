# 2. 터미널과 편집기

!!! abstract "이 페이지에서 할 것"
    - 명령을 입력할 터미널을 엽니다.
    - YAML 파일을 만들 편집기(VS Code)를 설치합니다.
    - 실습 폴더 `k8s-labs`를 만듭니다.

**소요 시간** 약 10분

## 터미널 열기

=== "Windows (PowerShell)"

    **시작 버튼 → "터미널" 또는 "PowerShell" 검색 → 실행**합니다. 관리자 권한은 필요 없습니다.

    이 가이드의 Windows 명령은 모두 **PowerShell** 기준입니다. 명령 프롬프트(cmd)나 WSL 우분투 창에서는 일부 명령이 다르게 동작합니다.

    프롬프트가 `PS C:\Users\이름>`처럼 `PS`로 시작하면 PowerShell입니다.

=== "Mac (터미널)"

    **Spotlight(⌘ + Space) → "터미널" 검색 → 실행**합니다. iTerm2를 써도 됩니다.

    이 가이드의 Mac 명령은 기본 셸인 **zsh** 기준입니다.

## VS Code 설치

1. [VS Code 다운로드](https://code.visualstudio.com/)에서 내 OS에 맞는 파일을 받아 설치합니다.
2. 왼쪽 **확장(Extensions)** 아이콘에서 아래 두 개를 검색해 설치합니다.

| 확장 | 만든 곳 | 하는 일 |
|---|---|---|
| YAML | Red Hat | YAML 문법 오류 표시, 쿠버네티스 필드 자동 완성 |
| Kubernetes | Microsoft | 클러스터 리소스를 화면에서 보기 (선택) |

!!! tip "VS Code 안의 터미널을 써도 됩니다"
    VS Code 메뉴 **보기 → 터미널** (단축키 `` Ctrl + ` ``)을 열면 폴더와 터미널을 한 화면에서 볼 수 있습니다. Windows는 기본이 PowerShell, Mac은 zsh입니다.

## 실습 폴더 만들기

모든 실습은 홈 폴더 아래 `k8s-labs`에서 합니다.

=== "Windows (PowerShell)"

    ```powershell
    mkdir $HOME\k8s-labs
    cd $HOME\k8s-labs
    code .
    ```

=== "Mac (터미널)"

    ```bash
    mkdir -p ~/k8s-labs
    cd ~/k8s-labs
    code .
    ```

`code .`은 지금 폴더를 VS Code로 엽니다. `code: command not found`가 나오면 아래 막히면을 봅니다.

## 파일 만드는 법

실습에서 YAML이나 Dockerfile을 만들 때는 VS Code에서 만듭니다.

1. VS Code 왼쪽 탐색기에서 실습 폴더(예: `day2`)를 우클릭 → **새 파일**
2. 가이드에 적힌 이름 그대로 입력 (예: `pod.yaml`, `Dockerfile`)
3. 가이드의 내용을 복사해 붙여 넣고 저장 (`Ctrl + S`, Mac은 `⌘ + S`)

또는 각 실습 페이지의 **내려받기** 링크로 파일을 받아 실습 폴더에 넣어도 됩니다.

!!! warning "Windows 메모장을 쓰면"
    메모장은 `Dockerfile`을 `Dockerfile.txt`로 저장합니다. 저장할 때 **파일 형식을 "모든 파일"**로 바꾸고 이름을 `"Dockerfile"`처럼 큰따옴표로 감싸야 합니다. 이 문제 때문에 VS Code를 권장합니다.

!!! warning "YAML은 띄어쓰기가 문법입니다"
    YAML은 들여쓰기를 **스페이스 2칸**으로 합니다. 탭 문자를 쓰면 `error converting YAML to JSON: yaml: line N: found character that cannot start any token` 오류가 납니다. VS Code는 YAML 파일에서 탭을 스페이스로 바꿔 줍니다.

## Windows와 Mac 명령이 다른 곳

대부분의 `docker`, `kubectl` 명령은 똑같습니다. 다른 것은 아래 몇 가지입니다. 실습 페이지에서는 탭으로 나눠 두었습니다.

| 하는 일 | Windows (PowerShell) | Mac (터미널) |
|---|---|---|
| 웹 페이지 요청 | `curl.exe http://localhost:8080` | `curl http://localhost:8080` |
| 글자로 걸러 보기 | `kubectl get pods | Select-String web` | `kubectl get pods | grep web` |
| 폴더 만들고 이동 | `mkdir $HOME\k8s-labs\day1; cd $HOME\k8s-labs\day1` | `mkdir -p ~/k8s-labs/day1 && cd ~/k8s-labs/day1` |
| 긴 명령 줄 나누기 | 줄 끝에 백틱 `` ` `` | 줄 끝에 `\` |

!!! info "PowerShell에서 curl 대신 curl.exe"
    Windows PowerShell 5.1에서 `curl`은 `Invoke-WebRequest`의 별칭이라 출력 모양이 전혀 다릅니다. Windows 10, 11에 들어 있는 진짜 curl을 쓰려면 `curl.exe`라고 입력합니다.

## 막히면

??? failure "code : 'code' 용어가 cmdlet, 함수... 의 이름으로 인식되지 않습니다 / code: command not found"
    - Windows: VS Code를 설치할 때 **PATH에 추가** 옵션이 꺼져 있었습니다. VS Code를 다시 설치하며 켜거나, VS Code에서 **파일 → 폴더 열기**로 `k8s-labs`를 엽니다.
    - Mac: VS Code에서 `⌘ + Shift + P` → **Shell Command: Install 'code' command in PATH**를 실행하고 터미널을 새로 엽니다.

??? failure "이 시스템에서 스크립트를 실행할 수 없으므로... (PowerShell)"
    이 과정에서는 스크립트 파일을 실행하지 않으므로 보통 나오지 않습니다. 나오면 그 명령을 채팅에 그대로 올려 주세요.
