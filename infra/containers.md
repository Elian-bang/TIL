# 가상머신이 아니라 격리된 프로세스다

## 30초 요약

- 컨테이너는 가벼운 VM이 아니다. 호스트 커널을 그대로 쓰는 **그냥 프로세스**이고, 두 가지 커널 기능으로 격리돼 있을 뿐이다
- **namespace는 시야를, cgroup은 자원을 자른다.** namespace가 PID·네트워크·마운트를, cgroup이 CPU·메모리를 맡는다
- 그래서 부팅이 없다. VM은 OS를 띄우지만 컨테이너는 **프로세스 하나를 실행할 뿐이다**
- 이미지는 레이어의 겹침이다. 공통 레이어를 공유하므로 **이미지 100개가 디스크 100배를 먹지 않는다**
- 가장 자주 터지는 함정은 **cgroup 메모리 제한을 런타임이 못 보고 호스트 전체 메모리 기준으로 동작하는 것**이다

---

## 원리 — 왜 그런가

### 2-1. VM과 무엇이 다른가

```
가상머신                          컨테이너
┌──────────────┐                 ┌──────────────┐
│  앱          │                 │  앱          │
│  게스트 OS   │  ← 커널이 따로  │              │  ← 커널 없음
├──────────────┤                 ├──────────────┤
│  하이퍼바이저 │                 │  컨테이너 런타임 │
├──────────────┤                 ├──────────────┤
│  호스트 OS   │                 │  호스트 OS(커널 공유) │
└──────────────┘                 └──────────────┘
```

**핵심 차이는 커널이다.** VM은 게스트 OS를 통째로 띄우므로 부팅이 필요하고 수백 MB~GB를 먹는다. 컨테이너는 호스트 커널에 시스템 콜을 그대로 날리는 프로세스다. 부팅할 게 없으니 시작이 밀리초다.

대가는 격리 강도로 치른다. 커널을 공유하므로 **커널 취약점은 컨테이너 경계를 넘는다.** VM은 하이퍼바이저가 한 겹 더 막아 준다. "컨테이너는 보안 경계가 아니다"라는 말이 여기서 나온다.

커널이 하나라는 사실은 제약도 만든다. 리눅스 호스트에서 윈도우 컨테이너를 못 돌린다. macOS에서 Docker가 실제로는 **리눅스 VM을 하나 띄워 두고 그 안에서** 컨테이너를 돌리는 이유도 이것이다.

### 2-2. namespace, 무엇을 볼 수 있는가

namespace는 "이 프로세스가 보는 세계"를 잘라 낸다.

| namespace | 격리하는 것 | 결과 |
|---|---|---|
| PID | 프로세스 목록 | 컨테이너 안에서 자기가 **PID 1**로 보인다 |
| Network | 네트워크 인터페이스·포트 | 컨테이너마다 자기 IP. **포트 8080을 여럿이 동시에** 쓸 수 있다 |
| Mount | 파일시스템 트리 | 자기 루트만 보인다 |
| UTS | 호스트명 | 컨테이너마다 다른 호스트명 |
| IPC | 공유 메모리·세마포어 | 프로세스 간 통신 격리 |
| User | UID/GID 매핑 | 컨테이너 안의 root ≠ 호스트의 root |

위 여섯 개가 컨테이너에서 실제로 쓰이는 것들이고, man page가 정의하는 네임스페이스는 여덟 종이다. 표에 없는 둘은 cgroup 루트 디렉터리를 가리는 **Cgroup**(리눅스 4.6)과 부팅·모노토닉 시계를 가리는 **Time**(5.6)이다.

호스트에서 `ps`를 치면 컨테이너 안 프로세스가 그냥 보인다. 숨겨진 게 아니고 컨테이너 쪽에서만 안 보이는 것이다. 격리는 일방향이다.

> PID 1이라는 자리가 함정을 만든다. 네임스페이스의 첫 프로세스가 그 안의 init이 되고, *"becomes the parent of any child processes that are orphaned"*, 즉 **고아 프로세스를 거두는 역할**을 넘겨받는다. 애플리케이션이 그걸 안 하면 좀비 프로세스가 쌓인다.
> 시그널 쪽은 흔히 설명되는 것과 원리가 다르다. 앱이 `SIGTERM`을 무시하는 게 아니라 **커널이 전달하지 않는다.** man page의 표현은 *"Only signals for which the 'init' process has established a signal handler can be sent to the 'init' process by other members of the PID namespace."* 핸들러를 등록하지 않으면 기본 동작인 종료조차 일어나지 않는다. 그래서 graceful shutdown이 안 되고, 유예 시간이 지난 뒤 `SIGKILL`로 끝난다. `SIGKILL`은 상위 네임스페이스에서 보내면 강제로 전달되고 잡을 수도 없기 때문이다. 배포할 때 종료가 늘 강제 kill로 끝난다면 여기를 의심한다.

