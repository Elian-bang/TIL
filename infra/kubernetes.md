# 선언한 상태로 계속 되돌리는 기계

## 30초 요약

- K8s의 본질은 오케스트레이션 기능 목록이 아니라 **하나의 루프**다. "원하는 상태"와 "실제 상태"를 끝없이 비교해 좁힌다
- 그래서 명령형 대신 **선언형**이다. "파드를 3개 띄워라"가 아니라 "항상 3개인 상태"를 선언한다
- **파드가 죽는 건 정상 동작이다.** 노드 재배치·스케일다운·프로브 실패로 언제든 죽고 새로 뜬다. 상태를 파드 안에 두면 안 된다
- **request와 limit은 다른 물건이다.** request는 배치 계산용, limit은 강제 상한이다. 이 둘을 헷갈리면 노드가 과적된다
- **프로브를 잘못 걸면 K8s가 멀쩡한 앱을 계속 죽인다.** 자가 치유가 자가 파괴로 뒤집히는 지점

---

## 원리 — 왜 그런가

### 2-1. 조정 루프가 선언형을 만든다

배포 스크립트로 서버를 관리할 때의 문제는 **"지금 상태가 어떤지 아무도 모른다"는 것**이다. 스크립트는 실행 시점의 동작만 안다. 중간에 프로세스가 죽으면 아무도 되돌리지 않는다.

K8s는 문제를 뒤집는다. 명령 대신 목표를 등록하게 한다.

```
사용자     : "replicas: 3" 을 선언한다 (원하는 상태)
컨트롤러   : 무한 루프
             ├ 실제 상태를 관찰한다 → 파드가 2개다
             ├ 원하는 상태와 비교한다 → 1개 부족
             └ 좁히는 행동을 한다 → 파드 하나 생성
```

**이 루프가 절대 안 끝난다는 게 핵심이다.** 노드가 죽어 파드가 사라져도, 누가 수동으로 지워도, 루프가 다시 3개로 만든다. 자가 치유는 별도 기능이 아니라 이 루프의 부산물이다. 공식 문서의 정의도 같다. *"In Kubernetes, controllers are control loops that watch the state of your cluster, then make or request changes where needed. Each controller tries to move the current cluster state closer to the desired state."*

수렴이 끝나야 정상이라는 것도 아니다. 문서는 오히려 *"potentially, your cluster never reaches a stable state"* 라고 적는다. 안정 상태에 도달하는 것이 목표가 아니라, 컨트롤러가 계속 돌면서 유용한 변경을 낼 수 있으면 그걸로 된 것이다.

> 같은 발상이 [내구성과 복구](../database/basics/durability-and-recovery.md) §2-6에도 있다. DB 크래시 복구가 **멱등해서 도중에 또 죽어도 다시 하면 된다**는 성질이 그것이다. "목표 상태로 수렴시키는 반복 가능한 절차"라는 구조가 같다.

### 2-2. 왜 컨테이너가 아니라 파드인가

배치 단위가 컨테이너가 아니라 파드다. 파드는 컨테이너 하나 이상의 묶음이고, **같은 파드 안의 컨테이너는 네트워크 namespace와 볼륨을 공유한다**([컨테이너](containers.md) §2-2).

즉 같은 파드 안에서는 `localhost`로 서로를 부른다. 이런 단위가 왜 필요한가. 로그 수집기나 프록시처럼 **앱과 반드시 같은 곳에 붙어 있어야 하는 보조 컨테이너**(사이드카)를 표현하기 위해서다.

**파드는 언제든 죽는다.** 이건 장애가 아니라 설계다.

- 노드가 메모리 압박을 받으면 축출된다
- 스케일다운하면 없어진다
- 롤링 업데이트면 통째로 교체된다
- 프로브가 실패하면 재시작된다

그래서 파드 IP는 고정이 아니고, 파드 안 파일시스템은 휘발성이다. **상태를 파드에 두면 안 된다는 결론이 여기서 나온다.** [컨테이너](containers.md) §2-4의 "쓰기 가능 레이어는 사라진다"가 클러스터 규모로 확대된 것이다.

한 가지는 정확히 짚어야 한다. 파드는 고쳐지는 게 아니라 버려진다. 문서 표현으로 파드는 *"not "healed" or repaired but replaced"* 이고, *"Pods are not designed to be restarted on a different node"* 다. 노드가 죽었을 때 그 파드가 다른 노드로 옮겨 가는 게 아니라, 컨트롤러가 멀쩡한 노드에 **새 파드를 만든다.** 이름도 IP도 다른 물건이다.

Service가 그 위에 고정 주소를 얹는다. 파드가 계속 바뀌어도 Service 이름은 안 바뀌고, 그 뒤의 파드 목록을 K8s가 갱신한다.

### 2-3. request와 limit을 헷갈리면 노드가 터진다

