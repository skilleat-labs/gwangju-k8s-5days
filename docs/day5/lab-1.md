# 실습 1. securityContext와 PSA

!!! abstract "이번 실습에서 할 것"
    - 아무 설정 없는 Pod는 root로 돈다는 것을 봅니다.
    - 네임스페이스에 Pod Security Admission(PSA) 라벨을 걸어 위험한 Pod가 거부당하는 것을 봅니다.
    - 기준을 지킨 Pod는 같은 네임스페이스에서 통과하고, root가 아닌 사용자로 도는 것을 확인합니다.

**소요 시간** 약 12분, **교안** DAY 5 섹션 1

## 준비

[오늘의 실습](index.md)의 작업 폴더로 이동합니다.

=== "Windows (PowerShell)"

    ```powershell
    cd $HOME\k8s-labs\day5
    ```

=== "Mac (터미널)"

    ```bash
    cd ~/k8s-labs/day5
    ```

이번 실습에 쓰는 파일 두 개를 작업 폴더에 만듭니다. VS Code에서 새 파일로 만들어 붙여 넣거나 내려받습니다.

```yaml title="bad-pod.yaml"
--8<-- "files/day5/bad-pod.yaml"
```
[bad-pod.yaml 내려받기](../files/day5/bad-pod.yaml){ download }

`privileged: true`는 컨테이너에 호스트의 커널 권한을 거의 전부 넘기는 설정입니다. 거기에 nginx는 root로 돕니다.

```yaml title="good-pod.yaml"
--8<-- "files/day5/good-pod.yaml"
```
[good-pod.yaml 내려받기](../files/day5/good-pod.yaml){ download }

`restricted` 기준이 요구하는 네 가지를 넣었습니다.

| 필드 | 뜻 |
|---|---|
| `runAsNonRoot: true`, `runAsUser: 1000` | root로 뜨지 않고, 사용자 번호 1000으로 돈다 |
| `seccompProfile: {type: RuntimeDefault}` | 컨테이너 런타임의 기본 시스템 콜 필터를 쓴다 |
| `allowPrivilegeEscalation: false` | 실행 중에 권한을 올리지 못한다 |
| `capabilities: {drop: ["ALL"]}` | 리눅스 capabilities를 전부 버린다 |

중괄호 `{ }`는 YAML에서 한 줄로 쓰는 방법입니다. 두 줄로 풀어 써도 같습니다. 이미지는 nginx 대신 busybox로 `sleep`만 합니다. nginx는 기준은 통과해도 root가 아니면 80 포트와 캐시 폴더 권한 때문에 컨테이너가 죽을 수 있습니다.

## STEP 1. 보통 Pod는 누구로 도나

아무 설정 없이 nginx Pod를 띄우고, 컨테이너 안에서 `whoami`를 실행합니다.

```bash
kubectl run root-test --image=nginx:1.27
```

```bash
kubectl get pod root-test
```

`STATUS`가 `Running`이 되면 실행합니다.

```bash
kubectl exec root-test -- whoami
```

```text
root
```

공격자가 이 컨테이너의 셸을 얻으면 바로 root 권한을 갖게 됩니다.

## STEP 2. restricted 네임스페이스

`secure` 네임스페이스를 만들고, 그 안의 모든 Pod를 `restricted` 기준으로 검사하도록 라벨을 붙입니다.

```bash
kubectl create namespace secure
```

```bash
kubectl label ns secure pod-security.kubernetes.io/enforce=restricted
```

```text
namespace/secure created
namespace/secure labeled
```

라벨이 붙었는지 봅니다.

```bash
kubectl get ns secure --show-labels
```

