# 실습 5. 데이터가 사라진다 (emptyDir)

!!! abstract "이번 실습에서 할 것"
    - MySQL을 emptyDir로 띄우고 데이터를 한 줄 넣어 둡니다.
    - Pod를 지우면 서비스는 살아나지만 데이터는 사라지는 것을 직접 겪습니다.

**소요 시간** 약 12분, **교안** DAY 4 섹션 5

## 준비

`day4` 폴더에서 합니다.

**실습 2의 `mysql-secret`이 있어야 합니다.** 비밀번호가 `skilleat`인지 확인합니다.

```bash
kubectl get secret mysql-secret -o go-template="{{.data.MYSQL_ROOT_PASSWORD | base64decode}}"
```

```text
skilleat
```

`NotFound`가 나오면 실습 2의 STEP 10 명령으로 다시 만듭니다.

```bash
kubectl create secret generic mysql-secret --from-literal=MYSQL_ROOT_PASSWORD=skilleat
```

이번 실습에서 쓸 파일은 `mysql-emptydir.yaml`입니다.

```yaml title="mysql-emptydir.yaml"
--8<-- "files/day4/mysql-emptydir.yaml"
```
[mysql-emptydir.yaml 내려받기](../files/day4/mysql-emptydir.yaml){ download }

| 부분 | 뜻 |
|---|---|
| `envFrom` | `mysql-secret`의 모든 키를 이름 그대로 환경변수로 넣습니다. MySQL 이미지는 처음 뜰 때 `MYSQL_ROOT_PASSWORD`로 root 비밀번호를 정합니다. |
| `readinessProbe` | 섹션 3에서 배운 준비 확인입니다. `mysqladmin ping`이 127.0.0.1의 MySQL 서버가 응답하는지 봅니다. 초기화 중에는 `0/1`입니다. 서버가 살아 있는지만 보므로 비밀번호가 틀려 `Access denied`가 나도 성공으로 끝납니다. READY `1/1`이 SQL 로그인 성공을 뜻하지는 않습니다. |
| `/var/lib/mysql` | MySQL이 데이터를 쓰는 폴더입니다. |
| `emptyDir: {}` | Pod가 지워지면 함께 사라지는 폴더입니다. |

