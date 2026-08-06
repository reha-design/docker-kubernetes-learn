# Helm/Kustomize, CRD·Operator, 아키텍처 총정리 — 면접 대비 학습 정리

> 학습 목적: 이직/면접 대비. 지금까지는 YAML을 손으로 여러 개 만들고 각각 `kubectl apply`
> 했다면, 오늘은 그걸 패키지로 묶거나(Helm) 환경별로 변형하는(Kustomize) 도구, 그리고
> Kubernetes 자체를 확장하는 CRD/Operator 개념을 배운다. 환경: Windows 11 + Docker
> Desktop(WSL2) + minikube v1.38.1 (Kubernetes v1.35.1), Helm v4.2.3, PowerShell.

---

## 1. Helm — 여러 YAML을 패키지 하나로

### 개념

- **차트(Chart)**: Deployment/Service 같은 YAML들을 **템플릿화**해서 묶은 패키지. 고정값
  대신 `{{ .Values.replicaCount }}` 같은 변수를 써두고, 설치 시점에 값을 주입한다.
- **`values.yaml`**: 그 변수들의 기본값을 모아둔 파일. 이미지 태그, 복제본 수 같은 걸
  여기서 커스터마이징한다.
- **릴리스(Release)**: 차트를 실제로 설치한 하나의 인스턴스. 같은 차트를 이름만 다르게
  여러 번 설치할 수 있다(`dev-app`, `prod-app`처럼).
- **`helm upgrade`/`helm rollback`**: 릴리스 전체를 새 버전으로 올리거나 이전 버전으로
  되돌리는 명령. Day 7의 `kubectl rollout undo`와 개념이 같지만, **여러 YAML을 통째로**
  묶어서 처리한다는 게 다르다.

### 왜 이렇게 설계됐는가

앱 하나를 배포하는 데 보통 Deployment/Service/ConfigMap/Ingress 등 여러 YAML이 필요하고,
환경(dev/staging/prod)마다 이 값들이 조금씩 다르다. YAML을 매번 손으로 복사해 고치면
실수하기 쉽고, "지금 이 환경에 뭐가 설치돼 있는지"를 추적하기도 어렵다. Helm은 이걸
**템플릿(변하지 않는 구조) + values(환경별로 다른 값)**로 분리하고, 설치된 것을 "릴리스"라는
하나의 단위로 추적함으로써, 설치·업그레이드·롤백·삭제를 전부 하나의 명령으로 다룰 수 있게
설계됐다.

> **면접 답변**: "Helm은 여러 K8s YAML을 템플릿화해서 패키지(차트)로 묶는 도구입니다.
> values.yaml로 환경별 값을 주입하고, 설치된 하나의 단위를 릴리스라고 부릅니다. helm
> upgrade/rollback은 릴리스 전체를 새 버전으로 올리거나 되돌리는데, kubectl rollout
> undo와 마찬가지로 시간여행이 아니라 새 리비전을 만드는 방식으로 동작합니다."

### 직접 확인한 실습

**차트 생성과 템플릿 구조 확인**:
```powershell
helm create demo-chart
Get-Content demo-chart/templates/deployment.yaml | Select-String "replicas|image:"
```
```
  replicas: {{ .Values.replicaCount }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```
```powershell
Get-Content demo-chart/values.yaml | Select-String "replicaCount|repository|tag"
```
```
replicaCount: 1
  repository: nginx
  tag: ""
```
템플릿엔 `{{ .Values.xxx }}`로 자리만 비어있고, `values.yaml`에 실제 값이 있다 — 이 둘이
합쳐져서 최종 YAML이 만들어진다. (`helm create`가 만든 스캐폴드엔 `ingress.yaml`뿐 아니라
Day 10에서 배운 Gateway API의 `httproute.yaml`도 기본 포함돼 있었다.)

**설치 → 업그레이드 → 롤백 → 삭제**:
```powershell
helm install demo-release ./demo-chart
```
```
REVISION: 1
STATUS: deployed
```
```powershell
kubectl get all -l app.kubernetes.io/instance=demo-release
```
```
pod/demo-release-demo-chart-84d8f47875-rxfk4   0/1   ContainerCreating
service/demo-release-demo-chart   ClusterIP   10.109.116.42   80/TCP
deployment.apps/demo-release-demo-chart   0/1   1   0
replicaset.apps/demo-release-demo-chart-84d8f47875   1   1   0
```
릴리스명+차트명(`demo-release-demo-chart`)으로 리소스가 만들어졌다.

