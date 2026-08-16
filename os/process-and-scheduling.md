# 컨텍스트 스위칭은 왜 비싼가

## 30초 요약

- 프로세스와 스레드를 가르는 기준은 하나, **메모리 공간을 분리하는가**다. 분리하면 장애가 격리되지만 통신이 비싸고, 공유하면 통신이 싸지만 동기화가 필요하다
- 컨텍스트 스위칭 비용의 실체는 레지스터 저장·복원이 아니라 **CPU 캐시가 식는 것**이다
- 스케줄러가 전환을 일으키는 계기는 넷뿐이다. **더 높은 우선순위가 준비됨 · I/O 블로킹 · 타임 슬라이스 소진 · 자발적 양보**
- **타임 슬라이스는 응답성과 처리량의 거래다.** 짧을수록 골고루 돌지만 전환 횟수가 늘어 실제 일하는 시간이 줄어든다
- 같은 부등호가 이 저장소 곳곳에 반복된다: **동시성을 늘려도 실제 처리 자원이 안 늘면 대기만 늘어난다**

---

## 원리 — 왜 그런가

### 2-1. 프로세스 vs 스레드, 그리고 Virtual Thread의 배경

| | 프로세스 | 스레드 |
|---|---|---|
| 메모리 | **독립** | **힙·코드 영역 공유**, 스택은 따로 |
| 통신 | IPC 필요 (비쌈) | 메모리 공유 (싸지만 **동기화 필요**) |
| 컨텍스트 스위칭 | 비쌈 (메모리 맵 교체) | 상대적으로 쌈 |
| 장애 격리 | **하나가 죽어도 다른 건 산다** | 하나가 죽으면 프로세스 전체 |

