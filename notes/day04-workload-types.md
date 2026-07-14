# 워크로드 타입: PV/PVC/StorageClass, StatefulSet, Job/CronJob, DaemonSet — 면접 대비 학습 정리

> 학습 목적: 이직/면접 대비. [Day 3](day03-hpa-scheduling.md)까지는 주로 `kubectl run`/`create`
> 같은 명령형(imperative) 방식과 HPA/스케줄링을 다뤘다. Day 4부터는 YAML 매니페스트 기반
> 선언형(declarative) 워크플로(`kubectl apply -f`)로 전환하면서, 상태를 갖는
> 워크로드(StatefulSet), 배치성 워크로드(Job/CronJob), 노드별 데몬(DaemonSet), 그리고 이들을
> 뒷받침하는 스토리지 추상화(PersistentVolume/PVC/StorageClass)를 minikube에서 직접
> 실습하며 정리. 환경은 Day 1~3과 동일한 minikube(Windows + Docker driver, WSL2), PowerShell.

---

## 0. YAML 선언형 전환 (`kubectl apply`)

### 개념

- `kubectl create`/`run`: "이 리소스를 만들어라" — 1회성 실행, 이미 존재하면 에러.
- `kubectl apply -f`: "클러스터 상태가 이 YAML과 같아지게 해라" — 리소스가 없으면 생성하고,
  있으면 현재 상태와 diff를 계산해 필요한 부분만 patch한다. 같은 명령을 반복 실행해도 안전
  (idempotent).

### 왜 이렇게 설계됐는가

GitOps/CI-CD 파이프라인에서는 "이 YAML이 원하는 상태(desired state)"라는 걸 코드로
버전관리하고, 클러스터는 그 상태로 계속 수렴(reconcile)하도록 만드는 것이 목표다. 명령형
방식은 사람이 매번 "무엇을 할지" 순서대로 지시해야 하지만, 선언형은 "무엇을 원하는지"만
선언하면 컨트롤러가 알아서 현재 상태와 비교해 맞춰준다.

> **면접 답변**: "`kubectl create`와 `apply`의 차이는 무엇인가요?"
> "`create`는 명령형으로, 리소스를 1회성으로 생성하고 이미 존재하면 에러가 납니다. `apply`는
> 선언형으로, YAML에 정의된 desired state와 클러스터의 현재 상태를 비교해 diff만 반영하기
> 때문에 반복 실행해도 안전합니다. GitOps 파이프라인은 이 선언형 특성 덕분에 YAML을 버전관리
> 대상으로 삼고 클러스터를 그 상태로 계속 수렴시킬 수 있습니다."

---

## 1. PersistentVolume / PersistentVolumeClaim / StorageClass

### 개념

- **PV(PersistentVolume)**: 실제 스토리지 자원 (관리자가 미리 만들거나, StorageClass가 동적
  으로 프로비저닝).
- **PVC(PersistentVolumeClaim)**: 사용자가 "이런 크기/성능의 스토리지가 필요하다"고 요청하는
  것. PVC가 PV와 매칭(바인딩)된다.
- **StorageClass**: PV를 어떻게 동적으로 만들지에 대한 템플릿(provisioner, 파라미터 등).

### 왜 이렇게 설계됐는가

컨테이너의 파일시스템은 Pod가 죽으면(재시작/재스케줄) 함께 사라지는 임시(ephemeral)
자원이다. DB처럼 상태를 유지해야 하는 워크로드는 Pod 생명주기와 분리된 스토리지가 필요하다.
PV/PVC로 "스토리지 요청(PVC)"과 "실제 스토리지 구현(PV)"을 분리하면, 개발자는 인프라
세부사항(어떤 클라우드 디스크인지, NFS인지 등)을 몰라도 되고, 관리자는 StorageClass로
정책만 관리하면 된다.

