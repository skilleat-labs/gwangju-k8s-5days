# 실습 3. NetworkPolicy

!!! abstract "이번 실습에서 할 것"
    - 정책이 없으면 모든 Pod가 서로 통하는 것(평면 네트워크)을 눈으로 봅니다.
    - deny-all로 전부 막은 뒤, 이름 접속과 IP 접속을 따로 확인합니다.
    - backend → db와 DNS만 엽니다.
    - backend만 db에 닿고 hacker는 막히는 것을 확인합니다.

**소요 시간** 약 12분, **교안** DAY 5 섹션 3

!!! note "환경에 따라 다를 수 있음: 진행 경로는 강사가 미리 알려 줍니다"
    NetworkPolicy는 API 서버가 **저장만** 하고, 실제로 패킷을 막는 것은 클러스터의 네트워크 플러그인(CNI)입니다. CNI가 NetworkPolicy를 지원하지 않으면 `kubectl apply`는 `created`로 성공하고 `kubectl get networkpolicy`에도 보이는데, **트래픽은 그대로 통과합니다.**

    Docker Desktop이 NetworkPolicy를 강제하는지는 강사가 수업 전에 확인해서 섹션 3 시작 때 알려 줍니다.

    | 경로 | 강사 사전 점검 결과 | 여러분이 할 일 |
    |---|---|---|
    | A | 막혔다 | STEP 11~16을 모두 직접 하고, 차단까지 각자 화면에서 확인합니다 |
    | B | 막히지 않았다 | YAML 작성, `apply`, `describe`까지 각자 합니다. 각자 화면에서 모두 통과해도 **정상**입니다. 차단 결과는 Calico나 Cilium이 설치된 **강사 시연 화면**으로 봅니다 |

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

이번 실습에 쓰는 정책 파일 세 개를 작업 폴더에 만듭니다. 세 파일 모두 `namespace: np-lab`이 들어 있어서 `-n` 없이 적용합니다.

```yaml title="deny-all.yaml"
--8<-- "files/day5/deny-all.yaml"
```
[deny-all.yaml 내려받기](../files/day5/deny-all.yaml){ download }

`podSelector: {}`는 네임스페이스의 모든 Pod, `policyTypes`에 두 방향을 넣고 `ingress`, `egress` 목록을 안 쓰면 **두 방향 모두 전부 막힙니다.** 이름 찾기(DNS)도 밖으로 나가는 통신이라 같이 막힙니다.

```yaml title="allow-backend-db.yaml"
--8<-- "files/day5/allow-backend-db.yaml"
```
[allow-backend-db.yaml 내려받기](../files/day5/allow-backend-db.yaml){ download }

통신은 양쪽 끝이 있어서 정책이 두 개입니다. 위는 db 입장에서 `app=backend`가 80 포트로 **들어오는 것**만 받고, 아래는 backend 입장에서 `app=db`의 80 포트로 **나가는 것**만 허락합니다. deny-all이 두 방향을 다 막았으니 한쪽만 열면 여전히 막힙니다.

!!! info "허용은 합쳐지고, 양쪽 끝이 모두 열어야 통한다"
    - 연결 하나가 되려면 **출발 Pod의 Egress**와 **도착 Pod의 Ingress**가 둘 다 허용해야 합니다. 어느 한쪽이라도 허용하지 않으면 연결되지 않습니다.
    - 정책끼리는 충돌하지 않고 **더해집니다**. 한 Pod에 정책이 여러 개 걸리면 허용 목록의 합집합이 그 Pod의 허용 범위입니다.
    - 그래서 deny-all이 다른 정책을 "이기는" 것이 아닙니다. deny-all만 있을 때는 허용이 하나도 없어서 막혀 있던 것이고, 허용 정책을 더하면 그만큼 열립니다.

```yaml title="allow-dns.yaml"
--8<-- "files/day5/allow-dns.yaml"
```
[allow-dns.yaml 내려받기](../files/day5/allow-dns.yaml){ download }

np-lab의 모든 Pod가 kube-system의 CoreDNS(`k8s-app: kube-dns`)에게 53번 포트로 묻는 것만 엽니다. `namespaceSelector`와 `podSelector`가 한 덩어리(같은 `-` 아래)라서 "그 네임스페이스의 그 Pod"라는 AND 조건입니다.

