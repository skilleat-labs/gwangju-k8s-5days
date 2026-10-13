# 실습 6. 고장 난 미니 블로그 고치기

!!! abstract "이번 실습에서 할 것"
    - 세 군데가 고장 난 미니 블로그를 올립니다.
    - get, describe, logs, events 순서로 **증상에서 원인으로** 좁혀 갑니다.
    - 하나씩 고쳐서 글 등록까지 되게 만듭니다.

**소요 시간** 약 12분, **교안** DAY 5 섹션 6

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

고장 난 블로그 파일을 내려받아 작업 폴더에 넣습니다. 이 파일은 `broken` 네임스페이스에 Namespace, PVC, backend, frontend를 모두 만듭니다. Gateway는 없고 마지막에 port-forward로 엽니다.

[broken-blog.yaml 내려받기](../files/day5/broken-blog.yaml){ download }

!!! warning "YAML을 눈으로 훑어서 찾지 마세요"
    실무 YAML은 수백 줄이라 눈으로 찾을 수 없습니다. 반드시 아래 루틴대로 **증상부터** 보고, 증상이 가리키는 줄을 고칩니다. 파일 내용은 답을 다 찾은 뒤 맨 아래에서 볼 수 있습니다.

### 장애 찾기 루틴: 넓게 보고 좁혀 간다

| 순서 | 명령 | 보는 곳 |
|---|---|---|
| 1. get | `kubectl get pods,pvc -n broken` | STATUS, READY로 어디가 이상한지 |
| 2. describe | `kubectl describe pod -n broken -l app=backend` | 맨 아래 **Events**에 쿠버네티스가 본 원인 |
| 3. logs | `kubectl logs -n broken -l app=frontend` | 앱이 남긴 오류. 재시작됐으면 `--previous` |
| 4. events | `kubectl get events -n broken --sort-by=.lastTimestamp` | 네임스페이스 전체를 시간순으로 |

연결 문제(Pod는 Running인데 기능이 안 됨)는 events에 안 나옵니다. 그때는 Service 다음에 EndpointSlice를 봅니다 (3일차 실습 4).

## STEP 25. 고장 난 블로그 올리기

```bash
kubectl apply -f broken-blog.yaml
```

```text
namespace/broken created
persistentvolumeclaim/blog-pvc created
deployment.apps/backend created
service/backend-service created
deployment.apps/frontend created
service/frontend-service created
```

모두 `created`로 성공합니다. **apply가 성공했다고 앱이 동작하는 것은 아닙니다.**

## STEP 26. 루틴대로 좁혀 가기

아래 명령을 순서대로 쓰면서 증상을 하나씩 찾습니다.

```bash
kubectl get pods,pvc -n broken
```

```bash
kubectl describe pod -n broken -l app=backend
```

```bash
kubectl get events -n broken --sort-by=.lastTimestamp
```

```bash
kubectl get endpointslices -n broken
```

고장은 **한 번에 하나씩** 드러납니다. 하나를 찾으면 VS Code에서 broken-blog.yaml을 고치고 STEP 27로 다시 올린 뒤, STEP 26을 처음부터 다시 합니다.

찾은 원인은 채팅에 **"증상 → 원인"** 형식으로 올립니다. 예: `PVC Pending → ...`

??? tip "힌트 1: 첫 번째 고장"
    `get pods,pvc`에서 backend Pod와 PVC가 둘 다 `Pending`입니다. Pod는 PVC를 기다리는 중이니 **PVC 쪽**을 describe합니다.

    ```bash
    kubectl describe pvc blog-pvc -n broken
    ```

    Events에 나오는 StorageClass 이름이 클러스터에 실제로 있는지 봅니다.

    ```bash
    kubectl get storageclass
    ```

??? tip "힌트 2: 두 번째 고장"
    PVC가 `Bound`가 되면 backend Pod가 이번에는 `ErrImagePull` 또는 `ImagePullBackOff`입니다. describe의 Events에 찍힌 **이미지 이름과 태그**를 1일차에 쓴 이미지(`skilleat/backend:v3-kb5`)와 한 글자씩 비교합니다.

??? tip "힌트 3: 세 번째 고장"
    Pod가 모두 `Running`인데 화면에서 글 등록만 실패합니다. Service 뒤에 Pod가 붙어 있는지 봅니다.

    ```bash
    kubectl get endpointslices -n broken -l kubernetes.io/service-name=backend-service
    ```

    ENDPOINTS가 비어 있으면 Service의 SELECTOR와 Pod의 LABELS를 비교합니다.

    ```bash
    kubectl get svc backend-service -n broken -o wide
    ```

    ```bash
    kubectl get pods -n broken --show-labels
    ```

    frontend 로그에도 백엔드에 연결하지 못한 흔적이 남습니다.

    ```bash
    kubectl logs -n broken -l app=frontend
    ```

## STEP 27. 고친 뒤 통째로 다시 (PVC는 바꿀 수 없다)