`values.yaml`의 `replicaCount: 1` → `3`으로 수정 후:
```powershell
helm upgrade demo-release ./demo-chart
kubectl get pods -l app.kubernetes.io/instance=demo-release
```
```
NAME                                       READY   STATUS    AGE
demo-release-demo-chart-84d8f47875-96529   1/1     Running   34s
demo-release-demo-chart-84d8f47875-frpdd   1/1     Running   35s
demo-release-demo-chart-84d8f47875-rxfk4   1/1     Running   73s
```
파드가 정확히 3개로 늘어남.

```powershell
helm history demo-release
```
```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete
2         deployed    Upgrade complete
```

```powershell
helm rollback demo-release 1
kubectl get pods -l app.kubernetes.io/instance=demo-release
helm history demo-release
```
```
NAME                                       READY   STATUS    AGE
demo-release-demo-chart-84d8f47875-96529   1/1     Running   81s
```
```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete
2         superseded  Upgrade complete
3         deployed    Rollback to 1
```
파드는 정확히 1개로 복귀했고, **`REVISION 3`이 "Rollback to 1"이라는 설명으로 새로
생겼다** — 1로 되돌아간 게 아니라 새 리비전이 만들어진 것. Day 7의 `kubectl rollout
undo`와 동일한 원리가 Helm에서도 그대로 확인됨.

```powershell
helm uninstall demo-release
kubectl get all -l app.kubernetes.io/instance=demo-release
```
```
release "demo-release" uninstalled
No resources found in default namespace.
```
한 줄로 관련 리소스가 전부 삭제됨 — 지금까지 리소스마다 따로 `kubectl delete`하던 것과
대비된다.

---

## 2. Kustomize — 템플릿 없이 base + overlay로 패치

### 개념

Helm이 "빈칸(`{{ }}`)에 값을 채워넣는" 템플릿 방식이라면, Kustomize는 **원본 YAML은
그대로 두고, 환경별로 패치(수정 메모)만 따로 얹는** 방식이다. 템플릿 문법이 아예 없고
전부 순수한 YAML이다.

- **`base/`** — 공통 원본 YAML
- **`overlays/dev/`, `overlays/prod/`** — 각 환경마다 "base에서 이것만 바꿔라"는 패치만
  담은 폴더
- **`kustomization.yaml`** — "이 리소스들을 쓰고, 이 패치를 적용해라"고 적는 설정 파일

### 왜 이렇게 설계됐는가

템플릿 언어(Go template)는 강력하지만 문법을 새로 배워야 하고, `{{ if }}`가 중첩되면
YAML도 아니고 코드도 아닌 애매한 파일이 된다. Kustomize는 "원본은 항상 유효한 순수
YAML로 유지하고, 차이만 패치로 표현"하는 쪽을 택해서 진입장벽을 낮추고, `kubectl`에
`-k` 옵션으로 직접 내장(별도 설치 불필요)될 만큼 단순하게 설계됐다. (다만 kubectl에
내장된 kustomize 버전은 standalone `kustomize` CLI보다 뒤처지는 경우가 많으므로, 최신
기능을 쓸 땐 `kubectl version`과 별개로 내장 버전을 확인해야 한다.)

> **면접 답변**: "Kustomize는 템플릿 문법 없이, 공통 base YAML에 환경별 패치를 얹어
> 다른 결과물을 만드는 도구입니다. Helm처럼 릴리스 개념이나 버전 이력은 없고, 그냥
> `kubectl apply -k`로 렌더링된 YAML을 적용할 뿐이라 git으로 직접 버전 관리합니다."

### 직접 확인한 실습

`base/deployment.yaml`(replicas: 1, `nginx:1.26`) + `overlays/dev`(패치 없음, base
그대로) + `overlays/prod`(replicas: 3, `nginx:1.27`로 패치):

```powershell
kubectl kustomize kustomize-demo/overlays/dev | Select-String "replicas|image:"
```
```
  replicas: 1
      - image: nginx:1.26
```
```powershell
kubectl kustomize kustomize-demo/overlays/prod | Select-String "replicas|image:"
```
```
  replicas: 3
      - image: nginx:1.27
```
같은 base가 overlay에 따라 다르게 렌더링됨을 확인.