CoreDNS Pod에 `k8s-app=kube-dns` 라벨이 있는지 미리 봅니다.

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

```text
NAME                       READY   STATUS    RESTARTS   AGE
coredns-7db6d8ff4d-abcde   1/1     Running   0          4d
coredns-7db6d8ff4d-fghij   1/1     Running   0          4d
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`coredns` Pod가 보이면 allow-dns.yaml을 그대로 씁니다.

## STEP 11. db, backend, hacker

`np-lab` 네임스페이스에 Pod 세 개를 띄웁니다. 라벨이 중요합니다. db는 `app=db`, backend는 `app=backend`, 공격자 역할의 hacker는 `app=web`입니다. db에는 Service를 붙여 `db`라는 이름으로 찾게 합니다.

```bash
kubectl create ns np-lab
```

```bash
kubectl run db --image=nginx:1.27 -l app=db -n np-lab
```

```bash
kubectl expose pod db --port=80 -n np-lab
```

```bash
kubectl run backend --image=busybox:1.36 -l app=backend -n np-lab -- sleep 3600
```

```bash
kubectl run hacker --image=busybox:1.36 -l app=web -n np-lab -- sleep 3600
```

세 Pod가 모두 Ready가 될 때까지 기다립니다. 이미지를 받는 중에 exec하면 실패합니다.

```bash
kubectl wait --for=condition=Ready pod --all -n np-lab --timeout=120s
```

라벨과 상태를 봅니다.

```bash
kubectl get pods -n np-lab --show-labels
```

```text
NAME      READY   STATUS    RESTARTS   AGE   LABELS
backend   1/1     Running   0          20s   app=backend
db        1/1     Running   0          30s   app=db
hacker    1/1     Running   0          15s   app=web
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

## STEP 12. 정책 없이: hacker도 db에 닿는다

```bash
kubectl exec -n np-lab hacker -- wget -qO- -T 3 db
```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

`-T 3`은 3초 안에 답이 없으면 포기하라는 뜻입니다. 정책이 없으니 공격자 역할의 hacker도 db에 바로 닿습니다.

db Pod의 IP를 적어 둡니다. STEP 14에서 DNS를 거치지 않고 IP로 접속해 볼 때 씁니다.

```bash
kubectl get pod db -n np-lab -o wide
```

```text
NAME   READY   STATUS    RESTARTS   AGE   IP          NODE             ...
db     1/1     Running   0          2m    10.1.0.57   docker-desktop   ...
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

## STEP 13. 전부 막기

```bash
kubectl apply -f deny-all.yaml
```

```text
networkpolicy.networking.k8s.io/deny-all created
```

## STEP 14. 이름으로 한 번, IP로 한 번

deny-all은 DNS 질문(밖으로 나가는 통신)까지 막습니다. 그래서 이름으로 접속할 때와 IP로 접속할 때를 나눠 봐야 무엇이 막혔는지 구분됩니다.

이름으로 접속합니다.

```bash
kubectl exec -n np-lab hacker -- wget -qO- -T 3 db
```

```text
wget: bad address 'db'
command terminated with exit code 1
```
<small>출력은 예시입니다. DNS 응답을 기다리느라 3초보다 오래 걸릴 수 있습니다.</small>

이름을 IP로 바꾸는 단계에서 실패했습니다. 이번에는 STEP 12에서 적어 둔 IP로 접속합니다. `<db IP>` 자리에 내 IP를 넣습니다.

```bash
kubectl exec -n np-lab hacker -- wget -qO- -T 3 <db IP>
```

```text
wget: download timed out
command terminated with exit code 1
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

DNS를 거치지 않았는데도 실패합니다. db로 가는 연결 자체가 막혔다는 뜻입니다.

!!! note "경로 B라면"
    두 명령 모두 `Welcome to nginx!`가 나옵니다. 여러분 YAML이 틀린 것이 아니라 이 클러스터의 CNI가 막지 않는 것입니다. 남은 STEP도 그대로 따라 하고, 차단 결과는 강사 시연 화면으로 봅니다.

## STEP 15. backend → db와 DNS만 열기

