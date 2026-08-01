# 트러블슈팅 집중 — describe/logs/events 워크플로 — 면접 대비 학습 정리

> 학습 목적: 이직/면접 대비. CKA 커리큘럼에서 가장 비중이 큰 도메인(30%)이 트러블슈팅이다.
> [Day 3](day03-hpa-scheduling.md)의 Pending/OOMKilled, [Day 5](day05-docker-basics-review.md)의
> ErrImagePull에 이어, 오늘은 CrashLoopBackOff·ImagePullBackOff·Service Selector 불일치·
> 멀티 컨테이너 로그까지 실제로 장애를 재현하고 `describe`/`logs`/`events` 표준 절차로
> 진단하는 근육을 만드는 데 집중. 특히 세션 시작 시 클러스터에 떠 있던 **실제(연출 아닌)
> 장애 2건**을 먼저 진단했다. 환경: minikube(Windows + Docker driver, WSL2), PowerShell.

---

## 0. 실전 사례 — 클러스터 재시작 직후의 자가 치유 (연출 아닌 실제 장애)

### 상황

`minikube start` 직후 `kubectl get pods -A`를 확인했더니 두 컴포넌트가 비정상이었다:
```
kube-system   coredns-7d764666f9-gq4f9   0/1   Running   4 (37s ago)
kube-system   storage-provisioner        0/1   Error     8 (17d ago)
```

### 진단 절차 — `describe` → `logs --previous`

`storage-provisioner`를 `describe`한 핵심 부분:
```
State:          Running
  Started:      Sat, 01 Aug 2026 11:59:42 +0900
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
  Started:      Sat, 01 Aug 2026 11:59:10 +0900
  Finished:     Sat, 01 Aug 2026 11:59:31 +0900
Ready:          True
Restart Count:  9
...
Events:
  Normal   SandboxChanged  83s  kubelet  Pod sandbox changed, it will be killed and re-created.
  Warning  BackOff         60s  kubelet  Back-off restarting failed container storage-provisioner ...
  Normal   Pulled/Created/Started  50s~82s  kubelet  (재생성)
```

직전에 죽은 인스턴스의 로그(`--previous`)로 정확한 원인 확인:
```powershell
PS> kubectl logs storage-provisioner -n kube-system --previous
I0801 02:59:10 Initializing the minikube storage provisioner...
F0801 02:59:31 error getting server version: Get "https://10.96.0.1:443/version?timeout=32s": dial tcp 10.96.0.1:443: connect: connection refused
```

`coredns`도 `describe`로 같은 패턴 확인: `Last State: Terminated, Exit Code: 255`,
Events에 `Unhealthy ... statuscode: 503`(readiness probe 실패)이 재부팅 시점에 몰려있었다.

### 결과 해석

- `10.96.0.1:443`은 기본 `kubernetes` Service(API 서버로 가는 창구)의 ClusterIP.
- minikube가 막 재시작된 시점에 storage-provisioner가 API 서버에 연결을 시도했는데
  아직 준비되지 않아 `connection refused` → **Fatal 로그(`F` 접두사)를 남기고 즉시 종료**.
- coredns도 자신의 readiness 엔드포인트가 아직 안정화되지 않아 503을 반환하다가, 안정화 후
  통과.
- **재시작 시점(클러스터 부팅 직후)에 몰려있고, 그 이후 반복이 없다면 "컴포넌트 간 준비
  순서 경쟁(race condition)에 의한 일시적 실패 → self-healing"으로 판단할 수 있다.**
  `Restart Count`가 커 보여도(9회) 지금 `Ready: True`라면 심각한 장애가 아니다.

> **면접 답변**: "재시작 횟수가 많은 Pod를 어떻게 진단하나요?"
> "`Restart Count`만으로 심각도를 판단하지 않고, 먼저 현재 `Ready` 상태를 확인합니다.
> 그다음 `describe`의 `Last State`와 `Events`에서 재시작 시점의 패턴(예: 특정 시간대에
> 몰려있는지)을 보고, `logs --previous`로 죽기 직전의 실제 에러를 확인합니다. 클러스터 재시작
> 직후 의존 컴포넌트(API 서버)가 아직 준비되지 않아 생기는 일시적 충돌이고 지금 Ready
> 상태이며 이후 반복이 없다면, 정상적인 self-healing으로 판단합니다."

