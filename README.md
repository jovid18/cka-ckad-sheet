# cka-ckad-sheet

cka/ckad 기출 풀면서 공부한 내용 정리

---

# Lightning Lab

## Case 1) k8s 버전 업그레이드

### 상황 파악

- 노드 구성: `control plane`, `node01`
- 먼저 실행 중인 팟이 있다면 옮겨줘야 함 (node drain)
- 팟 위치 확인:
  ```bash
  k get pods -o wide
  ```
- 확인 결과 팟이 `node01`에 있음 → control plane은 먼저 업그레이드해도 됨

### 업그레이드 순서

1. control plane upgrade
2. node01 drain
3. node01 upgrade

---

### 1. Control Plane 업그레이드

#### 1-1. kubeadm 버전 올리기

처음에는 그냥 plan을 돌려봤는데 1.34.0 → 1.34.8까지밖에 안 됨.
kubeadm 자체가 1.35가 아니기 때문 → apt source부터 수정해야 함.

```bash
# 현재 plan 확인 (1.34.x까지밖에 안 나옴)
sudo kubeadm upgrade plan

# apt source 버전을 1.35로 수정
sudo vim /etc/apt/sources.list.d/kubernetes.list

# 패키지 목록 갱신 후 설치 가능한 버전 확인 (madison)
sudo apt update
sudo apt-cache madison kubeadm

# kubeadm 설치 (hold 해제 → 설치 → 다시 hold)
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.35.0-1.1
sudo apt-mark hold kubeadm

# 버전 확인 (1.35.0)
kubeadm version
```

#### 1-2. control plane upgrade 적용 + kubelet/kubectl 교체

```bash
# 업그레이드 적용 (시간 좀 걸림)
sudo kubeadm upgrade apply v1.35.0

# control plane drain
kubectl drain controlplane --ignore-daemonsets

# kubelet, kubectl 설치
sudo apt-get install -y kubelet=1.35.0-1.1 kubectl=1.35.0-1.1

# 데몬 리로드 & kubelet 재시작
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

---

### 2. node01 업그레이드

```bash
# control plane에서 node01 drain
k drain node01 --ignore-daemonsets

# node01로 접속
ssh node01
```

node01 내부에서:

```bash
# apt source 수정 후 설치 가능 버전 확인
vim /etc/apt/sources.list.d/kubernetes.list
sudo apt-cache madison kubeadm

# kubeadm 설치 후 노드 업그레이드
sudo apt-get install -y kubeadm=1.35.0-1.1
sudo kubeadm upgrade node

# kubelet, kubectl 교체 (hold 해제 필요)
sudo apt-mark unhold kubelet
sudo apt-get install -y kubelet=1.35.0-1.1 kubectl=1.35.0-1.1
```

control plane으로 돌아와서:

```bash
k uncordon node01
```

---

### 3. 버전 확인

```bash
k get nodes
```

`VERSION` 컬럼이 모두 `v1.35.0`으로 나오면 성공.

---

## Case 2) Deployment 커스텀 컬럼으로 출력

`custom-columns`를 이용해 원하는 필드만 뽑아 정렬해서 파일로 저장하는 방법 (암기 필수).

```bash
k -n admin2406 get deployments.apps \
  --sort-by='.metadata.name' \
  -o custom-columns=\
DEPLOYMENT:.metadata.name,\
CONTAINER_IMAGE:.spec.template.spec.containers[].image,\
READY_REPLICAS:.status.readyReplicas,\
NAMESPACE:.metadata.namespace \
  > /opt/admin2406_data
```

포인트:

- `--sort-by`는 JSONPath 형식
- `-o custom-columns=헤더:JSONPath,...` 콤마로 컬럼 구분
- 컨테이너가 여러 개일 수 있으니 `containers[].image` 처럼 인덱스 또는 `[*]` 사용

---

## Case 3) kubeconfig에서 이상한 점 찾기

문제: `/root/CKA/admin.kubeconfig` 파일이 주어지고 동작하지 않음 → 원인 찾아 고치기.

### 1. kubeconfig 확인

```bash
cat /root/CKA/admin.kubeconfig
```

`server: https://controlplane:4380` 으로 되어 있음 → 포트가 의심스러움.

### 2. 실제 apiserver 포트 확인

```bash
k describe pod kube-apiserver-controlplane -n kube-system | grep -i port
# Port:      6443/TCP (probe-port)
# Host Port: 6443/TCP (probe-port)
```

실제로는 `6443` 포트. kubeconfig가 `4380`을 가리키고 있어 잘못됨.

### 3. 수정

```bash
vim /root/CKA/admin.kubeconfig
# server: https://controlplane:6443 으로 수정
```

---

## Case 4) Deployment 생성 + 롤링 업데이트 + annotation

> Create a new deployment called `nginx-deploy`, with image `nginx:1.16` and 1 replica.
> Next, upgrade the deployment to version `1.17` using rolling update and add the annotation message `Updated nginx image to 1.17.`

### 1. dry-run으로 YAML 생성

```bash
# dry-run 옵션을 변수로 (자주 쓰니 export 해두면 편함)
export do='--dry-run=client -o yaml'

# manifest 생성
kubectl create deployment nginx-deploy --image=nginx:1.16 --replicas=1 $do > ndp.yaml
```

### 2. 적용 후 이미지 업그레이드 + annotation

```bash
k apply -f ndp.yaml

# 이미지 버전 변경 + annotation 추가는 edit으로
k edit deployments.apps nginx-deploy
```

`k edit`에서:

- `spec.template.spec.containers[].image`: `nginx:1.16` → `nginx:1.17`
- `spec.template.metadata.annotations`에 `message: "Updated nginx image to 1.17."` 추가

> 참고: 단순 이미지 변경만이면 `k set image deployment/nginx-deploy nginx=nginx:1.17`,
> annotation은 `k annotate deployment nginx-deploy message="Updated nginx image to 1.17."` 도 가능.

---

## Case 5) alpha-mysql Deployment 트러블슈팅 (PV/PVC 바인딩)

> A new deployment called `alpha-mysql` has been deployed in the `alpha` namespace. However, the pods are not running. Troubleshoot and fix the issue. The deployment should make use of the persistent volume `alpha-pv` to be mounted at `/var/lib/mysql` and should use the environment variable `MYSQL_ALLOW_EMPTY_PASSWORD=1` to make use of an empty root password.
>
> **Important: Do not alter the persistent volume.**

핵심 제약: **PV는 절대 건드리면 안 됨** → 결국 PVC를 PV에 맞춰서 수정해야 한다는 뜻.

### 1. 상황 파악 (PV / PVC / StorageClass 한 번에 보기)

```bash
k -n alpha get pv
k -n alpha get pvc
k -n alpha get storageclasses.storage.k8s.io
```

확인된 상태:

```
# PV: 이미 Available, 1Gi, RWO, storageClass=slow
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      STORAGECLASS
alpha-pv   1Gi        RWO            Retain           Available   slow

# PVC: Pending, storageClass=slow-storage (PV의 slow와 다름!)
NAME          STATUS    STORAGECLASS
alpha-claim   Pending   slow-storage

# StorageClass: slow 하나만 존재
NAME   PROVISIONER                    VOLUMEBINDINGMODE
slow   kubernetes.io/no-provisioner   WaitForFirstConsumer
```

여기서 보이는 문제는 두 가지:

1. **PVC 이름이 `alpha-claim`** → Deployment가 참조하는 이름과 다를 가능성이 큼
2. **PVC의 storageClass가 `slow-storage`** → PV의 `slow`와 매칭 안 됨 (binding 불가)

> 💡 PVC가 PV에 바인딩되려면 `storageClassName`, `accessModes`, `storage` 용량이 PV 조건을 만족해야 함.
> `kubernetes.io/no-provisioner` + `WaitForFirstConsumer`는 동적 프로비저닝 X → 정적으로 만들어둔 PV에 매칭되어야 한다는 의미.

### 2. Deployment가 실제로 어떤 PVC 이름을 원하는지 확인

```bash
k describe pod -n alpha <alpha-mysql-xxxx>
```

핵심 부분:

```
Volumes:
  mysql-data:
    ClaimName:  mysql-alpha-pvc   # ← Deployment가 요구하는 PVC 이름
Events:
  Warning  FailedScheduling  persistentvolumeclaim "mysql-alpha-pvc" not found
```

→ Deployment는 `mysql-alpha-pvc`라는 이름의 PVC를 찾고 있는데, 실제로는 `alpha-claim`이 만들어져 있음.
**기존 PVC를 지우고, 이름·스펙을 맞춘 새 PVC로 다시 만들어야 함.**

### 3. PVC 수정 전략

PV는 건드릴 수 없으니 PVC를 PV 스펙에 맞춰 작성:

| 항목               | PV (`alpha-pv`) | PVC가 가져야 할 값                             |
| ------------------ | --------------- | ---------------------------------------------- |
| `name`             | -               | `mysql-alpha-pvc` (Deployment가 참조하는 이름) |
| `storageClassName` | `slow`          | `slow`                                         |
| `accessModes`      | `ReadWriteOnce` | `ReadWriteOnce`                                |
| `storage` 요청     | `1Gi` 제공      | `1Gi` 이하 요청 (1Gi로 맞춤)                   |

> ⚠️ 처음에는 `ReadWriteMany` + `2Gi`로 잘못 작성했더니 PV 조건을 못 맞춰서 계속 Pending이었음. → PV 스펙 그대로 따라가는 게 안전.

### 4. 기존 PVC 삭제 후 새로 작성

```bash
# 기존 잘못된 PVC 제거
k -n alpha delete pvc alpha-claim

# 새 PVC manifest 작성
vim pvc.yaml
```

`pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-alpha-pvc # Deployment가 참조하는 이름과 동일
  namespace: alpha
spec:
  accessModes:
    - ReadWriteOnce # PV와 동일
  resources:
    requests:
      storage: 1Gi # PV 용량 이하
  storageClassName: slow # PV와 동일 (slow-storage 아님!)
  volumeMode: Filesystem
```

```bash
k apply -f pvc.yaml
```

### 5. 결과 확인

```bash
k -n alpha get pvc
# mysql-alpha-pvc   Bound   alpha-pv   1Gi   RWO   slow

k -n alpha get pods
# alpha-mysql-c558fb4bf-d7xjf   1/1   Running   0   10m
```

Pod이 정상 `Running` 상태가 되면 완료.

### 정리: PVC가 Pending일 때 체크리스트

1. `kubectl describe pvc <name>` → 이벤트에서 원인 확인
2. PV가 `Available` 상태인지 (`k get pv`)
3. PVC ↔ PV의 다음 4가지가 일치하는가:
   - `storageClassName`
   - `accessModes`
   - 요청 storage ≤ PV capacity
   - `volumeMode` (대부분 Filesystem이라 생략 가능하지만 명시되어 있으면 맞춰주기)
4. Deployment가 참조하는 `claimName`과 PVC 이름이 같은가
5. `WaitForFirstConsumer`인 경우, 실제로 Pod이 스케줄링 시도를 해야 바인딩이 시작됨

---

## Case 6) ETCD 스냅샷 백업

> Take the backup of ETCD at the location `/opt/etcd-backup.db` on the controlplane node.

ETCD는 클러스터의 모든 상태(파드, 서비스, 시크릿 등)를 저장하는 key-value 저장소.
백업은 `etcdctl snapshot save`로 떠두며, **TLS 인증서 경로**와 **endpoint**를 정확히 지정하는 게 핵심.

### 1. ETCD 인증서 경로 확인 (manifest에서 그대로 따옴)

ETCD는 kubeadm 클러스터에서 static pod로 떠 있음 → manifest에서 인증서/엔드포인트 경로를 그대로 가져오면 됨.

```bash
cat /etc/kubernetes/manifests/etcd.yaml
```

