# 실습 3. Pod 만들기 (명령형, 선언형)

!!! abstract "이번 실습에서 할 것"
    - 명령형(`kubectl run`)과 선언형(`kubectl apply -f`) 두 방식으로 Pod를 하나씩 만듭니다.
    - 두 방식으로 만든 Pod가 똑같이 Running인지 봅니다.
    - `kubectl explain`으로 YAML 필드의 뜻을 찾아봅니다.

**소요 시간** 약 12분, **교안** DAY 2 섹션 3

## 준비

작업 폴더로 이동합니다. STEP 2에서 이 폴더에 `pod.yaml` 파일이 생깁니다.

=== "Windows (PowerShell)"

    ```powershell
    cd $HOME\k8s-labs\day2
    ```

=== "Mac (터미널)"

    ```bash
    cd ~/k8s-labs/day2
    ```

1일차의 `docker run -d --name web -p 8080:80 nginx:1.27`이 쿠버네티스에서는 어떻게 바뀌는지 봅니다.

## STEP 1. 명령형으로 빠르게

`kubectl run` 한 줄로 `web`이라는 Pod를 만듭니다. `docker run`과 거의 같습니다.

```bash
kubectl run web --image=nginx:1.27
```

```text
pod/web created
```

```bash
kubectl get pods
```

```text
NAME   READY   STATUS    RESTARTS   AGE
web    1/1     Running   0          10s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

처음에는 `ContainerCreating`으로 보일 수 있습니다. 이미지를 받는 중이니 몇 초 뒤 다시 실행합니다. 계속 지켜보고 싶으면 `kubectl get pods -w`를 쓰고, 다 보면 ++ctrl+c++로 빠져나옵니다.

## STEP 2. YAML 초안을 뽑아 선언형으로

같은 명령에 `--dry-run=client -o yaml`을 붙이면 Pod를 실제로 만들지 않고 **YAML 초안만** 출력합니다. 이것을 `pod.yaml` 파일로 저장합니다. YAML을 처음부터 손으로 쓰지 않아도 되는 실무 요령입니다.

=== "Windows (PowerShell)"

    ```powershell
    kubectl run web2 --image=nginx:1.27 --dry-run=client -o yaml | Out-File -Encoding utf8 pod.yaml
    ```

    !!! info "Windows는 왜 `>` 대신 `Out-File -Encoding utf8`을 쓰나요"
        Windows에 기본으로 들어 있는 **Windows PowerShell 5.1**에서 `>`로 저장하면 파일이 **UTF-16** 인코딩으로 저장됩니다. 요즘 kubectl은 UTF-16 파일도 대부분 읽어 내지만, 버전에 따라 apply가 `error converting YAML to JSON`으로 실패하거나, 다른 도구에서 파일이 깨져 보일 수 있습니다. `| Out-File -Encoding utf8`로 저장하면 이런 문제가 없습니다.

        PowerShell 7(`pwsh`)은 `>`도 UTF-8로 저장합니다. 어느 쪽인지 헷갈리면 위 명령을 그대로 쓰면 됩니다.

=== "Mac (터미널)"

    ```bash
    kubectl run web2 --image=nginx:1.27 --dry-run=client -o yaml > pod.yaml
    ```

<small>교안 화면에는 명령이 두 줄로 나뉘어 있지만, 가이드에서는 한 줄로 이어서 씁니다.</small>

만든 파일을 VS Code로 열어 봅니다 (`code pod.yaml`). 아래와 같은 내용입니다.

```yaml title="pod.yaml"
--8<-- "files/day2/pod.yaml"
```
[pod.yaml 내려받기](../files/day2/pod.yaml){ download }

| 부분 | 뜻 |
|---|---|
| `apiVersion`, `kind`, `metadata`, `spec` | 모든 오브젝트의 뼈대 4필드가 그대로 있습니다 |
| `labels: run: web2` | `kubectl run`이 자동으로 붙인 라벨. 실습 4에서 이 라벨로 Pod를 골라 봅니다 |
| `resources`, `dnsPolicy`, `restartPolicy` | 기본값이 적힌 것입니다 |
| `status: {}` | 쿠버네티스가 채우는 칸이라 지워도 됩니다 |

!!! note "환경에 따라 다를 수 있음"
    kubectl 버전에 따라 `metadata` 아래에 `creationTimestamp: null` 줄이 하나 더 나올 수 있습니다. 그대로 두어도 됩니다.

이제 이 파일로 Pod를 만듭니다.

```bash
kubectl apply -f pod.yaml
```

```text
pod/web2 created
```

```bash
kubectl get pods
```

```text
NAME   READY   STATUS    RESTARTS   AGE
web    1/1     Running   0          2m
web2   1/1     Running   0          8s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

