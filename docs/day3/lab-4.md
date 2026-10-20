# 실습 4. EndpointSlice 읽고, 빈 목록으로 원인 찾기

!!! abstract "이번 실습에서 할 것"
    - Service 뒤의 실제 주소 목록 **EndpointSlice**를 읽습니다.
    - Pod를 늘리면 목록이 따라 바뀌는 것을 실시간으로 봅니다.
    - 라벨 오타로 일부러 고장 낸 Service를, 빈 EndpointSlice를 보고 원인을 찾아 고칩니다.

**소요 시간** 약 14분 (앞 6분 STEP 10~12, 뒤 8분 STEP 13~15), **교안** DAY 3 섹션 4

!!! tip "오늘 가장 중요한 실습입니다"
    "서비스에 접속이 안 돼요"라는 문의의 상당수가 이 실습의 순서로 풀립니다. 4일차 readinessProbe 실습과 5일차 고장 난 미니 블로그 고치기에서 같은 명령을 그대로 씁니다.

## 준비

- [실습 1](lab-1.md)의 `web` Deployment(Pod 3개)와 `web` Service가 있어야 합니다.

```bash
kubectl get deploy web
```

```bash
kubectl get svc web
```

- STEP 12에서 **터미널을 하나 더** 엽니다. 새 터미널도 작업 폴더로 이동합니다.

## 미리 알아 둘 것

Service는 Pod로 직접 보내지 않습니다. 컨트롤 플레인이 selector에 맞는 Pod들의 IP를 모아 **주소 목록(EndpointSlice)**을 만들어 두고, 각 노드의 kube-proxy가 이 목록을 읽어 트래픽 규칙을 만듭니다.

```text
Service web (selector app: web)
  └─ EndpointSlice web-xxxxx
        10.1.0.12:80  ready
        10.1.0.13:80  ready     ──▶  kube-proxy가 읽어 규칙을 만든다
        10.1.0.17:80  ready
```

| 알아 둘 것 | 내용 |
|---|---|
| 누가 만드나 | 사람이 아니라 controller-manager 안의 EndpointSlice 컨트롤러가 자동으로 만들고 고칩니다 |
| 어떻게 찾나 | 이름 뒤에 무작위 꼬리가 붙어서, 라벨 `kubernetes.io/service-name=web`으로 찾습니다 |
| 옛 방식 | `Endpoints`는 쿠버네티스 v1.33부터 deprecated입니다. 읽기만 해도 경고가 뜹니다 |
| 조각 크기 | 조각 하나에 기본 100개 주소. 넘으면 조각이 늘어납니다 |

## STEP 10. web의 조각 보기

`web` Service의 EndpointSlice만 라벨로 골라 봅니다. 명령이 길어 오타가 잦으니 복사해서 씁니다.

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web
```

```text
NAME        ADDRESSTYPE   PORTS   ENDPOINTS                       AGE
web-x7k2p   IPv4          80      10.1.0.12,10.1.0.13,10.1.0.17   20m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

| 칸 | 뜻 |
|---|---|
| `NAME` | Service 이름 + 무작위 꼬리 |
| `ADDRESSTYPE` | IPv4 또는 IPv6 |
| `PORTS` | 트래픽을 보낼 포트 (targetPort) |
| `ENDPOINTS` | 목록에 든 Pod IP. 비었는지 아닌지가 **핵심**입니다. 단, ready 여부는 표시하지 않습니다 |

!!! warning "주소가 보인다고 모두 트래픽을 받는 것은 아닙니다"
    `ENDPOINTS` 칸은 목록에 든 주소를 보여 줄 뿐, 그 주소가 **ready인지는 표시하지 않습니다.** 준비가 안 된 Pod의 주소도 이 칸에 나옵니다. 트래픽을 받는지는 아래 `-o yaml`의 `conditions.ready`로 확인합니다. 4일차 readinessProbe 실습에서 주소는 남아 있는데 `ready: false`인 경우를 직접 봅니다.

`ENDPOINTS`의 IP가 실제 Pod IP와 같은지 비교해 봅니다.

```bash
kubectl get pods -l app=web -o wide
```