필요한 값만 추리면:

```yaml
- --cert-file=/etc/kubernetes/pki/etcd/server.crt # → --cert
- --key-file=/etc/kubernetes/pki/etcd/server.key # → --key
- --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt # → --cacert
- --listen-client-urls=https://127.0.0.1:2379,... # → --endpoints
```

> 💡 manifest의 플래그 이름과 `etcdctl`의 플래그 이름이 살짝 다르니 주의:
>
> - `--cert-file` → `--cert`
> - `--key-file` → `--key`
> - `--trusted-ca-file` → `--cacert`

### 2. `etcdctl snapshot save` 사용법 확인

```bash
ETCDCTL_API=3 etcdctl snapshot save --help
```

필수 옵션 정리:

| 옵션          | 설명                                     |
| ------------- | ---------------------------------------- |
| `<filename>`  | 저장할 스냅샷 파일 경로 (위치 인자)      |
| `--endpoints` | ETCD 서버 주소 (기본값 `127.0.0.1:2379`) |
| `--cacert`    | CA 인증서 (서버 검증용)                  |
| `--cert`      | 클라이언트 인증서                        |
| `--key`       | 클라이언트 키                            |

> 💡 `ETCDCTL_API=3`은 etcdctl v2 API와 구분하기 위한 환경변수. snapshot 명령은 v3 API에서만 동작하므로 항상 붙여줘야 안전.

### 3. 실제 백업 실행

```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

성공하면 `Snapshot saved at /opt/etcd-backup.db` 메시지가 출력됨.

### 4. 백업 검증 (선택)

```bash
ETCDCTL_API=3 etcdctl --write-out=table snapshot status /opt/etcd-backup.db
```

`HASH`, `REVISION`, `TOTAL KEYS`, `TOTAL SIZE`가 표 형태로 나오면 백업 파일이 정상.

### 정리: ETCD 백업 시 기억할 것

1. **인증서 3종 + endpoint**는 항상 `/etc/kubernetes/manifests/etcd.yaml`에서 그대로 복사
2. `etcdctl` 플래그 이름은 manifest와 다름 (`--cert-file` → `--cert` 등)
3. `ETCDCTL_API=3` 환경변수 꼭 붙이기
4. 파일 경로는 위치 인자 (`save <filename>`) — 옵션 아님
5. 복원은 `etcdctl snapshot restore <file>` + static pod manifest의 `--data-dir` 갱신이 필요 (복원 문제가 따로 나오면 별도 케이스로)

## Case 7) Secret을 volume으로 마운트한 Pod 만들기

> Create a pod called `secret-1401` in the `admin1401` namespace using the `busybox` image. The container within the pod should be called `secret-admin` and should sleep for 4800 seconds.
>
> The container should mount a read-only secret volume called `secret-volume` at the path `/etc/secret-volume`. The secret being mounted has already been created for you and is called `dotfile-secret`.

요구사항을 분해하면:

| 항목          | 값                                |
| ------------- | --------------------------------- |
| Pod 이름      | `secret-1401`                     |
| Namespace     | `admin1401`                       |
| 이미지        | `busybox`                         |
| 컨테이너 이름 | `secret-admin` (Pod 이름과 다름!) |
| 컨테이너 동작 | `sleep 4800`                      |
| Volume 이름   | `secret-volume`                   |
| Volume 종류   | secret (`dotfile-secret`)         |
| Mount 경로    | `/etc/secret-volume`              |
| Mount 옵션    | read-only                         |

> ⚠️ Pod 이름(`secret-1401`)과 컨테이너 이름(`secret-admin`)이 다르다는 점을 놓치기 쉬움. `kubectl run`은 기본적으로 둘을 같게 만들기 때문에 manifest에서 컨테이너 이름을 따로 수정해야 함.

### 1. dry-run으로 베이스 manifest 생성

```bash
kubectl run secret-1401 \
  --namespace=admin1401 \
  --image=busybox \
  --dry-run=client -o yaml \
  --command -- sleep 4800 \
  > pod.yaml
```

> 💡 `--command` 다음의 `--`는 `kubectl`의 옵션 파싱을 멈추는 구분자. 이후 인자(`sleep 4800`)는 컨테이너의 `command`로 들어감.

생성된 초안:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: secret-1401
  name: secret-1401
  namespace: admin1401
spec:
  containers:
    - command:
        - sleep
        - '4800'
      image: busybox
      name: secret-1401 # ← 이걸 secret-admin으로 바꿔야 함
      resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

### 2. 컨테이너 이름 + volume / volumeMount 추가

수정한 최종 manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: secret-1401
  name: secret-1401
  namespace: admin1401
spec:
  containers:
    - command:
        - sleep
        - '4800'
      image: busybox
      name: secret-admin # 컨테이너 이름 변경
      resources: {}
      volumeMounts:
        - name: secret-volume # 아래 volumes[].name과 일치해야 함
          mountPath: /etc/secret-volume
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: dotfile-secret # 미리 만들어져 있는 secret 이름
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

포인트:

- `volumes[]`는 `spec` 하위, `volumeMounts[]`는 `containers[]` 하위 (두 곳을 헷갈리지 말 것)
- `volumeMounts[].name`과 `volumes[].name`이 **문자열로 정확히 일치**해야 마운트가 연결됨
- secret을 volume으로 쓸 때는 `volumes[].secret.secretName`에 secret 객체 이름을 넣음 (volume 이름과 다를 수 있음)
- read-only는 `volumeMounts[].readOnly: true`로 지정 (secret volume은 기본적으로 read-only지만 명시 요구가 있으면 적어주는 게 안전)

### 3. 적용 및 확인

```bash
k apply -f pod.yaml
# pod/secret-1401 created

k -n admin1401 get pod
# NAME          READY   STATUS    RESTARTS   AGE
# secret-1401   1/1     Running   0          6s
```

마운트가 실제로 됐는지 확인하고 싶으면:

```bash
k -n admin1401 exec secret-1401 -- ls /etc/secret-volume
# dotfile-secret 안의 key들이 파일로 보이면 OK
```

### 정리: secret을 volume으로 마운트할 때 체크리스트

1. `kubectl run ... --dry-run=client -o yaml`로 베이스 만들고, 요구사항대로 손보는 흐름이 가장 빠름
2. Pod 이름 ≠ 컨테이너 이름인 경우가 자주 나옴 → `containers[].name` 반드시 다시 확인
3. volume 정의(`spec.volumes`)와 mount(`spec.containers[].volumeMounts`)는 **이름으로 연결**됨
4. secret을 volume으로 쓰면 각 key가 mount 경로 아래 **파일로** 노출됨 (env로 주입하는 방식과 다름)
5. `--command -- sleep 4800` 처럼 `--` 구분자를 빼먹으면 `kubectl`이 옵션으로 해석하려다 에러

# Mock Test 1

## Case 1) 3개 컨테이너 + emptyDir 공유 볼륨 Pod

> Create a Pod `mc-pod` in the `mc-namespace` namespace with three containers. The first container should be named `mc-pod-1`, run the `nginx:1-alpine` image, and set an environment variable `NODE_NAME` to the node name. The second container should be named `mc-pod-2`, run the `busybox:1` image, and continuously log the output of the `date` command to the file `/var/log/shared/date.log` every second. The third container should have the name `mc-pod-3`, run the image `busybox:1`, and print the contents of the `date.log` file generated by the second container to stdout. Use a shared, non-persistent volume.

요구사항을 분해하면:

| 항목        | 값                                                        |
| ----------- | --------------------------------------------------------- |
| Pod 이름    | `mc-pod`                                                  |
| Namespace   | `mc-namespace`                                            |
| 컨테이너 1  | `mc-pod-1` / `nginx:1-alpine` / env `NODE_NAME`=노드명    |
| 컨테이너 2  | `mc-pod-2` / `busybox:1` / 1초마다 `date` → 공유 파일     |
| 컨테이너 3  | `mc-pod-3` / `busybox:1` / 공유 파일 stdout으로 출력      |
| 공유 볼륨   | non-persistent → `emptyDir`                               |
| 마운트 경로 | `/var/log/shared` (이름은 자유, `date.log`가 들어가는 곳) |

### 1. dry-run으로 베이스 manifest 생성

컨테이너가 여러 개라도 `kubectl run`은 한 개 컨테이너짜리 베이스를 만들어주는 용도로만 씀. 나머지는 manifest에서 직접 추가.

```bash
kubectl run mc-pod \
  --image=nginx:1-alpine \
  --namespace=mc-namespace \
  --dry-run=client -o yaml \
  > mc-pod.yaml
```

> ⚠️ 처음에 `--env="NODE_NAME=controlplane"`처럼 노드명을 하드코딩하려고 했지만, 문제는 "노드명을 환경변수로 주입"이라는 의미라 **downward API(`fieldRef: spec.nodeName`)** 를 써야 함. `--env`로는 fieldRef를 못 만드니 manifest 직접 수정이 필수.

### 2. 최종 manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: mc-pod
  name: mc-pod
  namespace: mc-namespace
spec:
  containers:
    - image: nginx:1-alpine
      name: mc-pod-1
      env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName # 노드명을 동적으로 주입
    - image: busybox:1
      name: mc-pod-2
      command: ['/bin/sh', '-c']
      args: ['while true; do date >> /var/log/shared/date.log; sleep 1; done']
      volumeMounts:
        - name: shared-log
          mountPath: /var/log/shared
    - image: busybox:1
      name: mc-pod-3
      command: ['/bin/sh', '-c']
      args: ['tail -f /var/log/shared/date.log']
      volumeMounts:
        - name: shared-log
          mountPath: /var/log/shared
  volumes:
    - name: shared-log
      emptyDir: {} # non-persistent 공유 볼륨
  restartPolicy: Always
```

포인트:

- `emptyDir: {}` 는 Pod 라이프사이클 동안만 유지되는 임시 볼륨 → "non-persistent" 요구사항에 정확히 맞음
- `volumeMounts[].name` ↔ `volumes[].name` 이 **두 컨테이너 모두에서** 같아야 같은 볼륨을 공유
- writer/reader 패턴: 2번 컨테이너는 `>>` 로 append, 3번은 `tail -f` 로 follow → 같은 파일을 동시에 쓰고 읽음
- `command` 와 `args` 를 분리할 때, 셸 파이프/리다이렉트가 들어가면 반드시 `["/bin/sh", "-c"]` 로 셸을 거쳐야 동작

### 3. 적용 및 확인

```bash
k apply -f mc-pod.yaml

k -n mc-namespace get pod mc-pod
# READY 3/3, STATUS Running 이면 정상

# 3번 컨테이너 로그로 date.log 내용 흘러나오는지 확인
k -n mc-namespace logs mc-pod -c mc-pod-3 -f

# 1번 컨테이너에 NODE_NAME 제대로 들어갔는지 확인
k -n mc-namespace exec mc-pod -c mc-pod-1 -- printenv NODE_NAME
```

### 정리: 멀티컨테이너 Pod + 공유 볼륨 체크리스트

1. `kubectl run` 으로 베이스만 뽑고 추가 컨테이너/볼륨은 **manifest 직접 수정**
2. "노드명/Pod명/네임스페이스" 같은 동적 값은 `--env` 가 아니라 **downward API (`fieldRef`)**
   - `spec.nodeName`, `metadata.name`, `metadata.namespace`, `status.podIP` 등
3. 컨테이너끼리 파일 공유는 `emptyDir` + 양쪽에 동일한 `volumeMounts.name`
4. 셸 리다이렉트(`>>`, `|`, `&&`)는 `["/bin/sh", "-c"]` 없이는 안 먹음
5. 검증은 `logs -c <container>` 와 `exec -c <container>` 로 컨테이너별로 따로

---

## Case 2) node01에 cri-docker 설치 + 서비스 등록

