# 실습 1. ConfigMap 만들고 두 방법으로 넣기

!!! abstract "이번 실습에서 할 것"
    - 명령 한 줄로 ConfigMap을 만들고 저장된 모습을 봅니다.
    - 같은 ConfigMap을 환경변수와 파일(볼륨) 두 길로 Pod에 넣어 읽습니다.
    - 값을 바꾼 뒤 환경변수와 파일이 다르게 바뀌는 것을 직접 확인합니다.

**소요 시간** 약 15분, **교안** DAY 4 섹션 1

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

## STEP 2. 앞 장 YAML로 Pod 올리고 Ready 기다리기

준비에서 만든 `cm-pod.yaml`을 올립니다.

```bash
kubectl apply -f cm-pod.yaml
```

```text
pod/cm-pod created
```

Pod가 Ready가 될 때까지 기다립니다. 이미지를 받는 중에 exec를 하면 실패하기 때문입니다. 120초 안에 Ready가 되지 않으면 오류로 끝나니, 그때는 `kubectl get pod cm-pod`와 `kubectl describe pod cm-pod`로 상태를 봅니다.

```bash
kubectl wait --for=condition=Ready pod/cm-pod --timeout=120s
```

```text
pod/cm-pod condition met
```

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

## STEP 4. ConfigMap 값 바꾸기 (blue → green)

같은 이름의 ConfigMap을 `APP_COLOR=green`으로 다시 만들어 덮어씁니다. `--dry-run=client -o yaml`로 YAML만 만들고, 그것을 `kubectl apply -f -`로 바로 적용합니다. 한 줄로 입력합니다.

```bash
kubectl create configmap app-config --from-literal=APP_COLOR=green --from-literal=LOG_LEVEL=info --dry-run=client -o yaml | kubectl apply -f -
```

```text
Warning: resource configmaps/app-config is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. ...
configmap/app-config configured
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`Warning`은 STEP 1에서 `create`로 만든 것을 `apply`로 바꿨다는 안내입니다. 자동으로 채워지니 무시합니다. 마지막 줄이 `configured`면 됩니다.

## STEP 5. 떠 있는 Pod에서 다시 읽기

환경변수와 파일을 다시 읽습니다.

```bash
kubectl exec cm-pod -- printenv APP_COLOR
```

```text
blue
```

```bash
kubectl exec cm-pod -- cat /etc/config/APP_COLOR
```

```text
green
```

환경변수는 그대로 `blue`입니다. 컨테이너가 시작할 때 한 번만 읽기 때문입니다. 파일은 잠시 뒤 `green`으로 바뀝니다. 아직 `blue`면 1분쯤 기다렸다가 `cat` 명령을 다시 실행합니다.

!!! info "파일은 언제 바뀌나요"
    kubelet이 주기적으로 ConfigMap을 확인해 파일을 맞춥니다. 공식 문서 기준으로 kubelet 동기화 주기(기본 1분)에 캐시 지연을 더한 시간까지 걸릴 수 있습니다. `subPath`로 파일 하나만 붙였다면 파일도 바뀌지 않습니다.

!!! note "nginx 화면은 바뀌지 않습니다"
    이 실습의 nginx는 `APP_COLOR` 값을 쓰지 않습니다. 설정이 컨테이너까지 전달되는지 확인하는 용도입니다. 실제 앱의 동작이 바뀌는지는 앱이 값을 언제 읽는지(시작할 때 한 번, 또는 파일을 다시 읽기)에 달려 있습니다.

## STEP 6. Pod를 새로 만들고 다시 읽기

Pod를 지우고 같은 파일로 다시 만듭니다. 새 컨테이너는 시작할 때 새 값을 읽습니다.

```bash
kubectl delete pod cm-pod
```

```bash
kubectl apply -f cm-pod.yaml
```

새 Pod가 Ready가 된 뒤 읽습니다.

```bash
kubectl wait --for=condition=Ready pod/cm-pod --timeout=120s
```

```text
pod/cm-pod condition met
```

```bash
kubectl exec cm-pod -- printenv APP_COLOR
```

```text
green
```

환경변수를 확실히 반영하려면 Pod를 새로 띄웁니다. Deployment라면 실습 2의 rollout restart를 씁니다.

## 성공하면

- STEP 1: `data` 아래에 `APP_COLOR: blue`, `LOG_LEVEL: info` 두 키가 보입니다.
- STEP 3: `blue`, `info`가 출력됩니다.
- STEP 5: 환경변수는 `blue` 그대로, 파일은 잠시 뒤 `green`입니다.
- STEP 6: 새 Pod의 환경변수는 `green`입니다.

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
    교안 장표의 STEP 1, STEP 4 명령은 여러 줄로 나뉘어 있습니다. 이 가이드처럼 **한 줄로** 붙여 입력하면 Windows, Mac 모두 됩니다.

??? failure "STEP 5에서 파일이 계속 blue"
    kubelet이 아직 파일을 맞추지 않았습니다. 1분쯤 기다렸다가 다시 `cat`을 실행합니다. STEP 4의 마지막 줄이 `configured`였는지도 확인합니다 (`unchanged`면 값이 그대로입니다).

??? failure "error: unable to upgrade connection: container not found (STEP 6)"
    새 Pod가 아직 뜨는 중입니다. `kubectl get pod cm-pod`로 `Running`을 보고 다시 실행합니다.

## 정리

Pod만 지웁니다. `app-config`는 남겨도 됩니다 (하루 마지막에 지웁니다).

```bash
kubectl delete pod cm-pod
```