이름이 비슷해서 같은 걸로 보이는데 **역할이 완전히 다르다.**

| | request | limit |
|---|---|---|
| 용도 | **스케줄러가 배치를 계산할 때 쓰는 값** | **런타임 강제 상한**(cgroup) |
| 의미 | "최소 이만큼은 보장해 달라" | "이걸 넘으면 제재한다" |
| 초과 시 | (초과 가능) | CPU는 **스로틀링**, 메모리는 **OOM Kill** |

**스케줄러는 limit이 아니라 request의 합으로 노드를 채운다.** 공식 문서 표현이 그대로다. *"the kube-scheduler ensures that for each resource type, the sum of the resource requests of the scheduled containers is less than the capacity of the node"* 그래서 request를 실제보다 작게 잡으면 노드에 파드가 과적되고, 부하가 몰리는 순간 전부 자원 경쟁에 빠진다. 반대로 노드에 여유가 있으면 컨테이너가 request를 넘겨 쓰는 것은 허용된다. request는 보장선이지 상한이 아니다.

CPU와 메모리는 초과 처리 방식이 다르다는 것도 중요하다.

- CPU 초과 → 스로틀링. 죽지 않고 느려진다. 그래서 원인 파악이 어렵다. "응답이 튀는데 에러는 없다"가 이 경우다
- 메모리 초과 → OOM Kill. 즉시 죽는다([컨테이너](containers.md) §2-3)

스로틀링을 거는 주체는 커널이다. 문서는 *"By default, the kubelet uses CFS quota to enforce pod CPU limits"* 라고 적고, 한도 근처에서 *"the kernel will restrict access to the CPU corresponding to the container's limit"* 라고 설명한다. 애플리케이션이 협조할 여지가 없는 하드 리밋이라는 뜻이다. CFS는 정해진 주기마다 쿼터를 새로 채우므로, 짧게 몰아 쓰는 워크로드는 주기가 끝나기 전에 쿼터를 소진하고 남은 시간을 통째로 대기한다. 평균 사용률이 한도의 절반인데도 지연이 튀는 이유가 여기 있다.

CPU limit을 너무 낮게 잡으면 GC 스레드까지 스로틀링당한다. JVM 애플리케이션에서 **stop-the-world가 비정상적으로 길어지는** 원인이 되기도 한다 → [Java](../java/jvm-gc-concurrency.md)

### 2-4. 프로브, 자가 치유가 자가 파괴로 뒤집히는 자리

K8s는 컨테이너가 살아 있는지 스스로 판단하지 않는다. **애플리케이션에게 묻는다.**

| 프로브 | 묻는 것 | 실패하면 |
|---|---|---|
| **liveness** | "살아 있나?" | **컨테이너를 재시작한다** |
| **readiness** | "트래픽 받을 준비 됐나?" | **Service에서 뺀다**(죽이진 않는다) |
| **startup** | "초기화 끝났나?" | **컨테이너를 재시작한다.** 성공 전까지 위 둘은 아예 실행되지 않는다 |

startup 프로브를 "유예 장치"로만 외우면 절반만 아는 것이다. 실패했을 때의 동작은 liveness와 같다. *"If the startup probe fails, the kubelet kills the container, and the container is subjected to its restart policy."* 유예는 성공하기 전까지의 성질이고, `failureThreshold`를 다 쓰면 그때부터는 재시작 루프다.

readiness 실패의 실체도 적어 둔다. Service 설정을 건드리는 게 아니라 **EndpointSlice에서 파드 IP가 빠진다.** 파드는 그대로 살아 있고 트래픽만 안 온다.

이 셋을 헷갈리면 사고가 난다. 가장 흔한 실패는 **liveness에 DB 연결 확인을 넣는 것**이다.

```
DB가 잠깐 느려진다
  → liveness 실패
  → K8s가 모든 파드를 재시작
  → 재시작한 파드들이 동시에 커넥션 풀을 새로 채운다
  → DB가 더 느려진다
  → 다시 liveness 실패 …
```

**멀쩡한 앱을 K8s가 계속 죽이는 상태**가 된다. liveness는 "재시작하면 나아지는 문제"만 봐야 한다. 데드락에 빠진 자기 자신 정도가 그렇다. 외부 의존성은 readiness의 몫이다. 트래픽만 빼면 되지 죽일 이유가 없다. 공식 문서의 역할 분담도 같다. *"The liveness probe passes when the app itself is healthy, but the readiness probe additionally checks that each required back-end service is available."*

**startup 프로브가 없으면 무거운 앱이 영원히 못 뜬다.** 기동에 60초 걸리는 JVM 앱에 liveness를 10초로 걸면, 기동 중에 실패 판정을 받아 재시작되고 그걸 무한 반복한다.