> This question needs to be solved on node `node01`. To access the node using SSH, use the credentials below:
>
> - username: `bob`
> - password: `caleston123`
>
> As an administrator, you need to prepare `node01` to install kubernetes. One of the steps is installing a container runtime. Install the `cri-docker_0.3.16.3-0.debian.deb` package located in `/root` and ensure that the `cri-docker` service is running and enabled to start on boot.

### 1. node01로 SSH 접속

control plane 셸의 사용자명과 노드의 사용자명이 다르므로 `user@host` 형식으로 명시해야 함.

```bash
# ❌ 그냥 ssh node01 → 현재 사용자(root)로 접속 시도 → 비번 안 받음
ssh node01

# ✅ bob 계정으로 명시
ssh bob@node01
# 처음 접속이라 host key 확인 → yes
# 비밀번호: caleston123
```

> ⚠️ host key 확인 프롬프트에서 `no` 누르면 `Host key verification failed.` 로 끊김. 처음 접속하는 노드는 무조건 `yes`.

### 2. root로 권한 상승

`dpkg -i`, `systemctl` 모두 root 권한 필요.

```bash
sudo -i
# 이후 프롬프트가 root@node01 으로 바뀜
```

### 3. .deb 패키지 설치

```bash
ls /root
# cri-docker_0.3.16.3-0.debian.deb

dpkg -i /root/cri-docker_0.3.16.3-0.debian.deb
```

설치 로그 마지막 부분에 아래가 뜨면 **systemd unit은 등록됐지만 서비스가 자동 기동되지 않은 상태**:

```
/usr/sbin/policy-rc.d returned 101, not running 'start cri-docker.service cri-docker.socket'
```

### 4. 서비스 start + enable

```bash
systemctl start cri-docker
systemctl enable cri-docker
systemctl status cri-docker
# Active: active (running) 확인
# Loaded: ... ; enabled;  ← enabled 인지도 같이 확인
```

> 💡 `systemctl enable --now cri-docker` 한 줄로 start + enable 동시 처리 가능. 시험에선 타이핑 줄이는 게 이득.

### 정리: 패키지로 서비스 설치할 때 체크리스트

1. SSH는 사용자명 다르면 `user@host` 로
2. 호스트 키 처음 보면 `yes` (안 그러면 연결 자체가 안 됨)
3. root 권한 필요 → `sudo -i`
4. `.deb` 는 `dpkg -i <file>`
5. `policy-rc.d returned 101` 메시지 = **수동으로 start/enable 해줘야 함**
6. `systemctl enable --now <svc>` 로 start + 부팅시 자동시작 한 번에
7. `systemctl status <svc>` 로 `active (running)` + `enabled` 둘 다 확인

---

## Case 3) VPA 관련 CRD 이름만 파일로 저장

> On controlplane node, identify all CRDs related to VerticalPodAutoscaler and save their names into the file `/root/vpa-crds.txt`.

CRD 목록에서 이름만 필터링해서 파일로 떨구는 단순 작업. 핵심은 `grep` + `awk` + 리다이렉트 조합.

### 1. 전체 CRD 훑어보기

```bash
k get crds
# NAME                                                  CREATED AT
# ...
# verticalpodautoscalercheckpoints.autoscaling.k8s.io   2026-05-15T18:27:03Z
# verticalpodautoscalers.autoscaling.k8s.io             2026-05-15T18:27:03Z
```

`verticalpodautoscaler`가 들어간 CRD 두 개를 확인:

- `verticalpodautoscalercheckpoints.autoscaling.k8s.io`
- `verticalpodautoscalers.autoscaling.k8s.io`

### 2. 이름만 뽑아서 파일로 저장

```bash
k get crds | grep -i verticalpodautoscaler | awk '{print $1}' > /root/vpa-crds.txt
```

- `grep -i` 로 대소문자 무시 매칭
- `awk '{print $1}'` 로 첫 컬럼(`NAME`)만 추출 → `CREATED AT` 컬럼은 버림
- `>` 로 파일 덮어쓰기 (append 필요 없으면 `>` 가 안전)

### 3. 결과 확인

```bash
cat /root/vpa-crds.txt
# verticalpodautoscalercheckpoints.autoscaling.k8s.io
# verticalpodautoscalers.autoscaling.k8s.io
```

> 💡 `k get crds -o name` 을 쓰면 `customresourcedefinition.apiextensions.k8s.io/<name>` 형식으로 prefix가 붙어서 나옴 → 이름만 필요할 땐 헤더 있는 기본 출력에 `awk '{print $1}'` 가 더 깔끔.

---

## Case 4) messaging Pod을 ClusterIP 서비스로 노출 (imperative)

> Create a service named `messaging-service` to expose the `messaging` pod within the cluster on port `6379`. The messaging pod is running in the `default` namespace.
>
> Use imperative commands.

요구사항을 분해하면:

| 항목        | 값                                        |
| ----------- | ----------------------------------------- |
| 서비스 이름 | `messaging-service`                       |
| 노출 대상   | `messaging` Pod (default ns)              |
| 포트        | `6379` (cluster 내부 노출)                |
| 노출 범위   | "within the cluster" → ClusterIP (기본값) |
| 방식        | imperative command                        |

### 1. 대상 Pod 확인

서비스가 매칭할 selector는 `kubectl expose` 가 Pod의 label에서 자동으로 가져가므로, 먼저 Pod이 존재하는지만 확인.

```bash
k get pod messaging
# NAME        READY   STATUS    RESTARTS   AGE
# messaging   1/1     Running   0          1m
```

### 2. `kubectl expose --help` 의 예시를 그대로 활용

`kubectl expose --help` 에 나오는 예시:

```bash
# Create a second service based on the above service, exposing the container port 8443
# as port 443 with the name "nginx-https"
kubectl expose service nginx --port=443 --target-port=8443 --name=nginx-https
```

→ 동일한 패턴으로 `service` 대신 `pod`, 포트를 6379로 바꾸면 끝.

> 💡 시험장에서 옵션이 기억 안 나면 `--help` 의 Examples 섹션을 그대로 복사해서 값만 갈아끼우는 게 가장 빠름.

### 3. 실행

```bash
kubectl expose pod messaging \
  --port=6379 \
  --target-port=6379 \
  --name=messaging-service
# service/messaging-service exposed
```

- `--port`: 서비스가 외부로 노출하는 포트
- `--target-port`: Pod 안 컨테이너가 듣는 포트 (생략하면 `--port` 와 동일하게 설정됨 → 이 문제처럼 둘이 같으면 생략 가능)
- `--type` 미지정 → 기본 `ClusterIP` (문제 요구인 "within the cluster" 에 맞음)

### 4. 확인

```bash
k get svc messaging-service
# NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
# messaging-service   ClusterIP   172.20.62.51   <none>        6379/TCP   3m12s

k describe svc messaging-service | grep -E 'Selector|Endpoints'
# Selector:    tier=msg
# Endpoints:   172.17.0.8:6379
```

두 줄에서 읽어낼 수 있는 것:

- `TYPE=ClusterIP`, `EXTERNAL-IP=<none>` → 문제의 "within the cluster" 요구를 정확히 만족 (외부 노출 X).
- `PORT(S)=6379/TCP` → 요청한 포트 그대로 노출됨.
- `Selector: tier=msg` → **내가 selector를 적어준 적이 없는데도** `expose` 가 `messaging` Pod의 label(`tier=msg`)을 그대로 가져와서 만들어준 결과. label-free Pod였으면 여기가 비어 endpoints도 안 잡혔을 것.
- `Endpoints: 172.17.0.8:6379` → 실제 Pod IP가 등록됨 = selector 매칭 성공. 비어 있으면 selector ↔ Pod label 불일치를 의심해야 함.

> 💡 `kubectl expose pod <name>` 은 해당 Pod의 **모든 label** 을 selector로 가져옴. 그래서 Pod에 라벨이 여러 개 붙어 있으면 그중 하나만 같은 다른 Pod까지 같이 잡히지 않고, **AND 조건**으로 그 Pod만 정확히 매칭됨.

### 정리: imperative로 서비스 만들 때 체크리스트

1. "within the cluster" = `ClusterIP` (default), "from outside the node" = `NodePort`, "external LB" = `LoadBalancer`
2. `kubectl expose <pod|deployment|service> <name>` — 대상 리소스 종류 헷갈리지 말 것
3. `--port` 와 `--target-port` 가 같으면 `--target-port` 생략 가능
4. selector는 자동으로 대상 리소스의 label에서 가져옴 → label이 없거나 안 맞으면 `Endpoints` 가 빈다
5. `--help` 의 Examples 블록을 베이스로 복붙하는 게 가장 안전한 imperative 패턴

---

## Case 5) Deployment imperative 생성

> Create a deployment named `hr-web-app` using the image `kodekloud/webapp-color` with 2 replicas.

단순 생성 문제. `kubectl create deployment`에 `--replicas` 옵션이 포함되어 있어서 한 줄로 끝남.

```bash
kubectl create deployment hr-web-app \
  --image=kodekloud/webapp-color \
  --replicas=2
```

> 💡 `--replicas` 플래그를 잊었다면 일단 만들고 `k scale deployment hr-web-app --replicas=2`로 늘려도 됨. 한 줄로 끝내려면 `--replicas`가 더 깔끔.

---

## Case 6) orange Pod init container 트러블슈팅

> A new application `orange` is deployed. There is something wrong with it. Identify and fix the issue.

### 1. 전수 조사

```bash
k get all
# pod/orange   0/1   Init:CrashLoopBackOff   1 (11s ago)   14s
```

`Init:CrashLoopBackOff` → init container가 계속 실패하고 있음.

### 2. describe로 원인 추적

```bash
k describe pod orange
```

핵심 부분:

```
Init Containers:
  init-myservice:
    Command:
      sh
      -c
      sleeeep 2;
    State:    Terminated
      Reason: Error
      Exit Code: 127
```

`Exit Code 127` = command not found. command가 `sleeeep`으로 오타.

### 3. 수정 후 재배포

Pod의 `containers`/`initContainers` 필드는 in-place 수정 불가 → yaml 추출 → 삭제 → 재적용.

```bash
# 현재 spec을 yaml로 추출
k get pod orange -o yaml > orange.yaml

# orange.yaml에서 sleeeep → sleep 으로 오타 수정

# 기존 Pod 삭제 후 재배포
k delete pod orange --force
k apply -f orange.yaml
```

### 4. 확인

```bash
k get pods
# orange   1/1   Running   0   8s
```

> 💡 `Exit Code 127`은 거의 항상 "명령어 자체가 PATH에 없음". 원인은 보통 셋 중 하나 — command 오타, 잘못된 바이너리 경로, 이미지에 해당 명령어 미포함.

### 정리: Init 컨테이너 디버깅 체크리스트

1. `k describe pod <name>` → `Init Containers` 섹션의 State / Reason / Exit Code 먼저 확인
2. Exit Code별 대표 원인:
   - `0` → 정상 종료 (CrashLoop면 재시작 정책 의심)
   - `1` → 스크립트 내부 오류 (앱 로그 확인)
   - `127` → command 자체가 없음 (오타/PATH)
   - `137` → SIGKILL (OOM 가능성)
3. Pod의 `command`/`image` 수정은 in-place 불가 → yaml 추출 → 삭제 → 재적용

---

## Case 7) Deployment를 고정 NodePort로 노출

> Expose the `hr-web-app` created in the previous task as a service named `hr-web-app-service`, accessible on port `30082` on the nodes of the cluster.
>
> The web application listens on port `8080`.

### ⚠️ 처음 시도 (실패)

`--port`에 nodePort 번호를 잘못 매핑한 케이스.

```bash
k expose deployment hr-web-app \
  --name=hr-web-app-service \
  --port=30082 \
  --target-port=8080 \
  --type=NodePort
```

