# Docker & Kubernetes 학습

도커와 쿠버네티스를 학습하며 실습 코드와 정리 노트를 기록하는 레포.

## 목표

- Docker: 이미지 빌드, 컨테이너 실행, 네트워크/볼륨, Docker Compose
- Kubernetes: Pod/Deployment/Service 등 핵심 리소스, kubectl, 로컬 클러스터 실습

## 구조

- `notes/` — 일차별 학습 정리 (md)

## 학습 기록

- [Day 1 — Docker 핵심 개념](notes/day01-docker-basics.md)
- [Day 2 — Kubernetes 기초 (minikube, Pod/Deployment/Service, ConfigMap/Secret, DNS)](notes/day02-kubernetes-basics.md)
- [Day 3 — HPA 심화와 스케줄링/자원 관리 (HPA max, 노드 자원 장부, Pending vs OOMKilled)](notes/day03-hpa-scheduling.md)
- [Day 4 — 워크로드 타입 (PV/PVC/StorageClass, StatefulSet, Job/CronJob, DaemonSet, YAML 선언형 전환)](notes/day04-workload-types.md)
- [Day 5 — 복습(day01~02 문답) + Docker 기초 (이미지 vs 컨테이너, Pending vs ErrImagePull, 컨테이너 vs VM)](notes/day05-docker-basics-review.md)
- [Day 6 — Dockerfile과 이미지 레이어(캐싱, 멀티스테이지 빌드) + Docker 네트워킹(bridge/커스텀 네트워크 DNS/격리)](notes/day06-dockerfile-networking.md)
- [Day 7 — Docker→K8s 브리지 (ENTRYPOINT vs CMD, 이미지 유통 경로, imagePullPolicy) + Probe/배포 전략/QoS + Docker Compose](notes/day07-docker-k8s-bridge.md)
- [Day 8 — 스케줄링 제어 (nodeSelector/affinity/taint·toleration, 멀티노드 CNI 트러블슈팅) + init 컨테이너/사이드카](notes/day08-scheduling.md)
- [Day 9 — Namespace/RBAC/ServiceAccount (Role·RoleBinding, 최소 권한 원칙) + ResourceQuota/LimitRange + SecurityContext](notes/day09-namespace-rbac.md)
- [Day 10 — NetworkPolicy(CNI 의존성, deny-all/선택적 allow) + Gateway API(Envoy Gateway, Ingress와 비교, NetworkPolicy와의 실전 충돌)](notes/day10-networkpolicy-gateway.md)
- [Day 11 — 트러블슈팅 집중 (CrashLoopBackOff 지수 백오프 실측, ImagePullBackOff, Service Selector 불일치, describe/logs/events 워크플로)](notes/day11-troubleshooting.md)
- [Day 12 — Helm/Kustomize + CRD·Operator + 아키텍처 총정리, 모의 면접](notes/day12-helm-kustomize-crd.md)
- [복습 문답 1회차 — Day 1~7 전체 점검, 취약 지점 평가와 복습 우선순위](notes/review01-day01-07-quiz.md)
- [회상형 워크북 1호 — 취약 지점 인출 훈련 (먼저 쓰고, 정답 펼쳐 대조)](notes/review01-workbook.md)
- [실습 가이드 — cgroup으로 requests/limits 실측 검증 (재현 가능한 절차)](notes/lab-cgroup-verification.md)
- [개관 노트 — 부하분산 전체 그림 (DNS부터 DB까지)](notes/load-balancing-overview.md)
- [개관 노트 — 장애격리 전체 그림 (컨테이너부터 컨트롤 플레인까지)](notes/fault-isolation-overview.md)
- [보충 학습 — 리버스 프록시와 nginx (root/proxy_pass/upstream, K8s Service와 비교)](notes/nginx-reverse-proxy.md)

## 학습 로드맵 (면접 대비, Day 13~)

