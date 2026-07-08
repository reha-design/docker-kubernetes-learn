# Docker 핵심 개념 — 면접 대비 학습 정리

> 학습 목적: 쿠버네티스 학습을 위한 선행 지식으로서 Docker의 핵심 설계 원리를 이해하고,
> 면접에서 "왜 이렇게 설계됐는지"를 설명할 수 있는 수준까지 정리.

---

## 1. 컨테이너 vs 가상머신(VM) — 왜 이렇게 설계됐는가

### 구조 비교

| | 가상머신 (VM) | 컨테이너 (Docker) |
|---|---|---|
| 스택 | 하드웨어 → 호스트OS → 하이퍼바이저 → **게스트 OS** → 앱 | 하드웨어 → 호스트OS → 컨테이너 엔진 → 앱 |
| 커널 | VM마다 별도의 커널 보유 | **호스트 커널을 공유** |
| 격리 방식 | 하드웨어 가상화 (하이퍼바이저) | 커널 기능 활용: **namespace**(격리된 것처럼 보이게) + **cgroup**(자원 사용량 제한) |
| 부팅 속도 | 수십 초~분 (OS 전체 부팅) | 1초 내외 (커널 부팅 과정 없이 프로세스만 fork/exec) |
| 무게 | VM 1개 = OS 전체 (수백 MB~GB) | 컨테이너 1개 = 프로세스 + 얇은 레이어 |

### 왜 이렇게 설계했는가
완전한 하드웨어 가상화 대신 **OS 커널 레벨 격리**를 선택한 이유는 **속도와 밀도(density)**다.
서버 한 대에 컨테이너는 수백 개를 올릴 수 있지만, VM은 그 정도로 못 올린다.
대신 트레이드오프도 있다: 커널을 공유하기 때문에 VM만큼 강한 보안 격리는 아니다
(커널 취약점이 같은 호스트의 모든 컨테이너에 영향을 줄 수 있음).

> **심화 대비**: 이 격리 약점을 보완하는 장치로 seccomp(시스템 콜 필터링),
> capabilities 제한, AppArmor/SELinux가 기본 적용되며, 더 강한 격리가 필요하면
> gVisor·Kata Containers처럼 컨테이너를 경량 샌드박스/VM 안에서 돌리는 런타임도 있다.

### 면접 답변 템플릿
> "컨테이너는 하이퍼바이저처럼 하드웨어를 가상화하는 게 아니라 호스트 커널을 그대로 공유하고,
> 리눅스의 namespace와 cgroup으로 격리만 흉내 낸다. 그래서 `uname -a`를 치면 호스트와
> 완전히 같은 커널 버전이 나온다. 이 설계 덕분에 부팅 과정이 없어 컨테이너는 훨씬 가볍고 빠르지만,
> 커널을 공유하는 만큼 VM 대비 격리 수준은 낮다."

### 직접 확인한 실습 (Windows/WSL2 환경)

```powershell
wsl -d docker-desktop uname -a
docker run --rm ubuntu:22.04 uname -a
```

**실제 실행 결과:**

```
# wsl -d docker-desktop uname -a
Linux docker-desktop 6.6.87.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun 5 18:30:46 UTC 2025 x86_64 Linux

# docker run --rm ubuntu:22.04 uname -a
Linux 07850d6e343f 6.6.87.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun 5 18:30:46 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
```

→ 컨테이너 이름(`07850d6e343f`, 컨테이너 고유 ID)은 다르지만, **커널 버전(`6.6.87.2-microsoft-standard-WSL2`)은 완전히 동일**함을 확인.
(Windows는 리눅스 커널이 없으므로, Docker Desktop이 내부적으로 WSL2라는 경량 리눅스 VM을 띄우고
컨테이너들이 그 커널을 공유하는 구조. 커널 이름에 `microsoft-standard-WSL2`가 붙어있는 것도 WSL2용으로
빌드된 커널이라는 증거)

---

## 2. 이미지(Image)와 레이어 — copy-on-write

### 핵심 개념
- **이미지** = 읽기 전용(immutable) 레이어들의 스택. "이 컨테이너가 무엇을 실행하는가"에 대한 답.
- **컨테이너** = 이미지 위에 얹는 아주 얇은 **쓰기 가능 레이어(writable layer)** 하나.
- 컨테이너 안에서 파일을 수정하면 원본 이미지 레이어는 절대 건드리지 않고, 그 변경사항은
  자신만의 쓰기 레이어에만 기록된다. 이걸 **copy-on-write**라고 부른다.
