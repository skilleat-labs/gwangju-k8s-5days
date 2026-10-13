# 실습 3. readiness를 일부러 실패시키기

!!! abstract "이번 실습에서 할 것"
    - readinessProbe가 실패한 Pod가 살아 있는 채로 트래픽에서만 빠지는 것을 봅니다.
    - 3일차에 본 EndpointSlice의 `ready`가 `false`로 바뀌는 것을 직접 확인합니다.
    - 재시작(liveness)과 트래픽 제외(readiness)의 차이를 봅니다.

**소요 시간** 약 12분, **교안** DAY 4 섹션 3

## 준비

`day4` 폴더에서 합니다. 이번 실습에서 쓸 파일은 `probe-web.yaml`입니다.

```yaml title="probe-web.yaml"
--8<-- "files/day4/probe-web.yaml"
```
[probe-web.yaml 내려받기](../files/day4/probe-web.yaml){ download }

| 부분 | 뜻 |
|---|---|
| `readinessProbe` | 5초마다 `/index.html`을 요청합니다. 응답 코드가 200 이상 400 미만이면 통과입니다. |
| `failureThreshold: 1` | 한 번만 실패해도 준비 안 됨으로 봅니다 (실습에서 빨리 보려고). |
| liveness 없음 | 그래서 실패해도 재시작하지 않습니다. RESTARTS는 0으로 남습니다. |

EndpointSlice가 무엇인지 기억나지 않으면 [3일차 실습 4 EndpointSlice와 연결 장애](../day3/lab-4.md)를 먼저 봅니다.

## STEP 10. 배포하고 Service 붙이기

```bash
kubectl apply -f probe-web.yaml
```

3일차에 배운 expose로 같은 이름의 Service를 붙입니다.

```bash
kubectl expose deployment web-probe --port 80
```

```text
deployment.apps/web-probe created
service/web-probe exposed
```

## STEP 11. 두 Pod 모두 ready인지 (DAY 3)

Pod 두 개의 이름과 IP를 봅니다. 이 출력은 STEP 12, 14, 15에서 다시 보니 창에 남겨 둡니다.

```bash
kubectl get pods -l app=web-probe -o wide
```

```text
NAME                         READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
web-probe-6d8f7c9b5d-abcde   1/1     Running   0          30s   10.1.0.21   docker-desktop   <none>           <none>
web-probe-6d8f7c9b5d-fghij   1/1     Running   0          30s   10.1.0.22   docker-desktop   <none>           <none>
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

3일차처럼 Service 이름 라벨로 EndpointSlice를 봅니다.

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web-probe -o yaml
```

```text
endpoints:
- addresses:
  - 10.1.0.21
  conditions:
    ready: true
    serving: true
    terminating: false
- addresses:
  - 10.1.0.22
  conditions:
    ready: true
    serving: true
    terminating: false
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다. 실제 출력은 더 길고, `endpoints:` 부분만 옮겼습니다.</small>

주소 두 개 모두 `ready: true`입니다.

## STEP 12. Pod 하나의 점검 파일 지우기

STEP 11 목록에서 Pod 이름 하나를 복사해 `<Pod 이름>` 자리에 넣습니다. 꺾쇠 `< >`는 지우고 이름만 넣습니다.

```bash
kubectl exec <Pod 이름> -- rm /usr/share/nginx/html/index.html
```

예를 들어 Pod 이름이 `web-probe-6d8f7c9b5d-fghij`라면 이렇게 입력합니다.

```bash
kubectl exec web-probe-6d8f7c9b5d-fghij -- rm /usr/share/nginx/html/index.html
```

이제 이 Pod에 `/index.html`을 요청하면 404가 나서 readinessProbe가 실패합니다. 지운 Pod 이름과 IP를 기억해 둡니다.

## STEP 13. READY 0/1, RESTARTS는 0 그대로

5초쯤 기다린 뒤 봅니다.

```bash
kubectl get pods -l app=web-probe
```

```text
NAME                         READY   STATUS    RESTARTS   AGE
web-probe-6d8f7c9b5d-abcde   1/1     Running   0          2m
web-probe-6d8f7c9b5d-fghij   0/1     Running   0          2m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

