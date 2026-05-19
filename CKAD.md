# Lightning Lab 1

## Case 1) PV / PVC / Pod 한 세트로 hostPath 마운트

> Create a PersistentVolume named `log-volume` with:
>
> - `StorageClassName: manual` (already created — do not create it again)
> - `AccessModes: ReadWriteMany (RWX)`
> - `Capacity: 1Gi`
> - `hostPath: /opt/volume/nginx`
>
> Then create a PVC `log-claim` (requests ≥ 200Mi, binds to `log-volume`).
> Then create a Pod `logger` (`nginx:alpine`) mounting `log-claim` at `/var/www/nginx`.
> All resources in `default` namespace.

PV → PVC → Pod 순서로 만들어 바인딩 사슬을 완성하는 기본 패턴.

### 1. PersistentVolume

```yaml
# log-volume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: log-volume
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  storageClassName: manual
  hostPath:
    path: /opt/volume/nginx
```

```bash
k apply -f log-volume.yaml
```

### 2. PersistentVolumeClaim

PV와 매칭되도록 `storageClassName`, `accessModes`를 동일하게 맞춤. 용량은 PV(1Gi) 이하인 200Mi 요청.

```yaml
# log-claim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: log-claim
spec:
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
      storage: 200Mi
  storageClassName: manual
```

```bash
k apply -f log-claim.yaml

k get pvc
# NAME        STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS
# log-claim   Bound    log-volume   1Gi        RWX            manual
```

`Bound` 떨어지면 PVC 준비 완료.

### 3. Pod manifest (run 으로 베이스 생성 후 volume 직접 추가)

`kubectl run`은 volume 옵션이 없어서 dry-run 으로 베이스만 뽑고 yaml에서 직접 추가.

```bash
export do='--dry-run=client -o yaml'
kubectl run logger --image=nginx:alpine $do > logger.yaml
```

`logger.yaml` 에 `volumes` / `volumeMounts` 추가:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: logger
  labels:
    run: logger
spec:
  containers:
    - name: logger
      image: nginx:alpine
      volumeMounts:
        - name: log-volume
          mountPath: /var/www/nginx
  volumes:
    - name: log-volume
      persistentVolumeClaim:
        claimName: log-claim
```

```bash
k apply -f logger.yaml
```

> 💡 Pod 안의 volume 이름(`log-volume`)이 PV 이름과 같을 필요는 없음 — PVC를 통해 PV 가 묶이고, Pod 은 `claimName: log-claim` 으로만 참조함.

---

## Case 2) NetworkPolicy 트러블슈팅 — default-deny 를 selective allow 로 교체

> `secure-pod` 와 `secure-service` 가 이미 떠있고, `webapp-color` → `secure-pod` 인커밍 연결이 실패.
> NetworkPolicy 를 수정/생성해서 `webapp-color` pod 에서 TCP 80 으로 `secure-pod` 에 인바운드 허용.
> 기존 객체는 삭제/재생성 금지 (명시적으로 시키는 경우 제외).

핵심: 기존에 `podSelector: {}` + `policyTypes: [Ingress]` 인 **모든 Pod 인바운드 deny** 정책이 깔려 있음. 같은 이름으로 manifest 를 덮어써서 selective allow 로 바꾸는 게 가장 깔끔.

### 1. 현재 상태 파악

```bash
# 서비스의 selector 와 타겟 Pod 라벨 일치 여부
k get svc secure-service -o yaml | grep -A2 selector:
# selector:
#   run: secure-pod          ← Pod 라벨과 매칭 OK

# 깔린 NetworkPolicy
k get networkpolicies
# NAME           POD-SELECTOR   AGE
# default-deny   <none>         2m

k get networkpolicies default-deny -o yaml
# spec:
#   podSelector: {}          ← 모든 Pod 대상
#   policyTypes: [Ingress]   ← 인바운드 전부 차단 (allow rule 없음)
```

양쪽 Pod 라벨도 확인:

```bash
k describe pod secure-pod   | grep -i labels
# Labels: run=secure-pod
k describe pod webapp-color | grep -i labels
# Labels: name=webapp-color
```

> ⚠️ `secure-pod` 라벨 key 는 `run`, `webapp-color` 라벨 key 는 `name`. **양쪽이 다른 key 를 쓰는 함정** — 정책 작성 시 그대로 옮겨 적어야 함.

### 2. default-deny 를 같은 이름으로 덮어쓰기 (selective allow)

기존 객체는 삭제하지 않고 manifest 만 apply → `configured` 로 갱신됨.

```yaml
# deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: secure-pod # 대상: secure-pod 만
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              name: webapp-color # 출처: webapp-color
      ports:
        - protocol: TCP
          port: 80
