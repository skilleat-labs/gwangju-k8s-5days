# 실습 6. 앱 두 개와 HTTPRoute로 경로 나누기

!!! abstract "이번 실습에서 할 것"
    - 테스트 앱 두 개(app1, app2)를 띄웁니다.
    - 앱팀 역할로 HTTPRoute를 Gateway `eg`에 붙입니다.
    - 문 하나로 들어온 요청이 `/app1`, `/app2` 경로에 따라 두 앱으로 갈라지는 것을 봅니다.

**소요 시간** 약 14분 (앞 6분 STEP 19~20, 뒤 8분 STEP 21~23), **교안** DAY 3 섹션 6

## 준비

- [실습 5](lab-5.md)의 Gateway `eg`가 `PROGRAMMED True`여야 합니다.

```bash
kubectl get gateway eg
```

- STEP 22에서 **터미널을 하나 더** 엽니다.

요청 하나가 지나가는 길은 아래와 같습니다. 문은 하나인데 길이 둘로 갈라집니다.

```text
curl /app1 ─▶ Gateway eg (Envoy 프록시) ─▶ HTTPRoute 규칙 비교 ─┬─▶ Service app1 ─▶ Pod app1
                                                                └─▶ Service app2 ─▶ Pod app2
```

## STEP 19. 앱 두 개 띄우기

`hashicorp/http-echo`는 어떤 경로로 요청해도 `-text`로 준 글자를 그대로 돌려주는 작은 웹 서버입니다. 어느 앱이 응답했는지 바로 보입니다. 기본으로 **5678번** 포트를 들으므로 Service는 `port: 80`, `targetPort: 5678`로 했습니다.

```yaml title="app1.yaml" hl_lines="14 15 25"
--8<-- "files/day3/app1.yaml"
```
[app1.yaml 내려받기](../files/day3/app1.yaml){ download }

`app2.yaml`은 `app1`을 모두 `app2`로 바꾼 사본입니다.

??? example "app2.yaml 전체 보기"
    ```yaml title="app2.yaml"
    --8<-- "files/day3/app2.yaml"
    ```

[app2.yaml 내려받기](../files/day3/app2.yaml){ download }

| 줄 | 뜻 |
|---|---|
| `image: hashicorp/http-echo:1.0` | 받은 요청에 정해 둔 글자를 돌려주는 서버. amd64, arm64 모두 있어 Apple Silicon Mac에서도 됩니다 |
| `args: ["-text=app1"]` | 돌려줄 글자 |
| `targetPort: 5678` | http-echo가 듣는 포트 |
| `---` | 한 파일에 Deployment와 Service를 이어 붙임 |

`-f`를 두 번 써서 한 번에 적용합니다.

```bash
kubectl apply -f app1.yaml -f app2.yaml
```

```text
deployment.apps/app1 created
service/app1 created
deployment.apps/app2 created
service/app2 created
```

이미지를 받는 동안 기다립니다. 두 줄 모두 `successfully rolled out`이 나오면 다음으로 갑니다.

```bash
kubectl rollout status deployment/app1 --timeout=120s
```

```bash
kubectl rollout status deployment/app2 --timeout=120s
```

라벨 셀렉터로 두 앱의 Pod만 골라 봅니다.

```bash
kubectl get pods -l 'app in (app1,app2)'
```

```text
NAME                    READY   STATUS    RESTARTS   AGE
app1-6d8f7c9b4-2kx8p    1/1     Running   0          15s
app2-5f9c8d7b6-q7w3z    1/1     Running   0          15s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

## STEP 20. 경로 붙이기 (앱팀 역할)

HTTPRoute를 `route.yaml`로 저장합니다. 교안 화면에는 `/app1` 규칙만 있지만, 실제 파일에는 `/app2` 규칙이 같은 모양으로 한 번 더 들어갑니다.

```yaml title="route.yaml" hl_lines="6 8 13"
--8<-- "files/day3/route.yaml"
```
[route.yaml 내려받기](../files/day3/route.yaml){ download }

| 줄 | 뜻 |
|---|---|
| `parentRefs: - name: eg` | 어느 문(Gateway)에 붙을지. 실습 5의 `eg` |
| `rules` | 규칙 목록. 규칙마다 `matches`와 `backendRefs`가 짝 |
| `matches.path` | 어떤 요청인지. `PathPrefix /app1`은 "/app1로 시작하면" |
| `backendRefs` | 보낼 Service 이름과 **Service의 port** (컨테이너 포트 5678이 아님) |

```bash
kubectl apply -f route.yaml
```

```text
httproute.gateway.networking.k8s.io/app-route created
```

```bash
kubectl get httproute
```

```text
NAME        HOSTNAMES   AGE
app-route               10s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`HOSTNAMES`가 비어 있는 것은 **정상**입니다. 호스트 조건을 주지 않았으니 어떤 호스트 이름으로 와도 받습니다.