> ⚠️ `--port`는 **서비스 자체가 노출하는 포트**(ClusterIP 단의 포트)이지 nodePort가 아님. 그리고 `kubectl expose`에는 nodePort를 지정할 옵션이 없음 → 30000번대에서 랜덤 할당되어버림.

### 1. 잘못된 서비스 제거

```bash
k delete svc hr-web-app-service
```

### 2. dry-run으로 yaml 뽑고 nodePort 직접 박기

```bash
k expose deployment hr-web-app \
  --name=hr-web-app-service \
  --port=8080 \
  --target-port=8080 \
  --type=NodePort \
  --dry-run=client -o yaml > hr-svc.yaml
```

`hr-svc.yaml`에 `nodePort: 30082` 추가:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: hr-web-app
  name: hr-web-app-service
spec:
  type: NodePort
  selector:
    app: hr-web-app
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30082 # 명시적으로 지정
      protocol: TCP
```

```bash
k apply -f hr-svc.yaml
```

### 3. 확인

```bash
k get svc
# hr-web-app-service   NodePort   172.20.109.199   <none>   8080:30082/TCP   4s
```

`PORT(S)`가 `8080:30082/TCP`면 OK — 왼쪽이 service port, 오른쪽이 nodePort.

> 💡 NodePort를 **특정 번호로 고정**해야 하는 문제는 `kubectl expose`만으로는 불가능 → 항상 dry-run으로 yaml 뽑아서 `nodePort:` 직접 명시.

---

## Case 8) PersistentVolume 생성 (hostPath)

> Create a Persistent Volume with the given specification:
>
> - Volume name: `pv-analytics`
> - Storage: `100Mi`
> - Access mode: `ReadWriteMany`
> - Host path: `/pv/data-analytics`

`pv.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-analytics
spec:
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 100Mi
  hostPath:
    path: /pv/data-analytics
```

```bash
k apply -f pv.yaml
```

### 확인

```bash
k get pv
# NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      AGE
# pv-analytics   100Mi      RWX            Retain           Available   6s
```

> 💡 PV에 `storageClassName`을 명시하지 않으면 기본값이 `""`(빈 문자열). 정적 바인딩 시에는 PVC도 `storageClassName: ""`로 맞춰주면 매칭됨.

---

## Case 9) HPA + scaleDown stabilization window

> Create a Horizontal Pod Autoscaler (HPA) with name `webapp-hpa` for the deployment named `kkapp-deploy` in the `default` namespace with the `webapp-hpa.yaml` file located under the root folder.
>
> Ensure that the HPA scales the deployment based on CPU utilization, maintaining an average CPU usage of 50% across all pods.
>
> Configure the HPA to cautiously scale down pods by setting a stabilization window of 300 seconds to prevent rapid fluctuations in pod count.

### 1. 초기 파일 상태 (metrics / behavior 빠져 있음)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kkapp-deploy
  minReplicas: 2
  maxReplicas: 10
```

### 2. metrics + behavior 추가

`webapp-hpa.yaml` 최종:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kkapp-deploy
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

```bash
k apply -f webapp-hpa.yaml
```

포인트:

- "average CPU usage 50%" → `target.type: Utilization` + `averageUtilization: 50` (CPU만 보면 됨)
- "stabilization window 300s" + "cautiously scale down" → `behavior.scaleDown.stabilizationWindowSeconds: 300`
- `behavior` 필드는 `autoscaling/v2`에서만 지원 (`v1` X)

> 💡 `behavior`에는 `scaleUp` / `scaleDown` 두 정책을 따로 줄 수 있음. 문제에 "cautiously scale down"이라고 명시되어 있으면 `scaleDown`만 건드리면 됨.

---

## Case 10) VPA (Recreate 모드)

> Deploy a Vertical Pod Autoscaler (VPA) with name `analytics-vpa` for the deployment named `analytics-deployment` in the `default` namespace.
>
> The VPA should automatically adjust the CPU and memory requests of the pods to optimize resource utilization. Ensure that the VPA operates in `Recreate` mode, allowing it to evict and recreate pods with updated resource requests as needed.

`vpa.yaml`:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: analytics-vpa
  namespace: default
spec:
  targetRef:
    apiVersion: 'apps/v1'
    kind: Deployment
    name: analytics-deployment
  updatePolicy:
    updateMode: 'Recreate'
```

```bash
k apply -f vpa.yaml
```

포인트:

- VPA는 코어 리소스가 아니라 **CRD** (`autoscaling.k8s.io/v1`) → 클러스터에 VPA 컴포넌트가 미리 설치되어 있어야 함
- `updateMode` 종류:
  - `Off` → 권장값만 계산 (실제 변경 X)
  - `Initial` → Pod 생성 시점에만 권장값 적용
  - `Recreate` → Pod을 evict 후 재생성하면서 권장값 반영 (문제에서 요구)
  - `Auto` → 현재는 `Recreate`와 동일
- HPA(가로/replicas)와 VPA(세로/requests)는 같은 metric(CPU/memory)에 동시에 걸면 충돌 → 보통 둘 중 하나만

> 💡 VPA CRD가 설치되어 있는지 확인은 `k get crd | grep -i verticalpodautoscaler` (Mock Test 1 Case 3에서 다룬 그 CRD들).

---

## Case 11) Gateway 리소스 생성

> Create a Kubernetes Gateway resource with the following specifications:
>
> - Name: `web-gateway`
> - Namespace: `nginx-gateway`
> - Gateway Class Name: `nginx`
> - Listeners:
>   - Protocol: HTTP
>   - Port: 80
>   - Name: http

`gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

```bash
k apply -f gateway.yaml
```

포인트:

- API group은 `gateway.networking.k8s.io/v1` (Ingress의 `networking.k8s.io/v1`과 다름)
- `gatewayClassName`이 가리키는 `GatewayClass`가 이미 클러스터에 설치되어 있어야 함 → `k get gatewayclass`로 확인 가능
- `listeners[]`는 배열 → 여러 포트/프로토콜 동시 노출 가능. 문제처럼 하나면 한 항목만

> 💡 Gateway API는 Ingress의 후속 표준. 시험에선 구조 외워서 그대로 작성하는 게 빠름 — 헷갈리면 `k explain gateway.spec.listeners`로 필드 확인.

---

## Case 12) Helm chart 업그레이드

> One co-worker deployed a podinfo helm chart `kk-mock1` in the `kk-ns` namespace on the cluster. A new update is pushed to the helm chart, and the team wants you to update the helm repository to fetch the new changes.
>
> After updating the helm chart, upgrade the helm chart version to `6.11.2`.

### 1. 현재 상태 확인

```bash
helm repo list
# NAME       URL
# kk-mock1   https://stefanprodan.github.io/podinfo

helm list -n kk-ns
# NAME       NAMESPACE   REVISION   STATUS     CHART            APP VERSION
# kk-mock1   kk-ns       1          deployed   podinfo-6.11.0   6.11.0
```

설치된 chart는 `podinfo-6.11.0` → 목표는 `6.11.2`.

### 2. repo 갱신 (새 버전을 받아오려면 먼저 update 필요)

```bash
helm repo update kk-mock1
# ...Successfully got an update from the "kk-mock1" chart repository
```

### 3. 사용 가능한 버전 확인

```bash
helm search repo kk-mock1/podinfo --versions
# NAME               CHART VERSION   APP VERSION
# kk-mock1/podinfo   6.11.2          6.11.2
```

### 4. 업그레이드

```bash
helm upgrade kk-mock1 kk-mock1/podinfo --version 6.11.2 -n kk-ns
```

`helm upgrade <release> <chart>` 형식이 헷갈리기 쉬움:

- `<release>` = 현재 클러스터에 설치된 이름 (`kk-mock1`)
- `<chart>` = repo 안의 chart 경로 (`kk-mock1/podinfo`) — repo 이름과 release 이름이 같아서 혼동되지만 다른 인자임

### 5. 확인

```bash
helm list -n kk-ns
# kk-mock1   kk-ns   2   deployed   podinfo-6.11.2   6.11.2

k -n kk-ns get deployments.apps
# kk-mock1-podinfo   1/1   1   1   10m
```

`REVISION`이 2로 올라가고 `CHART`가 `podinfo-6.11.2`면 성공.

### 정리: helm upgrade 체크리스트

1. `helm repo update <repo>` — 새 버전 정보부터 동기화 (안 하면 옛날 버전만 보임)
2. `helm search repo <repo>/<chart> --versions` — 실제로 받을 수 있는 버전 확인
3. `helm upgrade <release> <repo>/<chart> --version <ver> -n <ns>` — release / chart / namespace 셋 다 정확히
4. 결과는 `helm list -n <ns>`의 `REVISION` 증가 + `CHART` 버전 변경으로 확인
5. 롤백 필요하면 `helm rollback <release> <revision>`

# Mock Test 2

## Case 1) Default StorageClass 생성 (no-provisioner)

> Create a StorageClass named `local-sc` with the following specifications and set it as the default storage class:
>
> - The provisioner should be `kubernetes.io/no-provisioner`
> - The volume binding mode should be `WaitForFirstConsumer`
> - Volume expansion should be enabled

### 1. 핵심 필드 정리

| 항목                   | 값                                                               |
| ---------------------- | ---------------------------------------------------------------- |
| `provisioner`          | `kubernetes.io/no-provisioner` (수동 PV 바인딩용)                |
| `volumeBindingMode`    | `WaitForFirstConsumer` (Pod 스케줄 후 바인딩)                    |
| `allowVolumeExpansion` | `true`                                                           |
| default 지정           | annotation `storageclass.kubernetes.io/is-default-class: "true"` |

### 2. manifest 작성 & 적용

```yaml
# local-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: 'true'
provisioner: kubernetes.io/no-provisioner
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

```bash
k apply -f local-sc.yaml

k get sc
# local-sc (default)   kubernetes.io/no-provisioner   ...   WaitForFirstConsumer   true   ...
```

> 💡 default StorageClass는 어디까지나 **annotation**으로 지정. `spec` 필드가 따로 있는 게 아님.
> 💡 `no-provisioner`는 동적 프로비저닝 안 함 → PV를 수동으로 만들어 줘야 PVC가 바인딩됨. `WaitForFirstConsumer`와 함께 쓰는 게 local volume의 정석 패턴.

---

## Case 2) Sidecar 컨테이너로 로그 tail (Deployment)

> Create a deployment named `logging-deployment` in the namespace `logging-ns` with 1 replica:
>
> - Main container `app-container` (image `busybox`) runs: `sh -c "while true; do echo 'Log entry' >> /var/log/app/app.log; sleep 5; done"`
> - Sidecar container `log-agent` (image `busybox`) runs: `tail -f /var/log/app/app.log`
> - `log-agent` logs should display the entries logged by `app-container`.

### 1. 사이드카는 native sidecar (initContainer + restartPolicy: Always) 패턴

K8s 1.28+ 부터 `initContainers`에 `restartPolicy: Always`를 주면 **메인 컨테이너와 같이 동작하는 사이드카**가 됨. 메인보다 먼저 시작하고, 메인이 죽어도 같이 안 죽는 게 장점.

### 2. manifest

```yaml
# logging-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logging-deployment
  namespace: logging-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: logger
  template:
    metadata:
      labels:
        app: logger
    spec:
      volumes:
        - name: log-volume
          emptyDir: {}
      initContainers:
        - name: log-agent
          image: busybox
          command: ['sh', '-c', 'touch /var/log/app/app.log; tail -f /var/log/app/app.log']
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/app
          restartPolicy: Always # native sidecar
      containers:
        - name: app-container
          image: busybox
          command:
            ['sh', '-c', "while true; do echo 'Log entry' >> /var/log/app/app.log; sleep 5; done"]
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/app
```

> ⚠️ `tail -f`는 파일이 없으면 즉시 죽음 → 사이드카가 메인보다 먼저 떠도 안전하게 `touch`로 파일을 먼저 만들어 둠.

### 3. 적용 & 확인

```bash
k apply -f logging-deployment.yaml