> **면접 답변**: "PV/PVC/StorageClass의 관계를 설명해주세요."
> "PV는 실제 스토리지 리소스, PVC는 그에 대한 사용자의 요청서입니다. StorageClass는 PVC
> 요청이 들어왔을 때 PV를 동적으로 만들어주는 프로비저닝 템플릿 역할을 합니다. 이렇게
> 분리해두면 애플리케이션은 스토리지의 물리적 구현을 몰라도 되고, 인프라 팀은 StorageClass
> 정책만 바꿔서 스토리지 백엔드를 교체할 수 있습니다."

### 직접 확인한 실습

기본 StorageClass 확인:
```powershell
PS C:\Users\user\k8s-practice> kubectl get storageclass
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false                  2d20h
```
- `PROVISIONER: k8s.io/minikube-hostpath`: minikube 노드의 로컬 디스크(hostPath)를 동적으로
  볼륨으로 만들어주는 방식.
- `RECLAIMPOLICY: Delete`: PVC가 삭제되면 실제 볼륨 데이터도 함께 삭제됨.
- `VOLUMEBINDINGMODE: Immediate`: PVC 생성 즉시 바로 볼륨을 프로비저닝 (vs
  `WaitForFirstConsumer`: Pod가 스케줄될 때까지 바인딩을 미뤄, 멀티노드 환경에서 노드 위치를
  고려함).