!!! note "READY 1/1은 로그인 확인이 아닙니다"
    `mysqladmin ping`은 서버가 실행 중이면 `Access denied`가 나도 종료 코드 0으로 끝납니다. 그래서 READY `1/1`은 서버가 살아 있다는 뜻이지, root 비밀번호로 SQL 로그인이 된다는 뜻은 아닙니다. 자격증명까지 확인하려면 인증해서 `SELECT 1`을 실행하는 더 엄격한 검사를 쓸 수 있습니다. 이 실습은 `ping`으로 둡니다.
    근거: [MySQL 8.0 mysqladmin](https://dev.mysql.com/doc/refman/8.0/en/mysqladmin.html)

## STEP 26. MySQL 올리고 READY 1/1 기다리기

```bash
kubectl apply -f mysql-emptydir.yaml
```

```text
deployment.apps/mysql created
```

READY가 `1/1`이 될 때까지 지켜봅니다. 이미지가 크고 첫 기동 때 초기화를 해서 1분쯤 걸릴 수 있습니다. `Running`이 먼저 보여도 READY가 `0/1`이면 아직 초기화 중입니다. `1/1`이 보이면 `Ctrl + C`로 멈춥니다.

```bash
kubectl get pods -l app=mysql -w
# READY 확인 후 Ctrl+C
```

```text
NAME                     READY   STATUS              RESTARTS   AGE
mysql-5c7d8f9b6d-x2k4p   0/1     ContainerCreating   0          3s
mysql-5c7d8f9b6d-x2k4p   0/1     Running             0          20s
mysql-5c7d8f9b6d-x2k4p   1/1     Running             0          45s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

로그로도 확인합니다. 마지막 줄 근처에 `ready for connections`와 `port: 3306`이 보이면 접속을 받을 준비가 된 것입니다.

```bash
kubectl logs deploy/mysql --tail 3
```

```text
... [System] [MY-011323] [Server] X Plugin ready for connections. Bind-address: '::' port: 33060, socket: /var/run/mysqld/mysqlx.sock
... [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.xx'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server - GPL.
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

!!! note "Running과 준비 완료는 다릅니다"
    readinessProbe가 없으면 MySQL이 초기화 중이어도 `1/1 Running`으로 보입니다. 초기화 중에는 `port: 0`인 임시 서버가 잠깐 뜨기도 하니, 로그로 볼 때는 `port: 3306` 줄을 찾습니다.

**Pod 이름을 기억해 둡니다.** STEP 28에서 새 Pod와 비교합니다.

## STEP 27. 접속해서 데이터 넣기 (비밀번호 skilleat)

READY가 `1/1`이 된 뒤 MySQL에 접속합니다. `deploy/mysql`이라고 쓰면 Pod 이름을 몰라도 됩니다.

```bash
kubectl exec -it deploy/mysql -- mysql -uroot -pskilleat
```

```text
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
...
mysql>
```

첫 줄의 Warning은 비밀번호를 명령에 썼다는 경고입니다. 실습이라 무시합니다.

`mysql>` 프롬프트가 나오면 한 줄씩 입력합니다. 데이터베이스 `blog`, 테이블 `post`를 만들고 글 한 줄을 넣습니다.

```sql
CREATE DATABASE blog;
CREATE TABLE blog.post (id INT, title VARCHAR(50));
INSERT INTO blog.post VALUES (1, 'hello');
SELECT * FROM blog.post;
exit
```

```text
+------+-------+
| id   | title |
+------+-------+
|    1 | hello |
+------+-------+
1 row in set (0.00 sec)
```

??? tip "mysql> 프롬프트 없이 한 줄씩 실행하기"
    `mysql>` 안에서 입력하기 불편하면 `-e`로 SQL을 한 줄씩 넘겨도 됩니다. 두 OS에서 똑같이 동작합니다 (바깥은 큰따옴표, 안의 글자는 작은따옴표).

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

    ```bash
    kubectl exec -it <Pod 이름> -- mysql -uroot -pskilleat -e "SELECT * FROM blog.post;"
    ```

## STEP 28. Pod 지우기 (Deployment가 새로 만든다)

장애를 흉내 내어 MySQL Pod를 지웁니다.

```bash
kubectl delete pod -l app=mysql
```

```text
pod "mysql-5c7d8f9b6d-x2k4p" deleted
```

Deployment가 곧바로 새 Pod를 만듭니다. READY가 `1/1`이 되면 `Ctrl + C`로 멈춥니다.

```bash
kubectl get pods -l app=mysql -w
# READY 확인 후 Ctrl+C
```

```text
NAME                     READY   STATUS              RESTARTS   AGE
mysql-5c7d8f9b6d-q8w7e   0/1     ContainerCreating   0          2s
mysql-5c7d8f9b6d-q8w7e   0/1     Running             0          5s
mysql-5c7d8f9b6d-q8w7e   1/1     Running             0          35s
```
<small>출력은 예시입니다. 이름, IP, 시간은 사람마다 다릅니다.</small>

STEP 26과 Pod 이름이 다릅니다. **살아난 Pod는 같은 Pod가 아니라 새 Pod**입니다.

## STEP 29. 다시 접속해서 찾아보기

새 Pod의 READY가 `1/1`이 된 뒤 접속합니다.

```bash
kubectl exec -it deploy/mysql -- mysql -uroot -pskilleat
```

```sql
SHOW DATABASES;
USE blog;
```

```text
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

ERROR 1049 (42000): Unknown database 'blog'
```

`blog`가 목록에 없고, `USE blog`는 `Unknown database` 오류가 납니다. `exit`로 나옵니다.

??? tip "mysql> 프롬프트 없이 확인하기"
    `kubectl get pods -l app=mysql`로 **새** Pod 이름을 복사해 넣습니다.

    ```bash
    kubectl exec -it <Pod 이름> -- mysql -uroot -pskilleat -e "SHOW DATABASES;"
    ```

옛 Pod의 emptyDir이 Pod와 함께 지워졌고, 새 Pod는 빈 폴더에서 MySQL을 처음부터 다시 초기화했습니다. 비밀번호는 Secret에서 다시 읽으니 접속은 됩니다. **서비스는 살아났는데 데이터는 사라진** 장애입니다.

## 성공하면

- STEP 27: `SELECT`에 `1`, `hello`가 나옵니다.
- STEP 28: 새 Pod 이름이 STEP 26과 다릅니다.
- STEP 29: `SHOW DATABASES`에 `blog`가 없습니다.

## 막히면

??? failure "ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock'"
    MySQL이 아직 초기화 중입니다. READY가 `0/1`일 때 접속하면 납니다. `kubectl get pods -l app=mysql`로 `1/1`을 확인하고 다시 접속합니다.

??? failure "ERROR 1045 (28000): Access denied for user 'root'@'localhost'"
    비밀번호가 다릅니다. Secret 값을 확인합니다.

    ```bash
    kubectl get secret mysql-secret -o go-template="{{.data.MYSQL_ROOT_PASSWORD | base64decode}}"
    ```

    `pass1234`가 나오면 실습 2의 STEP 10을 하지 않은 것입니다. 이번 실습은 `-ppass1234`로 접속하거나, Secret을 `skilleat`으로 다시 만든 뒤 `kubectl delete pod -l app=mysql`로 Pod를 새로 띄웁니다 (emptyDir이라 새로 초기화되며 새 비밀번호가 적용됩니다).

??? failure "Pod가 CreateContainerConfigError"
    `mysql-secret`이 없습니다. 준비의 `kubectl create secret generic ...` 명령으로 만들면 잠시 뒤 스스로 풀립니다.

??? failure "Pod가 오래 ContainerCreating"
    `mysql:8.0` 이미지를 받는 중입니다. 수백 MB라 몇 분 걸릴 수 있습니다. `kubectl describe pod -l app=mysql`의 Events에서 `Pulling image`를 확인하고 기다립니다.

??? failure "error: unable to upgrade connection: container not found / pod is not running"
    STEP 28 직후 지워지는 Pod에 붙었습니다. 새 Pod의 READY가 `1/1`이 된 것을 보고 다시 접속합니다.

??? failure "'<' 연산자는 나중에 사용하도록 예약되어 있습니다 (PowerShell) / zsh: no such file or directory: Pod (Mac)"
    꺾쇠 `< >`까지 입력했습니다. 실제 Pod 이름만 넣습니다.

## 정리

**지우지 마세요.** 실습 6의 STEP 31에서 이 `mysql` Deployment를 지우고 PVC 버전으로 바꿉니다. `mysql-secret`도 실습 6에서 그대로 씁니다.
