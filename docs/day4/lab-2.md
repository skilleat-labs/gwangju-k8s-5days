# 실습 2. Secret 만들고 값 바꾸기

!!! abstract "이번 실습에서 할 것"
    - Secret을 만들고, base64로 적힌 값이 명령 한 줄로 평문으로 풀리는 것을 봅니다.
    - Secret 값을 바꿔도 떠 있는 Pod는 옛 값 그대로이고, rollout restart로 새 Pod를 띄워야 새 값을 읽는 것을 봅니다.

**소요 시간** 약 12분, **교안** DAY 4 섹션 2

## 준비

`day4` 폴더에서 합니다. 이번 실습에서 쓸 파일은 `secret-demo.yaml`입니다.

```yaml title="secret-demo.yaml"
--8<-- "files/day4/secret-demo.yaml"
```
[secret-demo.yaml 내려받기](../files/day4/secret-demo.yaml){ download }

| 부분 | 뜻 |
|---|---|
| `kind: Deployment` | rollout restart는 Deployment에 쓰는 명령이라 Pod 대신 Deployment로 만듭니다. |
| `secretKeyRef` | `mysql-secret`에서 키 `MYSQL_ROOT_PASSWORD`를 골라 같은 이름의 환경변수로 넣습니다. 실습 1의 `configMapKeyRef`와 모양이 같습니다. |

!!! warning "mysql-secret은 실습 5, 6에서 다시 씁니다"
    MySQL 공식 이미지는 `MYSQL_ROOT_PASSWORD` 환경변수로 root 비밀번호를 정합니다. 오늘 만드는 `mysql-secret`을 3교시 MySQL이 그대로 씁니다. 실습이 끝나도 **지우지 마세요.**

## STEP 4. Secret 만들기 (3교시 MySQL에서 다시 씀)

```bash
kubectl create secret generic mysql-secret --from-literal=MYSQL_ROOT_PASSWORD=pass1234
```

```text
secret/mysql-secret created
```

저장된 모습을 봅니다.

```bash
kubectl get secret mysql-secret -o yaml
```

```text
apiVersion: v1
data:
  MYSQL_ROOT_PASSWORD: cGFzczEyMzQ=
kind: Secret
metadata:
  creationTimestamp: "2026-01-01T00:00:00Z"
  name: mysql-secret
  namespace: default
  resourceVersion: "12345"
  uid: 1a2b3c4d-...
type: Opaque
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`pass1234`가 `cGFzczEyMzQ=`로 바뀌어 저장됐습니다. 암호처럼 보이지만 base64 **인코딩**일 뿐입니다.

## STEP 5. base64 풀기

키 없이 명령 한 줄로 원래 값이 나옵니다. OS마다 명령이 다릅니다.

=== "Windows (PowerShell)"

    PowerShell에는 `base64` 명령이 없습니다. .NET 기능으로 풉니다.

    ```powershell
    [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String("cGFzczEyMzQ="))
    ```

=== "Mac (터미널)"

    ```bash
    echo "cGFzczEyMzQ=" | base64 -d
    ```

```text
pass1234
```

**두 OS 공통 방법**도 있습니다. kubectl이 직접 값을 꺼내 풀어 줍니다.

```bash
kubectl get secret mysql-secret -o go-template="{{.data.MYSQL_ROOT_PASSWORD | base64decode}}"
```

```text
pass1234
```

!!! info "Mac에서 끝에 `%`가 붙어 보이면"
    go-template 출력에는 줄바꿈이 없어서, zsh가 `pass1234%`처럼 줄 끝 표시 `%`를 붙입니다. `%`는 값이 아닙니다.

!!! tip "base64 값만 꺼내 보기"
    YAML 전체 대신 값만 보려면 jsonpath를 씁니다. 나온 값을 위 탭 명령의 따옴표 안에 넣어 풀면 됩니다.

    ```bash
    kubectl get secret mysql-secret -o jsonpath="{.data.MYSQL_ROOT_PASSWORD}"
    ```

!!! danger "Secret이라는 이름에 속지 마세요"
    base64는 암호화가 아닙니다. `get secret` 권한이 있으면 누구나 비밀번호를 봅니다. 진짜 보호는 etcd 저장 암호화와 RBAC로 읽을 사람을 제한하는 것입니다. RBAC는 5일차에 합니다.

## STEP 6. Secret 쓰는 Deployment 올리기

```bash
kubectl apply -f secret-demo.yaml
```

```text
deployment.apps/secret-demo created
```

Pod가 `Running`이 되면 컨테이너 안의 환경변수를 봅니다. `deploy/secret-demo`라고 쓰면 Pod 이름을 몰라도 그 Deployment의 Pod 하나에 붙습니다.

```bash
kubectl exec deploy/secret-demo -- printenv MYSQL_ROOT_PASSWORD
```

```text
pass1234
```

컨테이너 안에서는 평문입니다. 앱이 써야 하니 당연합니다.

## STEP 7. 값 바꾸기 (지우고 다시 만들기)

Secret을 지우고 같은 이름으로 새 값 `skilleat`을 넣어 다시 만듭니다.

```bash
kubectl delete secret mysql-secret
```

```bash
kubectl create secret generic mysql-secret --from-literal=MYSQL_ROOT_PASSWORD=skilleat
```

```text
secret "mysql-secret" deleted
secret/mysql-secret created
```

## STEP 8. 떠 있는 Pod는 아직 옛 값

```bash
kubectl exec deploy/secret-demo -- printenv MYSQL_ROOT_PASSWORD
```

```text
pass1234
```

Secret은 바뀌었지만 컨테이너는 모릅니다. 환경변수는 컨테이너가 **시작할 때 한 번만** 읽기 때문입니다.

## STEP 9. Pod를 새로 교체하고 다시 확인

rollout restart로 Pod를 새로 띄웁니다. 이미지나 설정은 그대로 두고, 2일차의 롤링 업데이트 방식으로 Pod만 교체합니다.

```bash
kubectl rollout restart deployment/secret-demo
```

```text
deployment.apps/secret-demo restarted
```

교체가 끝날 때까지 기다립니다.

```bash
kubectl rollout status deployment/secret-demo
```

```text
Waiting for deployment "secret-demo" rollout to finish: 1 old replicas are pending termination...
deployment "secret-demo" successfully rolled out
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`successfully rolled out`을 본 뒤 다시 확인합니다.