같은 `kubectl apply -f pod.yaml`을 한 번 더 실행해 봅니다. `pod/web2 unchanged`가 나옵니다. 선언형은 몇 번 실행해도 결과가 같습니다.

## STEP 3. 필드 뜻은 explain으로

`kubectl explain`은 클러스터 안에 들어 있는 필드 사전입니다. 점(`.`)으로 경로를 좁혀 갑니다.

```bash
kubectl explain pod.spec.containers.image
```

```text
KIND:       Pod
VERSION:    v1

FIELD: image <string>

DESCRIPTION:
    Container image name. More info:
    https://kubernetes.io/docs/concepts/containers/images This field is
    optional to allow higher level config management to default or override
    container images in workload controllers like Deployments and StatefulSets.
```
<small>출력은 예시입니다. kubectl 버전에 따라 문구가 조금 다릅니다.</small>

`kubectl explain pod.spec`처럼 경로를 짧게 주면 그 아래에 어떤 필드가 있는지 목록이 나옵니다.

## 성공하면

- `web`, `web2` 모두 `Running`
- `explain`에 image 설명이 나옵니다.

## 막히면

??? failure "STATUS가 ErrImagePull 또는 ImagePullBackOff"
    이미지 이름이나 태그에 오타가 있습니다 (예: `nginx:1.72`, `ngnix:1.27`). Pod를 지우고 올바른 이름으로 다시 만듭니다.

    ```bash
    kubectl delete pod web
    ```

    ```bash
    kubectl run web --image=nginx:1.27
    ```

    오타가 아닌데도 이 상태면 인터넷 연결이나 회사 프록시를 확인합니다. `kubectl describe pod web` 맨 아래 Events에 정확한 이유가 나옵니다.

??? failure "Error from server (AlreadyExists): pods \"web\" already exists"
    같은 이름의 Pod가 이미 있습니다. 앞에서 이미 만든 것이면 그대로 진행합니다. 다시 만들고 싶으면 `kubectl delete pod web` 후 실행합니다.

??? failure "error: error parsing pod.yaml: error converting YAML to JSON: ..."
    Windows에서 `>`로 저장해서 파일 인코딩이 맞지 않거나, 파일을 고치다가 들여쓰기가 틀어진 경우입니다.

    - Windows는 STEP 2의 `| Out-File -Encoding utf8 pod.yaml` 명령으로 파일을 다시 만듭니다.
    - VS Code 오른쪽 아래 상태 표시줄에 `UTF-16 LE`가 보이면 그 글자를 눌러 **Save with Encoding → UTF-8**로 다시 저장해도 됩니다.
    - `found character that cannot start any token`이면 들여쓰기에 탭이 들어간 것입니다. 탭을 공백 2칸으로 바꿉니다.

??? failure "error: the path \"pod.yaml\" does not exist"
    작업 폴더가 아닌 곳에서 실행했거나 파일 이름이 다릅니다. `ls`로 `pod.yaml`이 보이는지 확인합니다. 메모장으로 저장해서 `pod.yaml.txt`가 된 경우 VS Code에서 이름을 `pod.yaml`로 바꿉니다.

## 정리

`web`, `web2` Pod는 **실습 4에서 계속 씁니다. 지우지 마세요.**
