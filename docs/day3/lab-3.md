# 실습 3. NodePort, LoadBalancer, port-forward로 각각 접속

!!! abstract "이번 실습에서 할 것"
    - 클러스터 밖(내 PC)에서 Service로 들어오는 세 가지 방법을 써 봅니다.
    - 세 방법 모두 같은 nginx 페이지가 열리는 것을 확인합니다.

**소요 시간** 약 12분, **교안** DAY 3 섹션 3

## 준비

- [실습 1](lab-1.md)의 `web` Deployment와 `web` Service가 남아 있는지 확인합니다.

```bash
kubectl get deploy,svc
```

- STEP 9에서 **터미널을 하나 더** 엽니다. 새 터미널도 작업 폴더로 이동합니다.

| 방법 | 접속 주소 (Docker Desktop) | 주로 쓰는 곳 | 주의 |
|---|---|---|---|
| ClusterIP | 클러스터 안에서만 | Pod끼리 호출 | 브라우저로는 안 열림 |
| NodePort | `localhost:30080` | 테스트, 온프레미스 | 포트 범위 30000~32767 |
| LoadBalancer | `localhost:8081` | 클라우드 운영 | Service마다 LB 하나 = 비용 |
| port-forward | `localhost:8082` | 개발, 디버깅 | 터미널 닫으면 끊김 |

## STEP 7. NodePort → 브라우저 localhost:30080

NodePort Service YAML을 `web-np.yaml`로 저장합니다. 실습 1의 `web-svc.yaml`에서 강조한 세 줄이 다릅니다. 이름은 `web`과 겹치지 않게 `web-np`로 했습니다. 이번에는 일부러 **포트 번호 세 개를 모두 다르게** 했습니다.

```yaml title="web-np.yaml" hl_lines="6 10 12"
--8<-- "files/day3/web-np.yaml"
```
[web-np.yaml 내려받기](../files/day3/web-np.yaml){ download }

| 줄 | 뜻 |
|---|---|
| `type: NodePort` | ClusterIP 위에 노드 포트를 더합니다 |
| `nodePort: 30080` | 노드가 밖에 여는 문. 생략하면 30000~32767 중 자동 배정 |
| `port: 8080` | Service가 받는 문. 클러스터 안에서는 `web-np:8080` |
| `targetPort: 80` | nginx가 **실제로 듣는** 포트. 그대로 80 |

```bash
kubectl apply -f web-np.yaml
```

```bash
kubectl get svc web-np
```

```text
NAME     TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
web-np   NodePort   10.104.33.21    <none>        8080:30080/TCP   5s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`PORT(S)`의 `8080:30080`에서 콜론 앞이 Service의 `port`, 뒤가 `nodePort`입니다. 요청은 **nodePort 30080 → port 8080 → targetPort 80** 순서로 들어갑니다.

```text
내 PC 브라우저 localhost:30080 ─▶ 노드 30080 ─▶ Service web-np 8080 ─▶ nginx 컨테이너 80
```

브라우저에서 [http://localhost:30080](http://localhost:30080)을 엽니다. 터미널로 확인하려면 아래처럼 합니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost:30080
    ```

    그냥 `curl`은 PowerShell에서 Invoke-WebRequest의 별칭이라 출력 모양이 다릅니다. 꼭 `curl.exe`로 씁니다.

=== "Mac (터미널)"

    ```bash
    curl http://localhost:30080
    ```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

??? tip "(선택) 클러스터 안에서는 web-np:8080"
    NodePort Service도 ClusterIP를 함께 받습니다. 클러스터 안의 Pod는 nodePort가 아니라 Service의 `port` 8080으로 부릅니다.

    ```bash
    kubectl run tmp --rm -it --restart=Never --image=busybox:1.36 -- wget -qO- web-np:8080
    ```

    `web-np:80`으로 부르면 Service가 그 포트를 받지 않으므로 접속이 실패합니다.

!!! note "환경에 따라 다를 수 있음"
    Docker Desktop의 kubeadm 방식에서 NodePort가 `localhost:30080`으로 열리지 않는 경우가 있습니다. 브라우저가 연결하지 못하면 이 STEP은 강사 화면으로 확인하고, STEP 9의 port-forward로 같은 페이지를 열어 봅니다.

## STEP 8. LoadBalancer → localhost:8081

이번에는 YAML 없이 `expose` 명령으로 LoadBalancer Service를 만듭니다. Service 포트는 8081, 컨테이너 포트는 80입니다.

```bash
kubectl expose deployment web --name web-lb --type LoadBalancer --port 8081 --target-port 80
```

```text
service/web-lb exposed
```

```bash
kubectl get svc web-lb
```

```text
NAME     TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
web-lb   LoadBalancer   10.98.150.7     localhost     8081:31234/TCP   5s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`EXTERNAL-IP`가 `localhost`면 [http://localhost:8081](http://localhost:8081)로 열립니다. `31234`처럼 뒤에 붙은 번호는 자동 배정된 nodePort입니다. LoadBalancer는 NodePort 위에 한 겹을 더 얹은 구조라서 이 번호가 함께 생깁니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost:8081
    ```