```

```bash
k apply -f deny.yaml
# networkpolicy.networking.k8s.io/default-deny configured
```

> 💡 `policyTypes: [Ingress]` + `ingress: [...]` 구조: 명시한 rule 에 매칭되는 트래픽만 허용, 나머지는 차단. allow rule 을 추가하는 것 = "이 트래픽만 통과시킨다" 라는 뜻.

### 정리: NetworkPolicy 트러블슈팅 체크리스트

1. `k get networkpolicies` 로 깔린 정책 전부 확인
2. 각 정책의 `podSelector` (대상) / `policyTypes` / 실제 rule 구분
3. 양쪽 Pod 의 **라벨 key/value** 확인 (`run=` vs `name=` 같은 작은 차이가 자주 함정)
4. 서비스가 의도한 Pod 을 정말 가리키는지 (`selector` ↔ Pod 라벨)
5. 같은 이름으로 manifest apply → `configured` 로 덮어쓰기 가능 (객체 삭제 없이 수정)

---

## Case 3) ConfigMap envFrom + emptyDir + 무한 루프

> `dvl1987` namespace 에 `time-config` ConfigMap (`TIME_FREQ=10`) 생성.
> `time-check` Pod (`busybox` 이미지)이 `while true; do date; sleep $TIME_FREQ; done` 를 실행해서 결과를 `/opt/time/time-check.log` 로 출력.
> `/opt/time` 은 Pod lifecycle 동안 유지되는 volume 으로 마운트.

### 1. Namespace + ConfigMap

```bash
k create ns dvl1987
```

```yaml
# time-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-config
  namespace: dvl1987
data:
  TIME_FREQ: '10' # 숫자도 string 으로 (configmap 은 string-only)
```

```bash
k apply -f time-config.yaml
```

### 2. Pod manifest (dry-run + 수동 보강)

```bash
k run time-check -n dvl1987 --image=busybox $do > time-check.yaml
```

`time-check.yaml` 에 `args`, `envFrom`, `volumeMounts`, `volumes` 추가:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: time-check
  namespace: dvl1987
spec:
  containers:
    - name: time-check
      image: busybox
      args:
        - /bin/sh
        - -c
        - 'while true; do date >> /opt/time/time-check.log; sleep $TIME_FREQ; done'
      envFrom:
        - configMapRef:
            name: time-config
      volumeMounts:
        - name: config-volume
          mountPath: /opt/time
  volumes:
    - name: config-volume
      emptyDir: {}
  restartPolicy: Always
```

```bash
k apply -f time-check.yaml
```

> 💡 `date` 자체는 stdout 으로 찍는 명령이라, **컨테이너 안에서 `>>` 로 리다이렉트**해서 로그 파일에 누적시킴. Pod 종료 후 데이터가 필요 없으니 `emptyDir` 로 충분 (Pod lifecycle 동안만 유지).

### 3. 검증

```bash
k -n dvl1987 exec time-check -- cat /opt/time/time-check.log
# Mon May 18 13:21:16 UTC 2026
# Mon May 18 13:21:26 UTC 2026
# Mon May 18 13:21:36 UTC 2026
# ...
```

10초 간격으로 timestamp 가 누적되면 성공.

> ⚠️ `args` 안 명령을 flow-style 한 줄 (`[/bin/sh, -c, '...']`) 로 적을 때 single quote / yaml 인용 충돌이 잦음. 블록 스타일 (`- /bin/sh`, `- -c`, `- "..."`) 이 더 안전.

---

## Case 4) Deployment RollingUpdate 전략 + 업그레이드 + 롤백

> `nginx-deploy` Deployment 생성: container `nginx`, image `nginx:1.16`, replicas 4, strategy `RollingUpdate` with `maxSurge=1`, `maxUnavailable=2`.
> 그다음 `1.17` 로 업그레이드, 모든 Pod 이 올라간 뒤 이전 버전으로 롤백.

### 1. Deployment manifest (전략은 수동 보강)

`kubectl create deployment` 는 strategy 파라미터를 못 받으니 dry-run 후 직접 추가.

```bash
kubectl create deployment nginx-deploy --image=nginx:1.16 --replicas=4 $do > nd.yaml
```

`nd.yaml` 의 `spec` 에 strategy 추가:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx-deploy
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx-deploy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2
      maxSurge: 1
  template:
    metadata:
      labels:
        app: nginx-deploy
    spec:
      containers:
        - name: nginx
          image: nginx:1.16
```

```bash
k apply -f nd.yaml
```

### 2. 이미지 업그레이드 (1.16 → 1.17)

둘 다 가능 — 어느 쪽이든 동일한 새 ReplicaSet 이 만들어짐.

```bash
# 한 줄 명령으로
k set image deployment/nginx-deploy nginx=nginx:1.17

# or 에디터로 image 라인만 수정
k edit deployment nginx-deploy
```

진행 확인:

```bash
k describe deployment nginx-deploy
# Annotations: deployment.kubernetes.io/revision: 2
# Pod Template:
#   nginx:
#     Image: nginx:1.17
# NewReplicaSet: nginx-deploy-5f98968576 (4/4 replicas created)
```

revision 이 `2` 로 올라가고 새 ReplicaSet 이 4/4 까지 가면 완료.

### 3. 롤백

```bash
k rollout undo deployment nginx-deploy
# deployment.apps/nginx-deploy rolled back
```

확인:

```bash
k describe deployment nginx-deploy
# Annotations: deployment.kubernetes.io/revision: 3
# Pod Template:
#   nginx:
#     Image: nginx:1.16            ← 다시 1.16
# OldReplicaSets: nginx-deploy-5f98968576 (0/0)
# NewReplicaSet:  nginx-deploy-794ccddf5 (4/4)
```

revision 은 계속 증가하지만(`3`) 이미지 자체는 이전(`1.16`) ReplicaSet 으로 복귀.

> 💡 특정 revision 으로 가려면 `k rollout history deployment/nginx-deploy` 로 목록 확인 후 `k rollout undo deployment/nginx-deploy --to-revision=N`.

---

## Case 5) Redis Deployment — resources + 두 종류 volume 한 번에

> `default` ns 에 `redis` Deployment: image `redis:alpine`, replicas 1, label `app=redis`, CPU request `200m`, containerPort `6379`.
> Volumes:
>
> - emptyDir `data` → `/redis-master-data`
> - ConfigMap volume `redis-config` → `/redis-master` (ConfigMap 은 이미 존재 — 재생성 금지)

요구사항을 한 번에 만족하는 manifest 를 dry-run 으로 만들고 resource / port / volume 만 보강하는 패턴.

### 1. Deployment manifest

```bash
kubectl create deployment redis --image=redis:alpine --replicas=1 $do > redis.yaml
```

`redis.yaml` 에 resources, ports, volumes 추가:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:alpine
          resources:
            requests:
              cpu: '200m'
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: redis-config
              mountPath: /redis-master
            - name: data
              mountPath: /redis-master-data
      volumes:
        - name: redis-config
          configMap:
            name: redis-config
        - name: data
          emptyDir: {}
```