Gateway가 이 경로를 받아들였는지 봅니다.

```bash
kubectl describe httproute app-route
```

```text
...
Status:
  Parents:
    Conditions:
      Message:               Route is accepted
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
      Message:               Resolved all the Object references for the Route
      Reason:                ResolvedRefs
      Status:                True
      Type:                  ResolvedRefs
    Controller Name:         gateway.envoyproxy.io/gatewayclass-controller
    Parent Ref:
      Name:                  eg
...
```
<small>출력은 예시이며 일부를 줄였습니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

| 조건 | True면 |
|---|---|
| `Accepted` | Gateway `eg`가 이 경로를 받아들였습니다 |
| `ResolvedRefs` | `backendRefs`의 Service(app1, app2)를 찾았습니다 |

## STEP 21. Envoy 프록시 Service 이름 찾기

Envoy Gateway가 `eg` 문을 위해 만든 Service를 라벨로 찾습니다.

```bash
kubectl get svc -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-name=eg
```

```text
NAME                        TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
envoy-default-eg-e41e7b31   LoadBalancer   10.97.204.11   localhost     80:31480/TCP   10m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

이름이 `envoy-default-eg-`로 시작하고 뒤에 꼬리가 붙어 있습니다. 사람마다 꼬리가 다르니 **내 화면의 NAME**을 복사해 둡니다.

## STEP 22. 새 터미널에서 port-forward

**새 터미널**을 열고 작업 폴더로 이동한 뒤, 내 PC의 8888 포트를 Envoy 프록시 Service의 80번에 연결합니다. 방법은 두 가지 중 편한 것을 씁니다.

**방법 1. 이름을 직접 붙여 넣기** (두 OS 공통)

STEP 21에서 복사한 NAME을 `svc/` 뒤에 붙입니다. 아래 `envoy-default-eg-e41e7b31`은 예시이니 **내 이름으로 바꿉니다.**

```bash
kubectl -n envoy-gateway-system port-forward svc/envoy-default-eg-e41e7b31 8888:80
```

**방법 2. 변수에 담아 쓰기**

이름을 직접 옮겨 적지 않고 변수에 담습니다. `-o name`은 `service/envoy-default-eg-...` 모양으로 이름만 돌려줍니다. 변수는 **port-forward를 실행할 바로 그 새 터미널**에서 만들어야 합니다 (다른 터미널에서 만든 변수는 보이지 않습니다).

=== "Windows (PowerShell)"

    ```powershell
    $SVC = kubectl get svc -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-name=eg -o name
    ```

    ```powershell
    $SVC
    ```

    ```powershell
    kubectl -n envoy-gateway-system port-forward $SVC 8888:80
    ```

=== "Mac (터미널)"

    ```bash
    export ENVOY_SERVICE=$(kubectl get svc -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-name=eg -o name)
    ```

    ```bash
    echo $ENVOY_SERVICE
    ```

    ```bash
    kubectl -n envoy-gateway-system port-forward $ENVOY_SERVICE 8888:80
    ```

가운데 명령은 변수에 이름이 잘 들어갔는지 확인하는 것입니다. `service/envoy-default-eg-...`가 한 줄 나오면 됩니다.

```text
Forwarding from 127.0.0.1:8888 -> 10080
Forwarding from [::1]:8888 -> 10080
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

오른쪽 번호는 Service의 80번이 가리키는 Envoy 컨테이너의 실제 포트라 80이 아닐 수 있습니다. 이 터미널은 그대로 둡니다.

## STEP 23. 경로별로 요청