```bash
kubectl exec deploy/secret-demo -- printenv MYSQL_ROOT_PASSWORD
```

```text
skilleat
```

!!! warning "3교시 MySQL 비밀번호는 skilleat입니다"
    실습 5, 6에서 `mysql -uroot -pskilleat`으로 접속합니다.

## 성공하면

- STEP 4: `data`에 `MYSQL_ROOT_PASSWORD: cGFzczEyMzQ=`가 보입니다.
- STEP 5, 6: `pass1234`가 나옵니다.
- STEP 8: 값을 바꿨는데도 `pass1234` 그대로입니다.
- STEP 9: `skilleat`으로 바뀝니다.

## 막히면

??? failure "error: failed to create secret secrets \"mysql-secret\" already exists"
    이미 만든 Secret입니다. STEP 4라면 그대로 다음 STEP으로 갑니다. STEP 7이라면 앞의 `kubectl delete secret mysql-secret`을 먼저 했는지 확인합니다.

??? failure "base64 : 'base64' 용어가 cmdlet, 함수... 의 이름으로 인식되지 않습니다 (PowerShell)"
    PowerShell에는 `base64` 명령이 없습니다. STEP 5의 **Windows (PowerShell)** 탭 명령이나 두 OS 공통 go-template 명령을 씁니다.

??? failure "Mac에서 base64 -d 결과가 이상하거나 invalid input"
    값을 복사할 때 앞뒤 글자가 빠졌거나 더 붙었습니다. 끝의 `=`까지 정확히 복사합니다. 직접 인코딩해 볼 때는 `echo -n`을 씁니다. `-n`을 빼면 줄바꿈까지 인코딩되어 `cGFzczEyMzQK`처럼 끝이 달라집니다.

??? failure "STEP 9에서 아직 pass1234가 나옴"
    rollout이 끝나기 전에 exec를 하면 지워지는 옛 Pod에 붙어 옛 값이 나올 수 있습니다. `kubectl rollout status deployment/secret-demo`에서 `successfully rolled out`을 본 뒤 다시 확인합니다.

??? failure "error: unable to upgrade connection: container not found / pod not found"
    Pod가 아직 뜨는 중이거나 교체 중입니다. `kubectl get pods -l app=secret-demo`로 `Running`을 확인한 뒤 다시 합니다.

## 정리

Deployment만 지웁니다.

```bash
kubectl delete deployment secret-demo
```

!!! warning "mysql-secret은 지우지 마세요"
    실습 5, 6의 MySQL이 씁니다.
