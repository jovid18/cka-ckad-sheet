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