# 사이드카 로그가 메인이 쓴 내용을 받는지 확인
kubectl -n logging-ns logs deploy/logging-deployment -c log-agent -f
# Log entry
# Log entry
# ...
```

### 정리: 멀티컨테이너 로깅 패턴 체크리스트

1. 공유 파일은 `emptyDir` + 두 컨테이너에 같은 `volumeMounts`
2. 사이드카는 `initContainers` + `restartPolicy: Always` (1.28+ native sidecar)
3. `tail -f` 사이드카는 파일이 없을 때를 대비해 `touch` 먼저
4. 검증은 `-c <container>` 로 컨테이너 지정해서 로그 봄

---

## Case 3) Ingress 생성 (ingressClassName 함정)

> A Deployment `webapp-deploy` is running in the `ingress-ns` namespace and is exposed via Service `webapp-svc`. Create an Ingress `webapp-ingress` in the same namespace:
>
> - `pathType: Prefix`
> - Route `/` to backend service on port `80`
> - Host: `kodekloud-ingress.app`
>
> Test: `curl -s http://kodekloud-ingress.app/`

### 1. 처음 만든 manifest (불완전)

```yaml
# webapp-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: ingress-ns
spec:
  rules:
    - host: kodekloud-ingress.app
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: webapp-svc
                port:
                  number: 80
```

```bash
k apply -f webapp-ingress.yaml
curl -s http://kodekloud-ingress.app/
# <html><head><title>404 Not Found</title></head>...
# nginx 404 → Ingress 컨트롤러까지는 도달했지만, 룰이 적용 안 된 상태
```

### 2. `ingressClassName` 누락이 원인

```bash
kubectl get ingressclass
# NAME    CONTROLLER             PARAMETERS   AGE
# nginx   k8s.io/ingress-nginx   <none>       7m
```

클러스터에 IngressClass가 default로 마크돼 있지 않으면 Ingress가 어떤 컨트롤러를 쓸지 모름 → `spec.ingressClassName: nginx` 명시.

```yaml
spec:
  ingressClassName: nginx # 추가
  rules:
    - host: kodekloud-ingress.app
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: webapp-svc
                port:
                  number: 80
```

```bash
k apply -f webapp-ingress.yaml
curl -s http://kodekloud-ingress.app/
# <h1>Welcome to nginx!</h1>  ← 정상
```

> ⚠️ Ingress 만들었는데 404가 나오면, **컨트롤러까지는 갔지만 룰이 매치 안 됐다**는 신호. 의심 순서: (1) `ingressClassName` 누락, (2) `host`/`path` 오타, (3) backend `service.name` / `port` 오타, (4) Service의 `selector`가 실제 Pod 라벨과 안 맞음.

### 정리: Ingress 404 트러블슈팅 체크리스트

1. `kubectl get ingressclass` 로 컨트롤러 확인 → default 아니면 `spec.ingressClassName` 명시
2. `kubectl describe ingress <name> -n <ns>` 로 backend가 정상 endpoint를 잡았는지 확인
3. `kubectl get svc,ep -n <ns>` 로 Service endpoint가 비어있는지 확인 (selector 미스매치)
4. `curl -H "Host: <host>" http://<ingress-ip>/` 로 호스트 헤더만의 문제인지 분리

---

## Case 4) Deployment imperative 생성 + 이미지 롤링 업데이트 (apply)

> Create a new deployment called `nginx-deploy`, image `nginx:1.16`, 1 replica. Upgrade the deployment to `nginx:1.17` using rolling update.
>
> Note: Use the `kubectl apply` command to create or update the deployment.

### 1. dry-run으로 manifest 뽑기

```bash
kubectl create deployment nginx-deploy \
  --image=nginx:1.16 --replicas=1 \
  $do > nginx-deploy.yaml
```

```yaml
# nginx-deploy.yaml (핵심 부분만)
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-deploy
  template:
    spec:
      containers:
        - image: nginx:1.16
          name: nginx
```

```bash
k apply -f nginx-deploy.yaml
# deployment.apps/nginx-deploy created
```

### 2. 이미지 변경 후 다시 apply (롤링 업데이트)

`strategy`를 별도로 지정하지 않으면 Deployment 기본 전략이 `RollingUpdate`라 그냥 manifest의 이미지 태그만 바꿔 다시 apply하면 됨.

```bash
# nginx-deploy.yaml 의 image: nginx:1.16 → nginx:1.17 로 수정
k apply -f nginx-deploy.yaml
# deployment.apps/nginx-deploy configured

k rollout status deploy/nginx-deploy
# deployment "nginx-deploy" successfully rolled out
```

> 💡 문제에 "use `kubectl apply`" 라고 명시돼 있으면 `kubectl set image`나 `kubectl edit` 대신 **manifest 수정 후 apply** 흐름을 지켜야 함. 채점 스크립트가 `kubectl.kubernetes.io/last-applied-configuration` annotation을 보고 판단할 수 있음.

---

## Case 5) CSR로 john 인증서 발급 + Role/RoleBinding

> Create a new user called `john`. Grant him access to the cluster using a CSR named `john-developer`. Create a role `developer` granting John permission to `create, list, get, update, delete` pods in the `development` namespace. Private key at `/root/CKA/john.key`, CSR at `/root/CKA/john.csr`.
>
> Important: As of k8s 1.19, the CSR object expects a `signerName`.

### 1. CSR 파일을 base64로 인코딩

```bash
cat /root/CKA/john.csr | base64 -w 0
# LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0K... (한 줄로 출력)
```

> ⚠️ `-w 0` 빼먹으면 줄바꿈이 들어가서 yaml에 그대로 못 붙임. 항상 `-w 0`.

### 2. CertificateSigningRequest 생성 & 승인

```yaml
# csr.yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john-developer
spec:
  signerName: kubernetes.io/kube-apiserver-client # 사용자 인증용 시그너
  request: <위에서 뽑은 base64 문자열>
  usages:
    - client auth
```

```bash
k apply -f csr.yaml

k get csr
# NAME             AGE   SIGNERNAME                            CONDITION
# john-developer   7s    kubernetes.io/kube-apiserver-client   Pending

k certificate approve john-developer

k get csr
# john-developer   65s   ...   Approved,Issued
```

### 3. Role 생성 (development 네임스페이스 안에서 pods CRUD)

```yaml
# developer.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: development
rules:
  - apiGroups: [''] # core API group
    resources: ['pods']
    verbs: ['create', 'list', 'get', 'update', 'delete']
```

```bash
k apply -f developer.yaml
```

### 4. RoleBinding으로 john에 Role 연결

```yaml
# developer-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-role-binding
  namespace: development
subjects:
  - kind: User
    name: john
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

```bash
k apply -f developer-binding.yaml
```

### 5. `auth can-i` 로 권한 검증

```bash
# 허용돼야 하는 것들 (전부 yes)
kubectl auth can-i create pods --as=john -n development   # yes
kubectl auth can-i list   pods --as=john -n development   # yes
kubectl auth can-i get    pods --as=john -n development   # yes
kubectl auth can-i update pods --as=john -n development   # yes
kubectl auth can-i delete pods --as=john -n development   # yes

# 거부돼야 하는 것들 (전부 no)
kubectl auth can-i create pods         --as=john -n default       # no  (네임스페이스 다름)
kubectl auth can-i list   deployments  --as=john -n development   # no  (리소스 다름)
```

### 정리: CSR 기반 사용자 추가 체크리스트

1. `cat <csr> | base64 -w 0` 으로 인코딩 (줄바꿈 금지)
2. CSR yaml에 `signerName: kubernetes.io/kube-apiserver-client` 와 `usages: ["client auth"]` 명시
3. `kubectl certificate approve <name>` 으로 승인 → `Approved,Issued`
4. Role / RoleBinding은 **같은 네임스페이스** (Role은 네임스페이스 스코프)
5. RoleBinding `subjects.name`은 CSR의 CN(`CN=john`)과 정확히 일치해야 함
6. 마지막에 꼭 `kubectl auth can-i ... --as=<user>` 로 양/음성 케이스 모두 검증

---

## Case 6) ClusterIP 서비스 + Pod IP DNS 조회 (busybox)

> Create an nginx pod `nginx-resolver` and expose it internally using a ClusterIP service `nginx-resolver-service`. From within the cluster verify:
>
> - DNS resolution of the service name
> - Network reachability of the pod using its IP address
>
> Use `busybox:1.28` to perform the lookups.
> Save service DNS lookup → `/root/CKA/nginx.svc`, pod IP lookup → `/root/CKA/nginx.pod`.

### 1. Pod + ClusterIP 서비스 imperative 생성

```bash
kubectl run nginx-resolver --image=nginx
# pod/nginx-resolver created

k expose pod nginx-resolver \
  --name=nginx-resolver-service \
  --port=80 --target-port=80
# service/nginx-resolver-service exposed
```

### 2. 서비스명 DNS 조회 → `nginx.svc`

busybox 임시 Pod을 띄워서 nslookup만 실행하고 자동 삭제 (`--rm -it --restart=Never`).

```bash
k run busybox --image=busybox:1.28 --rm -it --restart=Never \
  -- nslookup nginx-resolver-service > /root/CKA/nginx.svc

cat /root/CKA/nginx.svc
# Server:    172.20.0.10
# Address 1: 172.20.0.10 kube-dns.kube-system.svc.cluster.local
#
# Name:      nginx-resolver-service
# Address 1: 172.20.95.147 nginx-resolver-service.default.svc.cluster.local
```

### 3. Pod IP DNS 조회 → `nginx.pod`

Pod IP를 직접 nslookup 하려면 **`172.17.1.14` 가 아니라 `172-17-1-14.<ns>.pod.cluster.local`** 형태로 변환해야 함 (점을 하이픈으로).

```bash
k get pod -o wide | grep nginx-resolver
# nginx-resolver   1/1   Running   0   14m   172.17.1.14   node01   <none>   <none>

k run busybox --image=busybox:1.28 --rm -it --restart=Never \
  -- nslookup 172-17-1-14.default.pod.cluster.local > /root/CKA/nginx.pod

cat /root/CKA/nginx.pod
# Name:      172-17-1-14.default.pod.cluster.local
# Address 1: 172.17.1.14 172-17-1-14.nginx-resolver-service.default.svc.cluster.local
```

> 💡 Pod의 DNS 이름 규칙: `<pod-ip-with-dashes>.<namespace>.pod.cluster.local`. IP의 `.`을 `-`로만 바꾸면 됨.
> ⚠️ `nslookup <pod-ip>` (숫자 IP) 는 PTR 역방향 조회를 시도하다 실패하기 쉽고, 채점에서 요구하는 출력 포맷과도 다름. 반드시 **하이픈 형태 DNS 이름**으로 조회.

### 정리: 클러스터 내부 DNS 조회 체크리스트

1. 임시 검증 Pod은 `--rm -it --restart=Never` 로 자동 정리
2. busybox는 **1.28** 버전을 써야 `nslookup`이 정상 동작 (최신 버전엔 빠져 있음)
3. Service DNS: `<svc>.<ns>.svc.cluster.local`
4. Pod DNS: `<pod-ip-with-dashes>.<ns>.pod.cluster.local`
5. 결과 저장은 `> /path` 리다이렉트로 — 컨테이너 stdout이 그대로 파일에 들어감

---

## Case 7) node01에 Static Pod 생성

> Create a static pod on `node01` called `nginx-critical` with the image `nginx`. Make sure that it is recreated/restarted automatically in case of a failure.
>
> For example, use `/etc/kubernetes/manifests` as the static Pod path.

### 1. node01로 SSH 접속 후 manifest 디렉토리로 이동

Static Pod의 manifest 파일은 kubelet이 감시하는 디렉토리에 두면 됨 (보통 `/etc/kubernetes/manifests`).

```bash
ssh node01
cd /etc/kubernetes/manifests/
```

### ⚠️ `kubectl run --dry-run` 으로 yaml 뽑는 시도가 실패

node01에는 보통 kubeconfig가 없어서 `kubectl run`이 동작하지 않음.

```bash
kubectl run nginx-critical --image=nginx --dry-run=client -o yaml > nginx-critical.yaml
# Error from server (NotFound): the server could not find the requested resource
```

> ⚠️ 워커 노드(`node01`)에는 API 서버에 접근할 kubeconfig가 없을 수 있음 → `kubectl` 명령은 control plane에서 쓰고, **노드 위에서는 manifest 파일을 직접 작성**하는 게 안전.

### 2. manifest 파일 직접 작성

```bash
cat <<EOF > /etc/kubernetes/manifests/nginx-critical.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-critical
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
          protocol: TCP
