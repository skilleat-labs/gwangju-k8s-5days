# 4. 막혔을 때 (Windows, Mac)

5일 동안 자주 나오는 문제를 모았습니다. 여기서 해결이 안 되면 **오류 문구를 그대로 복사해서** 채팅에 올려 주세요. 화면 사진보다 글자가 빨리 해결됩니다.

## 도커

??? failure "Cannot connect to the Docker daemon / error during connect"
    Docker Desktop이 꺼져 있거나 아직 켜지는 중입니다. Docker Desktop을 실행하고 왼쪽 아래가 **Engine running**이 될 때까지 기다립니다.

??? failure "Bind for 0.0.0.0:8080 failed: port is already allocated"
    그 포트를 이미 다른 컨테이너나 프로그램이 쓰고 있습니다.

    ```bash
    docker ps
    ```

    같은 포트를 쓰는 컨테이너가 있으면 지웁니다 (`docker rm -f 이름`). 컨테이너가 아니라 다른 프로그램이면 포트 번호를 바꿉니다 (예: `-p 8081:80`).

??? failure "Conflict. The container name \"/web\" is already in use"
    같은 이름의 컨테이너가 이미 있습니다 (멈춘 것 포함). `docker rm -f web`으로 지우고 다시 실행합니다.

??? failure "pull access denied / manifest unknown / not found"
    이미지 이름이나 태그에 오타가 있습니다. 가이드의 이름과 한 글자씩 비교합니다.

??? failure "toomanyrequests: You have reached your pull rate limit"
    Docker Hub는 로그인하지 않은 사용자의 이미지 내려받기 횟수를 제한합니다. 같은 회사 네트워크에서 여러 명이 동시에 받으면 걸릴 수 있습니다. Docker Desktop에서 무료 계정으로 로그인하거나 잠시 뒤 다시 시도합니다.

## 쿠버네티스

??? failure "The connection to the server localhost:8080 was refused / Unable to connect to the server"
    클러스터가 꺼져 있거나 kubectl이 다른 클러스터를 보고 있습니다.

    ```bash
    kubectl config use-context docker-desktop
    kubectl get nodes
    ```

??? failure "error: error validating \"x.yaml\": ... / error converting YAML to JSON"
    YAML 들여쓰기 오류입니다. 탭 문자 대신 스페이스 2칸을 썼는지, 같은 단계의 줄이 같은 칸에서 시작하는지 봅니다. VS Code에서 줄 번호(오류의 `line N`)로 가서 확인합니다.

??? failure "error: the path \"x.yaml\" does not exist"
    지금 터미널 폴더에 그 파일이 없습니다. `ls`로 파일이 있는지 확인하고, 없으면 실습 폴더로 `cd` 합니다. Windows 메모장으로 만들었다면 이름이 `x.yaml.txt`일 수 있습니다.

??? failure "Pod가 ErrImagePull / ImagePullBackOff"
    이미지 이름이나 태그 오타, 또는 인터넷 문제입니다. `kubectl describe pod 이름` 맨 아래 **Events**에서 이유를 봅니다.

??? failure "Pod가 CrashLoopBackOff"
    컨테이너가 시작했다가 계속 죽습니다. `kubectl logs 이름 --previous`로 죽기 직전 로그를 봅니다.

??? failure "Pod가 Pending"
    `kubectl describe pod 이름`의 Events를 봅니다. 메모리가 부족하거나(Insufficient memory), PVC가 아직 묶이지 않았을(unbound PersistentVolumeClaims) 수 있습니다.

## Windows 전용

??? failure "curl 결과가 StatusCode, Content 같은 표로 나옴"
    PowerShell의 `curl`은 Invoke-WebRequest입니다. `curl.exe`라고 입력합니다.

??? failure "grep : 'grep' 용어가 ... 인식되지 않습니다"
    PowerShell에는 grep이 없습니다. `Select-String`을 씁니다. 예: `kubectl get pods | Select-String web`

??? failure "줄 끝에 \\ 를 쓴 명령이 이상하게 실행됨"
    PowerShell은 줄 이어 쓰기에 백틱(`` ` ``)을 씁니다. 가이드의 Windows 탭 명령을 복사하거나, 한 줄로 이어서 입력합니다.

??? failure "VS Code 터미널이 cmd 또는 bash로 열림"
    터미널 오른쪽 위 `+` 옆 화살표 → **PowerShell**을 고릅니다.

## Mac 전용

??? failure "Apple Silicon에서 Pod가 exec format error 또는 no match for platform"
    그 이미지가 Intel(amd64)용으로만 만들어졌습니다. 실습 페이지의 **Apple Silicon Mac** 안내에 있는 대체 이미지를 씁니다.

??? failure "zsh: no matches found: ..."
    zsh는 `*`, `?`, `[ ]`를 파일 이름 패턴으로 해석합니다. 그 부분을 큰따옴표로 감쌉니다.