**컨텍스트 스위칭 비용**: 레지스터 저장·복원 + CPU 캐시 무효화. 캐시 미스가 늘어난다.
→ **스레드를 많이 만든다고 빨라지지 않는 이유**이고, Virtual Thread가 나온 배경이다 (JVM 안에서 전환하니 OS 컨텍스트 스위칭이 없다 → [Java](../java/jvm-gc-concurrency.md) §2-6).
→ 커넥션 풀을 크게 잡으면 안 되는 이유도 같다 (→ [트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-10).

같은 논리가 DB에서도 반복된다. PostgreSQL이 커넥션마다 프로세스를 띄우고 MySQL이 스레드를 띄우는 차이가, PG에서 PgBouncer 같은 커넥션 풀러가 사실상 필수가 되는 이유다(→ [PostgreSQL 복제와 운영](../postgresql/replication-and-ops.md)).

### 2-2. 무엇을 공유하고 무엇을 따로 갖는가

위 표의 "메모리 공유"는 뭉뚱그린 말이다. `pthreads(7)`이 경계를 정확히 그어 준다.

| 프로세스 전체가 공유 | 스레드마다 따로 |
|---|---|
| 전역 메모리(**data·heap 세그먼트**) | **스택** (지역 변수) |
| PID, 부모 PID | 스레드 ID |
| **열린 파일 디스크립터** | 시그널 마스크 |
| 시그널 처리 방식(disposition) | `errno` |
| 현재 디렉터리, 리소스 한도 | **스케줄링 정책·우선순위**, CPU 어피니티 |

읽을 때 두 줄이 실무를 가른다. 먼저 **"열린 파일 디스크립터가 공유"**. 발송 워커 스레드 하나가 소켓을 닫으면 그 fd는 프로세스 전체에서 닫힌다. 스레드별로 격리되지 않는다. 그리고 "스케줄링 정책·우선순위가 스레드마다 따로". 우선순위는 프로세스가 아니라 스레드 단위로 붙으니, §2-8의 `nice`를 배치 워커 스레드에만 따로 줄 수 있다.

### 2-3. 프로세스 상태 전이: 워커는 대부분 안 돌고 있다

```
       생성 ──→ 준비(Ready) ──디스패치──→ 실행(Running) ──→ 종료
                    ↑                          │
                    │                          │ I/O 요청 · 락 대기
        완료 이벤트 └──── 대기(Blocked) ←──────┘
                    ↑                          │
                    └────── 선점(§2-4) ────────┘
```

**핵심은 준비와 실행이 다르다는 것이다.** "실행 가능한데 CPU를 못 받고 줄 서 있는" 상태가 준비다. 이 둘을 구분하지 않으면 "CPU가 100%인데 왜 안 빠르지"와 "런큐가 길어 못 도는 것"을 못 가른다.

리눅스는 `ps`에서 이렇게 보여준다.

| 코드 | 의미 (`ps(1)` 원문) | 발송 시스템에서 |
|---|---|---|
| `R` | running or runnable (on run queue) | 실행 중과 준비를 한 글자로 묶었다 |
| `S` | interruptible sleep (waiting for an event to complete) | 채널사 API 응답 대기. 정상 상태 |
| `D` | uninterruptible sleep (usually I/O) | 디스크·NFS 대기. 여기 몰려 있으면 I/O 병목 |
| `T` | stopped by job control signal | |
| `Z` | defunct("zombie"), terminated but not reaped by its parent | 부모가 종료 코드를 안 거둬간 것 |

`R`이 실행과 준비를 함께 가리킨다는 게 로드 애버리지를 읽을 때의 함정이다. 로드가 높다는 건 "CPU가 바쁘다"가 아니라 "돌 수 있는데 순서를 기다리는 것들이 많다"에 가깝다.

워커 100개를 띄워도 `R`은 코어 수를 넘어 동시에 돌 수 없다. 나머지는 `S`(채널사 응답 대기)거나 준비 큐다. 발송 스레드를 늘려서 얻는 건 "`S`로 놀고 있는 시간을 다른 스레드가 채우는 것"뿐이고, 이미 CPU가 포화면 늘려도 준비 큐만 길어진다(§2-10).

### 2-4. PCB와 컨텍스트 스위치, 진짜 비용은 어디인가

**PCB(Process Control Block)**: 커널이 프로세스마다 들고 있는 관리 구조체. PID·상태·프로그램 카운터·레지스터 값·메모리 맵 포인터·열린 파일 목록·스케줄링 정보가 들어간다. 전환이란 현재 것을 PCB에 밀어 넣고 다음 것을 PCB에서 꺼내오는 일이다.

그런데 이 저장·복원 자체는 레지스터 몇십 개 복사라 그렇게 비싸지 않다. **비싼 건 그다음이다.** 전환 직후 새 프로세스가 건드리는 주소는 캐시에 없어서 미스가 줄줄이 나고, CPU를 받았는데도 메모리를 기다린다. §2-1이 "캐시가 무효화된다"고 한 게 이것이다.

그리고 프로세스 전환은 스레드 전환보다 한 겹 더 비싸다. 주소 공간이 바뀌므로 페이지 테이블을 갈아 끼워야 하는데([메모리와 페이징](memory-and-paging.md) §2-1), 스레드끼리는 §2-2대로 주소 공간이 같아서 그 단계가 통째로 없다.

> 전환 비용이 정확히 얼마인지는 CPU·워크로드에 따라 크게 다르다. 중요한 건 절대값이 아니라 "임계 구역보다 비싼가"다. 이 비교가 [동시성 원시타입](concurrency-primitives.md) §2-6에서 스핀락이냐 블로킹 락이냐를 가른다.

### 2-5. 스케줄러는 언제 끼어드나

`sched(7)`은 리눅스의 전제를 한 문장으로 못 박는다.

> "All scheduling is preemptive: if a thread with a higher static priority becomes ready to run, the currently running thread will be preempted and returned to the wait list for its static priority level."

전환을 일으키는 계기는 넷이다.

| 계기 | 성격 | `sched(7)` 근거 |
|---|---|---|
| **더 높은 우선순위가 준비됨** | 선점 | 위 인용 |
| **I/O로 블로킹** | 자발 | "A SCHED_FIFO thread runs until either it is blocked by an I/O request" |
| **타임 슬라이스 소진** | 선점 | SCHED_RR: 퀀텀만큼 돌면 "put at the end of the list" |
| **명시적 양보** | 자발 | `sched_yield(2)`의 "will be put at the end of the list" |

비선점 정책은 앞의 두 선점 계기가 없는 것이다. `SCHED_FIFO`가 그 예다. `sched(7)`은 `SCHED_RR`을 "a simple enhancement of SCHED_FIFO"로 소개하며 차이를 "각 스레드가 최대 퀀텀 동안만 돈다"로 설명한다. 뒤집으면 FIFO에는 퀀텀이 없다. 같은 우선순위끼리는 스스로 블로킹하거나 양보할 때까지 CPU를 안 놓는다.

이게 왜 중요한가. **비선점 정책에서 무한 루프 하나는 그 우선순위 아래를 전부 멈춰 세운다.** 그래서 커널이 §2-7의 안전장치를 따로 둔다.

### 2-6. 스케줄링 알고리즘 넷은 각각 무엇을 포기했나

| 알고리즘 | 규칙 | 무엇을 포기했나 |
|---|---|---|
| **FCFS** | 도착 순 | 긴 작업 하나가 뒤를 다 막는다(**콘보이 효과**). 응답성 없음 |
| **SJF / SRTF** | 남은 실행 시간이 짧은 것부터 | 평균 대기 시간은 최소인데 실행 시간을 미리 알아야 한다. 짧은 작업이 계속 오면 긴 작업은 **기아** |
| **RR** | 퀀텀만큼 돌리고 큐 뒤로 | 응답성은 얻는다. 대신 전환 횟수가 늘고 모든 작업을 똑같이 대한다(짧은 작업이 손해) |
| **MLFQ** | 우선순위 큐 여러 개. 퀀텀을 다 쓰면 강등, I/O로 일찍 놓으면 유지 | 실행 시간을 몰라도 SJF를 흉내낸다. 과거 행동으로 추정한다. 대신 강등만 있으면 기아 → §2-7의 승격이 필수 |

**SJF는 [메모리와 페이징](memory-and-paging.md) §2-3의 OPT와 정확히 같은 자리에 있다.** 최적이지만 미래를 알아야 해서 구현할 수 없고, 평가 기준선으로만 쓴다. OS는 두 곳에서 같은 벽을 만나 같은 방식으로 우회한다. 과거 행동으로 미래를 근사하는 것이다(페이지 교체는 참조 비트, 스케줄러는 큐 강등).

MLFQ의 착상이 발송 워커 설계와 겹친다. I/O를 기다리며 CPU를 일찍 놓는 발송 워커는 응답성 우대를 받고, 템플릿을 대량 렌더링하며 퀀텀을 다 쓰는 배치 작업은 아래 큐로 내려간다. 우선순위를 사람이 붙이는 대신 관측된 행동으로 스스로 갈리게 하는 것이다.

> 이 넷은 교과서 모델이지 리눅스 구현이 아니다. 리눅스가 실제로 무엇을 쓰는지는 §2-8.

### 2-7. 커널은 기아를 어떻게 막아 뒀나

**기아(starvation)**: 낮은 우선순위가 영원히 차례를 못 받는 상태. 데드락과는 다르다. 데드락은 아무도 못 가지만([트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-7), 기아는 시스템은 잘 도는데 특정 작업만 안 도는 것이라 모니터링에 안 잡힌다.

**에이징(aging)**: 기다린 시간에 비례해 우선순위를 올려 언젠가는 반드시 차례가 오게 하는 처방. 리눅스에 실제로 들어 있다.

`sched(7)`의 `SCHED_OTHER` 설명:

> "The dynamic priority is based on the nice value ... and is increased for each time quantum the thread is ready to run, but denied to run."

**"준비됐는데 못 돈 만큼 올라간다."** 이 한 줄이 에이징 그 자체다.

CFS는 자료구조로 같은 보장을 만든다. "giving a chance for every task to become the 'leftmost task' and thus get on the CPU within a deterministic amount of time."

그리고 §2-5가 예고한 안전장치. 실시간 스레드가 무한 루프를 돌면 일반 스레드는 영원히 못 돈다. 그래서 커널이 실시간 정책의 CPU 지분에 상한을 건다.

| `/proc/sys/kernel/` | 의미 | 기본값 |
|---|---|---|
| `sched_rt_period_us` | 100% CPU 대역폭에 해당하는 주기 | 1,000,000 |
| `sched_rt_runtime_us` | 그중 실시간·데드라인이 쓸 수 있는 몫 | 950,000 |

5%를 비실시간에 강제로 남긴다. 실시간 태스크가 폭주해도 로그인 셸이 살아 있는 이유다. 우선순위 체계를 만들면 기아가 따라오고, 기아를 막는 장치를 같이 넣어야 그 체계가 운영 가능해진다.

> 기아의 또 다른 모양이 우선순위 역전이다. 높은 우선순위가 낮은 쪽이 쥔 락을 기다리는 사이 중간 우선순위가 낮은 쪽을 계속 밀어내면 높은 쪽이 멈춘다. POSIX의 처방은 락을 쥔 쪽의 우선순위를 끌어올리는 것이고(`PTHREAD_PRIO_INHERIT`), 그러려면 락에 주인이 있어야 한다 → [동시성 원시타입](concurrency-primitives.md) §2-4.

### 2-8. 리눅스는 실제로 무엇을 쓰나

`sched(7)`이 정책을 이렇게 나눈다.

| 정책 | 정적 우선순위 | 성격 |
|---|---|---|
| `SCHED_OTHER`(=`SCHED_NORMAL`) | **0만 가능** | 기본 시분할. `nice`로 조절 |
| `SCHED_BATCH` | 0만 가능 | "the scheduler will always assume that the thread is CPU-intensive" |
| `SCHED_IDLE` | 0만 가능, **nice 무시** | nice +19보다도 아래 |
| `SCHED_FIFO` / `SCHED_RR` | **1~99** | "real-time threads always have higher priority than normal threads" |
| `SCHED_DEADLINE` (3.14+) | — | "the highest priority (user controllable) threads in the system" |

일반 애플리케이션은 사실상 전부 `SCHED_OTHER`, 정적 우선순위 0이다. §2-6의 알고리즘 표는 여기에 적용되지 않는다. 이 정책 안에서 순서를 정하는 게 CFS다.

커널 문서는 CFS가 "models an 'ideal, precise multi-tasking CPU'"라고 적는다. 모든 태스크가 `1/N` 속도로 동시에 도는 이상적인 CPU를 흉내 낸다는 뜻이다. 도구는 vruntime("actual runtime normalized to the total number of running tasks")이고, 규칙은 하나다. "it always tries to run the task with the smallest `p->se.vruntime` value (i.e., the task which executed least so far)." 레드블랙 트리에 vruntime 순으로 꽂아 두고 맨 왼쪽을 뽑는다.

여기서 §2-9로 이어지는 문장이 나온다.

> "The CFS scheduler has no notion of 'timeslices' in the way the previous scheduler had."

**고정 퀀텀이 없다.** "몇 ms씩 준다"가 아니라 "가장 덜 쓴 놈을 준다"로 문제를 바꿔 버렸다. 그래서 §2-9의 슬라이스 튜닝 논의는 리눅스 일반 정책에서는 직접 적용되지 않는다.

실무에서 만지는 손잡이는 `nice` 하나다. `-20`(높음)~`+19`(낮음)이고, `sched(7)`이 효과를 수치로 준다. "each unit of difference in the nice values of two processes results in a factor of 1.25 in the degree to which the scheduler favors the higher priority process." 5단계 차이면 약 3배(1.25⁵ ≈ 3.05)다.

API 서버와 야간 대량 발송 배치가 같은 장비에 있다면 배치 쪽 nice를 올려(우선순위를 낮춰) 실시간 요청의 응답 시간을 지키는 게 첫 수다. `SCHED_BATCH`나 `SCHED_IDLE`은 더 강한 처방이다.

### 2-9. 타임 슬라이스가 짧으면 왜 오히려 느려지나

퀀텀을 `T`, 전환 비용을 `C`라 하면 실제 일하는 비율은 `T / (T + C)`다.

```
T = 100ms, C = 1ms  →  99%가 일
T =  10ms, C = 1ms  →  91%가 일
T =   2ms, C = 1ms  →  67%가 일     ← 3분의 1을 전환에 쓴다
```

| | 짧은 슬라이스 | 긴 슬라이스 |
|---|---|---|
| 응답성 | **좋다** (내 차례가 빨리 온다) | 나쁘다 (앞이 길면 오래 기다린다) |
| 전환 오버헤드 | **크다** | 작다 |
| 캐시 | 계속 식는다 | **덥혀 놓고 쓴다** |

여기서 §2-4의 캐시 이야기가 두 번째로 돌아온다. 전환 비용 `C`는 고정값이 아니다. 워크로드가 캐시를 얼마나 쓰느냐에 따라 커진다. 캐시에 크게 의존하는 작업일수록 짧은 슬라이스의 손해가 크다.

**그런데 발송 워커에는 이 튜닝이 대체로 무의미하다.** 워커 시간의 대부분은 채널사 API 응답 대기(§2-3의 `S`)고, I/O 바운드 작업은 퀀텀을 다 쓰기 전에 스스로 CPU를 놓는다. 슬라이스 길이가 실제로 걸리는 구간은 템플릿 렌더링·암호화·압축 같은 CPU 바운드 부분이다. 어디가 병목인지 모르는 채로 슬라이스를 만지면 아무 일도 안 일어난다.

> `SCHED_RR`의 퀀텀은 `sched_rr_get_interval(2)`로 조회한다. `SCHED_OTHER`에는 §2-8대로 고정 퀀텀 개념 자체가 없다.

### 2-10. 이 저장소를 관통하는 같은 부등호

여기까지의 이야기는 한 문장으로 압축된다.

> **동시 실행 단위를 N배로 늘려도 코어 수·디스크 대역폭·채널사 TPS 한도는 그대로다. 늘어난 건 대기 큐와 전환 비용뿐이다.**

같은 부등호가 이 저장소에서 이름만 바꿔 네 번 나온다.

| 문서 | 이름 | 자원이 안 늘 때 |
|---|---|---|
| [메모리와 페이징](memory-and-paging.md) §2-5 | **스래싱** | 활성 페이지 합 > 물리 프레임 → 폴트 급증. CPU 유휴를 스케줄러가 "일이 부족하다"로 오독해 더 올린다 |
| [트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-10 | **커넥션 풀** | 풀을 키워도 DB 코어·디스크는 그대로 → 락 경합과 대기만 증가 |
| [대용량 처리](../system-design/high-throughput.md) §2-2 | **백프레셔** | 수용 못 할 양을 받으면 큐가 메모리를 먹고 어딘가가 터진다 |
| [전송 계층](../network/transport-layer.md) §2-3 | **흐름 제어** | 수신 버퍼보다 많이 보내면 버려지고 재전송된다 |

스래싱 항목이 특히 스케줄러 이야기다. 그 악순환의 결정적 고리가 "CPU가 논다 → 프로세스를 더 올리자"라는 스케줄러의 판단이다. CPU 유휴는 "일이 부족하다"의 신호일 수도 있고 "다들 I/O를 기다린다"의 신호일 수도 있는데, 지표만으로는 구분되지 않는다. §2-3의 `R`과 `D`를 같이 봐야 갈린다.

발송 도메인으로 옮기면, 채널사가 초당 100건만 받는데 워커를 500개로 늘리면 처리량은 그대로고 늘어난 건 셋이다. 대기 워커 400개, 그만큼의 스택 메모리, 500개를 번갈아 재우고 깨우는 컨텍스트 스위칭. 병목이 우리 CPU가 아닐 때 동시성을 늘리는 건 비용만 사는 것이다.

---

**다음으로 읽을 것**

- 공유 메모리를 여럿이 건드릴 때 → [동시성 원시타입](concurrency-primitives.md)
- 그 스레드가 실제로 무엇을 기다리는가 → [메모리와 페이징](memory-and-paging.md)
- 같은 부등호가 DB에서 반복되는 자리 → [트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-10

---

> **기준 버전**: Linux man-pages 및 커널 문서 기준(리눅스 6.x). 알고리즘 표(§2-6)는 특정 OS에 매이지 않는 개념 수준 서술
> **확인한 출처**:
> - `sched(7)` — 정책 목록·정적 우선순위 범위·"All scheduling is preemptive"·nice 1.25배·동적 우선순위 상승·RT 스로틀링 기본값: https://man7.org/linux/man-pages/man7/sched.7.html
> - `pthreads(7)` — 스레드 간 공유/비공유 항목(§2-2): https://man7.org/linux/man-pages/man7/pthreads.7.html
> - `ps(1)` PROCESS STATE CODES — `R`/`S`/`D`/`T`/`Z` 원문(§2-3): https://man7.org/linux/man-pages/man1/ps.1.html
> - 커널 CFS 설계 문서 — "ideal, precise multi-tasking CPU"·vruntime·"no notion of timeslices"·rbtree leftmost·deterministic amount of time: https://www.kernel.org/doc/html/latest/scheduler/sched-design-CFS.html
> - `pthread_mutexattr_getprotocol(3p)` — `PTHREAD_PRIO_INHERIT`(§2-7 우선순위 역전 각주): https://man7.org/linux/man-pages/man3/pthread_mutexattr_getprotocol.3p.html
> **미확인**:
> - **FCFS·SJF·RR·MLFQ와 콘보이 효과·에이징이라는 용어**(§2-6, §2-7) — 교과서(OSTEP·Silberschatz) 개념으로 서술했고 원전 대조는 하지 않았다. man page에는 이 이름들이 없다
> - **컨텍스트 스위칭이 TLB를 비운다**(§2-4) — 페이지 테이블 교체가 필요하다는 것까지는 [메모리와 페이징](memory-and-paging.md) §2-1과 `pthreads(7)`에서 따라 나오지만, TLB 플러시 동작과 ASID/PCID로 회피하는지는 대조하지 않았다
> - **`D` 상태에서 `SIGKILL`이 안 먹는다** — "uninterruptible"이라는 단어에서 추론했을 뿐 명시적 서술을 확인하지 못했다. 본문에는 쓰지 않았다
> - **EEVDF가 CFS를 완전히 대체했는지**(§2-8) — 커널 문서에 EEVDF 항목이 있고 6.6에서 전환이 시작됐다고 되어 있으나, 대체 완료 여부가 명확하지 않아 본문은 CFS 기준으로 썼다
> - **`T`(job control)와 `t`(디버거)의 구분** — `ps(1)`에 둘 다 있으나 표에는 `T`만 실었다
> **미작성**: IPC 상세(파이프·공유 메모리·시그널) · `fork`/`exec`와 COW · 멀티코어 로드 밸런싱과 CPU 어피니티 · cgroup CPU 제한(컨테이너에서의 스케줄링) · 커널 선점(`PREEMPT_RT`)