```bash
k apply -f redis.yaml
```

### 2. 검증

```bash
k get deploy redis
# NAME    READY   UP-TO-DATE   AVAILABLE
# redis   1/1     1            1

k describe pod -l app=redis | grep -A2 -E 'Mounts|Volumes|Requests'
# Requests:
#   cpu: 200m
# Mounts:
#   /redis-master      from redis-config (rw)
#   /redis-master-data from data (rw)
```

> 💡 ConfigMap 을 **volume** 으로 마운트하면 각 key 가 파일로 풀림 (ex. ConfigMap 에 `redis.conf` key 가 있으면 `/redis-master/redis.conf` 로 생성). `envFrom` 으로 환경변수에 주입하는 것과는 다른 패턴.
> ⚠️ "ConfigMap 은 이미 만들어져 있음, 재생성 금지" 류 단서를 무시하고 `k create configmap` 치지 않도록 주의 — 시험에서 흔히 나오는 조건.

# Mock Test 1

## Case 1) Pod imperative 생성 (특정 이미지)

> Deploy a pod named `nginx-448839` using the `nginx:alpine` image.

```bash
k run nginx-448839 --image=nginx:alpine
# pod/nginx-448839 created
```

```bash
k get pod
# NAME           READY   STATUS    RESTARTS   AGE
# nginx-448839   1/1     Running   0          10s
```

---

## Case 2) Namespace 생성

> Create a namespace named `apx-z993845`.

```bash
k create ns apx-z993845
# namespace/apx-z993845 created
```

---

## Case 3) Deployment imperative 생성 (replicas 지정)

> Create a new Deployment named `httpd-frontend` with 3 replicas using image `httpd:2.4-alpine`.

```bash
kubectl create deployment httpd-frontend \
  --image=httpd:2.4-alpine \
  --replicas=3
# deployment.apps/httpd-frontend created
```

> 💡 `kubectl create deployment` 에서 `--replicas` 한 번에 지정 가능 — 별도로 `kubectl scale` 안 쳐도 됨.

---

## Case 4) Pod imperative 생성 + 라벨

> Deploy a messaging pod using the `redis:alpine` image with the labels set to `tier=msg`.

```bash
kubectl run messaging \
  --image=redis:alpine \
  --labels="tier=msg"
# pod/messaging created
```

```bash
k describe pod messaging | grep -i labels
# Labels: tier=msg
```

> 💡 `kubectl run --labels=` 로 Pod 생성 시점에 라벨을 박을 수 있음. 나중에 `k label pod` 로 붙이지 않아도 됨.

---

## Case 5) ReplicaSet — 잘못된 이미지 이름 수정

> ReplicaSet `rs-d33393` 가 생성됐는데 Pod 가 안 뜸. 원인을 찾아 고치고 Ready replicas 4를 만들 것.

### 1. 상태 확인

```bash
k get rs rs-d33393
# NAME        DESIRED   CURRENT   READY   AGE
# rs-d33393   4         4         0       37s

k get pod
# rs-d33393-8b8gp   0/1   InvalidImageName   0   17s
# ...
```

→ `InvalidImageName` 이미지 이름이 잘못된 거.

### 2. ReplicaSet template 수정

```bash
k edit rs rs-d33393
# spec.template.spec.containers[0].image:
#   busyboxXXXXXXX → busybox
```

### 3. 기존 Pod 강제 삭제 → ReplicaSet 가 새 template으로 재생성

```bash
k delete pod -l name=busybox-pod --force
```

```bash
k get pod
# rs-d33393-9lmh2   1/1   Running   0   8s
# rs-d33393-dtl2b   1/1   Running   0   9s
# rs-d33393-q6fh2   1/1   Running   0   9s
# rs-d33393-tspmk   1/1   Running   0   9s
```

> ⚠️ ReplicaSet 의 template 을 바꿔도 **기존 Pod 는 자동으로 재생성되지 않음** (Deployment 와 다른 점). 직접 지워야 새 template 으로 다시 만들어짐. selector 의 라벨로 한 번에 지우는 게 편함.

---

## Case 6) Cross-namespace Service 노출

> `marketing` namespace 의 redis deployment 를 ClusterIP 서비스 `messaging-service` 로 port 6379에 노출.

```bash
kubectl expose deployment redis \
  -n marketing \
  --port=6379 \
  --name=messaging-service
# service/messaging-service exposed
```