```bash
kubectl apply -f allow-backend-db.yaml
```

```bash
kubectl apply -f allow-dns.yaml
```

```text
networkpolicy.networking.k8s.io/db-from-backend created
networkpolicy.networking.k8s.io/backend-to-db created
networkpolicy.networking.k8s.io/allow-dns created
```

걸린 정책을 봅니다.

```bash
kubectl get networkpolicy -n np-lab
```

```text
NAME              POD-SELECTOR   AGE
allow-dns         <none>         5s
backend-to-db     app=backend    8s
db-from-backend   app=db         8s
deny-all          <none>         1m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`<none>`은 `podSelector: {}`, 즉 모든 Pod라는 뜻입니다. 정책 하나를 자세히 읽으려면 `kubectl describe networkpolicy db-from-backend -n np-lab`을 씁니다.

## STEP 16. 누가 통하나

```bash
kubectl exec -n np-lab backend -- wget -qO- -T 3 db
```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

```bash
kubectl exec -n np-lab hacker -- wget -qO- -T 3 db
```

```text
wget: download timed out
command terminated with exit code 1
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

backend만 db에 닿고, hacker는 막힙니다. 이번에는 DNS가 열려 있어서 `bad address`가 아니라 접속 시간 초과로 실패합니다. backend는 deny-all, backend-to-db, allow-dns의 허용을 더한 만큼 나갈 수 있고, db는 db-from-backend가 허용한 backend만 받습니다. web Pod가 뚫려도 db로 곧장 건너가는 길이 끊겼습니다.

실무 클러스터를 만들 때는 NetworkPolicy를 지원하는 CNI인지 반드시 확인하고, 정책을 건 뒤에는 이 실습처럼 **실제 접속으로 검증**합니다.

## 성공하면

- STEP 12: `Welcome to nginx!` HTML이 나온다
- STEP 14: 이름은 `bad address`, IP는 타임아웃 (경로 A)
- STEP 16: backend 성공, hacker 타임아웃 (경로 A)
- 경로 B: 모두 통과하고, 차단은 강사 시연 화면으로 확인

## 막히면

??? failure "error: unable to upgrade connection: container not found"
    Pod가 아직 생성 중입니다. `kubectl get pods -n np-lab`으로 셋 다 `Running`인지 보고 진행합니다.

??? failure "STEP 16에서 backend도 실패"
    1. allow-dns.yaml을 적용했는지 봅니다. DNS가 막혀 있으면 `bad address 'db'`가 나옵니다.
    2. `kubectl get pods -n np-lab --show-labels`로 backend 라벨이 `app=backend`, db 라벨이 `app=db`인지 봅니다.
    3. 정책 파일을 손으로 만들었다면 들여쓰기를 내려받기 파일과 비교합니다. 들여쓰기가 다르면 전혀 다른 규칙이 됩니다. `kubectl describe networkpolicy -n np-lab`으로 읽어 봅니다.

??? failure "STEP 14에서 hacker가 여전히 Welcome to nginx!"
    맨 위의 **환경에 따라 다를 수 있음** 표에서 경로 B입니다. CNI가 NetworkPolicy를 강제하지 않는 클러스터입니다. 정책 파일은 그대로 두고 강사 시연으로 차단 결과를 봅니다.

??? failure "allow-dns를 고쳐 보다가 kube-system 전체로 열림"
    `podSelector` 앞에 `-`를 붙이면 `namespaceSelector`와 별개 항목이 되어 OR 조건이 됩니다. "kube-system의 아무 Pod" 또는 "np-lab의 아무 Pod"로 넓어집니다. 내려받기 파일처럼 `-` 없이 `namespaceSelector`와 같은 칸에 둡니다.

??? failure "error: the path \"deny-all.yaml\" does not exist"
    작업 폴더에 파일이 없습니다. `ls`로 확인하고 `cd`로 작업 폴더로 이동합니다. Windows 메모장으로 만들었다면 이름이 `deny-all.yaml.txt`일 수 있습니다.

## 정리

다음 실습에서 쓰지 않습니다. 네임스페이스를 지우면 Pod, Service, 정책이 함께 지워집니다.

```bash
kubectl delete ns np-lab
```