```text
NAME                  READY   STATUS    RESTARTS   AGE   IP          NODE             ...
web-7d9f8c6b5-abcde   1/1     Running   0          1d    10.1.0.12   docker-desktop   ...
web-7d9f8c6b5-fghij   1/1     Running   0          1d    10.1.0.13   docker-desktop   ...
web-7d9f8c6b5-klmno   1/1     Running   0          1d    10.1.0.17   docker-desktop   ...
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

이번에는 `-o yaml`로 조각 안을 열어 봅니다.

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web -o yaml
```

```yaml
apiVersion: v1
items:
- addressType: IPv4
  apiVersion: discovery.k8s.io/v1
  endpoints:
  - addresses:
    - 10.1.0.12
    conditions:
      ready: true
      serving: true
      terminating: false
    nodeName: docker-desktop
    targetRef:
      kind: Pod
      name: web-7d9f8c6b5-abcde
      namespace: default
  - addresses:
    - 10.1.0.13
    conditions:
      ready: true
    # ... (Pod마다 한 덩어리씩 3개)
  kind: EndpointSlice
  metadata:
    labels:
      endpointslice.kubernetes.io/managed-by: endpointslice-controller.k8s.io
      kubernetes.io/service-name: web
    name: web-x7k2p
  ports:
  - name: ""
    port: 80
    protocol: TCP
kind: List
```
<small>출력은 예시이며 일부를 줄였습니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

아래 다섯 군데만 찾아 읽으면 됩니다.

| 필드 | 뜻 |
|---|---|
| `labels.kubernetes.io/service-name` | 어느 Service의 조각인지 |
| `addressType` | 주소 종류 (IPv4) |
| `endpoints[].addresses` | Pod의 IP. Pod마다 한 덩어리 |
| `endpoints[].conditions.ready` | **true여야 트래픽을 받습니다** |
| `ports` | 조각 안의 모든 주소에 공통인 포트 |

`targetRef`를 보면 이 주소가 어느 Pod인지 이름까지 나옵니다. `managed-by` 라벨은 이 조각을 컨트롤러가 만들었다는 표시입니다.

!!! info "conditions 세 값"
    - **serving**: Pod가 응답할 수 있는 상태인지. Pod의 Ready와 같습니다 (4일차 readinessProbe가 정합니다).
    - **terminating**: Pod 삭제가 시작되어 종료 중인지.
    - **ready**: serving이면서 terminating이 아닐 때 true. kube-proxy는 보통 ready가 true인 주소로만 보냅니다.

## STEP 11. 옛 API와 비교 (경고 확인)

옛 주소 목록 오브젝트인 `Endpoints`로도 봅니다.

```bash
kubectl get endpoints web
```

```text
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME   ENDPOINTS                                AGE
web    10.1.0.12:80,10.1.0.13:80,10.1.0.17:80   20m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

결과는 나오지만 맨 위에 **Warning 한 줄**이 붙습니다. 쿠버네티스 v1.33부터 `Endpoints` API가 deprecated(쓰지 말라고 권고) 되었기 때문입니다. 바로 없어지지는 않지만 앞으로는 `kubectl get endpointslices`로 봅니다.

| | Endpoints (옛 방식) | EndpointSlice |
|---|---|---|
| 구조 | Service 하나에 목록 하나, 통째로 | 기본 100개씩 조각으로 나눔 |
| Pod 1개가 바뀌면 | 전체 목록을 모든 노드로 다시 전송 | 바뀐 조각만 전송 |
| 1000개 넘으면 | 잘리고 `over-capacity: truncated` 주석 | 조각을 늘리면 됨 |

!!! note "Warning이 안 보이면"
    Docker Desktop의 쿠버네티스 버전이 1.33보다 낮으면 경고 없이 결과만 나옵니다. `kubectl version`의 `Server Version`을 확인합니다. 결과는 같으니 그대로 진행합니다.

## STEP 12. 새 터미널에서 지켜보며 5개로

**터미널 두 개**를 씁니다. 먼저 **새 터미널(터미널 2)**에서 `-w`(watch)로 목록을 지켜봅니다. 이 명령은 끝나지 않고 계속 떠 있습니다.

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web -w
```

