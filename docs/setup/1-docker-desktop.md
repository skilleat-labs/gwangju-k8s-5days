# 1. Docker Desktop 설치

!!! abstract "이 페이지에서 할 것"
    - 내 PC에 Docker Desktop을 설치합니다.
    - `docker version`으로 설치가 끝났는지 확인합니다.

**소요 시간** 약 20분 (Windows는 WSL 설치 때문에 재부팅이 필요할 수 있습니다)

## 내 PC가 되는지 확인

=== "Windows (PowerShell)"

    | 항목 | 필요한 것 |
    |---|---|
    | Windows | Windows 10 22H2 (빌드 19045) 이상, 또는 Windows 11 23H2 (빌드 22631) 이상, 64비트 |
    | 메모리 | 8GB 이상 (16GB 권장) |
    | 가상화 | BIOS에서 하드웨어 가상화가 켜져 있어야 함 |
    | WSL | WSL 2.1.5 이상 |

    Windows 버전은 `Win + R` → `winver` 로 확인합니다.

    가상화가 켜져 있는지는 **작업 관리자 → 성능 → CPU** 오른쪽 아래 **가상화: 사용**으로 확인합니다.

=== "Mac (터미널)"

    | 항목 | 필요한 것 |
    |---|---|
    | macOS | 최신 버전과 그 이전 두 버전 |
    | 메모리 | 4GB 이상 (8GB 이상 권장) |
    | 칩 | Apple Silicon(M1~M4) 또는 Intel. 설치 파일이 다릅니다 |

    칩 종류는 화면 왼쪽 위 **사과 메뉴 → 이 Mac에 관하여**의 **칩** 또는 **프로세서** 줄에서 확인합니다.

<small>출처: [Docker Desktop Windows 설치](https://docs.docker.com/desktop/setup/install/windows-install/), [Mac 설치](https://docs.docker.com/desktop/setup/install/mac-install/)</small>

## 설치

=== "Windows (PowerShell)"

    **① WSL 준비**

    PowerShell을 **관리자 권한**으로 엽니다 (시작 버튼 우클릭 → 터미널(관리자)).

    ```powershell
    wsl --install
    ```

    이미 설치돼 있으면 최신으로 올립니다.

    ```powershell
    wsl --update
    wsl --version
    ```

    `wsl --install` 후 재부팅하라는 안내가 나오면 재부팅합니다.

    **② Docker Desktop 설치**

    1. [Docker Desktop 다운로드](https://www.docker.com/products/docker-desktop/)에서 **Download for Windows - AMD64**를 받습니다.
    2. `Docker Desktop Installer.exe`를 실행합니다.
    3. 설정 화면에서 **Use WSL 2 instead of Hyper-V**에 체크합니다.
    4. 설치가 끝나면 **Close**를 누릅니다.
    5. 시작 메뉴에서 **Docker Desktop**을 찾아 실행합니다. 자동으로 켜지지 않습니다.
    6. 처음 실행하면 이용 약관(Subscription Service Agreement) 동의 창이 나옵니다. 동의해야 실행됩니다.
    7. 로그인 화면은 **건너뛰어도 됩니다** (Skip). 이 과정은 Docker 계정이 없어도 됩니다.

=== "Mac (터미널)"

    1. [Docker Desktop 다운로드](https://www.docker.com/products/docker-desktop/)에서 내 칩에 맞는 파일을 받습니다.
        - Apple Silicon: **Download for Mac - Apple Silicon**
        - Intel: **Download for Mac - Intel Chip**
    2. `Docker.dmg`를 열고 Docker 아이콘을 **Applications** 폴더로 끌어 놓습니다.
    3. 응용 프로그램에서 **Docker**를 실행합니다.
    4. 이용 약관(Subscription Service Agreement)에 동의합니다. 동의해야 실행됩니다.
    5. 로그인 화면은 **건너뛰어도 됩니다** (Skip).

    Apple Silicon Mac은 Rosetta 2를 설치해 두면 좋습니다.

    ```bash
    softwareupdate --install-rosetta --agree-to-license
    ```

## 설치 확인

Docker Desktop 화면 왼쪽 아래가 **Engine running**(초록색)이 될 때까지 기다린 뒤, 터미널을 **새로** 열고 입력합니다.

```bash
docker version
```

```text
Client:
 Version:           28.x.x
 ...
Server: Docker Desktop 4.xx.x
 Engine:
  Version:          28.x.x
 ...
```
<small>출력은 예시입니다. 버전 숫자는 사람마다 다릅니다.</small>

**Client와 Server가 둘 다 나오면 성공**입니다. Server 부분이 없고 오류가 나오면 Docker Desktop이 아직 켜지는 중이거나 꺼져 있는 것입니다.

```bash
docker run hello-world
```

`Hello from Docker!`가 보이면 1일차 준비가 끝났습니다.

## 메모리 늘리기 (권장)

2일차부터 쿠버네티스를 켜면 메모리를 더 씁니다.

=== "Windows (PowerShell)"

    WSL 2를 쓰는 Windows는 Docker Desktop 설정이 아니라 WSL이 메모리를 정합니다. 기본값(PC 메모리의 절반)으로 충분합니다. PC 메모리가 8GB라 느리면 다른 프로그램(브라우저 탭 등)을 줄여 주세요.

=== "Mac (터미널)"

    Docker Desktop **Settings(톱니바퀴) → Resources → Advanced**에서 **Memory**를 **4GB 이상**으로 맞추고 **Apply & restart**를 누릅니다.

## 막히면

??? failure "WSL 2 installation is incomplete / WSL update failed (Windows)"
    관리자 PowerShell에서 `wsl --update`를 실행하고 PC를 재부팅한 뒤 Docker Desktop을 다시 켭니다.

??? failure "Virtualization support not detected (Windows)"
    BIOS에서 가상화가 꺼져 있습니다. PC 제조사마다 이름이 다릅니다 (Intel VT-x, Intel Virtualization Technology, AMD-V, SVM Mode). BIOS에서 켜고 재부팅합니다. 회사 PC라 BIOS에 들어갈 수 없으면 강사에게 알려 주세요.

??? failure "docker: command not found / 'docker'은(는) 내부 또는 외부 명령... 이 아닙니다"
    Docker Desktop 설치 직후 이미 열려 있던 터미널은 새 경로를 모릅니다. 터미널을 닫고 새로 엽니다. 그래도 안 되면 PC를 재시작합니다.

??? failure "\"Docker.app\" is damaged and can't be opened (Mac)"
    공식 문제 해결 문서를 따릅니다: [Docker Desktop for Mac 문제 해결](https://docs.docker.com/desktop/troubleshoot-and-support/troubleshoot/).