```powershell
kubectl apply -k kustomize-demo/overlays/prod
kubectl get pods -l app=kustomize-demo
```
```
NAME                             READY   STATUS              AGE
kustomize-demo-994796c98-75v64   0/1     ContainerCreating   1s
kustomize-demo-994796c98-npl2c   0/1     ContainerCreating   1s
kustomize-demo-994796c98-zp9qb   0/1     ContainerCreating   1s
```
정확히 3개(prod overlay의 패치대로) 생성됨.

## 3. Helm vs Kustomize 비교

| | Helm | Kustomize |
|---|---|---|
| 방식 | 템플릿(`{{ }}`)에 값 주입 | 순수 YAML + 패치 |
| 학습 곡선 | 템플릿 문법을 새로 배워야 함 | 그냥 YAML이라 진입장벽 낮음 |
| 버전 관리 | 릴리스 단위(`helm history`, `rollback`) | 없음 — git으로 직접 관리 |
| 배포 단위 | 릴리스(설치된 인스턴스) | `kubectl apply -k` 결과물 |
| 재사용 | 공개 차트 저장소(Artifact Hub) 생태계 큼 | 프로젝트 내부 환경별 변형에 특화 |

핵심 차이 한 줄: **Helm은 "패키지를 설치하고 버전 관리"하는 도구, Kustomize는 "같은
YAML을 환경별로 살짝 다르게 찍어내는" 도구.** 실무에서는 Helm으로 설치한 뒤 Kustomize로
후처리 패치하는 식으로 같이 쓰기도 한다.

---

## 4. CRD/Operator — Kubernetes 자체를 확장하기

### 개념

- **CRD(Custom Resource Definition)** = 새로운 리소스 **종류**를 정의하는 것(명사).
  등록하면 `kubectl get`/`apply`가 내장 리소스처럼 동작하지만, **그 자체는 아무 동작도
  하지 않는다** — etcd에 저장되는 구조화된 데이터일 뿐. 예:
  ```yaml
  apiVersion: mygroup.example.com/v1
  kind: Database
  metadata:
    name: my-postgres
  spec:
    engine: postgres
    version: "15"
    storageSize: 10Gi
  ```
  이걸 apply해도 실제 Postgres는 안 뜬다 — 그냥 양식이 저장된 것뿐.
- **Operator** = 그 CRD 인스턴스를 지켜보다가 실제 작업(StatefulSet 생성, 버전 업그레이드
  등)을 대신하는 **컨트롤러**(동사). "CRD + 그걸 감시하는 커스텀 컨트롤러"의 조합을
  Operator 패턴이라 부른다.

### 왜 이렇게 설계됐는가

Deployment/ReplicaSet/StatefulSet 컨트롤러가 이미 "원하는 상태 읽기 → 실제 상태 확인 →
다르면 맞추기"라는 **Day 4의 YAML 선언형 전환과 동일한 리컨실 패턴**으로 동작하고 있다.
CRD/Operator는 그 패턴을 3rd party가 자기 도메인(DB, 인증서, 메시지 큐 등)에 대해 쓸 수
있게 일반화한 것이다.

> **용어 주의**: 내장 컨트롤러를 "K8s가 기본 내장한 Operator"라고 부르면 면접에서
> 지적당할 수 있다. 공식 정의상 **Operator는 커스텀 리소스(CRD)를 사용하는 확장**을
> 가리키고, 내장 컨트롤러는 내장 리소스를 다루므로 그냥 **컨트롤러**다. 정확히는
> "Operator는 내장 컨트롤러와 **동일한 컨트롤러 패턴**을 커스텀 리소스에 적용한 것"이라고
> 말해야 한다.

### Go 코드로 본 Reconcile 패턴 (kubebuilder 스타일, 개념 확인용 — 실행하지 않음)

