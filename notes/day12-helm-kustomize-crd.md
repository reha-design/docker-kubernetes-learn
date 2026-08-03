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
