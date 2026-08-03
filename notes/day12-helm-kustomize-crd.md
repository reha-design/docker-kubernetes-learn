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
`-k` 옵션으로 직접 내장(별도 설치 불필요)될 만큼 단순하게 설계됐다.

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

Deployment/ReplicaSet/StatefulSet 컨트롤러 자체가 이미 K8s가 기본 내장한 Operator다.
CRD/Operator는 "원하는 상태 읽기 → 실제 상태 확인 → 다르면 맞추기"라는 **Day 4의 YAML
선언형 전환과 동일한 리컨실 패턴**을, 3rd party가 자기 도메인(DB, 인증서, 메시지 큐 등)에
대해 쓸 수 있게 일반화한 것이다.

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