### 2-3. cgroup, 얼마나 쓸 수 있는가

namespace가 시야를 자른다면 **cgroup은 자원을 자른다.** man page 정의로 *"allow processes to be organized into hierarchical groups whose usage of various types of resources can then be limited and monitored"*, 즉 CPU·메모리·디스크 I/O·네트워크 대역을 그룹 단위로 제한하고 계측한다.

v1과 v2의 차이는 한 줄이면 된다. v1은 컨트롤러(cpu, memory, blkio …)마다 서로 다른 계층을 마운트할 수 있어서 한 프로세스가 컨트롤러별로 다른 그룹에 속할 수 있었다. v2는 **단일 통합 계층**이다. 트리 하나에 프로세스를 한 번만 배치하면 모든 컨트롤러가 거기 따라붙는다.

메모리 제한을 넘으면 커널이 그 컨테이너 안의 프로세스를 죽인다. 유명한 **OOM Killed**(종료 코드 137)다. 애플리케이션 예외가 아니고 커널이 밖에서 죽인 것이라, 로그에 아무것도 안 남는 게 특징이다.

**여기가 이 저장소에서 가장 실무적인 연결점이다.**

```
컨테이너 메모리 제한 : 2GB (cgroup)
JVM 이 인식한 메모리 : 64GB (호스트 전체)      ← 옛 JVM은 cgroup을 못 봤다
JVM 기본 최대 힙     : 물리 메모리의 1/4 = 16GB
결과                 : 힙을 2GB 넘게 잡는 순간 OOM Killed
```

위 3행의 1/4은 어림이 아니라 JVM 인체공학의 기본값이다. GC 튜닝 가이드가 *"Maximum heap size of 1/4 of physical memory"* 라고 적는다. 그 "physical memory"가 무엇이냐를 바꾼 것이 `-XX:+UseContainerSupport`이고, `java` man page 표현으로 *"The default for this flag is true, and container support is enabled by default."* 리눅스 한정이다.

JVM은 자기가 잘못한 게 없다고 생각하고, 커널은 조용히 죽인다. **"로컬에서는 되는데 컨테이너에서만 죽는다"의 전형**이다. 지금은 JVM이 cgroup 제한을 인식하지만, 원리를 모르면 여전히 같은 함정에 빠진다. 힙 외에 메타스페이스·스레드 스택·네이티브 버퍼가 전부 cgroup 한도 안에서 계산되기 때문이다. → [Java](../java/jvm-gc-concurrency.md)

[메모리와 페이징](../os/memory-and-paging.md) §2-5의 스래싱도 여기서 다시 나온다. cgroup 메모리를 빡빡하게 잡으면 컨테이너 안에서 페이지 교체가 폭증한다. **CPU는 노는데 느린 상태가 컨테이너 단위로 재현된다.**

### 2-4. 이미지는 레이어의 겹침이다

이미지는 통짜 디스크 이미지가 아니라 **읽기 전용 레이어를 겹쳐 쌓은 것**이다.

```
[ 레이어 4 ] 내 애플리케이션 jar          ← 자주 바뀜
[ 레이어 3 ] 의존 라이브러리
[ 레이어 2 ] JDK
[ 레이어 1 ] 베이스 OS 파일들             ← 거의 안 바뀜
─────────────
[ 쓰기 가능 레이어 ]  ← 컨테이너 실행 중 생기는 변경. 컨테이너 삭제 시 사라짐
```

공통 레이어는 여러 이미지가 공유한다. 같은 JDK 베이스를 쓰는 서비스 20개가 JDK 레이어를 20벌 갖지 않는다.

여기서 Dockerfile 작성 규칙이 유도된다. Docker 문서의 표현이 그대로다. *"If a layer changes, all other layers that come after it are also affected."* 명령 하나가 바뀌면 그 앞은 캐시가 살고 뒤는 전부 다시 만들어진다. 그래서 **자주 바뀌는 것을 뒤에** 둔다.

```dockerfile
COPY build.gradle .          # 거의 안 바뀜 → 위
RUN ./gradlew dependencies   # 의존성 캐시가 살아남는다
COPY src/ .                  # 자주 바뀜 → 아래
RUN ./gradlew build
```

순서를 뒤집으면 소스 한 줄 고칠 때마다 의존성을 전부 다시 받는다. "빌드가 왜 이렇게 느리지"의 흔한 원인이다.