> 💡 `expose` 도 `-n` 으로 namespace 명시 가능 — deployment 가 default 가 아닌 경우 빠뜨리기 쉬움. service 도 같은 namespace 에 만들어짐.

---

## Case 7) Pod 환경변수 수정 (값 변경)

> Pod `webapp-color` 의 환경변수 `APP_COLOR` 를 `green` 으로 변경.

### 1. 현재 spec 추출 + 런타임 필드 정리

`k get pod -o yaml` 로 받은 다음, `status`, `creationTimestamp`, `annotations`(calico), `resourceVersion`, `uid` 등 런타임 필드 제거하고 핵심만 남김.

```bash
k get pod webapp-color -o yaml > wc.yaml
# 에디터로 status 블록 삭제, env value: pink → green 변경
```

### 2. 핵심 수정 부분

```yaml
spec:
  containers:
    - env:
        - name: APP_COLOR
          value: green # pink → green
      image: kodekloud/webapp-color
      name: webapp-color
```

### 3. 재배포

```bash
k delete pod webapp-color
k apply -f wc.yaml
# pod/webapp-color created
```

> ⚠️ Pod 은 env 를 in-place 로 수정 불가 — 삭제 후 재배포 필요. Deployment 라면 `k set env deployment/<name> KEY=value` 한 줄로 끝남.

---

## Case 8) ConfigMap from-literal 생성

> ConfigMap `cm-3392845` 생성:
>
> - `DB_NAME=SQL3322`
> - `DB_HOST=sql322.mycompany.com`
> - `DB_PORT=3306`

```bash
kubectl create configmap cm-3392845 \
  --from-literal=DB_NAME=SQL3322 \
  --from-literal=DB_HOST=sql322.mycompany.com \
  --from-literal=DB_PORT=3306
# configmap/cm-3392845 created
```

```bash
k describe cm cm-3392845
# Data
# ====
# DB_HOST: sql322.mycompany.com
# DB_NAME: SQL3322
# DB_PORT: 3306
```

---

## Case 9) Secret from-literal 생성

> Secret `db-secret-xxdf` 생성:
>
> - `DB_Host=sql01`
> - `DB_User=root`
> - `DB_Password=password123`

```bash
kubectl create secret generic db-secret-xxdf \
  --from-literal=DB_Host=sql01 \
  --from-literal=DB_User=root \
  --from-literal=DB_Password=password123
# secret/db-secret-xxdf created
```

값 검증 (base64 디코딩):

```bash
kubectl get secret db-secret-xxdf -o json | jq -r '.data | map_values(@base64d)'
# {
#   "DB_Host": "sql01",
#   "DB_Password": "password123",
#   "DB_User": "root"
# }
```

> 💡 한 키만 보려면 `kubectl get secret <name> -o jsonpath='{.data.DB_Host}' | base64 -d`. 전부 보려면 위처럼 `jq map_values(@base64d)` 가 편함.

---

## Case 10) Pod SecurityContext — runAsUser + capabilities

> Pod `app-sec-kff3345` 를 root (`uid 0`) 로 실행하고 `SYS_TIME` capability 부여.

### 1. spec 추출

```bash
kubectl get pod app-sec-kff3345 -o yaml > pod.yaml
# status 블록, 런타임 필드 정리
```

### 2. container-level securityContext 추가

```yaml
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ['sleep', '4800']
      securityContext:
        runAsUser: 0
        capabilities:
          add: ['SYS_TIME']
```

### 3. 재배포

```bash
k delete pod app-sec-kff3345 --force
k apply -f pod.yaml
# pod/app-sec-kff3345 created
```

> ⚠️ `runAsUser` 는 Pod-level / container-level 양쪽 다 가능하지만 `capabilities` 는 **container-level 에서만** 적용됨. 둘 다 container `securityContext` 에 같이 두는 게 안전.

---

## Case 11) 다른 namespace 의 Pod 로그 추출

> `e-com-1123` Pod 의 로그를 `/opt/outputs/e-com-1123.logs` 로 export. namespace 는 직접 찾아야 함.

### 1. namespace 찾기

```bash
k get pod -A | grep e-com-1123
# e-commerce   e-com-1123   1/1   Running   ...
```

### 2. 로그 파일로 저장

```bash
k -n e-commerce logs e-com-1123 > /opt/outputs/e-com-1123.logs
```

```bash
cat /opt/outputs/e-com-1123.logs
# [2026-05-18 16:24:02,853] INFO in event-simulator: USER3 is viewing page2
# ...
```

> 💡 `-A` (= `--all-namespaces`) 로 모든 ns 검색 후 `grep` 이 가장 빠른 위치 추적법. namespace 모를 때 첫 수.

---

## Case 12) PersistentVolume hostPath 생성

> 다음 스펙으로 PV 생성:
>
> - name: `pv-analytics`
> - storage: `100Mi`
> - accessModes: `ReadWriteMany`
> - hostPath: `/pv/data-analytics`

```yaml
# pva.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-analytics
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /pv/data-analytics
```

```bash
k apply -f pva.yaml
# persistentvolume/pv-analytics created
```

> ⚠️ hostPath PV 는 기본 `volumeMode: Filesystem` 을 쓰는 게 자연스러움. 문제에서 별도 요구가 없으면 `volumeMode: Block` 넣지 말 것 — Block 은 raw block 디바이스용이라 hostPath 와 안 맞음 (실제 검증 자동화에서 깎일 수 있음).

