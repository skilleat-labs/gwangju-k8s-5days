# 3일차 Service와 Gateway

!!! abstract "오늘이 끝나면 할 수 있는 것"
    - Service로 변하지 않는 주소를 만들고, CoreDNS 덕분에 IP 대신 **이름**으로 접속합니다.
    - 밖에서 접속하는 방법(NodePort, LoadBalancer, port-forward)을 고르고, 연결이 안 될 때 **EndpointSlice**를 보고 원인을 찾습니다.
    - Envoy Gateway로 문 하나를 세우고, HTTPRoute로 `/app1`, `/app2`를 두 앱에 나눠 보냅니다.

어제는 Deployment로 자가복구, 스케일링을 풀었고, Pod를 하나씩 바꾸는 순차 교체 원리를 확인했습니다. 실제 요청이 끊기지 않는지는 Service와 readiness를 붙인 뒤 확인합니다. 남은 한계는 "Pod가 바뀌면 주소도 바뀐다"입니다. 오늘은 하루 종일 이 **주소** 이야기를 합니다.

## 오늘의 실습

| 실습 | 교안 섹션 | STEP | 소요 시간 | 하는 일 |
|---|---|---|---|---|
| [실습 1 Service 붙이기](lab-1.md) | 1 Service | 1~3 | 약 12분 | 어제 web Deployment에 ClusterIP Service를 붙이고 임시 Pod에서 접속 |
| [실습 2 이름으로 찾기 (CoreDNS)](lab-2.md) | 2 이름으로 찾기 | 4~6 | 약 12분 | nslookup으로 이름 풀기, 다른 네임스페이스에서 찾기 |
| [실습 3 밖에서 접속하기](lab-3.md) | 3 외부 노출 | 7~9 | 약 12분 | NodePort, LoadBalancer, port-forward로 같은 nginx 열기 |
| [실습 4 EndpointSlice와 연결 장애](lab-4.md) | 4 EndpointSlice | 10~15 | 약 14분 | 주소 목록 읽기, 늘어나는 주소 보기, 라벨 오타 장애 찾고 고치기 |
| [실습 5 Envoy Gateway 설치](lab-5.md) | 5 Gateway API | 16~18 | 약 12분 | Envoy Gateway 설치, GatewayClass와 Gateway 만들기 |
| [실습 6 HTTPRoute 경로 나누기](lab-6.md) | 6 경로로 나누기 | 19~23 | 약 14분 | 앱 두 개와 HTTPRoute로 `/app1`, `/app2` 나누기 |

!!! tip "실습 4가 오늘 가장 중요합니다"
    "서비스에 접속이 안 돼요"라는 문의의 상당수는 EndpointSlice 하나만 보면 풀립니다. 실습 4의 디버깅 순서(Service → EndpointSlice → Pod → 라벨)는 4일차, 5일차에도 그대로 씁니다.

## 시작 전 확인

Docker Desktop의 Kubernetes가 켜져 있는지 봅니다.

```bash
kubectl get nodes
```