파일을 지운 Pod는 READY가 `0/1`입니다. STATUS는 `Running`, RESTARTS는 `0`입니다. **살아 있지만 준비가 안 된** 상태입니다.

## STEP 14. 주소 목록에서 ready: false 확인

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web-probe -o yaml
```

```text
endpoints:
- addresses:
  - 10.1.0.21
  conditions:
    ready: true
    serving: true
    terminating: false
- addresses:
  - 10.1.0.22
  conditions:
    ready: false
    serving: false
    terminating: false
ports:
- port: 80
  protocol: TCP
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

| 주소 | 뜻 |
|---|---|
| `ready: true` 주소 | 파일이 있는 Pod. 트래픽을 받습니다. |
| `ready: false` 주소 | 파일을 지운 Pod. Service가 트래픽을 보내지 않습니다. |

STEP 11의 `get pods -o wide` IP와 맞춰 봅니다. `ready: false`인 IP가 STEP 12에서 파일을 지운 Pod의 IP입니다. 주소가 목록에서 사라지는 것이 아니라, **남아 있되 `ready`만 `false`**가 됩니다.

!!! note "환경에 따라 다를 수 있음"
    Docker Desktop 버전에 따라 주소 순서, `serving` 값, 출력 모양이 조금 다를 수 있습니다. 확인할 것은 **파일을 지운 Pod의 IP에서 `ready: false`** 한 가지입니다. YAML이 길어 찾기 어려우면 describe로 봅니다. 주소마다 `Conditions` 아래 `Ready:` 값이 나옵니다.

    ```bash
    kubectl describe endpointslices -l kubernetes.io/service-name=web-probe
    ```

## STEP 15. 되돌리기: 그 Pod를 지우면 새로 뜬다

STEP 12에서 파일을 지운 Pod를 지웁니다.

```bash
kubectl delete pod <Pod 이름>
```

```bash
kubectl get pods -l app=web-probe
```

```text
NAME                         READY   STATUS    RESTARTS   AGE
web-probe-6d8f7c9b5d-abcde   1/1     Running   0          4m
web-probe-6d8f7c9b5d-klmno   1/1     Running   0          8s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

Deployment가 새 Pod를 만들었고, 새 Pod에는 `index.html`이 있으니 다시 `1/1`입니다.

## 성공하면

- STEP 11: endpoints에 주소 2개, 둘 다 `ready: true`
- STEP 13: 파일을 지운 Pod만 `0/1 Running`, RESTARTS `0`
- STEP 14: 그 IP만 `ready: false`
- STEP 15: 다시 2개 모두 `1/1`

## 막히면

??? failure "error: the server doesn't have a resource type \"endpointslice...\""
    리소스 이름 오타입니다. `endpointslices`(또는 단수 `endpointslice`)를 정확히 입력합니다.

??? failure "'<' 연산자는 나중에 사용하도록 예약되어 있습니다 (PowerShell) / zsh: no such file or directory: Pod (Mac)"
    `<Pod 이름>`을 꺾쇠까지 그대로 입력했습니다. STEP 11 목록에서 실제 이름을 복사해 꺾쇠 없이 넣습니다.

??? failure "STEP 14에서 두 주소 모두 ready: true"
    파일을 지우고 아직 5초가 안 지났거나, 다른 Pod의 파일을 지웠습니다. 5초쯤 뒤 STEP 13에서 `0/1` Pod가 있는지 먼저 확인합니다.

??? failure "Error from server (AlreadyExists): services \"web-probe\" already exists"
    Service를 이미 만들었습니다. 그대로 STEP 11로 갑니다.

??? failure "교안 장표처럼 줄 끝 역슬래시(\)로 나눠 쳤더니 PowerShell에서 오류"
    이 가이드처럼 **한 줄로** 입력합니다.

## 정리

Deployment와 Service를 지웁니다.

```bash
kubectl delete deployment web-probe
```

```bash
kubectl delete service web-probe
```
