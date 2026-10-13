# 실습 5. 미니 블로그 종합 배포

!!! abstract "이번 실습에서 할 것"
    - 실습 4의 YAML 두 장으로 미니 블로그 전체를 띄우고, **Gateway를 거쳐** 브라우저로 글을 씁니다.
    - backend Pod, frontend Pod를 지우며 자가복구, 데이터 보존, 요청 연속성을 **따로따로** 확인합니다.
    - 4일차와 오늘 배운 운영, 보안 설정(readinessProbe, requests와 limits, 토큰 끄기)을 붙인 v2를 적용합니다.

**소요 시간** 약 23분, **교안** DAY 5 섹션 5

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

실습 4에서 만든 `miniblog-backend.yaml`, `miniblog-frontend.yaml`이 작업 폴더에 있어야 합니다. 빈칸을 다 못 채웠다면 정답 파일을 받아 이름을 바꿔 씁니다.

[miniblog-backend-answer.yaml 내려받기](../files/day5/miniblog-backend-answer.yaml){ download }

[miniblog-frontend-answer.yaml 내려받기](../files/day5/miniblog-frontend-answer.yaml){ download }

```bash
mv miniblog-backend-answer.yaml miniblog-backend.yaml
```

```bash
mv miniblog-frontend-answer.yaml miniblog-frontend.yaml
```

STEP 28에서 쓸 v2 파일 두 개도 받아 둡니다.

[miniblog-backend-v2.yaml 내려받기](../files/day5/miniblog-backend-v2.yaml){ download }

[miniblog-frontend-v2.yaml 내려받기](../files/day5/miniblog-frontend-v2.yaml){ download }

3일차에 설치한 Envoy Gateway가 살아 있는지 봅니다.

```bash
kubectl get pods -n envoy-gateway-system
```

`envoy-gateway`로 시작하는 Pod가 `Running`이면 됩니다. 없으면 [3일차 실습 5](../day3/lab-5.md)의 설치를 먼저 합니다.

### 1일차의 한계를 무엇이 푸나

| 1일차 한계 | 오늘 푸는 것 | 확인 |
|---|---|---|
| 한계 1 서버가 죽으면 | 노드가 여러 대면 Deployment가 다른 노드에 다시 띄운다 | 단일 노드라 말로만 |
| 한계 2 사람이 살린다 | Deployment 자가복구 | STEP 26 |
| 한계 3 포트를 바꿔 늘린다 | frontend replicas 2, 입구는 Service와 Gateway 하나 | STEP 27 |
| 한계 4 배포마다 사람이 바꾼다 | 선언한 대로 자동 교체 (frontend는 롤링 업데이트, backend는 Recreate) | STEP 28 |
| 한계 5 주소가 바뀐다 | Service 이름 `backend-service` | STEP 26 |

여기에 1일차에는 없던 것이 하나 더 있습니다. Pod가 바뀌어도 글이 남는 **PVC**입니다.

### 성공 조건 4가지는 따로 확인합니다

| 성공 조건 | 확인하는 방법 | 주의 |
|---|---|---|
| ① Gateway 경유 접속 | Envoy 프록시에 port-forward한 **localhost:8888** 로 열린다 (STEP 25) | 입구(Gateway)와 경로(HTTPRoute)까지 확인됩니다 |
| ② 앱 기능 | 글을 쓰고 목록에 보인다 (STEP 25) | frontend-service에 직접 port-forward해도 확인되지만, 그때는 Gateway를 **우회**하므로 ①은 증명되지 않습니다 |
| ③ backend 삭제 | 새 Pod가 뜨고 글이 남는다 (STEP 26) | backend가 1개라 새 Pod가 뜨는 몇 초 동안은 요청이 **실패할 수 있습니다** |
| ④ frontend 삭제 | Gateway 경유 요청이 남은 Pod로 계속 간다 (STEP 27) | 직접 port-forward는 Pod 하나에 붙어서, 그 Pod를 지우면 연결이 끊길 수 있습니다 |

