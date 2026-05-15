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
