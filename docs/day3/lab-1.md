# 실습 1. 어제 Deployment에 Service 붙이기

!!! abstract "이번 실습에서 할 것"
    - ClusterIP Service를 만듭니다.
    - 클러스터 안의 임시 Pod에서 Service 이름 `web`으로 접속합니다.
    - Pod를 모두 바꿔도 같은 이름으로 접속되는 것을 확인합니다.

**소요 시간** 약 12분, **교안** DAY 3 섹션 1

## 준비

- 작업 폴더로 이동합니다. ([오늘의 실습](index.md) 참고)
- 어제 만든 `web` Deployment를 씁니다. 없으면 STEP 1에서 다시 만듭니다.

=== "Windows (PowerShell)"

    ```powershell
    cd $HOME\k8s-labs\day3
    ```

=== "Mac (터미널)"

    ```bash
    cd ~/k8s-labs/day3
    ```

## STEP 1. Deployment 준비 (있으면 unchanged)

아래 파일을 `web-deploy.yaml`로 저장하고 apply합니다. 어제 만든 `web`이 같은 내용으로 남아 있으면 `unchanged`, 없으면 `created`가 나옵니다. 어느 쪽이든 오늘 실습과 같은 상태가 됩니다. 강조한 두 줄의 라벨 `app: web`이 다음 STEP의 Service selector와 똑같아야 합니다.

```yaml title="web-deploy.yaml" hl_lines="9 13"
--8<-- "files/day3/web-deploy.yaml"
```
[web-deploy.yaml 내려받기](../files/day3/web-deploy.yaml){ download }

```bash
kubectl apply -f web-deploy.yaml
```

```text
deployment.apps/web unchanged
```
<small>어제 것이 없었다면 `created`, 이미지 버전 등이 달랐다면 `configured`가 나옵니다.</small>

| 줄 | 뜻 |
|---|---|
| `app: web` (두 곳) | Service의 selector와 똑같아야 Service가 이 Pod를 찾습니다 |
| `replicas: 3` | Pod 3개. Service가 3개로 나눠 보냅니다 |
| `containerPort: 80` | nginx가 듣는 포트를 적어 둔 **표시**입니다. 이 줄이 포트를 열지는 않습니다. 실제로 듣는 포트는 앱(nginx)이 정합니다 |

## STEP 2. Service 만들고 확인

Service YAML을 `web-svc.yaml`로 저장합니다. `apiVersion`이 `apps/v1`이 아니라 **`v1`**인 것에 주의합니다.

```yaml title="web-svc.yaml" hl_lines="8"
--8<-- "files/day3/web-svc.yaml"
```
[web-svc.yaml 내려받기](../files/day3/web-svc.yaml){ download }

| 줄 | 뜻 |
|---|---|
| `name: web` | Service 이름. 이 이름이 그대로 **DNS 이름**이 됩니다 |
| `type: ClusterIP` | 클러스터 안에서만 쓰는 주소. 생략해도 기본값입니다 |
| `selector: app: web` | 이 라벨이 붙은 Pod를 찾습니다 |
| `port`, `targetPort` | Service 80번으로 받아 **nginx가 실제로 듣는** 80번으로 넘깁니다 |

```bash
kubectl apply -f web-svc.yaml
```

```bash
kubectl get svc web
```