```text
NAME        ADDRESSTYPE   PORTS   ENDPOINTS                       AGE
web-x7k2p   IPv4          80      10.1.0.12,10.1.0.13,10.1.0.17   25m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

**원래 터미널(터미널 1)**에서 Pod를 5개로 늘립니다.

```bash
kubectl scale deployment web --replicas=5
```

```text
deployment.apps/web scaled
```

터미널 2를 봅니다. 새 Pod가 IP를 받는 대로 줄이 새로 찍히며 주소가 늘어납니다.

```text
NAME        ADDRESSTYPE   PORTS   ENDPOINTS                       AGE
web-x7k2p   IPv4          80      10.1.0.12,10.1.0.13,10.1.0.17   25m
web-x7k2p   IPv4          80      10.1.0.12,10.1.0.13,10.1.0.17 + 1 more...   25m
web-x7k2p   IPv4          80      10.1.0.12,10.1.0.13,10.1.0.17 + 2 more...   25m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

주소가 3개를 넘으면 `+ 2 more...`처럼 줄여서 보여 줍니다. 막 생긴 Pod의 주소는 잠깐 `ready: false`였다가 true로 바뀔 수 있는데, 이 칸에서는 구분되지 않습니다. 전체와 ready 여부는 `-o yaml`로 봅니다.

!!! warning "주소 5개, READY 확인 후 Ctrl+C"
    `-w`는 스스로 끝나지 않습니다. 주소가 5개로 늘고 `kubectl get pods -l app=web`에서 5개 모두 READY `1/1`인 것을 확인한 뒤 터미널 2에서 `Ctrl + C`로 끝내고 다음 STEP으로 갑니다.

!!! info "사람이 목록을 고치지 않았습니다"
    replicas만 바꿨는데 목록이 저절로 바뀌었습니다. EndpointSlice 컨트롤러가 Pod 변화를 감시하다 조각을 고치고, kube-proxy가 바뀐 조각을 읽어 규칙을 갱신합니다. 어제 본 **조정 루프**가 여기서도 돌고 있습니다. 컨트롤러의 기준은 **Service의 selector** 하나뿐입니다. 다음 STEP에서 이 기준을 일부러 틀리게 만듭니다.

## STEP 13. 고장 내고 접속

이제 일부러 고장을 냅니다. `web-svc.yaml`을 복사해 `web-svc-broken.yaml`로 저장하고, selector의 `web`을 **`wbe`**로 한 글자 바꿉니다. (아래 파일을 그대로 내려받아도 됩니다.)

```yaml title="web-svc-broken.yaml" hl_lines="7"
--8<-- "files/day3/web-svc-broken.yaml"
```
[web-svc-broken.yaml 내려받기](../files/day3/web-svc-broken.yaml){ download }

| 줄 | 뜻 |
|---|---|
| `name: web` | 이름이 같아서 **기존 web Service를 덮어씁니다** (새 Service가 생기지 않음) |
| `app: wbe` | `web`을 `wbe`로 한 글자 바꿈 |
| 예상 | apply는 성공, 접속은 실패 |

```bash
kubectl apply -f web-svc-broken.yaml
```

```text
service/web configured
```

오류 없이 성공합니다. YAML 문법은 맞기 때문입니다. 이제 실습 1처럼 임시 Pod에서 접속합니다. `-T 3`이라 3초 뒤 포기합니다.

```bash
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -- wget -qO- -T 3 web
```