- 이를 가능하게 하는 실제 기술이 **overlay filesystem**: 이미지 레이어들(lowerdir)과 컨테이너의
  쓰기 레이어(upperdir)를 하나의 통합된 파일 뷰로 합쳐서 보여준다.

### 왜 이렇게 설계했는가 (3가지 이점)
1. **저장 공간 절약**: 이미지는 디스크에 한 벌만 존재. 여러 컨테이너가 참조만 함.
2. **시작 속도**: 부팅 과정이 없어 컨테이너 생성 시 새 프로세스 fork/exec + namespace 부여만 하면 됨.
3. **격리 유지**: overlay filesystem의 copy-on-write 덕분에 한 컨테이너의 변경이 다른 컨테이너나
   이미지에 전혀 영향을 주지 않음.

### 면접 답변 템플릿
> "도커 이미지는 읽기 전용 레이어의 스택이고, 컨테이너는 그 위에 쓰기 가능한 레이어 하나를 얹은 것이다.
> 이미지 레이어는 불변이라 여러 컨테이너가 안전하게 공유할 수 있고, 컨테이너 생성 시 커널 부팅
> 과정이 없어서 빠르며, overlay filesystem의 copy-on-write 덕분에 격리도 유지된다."

### 직접 확인한 실습

```powershell
docker run -d --name container-a ubuntu:22.04 sleep infinity
docker exec container-a bash -c "echo 'A만의 데이터' > /tmp/only-in-a.txt"
docker run -d --name container-b ubuntu:22.04 sleep infinity
docker exec container-b ls /tmp
```

→ `container-b`의 `/tmp`에 `only-in-a.txt`가 없음을 확인. 같은 이미지로 만든 컨테이너라도
각자 독립된 쓰기 레이어를 갖기 때문.

---

## 3. 볼륨(Volume)과 bind mount — 왜 필요한가

### 핵심 개념
컨테이너는 기본적으로 **stateless(휘발성)**로 설계됐다. 컨테이너를 삭제하면 쓰기 레이어도
함께 사라지므로, 그 안의 데이터도 전부 사라진다. 그래서 영구적으로 보존해야 할 데이터
(DB 데이터, 로그 등)는 컨테이너 내부가 아니라 **볼륨(volume)**이라는 별도의 저장 공간에
두고 mount해서 사용한다.

**이미지 vs 볼륨 — 서로 다른 축의 개념**

| | 이미지 (Image) | 볼륨 (Volume) |
|---|---|---|
| 답하는 질문 | 무엇을 실행하는가 | 데이터를 어디에 저장하는가 |
| 내용물 | 앱 코드, 라이브러리, OS 파일 | 순수 데이터 (DB 파일, 로그 등) |
| 성격 | 불변(immutable) | 가변(mutable) |
| 삭제 시 | 컨테이너를 못 띄움 | 컨테이너 삭제해도 안 지워짐 (명시적으로 지워야 함) |

### 용어 주의: volume mount vs bind mount는 서로 다른 마운트 타입
Docker 공식 분류에서 스토리지 마운트는 **volume, bind mount, tmpfs, named pipe** 네 가지로
구분되는 별개 타입이다. `docker inspect`로 확인하면 `-v mydata:/data`는 `"Type": "volume"`,
`-v /host/path:/data`는 `"Type": "bind"`로 나온다.

- **volume**: Docker 데몬이 관리하는 저장 공간 (`/var/lib/docker/volumes/...`). 이름으로 참조.
- **bind mount**: 호스트의 임의 경로를 직접 연결. Docker가 내용을 관리하지 않음.

면접에서 "볼륨은 bind mount다"라고 말하면 틀린 답이 된다. 정확한 표현은
"볼륨도 **내부적으로는 커널의 bind mount 메커니즘**으로 컨테이너 경로에 붙지만,
Docker 용어로는 별개의 마운트 타입"이다.

### 왜 이미지가 바뀌어도 볼륨 데이터는 그대로인가 (마운트의 원리)
컨테이너 실행 시 실제로 일어나는 순서:
1. 이미지의 읽기 전용 레이어 + 컨테이너의 새 쓰기 레이어를 overlay filesystem으로 합쳐서
   컨테이너의 루트 파일시스템(`/`)을 구성한다.
