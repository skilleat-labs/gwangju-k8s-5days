# 실습 6. PVC로 데이터 지키기

!!! abstract "이번 실습에서 할 것"
    - PVC를 만들고 PV가 자동으로 생기는 것(동적 프로비저닝)을 봅니다.
    - 실습 5와 같은 실험을 PVC로 다시 해서, Pod를 지워도 데이터가 남는 것을 봅니다.

**소요 시간** 약 12분, **교안** DAY 4 섹션 6

## 준비

`day4` 폴더에서 합니다. 실습 5의 `mysql` Deployment와 `mysql-secret`(비밀번호 `skilleat`)이 남아 있는 상태에서 시작합니다.

이번 실습에서 쓸 파일은 두 개입니다.

**1. PVC (저장 공간 요청서)**

```yaml title="mysql-pvc.yaml"
--8<-- "files/day4/mysql-pvc.yaml"
```
[mysql-pvc.yaml 내려받기](../files/day4/mysql-pvc.yaml){ download }

| 부분 | 뜻 |
|---|---|
| `storageClassName: hostpath` | Docker Desktop의 기본 StorageClass. 빼도 기본값이 쓰입니다. STEP 24에서 이름을 확인합니다. |
| `accessModes: [ReadWriteOnce]` | 한 노드에서 읽기, 쓰기 (RWO). |
| `storage: 1Gi` | 필요한 용량. |

**2. PVC를 쓰는 MySQL Deployment**

`mysql-emptydir.yaml`을 복사해 `mysql-pvc-deploy.yaml`로 저장하고 두 곳만 고칩니다. 아래 파일에서 바뀐 줄은 강조 표시했습니다.

```yaml title="mysql-pvc-deploy.yaml" hl_lines="6 7 19 20 21"
--8<-- "files/day4/mysql-pvc-deploy.yaml"
```
[mysql-pvc-deploy.yaml 내려받기](../files/day4/mysql-pvc-deploy.yaml){ download }

| 바뀐 곳 | 뜻 |
|---|---|
| `strategy: type: Recreate` | 옛 Pod를 먼저 끄고 새 Pod를 띄웁니다. 롤링 업데이트는 새 Pod를 먼저 띄우는데, 두 MySQL이 한 데이터 폴더를 동시에 열면 안 됩니다. |
| `persistentVolumeClaim: claimName: mysql-pvc` | `emptyDir` 대신 PVC 이름을 적습니다. Pod는 PV를 모르고 PVC 이름만 압니다. |

Secret, `mountPath` 등 나머지는 실습 5와 같습니다.

## STEP 24. 기본 StorageClass 확인

```bash
kubectl get storageclass
```

```text
NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
hostpath (default)   docker.io/hostpath   Delete          Immediate           false                  3d
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

이름 옆에 `(default)`가 있는 것이 기본 StorageClass입니다.

!!! note "환경에 따라 다를 수 있음"
    Docker Desktop 버전과 클러스터 방식에 따라 결과가 다를 수 있습니다.

    - **이름이 `hostpath`가 아니면**: `mysql-pvc.yaml`의 `storageClassName:` 값을 그 이름으로 바꾸거나, `storageClassName:` 줄을 지웁니다. 줄을 지우면 `(default)` StorageClass가 쓰입니다.
    - **`VOLUMEBINDINGMODE`가 `WaitForFirstConsumer`면**: STEP 25에서 PVC가 `Pending`이고 PV가 아직 없는 것이 정상입니다. STEP 26에서 Pod가 PVC를 쓰는 순간 PV가 생기고 `Bound`가 됩니다.
    - **`No resources found`로 StorageClass가 하나도 없으면**: PVC가 계속 `Pending`입니다. 강사에게 알려 주세요.

## STEP 25. emptyDir MySQL 지우고 PVC 만들기

실습 5의 emptyDir MySQL Deployment를 먼저 지웁니다. 같은 이름의 Deployment에 `strategy`만 바꿔 apply하면 오류가 나기 때문입니다 (아래 막히면 참고).

```bash
kubectl delete deployment mysql
```

PVC를 만듭니다.

```bash
kubectl apply -f mysql-pvc.yaml
```

```text
deployment.apps "mysql" deleted
persistentvolumeclaim/mysql-pvc created
```

PVC와 PV를 한 번에 봅니다.

```bash
kubectl get pvc,pv
```

```text
NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/mysql-pvc   Bound    pvc-3f1e2d4c-5b6a-4c7d-8e9f-0a1b2c3d4e5f   1Gi        RWO            hostpath       5s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   AGE
persistentvolume/pvc-3f1e2d4c-5b6a-4c7d-8e9f-0a1b2c3d4e5f   1Gi        RWO            Delete           Bound    default/mysql-pvc   hostpath       5s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