```text
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   2d    v1.3x.x
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`docker-desktop`이 `Ready`가 아니면 [Kubernetes 켜기](../setup/3-kubernetes.md)의 막히면을 봅니다.

## 작업 폴더 만들기

오늘 실습은 모두 `~/k8s-labs/day3` 폴더에서 합니다.

=== "Windows (PowerShell)"

    ```powershell
    mkdir $HOME\k8s-labs\day3; cd $HOME\k8s-labs\day3
    ```

=== "Mac (터미널)"

    ```bash
    mkdir -p ~/k8s-labs/day3 && cd ~/k8s-labs/day3
    ```

터미널을 새로 열 때마다 이 폴더로 다시 이동합니다. 실습 3, 4, 6에서는 터미널을 두 개 씁니다.

=== "Windows (PowerShell)"

    ```powershell
    cd $HOME\k8s-labs\day3
    ```

=== "Mac (터미널)"

    ```bash
    cd ~/k8s-labs/day3
    ```

## 오늘 쓰는 파일

VS Code에서 `day3` 폴더에 직접 만들거나, 아래 링크로 내려받아 `day3` 폴더에 넣습니다. 각 실습 페이지에도 같은 링크가 있습니다.

| 파일 | 실습 | 내용 |
|---|---|---|
| [web-deploy.yaml](../files/day3/web-deploy.yaml){ download } | 1 | 어제 만든 web Deployment (없을 때만) |
| [web-svc.yaml](../files/day3/web-svc.yaml){ download } | 1, 4 | ClusterIP Service `web` |
| [web-np.yaml](../files/day3/web-np.yaml){ download } | 3 | NodePort Service `web-np` |
| [web-svc-broken.yaml](../files/day3/web-svc-broken.yaml){ download } | 4 | 일부러 고장 낸 Service |
| [gateway.yaml](../files/day3/gateway.yaml){ download } | 5 | GatewayClass `eg`, Gateway `eg` |
| [app1.yaml](../files/day3/app1.yaml){ download } | 6 | 테스트 앱 app1 (Deployment, Service) |
| [app2.yaml](../files/day3/app2.yaml){ download } | 6 | 테스트 앱 app2 (Deployment, Service) |
| [route.yaml](../files/day3/route.yaml){ download } | 6 | HTTPRoute `app-route` |

## 어제 web Deployment가 없으면

오늘 실습은 어제 만든 `web` Deployment(nginx, Pod 3개, 라벨 `app: web`)를 그대로 씁니다. 먼저 남아 있는지 봅니다.

```bash
kubectl get deploy web
```

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    3/3     3            3           1d
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`Error from server (NotFound): deployments.apps "web" not found`가 나오면 아래 파일로 다시 만듭니다. 어제 이미지 버전을 바꿨거나 replicas가 3이 아니어도 아래 파일을 apply하면 오늘 실습과 같은 상태가 됩니다.

```yaml title="web-deploy.yaml"
--8<-- "files/day3/web-deploy.yaml"
```
[web-deploy.yaml 내려받기](../files/day3/web-deploy.yaml){ download }

```bash
kubectl apply -f web-deploy.yaml
```

```bash
kubectl get pods -l app=web
```

```text
NAME                   READY   STATUS    RESTARTS   AGE
web-7d9f8c6b5-abcde    1/1     Running   0          20s
web-7d9f8c6b5-fghij    1/1     Running   0          20s
web-7d9f8c6b5-klmno    1/1     Running   0          20s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

세 줄 모두 `1/1 Running`이면 준비가 끝났습니다.

## 오늘 끝나고 정리

실습 6까지 끝났으면 아래를 정리합니다. 각 실습 페이지의 **정리**를 따라왔다면 대부분 이미 지워져 있습니다.

!!! warning "Gateway eg는 오늘 꼭 지웁니다"
    Gateway `eg`는 HTTP **80번** 문을 씁니다. 남겨 두면 5일차 미니 블로그의 Gateway `blog-gw`(역시 80번)와 겹칩니다. 첫 번째 명령이 `eg`의 Gateway와 GatewayClass를 지웁니다. Envoy Gateway **설치**는 지우지 않습니다.

```bash
kubectl delete -f route.yaml -f app1.yaml -f app2.yaml -f gateway.yaml --ignore-not-found
```

```bash
kubectl delete svc web-np web-lb --ignore-not-found
```

```bash
kubectl delete namespace other --ignore-not-found
```

```bash
kubectl scale deployment web --replicas=3
```

!!! warning "지우지 마세요"
    - **Envoy Gateway 설치**(`envoy-gateway-system` 네임스페이스와 Gateway API CRD)는 **5일차 미니 블로그 종합 배포**에서 그대로 씁니다. `install.yaml`로 delete하지 마세요.
    - **web Deployment와 web Service**도 남겨 둡니다 (강의 마지막 장 안내). 4일차 readinessProbe 실습은 `web-probe`라는 새 Deployment를 따로 만들므로 이름이나 라벨이 겹치지 않습니다.

남은 것을 확인합니다.

```bash
kubectl get deploy,svc
```

```bash
kubectl get pods -n envoy-gateway-system
```

```text
NAME                             READY   STATUS    RESTARTS   AGE
envoy-gateway-6c7f5d9b8d-x2k7p   1/1     Running   0          1h
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`envoy-default-eg-`로 시작하는 프록시 Pod는 gateway.yaml을 지우면 함께 사라집니다. 컨트롤러 `envoy-gateway` Pod 하나만 남으면 정상입니다. Gateway가 하나도 안 남았는지도 봅니다.

```bash
kubectl get gateway -A
```

`No resources found`가 나오면 됩니다.