복구, 데이터 보존, 무중단은 서로 다른 성공입니다.

## STEP 20. 3일차의 eg가 남아 있으면 지운다

3일차 실습 6 끝에서 `eg` Gateway와 GatewayClass를 지웠습니다. 남아 있는지 봅니다.

```bash
kubectl get gateway -A
```

```text
No resources found
```

`eg`가 보이면 지웁니다. 다른 Gateway는 그대로 둡니다.

```bash
kubectl delete gateway eg
```

```bash
kubectl delete gatewayclass eg
```

## STEP 21. backend 파일 먼저 (Namespace 포함)

Namespace `webapp`이 backend 파일 맨 앞에 있으니 backend 파일을 먼저 적용합니다.

```bash
kubectl apply -f miniblog-backend.yaml
```

```bash
kubectl apply -f miniblog-frontend.yaml
```

```text
namespace/webapp created
persistentvolumeclaim/blog-pvc created
deployment.apps/backend created
service/backend-service created
deployment.apps/frontend created
service/frontend-service created
gatewayclass.gateway.networking.k8s.io/blog-gw-class created
gateway.gateway.networking.k8s.io/blog-gw created
httproute.gateway.networking.k8s.io/frontend-route created
```

## STEP 22. 다 떴는지

```bash
kubectl get pods,svc,pvc -n webapp
```

```text
NAME                            READY   STATUS    RESTARTS   AGE
pod/backend-6d9c7b8f5d-k2x7p    1/1     Running   0          40s
pod/frontend-7f6b9c4d8-9qwer    1/1     Running   0          35s
pod/frontend-7f6b9c4d8-zx3mn    1/1     Running   0          35s

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/backend-service    ClusterIP   10.96.120.15    <none>        5000/TCP   40s
service/frontend-service   ClusterIP   10.96.88.201    <none>        80/TCP     35s

NAME                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/blog-pvc   Bound    pvc-3f1e2d4c-5b6a-4c7d-8e9f-0a1b2c3d4e5f   1Gi        RWO            hostpath       40s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

Pod 3개(frontend 2, backend 1)가 `Running`, PVC가 `Bound`인지 봅니다. 처음에는 이미지를 받느라 `ContainerCreating`이 잠시 보일 수 있습니다.

```bash
kubectl get gateway,httproute -n webapp
```

```text
NAME                                        CLASS           ADDRESS     PROGRAMMED   AGE
gateway.gateway.networking.k8s.io/blog-gw   blog-gw-class   localhost   True         35s

NAME                                                 HOSTNAMES   AGE
httproute.gateway.networking.k8s.io/frontend-route               35s
```
<small>출력은 예시입니다. ADDRESS는 비어 있거나 다른 값일 수 있습니다.</small>

Gateway의 `PROGRAMMED`가 `True`면 입구가 준비된 것입니다.

## STEP 23. blog-gw의 Envoy 프록시 Service 찾기

3일차 실습 6과 같은 방법입니다. Envoy Gateway는 `blog-gw` 문을 위해 `envoy-gateway-system`에 Envoy 프록시 Service를 만듭니다. 라벨 두 개(Gateway 이름 `blog-gw`, Gateway 네임스페이스 `webapp`)로 찾습니다. 이름만으로 찾으면 다른 네임스페이스에 같은 이름의 Gateway가 있을 때 둘 다 잡힙니다.

```bash
kubectl get svc -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-name=blog-gw,gateway.envoyproxy.io/owning-gateway-namespace=webapp
```

```text
NAME                               TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
envoy-webapp-blog-gw-1a2b3c4d      LoadBalancer   10.96.45.120   localhost     80:31234/TCP   1m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

이름이 `envoy-webapp-blog-gw-`로 시작하고 뒤에 꼬리가 붙어 있습니다. 사람마다 꼬리가 다르니 **내 화면의 NAME**을 씁니다.

## STEP 24. 새 터미널에서 port-forward

