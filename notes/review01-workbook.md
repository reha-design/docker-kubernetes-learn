# 회상형 워크북 1호 — Day 1~7 취약 지점 (2026-07-20 기준)

> [복습 문답 1회차](review01-day01-07-quiz.md)에서 △/○ 받은 항목만 모은 인출 훈련용 워크북.
>
> **사용법**
> 1. 정답을 펼치기 전에 **"내 답변" 칸에 먼저 직접 쓴다** — 기억이 안 나도 아는 데까지 쥐어짜서 쓴다 (이 쥐어짜는 과정이 학습이다).
> 2. 다 쓴 뒤 `▶ 정답 확인`을 열어 대조한다.
> 3. 자가 채점표에 ◎/○/△ 기록. △였던 문제는 며칠 뒤 다음 회차에서 다시.
> 4. 노트 파일(day02 등)은 정답지가 헷갈릴 때만 참고서로 연다.

## 자가 채점표

| # | 항목 | 1회차 ( / ) | 2회차 ( / ) | 3회차 ( / ) |
|---|---|---|---|---|
| 1 | 이미지 레이어 구조 | △ | | |
| 2 | Dockerfile 레이어 캐시 | △ | | |
| 3 | Pod/Deployment/Service | ○ | | |
| 4 | ConfigMap vs Secret | ○ | | |
| 5 | Service DNS 동작 원리 | ○ | | |
| 6 | requests vs limits | ○ | | |
| 7 | HPA max 도달 이후 | | | |
| 8 | StatefulSet | | | |
| 9 | readiness vs liveness (이월) | | | |
| 10 | 무중단 배포의 두 설정 (이월) | | | |

---

## 1. 이미지 레이어 구조 (Day 1/5 — 지난 평가 ○)

**Q.** 이미지 하나로 컨테이너 10개를 띄웠다. 디스크에는 무엇이 공유되고, 컨테이너마다 무엇이 새로 생기나? 컨테이너 안에서 만든 파일이 재시작 후 사라지는 이유를 이 구조로 설명하라.

