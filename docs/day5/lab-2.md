# 실습 2. RBAC

!!! abstract "이번 실습에서 할 것"
    - `--as`로 ServiceAccount 입장이 되어, 그 토큰으로 무엇을 할 수 있는지 미리 확인합니다.
    - Pod 읽기만 되는 Role을 만들어 default ServiceAccount에 연결합니다.
    - 준 것만 되고 안 준 것은 안 되는 것, 토큰을 끈 Pod에는 토큰 파일이 없는 것을 확인합니다.

**소요 시간** 약 12분, **교안** DAY 5 섹션 2

!!! info "인증과 인가"
    API 서버는 요청마다 두 가지를 차례로 묻습니다. **인증**은 "누구인가"로, kubeconfig의 인증서나 ServiceAccount 토큰으로 확인합니다. 실패하면 `401 Unauthorized`입니다. **인가**는 "무엇을 할 수 있나"로, RBAC의 Role과 RoleBinding이 정합니다. 실패하면 `403 Forbidden`입니다. 이번 실습의 `--as`는 내 계정으로 인증한 채 그 SA인 척해서(impersonation), "이 SA라면 인가가 어떻게 나는지"를 물어보는 방법입니다.

    인증과 인가를 통과한 요청은 마지막으로 **어드미션**을 지납니다. 요청 내용이 정책에 맞는지 검사하는 단계로, 실습 1에서 restricted 네임스페이스가 위험한 Pod를 거부한 PSA가 여기서 동작합니다. 세 관문을 모두 통과해야 etcd에 기록됩니다.

    ```mermaid
    flowchart LR
        R["요청<br>kubectl, Pod"] --> A["① 인증<br>누구인가 (401)"] --> Z["② 인가<br>RBAC (403)"] --> M["③ 어드미션<br>PSA (거부)"] --> E["etcd 저장"]
    ```

## 준비

작업 폴더로 이동합니다.

=== "Windows (PowerShell)"

    ```powershell
    cd $HOME\k8s-labs\day5
    ```

=== "Mac (터미널)"

    ```bash
    cd ~/k8s-labs/day5
    ```

STEP 10에서 쓸 파일을 작업 폴더에 만듭니다.

```yaml title="no-token.yaml"
--8<-- "files/day5/no-token.yaml"
```
[no-token.yaml 내려받기](../files/day5/no-token.yaml){ download }

`automountServiceAccountToken: false`는 `spec` 바로 아래, `containers`와 같은 칸에 씁니다. 이 Pod에는 ServiceAccount 토큰 파일이 꽂히지 않습니다.

## STEP 7. 기본 SA가 할 수 있는 일

연습용 네임스페이스를 만들고, 그 네임스페이스의 `default` ServiceAccount가 할 수 있는 일을 전부 물어봅니다. Pod를 만들 때 SA를 정하지 않으면 이 `default` SA의 토큰이 꽂힙니다.

```bash
kubectl create ns rbac-lab
```

```bash
kubectl auth can-i --list -n rbac-lab --as=system:serviceaccount:rbac-lab:default
```

```text
Resources                                       Non-Resource URLs                      Resource Names   Verbs
selfsubjectreviews.authentication.k8s.io        []                                     []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                                     []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                     []               [create]
                                                [/.well-known/openid-configuration/]   []               [get]
                                                [/api/*]                               []               [get]
                                                [/api]                                 []               [get]
                                                [/apis/*]                              []               [get]
                                                [/apis]                                []               [get]
                                                [/healthz]                             []               [get]
                                                [/livez]                               []               [get]
                                                [/openid/v1/jwks/]                     []               [get]
                                                [/readyz]                              []               [get]
                                                [/version/]                            []               [get]
                                                [/version]                             []               [get]
```
<small>출력은 예시입니다. 목록은 쿠버네티스 버전에 따라 다릅니다.</small>

`--as=system:serviceaccount:<네임스페이스>:<이름>`은 "이 SA인 척하고 물어본다"는 뜻입니다. 자기 정보를 확인하는 몇 가지만 있고 **`pods` 줄이 없습니다.** 지금 이 토큰으로는 Pod를 볼 수 없습니다.

!!! note "환경에 따라 다를 수 있음"
    목록 맨 위에 `*.*` 와 `[*]`가 보이면, 이 클러스터에 모든 ServiceAccount에게 관리자 권한을 주는 바인딩이 있는 것입니다. 일부 로컬 클러스터 도구가 편의를 위해 이렇게 만들어 둡니다. 아래 명령으로 어떤 바인딩이 SA에게 `cluster-admin`을 주는지 찾아 강사에게 알려 주세요. 이 경우 STEP 9의 `delete`도 `yes`로 나옵니다. 실무 클러스터에서는 있으면 안 되는 설정입니다.

    === "Windows (PowerShell)"

        ```powershell
        kubectl get clusterrolebinding -o wide | Select-String cluster-admin
        ```

    === "Mac (터미널)"

        ```bash
        kubectl get clusterrolebinding -o wide | grep cluster-admin
        ```

## STEP 8. 읽기 전용 Role과 연결

Pod를 `get`, `list`만 할 수 있는 Role을 만들고, RoleBinding으로 `default` SA에 연결합니다.

