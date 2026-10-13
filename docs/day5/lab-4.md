# 실습 4. 미니 블로그 YAML 빈칸 채우기

!!! abstract "이번 실습에서 할 것"
    - 1일차 도커 미니 블로그를 쿠버네티스 YAML 두 장으로 옮깁니다.
    - 리소스 사이를 잇는 이름과 포트 빈칸 8개를 스스로 채웁니다.
    - `--dry-run=client`로 문법만 검사합니다 (아직 만들지 않습니다).

**소요 시간** 약 10분, **교안** DAY 5 섹션 4

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

3일차에 설치한 Envoy Gateway가 있어야 frontend 파일의 GatewayClass, Gateway, HTTPRoute가 검사를 통과합니다. [오늘의 실습](index.md#envoy-check)의 확인을 아직 안 했다면 먼저 합니다.

### 1일차와 비교

| 1일차 도커 | 5일차 쿠버네티스 |
|---|---|
| `docker network create blog-net` | Namespace `webapp` + Service 이름으로 찾기 |
| `docker run --name backend-service ...` | Deployment `backend` (replicas 1) + Service `backend-service:5000` |
| `docker run -p 8080:80 ... frontend` | Deployment `frontend` (replicas 2) + Service `frontend-service:80` + Gateway |
| 컨테이너 안 `/app/data/data.json` | PVC `blog-pvc`를 `/app/data`에 붙여 글 보존 |
| 죽으면 사람이 살린다 | 죽어도 Deployment가 다시 띄운다 |

backend를 replicas 1로 두는 이유: Docker Desktop의 기본 저장소(hostpath)는 `ReadWriteOnce`, 즉 노드 하나에서만 붙습니다. 이 앱은 글을 파일 하나에 쓰므로 Pod 두 개가 동시에 쓰면 꼬일 수 있습니다.

## STEP 16. 파일 2개를 편집기로 연다

빈칸 문제 파일 두 개를 내려받아 작업 폴더에 넣습니다.

[miniblog-backend-blank.yaml 내려받기](../files/day5/miniblog-backend-blank.yaml){ download }

[miniblog-frontend-blank.yaml 내려받기](../files/day5/miniblog-frontend-blank.yaml){ download }

교안과 같은 이름이 되도록 이름을 바꿉니다. `mv`는 PowerShell에서도 같은 뜻으로 동작합니다. VS Code 탐색기에서 이름을 바꿔도 됩니다.

```bash
mv miniblog-backend-blank.yaml miniblog-backend.yaml
```

```bash
mv miniblog-frontend-blank.yaml miniblog-frontend.yaml
```

```bash
code .
```

`miniblog-backend.yaml`에는 Namespace, PVC, backend Deployment, backend-service가 있습니다.

```yaml title="miniblog-backend.yaml (빈칸 문제)"
--8<-- "files/day5/miniblog-backend-blank.yaml"
```

`miniblog-frontend.yaml`에는 frontend Deployment와 Service, 입구인 GatewayClass, Gateway, HTTPRoute가 있습니다.

```yaml title="miniblog-frontend.yaml (빈칸 문제)"
--8<-- "files/day5/miniblog-frontend-blank.yaml"
```

GatewayClass `blog-gw-class`의 `controllerName`은 3일차에 설치한 Envoy Gateway가 맡겠다는 표시입니다. PVC는 `storageClassName`을 생략해 Docker Desktop의 기본 StorageClass를 씁니다.

## STEP 17. 빈칸 (1) ~ (8) 채우고 저장

밑줄 부분(`(1)____` 전체)을 지우고 알맞은 값을 씁니다. 따옴표 안의 빈칸 (2)는 따옴표를 남겨 둡니다.

| 빈칸 | 파일 | 힌트 |
|---|---|---|
| (1) | backend | 저장 공간을 요청하는 리소스 |
| (2) | backend | 노드 하나에서 읽고 쓰기 |
| (3) | backend | backend가 글을 저장하는 폴더 (1일차 data.json이 있던 곳) |
| (4) | backend | 위에서 만든 요청서 이름 |
| (5) | backend | frontend가 백엔드를 찾는 이름 (1일차 `docker run --name`) |
| (6) | backend | backend 컨테이너 포트 |
| (7) | frontend | 이 경로를 붙일 Gateway |
| (8) | frontend | frontend-service의 포트 |

다 채웠으면 저장(`Ctrl + S`, Mac은 `⌘ + S`)하고, 채팅에 (1)~(8) 답을 올립니다.

??? success "정답 보기 (1) ~ (4)"
    | 빈칸 | 정답 | 이유 |
    |---|---|---|
    | (1) | `PersistentVolumeClaim` | 4일차의 저장 공간 요청서 |
    | (2) | `ReadWriteOnce` | Docker Desktop hostpath에 맞는 접근 모드 |
    | (3) | `/app/data` | 1일차에 data.json이 있던 폴더. 여기를 PVC로 덮어야 Pod가 바뀌어도 글이 남는다 |
    | (4) | `blog-pvc` | 같은 파일 위쪽 PVC의 `metadata.name`과 글자 하나까지 같아야 한다 |

??? success "정답 보기 (5) ~ (8)"
    | 빈칸 | 정답 | 이유 |
    |---|---|---|
    | (5) | `backend-service` | frontend 이미지가 이 이름으로 백엔드를 찾는다. 1일차 컨테이너 이름이 하던 일을 Service 이름이 한다 |
    | (6) | `5000` | Flask 백엔드가 듣는 포트 |
    | (7) | `blog-gw` | HTTPRoute가 어느 Gateway에 붙을지 정하는 `parentRefs` |
    | (8) | `80` | frontend-service가 여는 포트 |

??? success "정답 파일 전체: miniblog-backend.yaml"
    ```yaml title="miniblog-backend-answer.yaml"
    --8<-- "files/day5/miniblog-backend-answer.yaml"
    ```
    [miniblog-backend-answer.yaml 내려받기](../files/day5/miniblog-backend-answer.yaml){ download }

??? success "정답 파일 전체: miniblog-frontend.yaml"
    ```yaml title="miniblog-frontend-answer.yaml"
    --8<-- "files/day5/miniblog-frontend-answer.yaml"
    ```
    [miniblog-frontend-answer.yaml 내려받기](../files/day5/miniblog-frontend-answer.yaml){ download }

!!! tip "시간 안에 다 못 채웠다면"
    정답 파일 두 개를 내려받아 이름을 `miniblog-backend.yaml`, `miniblog-frontend.yaml`로 바꿔 쓰면 됩니다. 실습 5는 이 두 파일로 진행합니다.

    ```bash
    mv miniblog-backend-answer.yaml miniblog-backend.yaml
    ```

    ```bash
    mv miniblog-frontend-answer.yaml miniblog-frontend.yaml
    ```

    같은 이름의 파일이 이미 있으면 Windows에서 오류가 납니다. 빈칸 파일을 먼저 지우거나 `mv -Force`(Windows) / `mv -f`(Mac)를 씁니다.

## STEP 18. 문법만 검사 (아직 만들지 않는다)

`--dry-run=client`는 실제로 만들지 않고 YAML 문법과 형식만 검사합니다.

```bash
kubectl apply --dry-run=client -f miniblog-backend.yaml
```

```text
namespace/webapp created (dry run)
persistentvolumeclaim/blog-pvc created (dry run)
deployment.apps/backend created (dry run)
service/backend-service created (dry run)
```

```bash
kubectl apply --dry-run=client -f miniblog-frontend.yaml
```

```text
deployment.apps/frontend created (dry run)
service/frontend-service created (dry run)
gatewayclass.gateway.networking.k8s.io/blog-gw-class created (dry run)
gateway.gateway.networking.k8s.io/blog-gw created (dry run)
httproute.gateway.networking.k8s.io/frontend-route created (dry run)
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

리소스마다 `created (dry run)`이 나오면 문법은 통과입니다. 단, dry-run은 **이름이 서로 맞는지까지는 검사하지 못합니다.** (4)나 (5)를 틀려도 통과하니 정답과 비교해 봅니다.

## 성공하면

- 리소스마다 `created (dry run)`
- 채팅에 (1)~(8) 답 올리기

## 막히면

??? failure "error parsing miniblog-backend.yaml: error converting YAML to JSON: yaml: line N: ..."
    들여쓰기나 따옴표 문제입니다. 오류의 `line N`으로 가서 위아래 줄과 칸이 맞는지, 빈칸 (2)의 따옴표가 짝이 맞는지 봅니다.

??? failure "found character that cannot start any token"
    탭 문자로 들여썼습니다. YAML은 스페이스만 씁니다. VS Code 오른쪽 아래 **Spaces: 2**인지 확인하고 그 줄을 다시 들여씁니다.

??? failure "no matches for kind \"(1)____...\" / Object 'Kind' is missing"
    빈칸을 채우지 않았거나 밑줄이 남아 있습니다. `(1)____` 전체를 지우고 정답만 남깁니다.

??? failure "no matches for kind \"GatewayClass\" in version \"gateway.networking.k8s.io/v1\""
    Gateway API CRD가 없습니다. 3일차 Envoy Gateway 설치가 지워졌습니다. [3일차 실습 5](../day3/lab-5.md)의 설치 STEP을 다시 합니다.

??? failure "mv : 'miniblog-backend.yaml' 파일이 이미 있으므로 만들 수 없습니다 (Windows)"
    같은 이름의 파일이 있습니다. `mv -Force 원래이름 새이름`으로 덮어씁니다.

??? failure "error: the path \"miniblog-backend.yaml\" does not exist"
    이름을 아직 안 바꿨거나 다른 폴더에 받았습니다. `ls`로 작업 폴더의 파일 이름을 봅니다. 브라우저가 `miniblog-backend-blank (1).yaml`처럼 이름을 바꿔 저장했을 수도 있습니다.

## 정리

아직 아무것도 만들지 않았습니다. 두 파일은 **지우지 마세요.** 실습 5에서 그대로 배포합니다.