**원래 터미널**에서 경로를 바꿔 가며 요청합니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe localhost:8888/app1
    ```

    ```powershell
    curl.exe localhost:8888/app2
    ```

    ```powershell
    curl.exe -i localhost:8888/
    ```

=== "Mac (터미널)"

    ```bash
    curl localhost:8888/app1
    ```

    ```bash
    curl localhost:8888/app2
    ```

    ```bash
    curl -i localhost:8888/
    ```

```text
app1
```

```text
app2
```

```text
HTTP/1.1 404 Not Found
date: Thu, 08 Oct 2026 05:12:34 GMT
content-length: 0
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`/app1`은 app1이, `/app2`는 app2가 답합니다. `/`는 어느 규칙에도 맞지 않아 Envoy가 **404**를 돌려줍니다. `-i`는 응답 헤더까지 보여 주는 옵션이라 첫 줄의 `404`를 볼 수 있습니다. **문은 하나인데 경로로 두 앱에 나뉘었습니다.**

!!! tip "브라우저로 확인해도 됩니다"
    [http://localhost:8888/app1](http://localhost:8888/app1), [http://localhost:8888/app2](http://localhost:8888/app2)를 엽니다. 화면에 `app1`, `app2` 글자만 보이면 됩니다. [http://localhost:8888/](http://localhost:8888/)은 빈 화면이나 "페이지를 찾을 수 없음"(404)이 나옵니다.

경로 매칭 규칙이 궁금하면 아래도 해 봅니다 (선택). Mac은 `curl.exe`를 `curl`로 바꿉니다.

| 요청 경로 | PathPrefix `/app1` | 이유 |
|---|---|---|
| `/app1/hello` | 맞음 → app1 | `/app1` 아래 경로 |
| `/app10` | **안 맞음 → 404** | 글자가 아니라 `/`로 나뉜 **조각 단위**로 비교 |
| `/` | 안 맞음 → 404 | 어느 규칙에도 없음 |

!!! info "경로로 나누기와 경로 바꾸기는 다릅니다"
    HTTPRoute는 **어느 Service로 보낼지**만 정하고 경로는 바꾸지 않습니다. `/app1/hello`로 요청하면 app1 Pod도 `/app1/hello`를 받습니다. http-echo는 경로를 보지 않고 같은 글자를 돌려줘서 오늘은 문제가 없었지만, 보통의 웹 앱은 `/app1`이라는 경로를 모르면 404를 냅니다. 이때는 `URLRewrite` 필터로 앞부분을 바꿔 보냅니다 (오늘 실습에서는 쓰지 않습니다).

    ```yaml
    rules:
    - matches:
      - path: { type: PathPrefix, value: /app1 }
      filters:
      - type: URLRewrite
        urlRewrite:
          path:
            type: ReplacePrefixMatch
            replacePrefixMatch: /
      backendRefs:
      - name: app1
        port: 80
    ```

    이렇게 하면 `/app1/hello`가 백엔드에는 `/hello`로 갑니다. URLRewrite는 Gateway API의 **Extended** 지원 기능이라 구현체가 지원하는지 확인하고 씁니다 (Envoy Gateway는 지원). [공식 가이드](https://gateway-api.sigs.k8s.io/guides/http-redirect-rewrite/)

!!! note "환경에 따라 다를 수 있음"
    STEP 21에서 Envoy Service의 `EXTERNAL-IP`가 `localhost`였다면 port-forward 없이 80번으로 바로 될 수도 있습니다. 브라우저에서 [http://localhost/app1](http://localhost/app1)을 열거나, `curl.exe localhost/app1`(Mac은 `curl localhost/app1`)을 시도해 봅니다. 안 되면(80번을 다른 프로그램이 쓰는 경우 포함) port-forward 방법을 씁니다.

## 성공하면

- STEP 19: app1, app2 Pod가 `Running`입니다.
- STEP 20: `HOSTNAMES`가 비어 있고(정상), describe에 `Accepted True`, `ResolvedRefs True`가 보입니다.
- STEP 23: `/app1` → `app1`, `/app2` → `app2`, `/` → `HTTP/1.1 404`.

## 막히면

??? failure "describe에 ResolvedRefs False (Reason: BackendNotFound 등)"
    `backendRefs`의 Service를 찾지 못했습니다.

    1. `kubectl get svc app1 app2`로 Service 이름이 정확한지 봅니다.
    2. `backendRefs`의 `port`가 **Service의 port 80**인지 봅니다. 컨테이너 포트 5678을 쓰면 연결되지 않습니다.

    고친 뒤 `kubectl apply -f route.yaml`을 다시 합니다.

??? failure "/app1, /app2 모두 404"
    HTTPRoute가 Gateway에 붙지 않았습니다.

    1. `route.yaml`의 `parentRefs` 이름이 `eg`인지 봅니다.
    2. `kubectl describe httproute app-route`의 `Accepted`가 True인지 봅니다.
    3. 방금 apply했다면 Envoy에 설정이 전달되기까지 몇 초 걸립니다. 5초 뒤 다시 요청합니다.

??? failure "PowerShell에서 kubectl get pods -l 'app in (app1,app2)'가 오류"
    셸에 따라 따옴표와 괄호 처리가 다를 수 있습니다. 큰따옴표로 바꿔 보거나, 그냥 전체를 봅니다.

    ```bash
    kubectl get pods
    ```

??? failure "error: arguments in resource/name form must have a single resource and name"
    방법 2에서 변수가 비었거나 이름이 두 개 이상 들어갔습니다. `$SVC`(Mac은 `echo $ENVOY_SERVICE`)를 실행해 `service/envoy-default-eg-...` 한 줄만 나오는지 봅니다. 아무것도 안 나오면 변수를 **port-forward를 실행하는 그 터미널**에서 다시 만들거나, 방법 1로 이름을 직접 붙여 넣습니다.

??? failure "Error from server (NotFound): services \"envoy-default-eg-e41e7b31\" not found"
    예시 이름을 그대로 붙여 넣었습니다. STEP 21에서 **내 화면의 NAME**을 복사해 바꿉니다.

??? failure "Unable to listen on port 8888 ... address already in use"
    내 PC에서 다른 프로그램이 8888을 씁니다. 앞 번호만 바꿉니다 (예: `18888:80`). 요청 주소도 `localhost:18888/app1`로 바꿉니다.

??? failure "curl: (7) Failed to connect to localhost port 8888"
    port-forward 터미널이 꺼졌거나 다른 터미널에서 요청하고 있지 않은지 봅니다. port-forward 터미널에 `Forwarding from 127.0.0.1:8888`이 떠 있어야 합니다.

??? failure "app1 Pod가 ImagePullBackOff 또는 exec format error"
    `kubectl describe pod <Pod 이름>`의 Events를 봅니다. 이미지 이름 `hashicorp/http-echo:1.0`의 오타를 먼저 확인합니다. 네트워크 문제면 잠시 뒤 다시 시도합니다.

## 정리

port-forward 터미널은 `Ctrl + C`로 끝냅니다. 오늘 실습 6에서 만든 리소스와 **Gateway `eg`, GatewayClass `eg`**를 지웁니다. 4일차에는 쓰지 않습니다.

!!! warning "Gateway eg는 꼭 지웁니다"
    `eg`는 HTTP **80번** 문을 쥐고 있습니다. 남겨 두면 5일차 미니 블로그의 Gateway `blog-gw`도 80번을 열려고 해서 서로 겹칩니다. 문(Gateway, GatewayClass)만 지우고, 설치(Envoy Gateway)는 남깁니다.

```bash
kubectl delete -f route.yaml -f app1.yaml -f app2.yaml -f gateway.yaml
```

```text
httproute.gateway.networking.k8s.io "app-route" deleted
deployment.apps "app1" deleted
service "app1" deleted
deployment.apps "app2" deleted
service "app2" deleted
gatewayclass.gateway.networking.k8s.io "eg" deleted
gateway.gateway.networking.k8s.io "eg" deleted
```

!!! warning "지우지 마세요"
    - **Envoy Gateway 설치**(`envoy-gateway-system`)는 5일차 미니 블로그 종합 배포에서 그대로 씁니다. install.yaml로 delete하지 않습니다.
    - **web Deployment와 web Service**도 남겨 둡니다.

나머지 하루 정리는 [오늘의 실습](index.md)의 "오늘 끝나고 정리"를 봅니다.
