# 실습 2. 이름으로 찾기, 다른 네임스페이스에서 찾기

!!! abstract "이번 실습에서 할 것"
    - CoreDNS가 Service 이름을 ClusterIP로 바꿔 주는 것을 확인합니다.
    - 네임스페이스가 다르면 이름을 어떻게 써야 하는지 확인합니다.

**소요 시간** 약 12분, **교안** DAY 3 섹션 2

## 준비

- [실습 1](lab-1.md)의 `web` Deployment와 `web` Service가 있어야 합니다.

```bash
kubectl get svc web
```

`CLUSTER-IP` 값을 기억해 둡니다. STEP 4에서 같은 값이 나오는지 비교합니다.

## 이름 규칙 미리 보기

Service를 만들면 CoreDNS에 아래 형식의 이름이 등록됩니다.

```text
<서비스>.<네임스페이스>.svc.cluster.local
web     .default        .svc.cluster.local
```

같은 네임스페이스 안에서는 `web`만 써도 됩니다. Pod 안의 `/etc/resolv.conf`에 있는 **search 목록**이 뒷부분을 차례로 붙여 주기 때문입니다.

```text
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

궁금하면 아래 명령으로 직접 볼 수 있습니다 (선택).

```bash
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -- cat /etc/resolv.conf
```

## STEP 4. 이름을 IP로 바꿔 보기

임시 Pod에서 `nslookup`으로 `web`이라는 이름을 물어봅니다.

```bash
kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -- nslookup web
```

```text
Server:		10.96.0.10
Address:	10.96.0.10:53

Name:	web.default.svc.cluster.local
Address: 10.96.120.5

** server can't find web.svc.cluster.local: NXDOMAIN
** server can't find web.cluster.local: NXDOMAIN
...
pod "tmp" deleted
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

| 줄 | 뜻 |
|---|---|
| `Server` | 대답한 DNS 서버. CoreDNS 앞의 `kube-dns` Service 주소입니다 |
| `Name` | 긴 이름 `web.default.svc.cluster.local` |
| `Address` | 실습 1의 `kubectl get svc web`에서 본 **CLUSTER-IP**와 같아야 합니다 |

!!! note "NXDOMAIN 줄이 같이 나와도 성공입니다"
    busybox의 nslookup은 search 목록을 하나씩 돌면서 없는 이름에 대해 `NXDOMAIN` 줄을 함께 찍는 경우가 있어 실패처럼 보입니다. **`Name`과 `Address` 줄이 있으면 성공**입니다. 출력이 헷갈리면 `Address` 값을 `kubectl get svc web`의 CLUSTER-IP와 비교만 합니다.

CoreDNS가 어디서 도는지도 봅니다 (선택).

```bash
kubectl get svc -n kube-system kube-dns
```

```text
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   2d
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

여기 CLUSTER-IP가 STEP 4의 `Server`와 같습니다.

## STEP 5. 다른 네임스페이스 만들기

`other`라는 네임스페이스를 만듭니다. 네임스페이스는 클러스터 안을 나누는 칸막이입니다.

```bash
kubectl create namespace other
```

```text
namespace/other created
```

## STEP 6. other에서 짧은 이름 → 긴 이름

`other` 네임스페이스에 임시 Pod를 띄우고 **짧은 이름** `web`으로 접속합니다. `-T 3`은 3초만 기다리라는 옵션입니다.

```bash
kubectl run tmp -n other --rm -it --restart=Never --image=busybox:1.36 -- wget -qO- -T 3 web
```

```text
wget: bad address 'web'
pod "tmp" deleted
pod other/tmp terminated (Error)
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

**실패하는 것이 정상입니다.** `other`의 Pod는 search 목록의 첫 꼬리가 `other.svc.cluster.local`이라 `web.other.svc.cluster.local`을 찾는데, 그런 Service는 없습니다.

이번에는 네임스페이스까지 붙여 `web.default`로 접속합니다.

```bash
kubectl run tmp -n other --rm -it --restart=Never --image=busybox:1.36 -- wget -qO- web.default
```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
pod "tmp" deleted
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

같은 동네(네임스페이스)면 이름만, 다른 동네면 동네 이름까지 씁니다. 어디서 실행해도 같은 뜻이 되게 하려면 전체 이름 `web.default.svc.cluster.local`을 씁니다.

!!! warning "네임스페이스는 통신을 막지 않습니다"
    첫째 명령이 실패한 이유는 **이름을 못 찾아서**이지, 막혀서가 아닙니다. 둘째 명령처럼 이름만 맞게 쓰면 `other`에서 `default`의 Pod까지 그대로 통합니다. 네임스페이스는 리소스와 이름을 나누는 공간이고 네트워크를 격리하지 않습니다. 통신을 제한하려면 5일차에 배우는 **NetworkPolicy**를 따로 씁니다.

## 성공하면

- STEP 4: `Name`에 `web.default.svc.cluster.local`, `Address`에 web Service의 ClusterIP가 나옵니다.
- STEP 6: 첫째 명령(`web`)은 실패, 둘째 명령(`web.default`)은 nginx HTML이 나옵니다. 네임스페이스가 달라도 통신은 됩니다.

## 막히면

??? failure "nslookup 출력에 NXDOMAIN이 잔뜩 나와 실패처럼 보임"
    busybox nslookup의 특징입니다. `Name`, `Address` 줄이 있는지만 봅니다. 그래도 헷갈리면 `kubectl get svc web`의 CLUSTER-IP와 `Address`를 비교합니다.

??? failure "Error from server (AlreadyExists): namespaces \"other\" already exists"
    이미 만들어져 있습니다. 그대로 STEP 6으로 넘어가면 됩니다.

??? failure "Error from server (AlreadyExists): pods \"tmp\" already exists"
    남아 있는 임시 Pod를 지웁니다. `other`에 남았으면 `-n other`를 붙입니다.

    ```bash
    kubectl delete pod tmp -n other
    ```

??? failure "web.default로 해도 bad address"
    `web` Service가 `default` 네임스페이스에 있는지 확인합니다.

    ```bash
    kubectl get svc -A
    ```

    `NAMESPACE` 칸이 `default`가 아니면 그 이름을 붙입니다. 예: `web.<네임스페이스>`.

## 정리

`other` 네임스페이스는 다음 실습에 필요 없습니다. 네임스페이스를 지우면 안의 리소스도 함께 지워집니다.

```bash
kubectl delete namespace other
```

!!! warning "지우지 마세요"
    `web` Deployment와 `web` Service는 실습 3, 4에서 계속 씁니다.
