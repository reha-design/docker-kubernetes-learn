# 실습 가이드 — requests/limits가 실제로 cgroup에 어떻게 집행되는가

> 목적: Kubernetes의 requests/limits가 스케줄러의 "장부"로만 쓰이는 게 아니라, 실제로 Linux
> cgroup 파일 값으로 번역돼 커널이 직접 집행한다는 것을 눈으로 확인하는 재현 절차.
> 개념 설명은 [day03-hpa-scheduling.md §2](day03-hpa-scheduling.md#2-노드-자원-장부--kubectl-describe-node의-allocated-resources)
> 참고. 환경: Windows 11 + Docker Desktop(WSL2) + minikube, PowerShell.

## 사전 준비

**1) minikube가 켜져 있는지 확인**:
```powershell
minikube status
```
`Stopped`이거나 `kubeconfig: Misconfigured`가 뜨면(재부팅했거나 오래 방치한 뒤 흔히 발생),
아래로 재조정한다 — 기존 VM을 지우지 않고 빠진 부분(kubelet/apiserver/kubeconfig)만
복구하므로 안전하게 재실행 가능:
```powershell
minikube start
```

**2) 레포 루트로 이동** (매니페스트를 상대경로로 쓰기 위해):
```powershell
cd C:\Users\prist\docker-kubernetes-learn
```

## 1단계 — 테스트 파드 생성 (기존 매니페스트 재활용)

새로 YAML을 작성할 필요 없이 Day 7에서 쓴 매니페스트를 그대로 재사용한다.
`qos-guaranteed`와 `qos-burstable` 두 파드가 같이 생성되는데, 이번 실습엔 `qos-burstable`만
쓴다 (requests < limits라 두 값이 cgroup 파일에서 서로 다른 숫자로 나뉘어 보임):

```powershell
kubectl apply -f manifests/day07/qos-demo.yaml
kubectl wait --for=condition=Ready pod/qos-burstable --timeout=60s
```

`qos-burstable` 스펙: `cpu: requests=100m/limits=200m`, `memory: requests=64Mi/limits=128Mi`
(nginx:1.27 이미지, `manifests/day07/qos-demo.yaml` 참고).

## 2단계 — cgroup 버전 확인

```powershell
minikube ssh -- stat -fc %T /sys/fs/cgroup
```
`cgroup2fs`가 나오면 아래 파일명을 그대로 쓰면 된다. (다른 값이면 cgroup v1이라 파일명이
`cpu.cfs_quota_us`, `memory.limit_in_bytes` 등 legacy 이름으로 바뀐다.)

## 3단계 — limits 확인 (핵심, 필수)

```powershell
kubectl exec qos-burstable -- cat /sys/fs/cgroup/cpu.max
kubectl exec qos-burstable -- cat /sys/fs/cgroup/memory.max
```

**검증 방법**:
- `cpu.max`는 `<쿼터> <주기>` 형식. 주기가 100000(=100ms 기본값)이면, 쿼터÷100000이 코어
  수 → ×1000하면 밀리코어. limits.cpu와 같아야 한다.
- `memory.max`는 바이트 단위. limits.memory(Mi) × 1024 × 1024와 같아야 한다.

**2026-07-22 실제 확인된 값**: `cpu.max` = `20000 100000`(200m과 일치), `memory.max` =
`134217728`(128Mi와 일치). cgroup v2(`cgroup2fs`) 환경에서 검증 완료 — 자세한 해석은
[day03-hpa-scheduling.md §2](day03-hpa-scheduling.md#2-노드-자원-장부--kubectl-describe-node의-allocated-resources)에 기록.

## 4단계 — requests 확인 (참고용, 깔끔한 숫자가 아님)

```powershell
kubectl exec qos-burstable -- cat /sys/fs/cgroup/cpu.weight
```
**"100"처럼 딱 떨어지는 값을 기대하면 안 된다.** requests(100m)는 legacy `cpu.shares`
(1024 기준, 100m→약 102)로 먼저 환산된 뒤, v2 `cpu.weight`(1~10000 범위)로 재변환하는 공식을
거쳐 한 자릿수의 작은 값으로 나온다. 숫자 자체보다 "이 파일이 존재하고, requests를 바꾸면
이 값도 바뀐다"는 사실이 확인 포인트다.

## 5단계 (선택) — 실시간 스로틀링 확인

**연속으로 흐르는 `watch` 화면보다, 스냅샷 두 번을 찍어 비교하는 게 훨씬 확실하다** — 계속
갱신되는 터미널은 복사/붙여넣기 중 어느 시점 값인지 헷갈리기 쉽다.

```powershell
# 부하 걸기 (CPU를 최대한 쓰는 무한 루프를 백그라운드로)
kubectl exec qos-burstable -- sh -c "yes > /dev/null &"
```
5초 정도 기다린 뒤 첫 스냅샷:
```powershell
kubectl exec qos-burstable -- cat /sys/fs/cgroup/cpu.stat
```
또 5초 기다렸다가 같은 명령을 다시 실행. **`nr_throttled`와 `throttled_usec`가 두 스냅샷
사이에 증가**했다면, 200m 한도가 실시간으로 커널에 의해 집행되고 있다는 직접 증거다.

> **주의**: nginx는 idle 상태에서도(워커 프로세스 초기화 등) `nr_throttled`가 이미 0이 아닐
> 수 있다. 이상 현상이 아니라 "부하를 걸기 전부터도 200m 한도가 빡빡해서 이미 몇 번
> 부딪혔다"는 뜻일 뿐이다. 판단 기준은 절대값이 아니라 **두 스냅샷 사이의 증가 여부**다.

## 정리

```powershell
kubectl delete -f manifests/day07/qos-demo.yaml
```

## 트러블슈팅 메모 (실제로 겪었던 것들)

- **PowerShell에서 YAML 파일 새로 만들 때**: here-string 사용 —
  `@'...내용...'@ | Set-Content -Path 파일명.yaml -Encoding utf8`. 닫는 `'@`는 반드시 줄
  맨 앞(들여쓰기 없이)에 와야 하고, `-Encoding utf8`을 빠뜨리면 kubectl이 못 읽을 수 있다.
- **"액세스가 거부되었습니다" 에러**: 현재 폴더가 `C:\WINDOWS\System32`처럼 쓰기 권한 없는
  시스템 폴더인 경우 발생. 레포 루트나 홈 폴더로 `cd` 이동 후 재시도.
- **`minikube status`가 `Misconfigured`나 `connection refused`를 낼 때**: VM(`host`)은
  떠 있는데 그 안의 kubelet/apiserver 프로세스가 죽어있고 kubeconfig 포트가 예전 값을
  가리키는(stale) 상황. `minikube start`를 다시 실행하면 기존 VM을 지우지 않고도 대부분
  복구된다.
- **실습용 파일은 매번 새로 만들기 전에**: `manifests/dayNN/`에 재사용할 게 이미 있는지 먼저
  확인할 것 (이번 실습도 `manifests/day07/qos-demo.yaml`을 그대로 재활용했다). 정말 일회성
  스크래치가 필요하면 레포 안 `lab/`(`.gitignore` 처리되어 커밋되지 않음)에 만들고, 나중에
  남길 가치가 있다고 판단되면 `manifests/dayNN/`으로 정식 커밋한다.
