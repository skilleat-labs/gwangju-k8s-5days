# 실습 4. Pod 들여다보기와 라벨

!!! abstract "이번 실습에서 할 것"
    - `describe`의 Events를 읽고, 로그를 보고, Pod 안으로 들어가 봅니다.
    - 라벨로 Pod를 골라냅니다.
    - Pod는 지우면 끝이고, 같은 이름으로 다시 만들면 UID가 다른 새 Pod라는 것을 직접 봅니다.

**소요 시간** 약 12분 (STEP 1~2 7분, STEP 3~5 5분), **교안** DAY 2 섹션 4

## 준비

[실습 3](lab-3.md)에서 만든 `web`, `web2` Pod가 Running이어야 합니다.

```bash
kubectl get pods
```

```text
NAME   READY   STATUS    RESTARTS   AGE
web    1/1     Running   0          5m
web2   1/1     Running   0          3m
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

없으면 실습 3의 STEP 1, STEP 2를 다시 합니다.

!!! tip "get pods 읽는 법"
    - `READY`: 준비된 컨테이너 수 / 전체 컨테이너 수. `1/1`이면 정상, `0/1`이면 아직 요청을 받을 준비가 안 됐습니다.
    - `STATUS`: 지금 단계나 멈춘 이유. `ContainerCreating`(이미지 받는 중), `ImagePullBackOff`(이미지 이름, 태그 오류), `CrashLoopBackOff`(떴다가 계속 죽음). Running이어도 READY를 함께 봅니다.
    - `RESTARTS`: Pod는 그대로인데 안의 컨테이너가 다시 시작된 횟수. 숫자가 오르면 `kubectl logs POD이름 --previous`로 원인을 봅니다.

 1일차 도커 명령과 짝을 지어 보면 쉽습니다.

| 하는 일 | DAY 1 docker | DAY 2 kubectl |
|---|---|---|
| 목록 | `docker ps` | `kubectl get pods` |
| 로그 | `docker logs web` | `kubectl logs web` |
| 안으로 접속 | `docker exec -it web sh` | `kubectl exec -it web -- sh` |
| 자세히 보기 | `docker inspect web` | `kubectl describe pod web` |
| 삭제 | `docker rm -f web` | `kubectl delete pod web` |

## STEP 1. 자세히, 로그, 안으로

먼저 `describe`로 자세히 봅니다. 출력 **맨 아래 Events**를 읽습니다.

```bash
kubectl describe pod web
```

```text
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  6m    default-scheduler  Successfully assigned default/web to docker-desktop
  Normal  Pulling    6m    kubelet            Pulling image "nginx:1.27"
  Normal  Pulled     6m    kubelet            Successfully pulled image "nginx:1.27" in 5.1s
  Normal  Created    6m    kubelet            Created container: web
  Normal  Started    6m    kubelet            Started container web
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

Scheduled(scheduler가 노드 배정) → Pulled(kubelet이 이미지 받기) → Created → Started 순서입니다. 강의에서 본 "apply 한 번의 여정"이 그대로 기록되어 있습니다. Pod가 안 뜰 때 가장 먼저 보는 곳이 이 Events입니다.

!!! note "환경에 따라 다를 수 있음"
    이미지가 이미 노드에 있으면 `Pulling` 없이 `Container image "nginx:1.27" already present on machine`으로 나옵니다. Pod를 만든 지 1시간이 넘으면 Events가 비어 있을 수 있습니다.

로그를 봅니다.

```bash
kubectl logs web
```

```text
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
...
/docker-entrypoint.sh: Configuration complete; ready for start up
```
<small>출력은 예시입니다. 아직 아무도 접속하지 않았으니 nginx 시작 메시지만 보입니다.</small>

Pod 안으로 들어갑니다. 실행할 명령(`sh`) 앞에 **`--`를 꼭 넣습니다.** `docker exec`와 다른 점입니다.

```bash
kubectl exec -it web -- sh
```

프롬프트가 `#`으로 바뀌면 컨테이너 안입니다. nginx 기본 페이지 파일을 보고 나옵니다.

```bash
ls /usr/share/nginx/html
```

```text
50x.html  index.html
```

```bash
exit
```

안으로 들어가지 않고 명령 하나만 실행할 수도 있습니다. 컨테이너 안 명령은 큰따옴표로 감쌉니다.

```bash
kubectl exec web -- sh -c "ls /usr/share/nginx/html"
```

## STEP 2. 라벨 보기와 고르기

Pod에 붙은 라벨(이름표)을 봅니다.

```bash
kubectl get pods --show-labels
```