---

## Case 13) Deployment + ClusterIP Service + NetworkPolicy 한 세트

> `redis:alpine` 으로 redis Deployment (replicas=1, label `app=redis`) 만들고, ClusterIP service `redis` (port 6379) 로 노출. `access=redis` 라벨 단 Pod 만 접근 가능한 Ingress NetworkPolicy `redis-access` 생성.

### 1. Deployment

```bash
k create deployment redis \
  --image=redis:alpine \
  --replicas=1 \
  --dry-run=client -o yaml > redis.yaml
# label app=redis 는 create deployment 가 자동으로 박아줌
k apply -f redis.yaml
```

### 2. Service (ClusterIP, 기본값)

```bash
k expose deployment redis --port=6379 --name=redis
# service/redis exposed
```

### 3. NetworkPolicy

```yaml
# rnp.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-access
spec:
  podSelector:
    matchLabels:
      app: redis # 보호 대상: redis Pod
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              access: redis # 허용 출발지: access=redis 라벨 단 Pod
```

```bash
k apply -f rnp.yaml
```

> 💡 NetworkPolicy 의 최상위 `podSelector` 는 **정책 대상** (이 Pod 들에 정책을 건다), `from.podSelector` 는 **허용 출발지** — 헷갈리지 말 것.

---

## Case 14) 멀티 컨테이너 Pod (env + command)

> Pod `sega` 생성:
>
> - Container `tails`: `busybox`, command `sleep 3600`
> - Container `sonic`: `nginx`, env `NGINX_PORT=8080`

### 1. 한 컨테이너 기본 골격 뽑기

```bash
kubectl run sega \
  --image=nginx \
  --env="NGINX_PORT=8080" \
  --dry-run=client -o yaml > sega.yaml
```

### 2. 두 번째 컨테이너 추가

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: sega
  name: sega
spec:
  containers:
    - name: sonic
      image: nginx
      env:
        - name: NGINX_PORT
          value: '8080'
    - name: tails
      image: busybox
      command: ['sleep', '3600']
```

```bash
k apply -f sega.yaml
```

> 💡 imperative 명령으로는 한 컨테이너 Pod 만 만들 수 있음 — 멀티 컨테이너는 항상 `--dry-run=client -o yaml` 로 골격 뽑고 두 번째 컨테이너 손으로 추가하는 패턴.

# Lightning Lab 2

## Case 1) Readiness 실패 Pod 찾아서 고치고 livenessProbe 추가

> 클러스터의 여러 네임스페이스에 Pod 들이 떠 있다. **Not Ready** 상태인 Pod 을 찾아 트러블슈팅. 그 다음 같은 Pod 에 `ls /var/www/html/file_check` 가 실패하면 컨테이너를 재시작하도록 체크를 추가. delay 10s, period 60s. Pod 삭제 후 재생성 가능, probe 관련 warning 은 무시.

### 1. Not Ready Pod 찾기

```bash
k get pods -A
# dev1401   nginx1401   0/1   Running   ...   ← 얘만 READY 0/1
```

### 2. 원인 파악 — describe 이벤트 확인

```bash
k describe -n dev1401 pod nginx1401
# Warning  Unhealthy ... Readiness probe failed:
#   Get "http://172.17.1.2:8080/": dial tcp 172.17.1.2:8080: connect: connection refused
```

컨테이너는 `containerPort: 9080` 으로 떠 있는데 readinessProbe 가 `8080` 을 찌르고 있어서 연결 거부.

```yaml
ports:
  - containerPort: 9080
    protocol: TCP
readinessProbe:
  httpGet:
    path: /
    port: 8080 # ← 잘못된 포트
```

### 3. yaml 수정 — 포트 교정 + livenessProbe 추가

```bash
k get -n dev1401 pod nginx1401 -o yaml > dev.yaml
vim dev.yaml
```

```yaml
ports:
  - containerPort: 9080
    protocol: TCP
readinessProbe:
  failureThreshold: 3
  httpGet:
    path: /
    port: 9080 # 9080 으로 수정
    scheme: HTTP
  periodSeconds: 10
  successThreshold: 1
  timeoutSeconds: 1
livenessProbe:
  exec:
    command:
      - ls
      - /var/www/html/file_check
  initialDelaySeconds: 10
  periodSeconds: 60
```

### 4. 삭제 후 재적용

```bash
k delete -n dev1401 pod nginx1401
k apply -f dev.yaml
k get -n dev1401 pod nginx1401
# nginx1401   1/1   Running   0   8s
```

> 💡 `exec` 타입 probe 는 컨테이너 안에서 명령 실행 결과(exit code)로 판정 — `ls <file>` 실패 → liveness 실패 → 재시작.

---

## Case 2) CronJob — backoffLimit + activeDeadlineSeconds

> `dice` 라는 CronJob 을 매 1분마다 실행. Pod 템플릿은 `/root/throw-a-dice`. 이미지는 1~6 중 랜덤 값 반환하는 `throw-dice`(=`kodekloud/throw-dice`). non-parallel, 한 번만 완료, `backoffLimit: 25`. 20초 안에 끝내지 못하면 fail 후 종료.

### 1. 제공된 Pod 템플릿 확인

```bash
cat /root/throw-a-dice/throw-a-dice.yaml
# apiVersion: v1
# kind: Pod
# spec:
#   containers:
#     - image: kodekloud/throw-dice
#       name: throw-dice
#   restartPolicy: Never
```

### 2. CronJob 매니페스트 작성

```yaml
# cj.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: dice
spec:
  schedule: '* * * * *'
  jobTemplate:
    spec:
      parallelism: 1
      completions: 1
      backoffLimit: 25
      activeDeadlineSeconds: 20
      template:
        spec:
          containers:
            - image: kodekloud/throw-dice
              name: throw-dice
          restartPolicy: Never
