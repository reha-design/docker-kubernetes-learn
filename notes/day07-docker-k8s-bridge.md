# Docker→K8s 브리지 — 이미지 유통 경로와 ImagePullBackOff — 면접 대비 학습 정리

> 학습 목적: 이직/면접 대비. [Day 6](day06-dockerfile-networking.md)까지 Docker(빌드)와
> Day 1~4의 K8s(실행)를 각각 배웠다면, 오늘은 그 둘을 **연결**하는 날. 직접 빌드한 이미지를
> minikube에 배포하면서 "내 PC에서 빌드한 이미지를 쿠버네티스는 어떻게 아는가"라는 질문에
> 부딪히고, ImagePullBackOff를 일부러 체험한 뒤 고쳤다. ENTRYPOINT vs CMD와 K8s
> command/args의 대응 관계도 정리. 환경: Windows 11 + Docker Desktop(WSL2) +
> minikube v1.38.1 (Kubernetes v1.35.1, Docker 29.2.1 런타임), PowerShell.
> 실습 폴더: `C:\Users\user\docker-practice` (레포 외부).

---

## 1. ENTRYPOINT vs CMD — 그리고 K8s의 command/args

### 개념

둘 다 "컨테이너 시작 시 실행할 명령"을 정하지만 역할이 다르다.

- **ENTRYPOINT** = 고정된 실행 본체 (컨테이너의 "정체성"). `docker run <이미지> <인자>`로
  넘긴 인자가 **뒤에 붙는다**.
- **CMD** = 기본값. ENTRYPOINT가 있으면 "기본 인자"가 되고, `docker run`에서 인자를 주면
  **통째로 교체**된다.

| 실행 방법 | ENTRYPOINT | CMD |
|---|---|---|
| `docker run 이미지` | 실행됨 | 기본 인자로 붙음 |
| `docker run 이미지 <인자>` | 실행됨 | `<인자>`로 교체됨 |
| `docker run --entrypoint <명령> 이미지` | `<명령>`으로 교체됨 | 무시됨 |

### 왜 이렇게 설계됐는가

"이 컨테이너는 무엇을 하는 컨테이너인가"(ENTRYPOINT)와 "기본 설정은 무엇인가"(CMD)를
분리하기 위해서다. 예를 들어 ENTRYPOINT를 `["redis-server"]`로 고정하고 CMD로 기본 설정
파일만 지정해두면, 사용자는 `docker run redis /custom.conf`처럼 **인자만 바꿔서** 같은
컨테이너를 다르게 실행할 수 있다. 본체까지 바꾸려면 `--entrypoint`라는 명시적 스위치를
써야 하므로 "실수로 정체성을 바꾸는" 사고를 막는다.

**K8s와의 대응 관계** (면접 단골 함정):

| Docker | K8s Pod 스펙 | 이름이 헷갈리는 이유 |
|---|---|---|
| ENTRYPOINT | `command:` | K8s의 command는 Docker의 CMD가 **아니라** ENTRYPOINT를 덮어쓴다 |
| CMD | `args:` | Docker의 CMD를 덮어쓰는 건 K8s의 args |

> **면접 답변**: "ENTRYPOINT와 CMD의 차이는?"
> "ENTRYPOINT는 컨테이너의 고정 실행 본체이고 CMD는 그 기본 인자입니다. docker run에서
> 인자를 넘기면 CMD만 교체되고 ENTRYPOINT는 유지되며, 본체를 바꾸려면 --entrypoint
> 플래그를 명시해야 합니다. 쿠버네티스에서는 command 필드가 ENTRYPOINT를, args 필드가
> CMD를 덮어쓰는데, 이름 때문에 command가 CMD를 덮어쓴다고 착각하기 쉬운 부분입니다."

### 직접 확인한 실습

3줄짜리 `Dockerfile.ep`로 빌드:
```dockerfile
FROM alpine:3.20
ENTRYPOINT ["echo", "hello"]
CMD ["world"]
```

