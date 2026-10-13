# 실습 5. 미니 블로그 종합 배포

!!! abstract "이번 실습에서 할 것"
    - 실습 4의 YAML 두 장으로 미니 블로그 전체를 띄웁니다.
    - 브라우저에서 글을 씁니다.
    - backend Pod를 지워도 서비스가 돌아오고 글이 남는 것(자가복구와 PVC)을 확인합니다.

**소요 시간** 약 15분, **교안** DAY 5 섹션 5

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

3일차에 설치한 Envoy Gateway가 살아 있는지 봅니다.

```bash
kubectl get pods -n envoy-gateway-system
```

`envoy-gateway`로 시작하는 Pod가 `Running`이면 됩니다. 없으면 [3일차 실습 5](../day3/lab-5.md)의 설치를 먼저 합니다.

### 1일차의 한계를 무엇이 푸나

| 1일차 한계 | 오늘 푸는 것 | 확인 |
|---|---|---|
| 한계 1 서버가 죽으면 | 노드가 여러 대면 Deployment가 다른 노드에 다시 띄운다 | 단일 노드라 말로만 |
| 한계 2 사람이 살린다 | Deployment 자가복구 | STEP 22 |
| 한계 3 포트를 바꿔 늘린다 | frontend replicas 2, 포트는 Service 하나 | STEP 24 |
| 한계 4 배포마다 끊긴다 | 롤링 업데이트 (`kubectl set image`) | 2일차 실습 6 |
| 한계 5 주소가 바뀐다 | Service 이름 `backend-service` | STEP 22 |

여기에 1일차에는 없던 것이 하나 더 있습니다. Pod가 바뀌어도 글이 남는 **PVC**입니다.

## STEP 19. backend 파일 먼저 (Namespace 포함)

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

## STEP 20. 다 떴는지

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

## STEP 21. 브라우저 http://localhost → 글 쓰기

Envoy Gateway는 `blog-gw`를 위한 Envoy 프록시 Service를 `envoy-gateway-system`에 만듭니다. 이 Service의 `EXTERNAL-IP`를 봅니다.

```bash
kubectl get svc -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-name=blog-gw
```

```text
NAME                            TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
envoy-webapp-blog-gw-1a2b3c4d   LoadBalancer   10.96.45.120   localhost     80:31234/TCP   1m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`EXTERNAL-IP`가 `localhost`면 브라우저에서 **http://localhost** 를 엽니다. 글을 하나 써서 목록에 나오면 성공입니다.

명령으로 확인하려면 아래처럼 합니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost
    ```

=== "Mac (터미널)"

    ```bash
    curl http://localhost
    ```

!!! note "환경에 따라 다를 수 있음"
    `EXTERNAL-IP`가 `<pending>`이거나 http://localhost 가 열리지 않으면 port-forward로 frontend-service에 직접 붙어 진행합니다. 나머지 실습은 같습니다.

    ```bash
    kubectl port-forward -n webapp svc/frontend-service 8080:80
    ```

    ```text
    Forwarding from 127.0.0.1:8080 -> 80
    Forwarding from [::1]:8080 -> 80
    ```

    이 터미널은 켜 둔 채로 브라우저에서 **http://localhost:8080** 을 엽니다. 다음 STEP의 명령은 **새 터미널**에서 입력합니다.

    `localhost`가 안 열리는 흔한 이유는 3일차의 `eg` Gateway가 남아 있어서 80 포트를 먼저 차지한 경우입니다. 이때 3일차 페이지가 대신 열리거나(404) blog-gw 쪽 `EXTERNAL-IP`가 `<pending>`에 머뭅니다. port-forward로 진행하면 됩니다.

## STEP 22. backend Pod 지우기 (한계 2)

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

## STEP 23. 브라우저 새로고침 → 아까 쓴 글이 그대로

브라우저를 새로고침합니다. **STEP 21에서 쓴 글이 그대로 있습니다.** 글은 Pod가 아니라 PVC(`blog-pvc`)에 저장돼 있기 때문입니다.

backend가 1개라서 새 Pod가 뜨는 몇 초 동안은 글 목록이 안 보이거나 오류가 날 수 있습니다. 몇 초 뒤 다시 새로고침합니다.

## STEP 24. frontend 하나만 지우기 (한계 3)

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

