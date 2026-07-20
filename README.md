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

## 학습 로드맵 (면접 대비, Day 7~12)

CKA/CKAD 공식 커리큘럼(Kubernetes v1.35 기준, 2026-07 조회)과 Day 1~6 학습 기록을 대조해 확정.

- **Day 7 (브리지 실습 먼저)** — Docker→K8s 연결: 직접 빌드한 이미지를 minikube에 배포(`minikube image load`, imagePullPolicy, ImagePullBackOff 체험) + ENTRYPOINT vs CMD + Docker Compose 실습(web+db 이름 통신). 이후 본 주제: liveness/readiness/startup probe + 배포 전략(rolling update 파라미터, rollback, blue-green/canary) + QoS 클래스
- **Day 8** — 스케줄링: nodeSelector/affinity/taint/toleration + `minikube start --nodes 2` 멀티노드 실습 + init 컨테이너/사이드카 패턴
- **Day 9** — Namespace, RBAC, ServiceAccount + ResourceQuota/LimitRange + SecurityContext
- **Day 10** — NetworkPolicy + Gateway API (Ingress와 비교)
- **Day 11** — 트러블슈팅 집중: CrashLoopBackOff, ImagePullBackOff 등 — `kubectl describe/logs/events` 워크플로 (CKA 최대 도메인 30%)
- **Day 12** — Helm/Kustomize 실습 + CRD/Operator 개념 + 아키텍처 총정리, 모의 면접

> CKA의 kubeadm 클러스터 설치/HA 컨트롤플레인 항목은 개발자 포지션 면접 목적에 우선순위가 낮아 제외.