=== "Mac (터미널)"

    ```bash
    curl http://localhost:8081
    ```

!!! note "환경에 따라 다를 수 있음"
    클라우드(Azure, AWS)에서는 EXTERNAL-IP에 공인 IP가 들어오고, 클라우드가 없는 온프레미스에서는 `<pending>`으로 남습니다. Docker Desktop은 이 자리에 `localhost`를 채워 주는 것으로 알려져 있지만, kubeadm 방식에서는 아직 확인 중입니다.

    - `EXTERNAL-IP`가 `<pending>`이면 오류가 아닙니다. 이 STEP은 강사 화면으로 보고 넘어갑니다.
    - `localhost`로 나오는데 브라우저가 안 열리면 STEP 9의 port-forward로 같은 페이지를 엽니다.

## STEP 9. port-forward (새 터미널) → localhost:8082

port-forward는 Service 종류가 아니라 **kubectl 기능**입니다. 내 PC의 8082 포트를 클러스터 안으로 잠깐 잇는 터널입니다. 이 명령은 실행한 채로 계속 떠 있으므로 **새 터미널**에서 실행합니다.

```bash
kubectl port-forward svc/web 8082:80
```

```text
Forwarding from 127.0.0.1:8082 -> 80
Forwarding from [::1]:8082 -> 80
```

이 터미널은 그대로 두고, 원래 터미널이나 브라우저에서 [http://localhost:8082](http://localhost:8082)를 엽니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost:8082
    ```

=== "Mac (터미널)"

    ```bash
    curl http://localhost:8082
    ```

요청을 보낼 때마다 port-forward 터미널에 `Handling connection for 8082` 줄이 찍힙니다. 다 봤으면 port-forward 터미널에서 `Ctrl + C`로 끝냅니다.

!!! info "svc/web이라고 써도 Pod 하나에만 붙습니다"
    port-forward는 시작할 때 Service 뒤의 Pod **하나**를 골라 그 Pod에만 터널을 뚫습니다. 여러 Pod로 나눠 보내지 않고, 터미널을 닫으면 끊깁니다. 그래서 운영 트래픽용이 아니라 "나 혼자 잠깐 보는" 용도입니다. 실습 6에서 Gateway에 접속할 때 다시 씁니다.

## 성공하면

- STEP 7: `kubectl get svc web-np`에 `8080:30080/TCP`가 보이고 `localhost:30080`이 열립니다.
- STEP 8: `EXTERNAL-IP`가 `localhost`이고 `localhost:8081`이 열립니다.
- STEP 9: `Forwarding from 127.0.0.1:8082 -> 80`이 나오고 `localhost:8082`가 열립니다.
- 세 주소 모두 같은 `Welcome to nginx!` 페이지입니다.

## 막히면

??? failure "EXTERNAL-IP가 계속 <pending>"
    LoadBalancer를 채워 줄 구현이 없는 환경입니다. 오류가 아닙니다. 이 STEP은 강사 화면으로 보고 넘어가고, port-forward(STEP 9)로 접속합니다.

??? failure "The Service \"web-np\" is invalid: spec.ports[0].nodePort: Invalid value: 30080: provided port is already allocated"
    다른 Service가 이미 30080을 쓰고 있습니다. `kubectl get svc -A`로 누가 쓰는지 보고, `web-np.yaml`의 `nodePort`를 30081처럼 다른 번호(30000~32767)로 바꿔 다시 apply합니다.

??? failure "Error from server (AlreadyExists): services \"web-lb\" already exists"
    앞에서 이미 만들었습니다. `kubectl get svc web-lb`로 확인만 하면 됩니다. 처음부터 다시 하려면 `kubectl delete svc web-lb` 후 다시 expose합니다.

??? failure "Unable to listen on port 8082 ... bind: address already in use (또는 Only one usage of each socket address)"
    내 PC에서 다른 프로그램이 8082를 쓰고 있습니다. 앞 번호만 바꿉니다. 예: `kubectl port-forward svc/web 18082:80` 후 `localhost:18082` 접속.

??? failure "PowerShell에서 curl 결과가 StatusCode, Content 같은 표로 나옴"
    PowerShell의 `curl`은 Invoke-WebRequest의 별칭입니다. `curl.exe`로 입력하거나 브라우저로 확인합니다.

??? failure "브라우저에서 localhost:30080이 연결할 수 없음"
    1. `kubectl get svc web-np`의 `PORT(S)`가 `8080:30080/TCP`인지 봅니다.
    2. `kubectl get pods -l app=web`이 모두 `Running`인지 봅니다.
    3. 그래도 안 되면 환경 차이일 수 있습니다. STEP 9의 port-forward로 확인하고 넘어갑니다.

## 정리

NodePort와 LoadBalancer Service는 다음 실습에 필요 없습니다. port-forward 터미널은 `Ctrl + C`로 끝냅니다.

```bash
kubectl delete svc web-np web-lb
```

!!! warning "지우지 마세요"
    `web` Deployment와 `web` Service(ClusterIP)는 실습 4에서 씁니다.