Day 7~12 로드맵을 모두 소화한 뒤 재수립(2026-08). 12일간 내용 습득은 이어졌지만 인출
훈련은 Day 7 복습 이후 멈춰 있었고([회상형 워크북 1호](notes/review01-workbook.md)의
2·3회차 칸이 비어 있음, Day 8~12는 복습 자료 자체가 없음), Day 12가 명시한 다음 목표도
"모의 면접"이었다. 그래서 **새 내용을 더 쌓기 전에 정리 단계를 먼저** 둔다.

- **Day 13 — 모의 면접 1차 + 워크북 2호** (Day 8~12 범위)
  - Day 12에서 정리한 **메커니즘형 / 설계·트레이드오프형** 두 질문 유형을 구분해서 진행
  - 범위: Day 8(스케줄링 제어) ~ Day 12(Helm/Kustomize/CRD), 그리고 review01에서 이월된
    Q10(readiness vs liveness)·Q11(무중단 배포 두 설정)
  - 2026-08 정확도 검증에서 교정된 9개 항목(특히 축출 순위, cgroup v1/v2, ConfigMap 전파)은
    교정 직후라 우선 출제 — 갓 고친 내용일수록 인출로 굳혀야 한다
  - 결과: △/○ 항목만 모아 `notes/review02-workbook.md` 작성
- **Day 14 이후 — Day 13 결과를 보고 확정.** 현재 파악된 공백을 우선순위 순으로 나열하면:
  1. **관측성** — Prometheus/Grafana, PromQL, ServiceMonitor, 알람 룰. 12일 내내 metrics-server만
     써봤고 메트릭 수집·쿼리·알람은 전무한, 가장 큰 내용 공백. HPA 커스텀 메트릭으로 Day 3과 연결
  2. **GitOps(ArgoCD)** — Day 4의 선언형 전환과 Day 12의 Helm/Kustomize·`--prune` 문제가
     실제로 수렴하는 지점. 노트에서 세 번 언급됐으나 미실습
  3. **보안 심화(Pod Security Admission)** — Day 9는 SecurityContext(파드가 스스로 선언)까지만
     다뤘고, 클러스터가 강제하는 PSA와 PSS 세 등급은 미학습. 이미지 취약점 스캔/공급망 포함
  4. **서비스 메시(Istio)** — [부하분산 개관](notes/load-balancing-overview.md)과 Day 7 canary
     실습(이론 80:20 vs 실측 75:25)에서 "정밀 제어엔 이게 필요하다"고 두 번 예고된 주제

### 완료된 로드맵 (Day 7~12)

CKA/CKAD 공식 커리큘럼(Kubernetes v1.35 기준, 2026-07 조회)과 Day 1~6 학습 기록을 대조해 확정.

- **Day 7 (브리지 실습 먼저)** — Docker→K8s 연결: 직접 빌드한 이미지를 minikube에 배포(`minikube image load`, imagePullPolicy, ImagePullBackOff 체험) + ENTRYPOINT vs CMD + Docker Compose 실습(web+db 이름 통신). 이후 본 주제: liveness/readiness/startup probe + 배포 전략(rolling update 파라미터, rollback, blue-green/canary) + QoS 클래스
- **Day 8** — 스케줄링: nodeSelector/affinity/taint/toleration + `minikube start --nodes 2` 멀티노드 실습 + init 컨테이너/사이드카 패턴
- **Day 9** — Namespace, RBAC, ServiceAccount + ResourceQuota/LimitRange + SecurityContext
- **Day 10** — NetworkPolicy + Gateway API (Ingress와 비교)
- **Day 11** — 트러블슈팅 집중: CrashLoopBackOff, ImagePullBackOff 등 — `kubectl describe/logs/events` 워크플로 (CKA 최대 도메인 30%)
- **Day 12** — Helm/Kustomize 실습 + CRD/Operator 개념 + 아키텍처 총정리, 모의 면접

> CKA의 kubeadm 클러스터 설치/HA 컨트롤플레인 항목은 개발자 포지션 면접 목적에 우선순위가 낮아 제외.