```go
// Reconcile은 Database 리소스가 생성/수정되거나 주기적으로 다시 호출된다
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. "원하는 상태" 읽기 — 사용자가 YAML에 적은 것
    var db mygroupv1.Database
    if err := r.Get(ctx, req.NamespacedName, &db); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. "실제 상태" 확인 — 이 DB용 StatefulSet이 이미 떠 있는지
    var sts appsv1.StatefulSet
    err := r.Get(ctx, types.NamespacedName{Name: db.Name, Namespace: db.Namespace}, &sts)

    if errors.IsNotFound(err) {
        // 3. 없으면 spec대로 만든다
        newSts := buildStatefulSetFor(&db)
        return ctrl.Result{}, r.Create(ctx, newSts)
    }

    // 4. 있는데 버전이 다르면 업그레이드
    if sts.Spec.Template.Spec.Containers[0].Image != "postgres:"+db.Spec.Version {
        sts.Spec.Template.Spec.Containers[0].Image = "postgres:" + db.Spec.Version
        return ctrl.Result{}, r.Update(ctx, &sts)
    }

    return ctrl.Result{}, nil
}
```

이 함수는 한 번만 실행되는 게 아니라 **계속 반복 호출**된다 — Deployment 컨트롤러가
"replica 수가 맞는지 계속 확인"하는 것과 완전히 같은 방식.

### 실무 예시

- **cert-manager**의 `Certificate` CRD — 인증서 만료를 감시하다가 자동 갱신
- **CrunchyData/Percona의 DB Operator** — `PostgresCluster` CRD로 백업·장애조치·업그레이드
  를 전부 자동화 — 위 Go 코드가 실제로 하는 일과 거의 같음

> **면접 답변**: "CRD는 새로운 리소스 종류를 정의하는 것뿐이고, 실제 동작은 그 리소스를
> 감시하는 Operator(커스텀 컨트롤러)가 담당합니다. 이건 사실 Deployment 컨트롤러가
> 이미 하고 있는 '원하는 상태와 실제 상태를 비교해 맞추는' 패턴을, 3rd party가 자기
> 도메인에 대해 쓸 수 있게 확장한 것입니다."

---

## 5. 심화 학습 — CRD 단독 실습, configMapGenerator, Operator 내부 동작

앞서 배운 내용을 더 깊이 파고든 세 가지: (1) CRD 혼자서는 정말 아무 일도 안 일어나는지
직접 증명, (2) Kustomize의 `configMapGenerator`로 ConfigMap 변경 시 자동 롤아웃을
유발하는 법, (3) Operator가 변화를 어떻게 감지하는지의 내부 메커니즘.

### 5.1 CRD 단독으로는 아무 일도 안 일어난다 — 직접 증명

CRD를 "새 리소스 양식 등록"이라고 설명했는데, 말로만 하지 않고 직접 확인했다. 컨트롤러
없이 CRD와 인스턴스만 만들면 `replicas: 3`이라고 선언해도 실제로 파드가 하나도 안
생긴다는 걸 증명하는 실습이다.

`crd-demo/website-crd.yaml` (새 리소스 종류 `Website` 정의, shortName `ws`,
`spec.url`/`spec.replicas` 필드, 커스텀 컬럼 출력 포함)과 `crd-demo/website-instance.yaml`
(`url: https://example.com`, `replicas: 3`인 인스턴스 `my-blog`)을 만들어 순서대로
적용했다.

**1단계 — CRD 등록 확인**:
```powershell
kubectl apply -f crd-demo/website-crd.yaml
kubectl get crd websites.learn.example.com
```
```
customresourcedefinition.apiextensions.k8s.io/websites.learn.example.com created
NAME                         CREATED AT
websites.learn.example.com   2026-08-03T01:52:05Z
```

**2단계 — 인스턴스 생성, 내장 리소스처럼 동작하는지 확인**:
```powershell
kubectl apply -f crd-demo/website-instance.yaml
kubectl get website
kubectl get ws
```
```
website.learn.example.com/my-blog created
NAME      URL                   REPLICAS
my-blog   https://example.com   3
NAME      URL                   REPLICAS
my-blog   https://example.com   3
```
`kubectl get website`가 내장 리소스와 똑같이 동작하고, shortName `ws`도 먹히고,
CRD에 정의한 커스텀 컬럼(URL, Replicas)까지 정확히 출력된다.