```
PS> docker run --rm ep-test
hello world
PS> docker run --rm ep-test kubernetes
hello kubernetes
PS> docker run --rm --entrypoint hostname ep-test
e6fc57d64bb5
```

**결과 해석**: ①은 ENTRYPOINT(`echo hello`) + CMD(`world`)가 합쳐져 `hello world`.
②는 인자 `kubernetes`가 CMD만 교체해서 `hello kubernetes` — ENTRYPOINT는 그대로.
③은 `--entrypoint hostname`으로 본체를 갈아끼우자 CMD까지 무시되고 컨테이너
호스트명만 출력됐다.

---

## 2. 이미지 유통 경로 — ImagePullBackOff 일부러 체험하고 고치기

### 개념: 이미지 저장소는 3군데로 나뉘어 있다

```
[내 PC의 Docker 데몬]          [레지스트리 (Docker Hub 등)]        [minikube 노드]
 docker build 결과가             docker push로 올리는 곳           kubelet이 이미지를
 저장되는 곳                     (공유 창고)                        pull해서 저장하는 곳
```

- `docker build`는 **내 PC의 Docker 데몬 저장소**에만 이미지를 남긴다.
- minikube는 같은 PC 위에서 돌지만 사실상 **별개의 서버(노드)** — 자기만의 컨테이너
  런타임과 이미지 저장소를 가진다.
- K8s 노드는 이미지가 필요하면 ① 자기 저장소를 먼저 보고 ② 없으면 **레지스트리에서
  pull**한다. 노드가 "옆에 있는 내 PC의 Docker 데몬"을 뒤지는 일은 절대 없다.

그래서 로컬에서 빌드만 하고 배포하면: kubelet이 레지스트리에서
`docker.io/library/hello-k8s:v1`을 pull 시도 → 그런 저장소는 없음 → `ErrImagePull` →
재시도 간격을 늘리며 대기 → `ImagePullBackOff`.

실무 경로는 `빌드 → 레지스트리 push → K8s가 pull`(CI/CD가 하는 일이 바로 이것)이고,
로컬 학습 환경에서는 레지스트리를 생략하는 지름길로 `minikube image load`(내 PC 저장소 →
노드 저장소로 직접 복사)를 쓴다.

### 왜 이렇게 설계됐는가

K8s는 "노드 수십~수천 대" 규모를 전제한다. 어떤 노드에 Pod가 배치될지 모르므로, 모든
노드가 **공통으로 접근 가능한 중앙 창고(레지스트리)**에서 이미지를 받아오는 것이 유일하게
확장 가능한 구조다. 특정 개발자 PC의 로컬 저장소에 의존하는 순간 그 구조가 깨진다.
ErrImagePull과 ImagePullBackOff가 나뉘어 있는 것도 같은 맥락 — 레지스트리 장애나 네트워크
문제는 일시적일 수 있으므로 바로 포기하지 않고 **지수 백오프**(재시도 간격을 10초 →
20초 → 40초… 최대 5분으로 늘림)로 재시도하되, 무한 재시도로 레지스트리를 두들기지 않게
설계됐다.

> **면접 답변**: "ImagePullBackOff가 뜨면 어떻게 대응하나요?"
> "kubectl describe pod로 Events를 먼저 봅니다. pull access denied나 not found 메시지라면
> ①이미지 이름/태그 오타 ②레지스트리에 push가 안 됨 ③private 저장소인데 imagePullSecret이
> 없음, 이 세 가지를 순서대로 의심합니다. ErrImagePull은 pull이 방금 실패한 상태,
> ImagePullBackOff는 실패가 반복되어 지수 백오프로 재시도 대기 중인 상태로, 별개 에러가
> 아니라 같은 실패의 두 국면입니다."

### 직접 확인한 실습