---

## 1. CrashLoopBackOff — 지수 백오프의 실측

### 개념

컨테이너가 시작하자마자 비정상 종료(exit code ≠ 0)를 반복할 때, kubelet은 **재시작 간격을
점점 늘려가며(지수 백오프)** 재시도한다: `10s → 20s → 40s → 80s → 160s → 300s(상한)`, 이후
300초로 고정. Job의 `backoffLimit`과 달리 **재시도 자체를 포기하지 않는다**.

### 왜 이렇게 설계됐는가

간격 없이 계속 재시작하면 노드 자원(CPU, 이미지 pull 대역폭)과 API 서버에 부담을 준다.
반대로 재시도를 아예 멈추면 일시적 장애(방금 0번 사례처럼)로 인한 실패였을 경우 영영 복구가
안 된다. **점점 느리게, 하지만 계속** 재시도하는 절충안이 지수 백오프다.

> **면접 답변**: "CrashLoopBackOff는 어떻게 동작하나요?"
> "컨테이너가 계속 비정상 종료될 때, kubelet이 재시작 간격을 10초부터 시작해 실패할 때마다
> 2배씩 늘리다가 최대 5분에서 상한을 겁니다. 무한정 빠르게 재시도해서 자원을 낭비하지
> 않으면서도, Job의 backoffLimit과 달리 재시도 자체를 포기하지는 않습니다."

### 직접 확인한 실습

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: crash-demo
spec:
  containers:
    - name: crasher
      image: busybox
      command: ["sh", "-c", "echo 'about to crash'; exit 1"]
```

`kubectl get pod crash-demo -w`로 관찰한 `CrashLoopBackOff` 재진입 시점(AGE):
```
7s → 48s → 100s → 184s → 270s → 414s → 720s → 1020s → 1320s → 1620s
```
간격: `41s → 52s → 84s → 86s → 144s → 306s → 300s → 300s → 300s`

**결과 해석**: 초반엔 대기 시간이 계속 늘어나다가(약 2배씩) 마지막엔 정확히 **300초(5분)에서
고정**됐다. 공식 문서상의 `10s→20s→40s→80s→160s→300s(상한)` 알고리즘이 실측으로 그대로
확인됐다.

`describe`의 Events에도 누적 카운트로 확인됨:
```
Warning  BackOff    63s (x32 over 31m)  kubelet  Back-off restarting failed container crasher ...
Normal   Pulling    11s (x12 over 31m)  kubelet  Pulling image "busybox"
```

**부가 발견**: `Pulling image "busybox"`가 재시작마다(12회) 찍혔다. `image: busybox`처럼
태그를 생략하면 암묵적으로 `:latest`가 되고, `:latest` 태그의 기본 `imagePullPolicy`는
`Always`라서 매 재시작마다 레지스트리에 다시 확인하러 간다 ([Day 7](day07-docker-k8s-bridge.md)의
imagePullPolicy 내용과 연결). 로컬 캐시 덕분에 실제로는 1.5초 내외로 빠르게 끝난다.

`kubectl logs crash-demo`로 마지막 크래시 시점의 출력(`about to crash`) 확인.

---

## 2. ImagePullBackOff — "컨테이너가 아예 시작도 못한" 경우

### 개념

CrashLoopBackOff와 달리, 이미지 자체를 가져오지 못해 **컨테이너가 한 번도 시작되지 않은**
상태. `RESTARTS`가 계속 `0`으로 유지되는 것이 핵심 구분점 — 재시작이라는 개념 자체가 적용될
수 없다(시작한 적이 없으므로).

```
스케줄링 실패        이미지 준비 실패           실행 중 실패
   (Pending)      (ErrImagePull/BackOff)   (CrashLoopBackOff/OOMKilled)
   Node: <none>    Node는 있음, 컨테이너 시작 전   컨테이너 시작됨, Restart Count 증가