`<frontend Pod 이름 하나>` 자리에 위 목록의 이름 하나를 복사해 넣습니다.

```bash
kubectl delete pod -n webapp <frontend Pod 이름 하나>
```

바로 브라우저를 새로고침합니다. 남은 하나가 요청을 받으므로 **화면이 끊기지 않습니다.** 이것이 replicas를 2 이상으로 두는 이유입니다. 1일차처럼 포트를 바꿔 하나 더 띄울 필요도 없습니다.

!!! note "port-forward로 진행 중이라면"
    `kubectl port-forward svc/...`는 Service 뒤의 Pod **하나**에 붙습니다. 지운 Pod가 그 Pod였다면 port-forward 터미널에 `lost connection to pod`가 나오고 끊깁니다. port-forward 명령을 다시 실행하면 됩니다. Gateway(http://localhost)로 들어오면 이런 끊김이 없습니다.

## 성공하면

- Pod 3개 `Running`, PVC `Bound`
- Gateway `PROGRAMMED True`
- 글 등록 후 목록에 보임
- 새 backend Pod가 뜨고, 글이 사라지지 않는다
- frontend는 끊김 없음

## 막히면

??? failure "Error from server (NotFound): error when creating \"miniblog-frontend.yaml\": namespaces \"webapp\" not found"
    frontend 파일을 먼저 적용했습니다. `kubectl apply -f miniblog-backend.yaml`을 먼저 하고 frontend를 다시 적용합니다.

??? failure "backend Pod와 PVC가 Pending"
    PVC가 볼륨을 못 얻으면 그 PVC를 쓰는 Pod도 따라서 Pending입니다. `kubectl describe pvc blog-pvc -n webapp`의 Events를 봅니다. `storageClassName`에 없는 이름을 썼는지, `claimName`이 `blog-pvc`와 같은지 확인합니다. 실습 6에서 이 장애를 직접 고칩니다.

??? failure "Gateway PROGRAMMED가 False 또는 Unknown"
    1. `kubectl get gatewayclass`에서 `blog-gw-class`의 `ACCEPTED`가 `True`인지 봅니다. `False`나 비어 있으면 `controllerName` 오타입니다. `gateway.envoyproxy.io/gatewayclass-controller`와 한 글자씩 비교합니다.
    2. `kubectl get pods -n envoy-gateway-system`으로 Envoy Gateway 컨트롤러가 살아 있는지 봅니다.
    3. 시간이 해결하지 않으면 port-forward로 진행합니다.

??? failure "화면은 뜨는데 글 등록만 실패"
    backend Service 이름이 `backend-service`가 아니거나, selector가 backend Pod 라벨과 다릅니다. frontend는 정확히 `backend-service:5000`으로 백엔드를 찾습니다.

    ```bash
    kubectl get svc,endpointslices -n webapp
    ```

    `backend-service`가 있고 EndpointSlice에 주소가 있는지 봅니다.

??? failure "STEP 23에서 글이 사라짐"
    (3) `mountPath: /app/data`와 (4) `claimName: blog-pvc`를 확인합니다. mountPath가 다르면 글이 PVC가 아니라 컨테이너 안에 저장되어 Pod와 함께 사라집니다. 고친 뒤 `kubectl apply -f miniblog-backend.yaml`로 다시 적용합니다.

??? failure "Unable to listen on port 8080: ... bind: address already in use"
    8080을 다른 프로그램(1일차 컨테이너 등)이 쓰고 있습니다. `docker ps`로 8080을 쓰는 컨테이너가 있으면 지우거나, 포트를 바꿉니다: `kubectl port-forward -n webapp svc/frontend-service 8081:80` 후 http://localhost:8081

??? failure "Pod가 ImagePullBackOff"
    이미지 이름과 태그(`skilleat/backend:v3-kb5`, `skilleat/frontend:v3-kb5`)를 한 글자씩 비교합니다. 실습 6에서 이 장애도 직접 고칩니다.

## 정리

**지우지 마세요.** 실습 6에서 고장 난 블로그와 비교할 때 씁니다. 정리는 실습 6 끝에서 한꺼번에 합니다.

port-forward를 켜 두었다면 그 터미널에서 `Ctrl + C`로 멈춰도 됩니다.