2. `-v 볼륨이름:/경로` 옵션을 보고, **호스트(또는 WSL2)의 실제 디렉토리**
   (`/var/lib/docker/volumes/볼륨이름/_data`)를 그 경로 위에 마운트한다.
   커널 입장에서는 "그 경로에 원래 뭐가 있었든 상관없이, 앞으로 이 경로 접근은
   무조건 저 실제 디렉토리로 보내라"는 지시다.

2번 단계가 1번 단계를 완전히 덮어쓰기 때문에, 컨테이너가 어떤 이미지로 떠 있든 그 경로는
항상 같은 실제 저장소를 가리킨다. 즉 "이미지가 안 바뀌어서"가 아니라 **그 경로 자체가 이미지
레이어 스택과 무관한 별도의 마운트 지점이기 때문**이다.

### 비유 (게임 세이브 파일)
- 게임 클라이언트 파일(exe, 리소스) = 이미지
- 세이브 파일(`Documents/MyGame/Save/`) = 볼륨
- 게임을 패치해도 세이브 파일 경로는 코드에 고정되어 있어 그대로 유지됨 — bind mount와 동일한 원리
- 게임 언인스톨 시 "세이브 파일도 지울까요?" 체크박스 = `docker rm -v` 옵션과 대응

### 면접 답변 템플릿
> "컨테이너는 기본적으로 stateless하게 설계됐기 때문에, 영구 데이터는 volume이라는 별도
> 저장 공간에 두고 mount해서 쓴다. 볼륨은 이미지 레이어 스택과 완전히 별개의 마운트 지점이라,
> 컨테이너 생성 시 도커가 볼륨의 실제 디렉토리를 그 경로 위에 마운트하기 때문에 이미지가
> 바뀌어도 항상 동일한 물리적 파일을 가리킨다. 참고로 Docker 용어에서 volume과 bind mount는
> 별개의 마운트 타입이고, 볼륨은 Docker 데몬이 관리하는 저장소라는 점이 다르다."

### 직접 확인한 실습

```powershell
# 볼륨 없이 → 컨테이너 삭제 시 데이터 소실 확인
docker run -d --name no-volume-test ubuntu:22.04 sleep infinity
docker exec no-volume-test bash -c "echo '중요한 데이터' > /data.txt"
docker rm -f no-volume-test
docker run --rm ubuntu:22.04 cat /data.txt   # → No such file or directory

# 볼륨 사용 → 컨테이너 삭제해도 데이터 유지 확인
docker volume create mydata
docker run -d --name with-volume -v mydata:/data ubuntu:22.04 sleep infinity
docker exec with-volume bash -c "echo '영구 데이터' > /data/save.txt"
docker rm -f with-volume
docker run --rm -v mydata:/data ubuntu:22.04 cat /data/save.txt   # → 영구 데이터

# 이미지를 완전히 바꿔도 유지되는지 확인 (별도 마운트 지점 증명)
docker run --rm -v mydata:/data ubuntu:20.04 cat /data/save.txt   # → 영구 데이터 (동일)
```

**실제 실행 결과 (별도 마운트 지점 증명):**

```
# busybox(완전히 다른 이미지)로 확인
$ docker run --rm -v mydata:/data busybox ls -la /data
total 12
drwxr-xr-x    2 root     root          4096 Jul  8 10:40 .
drwxr-xr-x    1 root     root          4096 Jul  8 10:50 ..
-rw-r--r--    1 root     root            17 Jul  8 10:40 save.txt

# ubuntu:22.04로 저장했던 데이터를 ubuntu:20.04로 읽기
$ docker run --rm -v mydata:/data ubuntu:20.04 cat /data/save.txt
영구 데이터
```

→ `busybox`, `ubuntu:20.04` 모두 원래 데이터를 썼던 `ubuntu:22.04`와 다른 이미지인데도
`save.txt` 내용이 그대로 나옴. 볼륨은 이미지 레이어 스택과 무관한 별도 마운트 지점이라는 것의 직접적 증거.

### `docker rm` vs `docker rm -v` — named volume은 예외
`docker rm -v`의 `-v`는 **anonymous volume(이름 없이 생성된 볼륨)만** 정리 대상이다.
`mydata`처럼 **이름을 직접 지어 만든 named volume**은 `-v`를 줘도 삭제되지 않는다.
이는 실수로 인한 데이터 유실을 막기 위한 도커의 의도적인 안전장치이며, named volume을
지우려면 `docker volume rm 볼륨이름`을 명시적으로 실행해야 한다.