PVC의 `storageClassName`은 만든 뒤에 바꿀 수 없어서 `apply`만 다시 하면 `spec is immutable` 류의 오류가 납니다. 파일을 고칠 때마다 통째로 지우고 다시 만듭니다.

```bash
kubectl delete -f broken-blog.yaml
```

```bash
kubectl apply -f broken-blog.yaml
```

Namespace가 파일 안에 있어서 `delete -f`를 하면 `broken` 네임스페이스까지 지워지고, `apply -f`를 하면 다시 만들어집니다. 네임스페이스가 지워지는 동안 `apply`가 `object is being deleted` 오류를 내면 몇 초 뒤 다시 실행합니다.

세 군데를 다 고쳤으면 frontend를 열어 글 등록까지 확인합니다. 실습 5의 8080과 겹치지 않게 8081을 씁니다.

```bash
kubectl port-forward -n broken svc/frontend-service 8081:80
```

브라우저에서 **http://localhost:8081** 을 열고 글을 하나 씁니다. 목록에 나오면 끝입니다. 확인한 뒤 port-forward 터미널에서 `Ctrl + C`로 멈춥니다.

## 성공하면

- Pod `Running`, PVC `Bound`
- EndpointSlice에 주소가 있다
- port-forward로 글 등록

## 정답

다 찾은 뒤에 열어 봅니다.

??? success "정답 1: PVC Pending (StorageClass 없음)"
    **증상**: `get pods,pvc`에서 PVC와 backend Pod가 `Pending`.

    ```text
    NAME                             STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    persistentvolumeclaim/blog-pvc   Pending                                      fast-ssd       20s
    ```

    describe pvc의 Events:

    ```text
    Warning  ProvisioningFailed  ...  storageclass.storage.k8s.io "fast-ssd" not found
    ```
    <small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

    **원인**: 클러스터에 없는 `fast-ssd` StorageClass를 요청했습니다. `kubectl get storageclass`에는 `hostpath`만 있습니다.

    **고치기**: PVC의 `storageClassName: fast-ssd` 줄을 지웁니다 (기본 StorageClass 사용). 또는 `storageClassName: hostpath`로 바꿉니다.

??? success "정답 2: ImagePullBackOff (이미지 태그 오타)"
    **증상**: backend Pod가 `ErrImagePull` → `ImagePullBackOff`.

    ```text
    Warning  Failed   ...  Failed to pull image "skilleat/backend:v3-kb55": ... not found
    Warning  Failed   ...  Error: ErrImagePull
    Normal   BackOff  ...  Back-off pulling image "skilleat/backend:v3-kb55"
    ```
    <small>출력은 예시입니다. 실제로는 `failed to resolve reference` 같은 문구가 길게 붙습니다.</small>

    **원인**: 태그 `v3-kb55`가 Docker Hub에 없습니다.

    **고치기**: `image: skilleat/backend:v3-kb5`로 바로잡습니다.

??? success "정답 3: 빈 EndpointSlice (Service selector 오타)"
    **증상**: Pod는 모두 `Running`, 화면은 뜨는데 글 등록만 실패. EndpointSlice가 비어 있습니다.

    ```text
    NAME                    ADDRESSTYPE   PORTS     ENDPOINTS   AGE
    backend-service-8fz2q   IPv4          <unset>   <unset>     2m
    ```

    ```text
    NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE   SELECTOR
    backend-service   ClusterIP   10.96.77.10    <none>        5000/TCP   2m    app=backnd
    ```
    <small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

    **원인**: Service의 selector가 `app: backnd`, Pod 라벨은 `app=backend`입니다. 고른 Pod가 없어서 Service 뒤가 비었습니다. Service는 오류 이벤트를 남기지 않아서 events만 봐서는 못 찾습니다.

    **고치기**: `selector: {app: backend}`로 바로잡습니다.

??? success "고장 난 파일 전체 보기 (broken-blog.yaml)"
    ```yaml title="broken-blog.yaml"
    --8<-- "files/day5/broken-blog.yaml"
    ```

!!! info "왜 한 번에 하나씩 드러나나"
    PVC가 Pending이면 Pod가 노드에 배치되지 않아 이미지를 받으려는 시도조차 없습니다. 그래서 이미지 오류가 아직 안 보입니다. 이미지를 고쳐 Pod가 `Running`이 되어야 비로소 라벨 오류가 글 등록 실패로 드러납니다. 실무 장애도 이렇게 겹겹이 숨어 있습니다.

## 막히면

??? failure "error: no objects passed to apply / the path \"broken-blog.yaml\" does not exist"
    작업 폴더에 파일이 없습니다. `ls`로 확인하고 `cd`로 작업 폴더로 이동합니다. 브라우저가 `broken-blog (1).yaml`처럼 이름을 바꿔 저장했을 수 있습니다.

??? failure "The PersistentVolumeClaim \"blog-pvc\" is invalid: spec: Forbidden: spec is immutable after creation"
    PVC를 지우지 않고 `apply`만 다시 했습니다. STEP 27처럼 `kubectl delete -f broken-blog.yaml` 후 `apply`합니다.