```

### 왜 이렇게 설계됐는가

파드 시작은 **스케줄링 → 이미지 pull → 컨테이너 시작**의 3단계이고, 각 단계의 실패는 서로
다른 원인·해법을 갖는다. 이 계층을 구분해두면 트러블슈팅 시 "어디서 막혔는지"만 알아도
확인할 곳이 좁혀진다.

> **면접 답변**: "ImagePullBackOff와 CrashLoopBackOff를 어떻게 구분하나요?"
> "가장 빠른 구분은 `RESTARTS` 카운트입니다. ImagePullBackOff는 컨테이너가 한 번도 시작되지
> 못했으므로 항상 0이고, CrashLoopBackOff는 컨테이너가 시작됐다가 죽었으므로 카운트가
> 올라갑니다. `describe pod`의 State도 ImagePullBackOff는 `Waiting`, CrashLoopBackOff는
> `Terminated`(Last State)로 다르게 나타납니다."

### 직접 확인한 실습

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pull-demo
spec:
  containers:
    - name: puller
      image: nginx:this-tag-does-not-exist
```

```
PS> kubectl get pod pull-demo -w
pull-demo   0/1   ErrImagePull       0   3s
pull-demo   0/1   ImagePullBackOff   0   16s
...(RESTARTS 계속 0)

PS> kubectl describe pod pull-demo
Status:           Pending
Node:             minikube/192.168.49.2      ← 스케줄링은 성공
Containers:
  puller:
    State:          Waiting
      Reason:       ImagePullBackOff
    Restart Count:  0
Events:
  Warning  Failed  ...  Failed to pull image "nginx:this-tag-does-not-exist":
                        Error response from daemon: manifest for nginx:this-tag-does-not-exist
                        not found: manifest unknown: manifest unknown
```

**중요한 발견 — `Pending`이라는 Pod phase의 함정**: 이 Pod의 `Status`는 `Pending`이지만
`Node`는 정상 배정되어 있다. [Day 3](day03-hpa-scheduling.md)에서 본 스케줄링 실패
Pending(`Node: <none>`)과 겉보기 `Status`는 똑같이 `Pending`인데 원인이 완전히 다르다.
**`Pending`은 "아직 스케줄 안 됨"과 "스케줄은 됐지만 컨테이너가 아직 안 돎" 둘 다를
포괄하는 상위 상태**이므로, `Status` 필드만 보고 판단하지 말고 반드시 `Node` 필드와
Events를 같이 봐야 한다.

| | Day 3의 Pending | ImagePullBackOff |
|---|---|---|
| Node 배정 | `<none>` (배치 자체 실패) | 배정됨 (배치는 성공) |
| 막힌 단계 | 1단계: 스케줄링 | 2단계: 이미지 준비 |
| 원인 확인 | `FailedScheduling` 이벤트 | `Failed`(manifest not found) 이벤트 |

> **면접 답변**: "ImagePullBackOff와 Pending(스케줄링 실패)을 어떻게 구분하나요?"
> "둘 다 Pod phase는 Pending으로 보일 수 있지만, `describe pod`의 Node 필드로 구분합니다.
> 스케줄링 실패는 Node가 `<none>`이고 Events에 FailedScheduling이 찍히지만,
> ImagePullBackOff는 Node가 정상 배정되어 있고 대신 이미지 pull 관련 Failed 이벤트가
> 찍힙니다."

---

## 3. Service/Selector 불일치 — 에러 메시지 없이 조용히 실패하는 장애

### 개념

Service의 `selector`와 Pod의 실제 `labels`가 어긋나면, Deployment도 Service도 겉보기엔
전부 정상인데 트래픽만 전달되지 않는다. **어떤 리소스도 에러 상태를 보이지 않는다**는 게
가장 까다로운 지점 — 유일한 단서는 `Endpoints`(또는 EndpointSlice)가 비어있다는 것뿐이다.