기본값을 모르면 이 계산을 못 한다. `periodSeconds` 10초, `timeoutSeconds` **1초**, `failureThreshold` 3, `successThreshold` 1, `initialDelaySeconds` 0이다. 아무것도 안 적으면 판정까지 약 30초이고, 응답이 1초를 넘기면 그 회차는 실패로 센다. 헬스 엔드포인트가 DB를 한 번 찔러 보는 순간 1초는 쉽게 넘는다.

> §2-1의 조정 루프는 강력한 만큼, 방향을 잘못 잡으면 그만큼 강하게 틀린 곳으로 수렴한다. K8s 장애의 상당수가 "K8s가 고장 났다"보다 **"틀린 목표를 성실히 달성했다"에 가깝다.**

---

**다음으로 읽을 것**

- 파드 안에서 격리가 실제로 어떻게 되는가 → [컨테이너](containers.md)
- OOM Kill의 실체 → [컨테이너](containers.md) §2-3, [메모리와 페이징](../os/memory-and-paging.md)
- 재시작 폭풍이 왜 DB를 더 죽이는가 → [대용량 처리](../system-design/high-throughput.md) §2-4 서킷 브레이커

---

> **기준 버전**: kubernetes.io 현행 문서(1.3x 계열)로 대조
> **확인한 출처** (전부 kubernetes.io):
> - [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/) — 조정 루프 정의 원문 전체, 컨트롤러가 `spec`(원하는 상태)을 추적한다는 서술, **안정 상태에 도달하지 않아도 된다**는 원문(*"potentially, your cluster never reaches a stable state"*)
> - [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) — 최소 배포 단위 정의와 *"shared storage and network resources"*, 같은 파드 안에서 IP·네트워크 네임스페이스를 공유하고 `localhost`로 통신한다는 서술, **파드는 고쳐지지 않고 교체된다**(*"not "healed" or repaired but replaced"*), *"Pods are not designed to be restarted on a different node"*, 축출·노드 장애 등 파드가 사라지는 조건
> - [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) — **스케줄러가 request 합으로 배치한다**는 원문(§2-3의 기존 미확인 항목 해소), *"cpu limits are enforced by CPU throttling ... Containers may not use more CPU than is specified in their cpu limit."*, *"memory limits are enforced by the kernel with out of memory (OOM) kills."*, **여유가 있으면 request 초과 사용이 허용된다**는 서술, CPU 1단위 = 1코어 · millicpu 표기
> - [Control CPU Management Policies on the Node](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/) — *"By default, the kubelet uses CFS quota to enforce pod CPU limits."*, 스로틀링 여부에 따라 워크로드가 코어를 옮겨 다닌다는 서술
> - [kube-scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/) — feasible node 정의, filtering·scoring 2단계, `PodFitsResources`가 **request 기준으로 거른다**는 서술, 고려 요소에 *"individual and collective resource requirements"* 포함
> - [Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/) · [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) — 세 프로브의 실패 동작 원문(**startup 실패도 컨테이너 재시작**이라는 문장 포함. §2-4 표를 이 근거로 고쳤다), readiness 실패 시 **EndpointSlice에서 파드 IP 제거**, liveness는 앱 자신 · readiness는 백엔드 의존성이라는 역할 분담 원문, **기본값**(`initialDelaySeconds` 0 · `periodSeconds` 10 · `timeoutSeconds` 1 · `successThreshold` 1 · `failureThreshold` 3)
> - [Assign Memory Resources](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/) — 한도 초과 컨테이너의 `reason: OOMKilled` · `exitCode: 137` 실제 출력
> **미확인**: **CFS 주기(`cpu.cfs_period_us` 기본 100ms)와 쿼터 계산식** — 커널 쪽 문서라 대조하지 않았다. §2-3 끝의 "주기마다 쿼터를 새로 채우므로 버스트성 워크로드가 남은 시간을 대기한다"는 서술도 **kubernetes.io가 CFS 쿼터를 쓴다는 사실까지만 확인했고, 이 동작을 설명하는 원문은 확보하지 못했다** · request가 cgroup의 `cpu.shares`/`cpu.weight`로 번역된다는 서술은 널리 알려져 있으나 **해당 절이 잘려 원문을 확인하지 못해 본문에 넣지 않았다** · §2-4의 "기동 60초 JVM 앱에 liveness 10초" 예시는 **설명용 수치이고 실측이 아니다** · §2-4의 DB 재시작 폭풍 시나리오는 **문서가 경고하는 역할 분담에서 유도한 것이지 문서에 그 사례가 실려 있는 것은 아니다** · CPU를 압축 가능(compressible) 자원, 메모리를 압축 불가로 부르는 분류는 **현행 문서에서 그 표현을 찾지 못했다**
> **미작성**: Deployment/StatefulSet/DaemonSet 차이 · 롤링 업데이트 전략 · ConfigMap·Secret · Ingress · HPA · QoS 클래스(Guaranteed/Burstable/BestEffort)와 축출 순서