```text
wget: can't connect to remote host (10.96.120.5): Connection refused
pod "tmp" deleted
pod default/tmp terminated (Error)
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

!!! note "환경에 따라 다를 수 있음"
    주소 목록이 빈 Service로 접속하면 `Connection refused`가 나오는 환경도 있고, 3초 동안 기다리다 `wget: download timed out`이 나오는 환경도 있습니다. 둘 다 **접속 실패**이고, 이 실습에서는 같은 증상으로 봅니다.

!!! warning "여기가 이 장애가 어려운 이유입니다"
    - `apply`는 성공했습니다.
    - `kubectl get svc`를 보면 Service가 있고 ClusterIP도 있습니다.
    - `kubectl get pods`를 보면 Pod 5개가 모두 Running입니다.
    - Pod 로그에도 아무것도 없습니다. 요청이 Pod까지 오지 않았으니까요.

    **다 정상인데 접속만 안 됩니다.** Service와 Pod만 봐서는 원인을 못 찾고, 그 **사이**의 EndpointSlice를 봐야 합니다.

원인을 미리 알고 있지만, 모르는 척하고 아래 디버깅 순서대로 찾아봅니다.

## STEP 14. 순서대로 찾기

연결이 안 될 때 보는 순서입니다.

```text
① Service  →  ② EndpointSlice  →  ③ Pod  →  ④ 라벨
   있나?         비었나?             Running?     selector와 같나?
```

**① Service가 있나, 포트는 맞나**

```bash
kubectl get svc web
```

```text
NAME   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
web    ClusterIP   10.96.120.5    <none>        80/TCP    40m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

정상입니다. Service도 있고 포트도 80입니다.

**② EndpointSlice가 비었나** (가장 중요)

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web
```

```text
NAME        ADDRESSTYPE   PORTS     ENDPOINTS   AGE
web-x7k2p   IPv4          <unset>   <unset>     40m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

**`ENDPOINTS`가 비어 있습니다** (`<unset>`). 조각은 남아 있지만 안에 주소가 하나도 없습니다. Service는 있는데 요청을 보낼 Pod가 없다는 뜻입니다. 여기서 원인이 "Service와 Pod의 **연결**"이라는 것을 압니다.

!!! note "환경에 따라 다를 수 있음"
    빈 칸이 `<unset>` 대신 `<none>`이나 빈칸으로 보일 수 있습니다. 어느 쪽이든 **IP가 하나도 없으면 빈 목록**입니다.

**③ Pod는 Running이고 Ready인가**

```bash
kubectl get pods --show-labels
```

```text
NAME                  READY   STATUS    RESTARTS   AGE   LABELS
web-7d9f8c6b5-abcde   1/1     Running   0          1d    app=web,pod-template-hash=7d9f8c6b5
web-7d9f8c6b5-fghij   1/1     Running   0          1d    app=web,pod-template-hash=7d9f8c6b5
web-7d9f8c6b5-klmno   1/1     Running   0          1d    app=web,pod-template-hash=7d9f8c6b5
web-7d9f8c6b5-pqrst   1/1     Running   0          10m   app=web,pod-template-hash=7d9f8c6b5
web-7d9f8c6b5-uvwxy   1/1     Running   0          10m   app=web,pod-template-hash=7d9f8c6b5
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

Pod 5개 모두 `1/1 Running`입니다. Pod 쪽은 정상입니다. **`LABELS` 칸의 `app=web`**을 기억합니다.

**④ 라벨과 selector가 같은가**

```bash
kubectl describe svc web
```

```text
Name:              web
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=wbe
Type:              ClusterIP
IP:                10.96.120.5
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:
...
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

**`Selector: app=wbe`** 인데 Pod 라벨은 **`app=web`** 입니다. 글자 하나가 다릅니다. 이것이 원인입니다. `Endpoints:` 줄이 비어 있는 것도 ②와 같은 정보입니다.

| 본 것 | 결과 | 판단 |
|---|---|---|
| ① `get svc web` | 있음, 80/TCP | 정상 |
| ② `get endpointslices` | ENDPOINTS 비어 있음 | **연결이 끊김** |
| ③ `get pods --show-labels` | 5개 Running, 라벨 `app=web` | 정상 |
| ④ `describe svc web` | Selector `app=wbe` | **원인: selector 오타** |

!!! tip "목록이 비었나, 찼나로 갈린다"
    - **EndpointSlice가 비었으면**: selector와 Pod 라벨을 비교합니다.
    - **주소는 있는데 `ready: false`면**: Pod가 준비되지 않았습니다. `-o yaml`로 확인하고 Pod 상태를 봅니다 (4일차 readinessProbe).
    - **주소도 있고 ready인데 접속이 안 되면**: `targetPort`가 앱이 실제로 듣는 포트와 같은지 봅니다.