### 왜 이렇게 설계됐는가

Service는 label selector로 대상 Pod 집합을 동적으로 계산하는 방식이라([Day 2](day02-kubernetes-basics.md)),
selector와 label이 정확히 일치해야만 연결이 성립한다. 오타 하나만으로도 "전혀 다른 대상 집합"을
가리키게 되며, 이는 Kubernetes API 입장에서는 유효한 설정이라 에러가 아니다 — 그저 "그 라벨을
가진 Pod가 0개"일 뿐이다.

> **면접 답변**: "Service로 Pod에 접속이 안 될 때 어떻게 진단하나요?"
> "먼저 `kubectl get endpoints <service>`로 Service가 실제로 어떤 Pod들을 바라보고 있는지
> 확인합니다. 비어있다면 `describe svc`의 Selector와 `get pods --show-labels`의 실제 라벨을
> 비교합니다. Pod 자체는 Running이라 상태 확인만으로는 절대 못 잡아내는 유형이라, Service
> 트러블슈팅에서 가장 먼저 봐야 할 명령입니다."

### 직접 확인한 실습

Deployment의 Pod 라벨은 `app: web-app`인데, Service의 selector를 일부러
`app: web-app-typo`로 어긋나게 설정:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels: { app: web-app }
  template:
    metadata:
      labels: { app: web-app }
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc
spec:
  selector:
    app: web-app-typo    # 오타 상황 재현
  ports:
    - port: 80
      targetPort: 80
```

```
PS> kubectl get pods -l app=web-app
web-app-7bbff855cc-ft5mq   1/1   Running   ← Pod는 완전히 정상
web-app-7bbff855cc-tlvj4   1/1   Running

PS> kubectl get endpoints web-app-svc
NAME          ENDPOINTS   AGE
web-app-svc   <none>      9m5s          ← 그런데 연결 대상이 없음

PS> kubectl describe svc web-app-svc
Selector:                 app=web-app-typo    ← Pod 라벨(app=web-app)과 불일치
Endpoints:
Events:                   <none>              ← 에러 이벤트 자체가 없음
```

Selector를 수정해서 Pod 라벨과 일치시키자 즉시 복구:
```powershell
PS> kubectl patch service web-app-svc -p '{\"spec\":{\"selector\":{\"app\":\"web-app\"}}}'
PS> kubectl get endpoints web-app-svc
web-app-svc   10.244.0.51:80,10.244.0.52:80   11m
```

**결과 해석**: Selector 수정 하나로 endpoints가 즉시 채워졌다. Deployment 상태
(`2/2 Running`), Service 생성 여부, Events 전부 정상으로 보였던 상황에서 유일한 이상 신호는
`Endpoints: <none>`뿐이었다.

---

## 4. 멀티 컨테이너 Pod의 로그 스트림 — `-c` 옵션

### 개념

Pod 안에 컨테이너가 여러 개(메인 + 사이드카)면 로그도 컨테이너별로 완전히 분리되어 있다.
`kubectl logs <pod>`만 실행하면 최신 kubectl은 에러 대신 **첫 번째 컨테이너를 자동으로
선택**하고 어떤 컨테이너인지 경고로 알려준다. 다른 컨테이너를 보려면 `-c <컨테이너명>`.

### 왜 이렇게 설계됐는가

사이드카 패턴([Day 8](day08-scheduling.md))에서 메인 컨테이너와 사이드카는 서로 다른
프로세스이자 서로 다른 로그 스트림이다. 장애가 어느 쪽 컨테이너에서 발생했는지 구분해야
정확히 진단할 수 있으므로, 로그도 컨테이너 단위로 격리된다.

> **면접 답변**: "멀티 컨테이너 Pod의 로그는 어떻게 확인하나요?"
> "`kubectl logs <pod> -c <container>`로 컨테이너를 명시합니다. 지정하지 않으면 kubectl이
> 자동으로 첫 번째 컨테이너를 선택하고 어떤 컨테이너인지 경고로 알려줍니다. 사이드카
> 패턴에서는 메인 컨테이너와 사이드카의 로그가 완전히 분리되어 있으므로, 장애 원인이 어느
> 쪽인지 파악하려면 반드시 `-c`로 각각 확인해야 합니다."

### 직접 확인한 실습

```
PS> kubectl logs multi-log-demo
Defaulted container "app" out of: app, sidecar
app-log 05:28:59
app-log 05:29:04
...