**3단계, 가장 중요한 확인 — 파드가 실제로 생겼는가**:
```powershell
kubectl get pods
```
```
NAME                          READY   STATUS      RESTARTS        AGE
backend-7878cc679d-cvhsg      1/1     Running     1 (3d18h ago)   3d18h
secure-app-56b454dcc6-j2zjf   1/1     Running     1 (3d18h ago)   4d
test-client                   0/1     Completed   0               3d20h
test-client-allowed           0/1     Completed   0               3d19h
```
`my-blog`에 `replicas: 3`이라고 분명히 적었는데, 목록엔 `my-blog`나 `website`와
관련된 파드가 **하나도 없다.** 보이는 파드는 전부 Day 9/10에서 만든 것으로 이번
실습과 무관하다.

**증명된 것**: CRD 인스턴스는 etcd에 구조화된 데이터로 저장되고 `kubectl get`으로
조회까지 가능하지만, 그 자체는 아무 동작도 유발하지 않는다. `replicas: 3`이라는
값을 읽어서 실제로 파드 3개를 만드는 일은 오직 **그 리소스를 감시하는 컨트롤러
(Operator)가 있을 때만** 일어난다 — 이 데모에는 컨트롤러를 만들지 않았으므로 아무
일도 안 일어난 것이 당연한 결과다.

### 5.2 Kustomize `configMapGenerator` — 값이 바뀌면 이름도 바뀌게 만들기

**문제 상황**: ConfigMap을 고정된 이름(`app-config`)으로 두고 내용만 바꿔서 재적용하면,
**환경변수로 주입한 경우**(`env`/`envFrom`) 이미 떠 있는 Pod는 재시작되지 않는 한 새 값을
못 읽는다. 환경변수는 컨테이너 생성 시점에 한 번 확정되기 때문이다. Deployment 컨트롤러
입장에서도 "Pod 템플릿(참조하는 ConfigMap **이름**)이 안 바뀌었으니 아무것도 할 필요
없다"고 판단해서 롤아웃을 일으키지 않는다.