```bash
kubectl create role pod-reader -n rbac-lab --verb=get,list --resource=pods
```

```bash
kubectl create rolebinding read-pods -n rbac-lab --role=pod-reader --serviceaccount=rbac-lab:default
```

```text
role.rbac.authorization.k8s.io/pod-reader created
rolebinding.rbac.authorization.k8s.io/read-pods created
```

`--serviceaccount` 값은 `네임스페이스:이름` 형식입니다.

??? tip "같은 것을 YAML로 만들면 (교안 RBAC 3 장표)"
    위 명령 두 줄은 아래 YAML과 결과가 같습니다. 명령으로 이미 만들었다면 이 파일은 적용하지 않아도 됩니다. 적용하면 `unchanged` 또는 `configured`가 나옵니다.

    ```yaml title="pod-reader-role.yaml"
    --8<-- "files/day5/pod-reader-role.yaml"
    ```
    [pod-reader-role.yaml 내려받기](../files/day5/pod-reader-role.yaml){ download }

    - `rules`: `apiGroups: [""]`는 Pod 같은 핵심 리소스, `resources`는 대상, `verbs`는 허용할 동작입니다.
    - `subjects`: 누구에게 줄지, `roleRef`: 어떤 Role을 줄지입니다.
    - `roleRef`는 만든 뒤 바꿀 수 없습니다. 다른 Role을 연결하려면 RoleBinding을 지우고 다시 만듭니다.

## STEP 9. yes / no로 확인

같은 SA로 Pod 목록 조회와 삭제를 각각 물어봅니다.

```bash
kubectl auth can-i list pods -n rbac-lab --as=system:serviceaccount:rbac-lab:default
```

```text
yes
```

```bash
kubectl auth can-i delete pods -n rbac-lab --as=system:serviceaccount:rbac-lab:default
```

```text
no
```

준 것(`list`)만 되고, 안 준 것(`delete`)은 안 됩니다. `no`는 오류가 아니라 정상 답입니다.

시간이 남으면 네임스페이스를 `default`로 바꿔 물어봅니다. Role은 `rbac-lab` 안에서만 유효해서 `no`가 나옵니다.

```bash
kubectl auth can-i list pods -n default --as=system:serviceaccount:rbac-lab:default
```

## STEP 10. 토큰을 끈 Pod

```bash
kubectl apply -f no-token.yaml -n rbac-lab
```

```bash
kubectl wait --for=condition=Ready pod/no-token -n rbac-lab --timeout=120s
```

Ready가 되면 토큰이 꽂히는 경로를 봅니다.

```bash
kubectl exec -n rbac-lab no-token -- ls /var/run/secrets/kubernetes.io/serviceaccount
```

```text
ls: /var/run/secrets/kubernetes.io/serviceaccount: No such file or directory
command terminated with exit code 1
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

토큰 파일이 없습니다. 공격자가 이 컨테이너에 들어와도 들고 나갈 출입증이 없습니다.

??? info "비교: 토큰을 끄지 않은 Pod"
    아무 설정 없는 Pod에는 같은 경로에 `token` 파일이 있습니다.

    ```bash
    kubectl run with-token --image=busybox:1.36 -n rbac-lab -- sleep 3600
    ```

    ```bash
    kubectl wait --for=condition=Ready pod/with-token -n rbac-lab --timeout=120s
    ```

    ```bash
    kubectl exec -n rbac-lab with-token -- ls /var/run/secrets/kubernetes.io/serviceaccount
    ```

    ```text
    ca.crt
    namespace
    token
    ```

## 성공하면

- STEP 7: `pods` 줄이 없다
- STEP 8: role, rolebinding `created`
- STEP 9: list → `yes`, delete → `no`
- STEP 10: `No such file or directory`

## 막히면

??? failure "error: serviceaccount must be <namespace>:<name>"
    `--serviceaccount=default`처럼 이름만 썼습니다. `--serviceaccount=rbac-lab:default`로 네임스페이스를 앞에 붙입니다.

??? failure "STEP 9에서 list도 no"
    STEP 8의 Role이나 RoleBinding이 다른 네임스페이스에 만들어졌을 수 있습니다. 둘 다 `-n rbac-lab`이 붙었는지 봅니다.

    ```bash
    kubectl get role,rolebinding -n rbac-lab
    ```

??? failure "터미널에 no가 빨갛게 나옴"
    `can-i`는 답이 `no`면 종료 코드 1을 돌려줘서 일부 터미널이 오류처럼 표시합니다. 오류가 아니라 정상 답입니다.

??? failure "no-token.yaml 적용 시 unknown field \"automountServiceAccountToken\""
    `containers` 안에 넣었습니다. `spec` 바로 아래, `containers:`와 같은 칸에서 시작하도록 들여쓰기를 고칩니다.

??? failure "STEP 10에서 token 파일이 보임"
    위와 같은 들여쓰기 문제로 설정이 무시됐거나, 예전에 만든 같은 이름의 Pod가 남아 있는 것입니다. `kubectl delete pod no-token -n rbac-lab` 후 다시 적용합니다.

## 정리

다음 실습에서 쓰지 않습니다. 네임스페이스를 지우면 Role, RoleBinding, Pod가 함께 지워집니다.

```bash
kubectl delete ns rbac-lab
```
