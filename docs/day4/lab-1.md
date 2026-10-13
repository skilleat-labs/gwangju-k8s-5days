# 실습 1. ConfigMap 만들고 두 방법으로 넣기

!!! abstract "이번 실습에서 할 것"
    - 명령 한 줄로 ConfigMap을 만들고 저장된 모습을 봅니다.
    - 같은 ConfigMap을 환경변수와 파일(볼륨) 두 길로 Pod에 넣어 읽습니다.

**소요 시간** 약 12분, **교안** DAY 4 섹션 1

## 준비

`day4` 폴더에서 합니다. 아직 없으면 [오늘의 실습](index.md)에서 폴더를 만듭니다.

이번 실습에서 쓸 파일은 `cm-pod.yaml`입니다. VS Code에서 새 파일로 만들어 아래 내용을 붙여 넣거나 내려받습니다.

```yaml title="cm-pod.yaml"
--8<-- "files/day4/cm-pod.yaml"
```
[cm-pod.yaml 내려받기](../files/day4/cm-pod.yaml){ download }

| 부분 | 뜻 |
|---|---|
| `env` | `configMapKeyRef`로 `app-config`의 키 `APP_COLOR` 하나를 골라 같은 이름의 환경변수로 넣습니다. |
| `volumes` | `app-config` 전체를 `config`라는 볼륨으로 선언합니다. |
| `volumeMounts` | 그 볼륨을 `/etc/config`에 붙입니다. 키 하나가 파일 하나가 됩니다. |

중괄호 `{ }`는 YAML의 한 줄 표기입니다. 줄을 나눠 쓴 것과 뜻이 같습니다.

## STEP 1. ConfigMap 만들고 저장된 모습 보기

명령 한 줄로 ConfigMap을 만듭니다. 키 두 개(`APP_COLOR`, `LOG_LEVEL`)를 넣습니다.

```bash
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=LOG_LEVEL=info
```

```text
configmap/app-config created
```

저장된 모습을 YAML로 봅니다.

```bash
kubectl get configmap app-config -o yaml
```

```text
apiVersion: v1
data:
  APP_COLOR: blue
  LOG_LEVEL: info
kind: ConfigMap
metadata:
  creationTimestamp: "2026-01-01T00:00:00Z"
  name: app-config
  namespace: default
  resourceVersion: "12345"
  uid: 1a2b3c4d-...
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`data` 아래에 두 키가 평문 그대로 들어 있습니다. 그래서 ConfigMap에는 비밀번호를 넣지 않습니다 (실습 2의 Secret으로).

## STEP 2. 앞 장 YAML로 Pod 올리기

준비에서 만든 `cm-pod.yaml`을 올립니다.

```bash
kubectl apply -f cm-pod.yaml
```

```text
pod/cm-pod created
```

`Running`이 될 때까지 확인합니다.

```bash
kubectl get pod cm-pod
```

```text
NAME     READY   STATUS    RESTARTS   AGE
cm-pod   1/1     Running   0          10s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

## STEP 3. 환경변수와 파일로 읽기

환경변수로 들어온 값을 읽습니다.

```bash
kubectl exec cm-pod -- printenv APP_COLOR
```

```text
blue
```

볼륨으로 들어온 파일을 읽습니다.

```bash
kubectl exec cm-pod -- cat /etc/config/LOG_LEVEL
```

```text
info
```

!!! tip "더 해 보기"
    `/etc/config` 폴더에 키 이름과 같은 파일이 두 개 있는지 봅니다.

    ```bash
    kubectl exec cm-pod -- ls /etc/config
    ```

    ```text
    APP_COLOR
    LOG_LEVEL
    ```

!!! info "환경변수와 볼륨은 바뀌는 방식이 다릅니다"
    - 환경변수는 Pod가 시작할 때 한 번만 읽습니다. ConfigMap을 바꿔도 떠 있는 컨테이너의 값은 그대로입니다.
    - 볼륨 파일은 ConfigMap을 바꾸면 잠시 뒤 바뀝니다. 하지만 앱이 파일을 다시 읽어야 동작이 바뀝니다. `subPath`로 파일 하나만 붙였다면 파일도 바뀌지 않습니다.
    - 확실히 반영하려면 Pod를 새로 띄웁니다. 실습 2의 rollout restart에서 합니다.

## 성공하면

- STEP 1: `data` 아래에 `APP_COLOR: blue`, `LOG_LEVEL: info` 두 키가 보입니다.
- STEP 3: `blue`, `info`가 출력됩니다.

## 막히면

??? failure "Pod가 CreateContainerConfigError"
    Pod가 찾는 ConfigMap이 없습니다. ConfigMap 이름 오타(`app-config`)를 확인합니다.

    ```bash
    kubectl get configmap
    kubectl describe pod cm-pod
    ```

    `describe` 맨 아래 Events에 `configmap "app-config" not found`가 보이면 STEP 1을 다시 합니다. ConfigMap을 만들면 잠시 뒤 Pod가 스스로 풀립니다.

??? failure "error: failed to create configmap: configmaps \"app-config\" already exists"
    이미 만들어 둔 것입니다. 그대로 다음 STEP으로 갑니다.

??? failure "error converting YAML to JSON: yaml: line N: found character that cannot start any token"
    YAML 들여쓰기에 탭 문자가 들어갔습니다. 스페이스 2칸으로 바꿉니다. VS Code에서 만들면 탭이 스페이스로 바뀝니다.

??? failure "error: the path \"cm-pod.yaml\" does not exist"
    지금 폴더에 파일이 없거나 이름이 다릅니다. `ls`로 확인합니다. Windows 메모장으로 저장했다면 `cm-pod.yaml.txt`로 저장됐을 수 있습니다. 메모장에서 저장할 때 **파일 형식을 "모든 파일"**로 바꾸거나 VS Code로 만듭니다.

??? failure "줄 끝 역슬래시(\) 명령이 PowerShell에서 안 됨"
    교안 장표의 STEP 1 명령은 두 줄로 나뉘어 있습니다. 이 가이드처럼 **한 줄로** 붙여 입력하면 Windows, Mac 모두 됩니다.

## 정리

Pod만 지웁니다. `app-config`는 남겨도 됩니다 (하루 마지막에 지웁니다).

```bash
kubectl delete pod cm-pod
```
