# 실습 가이드 작성 규칙 (DAY 1~5 공통)

광주전남 데이터센터 IT 인프라 쿠버네티스 5일 과정의 MkDocs 실습 가이드. 수강생은 도커, 쿠버네티스 입문자. Windows와 Mac 노트북이 섞여 있다. 실습 환경은 Docker Desktop, 쿠버네티스는 Docker Desktop 내장 **kubeadm 단일 노드** (컨텍스트 `docker-desktop`).

## 원천
- 교안 빌드 스크립트: `../ppt-template/build_day{N}.js`. 각 `K.lab(...)`의 code, goal, ok, warn과 실습 앞 YAML 장표, 노트의 `[흔한 실수]`를 가이드로 옮긴다. **교안과 명령, 파일 이름, 리소스 이름, STEP 번호를 똑같이** 맞춘다 (수강생이 화면 장표와 가이드를 나란히 본다).
- 기획안: `../../5일과정_기획안.md`
- `docs/setup/*.md`와 `docs/index.md`(완성본)의 문체와 형식을 따른다.

## 파일
- 페이지: `docs/day{N}/index.md` (오늘의 실습 개요), `docs/day{N}/lab-1.md` ~ `lab-6.md`. mkdocs.yml nav는 이미 정해져 있으니 **nav와 다른 파일 이름을 만들지 않는다.** mkdocs.yml, docs/index.md, docs/setup/*, docs/files.md, GUIDE_RULES.md는 수정 금지.
- 실습에 쓰는 YAML, Dockerfile, html 등은 **실제 파일로** `docs/files/day{N}/` 에 만든다. 페이지에서는 snippets로 불러와 보여 주고 다운로드 링크를 단다:

  ````markdown
  ```yaml title="web-deploy.yaml"
  --8<-- "files/day2/web-deploy.yaml"
  ```
  [web-deploy.yaml 내려받기](../files/day2/web-deploy.yaml){ download }
  ````
- 빈칸 문제 파일과 정답 파일은 따로 (`*-blank.yaml`, `*-answer.yaml`). 정답은 페이지에서 `??? success "정답 보기"` 접힘 안에.

## 페이지 형식 (lab-N.md)
```markdown
# 실습 N. 제목

!!! abstract "이번 실습에서 할 것"
    - 목표 1~3줄 (교안 goal)

**소요 시간** 약 N분, **교안** DAY N 섹션 N

## 준비
(필요한 파일, 앞 실습에서 이어지는 상태, 작업 폴더로 이동)

## STEP n. 제목
설명 1~2문장 → 명령 → 확인할 결과(출력 예시) 

## 성공하면
- 교안 ok 항목

## 막히면
??? failure "오류 문구 그대로"
    원인과 해결

## 정리
(다음 실습에 필요 없는 리소스 지우는 명령. 다음 실습/다음 날에 계속 쓰면 "지우지 마세요"라고 명시)
```

## Windows / Mac 규칙 (가장 중요)
- 탭 이름은 **정확히** `=== "Windows (PowerShell)"` 와 `=== "Mac (터미널)"` 두 가지만 쓴다 (content.tabs.link로 한 번 고르면 전 페이지가 같이 바뀐다). 순서도 Windows 먼저.
- **두 OS에서 똑같이 동작하는 명령은 탭 없이 한 블록**으로 쓰고, 블록 위에 아무 말도 붙이지 않는다. docker, kubectl 명령은 대부분 같다.
- **명령은 한 줄로 쓴다.** 줄 이어 쓰기(`\`)는 PowerShell에서 안 된다. 너무 길어서 꼭 나눠야 하면 탭으로 나눠 Windows는 백틱(`` ` ``), Mac은 `\`.
- 탭이 필요한 대표 경우:
  | 하는 일 | Windows (PowerShell) | Mac (터미널) |
  |---|---|---|
  | 호스트에서 HTTP 요청 | `curl.exe http://localhost:8080` (그냥 `curl`은 Invoke-WebRequest 별칭이라 출력이 다름) | `curl http://localhost:8080` |
  | 출력에서 글자 찾기 | `kubectl get pods \| Select-String web` | `kubectl get pods \| grep web` |
  | base64 풀기 | `[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String("..."))` | `echo "..." \| base64 -d` |
  | base64 만들기 | `[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("..."))` | `echo -n "..." \| base64` |
  | 폴더 만들고 이동 | `mkdir $HOME\k8s-labs\day1; cd $HOME\k8s-labs\day1` | `mkdir -p ~/k8s-labs/day1 && cd ~/k8s-labs/day1` |
  | 파일 내용 보기 | `Get-Content x.yaml` (또는 `cat`, PowerShell 별칭) | `cat x.yaml` |
  | 환경변수 | `$env:NAME="v"` | `export NAME=v` |
  | 반복 관찰 | `kubectl get pods -w` (watch 명령 없음) | `kubectl get pods -w` |
- 쿠버네티스 쪽은 OS와 무관한 방법을 우선한다: secret 값 보기는 `kubectl get secret NAME -o jsonpath="{.data.KEY}"` 후 위 표로 풀기, 또는 **go-template**: `kubectl get secret NAME -o go-template="{{.data.KEY | base64decode}}"` (두 OS 공통, 큰따옴표).
- `kubectl exec POD -- sh -c "..."` 처럼 컨테이너 안 명령은 큰따옴표로 감싼다 (두 셸 공통).
- `kubectl apply -f - <<EOF` (heredoc) 금지. 항상 파일을 만들어 `kubectl apply -f 파일`.
- 파일 만들기: **VS Code로 만든다**가 기본 (setup/2 참고). 또는 실습 파일 내려받기. Windows 메모장 사용 시 `.txt` 붙는 문제를 해당 실습 `막히면`에 넣는다.
- 작업 폴더: 모든 실습은 `~/k8s-labs/day{N}` (Windows `$HOME\k8s-labs\day{N}`)에서 한다. 각 day index.md에 폴더 만들기 탭을 둔다.
- 포트: Windows는 8080 등을 다른 프로그램이 쓰는 경우가 있다. `port is already allocated` 대처를 막히면에 넣는다.
- Apple Silicon (M1~M4) Mac: amd64 전용 이미지는 Pod가 `exec format error`나 `no match for platform`으로 실패할 수 있다. 멀티 아키텍처 이미지만 쓰거나, 쓰면 `!!! warning "Apple Silicon Mac"`으로 대안을 준다.

## 문체
- 존댓말 해요체 대신 **"~합니다" 체** (setup 페이지 참고). 짧게.
- 가운뎃점(·) 쓰지 않는다. 나열은 쉼표.
- 이모지 쓰지 않는다.
- 출력 예시는 ```` ```text ```` 블록. Docker Desktop에서 실측하지 않은 출력은 블록 아래에 `<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>` 를 붙인다. 교안 노트에서 `[실측 필요]`였던 동작은 `!!! note "환경에 따라 다를 수 있음"`으로 대안을 같이 적는다 (예: NetworkPolicy가 안 막히면, LoadBalancer가 localhost로 안 열리면 port-forward).

## 검증
```bash
cd "<repo>" && mkdocs build --strict 2>&1 | tail -20
```
자기 일차 페이지에서 경고(링크 깨짐, snippet 파일 없음)가 0이 되게 한다. 다른 일차 파일이 아직 없어서 나는 경고는 무시해도 된다 (동시에 작업 중). 전체 nav 경고만 남으면 된다.