## STEP 15. 고치고 다시 확인

원래 YAML(`selector: app: web`)을 다시 apply합니다. 이름이 같으니 selector만 다시 바뀝니다.

```bash
kubectl apply -f web-svc.yaml
```

```text
service/web configured
```

곧바로 목록이 다시 채워집니다. STEP 12에서 5개로 늘렸으니 주소가 5개입니다.

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web
```

```text
NAME        ADDRESSTYPE   PORTS   ENDPOINTS                                   AGE
web-x7k2p   IPv4          80      10.1.0.12,10.1.0.13,10.1.0.17 + 2 more...   45m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

다시 접속합니다.

```bash
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -- wget -qO- -T 3 web
```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
pod "tmp" deleted
```

!!! info "짝 실습으로 해 보기 (선택)"
    시간이 남으면 짝과 함께 해 봅니다. 한 사람이 몰래 selector나 targetPort를 바꿔 apply하고, 다른 사람이 STEP 14 순서로 원인을 찾습니다.

## 성공하면

- STEP 10: 주소 3개, `-o yaml`에 `ready: true`가 보입니다.
- STEP 11: 결과 위에 `Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice` 한 줄이 보입니다.
- STEP 12: `-w` 화면에서 주소가 3개 → 5개(`+ 2 more...`)로 늘어납니다.
- STEP 13: apply는 성공, 접속은 실패합니다.
- STEP 14: `ENDPOINTS`가 비어 있고, `describe svc`의 `Selector`가 `app=wbe`입니다.
- STEP 15: 주소 5개가 돌아오고 접속에 성공합니다.

## 막히면

??? failure "No resources found in default namespace."
    라벨 이름에 오타가 있습니다. `kubernetes.io/service-name=web`을 그대로 복사해 다시 실행합니다. 단수 `endpointslice`로 써도 됩니다.

??? failure "error: the server doesn't have a resource type \"endpointslice\""
    리소스 이름 철자를 확인합니다. `endpointslices` (또는 `endpointslice`)입니다. `endpointsslice`처럼 s가 하나 더 들어가기 쉽습니다.

??? failure "-w 명령이 끝나지 않음"
    정상입니다. `-w`는 바뀔 때마다 줄을 새로 찍으며 계속 지켜봅니다. 다 봤으면 `Ctrl + C`로 끝냅니다. Windows에는 `watch` 명령이 없으니 항상 `-w`를 씁니다.

??? failure "STEP 12에서 주소가 늘지 않음"
    `kubectl get pods -l app=web`으로 새 Pod가 `Running`인지 봅니다. 이미지를 받는 중이면 `ContainerCreating`으로 잠시 머뭅니다. IP를 받은 Pod부터 목록에 들어가고, 트래픽은 `ready: true`가 된 뒤에 받습니다.

??? failure "STEP 13에서 접속이 성공해 버림"
    고장 난 파일이 적용되지 않았습니다.

    1. `kubectl describe svc web`의 `Selector`가 `app=wbe`인지 봅니다.
    2. `app=web` 그대로면 `web-svc-broken.yaml`을 저장했는지(VS Code 탭의 동그라미 표시), 정말 `wbe`로 바꿨는지 확인하고 다시 apply합니다.

??? failure "STEP 15 뒤에도 ENDPOINTS가 비어 있음"
    `web-svc.yaml`의 selector가 `app: web`인지 다시 봅니다. 실수로 원본 파일을 고쳤다면 [web-svc.yaml 내려받기](../files/day3/web-svc.yaml){ download }로 원본을 다시 받아 apply합니다.

??? failure "Error from server (AlreadyExists): pods \"tmp\" already exists"
    남아 있는 임시 Pod를 지우고 다시 실행합니다.

    ```bash
    kubectl delete pod tmp
    ```

## 정리

replicas를 원래대로 3으로 돌립니다.

```bash
kubectl scale deployment web --replicas=3
```

!!! warning "지우지 마세요"
    `web` Deployment와 `web` Service는 오늘 끝까지 남겨 둡니다. `web-svc-broken.yaml` 파일은 지워도 됩니다.
