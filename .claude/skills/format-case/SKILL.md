---
name: format-case
description: Format rough CKA/CKAD exam case notes into the project's standard README.md style and append them. Use when the user provides scratch notes plus a case identifier (e.g. "case 8", "Lightning Lab case 9") and wants them formatted and added to /Users/joseonghyeon/personal/cka-ckad-sheet/README.md.
---

# CKA/CKAD 케이스 정리

## 목적

사용자가 CKA/CKAD 기출 문제를 풀고 남긴 **거친 메모**를 받아, 이 레포 `README.md`의 기존 케이스(Case 1~7 등)와 **동일한 톤·구조**로 다듬어서 적절한 위치에 추가합니다.

## 사용자가 줄 입력

대략 다음과 같은 형태로 들어옵니다:

- 기출 문제 모음 이름 (예: `Lightning Lab`, `Mock Exam 1`) — 새 H1 섹션이거나 기존 섹션에 추가
- 케이스 번호 (예: `case 8`)
- 문제 원문 (영문 quote, 있을 수도 없을 수도)
- 사용자가 끼적인 풀이 메모 (명령어, 짧은 코멘트, 헷갈렸던 부분 등)

문제 번호와 모음 이름은 명시 안 될 수도 있음. 그럴 땐 `README.md`의 마지막 케이스 다음 번호로 자동 추정하고, 마지막 H1 섹션에 이어 붙입니다.

## 실행 순서

1. **`README.md` 먼저 읽기** — 어디까지 작성됐는지, 다음 케이스 번호가 몇 번인지, 어떤 H1 섹션(`# Lightning Lab` 등)에 들어갈지 파악.
2. **이미 같은 번호가 있는지 확인** — 있으면 덮어쓸지 추가할지 사용자에게 확인하지 말고, 한 번 알려준 뒤 새 번호로 진행(시스템 리마인더상 멈추지 말 것).
3. **메모를 아래 "포맷 규칙"대로 다듬기**.
4. **`README.md` 적절한 위치에 Edit으로 삽입** — 같은 H1 섹션의 마지막 케이스 뒤. 케이스 사이는 `---` 구분선.
5. 작성된 섹션의 핵심 변경만 1~2문장 요약으로 보고.

## 포맷 규칙 (기존 Case들 스타일 그대로)

### 1. 헤더

```markdown
## Case N) [짧고 명확한 한국어 제목]
```

- 제목은 "무엇을 하는 문제인지" 한 줄로. 예: `Deployment 커스텀 컬럼으로 출력`, `ETCD 스냅샷 백업`.

### 2. 문제 원문이 있으면 blockquote

```markdown
> Take the backup of ETCD at the location `/opt/etcd-backup.db` ...
```

- 영어 그대로. 줄바꿈 안 건드림.

### 3. 본문 구조

- `###` 으로 단계 나누기. 자주 쓰이는 패턴:
  - `### 상황 파악` — 현재 상태/제약 확인
  - `### 1. ~~~`, `### 2. ~~~` — 실제 풀이 순서
  - `### 정리: ~~ 체크리스트` — 마지막에 핵심 요약 (선택)
- 명령어는 항상 ` ```bash ` 코드블록, 매니페스트는 ` ```yaml `.
- `k`(=`kubectl`) 알리아스를 자연스럽게 섞어 씀. 단, `kubectl run`, `kubectl create deployment` 같이 첫 등장이나 강조해야 할 때는 풀네임.
- 코드블록 안에서 짧은 `#` 주석으로 의도 설명 OK.

### 4. 표·콜아웃 패턴

- 옵션·파라미터 비교, PV/PVC 스펙 매칭 등 **여러 값을 대조**할 때는 마크다운 테이블:

  ```markdown
  | 항목 | PV  | PVC |
  | ---- | --- | --- |
  ```

- 팁/인사이트는 `> 💡 ...` blockquote.
- 함정/주의사항은 `> ⚠️ ...` blockquote.
- 둘 다 한 줄로 끝나도 되고, 여러 줄 이어져도 됨.