PVC 생성:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Pod에 마운트:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-test-pod
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["sh", "-c", "echo 'hello from pvc' > /data/test.txt && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: test-pvc
```

적용 및 확인:
```powershell
PS C:\Users\user\k8s-practice> kubectl apply -f pvc.yaml
persistentvolumeclaim/test-pvc created
PS C:\Users\user\k8s-practice> kubectl apply -f pod-with-pvc.yaml
pod/pvc-test-pod created
PS C:\Users\user\k8s-practice> kubectl get pvc
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-pvc   Bound    pvc-0cabc724-a759-442a-8ec6-18968ab386aa   1Gi        RWO            standard       0s
PS C:\Users\user\k8s-practice> kubectl exec pvc-test-pod -- cat /data/test.txt
hello from pvc
```

**지속성(persistence) 검증** — Pod를 삭제하고 재생성해도 같은 PV가 재사용되는지 확인:
```powershell
PS C:\Users\user\k8s-practice> kubectl delete pod pvc-test-pod
pod "pvc-test-pod" deleted from default namespace
PS C:\Users\user\k8s-practice> kubectl apply -f pod-with-pvc.yaml
pod/pvc-test-pod configured
PS C:\Users\user\k8s-practice> kubectl get pvc test-pvc
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-pvc   Bound    pvc-0cabc724-a759-442a-8ec6-18968ab386aa   1Gi        RWO            standard       2m14s
```

**결과 해석**: `VOLUME` 이름(`pvc-0cabc724-...`)이 Pod 재생성 전후로 완전히 동일했다. Pod를
삭제해도 PVC와 그 안의 PV는 그대로 유지되며, 새 Pod가 같은 PVC를 참조하면 동일한 스토리지를
다시 마운트한다는 것을 실증했다.

**멀티노드 제약 (문서로 확인)**: 기본 `standard` StorageClass(`k8s.io/minikube-hostpath`)는
멀티노드 클러스터에서는 지원되지 않는다 — hostPath는 볼륨이 물리적으로 존재하는 노드에 Pod가
스케줄되어야만 동작하는데, 기본 프로비저너는 이 노드 인식을 못하기 때문이다. 멀티노드에서
동적 프로비저닝을 쓰려면 `csi-hostpath-driver` addon으로 바꿔야 한다 (minikube 공식 문서,
2026-07 확인). Day 6 멀티노드 실습 시 다시 짚을 예정.

---

## 2. StatefulSet

### 개념

Deployment는 Pod들이 서로 완전히 동일하고 대체 가능(interchangeable)하다고 가정한다.
DB 클러스터처럼 "각 인스턴스가 고유한 정체성과 자기 전용 데이터"를 가져야 하는 경우엔
StatefulSet을 쓴다.

**Deployment와 핵심 차이:**
- **안정적인 Pod 이름**: `<이름>-0`, `<이름>-1`, `<이름>-2`처럼 순번이 붙고, 재시작해도 이름이
  바뀌지 않음 (Deployment는 매번 랜덤 해시가 붙은 새 이름).
- **각 Pod 전용 PVC**: `volumeClaimTemplates`로 Pod마다 별도의 PVC가 자동 생성됨.
- **순차적 생성/삭제**: Pod-0이 Running+Ready가 된 후에야 Pod-1 생성 (스케일다운도 역순).
- **안정적인 네트워크 식별자**: Headless Service와 함께 쓰면
  `<pod이름>.<서비스이름>.<네임스페이스>.svc.cluster.local`로 각 Pod에 고정 DNS가 부여됨.

### 왜 이렇게 설계됐는가

분산 DB(MySQL 복제, Kafka, Zookeeper, Elasticsearch 등)는 "어떤 노드가 프라이머리/리더인지",
"각 노드가 자기 데이터를 잃지 않는지"가 중요하다. Pod 이름과 볼륨이 고정되어 있어야 재시작
후에도 "나는 pod-0이고, 내 데이터는 여기"라는 정체성을 유지할 수 있다. 실무에서는 순번을
Kafka의 `broker.id`나 Zookeeper 쿼럼 멤버 식별처럼 노드 역할 구분에 직접 활용한다.

> **면접 답변**: "StatefulSet은 왜 필요한가요?"
> "Deployment는 Pod를 무상태(stateless)로 취급해서 아무 Pod나 대체 가능하지만, StatefulSet은
> 각 Pod에 고유한 이름·전용 볼륨·순차적 생성 순서를 보장합니다. 그래서 리더 선출이 필요하거나
> 노드마다 별도 데이터를 유지해야 하는 분산 시스템(DB, 메시지 큐)에 적합합니다. Kafka의
> broker.id, Zookeeper의 쿼럼 멤버 찾기가 대표적인 활용 사례입니다."

### 직접 확인한 실습

Headless Service (`clusterIP: None` — 개별 Pod DNS를 위한 전제조건):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None
  selector:
    app: nginx-sts
  ports:
    - port: 80
```

StatefulSet (replicas 3, PVC 템플릿 포함):
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: nginx-headless
  replicas: 3
  selector:
    matchLabels:
      app: nginx-sts
  template:
    metadata:
      labels:
        app: nginx-sts
    spec:
      containers:
        - name: nginx
          image: nginx
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

적용 결과 — 순차 생성 확인:
```
PS C:\Users\user\k8s-practice> kubectl get pods -w
NAME    READY   STATUS              RESTARTS   AGE
web-0   0/1     ContainerCreating   0          1s
web-0   1/1     Running             0          11s
web-1   0/1     Pending             0          0s
web-1   0/1     ContainerCreating   0          1s
web-1   1/1     Running             0          4s
web-2   0/1     Pending             0          0s
web-2   0/1     ContainerCreating   0          1s
web-2   1/1     Running             0          3s
```
`web-0`이 Running+Ready가 된 뒤에야 `web-1`이 생성되고, `web-1`이 끝난 뒤 `web-2`가 생성되는
순차 패턴이 그대로 확인됐다.

DNS 확인:
```powershell
PS C:\Users\user\k8s-practice> kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup web-0.nginx-headless.default.svc.cluster.local
Server:         10.96.0.10
Address:        10.96.0.10:53

Name:   web-0.nginx-headless.default.svc.cluster.local
Address: 10.244.0.6
```
`get pods -o wide`로 확인한 `web-0`의 실제 Pod IP(`10.244.0.6`)와 정확히 일치했다.

**Pod별 전용 PVC 확인** (Day 6 세션 시작 시 정리 과정에서 확인한 출력):
```
NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/test-pvc    Bound    pvc-0cabc724-a759-442a-8ec6-18968ab386aa   1Gi        RWO            standard       2d3h
persistentvolumeclaim/www-web-0   Bound    pvc-199e5c00-c68f-444d-ab2a-2cbc3142ff61   1Gi        RWO            standard       2d3h
persistentvolumeclaim/www-web-1   Bound    pvc-6e26d8ef-018f-475e-9cd1-5287a9cec92d   1Gi        RWO            standard       2d3h
persistentvolumeclaim/www-web-2   Bound    pvc-7b91f872-e399-4b7c-a750-5c2c327352ef   1Gi        RWO            standard       2d3h
```
`volumeClaimTemplates`의 이름(`www`) + Pod 이름(`web-0/1/2`) 조합으로 **Pod마다 별도의
PVC**(`www-web-0`, `www-web-1`, `www-web-2`)가 자동 생성됐고, 각각 서로 다른 PV에 바인딩됐다.
Deployment에서 Pod들이 볼륨을 공유하는 것과 달리, StatefulSet의 각 Pod는 자기 전용 스토리지를
갖는다는 것이 실제 리소스로 확인된 것.

**정리 시 주의**: StatefulSet을 삭제해도 이 PVC들은 **의도적으로 남는다** — 실수로 StatefulSet을
지웠다 다시 만들어도 데이터가 유지되도록 한 안전장치. 완전히 정리하려면
`kubectl delete pvc www-web-0 www-web-1 www-web-2`처럼 명시적으로 삭제해야 한다.

**트러블슈팅**: 짧은 이름(`web-0.nginx-headless`)으로 `nslookup`을 시도했을 때는
`NXDOMAIN`이 떴다. 이는 클러스터 DNS 문제가 아니라 busybox의 musl libc 기반 `nslookup`이
resolv.conf의 search domain을 제대로 못 타는 known limitation 때문이었다. 전체
FQDN(`....default.svc.cluster.local`)으로 재시도하니 정상 해석됐다.

> **면접 답변**: "Headless Service는 무엇이고 왜 필요한가요?"
> "`clusterIP: None`으로 설정한 Service로, 단일 클러스터IP로 로드밸런싱하는 대신 DNS 조회 시
> 개별 Pod들의 IP를 직접 반환합니다. StatefulSet과 함께 쓰면
> `<Pod이름>.<서비스이름>.<네임스페이스>.svc.cluster.local` 형태의 고정 DNS가 각 Pod에
> 부여되어, Pod가 재시작돼 IP가 바뀌어도 이름으로 다시 찾을 수 있습니다. 분산 시스템의 피어
> 디스커버리에 활용됩니다."

---

## 3. Job / CronJob

### 개념

Deployment/StatefulSet은 "계속 떠 있어야 하는" 워크로드를 위한 것이지만, Job은 **"완료가
목적"**인 워크로드다. 배치 처리, 데이터 마이그레이션, 1회성 스크립트 실행 등에 쓴다.

- `completions`: 성공적으로 완료해야 하는 Pod 개수.
- `parallelism`: 동시에 몇 개까지 병렬 실행할지.
- `backoffLimit`: 실패 시 재시도 최대 횟수.
- `restartPolicy`는 `Never` 또는 `OnFailure`만 가능 (Deployment의 `Always`와 충돌하는 개념).

**CronJob**은 Job을 cron 스케줄(`분 시 일 월 요일`)에 따라 주기적으로 생성해주는 상위
리소스다. CronJob → Job → Pod의 3단 계층 구조로, 스케줄마다 새 Job을 만들기 때문에 실행
이력을 개별적으로 추적할 수 있다. `concurrencyPolicy`(Allow/Forbid/Replace)로 이전 실행이
안 끝난 채 다음 스케줄이 오는 경우를 제어하고, `successfulJobsHistoryLimit`/
`failedJobsHistoryLimit`로 오래된 Job 기록을 자동 정리한다.

### 왜 이렇게 설계됐는가

Deployment의 컨트롤러는 "Pod가 항상 N개 떠 있어야 한다"고 가정해 Pod가 정상 종료(exit 0)해도
계속 재시작시킨다. 배치 작업은 "끝나면 끝난 것"이어야 하므로 별도의 컨트롤러(Job)가
필요하다. CronJob이 Job을 재사용하지 않고 매번 새로 만드는 이유는, 각 실행 기록을 독립된
리소스로 남겨 실행 이력·로그·성공/실패 여부를 개별 추적하기 위함이다.

> **면접 답변**: "Job과 Deployment의 차이는 무엇인가요?"
> "Deployment는 Pod가 항상 실행 중이어야 한다고 가정하는 반면, Job은 Pod가 성공적으로
> 종료(exit 0)하는 것 자체가 목표입니다. 그래서 배치 처리나 1회성 마이그레이션 작업에는
> Job/CronJob을 씁니다. CronJob은 cron 스케줄마다 새 Job을 생성하는 CronJob → Job → Pod의
> 3단 계층 구조로, `successfulJobsHistoryLimit` 등으로 오래된 실행 기록을 자동 정리합니다."

### 직접 확인한 실습

**Job** (`completions: 3`, `parallelism: 2`):
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  completions: 3
  parallelism: 2
  backoffLimit: 2
  template:
    spec:
      containers:
        - name: hello
          image: busybox
          command: ["sh", "-c", "echo Hello from $HOSTNAME && sleep 5"]
      restartPolicy: Never
```

```
PS C:\Users\user\k8s-practice> kubectl get pods -w
hello-job-s4wt9   ContainerCreating -> Running -> Completed   (1번째, s5m6w와 동시 시작)
hello-job-s5m6w   ContainerCreating -> Running -> Completed   (2번째)
hello-job-x5csk   Pending -> ContainerCreating -> Running -> Completed   (3번째, 앞 2개 중 하나가
                                                                          끝난 뒤 시작)
PS C:\Users\user\k8s-practice> kubectl get job hello-job
NAME        STATUS     COMPLETIONS   DURATION   AGE
hello-job   Complete   3/3           21s        13m
```
`parallelism: 2`대로 처음 2개(`s4wt9`, `s5m6w`)가 동시에 뜨고, 하나가 끝나자 `completions: 3`을
채우기 위해 세 번째(`x5csk`)가 생성됐다. `kubectl get job`은 개별 Pod가 아니라 Job 컨트롤러
관점의 전체 목표 달성 여부(완료 개수, 소요 시간)를 요약해서 보여준다.

**CronJob** (매 1분마다 실행):
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: hello
              image: busybox
              command: ["sh", "-c", "echo Hello from CronJob at $(date)"]
          restartPolicy: OnFailure
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```

```
PS C:\Users\user\k8s-practice> kubectl get jobs
NAME                  STATUS     COMPLETIONS   DURATION   AGE
hello-cron-29729375   Complete   1/1           5s         3m2s
hello-cron-29729376   Complete   1/1           5s         2m2s
hello-cron-29729377   Complete   1/1           5s         62s
hello-cron-29729378   Running    0/1           2s         2s

# 잠시 후 (29729378 완료 시점)
PS C:\Users\user\k8s-practice> kubectl get jobs
NAME                  STATUS     COMPLETIONS   DURATION   AGE
hello-cron-29729376   Complete   1/1           5s         2m40s
hello-cron-29729377   Complete   1/1           5s         100s
hello-cron-29729378   Complete   1/1           5s         40s
```
`successfulJobsHistoryLimit: 3` 설정대로, 4번째(`29729378`)가 완료되자 가장 오래된
`29729375`가 자동으로 사라지고 최근 3개만 남았다. Job 이름 뒤의 숫자(`29729375` 등)는
유닉스 타임스탬프를 60(초)으로 나눈 값으로, 각 스케줄 실행을 구분하는 고유 ID다.

로그 확인:
```powershell
PS C:\Users\user\k8s-practice> kubectl logs -l job-name=hello-cron-29729377
Hello from CronJob at Sat Jul 11 09:37:02 UTC 2026
```

정리 (계속 켜두면 계속 Job이 쌓이므로 실습 후 삭제):
```powershell
kubectl delete -f cronjob.yaml
```

---

## 4. DaemonSet

### 개념

"클러스터의 **모든(또는 특정 라벨의) 노드마다 정확히 Pod 1개씩**" 배치하는 컨트롤러다.
Deployment는 "총 N개"를 관리하지만, DaemonSet은 "노드당 1개"를 관리한다 — 노드가 추가되면
자동으로 그 노드에도 Pod가 생기고, 노드가 제거되면 그 Pod도 함께 사라진다.

**대표 사용 사례**: 로그 수집기(Fluentd, Filebeat), 노드 모니터링 에이전트(Prometheus Node
Exporter), CNI/스토리지 드라이버 등 노드 인프라 컴포넌트.

### 왜 이렇게 설계됐는가

로그 수집이나 모니터링은 "몇 개가 떠 있는지"가 아니라 "**모든 노드를 빠짐없이 커버**"하는
게 목적이다. Deployment로 구현하면 노드 수가 바뀔 때마다 replicas를 수동으로 맞춰야 하지만,
DaemonSet은 노드 수 변화에 스케줄러 개입 없이 자동으로 반응한다.

> **면접 답변**: "DaemonSet은 Deployment와 어떻게 다른가요?"
> "Deployment는 원하는 개수(replicas)를 유지하지만, DaemonSet은 클러스터의 각 노드마다
> 정확히 1개의 Pod를 유지합니다. 로그 수집기나 모니터링 에이전트처럼 모든 노드를 빠짐없이
> 커버해야 하는 인프라성 워크로드에 적합하며, 노드가 늘어나거나 줄어들면 자동으로 Pod 수가
> 따라갑니다."

### 직접 확인한 실습

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
spec:
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      containers:
        - name: agent
          image: busybox
          command: ["sh", "-c", "while true; do echo alive on $NODE_NAME; sleep 30; done"]
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
```

```powershell
PS C:\Users\user\k8s-practice> kubectl get nodes
NAME       STATUS   ROLES           AGE    VERSION
minikube   Ready    control-plane   3d1h   v1.35.1

PS C:\Users\user\k8s-practice> kubectl get daemonset node-agent
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
node-agent   1         1         1       1            1           <none>          89m
```
노드가 `minikube` 1개뿐이라 `DESIRED`도 정확히 1로 계산됐다. `DESIRED`는 Deployment의
`replicas`처럼 사용자가 직접 정하는 숫자가 아니라, 조건에 맞는 노드 개수를 보고 컨트롤러가
자동으로 계산한 값이라는 게 핵심 차이다. minikube는 `minikube start --nodes N`(1.10.1+)으로
멀티노드 클러스터를 만들 수 있으므로, Day 6에서 노드를 2개로 늘리면 `DESIRED`도 2로 자동
증가하는 걸 다시 확인할 예정이다.

---

## 오늘 배운 것 전체 흐름 요약

1. Day 4부터 `kubectl create`/`run` 명령형 대신 YAML + `kubectl apply -f` 선언형 워크플로로
   전환했다 — desired state를 코드로 관리하고 클러스터가 그 상태로 수렴하도록 하는 방식.
2. PV/PVC/StorageClass는 "스토리지 요청(PVC)"과 "실제 구현(PV)"을 분리해, Pod 생명주기와
   무관하게 데이터를 유지한다. Pod를 삭제·재생성해도 같은 PV(같은 볼륨 이름)가 재사용됨을
   실습으로 확인했다. 단, 기본 hostPath 프로비저너는 멀티노드를 지원하지 않는다.
3. StatefulSet은 고정된 Pod 이름·전용 PVC·순차 생성/삭제로 각 Pod에 고유한 정체성을 부여해,
   리더 선출이나 노드별 데이터 분리가 필요한 분산 시스템에 쓰인다. Headless Service와 결합해
   Pod별 고정 DNS까지 확인했다.
4. Job은 "완료가 목적"인 워크로드로 `completions`/`parallelism`으로 배치 처리를 제어하고,
   CronJob은 이를 스케줄에 따라 반복 생성하며 history limit으로 오래된 기록을 자동 정리한다.
5. DaemonSet은 "노드당 정확히 1개"를 유지하는 컨트롤러로, replicas가 아닌 노드 개수에
   연동되어 로그 수집·모니터링 같은 노드 인프라 워크로드에 쓰인다.

**다음 학습 목표**: liveness/readiness/startup probe + 배포 전략(rolling update 파라미터,
rollback, blue-green/canary) + QoS 클래스.