(구조도: [notes/day01-docker-basics.md](day01-docker-basics.md#2-이미지image와-레이어--copy-on-write) 2번 섹션 참고 — 정답 확인 전에는 보지 말 것)

**내 답변 (1회차):** 이미지 하나로 컨테이너를 10개 띄웠을 경우, 디스크에서는 이미지만 공유되고 컨테이너마다 읽기 레이어가 새로 생긴다. 컨테이너 안에서 만든 파일이 있을 경우 컨테이너 재시작할 때, 기존 이미지 기반으로 재시작하기 때문에 작업했던 내용이 초기화될 수 있다.

> 채점: △ — "읽기 레이어가 새로 생긴다"는 오류(정답은 **쓰기 레이어**). 읽기 전용 레이어는 공유되는 것이지 새로 생기는 게 아님.

<details>
<summary>▶ 정답 확인</summary>

- 이미지의 **읽기 전용 레이어들은 10개 컨테이너가 전부 공유**한다 (디스크에 1벌).
- 컨테이너마다 새로 생기는 건 얇은 **쓰기 가능 레이어(writable layer)** 하나뿐. 그래서 디스크는 컨테이너 수만큼 늘지 않는다.
- 컨테이너 안에서 만든 파일은 이 쓰기 레이어에 기록되는데, **쓰기 레이어는 컨테이너와 운명 공동체**다. 컨테이너가 재생성되면(예: liveness 실패 재시작) 이미지에서 새 쓰기 레이어를 만들므로 이전 파일은 사라진다. Day 7에서 `/healthz`가 증발한 이유.
- 한 줄 정의: 이미지 = 읽기 전용 템플릿, 컨테이너 = 쓰기 레이어를 얹은 실행 중인 인스턴스 (클래스와 인스턴스 관계).

</details>

---

## 2. Dockerfile 레이어 캐시 (Day 6 — 지난 평가 △)

**Q.** `COPY . .`을 `RUN npm install`보다 앞에 두면 무슨 문제가 생기나? 캐시 무효화 규칙과 정석 순서까지 쓰라.

(구조도: [notes/day06-dockerfile-networking.md](day06-dockerfile-networking.md#2-레이어-캐싱--명령어-순서가-빌드-속도를-좌우한다) 2번 섹션 참고 — 정답 확인 전에는 보지 말 것)

**내 답변 (1회차):** 캐싱이 안 되고 다시 재작업하는 경우가 발생해.

> 채점: △ — 증상은 맞았지만 무효화 규칙(한 레이어 미스 → 그 뒤 전부 재실행)과 정석 순서가 빠짐. Day 6 실측 예제(flask-bad 5.5s vs flask-good 1.6s)로 보충 설명함.

<details>
<summary>▶ 정답 확인</summary>

- 규칙: 레이어별 캐시는 **하나가 미스나면 그 뒤 전부 재실행**된다 (위→아래 무효화).
- `COPY . .`이 앞이면 소스 한 줄만 고쳐도 그 레이어가 미스 → 뒤의 `npm install`까지 매번 재실행 → 의존성이 안 바뀌었는데도 빌드마다 낭비.
- 정석: `COPY package*.json .` → `RUN npm install` → `COPY . .` — **잘 안 바뀌는 것을 앞에, 자주 바뀌는 것을 뒤에.**
- 면접 한 줄: "레이어 캐시는 위에서 아래로 무효화되므로, 변경 빈도가 낮은 명령을 앞에 배치해 캐시 적중률을 높입니다."

</details>

---

## 3. Pod / Deployment / Service (Day 2 — 지난 평가 △, 1순위)

**Q.** 세 리소스가 각각 왜 필요한가? 특히 "Deployment만 있으면 되지, Service는 왜 따로 있나?"에 답하라.

(관계 구조도: [notes/day02-kubernetes-basics.md](day02-kubernetes-basics.md#5-service--왜-필요한가) 5번 섹션 참고 — 정답 확인 전에는 보지 말 것)

**내 답변 (1회차):** Pod는 실행 단위, Deployment는 개수·버전 관리, Service는 파드 IP 앞의 고정 창구

> 채점: ○ — 결론 요약은 정확. "Pod가 재생성될 때마다 IP가 바뀌어서 Deployment만으론 접속 문제를 못 푼다"는 인과관계 설명이 빠짐.

<details>
<summary>▶ 정답 확인</summary>

- **Pod**: 배포의 최소 단위. 컨테이너 1개 이상을 묶고 IP 하나를 받는다.
- **Deployment**: "파드를 N개로 유지하라"는 선언. 죽으면 재생성, 이미지 교체 시 롤링 업데이트 (내부적으로 ReplicaSet을 통해 관리).
- **Service**: 파드들의 고정 접속 창구. **파드는 재생성될 때마다 IP가 바뀌므로** Deployment가 개수를 유지해줘도 "어느 IP로 접속하나"는 못 푼다. Service가 불변의 이름/ClusterIP를 제공하고, 뒤의 파드 목록은 label selector로 자동 갱신한다. Ready 안 된 파드를 빼주는 것도 Service(EndpointSlice).
- 한 줄: "Pod는 실행 단위, Deployment는 개수·버전 관리, Service는 바뀌는 파드 IP 앞의 고정 창구."

</details>

---

## 4. ConfigMap vs Secret (Day 2 — 지난 평가 △)

**Q.** 설정을 이미지 밖으로 빼는 **핵심** 이유는? ConfigMap과 Secret의 차이 3가지는? "Secret은 암호화되어 저장된다"는 말은 맞나?

(구조도: [notes/day02-kubernetes-basics.md](day02-kubernetes-basics.md#6-configmap과-secret--설정과-민감정보-분리) 6번 섹션 참고 — 정답 확인 전에는 보지 말 것)

**내 답변 (1회차):** 핵심 이유는 환경별로 이미지 재빌드 안 해도 되게 하려고. 차이: ① ConfigMap은 일반 설정, Secret은 민감정보 ② RBAC 권한 따로 지정 가능. Secret은 암호화되는 것이 아닌 base64로 인코딩되는 것.

> 채점: ○ — 핵심 이유를 부수 효과(재빌드 회피)와 혼동(정정: 핵심은 "같은 이미지를 전 환경에서 그대로 쓰기 위해"). 차이 3가지 중 2개(민감정보 구분, RBAC)는 맞았고 ③ etcd 암호화 대상 지정 가능이 빠짐. base64 vs 암호화 구분은 정확.

<details>
<summary>▶ 정답 확인</summary>

- 핵심 이유: **같은 이미지를 dev/staging/prod 전 환경에서 그대로 쓰기 위해.** 환경별로 다른 건 설정뿐이니 설정만 주입하면 "테스트한 그 이미지"가 프로덕션까지 간다. (재빌드 회피는 부수 효과)
- 차이: ① ConfigMap은 일반 설정, Secret은 민감 정보(비밀번호·토큰·인증서) ② Secret은 RBAC으로 접근 권한을 따로 좁힐 수 있다 ③ Secret은 etcd 암호화(encryption at rest) 대상으로 지정할 수 있다.
- **틀렸다** — Secret은 기본적으로 base64 **인코딩**일 뿐 암호화가 아니다 (면접 단골 함정). 디코딩하면 바로 보인다.

</details>

---

## 5. Service DNS 동작 원리 (Day 2 — 지난 평가 ○)

**Q.** 파드 A가 `my-service`라는 이름으로 다른 파드에 접속된다. 이름이 IP가 되기까지의 과정을 등장 컴포넌트(2개 이상) 중심으로 쓰라. Docker Compose의 이름 통신과 무엇이 같고 무엇이 다른가?

(구조도: [notes/day02-kubernetes-basics.md](day02-kubernetes-basics.md#7-service를-dns로-찾기--coredns-상세-설명) 7번 섹션 참고 — 정답 확인 전에는 보지 말 것)

**내 답변 (1회차):** CoreDNS가 my-service를 ClusterIP로 해석해주고, kube-proxy가 Ready 파드로 트래픽을 분배해준다. Compose와 같은 점은 이름으로 찾는 것이고, 다른 점은 로드밸런싱 여부.

> 채점: ○ — 컴포넌트 2개(CoreDNS, kube-proxy) 정확. Compose 비교도 핵심은 맞았으나 "Ready 파드만 골라줌"도 다른 점에 포함되면 더 완전한 답.

<details>
<summary>▶ 정답 확인</summary>

- **CoreDNS**가 `my-service` → Service의 **ClusterIP**(불변)로 해석 → 그 IP로 온 트래픽을 **kube-proxy**가 Ready 상태인 파드들로 분배.
- 같은 점: "이름으로 찾고 IP는 바뀌어도 된다"는 철학. Compose에선 도커 내장 DNS(127.0.0.11)가 서비스 이름을 컨테이너 IP로 풀어준다.
- 다른 점: Compose DNS는 이름→IP 해석까지만. K8s Service는 그 위에 **여러 파드로 로드밸런싱 + Ready인 파드만 골라줌**이 얹혀 있다.

</details>

---

## 6. requests vs limits (Day 3 — 지난 평가 △, 2순위)

**Q.** requests와 limits를 각각 정의하라. Pending과 OOMKilled는 각각 어느 쪽과 관련 있고, 그 이유는? 이 조합이 결정하는 Day 7 개념은?

(구조도: [notes/day03-hpa-scheduling.md](day03-hpa-scheduling.md#4-핵심-개념-비교-정리--pending-vs-oomkilled) 4번 섹션 참고 — 정답 확인 전에는 보지 말 것)

**내 답변 (1회차):** requests: 컨테이너가 필요로 하는 최소보장 예약량, 스케줄러가 파드를 어느 노드에 놓을지 결정할 때 보는 숫자. limits: 컨테이너가 쓸 수 있는 사용 상한선. Pending은 requests 관련 — 노드마다 이미 예약된 requests 합계 장부가 있는데, 새 파드가 요구하는 requests만큼 빈자리가 있는 노드가 클러스터 어디에도 없으면 배치되지 않고 Pending으로 남음. OOMKilled는 limits 관련. 조합이 결정하는 개념은 몰랐음.

> 채점: ○ — requests/limits 정의와 Pending 인과관계까지 정확. 조합이 결정하는 **QoS 클래스**(Guaranteed/Burstable/BestEffort, 노드 메모리 압박 시 축출 순위)만 놓침 — 이번에 보충 설명함.

<details>
<summary>▶ 정답 확인</summary>

- **requests** = 최소 보장 **예약량**. 스케줄러가 파드를 어느 노드에 놓을지 정할 때 보는 숫자.
- **limits** = 사용 **상한선**. 실행 중에 kubelet/커널이 강제하는 숫자.
- **Pending ← requests**: 모든 노드의 장부를 봐도 requests만큼 빈 자원이 있는 노드가 없으면 배치 불가. limits는 스케줄링에 안 쓰인다.
- **OOMKilled ← limits**: 실행 중 메모리가 limits를 넘으면 커널이 그 컨테이너를 죽인다 (그 자리에서 재시작).
- 기억법: **"Pending은 입장 전(requests로 심사), OOMKilled는 입장 후(limits로 단속)."**
- 조합이 결정하는 것: **QoS 클래스** — requests==limits면 Guaranteed, 일부만 있으면 Burstable, 없으면 BestEffort. 노드 메모리 압박 시 축출 순위.

</details>

---

## 7. HPA max 도달 이후 (Day 3 — 지난 평가 ○)

**Q.** HPA는 기본적으로 **무엇을 무엇에 대비한 비율**로 보나? requests가 없으면 HPA는 어떻게 되나? max replicas 도달 후에도 부하가 계속 오르면?

(구조도: [notes/day03-hpa-scheduling.md](day03-hpa-scheduling.md#1-hpa의-max상한선-테스트--왜-max까지-안-늘어나는가) 1번 섹션 참고 — 정답 확인 전에는 보지 말 것)

**내 답변 (1회차):**

<br><br><br><br>

<details>
<summary>▶ 정답 확인</summary>

- 기본 지표: metrics-server가 수집한 **CPU 사용률, requests 대비 %**.
- requests가 없으면 % 계산 자체가 불가능해서 **HPA가 동작하지 못한다** (Day 3에서 확인).
- max 도달 후: **max에서 멈춘다.** 초과 부하는 기존 파드들이 나눠 받으며 응답 지연/실패. 무한 증식 방지가 우선인 설계. 대응: max 상향, 노드 증설(cluster autoscaler), 앞단 부하 제한.

</details>

---

## 8. StatefulSet (Day 4 — 지난 평가 △)

**Q.** Deployment 대신 StatefulSet을 쓰는 경우는? StatefulSet이 보장하는 두 가지는? Deployment 파드로는 왜 안 되나?

(구조도: [notes/day04-workload-types.md](day04-workload-types.md#2-statefulset) 2번 섹션 참고 — 정답 확인 전에는 보지 말 것)

**내 답변 (1회차):**

<br><br><br><br>

<details>
<summary>▶ 정답 확인</summary>

- **언제**: 파드들이 서로 **구별되어야 하는** 상태ful 워크로드 — DB(primary/replica 구분), Kafka/Zookeeper 같은 클러스터형 미들웨어.
- **보장 ①**: 고정된 이름과 순서 — `mysql-0, mysql-1, ...` 번호가 붙고 재시작해도 같은 이름, 생성/삭제도 순서대로.
- **보장 ②**: 파드별 전용 스토리지 — volumeClaimTemplates로 파드마다 자기 PVC가 붙고, 재시작해도 자기 데이터를 되찾는다 (Day 4 실습에서 검증).
- Deployment가 안 되는 이유: 파드가 전부 동일한 복제품(이름 랜덤, 스토리지 공유/없음)이라 "내가 1번, 쟤가 2번" 개념이 없다.

</details>

---

## 9. readiness vs liveness — Day 7 이월 재시험

**Q.** 두 프로브가 각각 실패하면 무슨 일이 일어나나? 왜 하나로 합치지 않고 분리했나? (Day 7 실습에서 본 숫자를 근거로 들 수 있으면 만점)

(구조도: [notes/day07-docker-k8s-bridge.md](day07-docker-k8s-bridge.md#4-liveness--readiness--startup-probe--running과-ready는-다르다) 4번 섹션 참고 — 정답 확인 전에는 보지 말 것)

**내 답변 (1회차):**

<br><br><br><br>

<details>
<summary>▶ 정답 확인</summary>

- **readiness 실패** → Service 엔드포인트에서 **제외만** 되고 재시작은 없다. 실습 근거: 404로 **126회 실패하는 동안 RESTARTS 0**.
- **liveness 실패** → 컨테이너 **재시작** (파드 재생성 아님 — IP/이름 유지, RESTARTS +1). 실습 근거: 403 × 3회(failureThreshold) → Killing → 재시작.
- 분리한 이유: **처방이 정반대**라서. 데드락은 재시작이 답이지만, "DB가 잠깐 느려서 준비 안 됨"을 재시작으로 처방하면 멀쩡한 파드를 계속 죽이는 재시작 폭풍이 된다.
- 보너스: 기동이 느린 앱은 **startupProbe**로 기동 구간을 따로 보호 — 기동 여유는 길게, 운영 중 장애 감지는 빠르게.

</details>

---

## 10. 무중단 배포의 두 설정 — Day 7 이월 재시험

**Q.** 롤링 업데이트 중 새 버전 이미지가 고장(ImagePullBackOff)났는데도 서비스가 무중단이었다. 이를 가능하게 한 설정 두 가지와 각각의 역할은?

(구조도: [notes/day07-docker-k8s-bridge.md](day07-docker-k8s-bridge.md#5-배포-전략--rolling-update-파라미터-롤백-blue-greencanary) 5번 섹션 참고 — 정답 확인 전에는 보지 말 것)

**내 답변 (1회차):**

<br><br><br><br>

<details>
<summary>▶ 정답 확인</summary>

- **① `maxUnavailable: 0`** — 교체 중 원하는 개수에서 하나도 모자라면 안 된다는 설정. 새 파드가 합격하기 전엔 옛 파드를 절대 안 죽인다 (maxSurge로 여유분만 추가).
- **② readiness probe** — 새 파드의 **합격 기준**. 새 파드가 Ready가 안 되니 롤아웃이 그 자리에서 멈추고, 옛 파드 3개가 계속 트래픽을 받았다 (실습에서 14분간 무중단).
- 둘의 관계: maxUnavailable=0은 "옛것을 지키는 규칙", readiness는 "새것을 검증하는 기준". readiness가 없으면 "떴다=준비됐다"로 간주되어 고장난 버전으로 전부 교체돼버린다.
- 복구: `kubectl rollout undo` — 0개로 보관된 옛 ReplicaSet 템플릿으로 **새 리비전**을 만든다 (시간 여행이 아님).

</details>
