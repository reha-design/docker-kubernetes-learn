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

## 학습 로드맵 (면접 대비, Day 4~9)

- **Day 4** — 워크로드 타입: PersistentVolume/PVC/StorageClass + StatefulSet + Job/CronJob/DaemonSet. 이날부터 YAML 선언형(`kubectl apply`) 전환
- **Day 5** — liveness/readiness/startup probe + 배포 전략(rolling update 파라미터, rollback, blue-green/canary) + QoS 클래스
- **Day 6** — 스케줄링: nodeSelector/affinity/taint/toleration + `minikube start --nodes 2` 멀티노드 실습(노드 장애 시나리오)
- **Day 7** — Namespace, RBAC, ServiceAccount
- **Day 8** — 트러블슈팅 집중: CrashLoopBackOff, ImagePullBackOff, Pending, OOMKilled — `kubectl describe/logs/events` 워크플로
- **Day 9** — 아키텍처 총정리 + 모의 면접, Helm/GitOps(ArgoCD)/서비스메시(Istio) 개념 정리

> 일정 압축 시: Day 6 스케줄링을 Day 8 트러블슈팅(Pending 원인 분석)에 흡수 가능.