**새 터미널**을 열고 작업 폴더로 이동한 뒤, 내 PC의 8888 포트를 Envoy 프록시 Service의 80번에 연결합니다. 방법은 두 가지 중 편한 것을 씁니다.

**방법 1. 이름을 직접 붙여 넣기** (두 OS 공통)

아래 `envoy-webapp-blog-gw-1a2b3c4d`는 예시이니 **STEP 23의 내 이름으로 바꿉니다.**

```bash
kubectl -n envoy-gateway-system port-forward svc/envoy-webapp-blog-gw-1a2b3c4d 8888:80
```

**방법 2. 변수에 담아 쓰기**

변수는 **port-forward를 실행할 바로 그 새 터미널**에서 만들어야 합니다 (다른 터미널에서 만든 변수는 보이지 않습니다).

=== "Windows (PowerShell)"

    ```powershell
    $SVC = kubectl get svc -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-name=blog-gw,gateway.envoyproxy.io/owning-gateway-namespace=webapp -o name
    ```

    ```powershell
    $SVC
    ```

    ```powershell
    kubectl -n envoy-gateway-system port-forward $SVC 8888:80
    ```

=== "Mac (터미널)"

    ```bash
    export ENVOY_SERVICE=$(kubectl get svc -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-name=blog-gw,gateway.envoyproxy.io/owning-gateway-namespace=webapp -o name)
    ```

    ```bash
    echo $ENVOY_SERVICE
    ```

    ```bash
    kubectl -n envoy-gateway-system port-forward $ENVOY_SERVICE 8888:80
    ```

가운데 명령은 변수에 이름이 잘 들어갔는지 확인하는 것입니다. `service/envoy-webapp-blog-gw-...`가 한 줄 나오면 됩니다.

```text
Forwarding from 127.0.0.1:8888 -> 10080
Forwarding from [::1]:8888 -> 10080
```
<small>출력은 예시입니다. 오른쪽 번호는 Envoy 컨테이너의 실제 포트라 80이 아닐 수 있습니다.</small>

이 터미널은 **켜 둔 채로** 둡니다. 다음 STEP의 명령은 원래 터미널에서 입력합니다.

## STEP 25. http://localhost:8888 → 글 쓰기

브라우저에서 **http://localhost:8888** 을 엽니다. 글을 하나 써서 목록에 나오면 성공 조건 ①과 ②를 확인한 것입니다.

명령으로 확인하려면 원래 터미널에서 아래처럼 합니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost:8888
    ```

=== "Mac (터미널)"

    ```bash
    curl http://localhost:8888
    ```

!!! note "앱 기능만 따로 볼 때 (Gateway 우회)"
    Gateway 쪽이 막혀서 원인을 나누고 싶으면, frontend-service에 직접 붙어 앱만 봅니다. 이 방법은 Gateway와 HTTPRoute를 거치지 않으므로 **성공 조건 ②만** 확인됩니다.

    ```bash
    kubectl port-forward -n webapp svc/frontend-service 8080:80
    ```

    브라우저에서 **http://localhost:8080** 을 엽니다. 이 port-forward는 frontend Pod **하나**에 붙습니다.

!!! note "환경에 따라 다를 수 있음"
    STEP 23에서 `EXTERNAL-IP`가 `localhost`였다면 port-forward 없이 http://localhost 로도 열릴 수 있습니다. 실습은 모두 같은 결과를 보도록 **localhost:8888** 기준으로 진행합니다.

## STEP 26. backend Pod 지우기 (성공 조건 ③)

backend Pod를 지우고, 무슨 일이 일어나는지 지켜봅니다.

```bash
kubectl delete pod -n webapp -l app=backend
```

```bash
kubectl get pods -n webapp -w
```

```text
NAME                       READY   STATUS              RESTARTS   AGE
backend-6d9c7b8f5d-k2x7p   1/1     Terminating         0          5m
backend-6d9c7b8f5d-p8v4n   0/1     ContainerCreating   0          1s
frontend-7f6b9c4d8-9qwer   1/1     Running             0          5m
frontend-7f6b9c4d8-zx3mn   1/1     Running             0          5m
backend-6d9c7b8f5d-p8v4n   1/1     Running             0          3s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

