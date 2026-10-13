# 실습 5. Envoy Gateway 설치, Gateway 만들기

!!! abstract "이번 실습에서 할 것"
    - Gateway API 구현체인 Envoy Gateway를 설치합니다.
    - 인프라팀 역할로 GatewayClass와 Gateway(문)를 만듭니다.
    - 컨트롤러가 Envoy 프록시를 자동으로 만들어 주는 것을 봅니다.

**소요 시간** 약 12분, **교안** DAY 3 섹션 5

## 준비

- 작업 폴더로 이동합니다.
- 설치할 때 이미지를 내려받으므로 인터넷이 필요합니다. 강의 설명 중에 STEP 16을 미리 실행해 두어도 됩니다.

오늘 만들 것은 아래 세 단계입니다. 이 실습에서 앞의 둘을, 실습 6에서 HTTPRoute를 만듭니다.

| 리소스 | 질문 | 오늘 값 | 누가 |
|---|---|---|---|
| GatewayClass | 어떤 구현체가 맡나? | Envoy Gateway (`eg`) | 인프라팀 |
| Gateway | 어떤 문을 여나? | HTTP 80번 (`eg`) | 인프라팀 |
| HTTPRoute | 어디로 보내나? | `/app1` → app1, `/app2` → app2 | 앱팀 (실습 6) |

## STEP 16. 설치 (Gateway API CRD 포함) 후 대기

공식 `install.yaml`에는 Gateway API CRD와 Envoy Gateway 컨트롤러가 함께 들어 있습니다. 파일이 커서 `--server-side`로 적용합니다.

```bash
kubectl apply --server-side -f https://github.com/envoyproxy/gateway/releases/download/v1.9.2/install.yaml
```

```text
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io serverside-applied
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io serverside-applied
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io serverside-applied
...
namespace/envoy-gateway-system serverside-applied
...
deployment.apps/envoy-gateway serverside-applied
```
<small>출력은 예시이며 일부를 줄였습니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

줄이 수십 개 지나갑니다. 끝까지 오류(`error`) 없이 끝나면 됩니다. 이어서 컨트롤러가 준비될 때까지 기다립니다 (최대 5분).

```bash
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

```text
deployment.apps/envoy-gateway condition met
```

!!! info "버전"
    이 과정은 Envoy Gateway **v1.9.2**를 씁니다 (2026-10-08 공식 문서 기준). 다른 버전 주소를 쓰면 화면 출력이 조금 다를 수 있습니다.

## STEP 17. 문 만들기 (인프라팀 역할)

GatewayClass와 Gateway를 `---`로 이어 붙인 파일을 `gateway.yaml`로 저장합니다.

```yaml title="gateway.yaml" hl_lines="6 13"
--8<-- "files/day3/gateway.yaml"
```
[gateway.yaml 내려받기](../files/day3/gateway.yaml){ download }

| 줄 | 뜻 |
|---|---|
| `controllerName` | Envoy Gateway가 이 클래스를 맡는다는 표시. 정해진 값을 그대로 씁니다. **오타가 나면 아무도 맡지 않습니다** |
| `gatewayClassName: eg` | 위에서 만든 GatewayClass 이름 |
| `listeners` | HTTP 80번 문을 엽니다 |

```bash
kubectl apply -f gateway.yaml
```

```text
gatewayclass.gateway.networking.k8s.io/eg created
gateway.gateway.networking.k8s.io/eg created
```

```bash
kubectl get gatewayclass,gateway
```

```text
NAME                                         CONTROLLER                                      ACCEPTED   AGE
gatewayclass.gateway.networking.k8s.io/eg    gateway.envoyproxy.io/gatewayclass-controller   True       15s

NAME                                   CLASS   ADDRESS     PROGRAMMED   AGE
gateway.gateway.networking.k8s.io/eg   eg      localhost   True         15s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

| 칸 | 뜻 |
|---|---|
| GatewayClass `ACCEPTED True` | Envoy Gateway 컨트롤러가 이 클래스를 맡았습니다 |
| Gateway `PROGRAMMED True` | 문이 준비되었습니다 (프록시가 설정됨) |
| `ADDRESS` | 문의 주소. Docker Desktop에서는 `localhost`가 들어올 수 있습니다 |

처음에는 `PROGRAMMED`가 `Unknown`이나 `False`일 수 있습니다. 프록시 이미지를 내려받는 동안이니 30초쯤 뒤 다시 실행합니다.

## STEP 18. 컨트롤러가 만든 Envoy 프록시 보기

STEP 16 직후에는 `envoy-gateway` 컨트롤러 Pod 하나뿐이었습니다. 지금 다시 봅니다.

```bash
kubectl get pods,svc -n envoy-gateway-system
```

