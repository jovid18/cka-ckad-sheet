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