우리가 만들지 않은 PV가 `pvc-`로 시작하는 이름으로 생겼고, PVC와 PV 모두 `Bound`입니다. PVC가 오자 StorageClass가 PV를 자동으로 만든 것, 즉 **동적 프로비저닝**입니다.

## STEP 26. PVC를 쓰는 MySQL 올리기

```bash
kubectl apply -f mysql-pvc-deploy.yaml
```

```text
deployment.apps/mysql created
```

`1/1 Running`이 되면 `Ctrl + C`로 멈춥니다.

```bash
kubectl get pods -l app=mysql -w
```

```text
NAME                     READY   STATUS              RESTARTS   AGE
mysql-7b9c6d5f4e-h3j5k   0/1     ContainerCreating   0          2s
mysql-7b9c6d5f4e-h3j5k   1/1     Running             0          20s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

## STEP 27. 데이터 넣기 (실습 5와 같은 SQL)

`Running`이 되고 30초쯤 기다린 뒤 접속합니다.

```bash
kubectl exec -it deploy/mysql -- mysql -uroot -pskilleat
```

`mysql>` 프롬프트에서 실습 5와 똑같이 입력합니다.

```sql
CREATE DATABASE blog;
CREATE TABLE blog.post (id INT, title VARCHAR(50));
INSERT INTO blog.post VALUES (1, 'hello');
exit
```

??? tip "mysql> 프롬프트 없이 한 줄씩 실행하기"
    먼저 Pod 이름을 확인해 복사합니다.

    ```bash
    kubectl get pods -l app=mysql
    ```

    `<Pod 이름>` 자리에 꺾쇠 없이 붙여 넣고 실행합니다.

    ```bash
    kubectl exec -it <Pod 이름> -- mysql -uroot -pskilleat -e "CREATE DATABASE blog;"
    ```

    ```bash
    kubectl exec -it <Pod 이름> -- mysql -uroot -pskilleat -e "CREATE TABLE blog.post (id INT, title VARCHAR(50));"
    ```

    ```bash
    kubectl exec -it <Pod 이름> -- mysql -uroot -pskilleat -e "INSERT INTO blog.post VALUES (1, 'hello');"
    ```

## STEP 28. Pod 지우고 1/1 Running 뒤 다시 확인

실습 5처럼 Pod를 지웁니다.

```bash
kubectl delete pod -l app=mysql
```

새 Pod가 `1/1 Running`이 될 때까지 봅니다. Pod 이름이 바뀐 것도 확인합니다. `Ctrl + C`로 멈춥니다.

```bash
kubectl get pods -l app=mysql -w
```

30초쯤 기다린 뒤 다시 접속합니다.

```bash
kubectl exec -it deploy/mysql -- mysql -uroot -pskilleat
```

```sql
SELECT * FROM blog.post;
```

```text
+------+-------+
| id   | title |
+------+-------+
|    1 | hello |
+------+-------+
1 row in set (0.00 sec)
```

`exit`로 나옵니다.

??? tip "mysql> 프롬프트 없이 확인하기"
    `kubectl get pods -l app=mysql`로 **새** Pod 이름을 복사해 넣습니다.

    ```bash
    kubectl exec -it <Pod 이름> -- mysql -uroot -pskilleat -e "SELECT * FROM blog.post;"
    ```

Pod는 사라졌다 새로 생겼지만 PVC와 PV는 Pod 밖에 그대로 있었고, 새 Pod가 같은 PVC를 다시 붙였습니다. 실습 5와 같은 실험, **결과만 다릅니다.**

## 성공하면

- STEP 24: `hostpath (default)`가 보입니다.
- STEP 25: PVC, PV 모두 STATUS `Bound`입니다.
- STEP 28: 새 Pod에서도 `1`, `hello`가 남아 있습니다.

## 막히면

??? failure "PVC가 Pending"
    PVC의 Events를 봅니다.

    ```bash
    kubectl describe pvc mysql-pvc
    ```

    - `storageclass.storage.k8s.io "hostpath" not found`: `storageClassName` 오타이거나 이름이 다릅니다. STEP 24의 이름으로 고치고, PVC를 지운 뒤(`kubectl delete pvc mysql-pvc`) 다시 만듭니다.
    - `waiting for first consumer to be created before binding`: 바인딩 모드가 `WaitForFirstConsumer`입니다. 정상입니다. STEP 26으로 가면 `Bound`가 됩니다.

??? failure "The Deployment \"mysql\" is invalid: spec.strategy.rollingUpdate: Forbidden: may not be specified when strategy `type` is 'Recreate'"
    실습 5의 `mysql` Deployment를 지우지 않고 `mysql-pvc-deploy.yaml`을 apply했습니다. 지우고 다시 올립니다.

    ```bash
    kubectl delete deployment mysql
    ```

    ```bash
    kubectl apply -f mysql-pvc-deploy.yaml
    ```

??? failure "Pod가 Pending, Events에 unbound immediate PersistentVolumeClaims"
    PVC가 아직 `Bound`가 아닙니다. 위 **PVC가 Pending**을 봅니다.

??? failure "ERROR 2002 (HY000): Can't connect to local MySQL server through socket"
    새 Pod가 아직 기동 중입니다. 30초쯤 뒤 다시 접속합니다.

??? failure "STEP 27에서 ERROR 1007 (HY000): Can't create database 'blog'; database exists"
    이미 `blog`가 있습니다. STEP 27을 두 번 했거나, 앞에서 만든 데이터가 PVC에 남아 있는 것입니다. 그대로 STEP 28로 갑니다.

??? failure "ERROR 1045 (28000): Access denied for user 'root'@'localhost'"
    비밀번호가 다릅니다. `kubectl get secret mysql-secret -o go-template="{{.data.MYSQL_ROOT_PASSWORD | base64decode}}"`로 값을 확인합니다. PVC에는 **처음 초기화할 때의 비밀번호**가 저장되어, 그 뒤 Secret을 바꿔도 root 비밀번호는 바뀌지 않습니다.

## 정리

오늘의 마지막 실습입니다. 아래 순서대로 지웁니다.

```bash
kubectl delete deployment mysql
```

```bash
kubectl delete pvc mysql-pvc
```

```bash
kubectl delete secret mysql-secret
```

```bash
kubectl delete configmap app-config
```

!!! warning "PVC를 지우면 데이터도 지워집니다"
    reclaimPolicy가 `Delete`면(STEP 24의 `RECLAIMPOLICY` 칸) PVC를 지울 때 PV와 그 안의 데이터도 함께 지워집니다. 실제 운영에서는 PVC를 지우기 전에 반드시 백업합니다.

!!! note "환경에 따라 다를 수 있음"
    reclaimPolicy가 `Retain`이면 PVC를 지워도 PV가 `Released` 상태로 남습니다. `kubectl get pv`로 확인하고 남아 있으면 `kubectl delete pv <PV 이름>`으로 지웁니다.

빠뜨린 것이 있으면 [오늘의 실습](index.md)의 **하루를 마치고 한꺼번에 정리** 명령을 실행합니다.