### 5. "정리:" 체크리스트

문제 풀이가 끝난 뒤 다음 번에도 써먹을 만한 일반화 가능한 포인트가 있으면 마지막에 추가:

```markdown
### 정리: PVC가 Pending일 때 체크리스트

1. `kubectl describe pvc <name>` → 이벤트에서 원인 확인
2. ...
```

- 단순 문제(예: 컬럼 커스텀 출력)에는 생략 가능.

### 6. 톤

- 한국어, 반말체 (`~함`, `~됨`, `~안 됨`).
- 사용자가 실수했던 부분이 메모에 있으면 ⚠️로 보존 — 다음번 같은 함정을 피하는 게 핵심이라.
- 너무 풀어쓰지 않음. 짧고 정보 밀도 높게.

### 7. 케이스 간 구분

```markdown
---

## Case (N+1)) ...
```

- 마지막 케이스 뒤에는 `---` 안 붙임 (사용자가 다음 케이스 추가할 때 알아서 붙음).

## 메모를 다듬을 때 신경 쓸 것

- **명령어 누락 보강**: 메모에 `madison`만 있고 `apt update`가 없으면, 실제 동작에 필요한 선행 명령은 보충.
- **순서 재배치**: 사용자가 시행착오 순으로 적었어도, 정리된 풀이는 "맞는 순서"로 재구성. 단, 사용자가 빠진 시행착오는 ⚠️/💡로 살려둠 (다음에 같은 실수 안 하려고).
- **추측 금지**: 메모에 없는 명령/플래그를 임의로 만들지 않음. 모르는 부분은 `<...>` 플레이스홀더 또는 메모 그대로 둠.
- **번역하지 않음**: 영문 문제는 영문 그대로. 명령어/리소스 이름도 그대로.

## 예시 입출력

**사용자 입력 (대충 끼적임):**

```
case 8 - service expose
nginx pod 있음. 80포트 expose 하래.
근데 ClusterIP는 안되고 NodePort로.
nodePort: 30080
selector 어떻게 줘야하지 보고 했음. pod label run=nginx 였음.

k expose pod nginx --port=80 --type=NodePort --name=nginx-svc 했더니 nodePort 랜덤이라
yaml 뽑아서 nodePort: 30080 박았음.
```

**스킬이 만들어내는 출력 (README.md에 삽입될 형태):**

````markdown
---

## Case 8) Pod을 NodePort 서비스로 노출

> 실행 중인 `nginx` Pod을 80 포트에서 `NodePort` 타입 서비스로 노출. `nodePort`는 `30080`.

### 1. 기존 Pod 라벨 확인

서비스의 `selector`는 Pod의 라벨과 일치해야 함.

```bash
k get pod nginx --show-labels
# run=nginx
```

### 2. expose로 베이스 manifest 뽑기

`expose` 명령은 `nodePort`를 직접 지정할 수 없어서 일단 dry-run으로 yaml만 받고 수정.

```bash
k expose pod nginx \
  --port=80 \
  --type=NodePort \
  --name=nginx-svc \
  --dry-run=client -o yaml > svc.yaml
```

### 3. nodePort 직접 지정

```yaml
spec:
  type: NodePort
  selector:
    run: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

```bash
k apply -f svc.yaml
```

> 💡 `k expose`만으로는 `nodePort`를 못 박음 — 항상 yaml 뽑아서 수정하는 패턴이 안전.
````

> 위 예시에서 바깥쪽이 백틱 4개(` ```` `)인 이유: 안쪽에 들어가는 ` ```bash `/` ```yaml `(백틱 3개) 펜스가 깨지지 않도록 감싸는 트릭. 실제 `README.md`에 들어갈 때는 바깥 펜스 없이 안쪽 내용만 들어감.

## 마지막에 사용자에게 보고할 것

- 어느 H1 섹션 아래, 몇 번 케이스로 추가했는지.
- 메모에서 추측·보강한 부분이 있으면 1~2줄로 짚어주기 (사용자가 검수할 수 있게).