```

```bash
k apply -f cj.yaml
```

> 💡 요구사항 → 필드 매핑
>
> - non-parallel + 한 번만 완료 → `parallelism: 1`, `completions: 1`
> - 25번까지 재시도 → `backoffLimit: 25`
> - 20초 안에 안 끝나면 fail → `activeDeadlineSeconds: 20` (Job 레벨)
> - 매 1분 → `schedule: "* * * * *"`

---

## Case 3) 특정 노드에 Pod 스케줄 + Secret 볼륨 read-only 마운트

> `dev2406` 네임스페이스에 `my-busybox` Pod 생성. 컨테이너 이름 `secret`, 이미지 `busybox`, 3600초 sleep. `dotfile-secret` 시크릿을 `secret-volume` 이름으로 `/etc/secret-volume` 에 **read-only** 마운트. Pod 은 반드시 `controlplane` 노드에 스케줄.

### 1. controlplane 노드 taint 확인

```bash
k describe nodes controlplane
# Taints: <none>          ← taint 없으므로 nodeSelector 만으로 가능
# Labels: kubernetes.io/hostname=controlplane
```

> 💡 taint 가 있었으면 `tolerations` 도 같이 박아야 함. 여기선 없으니 `nodeSelector` 만으로 충분.

### 2. dry-run 으로 골격 뽑기

```bash
k run my-busybox -n dev2406 --image=busybox \
  --dry-run=client -o yaml > mb.yaml
```

### 3. yaml 수정 — nodeSelector, secret volume, 컨테이너 이름/커맨드

```yaml
# mb.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: my-busybox
  name: my-busybox
  namespace: dev2406
spec:
  nodeSelector:
    kubernetes.io/hostname: controlplane
  volumes:
    - name: secret-volume
      secret:
        secretName: dotfile-secret
  containers:
    - image: busybox
      name: secret
      command:
        - sleep
        - '3600'
      volumeMounts:
        - mountPath: /etc/secret-volume
          name: secret-volume
          readOnly: true
  restartPolicy: Always
```

```bash
k apply -f mb.yaml
```

> 💡 `k run` 의 `--command -- sleep 3600` 옵션으로도 sleep 을 줄 수 있지만, 어차피 yaml 손볼 거라 `command:` 블록을 직접 박는 게 깔끔.

---

## Case 4) Ingress 가상 호스트 라우팅 + rewrite-target

> 하나의 Ingress `ingress-vh-routing` 으로 두 호스트 라우팅.
>
> - `http://watch.ecom-store.com:30093/video` → `video-service`
> - `http://apparels.ecom-store.com:30093/wear` → `apparels-service`
>
> 백엔드 경로 재작성을 위해 `nginx.ingress.kubernetes.io/rewrite-target: /` 어노테이션 필수. 30093 은 Ingress Controller 포트.

### 1. 백엔드 서비스 / IngressClass 확인

```bash
k get svc
# apparels-service   ClusterIP   ...   8080/TCP
# video-service      ClusterIP   ...   8080/TCP

k get ingressclasses.networking.k8s.io
# NAME    CONTROLLER             AGE
# nginx   k8s.io/ingress-nginx   31m
```

### 2. Ingress 매니페스트

```yaml
# ig.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-vh-routing
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: watch.ecom-store.com
      http:
        paths:
          - path: /video
            pathType: Prefix
            backend:
              service:
                name: video-service
                port:
                  number: 8080
    - host: apparels.ecom-store.com
      http:
        paths:
          - path: /wear
            pathType: Prefix
            backend:
              service:
                name: apparels-service
                port:
                  number: 8080
```

```bash
k apply -f ig.yaml
```

> 💡 `rewrite-target: /` 가 있어야 `/video/foo` 요청이 백엔드에는 `/foo` 로 들어감. 안 박으면 백엔드가 `/video/foo` 그대로 받음.
> ⚠️ Service `port.number` 는 ClusterIP Service 의 `port` 값(8080), Pod 의 containerPort 가 아님.

---

## Case 5) 특정 컨테이너 로그에서 warning 만 파일로

> `default` 네임스페이스의 Pod `dev-pod-dind-878516` 에서 컨테이너 `log-x` 의 로그를 보고, warning 만 controlplane 의 `/opt/dind-878516_logs.txt` 로 redirect.

```bash
k logs dev-pod-dind-878516 -c log-x | grep -i warning > /opt/dind-878516_logs.txt
```

> 💡 멀티 컨테이너 Pod 은 `-c <컨테이너이름>` 으로 컨테이너 지정 안 하면 에러. `grep -i` 로 대소문자 무시(`WARNING` / `Warning` / `warning` 다 잡힘).

# Mock Test 2

## Case 1) Deployment 생성 + NodePort 서비스로 expose

> Create a deployment called `my-webapp` with image: `nginx`, label `tier:frontend` and 2 replicas. Expose the deployment as a NodePort service with name `front-end-service`, port: 80 and NodePort: 30083.

