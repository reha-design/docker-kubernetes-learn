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

## 학습 로드맵 (면접 대비)

- **Day 6** — Dockerfile 작성(핵심 명령어, 레이어 캐싱, 멀티스테이지 빌드) + Docker 네트워킹(bridge/host/none, 커스텀 네트워크). K8s 네트워킹(Service/DNS/Ingress)은 Day 2에서 이미 다뤄 제외

> Day 7 이후 로드맵은 기존 학습 내용 점검(빠진 주제 확인) 결과를 반영해 재확정 예정.
> 미학습 후보: probe/배포 전략/QoS, 스케줄링(affinity/taint)과 멀티노드, Namespace/RBAC,
> 트러블슈팅 워크플로(CrashLoopBackOff 등), 아키텍처 총정리 + Helm/GitOps 개념.