```text
NAME                                            READY   STATUS    RESTARTS   AGE
pod/envoy-default-eg-e41e7b31-6f9c8d7b5-k2x9p   2/2     Running   0          1m
pod/envoy-gateway-6c7f5d9b8d-x2k7p              1/1     Running   0          5m

NAME                                TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                   AGE
service/envoy-default-eg-e41e7b31   LoadBalancer   10.97.204.11    localhost     80:31480/TCP                              1m
service/envoy-gateway               ClusterIP      10.106.53.88    <none>        18000/TCP,18001/TCP,18002/TCP,19001/TCP   5m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

**`envoy-default-eg-`로 시작하는 Pod와 Service**가 새로 생겼습니다. 이름은 `envoy-<Gateway 네임스페이스>-<Gateway 이름>-<꼬리>` 모양입니다. 우리는 gateway.yaml 한 장만 적용했는데, 컨트롤러가 Gateway를 보고 실제 문 역할을 할 Envoy 프록시 Deployment와 Service를 만들어 준 것입니다.

| 이름 | 역할 |
|---|---|
| `envoy-gateway` | 컨트롤러. Gateway, HTTPRoute를 감시하고 프록시를 설정합니다 |
| `envoy-default-eg-...` | Envoy 프록시. **실제 사용자 요청이 지나가는 문**입니다 |

!!! note "환경에 따라 다를 수 있음"
    Envoy 프록시 Service는 기본이 `LoadBalancer` 타입입니다. Docker Desktop에서 `EXTERNAL-IP`가 `localhost`로 채워질 수도 있고, `<pending>`으로 남을 수도 있습니다. 어느 쪽이든 실습 6에서는 **port-forward**로 접속하므로 그대로 진행합니다.

## 성공하면

- STEP 16: `deployment.apps/envoy-gateway condition met`
- STEP 17: GatewayClass `ACCEPTED True`, Gateway `PROGRAMMED True`
- STEP 18: `envoy-default-eg-…`로 시작하는 Pod(2/2 Running)와 Service가 생겼습니다.

## 막히면

??? failure "error: resource mapping not found ... no matches for kind \"Gateway\" in version \"gateway.networking.k8s.io/v1\""
    Gateway API CRD가 아직 없습니다. STEP 16의 설치를 먼저 하고 다시 `kubectl apply -f gateway.yaml`을 합니다.

??? failure "metadata.annotations: Too long: must have at most 262144 bytes"
    설치 명령에서 `--server-side`를 빠뜨렸습니다. 파일이 커서 일반 apply로는 들어가지 않습니다. STEP 16 명령을 그대로 복사해 다시 실행합니다.

??? failure "error: timed out waiting for the condition on deployments/envoy-gateway"
    5분 안에 컨트롤러가 준비되지 않았습니다. 이미지를 받는 중일 수 있습니다.

    ```bash
    kubectl get pods -n envoy-gateway-system
    ```

    `ContainerCreating`이면 조금 더 기다렸다가 STEP 16의 `kubectl wait`를 다시 실행합니다. `ImagePullBackOff`나 `ErrImagePull`이면 네트워크 문제입니다. 회사 네트워크라면 휴대폰 핫스팟으로 바꿔 시도합니다.

??? failure "Unable to connect to the server ... github.com (또는 설치 주소 다운로드 실패)"
    install.yaml을 GitHub에서 받지 못했습니다. 브라우저로 설치 주소를 열어 파일이 받아지는지 확인합니다. 회사 프록시나 방화벽이 GitHub를 막으면 강사에게 알려 주세요.

??? failure "PROGRAMMED가 계속 False"
    1. GatewayClass의 `ACCEPTED`도 False거나 비어 있으면 **controllerName 오타**입니다. `gateway.envoyproxy.io/gatewayclass-controller`를 그대로 복사해 고치고 다시 apply합니다.
    2. ACCEPTED는 True인데 PROGRAMMED만 False면 이유를 봅니다.

        ```bash
        kubectl describe gateway eg
        ```

        `Conditions`의 `Reason`이 `AddressNotAssigned`처럼 주소 관련이면, LoadBalancer 주소를 받지 못한 환경 차이입니다. 프록시 Pod가 Running이면 실습 6의 port-forward로 그대로 진행할 수 있습니다.

??? failure "envoy-default-eg-... Pod가 보이지 않음"
    Gateway가 아직 처리되지 않았습니다. 30초쯤 뒤 다시 봅니다. 계속 없으면 위 PROGRAMMED False 항목을 확인합니다.

## 정리

!!! warning "지우지 마세요"
    - Gateway `eg`와 GatewayClass `eg`는 **실습 6**에서 씁니다.
    - Envoy Gateway 설치(`envoy-gateway-system`)는 **5일차 미니 블로그 종합 배포**에서도 씁니다. 오늘이 끝나도 지우지 않습니다.