```text
NAME   READY   STATUS    RESTARTS   AGE   LABELS
web    1/1     Running   0          8m    run=web
web2   1/1     Running   0          6m    run=web2
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

`kubectl run`이 자동으로 `run=이름` 라벨을 붙였습니다. 이 라벨로 하나만 골라냅니다.

```bash
kubectl get pods -l run=web
```

```text
NAME   READY   STATUS    RESTARTS   AGE
web    1/1     Running   0          8m
```

라벨과 셀렉터는 지금은 조회에만 쓰지만, 실습 5의 Deployment와 3일차의 Service가 자기 Pod를 찾을 때 바로 이 방법을 씁니다.

## STEP 3. 지금 IP와 UID를 적어 둔다

`-o wide`로 Pod의 IP와 NODE를 봅니다. **IP를 메모해 둡니다.**

```bash
kubectl get pod web -o wide
```

```text
NAME   READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
web    1/1     Running   0          9m    10.1.0.12   docker-desktop   <none>           <none>
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

UID도 봅니다. UID는 쿠버네티스가 오브젝트마다 붙이는 고유 번호입니다. **UID도 메모해 둡니다.**

```bash
kubectl get pod web -o jsonpath="{.metadata.uid}"
```

```text
3f6c2a1e-8b7d-4c1a-9e2f-0a1b2c3d4e5f
```
<small>출력은 예시입니다. UID는 사람마다 다릅니다.</small>

## STEP 4. 지우고 같은 이름으로 다시

Pod를 지웁니다.

```bash
kubectl delete pod web
```

```bash
kubectl get pods
```

```text
NAME   READY   STATUS    RESTARTS   AGE
web2   1/1     Running   0          8m
```

`web`은 사라졌고, 기다려도 다시 생기지 않습니다. Pod를 직접 만들면 아무도 다시 만들어 주지 않습니다. 1일차에 `docker kill`한 컨테이너와 같습니다.

같은 이름, 같은 이미지로 다시 만듭니다.

```bash
kubectl run web --image=nginx:1.27
```

## STEP 5. UID와 IP를 다시 본다

```bash
kubectl get pod web -o wide
```

```text
NAME   READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
web    1/1     Running   0          5s    10.1.0.15   docker-desktop   <none>           <none>
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

```bash
kubectl get pod web -o jsonpath="{.metadata.uid}"
```

```text
9a7d1c3b-2e4f-4b6a-8c0d-1e2f3a4b5c6d
```
<small>출력은 예시입니다. UID는 사람마다 다릅니다.</small>

STEP 3에서 적어 둔 값과 비교합니다. **UID가 다르면 성공입니다.** 이름도 이미지도 같지만 지운 Pod가 돌아온 것이 아니라 새 Pod입니다. IP도 대개 새로 받습니다. Pod IP는 Pod가 살아 있는 동안만 쓰는 임시 주소라서, 다른 앱이 Pod IP를 적어 두고 쓰면 안 됩니다. 1일차 도커의 한계 5번 "주소가 바뀐다"와 같은 문제이고, 3일차 Service가 이 문제를 풉니다.

!!! note "환경에 따라 다를 수 있음"
    노드가 한 대뿐이라 IP가 우연히 같은 번호로 다시 잡힐 수 있습니다. 그래도 UID가 다르면 새 Pod이고, "IP는 유지된다는 보장이 없다"는 것이 핵심입니다.

## 성공하면

- STEP 1: Events에 Scheduled → Pulled → Created → Started
- STEP 2: `-l run=web` 결과에 `web` 하나만
- STEP 4: 지운 `web`은 아무도 다시 안 만든다
- STEP 5: UID가 다르다. IP는 같을 수도 있다

## 막히면

??? failure "error: exec [POD] [COMMAND] is not supported anymore. Use exec [POD] -- [COMMAND] instead"
    `--`를 빼고 `kubectl exec -it web sh`로 실행했습니다. `kubectl exec -it web -- sh`처럼 실행할 명령 앞에 `--`를 넣습니다. 예전 kubectl은 오류 대신 `DEPRECATED` 경고만 보여 줍니다. 어느 쪽이든 `--`를 넣으면 됩니다.

??? failure "OCI runtime exec failed: exec failed: ... \"bash\": executable file not found in $PATH"
    이미지에 bash가 없습니다. `sh`를 씁니다.

??? failure "sh: kubectl: not found"
    아직 컨테이너 안에 있는 상태에서 kubectl 명령을 쳤습니다. 프롬프트가 `#`이면 `exit`로 먼저 빠져나옵니다.

??? failure "Error from server (NotFound): pods \"web\" not found"
    STEP 4에서 지운 뒤 다시 만들지 않았습니다. `kubectl run web --image=nginx:1.27`을 실행합니다.

??? failure "STEP 2에서 No resources found in default namespace."
    라벨 값 오타입니다. `--show-labels` 결과의 LABELS 칸에 적힌 그대로 `-l run=web`으로 씁니다. 등호 앞뒤에 공백을 넣지 않습니다.

## 정리

Pod를 직접 만드는 실습은 여기까지입니다. `web`, `web2` Pod를 지웁니다. 실습 5부터는 Deployment가 Pod를 대신 만들어 줍니다.

```bash
kubectl delete pod web web2
```

```text
pod "web" deleted
pod "web2" deleted
```

`pod.yaml` 파일은 지우지 않아도 됩니다.