> **면접 답변**: "`docker rm -v`는 anonymous volume만 정리하고, named volume은 데이터
> 손실 방지를 위해 별도의 `docker volume rm` 명령으로만 삭제되도록 설계되어 있다."

---

## 4. docker-compose vs Kubernetes

### compose 예시

```yaml
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    depends_on:
      - db
    environment:
      - DB_HOST=db
  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=secret
    volumes:
      - db_data:/var/lib/mysql
volumes:
  db_data:
```

> **주의**: 위처럼 짧은 문법의 `depends_on`은 **시작 순서만** 보장한다. db 컨테이너가 떴다고
> MySQL이 접속 가능하다는 뜻은 아니므로, "준비 완료"까지 기다리려면 db에 `healthcheck`를 정의하고
> `depends_on: { db: { condition: service_healthy } }` 형태의 긴 문법을 써야 한다.

### 닮은 점 / 다른 점

| | docker-compose | Kubernetes |
|---|---|---|
| 실행 범위 | 단일 호스트 | 여러 노드 |
| 컨테이너 장애 시 | `restart: always/on-failure` 정책으로 단순 재시작은 자동. 단, 프로세스가 살아있는 채 멈춘 상태(hang)는 감지 못 함 | liveness probe 등 헬스체크 기반 자동 감지·재시작 (self-healing) |
| 트래픽 증가 시 | 수동 스케일 (`--scale`) | 조건에 따라 자동 스케일링 (HPA) |
| 노드 장애 시 | 노드 개념 자체 없음 | 다른 노드로 자동 재배치 |

둘 다 "여러 컨테이너를 YAML로 선언하면 알아서 떠 있게 한다"는 선언적(declarative) 방식은 같지만,
쿠버네티스는 여기에 **장애 복구, 오토스케일링, 여러 노드 스케줄링** 같은 오케스트레이션 기능을 더한 것.

### 면접 답변 템플릿
> "docker-compose는 단일 호스트에서 여러 컨테이너를 선언적으로 관리하는 도구고, 쿠버네티스는
> 그 개념을 여러 노드로 확장하면서 장애 복구, 오토스케일링, 로드밸런싱 같은 오케스트레이션
> 기능을 추가한 것이다."

---

## 5. minikube란

쿠버네티스는 원래 여러 대의 서버(노드)를 묶어 운영하도록 설계됐지만, **minikube는 학습/개발용으로
단일 노드에 전체 쿠버네티스 컨트롤 플레인과 워커 기능을 압축해 넣은 로컬 클러스터**다.
`kubectl` 명령어와 YAML 문법은 실제 프로덕션 멀티노드 클러스터와 동일하게 동작하지만,
고가용성이나 노드 장애 시나리오 같은 건 재현할 수 없다.

### 환경 구성 시 참고 (Windows)
- Docker Desktop for Windows는 WSL2 안에서 동작하며, `docker-desktop`이라는 WSL2 배포판을
  사용한다. (과거 버전은 `docker-desktop-data` 배포판을 따로 두고 데이터를 저장했지만,
  최신 버전은 단일 배포판으로 통합됨 — `wsl -l -v`로 확인 결과 이 환경도 `docker-desktop` 하나만 존재)
- 설치: `winget install Kubernetes.minikube`
- 실행: `minikube start` (Docker Desktop이 정상 실행 중이어야 함)
- 확인: `kubectl get nodes`, `kubectl cluster-info`

---

## 오늘 배운 것 전체 흐름 요약

1. 컨테이너는 VM과 달리 호스트 커널을 공유하고 namespace/cgroup으로 격리한다 → 가볍고 빠름
2. 이미지는 불변 레이어의 스택, 컨테이너는 그 위 얇은 쓰기 레이어 (copy-on-write)
3. 컨테이너의 쓰기 레이어도 휘발성이므로, 영구 데이터는 volume으로 분리해 마운트한다
   (volume과 bind mount는 Docker 용어상 별개의 마운트 타입)
4. named volume은 `docker rm -v`로도 삭제되지 않는 안전장치가 있다
5. docker-compose는 단일 호스트용 선언적 다중 컨테이너 관리 (restart 정책으로 단순 재시작은
   자동 가능), K8s는 이를 다중 노드 + 헬스체크 기반 오케스트레이션으로 확장한 것
6. minikube는 학습용 단일 노드 압축 클러스터

**다음 학습 목표**: 쿠버네티스 Pod 개념 — "왜 컨테이너 하나가 아니라 Pod라는 단위를 K8s가
새로 만들었는지" 설계 이유부터 시작.