EOF
```

파일을 두기만 하면 kubelet이 알아서 감지해서 Pod을 띄움 — `kubectl apply` 같은 거 따로 안 해도 됨.

### 3. control plane에서 확인

```bash
k get pod -o wide
# NAME                    READY   STATUS    RESTARTS   AGE   IP            NODE
# nginx-critical-node01   1/1     Running   0          83s   172.17.1.11   node01
```

> 💡 Static Pod은 API에 등록될 때 이름 뒤에 노드 이름이 자동으로 붙음 (`nginx-critical` → `nginx-critical-node01`). 시험에서 헷갈리지 말 것.

### 정리: Static Pod 체크리스트

1. 작업은 **노드 위에서** — `ssh node01` 후 `/etc/kubernetes/manifests/`
2. manifest 파일을 그 경로에 두기만 하면 kubelet이 띄움
3. `kubectl run --dry-run` 으로 베이스를 못 받는 환경이면 처음부터 직접 작성
4. "자동 재시작" 요구는 static Pod 자체 특성으로 만족 — kubelet이 죽으면 다시 띄움
5. API 상으로 보면 이름에 `-<node-name>` suffix가 붙는 점 인지

---

## Case 8) HPA — 메모리 기반 (template 수정)

> Create a Horizontal Pod Autoscaler with name `backend-hpa` for the deployment named `backend-deployment` in the `backend` namespace with the `webapp-hpa.yaml` file located under the root folder.
>
> Ensure that the HPA scales the deployment based on memory utilization, maintaining an average memory usage of 65% across all pods.
>
> Configure the HPA with a minimum of 3 replicas and a maximum of 15.

### 1. 대상 Deployment 확인

```bash
k -n backend get deployments.apps
# NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
# backend-deployment   3/3     3            3           77s
```

### 2. 제공된 template 상태

`webapp-hpa.yaml` (초기):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kkapp-deploy
  minReplicas: 3
  maxReplicas: 15
```

`name`, `scaleTargetRef.name`이 잘못 들어가 있고 `metrics`가 빠져 있음.

### 3. 최종 manifest

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-deployment
  minReplicas: 3
  maxReplicas: 15
  metrics:
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 65
```

```bash
k apply -f webapp-hpa.yaml
```

### 4. 결과 확인

```bash
k describe hpa backend-hpa -n backend
# Reference:        Deployment/backend-deployment
# Metrics:          resource memory on pods (as a percentage of request):  <unknown> / 65%
# Min replicas:     3
# Max replicas:     15
```

> ⚠️ `<unknown>` + `FailedGetResourceMetric` 이벤트가 뜰 수 있음 — 클러스터에 metrics-server가 없으면 실제 측정값을 못 가져옴. HPA 스펙 자체는 정답이므로 시험에선 OK.

> 💡 HPA 메모리 기반 포인트: `metrics[0].resource.name: memory`, `target.type: Utilization`, `averageUtilization: 65`. CPU는 `name: cpu`로만 바꾸면 동일 (Case 9 in Mock Test 1 참고).

---

## Case 9) Gateway에 HTTPS listener 추가

> Modify the existing `web-gateway` on `cka5673` namespace to handle HTTPS traffic on port 443 for `kodekloud.com`, using a TLS certificate stored in a secret named `kodekloud-tls`.

### 1. 현재 Gateway 상태 확인

```bash
k get gateway -n cka5673
# NAME          CLASS       ADDRESS   PROGRAMMED   AGE
# web-gateway   kodekloud             Unknown      87s
```

기존 listener는 HTTP 80 하나만 있음.

### 2. `k edit`으로 HTTPS listener 추가

```bash
k edit gateway web-gateway -n cka5673
```

`spec.listeners`에 HTTPS 항목을 추가:

```yaml
spec:
  gatewayClassName: kodekloud
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      hostname: kodekloud.com
      allowedRoutes:
        namespaces:
          from: Same
    - name: https
      port: 443
      protocol: HTTPS
      hostname: kodekloud.com
      tls:
        certificateRefs:
          - name: kodekloud-tls
      allowedRoutes:
        namespaces:
          from: Same
```

### 3. 확인

```bash
k get gateway web-gateway -n cka5673 -o yaml
# spec.listeners[1]:
#   name: https
#   port: 443
#   protocol: HTTPS
#   tls:
#     certificateRefs:
#       - group: ""
#         kind: Secret
#         name: kodekloud-tls
#     mode: Terminate
```

저장 후 API 서버가 `tls.mode: Terminate`, `certificateRefs[].kind: Secret`을 기본값으로 채워줌.

> 💡 Gateway HTTPS listener의 필수 4종:
>
> 1. `protocol: HTTPS`
> 2. `port: 443`
> 3. `hostname` (TLS SNI 매칭)
> 4. `tls.certificateRefs[].name` (Secret 참조)

> ⚠️ HTTPS listener에서 `hostname`을 안 적으면 SNI 매칭이 안 돼서 TLS 핸드셰이크가 깨질 수 있음 — 문제에 호스트가 명시되면 반드시 같이 적기.

---

## Case 10) 취약 이미지 helm release 찾아서 삭제

> On the cluster, the team has installed multiple helm charts on a different namespace. By mistake, those deployed resources include one of the vulnerable images called `kodekloud/webapp-color:v1`. Find out the release name and uninstall it.

### 1. 해당 이미지를 쓰는 Pod 검색

전체 네임스페이스 Pod의 컨테이너 이미지를 custom-columns로 출력하고 grep:

```bash
k get pods -A -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[].image | grep -i kodekloud/webapp-color:v1
# atlanta-page-apd-655896796-4w89n   kodekloud/webapp-color:v1
# atlanta-page-apd-655896796-bsjfh   kodekloud/webapp-color:v1
# ...
```

Pod 이름 prefix가 `atlanta-page-apd-...` → helm release 이름과 매칭됨.

### 2. helm release 목록과 대조

```bash
helm list -A
# NAME                 NAMESPACE          REVISION   STATUS     CHART
# atlanta-page-apd     atlanta-page-04    1          deployed   atlanta-page-apd-0.1.0
# digi-locker-apd      digi-locker-02     1          deployed   digi-locker-apd-0.1.0
# security-alpha-apd   security-alpha-01  1          deployed   security-alpha-apd-0.1.0
# web-dashboard-apd    web-dashboard-03   1          deployed   web-dashboard-apd-0.1.0
```

→ `atlanta-page-apd` 가 범인.

### 3. uninstall

```bash
helm uninstall atlanta-page-apd -n atlanta-page-04
# release "atlanta-page-apd" uninstalled
```

### 4. 검증

```bash
k get pods -A -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[].image | grep -i kodekloud/webapp-color:v1
# (출력 없음)
```

> 💡 `-o custom-columns=NAME:...,IMAGE:.spec.containers[].image` 는 시험에서 자주 쓰는 패턴. 컨테이너가 여러 개인 Pod은 첫 번째만 잡히니 정밀하게 보려면 `.spec.containers[*].image`.

> 💡 helm release는 **namespace 단위로 관리**됨 → `helm uninstall <release>` 시 `-n <ns>` 반드시 같이 (또는 `helm list -A`로 namespace 먼저 확인).

---

## Case 11) NetworkPolicy — 가장 제한적인 정책 고르기

> You are requested to create a NetworkPolicy to allow traffic from frontend apps located in the `frontend` namespace, to backend apps located in the `backend` namespace, but not from the databases in the `databases` namespace. There are three policies available in the `/root` folder. Apply the **most restrictive** policy from the provided YAML files. Do not delete any existing policies.

### 1. 제공된 3개 정책 비교

| 파일             | from selector                                                       | 의미                                     | 적합?                         |
| ---------------- | ------------------------------------------------------------------- | ---------------------------------------- | ----------------------------- |
| `net-pol-1.yaml` | `namespaceSelector matchLabels: access=allowed`                     | `access=allowed` 라벨 붙은 모든 ns 허용  | ❌ 너무 일반적·불명확         |
| `net-pol-2.yaml` | `namespaceSelector: name=frontend` **+** `name=databases` (두 항목) | frontend **와** databases 양쪽 모두 허용 | ❌ databases가 들어가면 안 됨 |
| `net-pol-3.yaml` | `namespaceSelector: name=frontend` 만                               | frontend만 허용, databases·기타 ns 차단  | ✅ 정답                       |

`net-pol-3.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: net-policy-3
  namespace: backend
spec:
  podSelector: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: frontend
      ports:
        - protocol: TCP
          port: 80