Deployment가 곧바로 새 backend Pod를 만듭니다. 새 Pod의 IP는 바뀌었지만 frontend는 `backend-service`라는 이름으로 찾으므로 그대로 연결됩니다 (한계 5). 다 보았으면 `Ctrl + C`로 빠져나옵니다.

브라우저(localhost:8888)를 새로고침합니다. **아까 쓴 글이 그대로 있습니다.** 글은 Pod가 아니라 PVC(`blog-pvc`)에 저장돼 있기 때문입니다.

!!! warning "몇 초 실패는 정상입니다"
    backend가 1개라서 새 Pod가 뜨는 몇 초 동안은 글 목록이 안 보이거나 오류가 날 수 있습니다. 이것은 **복구**와 **데이터 보존**은 성공했지만 **무중단**은 아니라는 뜻입니다. 몇 초 뒤 다시 새로고침합니다.

## STEP 27. frontend 하나 지우기 (성공 조건 ④)

frontend Pod 이름을 보고, 둘 중 하나만 지웁니다.

```bash
kubectl get pods -n webapp -l app=frontend
```

```text
NAME                       READY   STATUS    RESTARTS   AGE
frontend-7f6b9c4d8-9qwer   1/1     Running   0          8m
frontend-7f6b9c4d8-zx3mn   1/1     Running   0          8m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`<frontend Pod 하나>` 자리에 위 목록의 이름 하나를 복사해 넣습니다.

```bash
kubectl delete pod -n webapp <frontend Pod 하나>
```

바로 **localhost:8888**(Gateway 경유)을 새로고침합니다. Envoy가 frontend-service 뒤의 남은 Pod로 보내므로 화면이 계속 나옵니다. 이것이 replicas를 2 이상으로 두는 이유입니다.

!!! note "frontend에 직접 port-forward 중이었다면"
    `kubectl port-forward svc/frontend-service ...`는 Service 뒤의 Pod **하나**에 붙습니다. 지운 Pod가 그 Pod였다면 port-forward 터미널에 `lost connection to pod`가 나오고 끊깁니다. Gateway 경유(8888) 쪽은 Envoy 프록시 Pod에 붙어 있어서 frontend를 지워도 끊기지 않습니다.

## STEP 28. 운영, 보안 설정을 붙인 v2 적용

기본 배포가 되었으니 배운 설정을 붙입니다. v2 파일에는 Deployment만 들어 있고, 바뀐 부분은 아래와 같습니다.

| 설정 | 하는 일 | 배운 곳 |
|---|---|---|
| `automountServiceAccountToken: false` | API 서버를 쓰지 않는 앱에 토큰을 꽂지 않는다 | 5일차 실습 2 |
| `readinessProbe` | 준비된 Pod에만 요청을 보낸다. frontend는 `/`, backend는 5000 포트 | 4일차 실습 3 |
| `resources` | 배치 기준(requests)과 메모리 한도(limits) | 4일차 실습 4 |
| `allowPrivilegeEscalation: false` | 실행 중 권한 상승을 막는다 | 5일차 실습 1 |

```yaml title="miniblog-frontend-v2.yaml"
--8<-- "files/day5/miniblog-frontend-v2.yaml"
```

```yaml title="miniblog-backend-v2.yaml"
--8<-- "files/day5/miniblog-backend-v2.yaml"
```

!!! info "runAsNonRoot는 왜 없나"
    `runAsNonRoot: true`는 이미지가 root 없이 돌도록 만들어졌을 때만 붙입니다. 기본 nginx 계열 이미지는 root로 시작하도록 만들어져 있어서, 이 줄만 붙이면 시작이 거부되거나 80 포트, 캐시 폴더 권한 때문에 죽을 수 있습니다. 또 `readOnlyRootFilesystem: true`는 컨테이너 기본 파일 시스템만 읽기 전용으로 만들고, `/app/data`처럼 따로 마운트한 쓰기 볼륨은 막지 않습니다. 이미지가 다른 곳(임시 폴더 등)에 쓰는지 확인한 뒤 붙입니다.

적용합니다. Pod 템플릿이 바뀌었으니 Pod가 새 이름으로 교체됩니다. frontend는 2일차의 롤링 업데이트로 하나씩 바뀌고, backend는 `strategy: Recreate`라서 옛 Pod를 먼저 내린 뒤 새 Pod를 띄웁니다. 그래서 backend가 바뀌는 몇 초 동안은 글 요청이 실패할 수 있습니다.

```bash
kubectl apply -f miniblog-backend-v2.yaml
```

```bash
kubectl apply -f miniblog-frontend-v2.yaml
```

```bash
kubectl rollout status deployment/frontend -n webapp
```

```text
deployment "frontend" successfully rolled out
```

## STEP 29. 붙었는지 확인

```bash
kubectl get pods -n webapp
```

모든 Pod가 새 이름으로 바뀌고 `READY`가 `1/1`이면 readinessProbe를 통과한 것입니다.

frontend 안에 토큰 파일이 없는지 봅니다.

```bash
kubectl exec -n webapp deploy/frontend -- ls /var/run/secrets/kubernetes.io/serviceaccount
```

```text
ls: cannot access '/var/run/secrets/kubernetes.io/serviceaccount': No such file or directory
command terminated with exit code 2
```
<small>출력은 예시입니다. 이미지에 따라 문구가 조금 다를 수 있습니다.</small>

마지막으로 localhost:8888을 새로고침해 **글이 그대로** 있는지 봅니다. 8888이 끊겨 있으면 STEP 24의 port-forward를 다시 실행합니다.

!!! info "backend는 왜 Recreate인가"
    backend는 글을 파일 하나에 쓰고, PVC는 `ReadWriteOnce`입니다. 기본값인 롤링 업데이트는 새 Pod를 먼저 띄우므로 잠시 두 Pod가 같은 파일에 쓸 수 있습니다. `Recreate`는 옛 Pod를 먼저 내려서 이 겹침을 없애는 대신, 업데이트 중 잠깐 API가 멈춥니다. 새 Pod가 오래 `ContainerCreating`이면 `kubectl describe pod -n webapp -l app=backend`의 Events를 보고 강사에게 알려 주세요.

## 성공하면

- Pod 3개 `Running`, PVC `Bound`, Gateway `PROGRAMMED True`
- ① localhost:8888(Gateway 경유)로 열린다
- ② 글 등록 후 목록에 보인다
- ③ backend를 지워도 새 Pod가 뜨고 글이 남는다 (몇 초 실패는 정상)
- ④ frontend 하나를 지워도 8888 화면이 계속 나온다
- v2 적용 후 READY `1/1`, 토큰 파일 없음, 글 그대로

## 막히면

??? failure "Error from server (NotFound): error when creating \"miniblog-frontend.yaml\": namespaces \"webapp\" not found"
    frontend 파일을 먼저 적용했습니다. `kubectl apply -f miniblog-backend.yaml`을 먼저 하고 frontend를 다시 적용합니다.

??? failure "backend Pod와 PVC가 Pending"
    PVC가 볼륨을 못 얻으면 그 PVC를 쓰는 Pod도 따라서 Pending입니다. `kubectl describe pvc blog-pvc -n webapp`의 Events를 봅니다. `storageClassName`에 없는 이름을 썼는지, `claimName`이 `blog-pvc`와 같은지 확인합니다. 실습 6에서 이 장애를 직접 고칩니다.

??? failure "Gateway PROGRAMMED가 False 또는 Unknown"
    1. `kubectl get gatewayclass`에서 `blog-gw-class`의 `ACCEPTED`가 `True`인지 봅니다. `False`나 비어 있으면 `controllerName` 오타입니다. `gateway.envoyproxy.io/gatewayclass-controller`와 한 글자씩 비교합니다.
    2. `kubectl get pods -n envoy-gateway-system`으로 Envoy Gateway 컨트롤러와 blog-gw 프록시 Pod가 살아 있는지 봅니다.
    3. `kubectl describe gateway blog-gw -n webapp`의 Conditions를 봅니다. 주소 관련 이유(LoadBalancer 주소를 못 받음)라면, 프록시 Pod가 Running인 한 STEP 24의 port-forward로 그대로 진행할 수 있습니다.

??? failure "STEP 23에서 No resources found"
    blog-gw의 프록시가 아직 만들어지지 않았습니다. STEP 22의 `PROGRAMMED`부터 다시 확인합니다. 라벨 값이 `blog-gw`, `webapp`인지(오타, `eg`, `default`로 남아 있지 않은지)도 봅니다.

??? failure "error: unable to forward port ... / 변수가 비어 있음"
    방법 2에서 변수가 비었거나 이름이 두 개 이상 들어갔습니다. `$SVC`(Mac은 `echo $ENVOY_SERVICE`)를 실행해 `service/envoy-webapp-blog-gw-...` 한 줄만 나오는지 봅니다. 아무것도 안 나오면 변수를 **port-forward를 실행하는 그 터미널**에서 다시 만들거나, 방법 1로 이름을 직접 붙여 넣습니다.

??? failure "localhost:8888 에서 404"
    Envoy까지는 왔지만 맞는 경로 규칙이 없습니다. `kubectl get httproute -n webapp`으로 `frontend-route`가 있는지, `parentRefs`가 `blog-gw`인지 봅니다. 3일차의 `eg` 쪽 Service에 port-forward했을 수도 있습니다. STEP 23의 라벨이 `blog-gw`, `webapp`인지 확인합니다.

??? failure "화면은 뜨는데 글 등록만 실패"
    backend Service 이름이 `backend-service`가 아니거나, selector가 backend Pod 라벨과 다릅니다. frontend는 정확히 `backend-service:5000`으로 백엔드를 찾습니다.

    ```bash
    kubectl get svc,endpointslices -n webapp
    ```

    `backend-service`가 있고 EndpointSlice에 주소가 있는지 봅니다.

??? failure "STEP 26에서 글이 사라짐"
    (3) `mountPath: /app/data`와 (4) `claimName: blog-pvc`를 확인합니다. mountPath가 다르면 글이 PVC가 아니라 컨테이너 안에 저장되어 Pod와 함께 사라집니다. 고친 뒤 `kubectl apply -f miniblog-backend.yaml`로 다시 적용합니다.

??? failure "STEP 29에서 READY 0/1 이 오래 감, 또는 OOMKilled"
    `kubectl describe pod -n webapp <Pod 이름>`의 Events와 `Last State`를 봅니다. `Readiness probe failed`면 probe 경로나 포트가 앱과 맞지 않는 것이고, `Reason: OOMKilled`면 메모리 한도가 작은 것입니다. 원래대로 돌리려면 `kubectl apply -f miniblog-frontend.yaml`(backend는 `miniblog-backend.yaml`)을 다시 적용하고 강사에게 알려 주세요.

??? failure "Unable to listen on port 8888: ... bind: address already in use"
    8888을 다른 프로그램(3일차 port-forward 등)이 쓰고 있습니다. 그 터미널에서 `Ctrl + C`로 멈추거나, `8889:80`처럼 앞 숫자를 바꾸고 http://localhost:8889 로 엽니다.

??? failure "Pod가 ImagePullBackOff"
    이미지 이름과 태그(`skilleat/backend:v3-kb5`, `skilleat/frontend:v3-kb5`)를 한 글자씩 비교합니다. 실습 6에서 이 장애도 직접 고칩니다.

## 정리

**지우지 마세요.** 실습 6에서 고장 난 블로그와 비교할 때 씁니다. 정리는 실습 6 끝에서 한꺼번에 합니다.

port-forward 터미널은 `Ctrl + C`로 멈춰도 됩니다.