### 1. Deployment 베이스 yaml 뽑고 라벨 추가

`kubectl create deployment` 에는 `--labels` 플래그가 없음 → dry-run 으로 뽑아서 `metadata.labels` 직접 추가.

```bash
kubectl create deployment my-webapp \
  --image=nginx \
  --replicas=2 \
  --dry-run=client -o yaml > deploy.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    tier: frontend # 추가
  name: my-webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-webapp
  template:
    metadata:
      labels:
        app: my-webapp
    spec:
      containers:
        - name: nginx
          image: nginx
```

### 2. NodePort Service — nodePort 직접 지정

`k expose` 는 `nodePort` 를 직접 못 박음 → dry-run 으로 base yaml 받고 `nodePort: 30083` 박기.

```bash
kubectl expose deployment my-webapp \
  --port=80 \
  --type=NodePort \
  --name=front-end-service \
  --dry-run=client -o yaml > svc.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: front-end-service
spec:
  type: NodePort
  selector:
    app: my-webapp
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30083
```

```bash
k apply -f deploy.yaml -f svc.yaml
```

```bash
k get svc front-end-service
# NAME                TYPE       CLUSTER-IP     PORT(S)        AGE
# front-end-service   NodePort   172.20.56.21   80:30083/TCP   59s
```

> 💡 `k create deployment` 와 `k expose` 모두 imperative 로 안 풀리는 옵션(`metadata.labels`, `nodePort`)이 있으면 dry-run + yaml 수정 패턴이 가장 안전.

---

## Case 2) Node Taint + Pod Toleration

> Add a taint to `node01` — `key: app_type, value: alpha, effect: NoSchedule`. Create a pod called `alpha` (image `redis`) with a toleration for that taint.

### 1. Taint 부여

```bash
kubectl taint nodes node01 app_type=alpha:NoSchedule
# node/node01 tainted
```

### 2. Pod 에 toleration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: alpha
spec:
  containers:
    - name: redis
      image: redis
  tolerations:
    - key: app_type
      operator: Equal
      value: alpha
      effect: NoSchedule
```

```bash
k apply -f alpha.yaml
# pod/alpha created
```

> 💡 toleration 의 `operator: Equal` 은 `value` 까지 일치해야 함. `operator: Exists` 는 key 만 있으면 매칭 → 이 경우 `value` 필드는 빼야 함.
> ⚠️ toleration 만 있다고 _반드시_ 그 노드에 스케줄되지는 않음 — taint 의 `NoSchedule` 을 _허용_ 해주는 것뿐, 다른 untaint 노드로도 갈 수 있음. 특정 노드 강제하려면 `nodeSelector` / `nodeAffinity` 같이 줘야 함.

---

## Case 3) NodeAffinity 로 controlplane 에만 Pod 배치

> Apply label `app_type=beta` to node `controlplane`. Create a deployment `beta-apps` (image `nginx`, 3 replicas) with `NodeAffinity` (`requiredDuringSchedulingIgnoredDuringExecution`) so all pods land on `controlplane` only.

### 1. Node 라벨링

```bash
kubectl label nodes controlplane app_type=beta
# node/controlplane labeled
```

### 2. controlplane taint 확인

```bash
k describe node controlplane | grep -i taint
# Taints: <none>
```

> 💡 보통 controlplane 에는 `node-role.kubernetes.io/control-plane:NoSchedule` taint 가 박혀 있어 일반 Pod 가 못 들어감. 이 환경은 taint 가 없어서 toleration 없이 NodeAffinity 만으로 충분.

### 3. Deployment with NodeAffinity

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: beta-apps
spec:
  replicas: 3
  selector:
    matchLabels:
      app: beta-apps
  template:
    metadata:
      labels:
        app: beta-apps
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: app_type
                    operator: In
                    values:
                      - beta
      containers:
        - name: nginx
          image: nginx
```

```bash
k apply -f beta.yaml
k get pod -l app=beta-apps -o wide
# NAME                        READY   STATUS    NODE
# beta-apps-84b9b6cff-6jdts   1/1     Running   controlplane
# beta-apps-84b9b6cff-6xmhd   1/1     Running   controlplane
# beta-apps-84b9b6cff-77rzh   1/1     Running   controlplane
```

---

## Case 4) Ingress — host + path 라우팅 + rewrite-target

> Create an Ingress for service `my-video-service` available at `http://ckad-mock-exam-solution.com:30093/video`.
>
> - annotation: `nginx.ingress.kubernetes.io/rewrite-target: /`
> - host: `ckad-mock-exam-solution.com`
> - path: `/video`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-video-service
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: ckad-mock-exam-solution.com
      http:
        paths:
          - path: /video
            pathType: Prefix
            backend:
              service:
                name: my-video-service
                port:
                  number: 8080