??? failure "Error from server (Forbidden): ... namespace broken because it is being terminated"
    네임스페이스가 아직 지워지는 중입니다. `kubectl get ns broken`에 아무것도 안 나올 때까지 몇 초 기다린 뒤 다시 `apply`합니다.

??? failure "kubectl logs: container \"backend\" in pod ... is waiting to start"
    컨테이너가 아직 시작되지 않았습니다 (Pending이거나 이미지를 못 받음). 로그가 없는 단계이니 describe의 Events를 봅니다.

??? failure "Unable to listen on port 8081"
    8081을 다른 프로그램이 쓰고 있습니다. `8082:80`처럼 앞 숫자를 바꾸고 그 포트로 엽니다.

## 정리

```bash
kubectl delete ns broken webapp
```

실습 5의 `webapp`과 이 실습의 `broken`이 함께 지워집니다. 과정 전체를 정리하려면 아래 과정 마무리를 따라 합니다.

## 과정 마무리 { #wrap-up }

5일 동안 수고하셨습니다. 1일차에 `docker run`으로 손수 띄우던 미니 블로그를, 오늘 YAML 두 장으로 쿠버네티스에 맡기고, 지키고, 고쳤습니다. 막히면 **get, describe, logs, events**부터 보세요.

| 일차 | 한 일 |
|---|---|
| 1일차 컨테이너 | 컨테이너는 격리된 프로세스. 미니 블로그를 도커로 띄우고 한계 5가지를 겪었다 |
| 2일차 구조와 배포 | 클러스터 구조, Pod, Deployment. 자가복구, 확장, 롤링 업데이트 |
| 3일차 서비스 | Service, DNS, EndpointSlice, Gateway API. 변하지 않는 주소와 입구 |
| 4일차 설정과 데이터 | ConfigMap, Secret, probe, limits, PVC. 이미지 밖으로 빼고 지킨다 |
| 5일차 보안과 종합 | root 금지, RBAC, NetworkPolicy. 미니 블로그를 YAML로 옮기고 고쳤다 |

### 과정 전체 정리

노트북을 원래대로 돌려놓고 싶을 때 아래 순서로 합니다. 계속 연습할 거라면 하지 않아도 됩니다.

**1. 5일차 네임스페이스 지우기.** 이미 지운 것은 건너뜁니다.

```bash
kubectl delete ns webapp broken secure rbac-lab np-lab --ignore-not-found
```

**2. Gateway 리소스 지우기.** GatewayClass는 네임스페이스에 속하지 않아서 따로 지웁니다. 3일차의 `eg`도 남아 있으면 함께 지웁니다.

```bash
kubectl delete gatewayclass blog-gw-class --ignore-not-found
```

```bash
kubectl delete gateway eg --ignore-not-found
```

```bash
kubectl delete gatewayclass eg --ignore-not-found
```

**3. Envoy Gateway 제거.** 3일차에 설치할 때 쓴 `install.yaml`로 지웁니다. 버전은 [3일차 실습 5](../day3/lab-5.md)에서 설치한 것과 같아야 합니다.

```bash
kubectl delete -f https://github.com/envoyproxy/gateway/releases/download/v1.9.2/install.yaml --ignore-not-found
```

`envoy-gateway-system` 네임스페이스와 Gateway API CRD가 함께 지워집니다. 1~2분 걸릴 수 있습니다. 3일차에 파일로 내려받아 설치했다면 `kubectl delete -f install.yaml`로 지워도 같습니다.

**4. 남은 것 확인.** 앞 일차에서 `default` 네임스페이스에 만든 것이 남아 있는지 봅니다.

```bash
kubectl get all,pvc,configmap,secret
```

`service/kubernetes`와 `configmap/kube-root-ca.crt`는 쿠버네티스 기본 리소스이니 그대로 둡니다. 나머지는 `kubectl delete 종류 이름`으로 지웁니다. 한꺼번에 비우고 싶으면 Docker Desktop의 Kubernetes 설정에서 **Reset cluster**를 누릅니다. 클러스터 안의 모든 리소스가 지워집니다.

**5. 클러스터 끄기.** Docker Desktop 왼쪽 메뉴 **Kubernetes** → **Stop**을 누릅니다. 버전에 따라 **Settings → Kubernetes**에서 **Enable Kubernetes**를 끄고 **Apply**를 누릅니다. 노트북 메모리를 돌려받습니다. 다시 쓸 때는 [Kubernetes 켜기](../setup/3-kubernetes.md)를 따라 켭니다.

**6. (선택) 이미지 지우기.** 디스크 공간이 필요하면 실습 이미지를 지웁니다.

```bash
docker rmi skilleat/frontend:v3-kb5 skilleat/backend:v3-kb5 nginx:1.27 busybox:1.36
```

사용 중이라 안 지워지는 이미지가 있으면 Docker Desktop을 다시 시작한 뒤 다시 실행합니다.