```

### 2. 적용

```bash
k apply -f net-pol-3.yaml
# networkpolicy.networking.k8s.io/net-policy-3 created
```

### 정리: NetworkPolicy 고를 때 체크리스트

1. `from:` 배열의 각 항목은 **OR** — 항목이 늘어날수록 허용 범위가 넓어짐 (더 느슨해짐)
2. 한 항목 안의 `namespaceSelector` + `podSelector` 조합은 **AND** — 더 좁아짐
3. 문제에서 "허용할 출발지" 하나만 명시되면 → 그 ns만 selector에 두는 정책이 가장 제한적
4. `access=allowed` 같은 메타 라벨에 의존하면 다른 ns가 같은 라벨을 달기만 해도 통과돼서 위험
5. 기존 정책 건드리지 말라는 지시 → `k apply -f` 로 새 정책 추가만, `delete` 금지

# Mock Test 3

## Case 1) kubeadm용 sysctl 파라미터 영구 적용

> You are an administrator preparing your environment to deploy a Kubernetes cluster using `kubeadm`. Adjust the following network parameters on the system to the following values, and make sure your changes persist reboots:
>
> - `net.ipv4.ip_forward = 1`
> - `net.bridge.bridge-nf-call-iptables = 1`

### 1. 어디에 써야 영구 적용되는가

런타임에만 박는 `sysctl -w` 는 재부팅 후 날아감. **`/etc/sysctl.d/*.conf`** 에 적어두면 부팅 시 자동 재적용됨. kubeadm 공식 문서가 안내하는 패턴 그대로.

참고: https://kubernetes.io/docs/setup/production-environment/container-runtimes/

### 2. 파일 생성 + 즉시 적용

```bash
# 영구 설정 파일 작성
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

# 재부팅 없이 바로 적용
sudo sysctl --system
```

### 3. 적용 확인

```bash
sysctl net.ipv4.ip_forward net.bridge.bridge-nf-call-iptables
# net.ipv4.ip_forward = 1
# net.bridge.bridge-nf-call-iptables = 1
```

> 💡 `sysctl --system` 은 `/etc/sysctl.d/`, `/usr/lib/sysctl.d/`, `/etc/sysctl.conf` 를 전부 재로딩. 출력에 다른 파일 경고가 잔뜩 나와도 마지막에 `k8s.conf` 가 적용되면 OK.
> 💡 `bridge-nf-call-iptables` 가 먹으려면 `br_netfilter` 모듈이 로드돼 있어야 함. kubeadm pre-flight 단계에서 보통 `/etc/modules-load.d/k8s.conf` 에 `br_netfilter` 도 같이 등록함 (이 문제는 sysctl만 묻고 있어서 생략).

---

## Case 2) ServiceAccount + ClusterRole + Pod 연결

> Create a new service account with the name `pvviewer`. Grant this Service account access to list all PersistentVolumes in the cluster by creating an appropriate cluster role called `pvviewer-role` and ClusterRoleBinding called `pvviewer-role-binding`.
> Next, create a pod called `pvviewer` with the image: `redis` and serviceAccount: `pvviewer` in the default namespace.

### 1. SA → ClusterRole → ClusterRoleBinding 순서로 imperative 생성

전부 명령형으로 가능. PV는 클러스터 스코프 리소스라서 **Role이 아니라 ClusterRole** 필요.

```bash
# 1) ServiceAccount
kubectl create serviceaccount pvviewer

# 2) ClusterRole — list verb on persistentvolumes
kubectl create clusterrole pvviewer-role \
  --verb=list --resource=persistentvolumes

# 3) ClusterRoleBinding — SA는 ns:name 형태로
kubectl create clusterrolebinding pvviewer-role-binding \
  --clusterrole=pvviewer-role \
  --serviceaccount=default:pvviewer
```

### 2. Pod에 serviceAccount 박기

`kubectl run` 으로는 `--serviceaccount` 플래그가 없으니 dry-run 으로 yaml 뽑아 수정.

```bash
k run pvviewer --image=redis --dry-run=client -o yaml > pvviewer.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pvviewer
  name: pvviewer
spec:
  serviceAccountName: pvviewer # ← 추가
  containers:
    - image: redis
      name: pvviewer
```

```bash
k apply -f pvviewer.yaml
```

> 💡 `--serviceaccount=<ns>:<name>` 포맷 절대 헷갈리지 말 것. ns 생략하면 default로 가긴 하지만 명시하는 게 안전.
> 💡 verb는 `list` 만 요구함. `get`, `watch` 까지 안 줘도 됨 — 시험 문제 그대로 최소로 맞춰주는 게 정답.

---

## Case 3) StorageClass 만들기 (rancher-sc)

> Create a StorageClass named `rancher-sc` with the following specifications:
>
> - The provisioner should be `rancher.io/local-path`
> - The volume binding mode should be `WaitForFirstConsumer`
> - Volume expansion should be enabled

### manifest

StorageClass는 imperative 생성 명령이 없어서 yaml 직접 작성.

```yaml
# rancher-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rancher-sc
provisioner: rancher.io/local-path
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

```bash
k apply -f rancher-sc.yaml
k get sc rancher-sc
```

> 💡 `volumeBindingMode` 와 `allowVolumeExpansion` 은 `spec` 안이 아니라 **최상위 필드**. `parameters:` 자리에 넣으면 안 됨 — 자주 틀리는 부분.

---

## Case 4) ConfigMap → Deployment envFrom 주입

> Create a ConfigMap named `app-config` in the namespace `cm-namespace` with the following key-value pairs:
>
> - `ENV=production`
> - `LOG_LEVEL=info`
>
> Then, modify the existing Deployment named `cm-webapp` in the same namespace to use the `app-config` ConfigMap by setting the environment variables `ENV` and `LOG_LEVEL` in the container from the ConfigMap.

### 1. ConfigMap 생성

imperative로도 됨: `k -n cm-namespace create cm app-config --from-literal=ENV=production --from-literal=LOG_LEVEL=info`. yaml 쪽이 더 명확함:

```yaml
# app-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: cm-namespace
data:
  ENV: 'production'
  LOG_LEVEL: 'info'
```

```bash
k apply -f app-config.yaml
```

### 2. Deployment에 envFrom 주입

키 이름 두 개를 그대로 env 변수로 매핑하면 되니까 `env:` 개별 매핑 대신 **`envFrom:`** 한 줄이면 끝.

```bash
k -n cm-namespace edit deployment cm-webapp
```

```yaml
spec:
  template:
    spec:
      containers:
        - name: nginx
          image: nginx
          envFrom:
            - configMapRef:
                name: app-config
```

> 💡 `envFrom` 은 ConfigMap의 모든 key를 컨테이너 env로 통째로 import. 두 개만 골라 쓰려면 `env: - valueFrom.configMapKeyRef` 로 명시. 이번 문제는 키 두 개가 전부니까 envFrom이 짧고 깔끔.
> 💡 Deployment를 edit 하면 자동으로 새 ReplicaSet으로 롤링됨 — 별도 `rollout restart` 불필요.

---

## Case 5) PriorityClass 적용 (기존 Pod 재생성 필요)

> Create a PriorityClass named `low-priority` with a value of `50000`. A pod named `lp-pod` exists in the namespace `low-priority`. Modify the pod to use the priority class you created. Recreate the pod if necessary.

### 1. PriorityClass 생성

```bash
kubectl create priorityclass low-priority --value=50000
```

### 2. 기존 Pod의 priorityClassName은 immutable

Pod의 `spec.priorityClassName` (그리고 자동 계산된 `priority`) 은 **생성 후 수정 불가**. `k edit` 으로 바꾸려고 하면 reject 됨 → 문제도 "Recreate the pod if necessary" 라고 미리 힌트를 줌.

```bash
# 현재 spec 덤프
k -n low-priority get pod lp-pod -o yaml > lp-pod.yaml
```

`lp-pod.yaml` 에서 정리할 것:

- `status:`, `metadata.creationTimestamp`, `resourceVersion`, `uid` 등 **런타임 필드 전부 제거**
- `spec.priority: 0` 줄 **삭제** (priorityClassName으로부터 자동 계산되어야 함)
- `spec.priorityClassName: low-priority` **추가**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lp-pod
  namespace: low-priority
  labels:
    run: lp-pod
spec:
  priorityClassName: low-priority # ← 추가
  containers:
    - image: nginx
      name: lp-pod
  # ... 나머지 spec 유지
```

### 3. 삭제 후 재생성

```bash
k -n low-priority delete pod lp-pod --force
k apply -f lp-pod.yaml

k get pod -n low-priority
# lp-pod   1/1   Running   0   18s
```

> ⚠️ `priority` 와 `priorityClassName` 둘 다 immutable. 덤프한 yaml에 `priority: 0` 이 남아있으면 새 우선순위와 충돌나거나 무시될 수 있음 → **삭제하고 priorityClassName만 남기는 게 안전**.
> 💡 `--force` 는 grace period 0 으로 즉시 삭제. 시험에선 시간 아끼려고 자주 쓰지만, 실무에선 데이터 정합성 위험.

---

## Case 6) NetworkPolicy — 모든 출발지 허용 (port 80)

> A pod called `np-test-1` and a service called `np-test-service` have been deployed in the default namespace. A default-deny NetworkPolicy is currently blocking all ingress traffic to pods in this namespace, which is why the service is unreachable.
>
> Create a new NetworkPolicy named `ingress-to-nptest` in the default namespace that allows ingress traffic from all sources to the `np-test-1` pod on port 80.
>
> **Important:** Don't delete any current objects deployed.

### 1. 핵심: `from:` 자체를 비우면 "모든 출발지 허용"

`from:` 을 명시하지 않은 `ingress` 룰은 **모든 namespace / 모든 pod / 모든 IP 에서의 트래픽을 허용**. `ports:` 만 적으면 됨.

```yaml
# ingress-to-nptest.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-to-nptest
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: np-test-1
  policyTypes:
    - Ingress
  ingress:
    - ports:
        - protocol: TCP
          port: 80
```

```bash
k apply -f ingress-to-nptest.yaml
```

### 2. 확인

```bash
kubectl describe networkpolicy ingress-to-nptest
# Allowing ingress traffic:
#   To Port: 80/TCP
#   From: <any> (traffic not restricted by source)
```

`From: <any>` 가 핵심 — 출발지 제한이 없다는 뜻.

> 💡 default-deny는 그대로 두고 **새 정책을 추가만** 함. NetworkPolicy는 여러 개가 OR 로 합쳐지므로, 더 느슨한 정책 하나만 추가하면 그쪽으로 트래픽이 뚫림.
> ⚠️ "Don't delete any current objects" 지시 → default-deny 건드리면 감점. `delete` 금지, `apply` 만.

---

## Case 7) Taint + Toleration 으로 스케줄 분리

> Taint the worker node `node01` to be Unschedulable. Once done, create a pod called `dev-redis`, image `redis:alpine`, to ensure workloads are not scheduled to this worker node. Finally, create a new pod called `prod-redis` and image: `redis:alpine` with toleration to be scheduled on `node01`.
>
> - `key: env_type`, `value: production`, `operator: Equal`, `effect: NoSchedule`

### 1. 노드 taint

```bash
kubectl taint nodes node01 env_type=production:NoSchedule
```

### 2. dev-redis — toleration 없음 (node01 회피)

toleration이 없으니 알아서 다른 노드(controlplane 등)로 스케줄됨.

```bash
k run dev-redis --image=redis:alpine
```

### 3. prod-redis — toleration 추가 (node01 허용)

`kubectl run` 으로는 toleration 지정 불가 → dry-run 으로 yaml 뽑고 수정.

```bash
k run prod-redis --image=redis:alpine --dry-run=client -o yaml > prod-redis.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prod-redis
  labels:
    run: prod-redis
spec:
  containers:
    - image: redis:alpine
      name: prod-redis
  tolerations:
    - key: 'env_type'
      operator: 'Equal'
      value: 'production'
      effect: 'NoSchedule'
```

```bash
k apply -f prod-redis.yaml
```

### 4. 배치 확인

```bash
k get pods -o wide
# dev-redis    Running   controlplane
# prod-redis   Running   node01
```

> 💡 Toleration의 4개 필드 (`key`, `operator`, `value`, `effect`) 는 taint와 **정확히 일치**해야 매칭. `operator: Exists` 면 `value` 생략 가능하지만, 문제에서 `Equal` 을 못박았으니 `value` 도 필수.
> ⚠️ Toleration은 "그 노드에 스케줄돼도 된다"는 허가일 뿐, **강제하지 않음**. 스케줄러가 다른 노드로 보낼 수도 있음. 강제로 node01에 박고 싶다면 `nodeSelector` 나 `nodeName` 을 같이 줘야 함 (이 문제는 요구 안 함).
> ⚠️ dry-run 출력의 metadata.name/labels 가 의도와 다르게 나오면 (예: `name: pod`) 직접 `prod-redis` 로 고쳐줄 것 — 안 그러면 다른 이름으로 생성됨.

---

## Case 8) PVC ↔ PV 바인딩 실패 (accessModes 불일치)

> A PersistentVolumeClaim named `app-pvc` exists in the namespace `storage-ns`, but it is not getting bound to the available PersistentVolume named `app-pv`. Inspect both the PVC and PV and identify why the PVC is not being bound and fix the issue so that the PVC successfully binds to the PV. **Do not modify the PV resource.**

### 1. PVC / PV 양쪽 스펙 비교

```bash
k -n storage-ns get pvc app-pvc -o yaml
k get pv app-pv -o yaml
```

| 항목         | PV (`app-pv`) | PVC (`app-pvc`) |
| ------------ | ------------- | --------------- |
| storage      | 1Gi           | 1Gi             |
| accessModes  | **RWO**       | **RWM**         |
| volumeMode   | Filesystem    | Filesystem      |
| storageClass | (없음)        | (없음)          |

→ `accessModes` 불일치. PV는 RWO인데 PVC가 RWM 요구 → 매칭 실패. 문제에서 "Do not modify the PV" 라고 못박았으니 **PVC 쪽을 RWO로 바꿈**.

### 2. PVC를 RWO로 재생성

PVC는 한번 만들어지면 `accessModes` 같은 핵심 필드를 in-place로 못 고침 → **삭제 후 재생성**.

```bash
k -n storage-ns delete pvc app-pvc
```

```yaml
# app-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
  namespace: storage-ns
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeMode: Filesystem
```

```bash
k apply -f app-pvc.yaml
```

### 3. 바인딩 확인

```bash
k -n storage-ns get pvc
# NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES
# app-pvc   Bound    app-pv   1Gi        RWO
```

> 💡 PVC ↔ PV 바인딩 매칭 조건: `storage`(요청 ≤ 용량), `accessModes`(요청이 PV가 지원하는 모드 안에 들어가야 함), `storageClassName`, `volumeMode`, `selector` 다섯 가지. `Pending` 이면 이 다섯부터 비교.
> ⚠️ "Do not modify the PV" 같은 단서가 있을 땐 어느 쪽을 손대야 하는지 이미 정해진 것 — 일치 방향 헷갈리지 말 것.

---

## Case 9) super.kubeconfig 포트 오류 수정

> A kubeconfig file called `super.kubeconfig` has been created under `/root/CKA`. There is something wrong with the configuration. Troubleshoot and fix it.

### 1. kubeconfig server 주소 의심

```bash
cat /root/CKA/super.kubeconfig
# server: https://controlplane:9999    ← 의심
```

### 2. 실제 kube-apiserver 포트 확인

```bash
k describe pod kube-apiserver-controlplane -n kube-system | grep -i port
#   Port:      6443/TCP (probe-port)
#   Host Port: 6443/TCP (probe-port)
#     --secure-port=6443
```

apiserver는 6443에서 listen — kubeconfig의 `9999` 가 잘못됨.

### 3. 포트를 6443으로 수정

`server: https://controlplane:6443` 으로 변경 후 검증:

```bash
k --kubeconfig=/root/CKA/super.kubeconfig get nodes
```

> 💡 kubeconfig 트러블슈팅 순서: ① `server:` URL/포트 → ② `certificate-authority` 경로 → ③ `user:` 의 client-cert/token → ④ `context:` 가 가리키는 cluster/user 이름이 실제로 정의돼 있는지. 위에서부터 차례로 확인하면 거의 다 잡힘.

---

## Case 10) Deployment scale 안 됨 — kube-controller-manager 망가짐

> We have created a new deployment called `nginx-deploy`. Scale the deployment to 3 replicas. Has the number of replicas increased? Troubleshoot and fix the issue.

### 1. 일단 스케일 명령

```bash
k scale deploy nginx-deploy --replicas=3
# deployment.apps/nginx-deploy scaled

k get deploy nginx-deploy
# NAME           READY   UP-TO-DATE   AVAILABLE
# nginx-deploy   1/3     1            1
```

Deployment의 `spec.replicas` 는 3으로 반영됐는데 ReplicaSet은 여전히 `desired: 1`. HPA도 없음 → **컨트롤러가 일을 안 하는 상황**.

```bash
k get deploy nginx-deploy -o yaml | grep -A2 "^spec:"
# spec:
#   progressDeadlineSeconds: 600
#   replicas: 3        ← Deployment 자체는 3 요구

k get hpa
# No resources found

k describe rs <rs-name>
# Annotations: deployment.kubernetes.io/desired-replicas: 1   ← RS는 1만 desired
```

### 2. control plane 컴포넌트 점검

```bash
k get pods -n kube-system
# kube-controller-manager-controlplane   0/1   CrashLoopBackOff   6 (42s ago)
```

→ Deployment → ReplicaSet 동기화를 담당하는 `kube-controller-manager` 가 죽어 있음.

### 3. CrashLoopBackOff 원인

```bash
k describe pod kube-controller-manager-controlplane -n kube-system
# Command:
#   kube-contro1ler-manager        ← 'l' 자리에 숫자 '1'
# Message: exec: "kube-contro1ler-manager": executable file not found in $PATH
```

### 4. static pod manifest 수정

```bash
vim /etc/kubernetes/manifests/kube-controller-manager.yaml
```

```yaml
spec:
  containers:
    - command:
        - kube-controller-manager # contro1ler → controller
        - --allocate-node-cidrs=true
        # ...
```

저장하면 kubelet이 static pod을 자동 재기동.

### 5. 결과 확인

```bash
k get pods -n kube-system
# kube-controller-manager-controlplane   1/1   Running

k get pods
# nginx-deploy-99bcf7fdc-4nbht   1/1   Running
# nginx-deploy-99bcf7fdc-n8hhh   1/1   Running
# nginx-deploy-99bcf7fdc-njzs9   1/1   Running
```

> ⚠️ Deployment `spec.replicas` 만 늘어나고 ReplicaSet이 따라오지 않으면 → **`kube-controller-manager` 상태부터**. Deployment 컨트롤러는 controller-manager 안에서 돈다.
> 💡 시험에서 `1`↔`l`, `0`↔`O`, `I`↔`l` 같은 글리프 함정이 흔함. `cat` 결과만 슥 보지 말고, command나 path는 한 글자씩 비교하거나 `which kube-controller-manager` 같이 실제로 실행 가능한지 따져볼 것.

---

## Case 11) HPA — 커스텀 메트릭 (Pods 타입)

> Create a Horizontal Pod Autoscaler `api-hpa` for `api-deployment` in the `api` namespace. Scale based on a custom metric `requests_per_second`, targeting an average value of 1000 across all pods. `minReplicas: 1`, `maxReplicas: 20`. Ignore errors due to metric not being tracked.

### 1. 커스텀 메트릭은 `autoscaling/v2` 필수

`autoscaling/v1` 은 CPU 비율만 지원. 커스텀 메트릭/메모리/외부 메트릭은 모두 `autoscaling/v2`.
공식 문서 [HPA walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/) 의 `Pods` 타입 예시를 그대로 차용.

### 2. manifest 작성

```yaml
# api-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-deployment
  minReplicas: 1
  maxReplicas: 20
  metrics:
    - type: Pods
      pods:
        metric:
          name: requests_per_second
        target:
          type: AverageValue
          averageValue: 1k
```

```bash
k apply -f api-hpa.yaml
```

| `metric.type` | 용도                                       | target 종류                    |
| ------------- | ------------------------------------------ | ------------------------------ |
| `Resource`    | CPU/Memory (내장)                          | `Utilization` / `AverageValue` |
| `Pods`        | Pod 당 평균값 (이 케이스)                  | `AverageValue` 만              |
| `Object`      | 단일 오브젝트 (예: Ingress 의 qps)         | `Value` / `AverageValue`       |
| `External`    | 클러스터 외부 (SQS 큐, 클라우드 메트릭 등) | `Value` / `AverageValue`       |

> 💡 "average value of 1000 across all pods" 같은 워딩 → `Pods` + `AverageValue: 1k`. `Utilization`(%) 은 `Resource` 타입에서만 쓰임.
> ⚠️ `averageValue: 1000` 도 동일 의미지만 시험에서는 `1k` 표기가 더 흔함 — 채점기는 둘 다 받음.

---

## Case 12) HTTPRoute 가중치 기반 트래픽 분기

> Configure the `web-route` to split traffic between `web-service` and `web-service-v2`. 80% → `web-service`, 20% → `web-service-v2`. Gateway/Service 들은 이미 생성돼 있음.

### 1. 환경 확인

```bash
k get gateway
# web-gateway   nginx   True   28s

k get svc
# web-service       ClusterIP   ...   80/TCP
# web-service-v2    ClusterIP   ...   80/TCP
```

### 2. HTTPRoute 작성 (`backendRefs[].weight`)

```yaml
# web-route.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
  labels:
    gateway: web-gateway
spec:
  parentRefs:
    - name: web-gateway
  rules:
    - backendRefs:
        - name: web-service
          port: 80
          weight: 80
        - name: web-service-v2
          port: 80
          weight: 20
```

```bash
k apply -f web-route.yaml
```

### 3. 채택 확인

```bash
k describe httproute web-route
# Status:
#   Parents:
#     Conditions:
#       Reason: Accepted        ← 게이트웨이가 라우트 수락
#       Reason: ResolvedRefs    ← 백엔드 Service 매칭 성공
```

> 💡 `weight` 는 퍼센트가 아니라 **상대값**. 80/20 이든 8/2 든 4/1 이든 결과 동일. 다만 문제에서 퍼센트로 요구하면 그대로 80/20 으로 적는 게 안전 (채점 정규식에 걸릴 가능성).
> ⚠️ `Accepted: True` 만 보고 끝내지 말 것. `ResolvedRefs` 가 False 면 backend Service 이름/네임스페이스/포트 오타 가능성.

---

## Case 13) Helm — 새 chart로 release 교체 (blue-green)

> `webpage-server-01` 이 Helm으로 배포돼 있음. `/root/new-version` 에 새 버전 chart가 있음. 검증 → `webpage-server-02` 라는 **새 release** 로 설치 → 확인 후 `webpage-server-01` 제거.

### 1. 현재 release 확인

```bash
helm list
# NAME                REVISION   CHART                     APP VERSION
# webpage-server-01   1          webpage-server-01-0.1.0   v1
```

### 2. 새 chart 검증

```bash
helm lint /root/new-version/
# [INFO] Chart.yaml: icon is recommended
# 1 chart(s) linted, 0 chart(s) failed
```

`failed: 0` 이면 통과. `INFO`/`WARNING` 은 경고일 뿐 막지 않음.

### 3. 새 release 설치

```bash
helm install webpage-server-02 /root/new-version/
# STATUS: deployed
# REVISION: 1

helm list
# webpage-server-01   1   webpage-server-01-0.1.0   v1
# webpage-server-02   1   webpage-server-02-0.1.1   v2
```

### 4. 새 release Pod이 Running 인 걸 확인한 뒤 구 release 제거

```bash
k get pods | grep webpage
# webpage-server-02-...   1/1   Running

helm uninstall webpage-server-01
# release "webpage-server-01" uninstalled
```

> 💡 `helm upgrade webpage-server-01 /root/new-version` 이 아니라 **새 release 이름으로 install** 하는 게 문제의 요구. 두 release 가 잠시 공존하다 구버전을 내리는 blue-green 패턴.
> ⚠️ uninstall 전에 반드시 새 release 의 Pod 가 `Running` 상태인지 먼저 확인 — 안 그러면 다운타임 발생.

---

## Case 14) Pod CIDR — kubeadm-config ConfigMap 에서 추출

> CNI 설치 전 cluster-wide Pod network CIDR을 확인. `kubeadm-config` ConfigMap의 `podSubnet` 값을 `x.x.x.x/x` 형식으로 `/root/pod-cidr.txt` 에 저장. (per-node CIDR 아님)

### 1. ConfigMap 에서 값 뽑기

```bash
k describe configmap kubeadm-config -n kube-system | grep -i podSubnet
#   podSubnet: 172.17.0.0/16
```

### 2. 파일에 CIDR 만 저장

```bash
echo "172.17.0.0/16" > /root/pod-cidr.txt
```

> 💡 헷갈리는 두 종류
>
> - **cluster-wide podSubnet** (`kubeadm-config` ConfigMap) → CNI 에 알려줄 값. 이 케이스의 정답.
> - **per-node podCIDR** (`k get node -o jsonpath='{.items[*].spec.podCIDR}'`) → 각 노드에 잘라 할당된 슬라이스. 항상 cluster-wide 의 부분집합.
>
> ⚠️ 문제의 Note 가 명시적으로 `kubectl get node` 의 값이 아니라고 짚는 이유 — 둘 중 무엇을 묻는지 헷갈리지 말 것.