```

```bash
k apply -f ig.yaml
```

검증:

```bash
curl http://ckad-mock-exam-solution.com:30093/video
# <!doctype html>
# <title>Hello from Flask</title>
# ...
```

> 💡 `rewrite-target: /` 가 있어야 `/video` 요청이 백엔드에는 `/` 로 들어감 — 없으면 백엔드가 `/video` 그대로 받아서 404 나기 쉬움.
> ⚠️ `service.port.number` 는 _Service_ 의 `port` 값(8080). 백엔드 Pod 의 containerPort 가 아님 — Ingress 는 Service 를 거치니까.

---

## Case 5) 기존 Pod 에 ReadinessProbe 추가

> Pod `pod-with-rprobe` 에 `httpGet` ReadinessProbe (`path: /ready`, `port: 8080`) 를 추가.

### 1. 현재 spec 추출 + 런타임 필드 정리

```bash
k get pod pod-with-rprobe -o yaml > pr.yaml
# 에디터로 status, uid, resourceVersion, creationTimestamp, nodeName,
# serviceAccount 자동주입분, volumes(kube-api-access-*),
# volumeMounts(serviceaccount), tolerations(자동 추가분) 등 정리
```

### 2. container 에 readinessProbe 추가

```yaml
spec:
  containers:
    - name: pod-with-rprobe
      image: kodekloud/webapp-delayed-start
      env:
        - name: APP_START_DELAY
          value: '180'
      ports:
        - containerPort: 8080
          protocol: TCP
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
```

### 3. 재배포

```bash
k delete pod pod-with-rprobe
k apply -f pr.yaml
```

> ⚠️ Pod 은 probe 필드를 in-place 로 수정 불가 → 삭제 후 재생성. (Deployment 라면 `k edit` 으로 바로 가능)
> 💡 `APP_START_DELAY=180` 때문에 앱이 뜨는 데 3분 걸림. `initialDelaySeconds` 안 줘도 readinessProbe 가 알아서 fail/retry 하다 ready 되면 success 처리 — 그래서 문제에서 별도 delay 요구 안 함.

---

## Case 6) LivenessProbe — exec command

> Create a pod `nginx1401` (image `nginx`) with a `livenessProbe` that restarts the container if `ls /var/www/html/probe` fails. `initialDelaySeconds: 10`, `periodSeconds: 60`. Ignore the warnings from the probe.

```bash
k run nginx1401 --image=nginx --dry-run=client -o yaml > nginx.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx1401
spec:
  containers:
    - name: nginx1401
      image: nginx
      livenessProbe:
        exec:
          command:
            - ls
            - /var/www/html/probe
        initialDelaySeconds: 10
        periodSeconds: 60
```

```bash
k apply -f nginx.yaml
```

> 💡 `exec` probe 는 컨테이너 안에서 명령을 실행해 exit code 0 이면 success. nginx 이미지에 `/var/www/html/probe` 가 없으므로 항상 fail → 재시작 반복. 문제에서 "Ignore the warnings from the probe" 라 한 이유.

---

## Case 7) Job — completions + backoffLimit

> Create a job called `whalesay` with image `busybox` and command `echo "cowsay I am going to ace CKAD!"`. `completions: 10`, `backoffLimit: 6`, `restartPolicy: Never`.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: whalesay
spec:
  completions: 10
  backoffLimit: 6
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: busybox
          image: busybox
          command:
            - sh
          args:
            - -c
            - echo "cowsay I am going to ace CKAD!"
```

```bash
k apply -f wj.yaml
k get job whalesay
# NAME       COMPLETIONS   DURATION   AGE
# whalesay   10/10         15s        20s
```

> 💡 `completions: N` 은 Pod 가 N번 *성공*해야 Job 완료. `parallelism` 은 동시에 몇 개를 굴릴지(기본 1). `backoffLimit` 은 fail Pod 누적 재시도 한도.
> ⚠️ Job 의 `restartPolicy` 는 `Never` 또는 `OnFailure` 만 허용. `Always` 는 invalid.

---

## Case 8) 멀티 컨테이너 Pod — 컨테이너별 env

> Create pod `multi-pod` with two containers:
>
> - `jupiter` (nginx) — env `type=planet`
> - `europa` (busybox, `command: sleep 4800`) — env `type=moon`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  containers:
    - name: jupiter
      image: nginx
      env:
        - name: type
          value: planet
    - name: europa
      image: busybox
      command:
        - sleep
        - '4800'
      env:
        - name: type
          value: moon
```

```bash
k apply -f mp.yaml
```

검증:

```bash
k exec multi-pod -c jupiter -- printenv type
# planet
k exec multi-pod -c europa -- printenv type
# moon
```

> 💡 `env` 는 *컨테이너별*로 따로 정의. Pod-level 이 아님 — 같은 키도 컨테이너마다 다른 값을 줄 수 있음.
> ⚠️ busybox 는 default command 가 없어서 `command: [sleep, "4800"]` 같은 게 없으면 즉시 종료(CrashLoopBackOff) — 멀티 컨테이너 Pod 에서 한 컨테이너가 죽으면 Pod 전체가 Not Ready 됨.

---

## Case 9) PersistentVolume — hostPath + Retain

> Create a PersistentVolume `custom-volume`: `size: 50MiB`, `reclaimPolicy: Retain`, `accessModes: ReadWriteMany`, `hostPath: /opt/data`.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: custom-volume
spec:
  capacity:
    storage: 50Mi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /opt/data
```

```bash
k apply -f pv.yaml
k get pv custom-volume
# NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS
# custom-volume   50Mi       RWX            Retain           Available
```

> 💡 수동 생성 PV 의 `persistentVolumeReclaimPolicy` 기본값은 `Retain` — 안 박아도 답으로 인정되지만, 문제에 명시되어 있으면 명시적으로 적어두는 게 안전.
> ⚠️ `50Mi` 는 50 _MiB_ (binary, 1024²). `50M` 은 50 _MB_ (decimal, 1000²) — 문제가 `50MiB` 면 `Mi` 가 정답.