> **주의 — 볼륨 마운트는 다르다**: ConfigMap을 **볼륨으로 마운트**한 경우는 kubelet이
> 주기적으로 동기화해서 **파일 내용이 자동으로 갱신된다**(공식 문서의 "Mounted ConfigMaps
> are updated automatically"). 갱신까지는 kubelet sync 주기 + 캐시 TTL만큼 지연이 있고,
> `subPath`로 마운트한 파일과 `immutable: true`인 ConfigMap은 예외로 갱신되지 않는다.
> 즉 "ConfigMap 변경은 Pod에 전혀 전파되지 않는다"는 건 **환경변수 주입에만 해당하는
> 이야기**다. 다만 파일이 갱신돼도 애플리케이션이 그 파일을 다시 읽어야(watch/reload)
> 실제로 반영되므로, 리로드 기능이 없는 앱이면 결국 롤아웃이 필요하다.
> (출처: kubernetes.io — Configure a Pod to Use a ConfigMap)

아래 실습은 `envFrom`(환경변수 주입)을 쓴 케이스라, 롤아웃을 강제할 필요가 있는 상황이다.

**해결책**: `configMapGenerator`는 ConfigMap을 **내용 기반 해시가 붙은 이름**으로
생성하고, 그 ConfigMap을 참조하는 모든 리소스(Deployment의 `envFrom` 등)의 참조
이름도 **자동으로 같은 해시 이름으로 재작성**한다. 내용이 바뀌면 해시가 바뀌고,
해시가 바뀌면 참조 이름이 바뀌고, 참조 이름이 바뀌면 Pod 템플릿이 바뀐 것으로
간주되어 정상적으로 롤링 업데이트가 일어난다.

> **왜 이렇게 설계됐는가**: ConfigMap 값이 바뀌었다고 해서 그 값을 쓰는 Pod를
> 자동으로 재시작해주는 메커니즘은 Kubernetes에 없다(볼륨 마운트 파일은 갱신되지만
> 프로세스 재시작이나 롤아웃은 일어나지 않는다). `configMapGenerator`는 이 문제를
> 새 메커니즘을 추가하는 대신, 기존에 이미 존재하는 "Pod 템플릿이 바뀌면 롤링
> 업데이트한다"는 Deployment 컨트롤러의 동작을 그대로 활용해서 우회한다 — 이름을
> 바꿔서 컨트롤러가 "이건 다른 리소스다"라고 판단하게 만드는 영리한 트릭이다.

`kustomize-demo/base/deployment.yaml`에 `app-config`를 참조하는 `envFrom`을
추가하고, `kustomize-demo/overlays/withconfig/kustomization.yaml`에
`configMapGenerator`로 `GREETING=hello-v1` 리터럴을 넣었다.

**1단계 — 렌더링 결과로 해시 이름 확인**:
```powershell
kubectl kustomize kustomize-demo/overlays/withconfig | Select-String "name: app-config|GREETING"
```
```
  GREETING: hello-v1
  name: app-config-tm2b4ck8d2
            name: app-config-tm2b4ck8d2
```
ConfigMap 이름과 Deployment의 `envFrom.configMapRef.name`이 똑같은 해시로 맞춰져
렌더링된다.

**2단계 — 실제 적용**:
```powershell
kubectl apply -k kustomize-demo/overlays/withconfig
kubectl get configmap
kubectl get deployment kustomize-demo -o jsonpath="{.spec.template.spec.containers[0].envFrom}"
```
```
configmap/app-config-tm2b4ck8d2 created
deployment.apps/kustomize-demo created
NAME                    DATA   AGE
app-config-tm2b4ck8d2   1      1s
kube-root-ca.crt        1      4d6h
[{"configMapRef":{"name":"app-config-tm2b4ck8d2"}}]
```

**3단계 — 값 변경 후 재적용, 롤아웃이 실제로 일어나는지 확인**:
`GREETING`을 `hello-v1` → `hello-v2`로 바꾸고 재적용:
```powershell
kubectl get replicaset -l app=kustomize-demo
kubectl apply -k kustomize-demo/overlays/withconfig
kubectl get configmap
kubectl get replicaset -l app=kustomize-demo
```
```
NAME                        DESIRED   CURRENT   READY   AGE
kustomize-demo-84bc85b764   1         1         1       39s

configmap/app-config-kg222654fm created
deployment.apps/kustomize-demo configured

NAME                    DATA   AGE
app-config-kg222654fm   1      0s
app-config-tm2b4ck8d2   1      40s
kube-root-ca.crt        1      4d6h

NAME                        DESIRED   CURRENT   READY   AGE
kustomize-demo-5d59c55865   1         1         1       2s
kustomize-demo-84bc85b764   0         0         0       42s
```
해시가 `tm2b4ck8d2` → `kg222654fm`로 바뀌었고, **새 ReplicaSet**(`5d59c55865`)이
생기고 기존 ReplicaSet(`84bc85b764`)은 0으로 스케일 다운됐다 — ConfigMap 내용
변경이 실제 롤링 업데이트로 이어지는 전체 체인이 확인됐다.

**실무 주의점**: 기존 `app-config-tm2b4ck8d2`는 자동으로 삭제되지 않고 그대로
남는다. `kubectl apply -k`만으로는 고아가 된 이전 리소스를 정리(prune)해주지 않는다.
정리 방법은 두 가지인데 둘 다 그냥 `--prune`만 붙이면 되는 게 아니다:

- **라벨 기반(기존 방식)** — `kubectl apply -k ... --prune -l <셀렉터>`처럼 **라벨
  셀렉터(또는 `--all`)를 반드시 함께** 줘야 한다. 게다가 kustomize가 생성한 ConfigMap에
  공통 라벨이 없으면 셀렉터에 걸리지 않으므로, `kustomization.yaml`에 `labels`
  (구 `commonLabels`)로 공통 라벨을 먼저 붙여둬야 동작한다. 이 방식은 server-side
  apply와 함께 쓸 수 없고 오탐/누락 이슈가 알려져 있다.
- **ApplySet 기반(신규 방식)** — `KUBECTL_APPLYSET=true` 환경변수 + `--prune
  --applyset=<이름>`. kubectl v1.27에서 도입됐고 **아직 alpha**라 프로덕션 의존은
  이르다(KEP-3659).

실무에서는 이래서 정리까지 자동으로 해주는 Argo CD / Flux 같은 GitOps 컨트롤러나,
릴리스 단위로 리소스를 추적하는 Helm에 맡기는 경우가 많다.

> **면접 답변**: "`configMapGenerator`는 ConfigMap 이름에 내용 기반 해시를 붙이고,
> 그 ConfigMap을 참조하는 리소스의 이름도 자동으로 재작성합니다. ConfigMap 값이
> 바뀌면 해시가 바뀌고, 참조 이름이 바뀌니 Pod 템플릿이 바뀐 것으로 간주되어
> Deployment 컨트롤러가 정상적으로 롤링 업데이트를 수행합니다. 참고로 ConfigMap을
> 볼륨으로 마운트한 경우엔 kubelet이 파일을 자동 갱신해주지만 롤아웃은 일어나지
> 않고, 환경변수로 주입하면 아예 갱신되지 않기 때문에 이런 해시 트릭이 필요합니다.
> 다만 이전 ConfigMap은 자동 삭제되지 않아서, 라벨 셀렉터를 붙인 `--prune`이나
> GitOps 도구 없이는 고아 리소스가 쌓입니다."

### 5.3 Operator는 변화를 어떻게 "알아채는가" — List+Watch, Informer, Work Queue (이론)

CRD 인스턴스를 만들거나 수정했을 때 Operator가 그 변화를 아는 방법은 **폴링이
아니라 API 서버가 실시간으로 밀어주는(push) 구독 방식**이다. Day 2에서 본
kube-proxy가 Service/EndpointSlice 변화를 감지하는 것과 같은 client-go
**informer 패턴**이다. (이 항목은 이론 확인 목적이라 실습 없이 개념만 정리한다.)

1. **Reflector** — `LIST`로 현재 전체 목록과 `resourceVersion`을 가져온 뒤,
   `WATCH ...?resourceVersion=N`으로 그 시점 이후의
   변화만 스트리밍받는다. 이 Watch는 끊기지 않고 계속 열려있는 HTTP 연결이다.
   연결이 끊기면 마지막 resourceVersion부터 재구독하고, 그 리비전이 etcd
   압축(compaction)으로 이미 사라졌으면 `410 Gone`을 받아 처음부터 다시 LIST한다.
   (`resourceVersion`은 내부적으로 etcd 리비전에 대응하지만, 공식 API 규약상
   **불투명(opaque) 문자열**이다 — 클라이언트가 숫자로 파싱하거나 크기를 비교해선
   안 되고, 서버에 되돌려주는 토큰으로만 써야 한다.)
2. **DeltaFIFO** — Reflector가 받은 이벤트(Added/Updated/Deleted/Sync)를 객체
   키별로 델타 목록에 **순서대로 누적**하는 큐. 흔히 "같은 객체의 이벤트가 합쳐진다"고
   설명되지만, 실제 client-go의 `dedupDeltas`는 **연속된 Deleted 델타 두 개**만
   하나로 합친다(re-list와 watch가 같은 삭제를 중복 전달하는 경우 대비). Added나
   Updated는 합쳐지지 않고 그대로 쌓인다. 중복 제거가 본격적으로 일어나는 곳은
   4번의 Work Queue다.
3. **Indexer(로컬 캐시)** — DeltaFIFO에서 꺼낸 이벤트를 스레드 세이프한 메모리
   캐시에 반영한다. Reconcile 로직은 API 서버에 매번 GET하지 않고 이 캐시를
   읽는다 — API 서버/etcd 부하를 줄이기 위한 설계다. 여러 컨트롤러가 같은
   리소스를 감시할 때도 **SharedInformer**로 캐시를 공유해 Watch 연결을 하나만
   유지한다.
4. **Work Queue** — 캐시 갱신 후 이벤트 핸들러가 **객체 이름(namespace/name)만**
   큐에 넣는다(변경 내용 자체는 안 넣음). 이 큐는 집합(set) 성질이 있어 같은
   키의 중복 이벤트는 하나로 합쳐지고, 에러 발생 시 지수 백오프로 재시도한다.
5. **Reconcile(namespace, name)** — 워커가 큐에서 이름을 꺼내 호출한다. "무엇이
   바뀌었는지"가 아니라 **원하는 상태와 실제 상태를 처음부터 다시 비교**하는
   레벨 기반(level-triggered) 방식이라, 이벤트를 하나 놓쳐도 다음 트리거 때
   자기 치유된다.
6. **Resync** — 일정 주기로 로컬 캐시의 모든 객체에 대해 update 알림을 다시
   전달해서 Reconcile을 재실행시킨다. client-go 주석대로 "authoritative storage와의
   상호작용을 추가하지 않는다" — **API 서버에 다시 묻는 게 아니라 이미 가진 캐시를
   그대로 재생(replay)**한다.
   그래서 resync는 **"watch 이벤트를 놓친 경우"를 고쳐주지 못한다**. 이벤트를
   놓쳤다면 캐시 자체가 틀린 상태이고, resync는 그 틀린 캐시를 다시 전달할 뿐이기
   때문이다. 놓친 이벤트에 대한 실제 안전망은 **watch 재연결 / `410 Gone` 후 재LIST /
   watch bookmark**가 담당한다. resync가 실제로 막아주는 건 *Reconcile이 실패했거나,
   컨트롤러가 관리하는 외부 실제 상태가 K8s 밖에서 변경(드리프트)된 경우*다.

```
API 서버 (etcd)
   │ LIST(resourceVersion 획득) → WATCH(그 시점부터 스트림)
   ▼
Reflector
   │ Added/Updated/Deleted + 키
   ▼
DeltaFIFO → Indexer(로컬 캐시) ── 주기적 Resync(캐시 재생, API 서버 조회 없음)
   │ 이벤트 핸들러 트리거
   ▼
Work Queue (키만, 중복 제거, 재시도 백오프)
   │
   ▼
Reconcile(namespace, name) → 원하는 상태 vs 실제 상태 비교 → 필요한 만큼만 쓰기
```

쓰기 요청에도 낙관적 동시성 제어가 걸려서, 요청에 담긴 `resourceVersion`이
서버의 현재 값과 다르면(그 사이 다른 주체가 수정) `409 Conflict`가 나고
Reconcile이 재시도된다 — Day 7 롤아웃에서 본 것과 같은 낙관적 락 패턴이다.

> **왜 이렇게 설계됐는가**: 이벤트 기반으로 "무엇이 바뀌었는지"를 정밀 추적하면
> 이벤트를 하나라도 놓쳤을 때 상태가 영원히 어긋날 위험이 있다. 레벨 기반으로
> "그냥 통째로 다시 비교"하면 이벤트를 놓쳐도 다음 트리거(또는 Resync) 때
> 알아서 맞춰지는 자기 치유 특성을 가진다. Deployment/ReplicaSet 등 K8s 내장
> 컨트롤러 전체가 이 원칙을 따른다.

> **면접 답변**: "Operator는 API 서버에 List+Watch로 리소스를 구독합니다. Watch는
> 열린 HTTP 스트림으로 변화를 실시간으로 받고, 받은 변화는 로컬 캐시(Indexer)를
> 갱신한 뒤 리소스 이름만 work queue에 넣습니다. 워커가 큐를 소비하며 Reconcile을
> 호출하는데, 무엇이 바뀌었는지가 아니라 원하는 상태와 실제 상태를 처음부터
> 다시 비교하는 레벨 기반 방식이라 이벤트를 놓쳐도 자기 치유가 됩니다. kube-proxy가
> Service/EndpointSlice 변화를 감지하는 것과 동일한 client-go informer 패턴입니다."

---

## 면접 준비 관점 — 메커니즘 질문 vs 설계 질문

인프라 면접이 "답이 정해져 있다"고 느껴지기 쉬운데, 정확히는 **도메인(인프라 vs 백엔드)이
아니라 질문 유형(메커니즘 vs 설계)이 정답의 고정 여부를 가른다.**

- **메커니즘 질문** (예: "Deployment와 StatefulSet의 차이는?", "taint의 NoExecute는 뭐가
  다른가?") — Kubernetes가 실제로 그렇게 동작하므로 정답이 하나다. 백엔드의 "ACID가
  뭐야?"와 같은 성격 — **워크북 recall 방식으로 준비.**
- **설계/트레이드오프 질문** (예: "Helm vs Kustomize 중 뭘 쓰겠는가", "멀티테넌트 클러스터의
  RBAC을 어떻게 설계하겠는가") — 정답 암기가 아니라 **판단 근거를 설명하는 게 중요**하다.
  백엔드의 "레이트 리미터를 어떻게 설계하겠는가"와 같은 성격.
- 인프라 면접이 유독 "고정적"으로 느껴졌던 건 도메인 차이가 아니라, **CKA 커리큘럼
  자체가 메커니즘 검증에 무게를 두기 때문**일 가능성이 크다. 실제로는 인프라도 설계형
  질문이 나온다 — 다음 "모의 면접"에서 이 두 유형을 구분해서 준비한다.