**(1) 이미지 없이 배포 → 실패 관찰.** `hello-k8s:v1`을 빌드하지 않은 상태(내 PC에도 없고,
레지스트리에도 없고, 노드에도 없음)로 배포했다:
```
PS> kubectl create deployment hello --image=hello-k8s:v1
deployment.apps/hello created
PS> kubectl get pods
NAME                    READY   STATUS              RESTARTS   AGE
hello-94f6c6458-9qm9s   0/1     ContainerCreating   0          3s
```

시간이 지나자 `ImagePullBackOff`로 전환:
```
PS> kubectl get pods
NAME                    READY   STATUS             RESTARTS   AGE
hello-94f6c6458-9qm9s   0/1     ImagePullBackOff   0          17m
```

**(2) `kubectl describe pod -l app=hello`로 원인 확인.** describe는 스펙 + 현재 상태 +
사건 기록(Events)을 합친 진단 리포트다. 결정적 단서들:

```
Containers:
  hello-k8s:
    Container ID:                        ← 비어 있음 = 컨테이너 생성조차 안 됨
    State:          Waiting
      Reason:       ImagePullBackOff
...
QoS Class:                   BestEffort  ← requests/limits를 안 줘서 받은 등급 (오늘 후반 주제)
Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  17m                   default-scheduler  Successfully assigned default/hello-94f6c6458-9qm9s to minikube
  Normal   Pulling    14m (x5 over 17m)     kubelet            Pulling image "hello-k8s:v1"
  Warning  Failed     14m (x5 over 17m)     kubelet            Failed to pull image "hello-k8s:v1": Error response from daemon: pull access denied for hello-k8s, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed     14m (x5 over 17m)     kubelet            Error: ErrImagePull
  Normal   BackOff    2m41s (x64 over 17m)  kubelet            Back-off pulling image "hello-k8s:v1"
  Warning  Failed     2m41s (x64 over 17m)  kubelet            Error: ImagePullBackOff
```

**Events 읽기 포인트**:
- `pull access denied ... repository does not exist or may require 'docker login'` —
  Docker Hub는 "없는 건지, 비공개라 권한이 없는 건지"를 구분해서 알려주지 않는다(존재
  여부 자체가 정보라서). 그래서 이 메시지 하나로 오타/미push/인증누락 세 경우를 모두
  의심해야 한다.
- `Pulling (x5)` vs `BackOff (x64)` — pull 시도는 17분간 5번뿐인데 백오프 대기 이벤트는
  64번. 지수 백오프가 동작 중이라는 흔적.
- `Scheduled`는 Normal — 스케줄링은 성공했고, 실패는 그 다음 단계(kubelet의 이미지
  확보)에서 났다는 것까지 Events가 단계별로 알려준다.

**(3) 해결: 빌드 → 노드로 복사 → Pod 재생성.**
```
PS> docker build -t hello-k8s:v1 -f Dockerfile.web .
 => naming to docker.io/library/hello-k8s:v1
PS> docker images hello-k8s
IMAGE          ID             DISK USAGE   CONTENT SIZE
hello-k8s:v1   89ac7266e544       73.6MB           21MB
PS> minikube image load hello-k8s:v1
PS> kubectl delete pod -l app=hello
pod "hello-94f6c6458-9l4ls" deleted from default namespace
PS> kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
hello-94f6c6458-ntv7b   1/1     Running   0          3s
```

`kubectl delete pod`는 백오프 타이머(최대 5분)를 기다리지 않고 즉시 재시도시키는 트릭 —
Deployment/ReplicaSet이 `replicas: 1`을 유지하려고 곧바로 새 Pod를 만들고(자기치유),
새 Pod는 노드에 이미지가 이미 있으니 pull 없이 시작된다. 17분 실패하던 Pod가 3초 만에 떴다.

**(4) 진짜 내 이미지인지 확인.** Dockerfile에서 바꿔둔 nginx 초기 페이지가 나오면 증명 완료:
```
PS> kubectl port-forward deployment/hello 8080:80
Forwarding from 127.0.0.1:8080 -> 80
# (다른 창에서)
PS> curl.exe http://localhost:8080
hello from my own image v1
```

---

## 3. imagePullPolicy — image load를 해도 계속 실패하는 함정