```text
NAME   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
web    ClusterIP   10.96.120.5    <none>        80/TCP    5s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`CLUSTER-IP`에 IP가 보이면 성공입니다. 이 IP는 Service를 지우기 전까지 바뀌지 않습니다.

!!! info "CLUSTER-IP를 브라우저에 넣으면 안 열립니다"
    ClusterIP는 클러스터 **안에서만** 통하는 가상 IP입니다. 내 PC 브라우저로는 열리지 않는 것이 정상입니다. 밖에서 여는 방법은 [실습 3](lab-3.md)에서 합니다.

## STEP 3. 임시 Pod에서 이름으로 접속

클러스터 안에 작은 임시 Pod(busybox)를 하나 띄워 `web`이라는 이름으로 요청합니다. `--rm`이라 명령이 끝나면 Pod가 자동으로 지워집니다.

```bash
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -- wget -qO- web
```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
pod "tmp" deleted
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`Welcome to nginx!`가 들어 있는 HTML이 나오면 성공입니다. IP를 하나도 적지 않고 **Service 이름**만으로 접속했습니다. 이름을 IP로 바꿔 준 것이 누구인지는 실습 2에서 봅니다.

이어서 web Pod 세 개를 **모두** 지웁니다. 어제 본 것처럼 ReplicaSet이 새 이름, 새 IP로 다시 만듭니다.

```bash
kubectl delete pod -l app=web
```

```bash
kubectl get pods -l app=web -o wide
```

새 Pod 세 개가 `1/1 Running`이 되면 위의 `kubectl run tmp ... wget -qO- web` 명령을 **그대로 다시** 실행합니다. 이번에도 `Welcome to nginx!`가 나옵니다. Pod는 전부 바뀌었는데 접속하는 쪽은 아무것도 바꾸지 않았습니다. 어제 남은 한계 "Pod가 바뀌면 주소도 바뀐다"가 Service로 풀렸습니다.

!!! note "번갈아 응답하는지는 여기서 보지 않습니다"
    nginx 기본 페이지는 어느 Pod가 답해도 똑같아서 요청이 여러 Pod로 나뉘는지는 이 화면으로 알 수 없습니다. 이 실습의 성공 조건은 "Pod가 바뀌어도 같은 이름으로 접속된다"입니다.

!!! tip "명령은 한 줄로"
    교안 화면에서는 이 명령이 두 줄로 나뉘어 있지만, PowerShell에서는 줄 끝 `\`가 동작하지 않습니다. 위처럼 한 줄로 복사해 입력합니다.

| 옵션 | 뜻 |
|---|---|
| `--rm` | 끝나면 Pod 자동 삭제 |
| `-it` | 터미널에 출력을 바로 보여 줌 |
| `--restart=Never` | Deployment 없이 Pod 하나만 만듦 |
| `-- wget -qO- web` | 컨테이너 안에서 실행할 명령. `web`에 요청해 결과를 화면에 출력 |

## 성공하면

- STEP 2: `kubectl get svc web`에 CLUSTER-IP가 보입니다.
- STEP 3: `Welcome to nginx!`가 들어 있는 HTML이 출력됩니다. Pod를 모두 지운 뒤 다시 실행해도 똑같이 나옵니다.

## 막히면

??? failure "wget: bad address 'web'"
    `web`이라는 이름을 찾지 못했습니다.

    1. `kubectl get svc`로 Service 이름이 정확히 `web`인지 봅니다.
    2. 임시 Pod를 다른 네임스페이스(`-n`)에 띄우지 않았는지 봅니다. 네임스페이스가 다르면 짧은 이름으로는 못 찾습니다. ([실습 2](lab-2.md))

??? failure "Error from server (AlreadyExists): pods \"tmp\" already exists"
    앞에서 실행한 임시 Pod가 지워지지 않고 남아 있습니다. 지우고 다시 실행합니다.

    ```bash
    kubectl delete pod tmp
    ```

??? failure "error: resource mapping not found ... no matches for kind \"Service\" in version \"apps/v1\""
    Service의 `apiVersion`을 `apps/v1`로 썼습니다. Service는 **`v1`**입니다. 파일을 고치고 다시 apply합니다.

??? failure "error converting YAML to JSON: yaml: line N: found character that cannot start any token"
    YAML 들여쓰기에 탭 문자가 들어갔습니다. 들여쓰기는 **스페이스 2칸**으로만 합니다. VS Code로 파일을 다시 만들거나 내려받기 링크를 씁니다.

??? failure "wget: can't connect to remote host (10.96.x.x): Connection refused"
    Service는 있는데 Pod까지 연결이 안 됩니다. 아래 둘을 확인합니다.

    - `targetPort`가 앱이 실제로 듣는 포트(nginx는 80)와 같은지. `containerPort`를 적었다고 그 포트가 열리지는 않습니다
    - `selector`가 Deployment의 라벨 `app: web`과 글자 하나까지 같은지. selector에는 Deployment **이름**이 아니라 Pod **라벨**을 적습니다.

    이런 연결 장애를 찾는 방법은 [실습 4](lab-4.md)에서 자세히 합니다.

??? failure "If you don't see a command prompt, try pressing enter. 다음에 아무것도 안 나옴"
    busybox 이미지를 처음 내려받는 중이거나, 출력이 붙기 전에 명령이 끝난 경우입니다. 10초쯤 기다려도 그대로면 `Ctrl + C`로 끝내고, `kubectl delete pod tmp` 후 다시 실행합니다.

??? failure "Pod를 지운 직후 다시 접속하니 실패함"
    새 Pod가 아직 `ContainerCreating`이거나 준비 중입니다. `kubectl get pods -l app=web`에서 세 개 모두 `1/1 Running`이 된 뒤 다시 실행합니다.

??? failure "Windows 메모장으로 저장했더니 web-svc.yaml.txt가 됨"
    메모장은 이름 뒤에 `.txt`를 붙입니다. 저장할 때 파일 형식을 **모든 파일**로 바꾸고 이름을 `"web-svc.yaml"`처럼 큰따옴표로 감쌉니다. VS Code를 쓰면 이 문제가 없습니다.

## 정리

!!! warning "지우지 마세요"
    `web` Deployment와 `web` Service는 실습 2, 3, 4에서 계속 씁니다. 임시 Pod `tmp`는 `--rm`이라 이미 지워졌습니다.