> 쓰기 가능 레이어가 컨테이너 삭제와 함께 사라진다는 게 상태 관리의 출발점이다. 로그나 업로드 파일을 컨테이너 안에 쓰면 재배포 때 증발한다. 그래서 볼륨이나 외부 스토리지로 뺀다.

---

**다음으로 읽을 것**

- cgroup 제한이 왜 스래싱을 부르는가 → [메모리와 페이징](../os/memory-and-paging.md) §2-5
- 컨테이너를 여러 대에 배치하는 문제 → [Kubernetes](kubernetes.md)
- JVM 힙과 컨테이너 한도의 관계 → [Java](../java/jvm-gc-concurrency.md)

---

> **기준 버전**: 리눅스 컨테이너(Docker/containerd) 기준. cgroup v2 가정. JVM 서술은 JDK 21 문서 기준
> **확인한 출처**:
> - [namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html) — 정의 원문(*"A namespace wraps a global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource."*), **네임스페이스 8종**과 도입 커널(IPC·Network·UTS 3.0, Mount·PID·User 3.8, **Cgroup 4.6**, **Time 5.6**)
> - [pid_namespaces(7)](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html) — 첫 프로세스가 PID 1이자 네임스페이스의 init이 된다는 점, **고아 프로세스를 거두는 역할** 원문, **핸들러를 등록한 시그널만 전달된다**는 원문(§2-2의 "`SIGTERM`을 무시한다"를 이 근거로 고쳤다), `SIGKILL`·`SIGSTOP`은 상위 네임스페이스에서 강제 전달되며 잡을 수 없다는 서술
> - [cgroups(7)](https://man7.org/linux/man-pages/man7/cgroups.7.html) — 정의 원문(*"allow processes to be organized into hierarchical groups whose usage of various types of resources can then be limited and monitored"*), v1 컨트롤러(cpu·cpuacct·memory·blkio·net_cls), **v1은 컨트롤러마다 다른 계층을 마운트할 수 있고 v2는 단일 통합 계층**이라는 차이
> - [Docker storage drivers](https://docs.docker.com/engine/storage/drivers/) — *"A Docker image is built up from a series of layers. Each layer represents an instruction in the image's Dockerfile. Each layer except the very last one is read-only."*, 컨테이너 생성 시 쓰기 가능 레이어가 얹히고 **삭제 시 함께 사라지며 이미지는 그대로**라는 원문, 공유 레이어가 `/var/lib/docker/`에 한 번만 저장된다는 서술 / [Build cache](https://docs.docker.com/build/cache/) — *"If a layer changes, all other layers that come after it are also affected."*
> - [java man page (JDK 21)](https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html) — `-XX:+UseContainerSupport`의 **기본값 true**와 컨테이너 메모리·프로세서 수 자동 감지 원문, `-XX:ActiveProcessorCount` / [GC 튜닝 가이드 Ergonomics (JDK 17)](https://docs.oracle.com/en/java/javase/17/gctuning/ergonomics.html) — **기본 최대 힙 = 물리 메모리의 1/4**, 초기 힙 1/64
> - [Assign Memory Resources (Kubernetes)](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/) — 메모리 한도 초과 컨테이너의 실제 상태 출력에서 **`reason: OOMKilled` · `exitCode: 137`**
> **미확인**: **JVM의 cgroup 인식 도입 버전**(8u191·10 계열로 알려져 있으나 릴리스 노트를 대조하지 못했다. 다만 [JDK 11 man page](https://docs.oracle.com/en/java/javase/11/tools/java.html)는 *"only available on Linux x64 platforms"*, JDK 21은 *"Linux only"* 로 문구가 달라 지원 범위가 넓어진 것은 확인된다) · **`-XX:MaxRAMPercentage`의 기본값**(man page가 길어 해당 항목까지 열지 못했다. 그래서 본문에서 이 플래그를 빼고 1/4 기본값만 남겼다) · **137 = 128 + SIGKILL(9)** 이라는 유도(관례이고 k8s 문서에서 확인한 것은 137이라는 값 자체다) · JVM이 cgroup **v1과 v2를 각각 언제부터 읽는지** · §2-1의 "시작이 밀리초"·"수백 MB~GB" 같은 수치는 **자릿수 감각용이며 실측이 아니다**
> **미작성**: OCI 이미지·런타임 스펙 · rootless 컨테이너와 user namespace 매핑 실무 · seccomp·AppArmor·capabilities · 멀티스테이지 빌드와 distroless · containerd·CRI-O의 역할 분담