```text
NAME     STATUS   AGE   LABELS
secure   Active   10s   kubernetes.io/metadata.name=secure,pod-security.kubernetes.io/enforce=restricted
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

라벨 이름의 마지막 부분이 모드(`enforce`, `warn`, `audit`), 값이 기준(`privileged`, `baseline`, `restricted`)입니다. `enforce`는 어기는 Pod를 아예 만들지 않습니다.

## STEP 3. 위험한 Pod 올리기

```bash
kubectl apply -f bad-pod.yaml -n secure
```

```text
Error from server (Forbidden): error when creating "bad-pod.yaml": pods "bad-pod" is forbidden: violates PodSecurity "restricted:latest": privileged (container "app" must not set securityContext.privileged=true), allowPrivilegeEscalation != false (container "app" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "app" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "app" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "app" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```
<small>출력은 예시입니다. 실제로는 한 줄로 길게 나오고, 항목 순서와 문구는 쿠버네티스 버전에 따라 다를 수 있습니다.</small>

읽는 법은 세 가지입니다.

- `Forbidden`: Pod가 실행된 뒤 막힌 것이 아니라 **아예 만들어지지 않았습니다.**
- `restricted:latest`: 어긴 기준과 버전입니다.
- 괄호 안: 어느 컨테이너의 어떤 필드를 어떻게 바꾸라는지 그대로 적혀 있습니다. 위 good-pod.yaml의 필드와 하나씩 짝지어 보세요.

## STEP 4. 규칙을 지킨 Pod

같은 `secure` 네임스페이스에 good-pod를 올립니다.

```bash
kubectl apply -f good-pod.yaml -n secure
```

```bash
kubectl get pod good-pod -n secure
```

```text
NAME       READY   STATUS    RESTARTS   AGE
good-pod   1/1     Running   0          8s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

## STEP 5. 누구로 도는지

```bash
kubectl exec -n secure good-pod -- id
```

```text
uid=1000 gid=0(root) groups=0(root)
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`uid=1000`이면 root가 아닙니다. 공격자가 이 컨테이너에 들어와도 root 권한이 없습니다.

`gid=0(root)`는 그룹 번호입니다. good-pod.yaml에 `runAsGroup`을 주지 않아서 이미지 기본값인 0이 남았습니다. root 사용자가 된 것은 아니지만, 그룹까지 바꾸려면 Pod `securityContext`에 `runAsGroup: 1000`을 더합니다.

## STEP 6. 정리

```bash
kubectl delete pod root-test
```

```bash
kubectl delete ns secure
```

네임스페이스를 지우면 그 안의 good-pod도 함께 지워집니다.

## 성공하면

- STEP 1: `root`
- STEP 3: `Forbidden`, `violates PodSecurity`
- STEP 4: `Running`
- STEP 5: `uid=1000` (root 아님)

## 막히면

??? failure "error: unable to upgrade connection: container not found / pod root-test does not have a host assigned"
    Pod가 아직 생성 중입니다. `kubectl get pod root-test`로 `Running`을 확인하고 몇 초 뒤 다시 실행합니다.

??? failure "STEP 4에서도 Forbidden"
    메시지 괄호 안에 나온 필드가 good-pod.yaml에 빠졌거나 들여쓰기가 틀린 것입니다. 특히 `seccompProfile` 줄을 빠뜨리면 아래처럼 다시 거부됩니다.

    ```text
    seccompProfile (pod or container "app" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
    ```

    내려받기 파일과 한 줄씩 비교합니다.

??? failure "good-pod가 CreateContainerConfigError"
    `runAsNonRoot: true`만 넣고 `runAsUser`를 빠뜨린 경우입니다. busybox 이미지는 기본 사용자가 root라서 "root로 뜨지 마라"와 충돌해 시작이 막힙니다. `runAsUser: 1000`을 같이 씁니다.

??? failure "Deployment로 올렸더니 Deployment는 생겼는데 READY 0/1"
    PSA는 Pod를 만들 때 검사합니다. Deployment 자체는 만들어지고, 그 아래 ReplicaSet이 Pod를 못 만들어 멈춥니다. 원인은 `kubectl describe rs -n 네임스페이스`의 Events에 `forbidden: violates PodSecurity`로 나옵니다.

??? failure "kubectl label 명령을 두 줄로 나눠 붙여 넣었더니 오류"
    PowerShell은 줄 끝 `\`를 줄 이어 쓰기로 알아듣지 못합니다. 이 페이지의 명령처럼 한 줄로 입력합니다.

??? failure "error: the path \"bad-pod.yaml\" does not exist"
    작업 폴더에 파일이 없습니다. `ls`로 확인하고, 없으면 `cd`로 작업 폴더로 이동합니다. Windows 메모장으로 만들었다면 이름이 `bad-pod.yaml.txt`일 수 있습니다. VS Code로 다시 만듭니다.

## 정리

STEP 6에서 이미 정리했습니다. 남은 것이 없는지 봅니다.

```bash
kubectl get pods
```

`root-test`가 없으면 됩니다.