### 개념

kubelet이 이미지를 구할 때 따르는 정책. `imagePullPolicy`를 생략하면 **태그에 따라
기본값이 달라진다** (Kubernetes 공식 문서 v1.35 기준, 2026-07 확인):

| 이미지 지정 방식 | 기본 imagePullPolicy | 의미 |
|---|---|---|
| `:latest` 또는 태그 생략 | `Always` | 매번 레지스트리에 확인 |
| 구체적 태그 (`:v1` 등) | `IfNotPresent` | 노드에 있으면 pull 안 함 |
| digest(`@sha256:...`) 지정 | `IfNotPresent` | 내용이 고정이므로 재확인 불필요 |

(`Never`도 있다 — 레지스트리를 아예 안 보고, 노드에 없으면 그냥 실패.)

### 왜 이렇게 설계됐는가

`:latest`는 **가리키는 내용이 계속 바뀌는 태그**라서, 노드에 캐시된 옛 latest를 쓰면
노드마다 다른 버전이 도는 사고가 난다. 그래서 latest만 `Always`로 강제 확인. 반대로
`:v1` 같은 고정 태그는 내용이 안 바뀐다는 약속이므로 캐시를 신뢰(`IfNotPresent`)해서
레지스트리 트래픽과 시작 시간을 아낀다. 이 설계 때문에 생기는 유명한 함정:
**`minikube image load`로 이미지를 노드에 넣어도 `:latest` 태그면 여전히 Always 정책이
레지스트리 pull을 시도하다 ImagePullBackOff가 난다.** 학습/로컬 환경에서 항상 버전 태그를
붙이는 이유. 오늘 실습이 성공한 것도 `:v1` 태그 → `IfNotPresent` → 노드에 복사해둔
이미지를 발견하고 pull을 생략했기 때문이다.

> **면접 답변**: "로컬 이미지를 minikube에 load했는데도 ImagePullBackOff가 계속 나면?"
> "imagePullPolicy를 의심합니다. 태그가 latest거나 생략되면 기본 정책이 Always라서 노드에
> 이미지가 있어도 매번 레지스트리 pull을 시도하다 실패합니다. v1 같은 구체적 태그를 쓰면
> 기본값이 IfNotPresent가 되어 노드의 로컬 이미지를 사용합니다. 그래서 로컬 개발 환경에서는
> 항상 버전 태그를 붙이고, 프로덕션에서는 반대로 불변 배포를 위해 digest 고정이나 CI가
> 찍는 고유 태그를 씁니다."

---

## 오늘의 트러블슈팅 노트

- **`kubectl create deplyment` 오타 → `error: unknown flag: --image`**: 에러가 오타
  자체가 아니라 플래그를 가리켜서 헷갈렸다. `deployment` 서브커맨드를 인식 못 하니
  `--image` 플래그도 모르는 것 — kubectl 에러는 "어느 단계에서 파싱이 깨졌는지"부터 봐야 한다.
- **`dial tcp 127.0.0.1:64314: connectex ... refused`**: minikube가 꺼진 상태
  (재부팅 후 단골 패턴). Docker는 살아 있어도(빌드 잘 됨) minikube는 별도로 `minikube start`
  해야 한다. kubectl이 접속하는 `127.0.0.1:<포트>`는 minikube가 kubeconfig에 등록해둔
  API 서버 주소.
- **`minikube image load` → `The image 'hello-k8s:v1' was not found`**: minikube 문제가
  아니라 **복사할 원본이 내 PC Docker 데몬에 없다**는 뜻. 빌드 단계를 건너뛴 게 원인이었다.
  이미지 관련 에러는 "지금 이 이미지가 세 저장소 중 어디에 있어야 하는가"부터 따지면 빠르다.
- **`kubectl port-forward`는 포그라운드 유지가 정상**: 터널을 유지하는 프로세스라 프롬프트가
  안 돌아온다. 다른 창에서 접속 테스트하고 Ctrl+C로 종료.