PS> kubectl logs multi-log-demo -c sidecar
sidecar-log 05:29:00
sidecar-log 05:29:05
...
```

각 컨테이너의 로그가 독립된 타임스탬프 흐름으로 완전히 분리되어 나왔다. `Defaulted
container "app" out of: app, sidecar` 메시지로 어떤 컨테이너가 자동 선택됐는지 명시적으로
알려준다.

---

## 5. 표준 트러블슈팅 워크플로 — 오늘 다룬 5가지 장애 유형 통합

| 장애 유형 | 1차 확인 | 핵심 단서 | 원인 계층 |
|---|---|---|---|
| Pending ([Day 3](day03-hpa-scheduling.md)) | `describe pod` → Node | `Node: <none>` | 스케줄링 (requests) |
| ImagePullBackOff | `get pod` → RESTARTS | `RESTARTS: 0` 고정 | 이미지 준비 |
| CrashLoopBackOff | `logs --previous` | Exit Code, 반복 주기 | 실행 중 (앱 자체) |
| OOMKilled ([Day 3](day03-hpa-scheduling.md)) | `describe pod` → Last State | `Reason: OOMKilled` | 실행 중 (limits) |
| Service 연결 안 됨 | `get endpoints` | `<none>` | 네트워킹 (selector) |

**공통 절차**: `kubectl get pods` (전체 스캔) → `kubectl describe pod` (State/Last
State/Events) → `kubectl logs [--previous] [-c 컨테이너]` (실제 에러 메시지) → 필요시
`kubectl get endpoints`(네트워킹 계열이면).

**가장 중요한 판단 습관**: `Restart Count`나 상태 이름만 보고 심각도를 판단하지 않는다.
**"지금 Ready인가"**, **"Node가 배정됐는가"**, **"언제 재시작이 몰려있는가"** 세 가지를
먼저 확인하면 대부분의 장애 유형이 좁혀진다.

---

## 오늘 배운 것 전체 흐름 요약

1. 클러스터 재시작 직후의 컴포넌트 재시작(storage-provisioner, coredns)은 API 서버 준비
   순서와의 경쟁 때문이며, `Ready: True`로 자가 치유됐다면 심각한 장애가 아니다 —
   Restart Count보다 현재 Ready 상태를 먼저 봐야 한다
2. CrashLoopBackOff의 재시작 간격은 10s→20s→40s→80s→160s→300s(상한)의 지수 백오프이며,
   실측으로 정확히 이 패턴이 확인됐다. Job의 backoffLimit과 달리 재시도를 포기하지 않는다
3. ImagePullBackOff는 컨테이너가 한 번도 시작되지 못한 상태로 RESTARTS가 항상 0이며,
   Pod phase가 똑같이 Pending이어도 Node 필드로 스케줄링 실패와 구분해야 한다
4. Service/Selector 불일치는 모든 리소스가 정상으로 보이는 채로 조용히 실패하며, 유일한
   단서는 `Endpoints`가 비어있다는 것뿐이다 — Service 트러블슈팅은 항상 endpoints부터 확인
5. 멀티 컨테이너 Pod의 로그는 `-c`로 컨테이너별로 분리해서 봐야 하며, 최신 kubectl은
   생략 시 첫 컨테이너를 자동 선택하고 경고로 알려준다

**다음 학습 목표**: Day 12 — Helm/Kustomize 실습 + CRD/Operator 개념 + 아키텍처 총정리,
모의 면접.
