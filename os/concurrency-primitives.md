# 동시성 원시타입 — 동기/비동기와 블로킹/논블로킹

## 30초 요약

- **두 축이 다르다.** 블로킹/논블로킹은 **호출한 쪽이 기다리는가**, 동기/비동기는 **결과를 누가 처리하는가**
- 경쟁 상태의 실체는 **한 줄처럼 보이는 `count++`이 읽기·더하기·쓰기 세 단계**라는 것이다. 무서운 건 **대부분의 경우 맞게 돈다**는 점이다
- 뮤텍스와 세마포어의 진짜 차이는 개수가 아니라 **소유권**이다 — 뮤텍스는 잠근 스레드가 주인이고, 세마포어는 주인이 없다
- **락이 지키는 건 상호 배제뿐이다.** 다른 코어가 그 값을 언제 보는가는 별개 축이고, 그게 메모리 가시성이다
- **기다리는 방법도 선택이다** — 스핀락은 CPU를 태우고, 블로킹 락은 컨텍스트 스위칭을 낸다. 임계 구역 길이가 손익분기점을 정한다

---

## 원리 — 왜 그런가

### 2-1. 동기/비동기 · 블로킹/논블로킹

**두 축이 다르다.** 헷갈리기 쉬우니 구분해서.
- **블로킹 / 논블로킹**: **호출한 쪽이 기다리는가** (제어권을 언제 돌려주는가)
- **동기 / 비동기**: **결과를 누가 처리하는가** (내가 확인하는가, 완료되면 알려주는가)

|  | 블로킹 | 논블로킹 |
|---|---|---|
| **동기** | 일반 I/O — 부르고 기다렸다 결과를 받는다 | 폴링 — 계속 "다 됐어?" 물어본다 |
| **비동기** | (드묾) | 콜백/Future — 완료되면 알려준다 |

**발송 구조 설명에 쓴다**: "발송 요청은 **접수만 하고 즉시 응답**하고, 실제 발송은 **비동기로** 큐를 통해 처리합니다."

### 2-2. 임계 구역과 경쟁 상태 — `sentCount++`은 한 줄이 아니다

§2-1까지는 스레드가 하나뿐인 세계다. 둘 이상이 **같은 메모리**를 건드리는 순간 새로운 축이 열린다.

발송 워커 두 개가 성공 건수를 센다. `sentCount++`은 한 줄로 보이지만 기계 수준에서는 셋이다 — **① 읽고(load) ② 더하고(add) ③ 쓴다(store).** 두 워커가 이렇게 겹치면:

```
워커 A: ① 읽는다 → 100
워커 B: ① 읽는다 → 100      ← A가 아직 안 썼다
워커 A: ②③ 101을 쓴다
워커 B: ②③ 101을 쓴다      ← 두 건을 보냈는데 카운터는 1만 올랐다
```

**경쟁 상태(race condition)**: 결과가 **스레드 실행 순서에 좌우되는** 상태.
**임계 구역(critical section)**: 공유 자원을 건드려서 한 번에 하나만 들어가야 하는 코드 구간.

가장 고약한 성질은 따로 있다. **①②③이 겹칠 확률은 낮아서 대부분은 맞게 돈다.** 개발·스테이징에서는 안 나고, 발송량이 몰리는 새벽에만 카운터가 조금씩 어긋난다. **재현이 안 되니 테스트로 못 잡고, 부하가 늘수록 확률이 올라간다.**

> 원자성만 문제인 게 아니다. **다른 코어가 그 값을 언제 보느냐**가 또 다른 축이고 그건 §2-8이다. [Java](../java/jvm-gc-concurrency.md) §2-5가 "가시성과 원자성은 다르다"로 같은 구분을 한다.

### 2-3. 상호 배제 — 무엇을 보장해야 락인가

임계 구역을 지키는 장치에 요구되는 조건은 셋이다.

| 조건 | 뜻 | 안 지키면 |
|---|---|---|
| **상호 배제** | 한 번에 하나만 들어간다 | §2-2가 그대로 재발 |
| **진행** | 아무도 안 들어가 있으면 들어가려는 쪽 중 하나는 반드시 들어간다 | 비어 있는데 아무도 못 들어가는 교착 |
| **유한 대기** | 무한정 기다리는 스레드가 없다 | **기아** → [프로세스와 스레드](process-and-scheduling.md) §2-7 |

**셋 다 필요하다.** 상호 배제만 보면 "전역 락 하나로 다 감싸기"가 정답 같지만, 그건 유한 대기와 §2-7의 "경쟁을 줄인다"는 방향을 통째로 버리는 것이다.

소프트웨어만으로도 이 셋을 만들 수 있다(피터슨 알고리즘). 그런데 실제 구현은 전부 **하드웨어가 주는 원자적 read-modify-write 명령**(CAS, test-and-set)에 기댄다. 소프트웨어 알고리즘은 짧은 임계 구역에 비해 느리고, **§2-8의 재정렬 때문에 그냥은 돌지도 않는다** — 컴파일러와 CPU가 순서를 바꾸면 알고리즘의 전제가 무너진다.

즉 **"락"은 하드웨어 원자 명령 위에 올린 얇은 층**이고, 그 위에 뮤텍스·세마포어·모니터가 얹힌다.

### 2-4. 뮤텍스 vs 세마포어 — 진짜 차이는 소유권이다

흔한 설명은 "뮤텍스는 1개, 세마포어는 N개"다. **틀리진 않지만 핵심이 아니다.**

| | 뮤텍스 | 세마포어 |
|---|---|---|
| 정의 | 잠금 상태 + **소유자** | **"an integer whose value is never allowed to fall below zero"** |
| **소유권** | **있다** — 잠근 스레드가 주인 | **없다** |
| 해제 주체 | **주인만** | **누구나** (`sem_post`) |
| 목적 | **상호 배제** | **자원 개수 제한 · 신호 전달** |

POSIX 명세가 소유권을 명시한다.

> `pthread_mutex_lock(3p)`: "This operation shall return with the mutex object referenced by *mutex* in the locked state **with the calling thread as its owner**."

반면 `sem_overview(7)`은 세마포어를 **정수와 두 연산**으로만 정의한다 — `sem_post`(증가), `sem_wait`(감소, 0이면 대기). **어느 스레드의 것인지는 개념에 없다.**

**이 차이가 실제로 세 가지 결과를 낳는다.**

**(1) 재잠금 판정이 가능해진다.** 같은 스레드가 자기가 쥔 뮤텍스를 또 잠그면 어떻게 되는가 — 주인을 알아야 답할 수 있는 질문이다. POSIX는 타입별로 다르게 정한다.

| 뮤텍스 타입 | 같은 스레드가 재잠금 |
|---|---|
| `PTHREAD_MUTEX_NORMAL` | **"deadlock"** — 자기 자신을 영원히 기다린다 |
| `PTHREAD_MUTEX_ERRORCHECK` | "error returned" |
| `PTHREAD_MUTEX_RECURSIVE` | 카운트를 올려 허용. 올린 만큼 풀어야 놓인다 |
| `PTHREAD_MUTEX_DEFAULT` | "undefined behavior" |

**(2) 잘못된 해제를 잡아낼 수 있다.** `pthread_mutex_unlock`은 `EPERM`을 낸다 — 조건이 "the current thread does not own the mutex"다. **세마포어에는 이 검사가 원리적으로 불가능하다.**

**(3) 우선순위 상속이 가능해진다. 이게 결정적이다.** 높은 우선순위 스레드가 낮은 쪽이 쥔 락을 기다리는 사이 중간 우선순위 스레드가 낮은 쪽을 계속 선점하면 높은 쪽이 멈춘다(**우선순위 역전**). POSIX의 처방은 락을 쥔 쪽을 끌어올리는 것이다.

> `pthread_mutexattr_getprotocol(3p)`, `PTHREAD_PRIO_INHERIT`: "that **owner thread** shall inherit the priority level of the calling thread **as long as it continues to own the mutex**."

**"주인"이 두 번 나온다.** 누구의 우선순위를 올릴지 알려면 주인을 알아야 하고, 언제 되돌릴지 알려면 소유가 끝나는 시점을 알아야 한다. **세마포어로는 이 처방을 쓸 수 없다** — 지금 그 자원을 누가 붙들고 있는지 세마포어는 모른다. → [프로세스와 스레드](process-and-scheduling.md) §2-7

**그래서 "값이 1인 세마포어 = 뮤텍스"가 아니다.** A가 `sem_wait`하고 B가 `sem_post`하는 구조가 가능한데, 그건 락이 아니라 **신호**다. 반대로 그 비대칭성이 세마포어의 쓸모다.

**발송 도메인에서 둘을 나눠 쓰면 이렇게 된다.** 발송 통계 카운터나 채널 커넥션 객체는 **뮤텍스** — 하나만 만지게 하고 만진 쪽이 푼다. **채널사 동시 호출을 20개로 제한**하는 건 **세마포어**(Java `Semaphore(20)`) — 자원을 세는 것이지 상호 배제가 아니고, [대용량 처리](../system-design/high-throughput.md) §2-3의 레이트 리미팅과 같은 목적이다.

### 2-5. 모니터 — 락과 조건 대기를 한 덩어리로

뮤텍스를 직접 쓰면 손이 많이 간다. **`lock()`은 잊지 않는데 `unlock()`을 잊는다** — 특히 중간에 예외가 튀면.

**모니터**는 그 문제를 언어 차원에서 없앤 구성이다. **지켜야 할 상태와 그것을 지키는 락을 한 객체에 묶고, 블록을 벗어나면 자동으로 푼다.** Java `synchronized`가 그 구현이고, `finally { unlock(); }`을 언어가 대신 써 주는 셈이다. 나머지 절반이 **조건 변수** — "락은 잡았는데 **조건이 아직 안 맞는** 경우"를 다룬다.

```
발송 워커: 큐에서 꺼내려는데 큐가 비었다
  → 락을 쥔 채로 기다리면 아무도 넣을 수 없다 (데드락)
  → wait(): 락을 놓고 잠든다
  → 누가 넣고 notify(): 깨어나 락을 다시 잡는다
```

**`wait()`이 락을 놓는다는 게 핵심이다.** 그냥 `sleep`으로는 안 된다 — 락을 쥔 채 자면 생산자가 영영 못 들어온다.

**조건 검사는 `if`가 아니라 `while`로 감싼다.** 깨어난 시점과 락을 다시 잡는 시점 사이에 다른 소비자가 먼저 큐를 비울 수 있기 때문이다. **"깨워졌다"는 "조건이 참이다"를 뜻하지 않는다.**

### 2-6. 스핀락 vs 블로킹 락 — 기다리는 방식의 거래

락이 이미 잡혀 있을 때 할 수 있는 일은 둘이다.

| | 스핀락 | 블로킹 락 |
|---|---|---|
| 대기 방식 | **"the calling thread spins, testing the lock until it becomes available"** | 대기 큐에 들어가 잠든다 |
| 비용 | **CPU를 계속 태운다** | **컨텍스트 스위칭 2회**(재우기 + 깨우기) |
| 유리한 조건 | 임계 구역 < 전환 비용 | 임계 구역 > 전환 비용 |
| 대기 중 CPU | 점유 | **양보** |

**손익분기점은 [프로세스와 스레드](process-and-scheduling.md) §2-4의 컨텍스트 스위칭 비용이다.** 임계 구역이 카운터 하나 증가시키는 정도면 재우고 깨우는 비용이 실제 작업보다 크다 — 그럴 땐 **잠깐 도는 게 싸다.** 반대로 임계 구역 안에서 I/O를 하거나 수십 마이크로초를 쓰면, 스핀은 그 시간 내내 코어 하나를 통째로 낭비한다.

**단일 코어에서 스핀락은 최악에 가깝다.** 락을 쥔 스레드가 돌아야 놓는데 내가 CPU를 붙들고 있으면 그 스레드가 못 돈다. **스핀락이 의미 있는 전제는 "락을 쥔 쪽이 다른 코어에서 지금 돌고 있다"이다.** 그래서 실무 구현은 대개 **섞는다** — 짧게 스핀해 보고 안 되면 잠든다. 커널의 adaptive mutex, JVM의 적응형 스피닝이 그 방식이다.

> 이 절이 [Redis](../datastore/redis.md) §2-1의 반대편이다. Redis는 **스레드를 하나로 고정해 대기 방식을 고를 문제 자체를 없앴다.** §2-9.

### 2-7. 그런데 락은 왜 이렇게 싼가 — futex

"락은 비싸니 줄여라"는 통념이 있는데, 리눅스에서는 **정확하지 않다.**

> `futex(7)`: "Futex operation occurs entirely in user space **for the noncontended case**. The kernel is involved only to arbitrate the **contended** case."

```
경쟁 없음: 유저 공간에서 원자 연산 하나. 시스템 콜 없음
경쟁 발생: FUTEX_WAIT로 커널에 재워 달라고 부탁
   해제 시: FUTEX_WAKE로 대기자를 깨운다
```

그리고 이게 바닥이다 — "Futexes are very basic and lend themselves well for building higher-level locking abstractions such as **mutexes, condition variables, read-write locks, barriers, and semaphores**." §2-4~2-6에서 본 것들이 전부 이 위에 올라가 있다.

**결론은 방향을 바꾼다. 비싼 건 락이 아니라 경쟁이다.** 아무도 안 다투는 락은 거의 공짜고, 다투는 순간 커널 진입 + 컨텍스트 스위칭 2회가 붙는다.

그래서 튜닝은 **락을 없애는 쪽이 아니라 경쟁을 줄이는 쪽**으로 간다. **임계 구역을 줄이고**(락 안에서 I/O·로깅·직렬화를 하지 않는다), **락을 쪼갠다**(발송 통계를 채널별로 나누면 SMS 워커와 알림톡 워커가 안 부딪힌다). [트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-6에서 InnoDB가 테이블이 아니라 **인덱스 레코드**에 락을 거는 것도 같은 방향이다.

### 2-8. 메모리 가시성 — 락이 없어도 깨지는 다른 축

§2-2는 **원자성**의 문제였다. 여기는 다른 축이다. **한 코어가 쓴 값을 다른 코어가 언제 보는가.** 전제부터가 직관과 다르다.

> "a CPU may actually perform the memory operations **in any order it likes**, provided program causality appears to be maintained. Similarly, **the compiler may also arrange the instructions it emits in any order it likes.**"

"program causality appears to be maintained"의 범위가 **자기 자신 기준**이라는 게 함정이다. 단일 스레드로 보면 결과가 같지만, **다른 코어에서 보는 순서는 내가 쓴 순서와 다를 수 있다.** 발송 워커를 깨우는 흔한 코드가 여기 빠진다.

```
스레드 A:  payload = buildMessage();   // ①
           ready = true;               // ②

스레드 B:  while (!ready) { }
           send(payload);              // ①이 아직 안 보일 수 있다
```

B는 `ready == true`를 봤는데 `payload`는 옛 값일 수 있다. **②가 ①보다 먼저 보이는 것을 막는 게 아무것도 없기 때문이다.**

**메모리 배리어**가 그 순서를 강제한다.

| 종류 | 보장 |
|---|---|
| **쓰기(store) 배리어** | "all the STORE operations specified before the barrier will appear to happen before all the STORE operations specified after the barrier" |
| **읽기(load) 배리어** | 위와 같되 LOAD에 대해 |
| **일반 배리어** | LOAD와 STORE 모두에 대해 |
| 주소 의존 배리어 | 읽기 배리어의 약한 형태. 상호 의존적인 로드에만 |

**두 가지를 반드시 같이 알아야 한다. (1) 배리어는 짝을 맞춰야 한다** — 위 예에서 A만 걸면 소용없다.

> "There is no guarantee that a CPU will see the correct order of effects from a second CPU's accesses, **even if the second CPU uses a memory barrier, unless the first CPU also uses a matching memory barrier.**"

**(2) 배리어는 순서를 보장하지 시각을 보장하지 않는다.**

> "There is no guarantee that any of the memory accesses specified before a memory barrier will be **complete** by the completion of a memory barrier instruction; the barrier can be considered to draw a line in that CPU's access queue..."

**"즉시 반영된다"가 아니라 "선을 긋는다"다.** B가 언제 보는지는 여전히 모른다. 아는 건 **보이는 순간 ①과 ②가 그 순서로 보인다**는 것뿐이다.

**실무에서 이 축이 잘 안 보이는 이유는 락이 배리어를 포함하기 때문이다.** 락을 제대로 쓰면 §2-2의 원자성과 이 절의 가시성이 한꺼번에 해결된다. **문제는 "락은 비싸니까"라며 걷어내고 플래그 하나로 최적화할 때 갑자기 튀어나온다** — 그리고 §2-2와 같은 이유로 재현이 안 된다. 자바에서 `volatile`이 가시성만 주고 원자성은 안 주는 것이 이 두 축의 분리를 그대로 보여준다(→ [Java](../java/jvm-gc-concurrency.md) §2-5).

### 2-9. 락을 피하는 길 — 그리고 그 대가

여기까지 오면 마지막 선택지가 보인다. **공유하지 않으면 동기화가 필요 없다.**

| 방법 | 대가 |
|---|---|
| **싱글 스레드 고정** — [Redis](../datastore/redis.md) §2-1 | 동기화 비용이 0. 대신 **코어를 하나만 쓴다.** 병목이 CPU가 아닐 때만 성립 |
| **불변 객체 / 스레드 로컬** | 공유가 없으니 경쟁도 없다. 대신 **복사 비용과 메모리** |
| **CAS 기반 원자 연산** (`AtomicLong`) | 락 없이 실패하면 재시도. **경쟁이 심하면 재시도가 폭발**한다 |
| **메시지 전달** — 상태 대신 큐 | 공유 상태를 없앤다. 대신 **큐 자체가 새 병목**이 된다 |

**마지막 줄이 이 저장소의 도메인이 이미 택한 답이다.** 발송 요청을 큐에 넣고 워커가 하나씩 꺼내면 발송 상태를 여럿이 동시에 건드릴 일이 애초에 생기지 않는다(→ [메시징 기초](../messaging/basics.md) §2-1). **동시성 문제를 잘 푸는 대신 만들지 않는 쪽으로 설계를 옮긴 것**이다. 그리고 셋째 줄은 §2-7의 결론을 되풀이한다 — CAS도 경쟁이 심해지면 재시도가 늘어 결국 느려진다. **락이든 CAS든 진짜 비용은 경쟁이다.**

---

**다음으로 읽을 것**

- 기다리는 스레드가 왜 비싼가, 스케줄러가 언제 전환하는가 → [프로세스와 스레드](process-and-scheduling.md) §2-1, §2-5
- 이 원시타입들이 DB에서 어떤 모양이 되는가 → [트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-4, §2-7
- 블로킹 I/O가 스레드 풀을 어떻게 고갈시키는가 → [신뢰성 패턴](../network/reliability-patterns.md) §2-1
- 동기화를 아예 없애는 반대 방향 → [Redis](../datastore/redis.md) §2-1

---

> **기준 버전**: POSIX(`pthreads`) 및 Linux man-pages·커널 문서 기준(리눅스 6.x). 코드 예시는 개념 설명용 의사코드
> **확인한 출처**:
> - `pthread_mutex_lock(3p)` — "with the calling thread as its owner"·`EPERM`("the current thread does not own the mutex")·타입별 재잠금 동작표(§2-4): https://man7.org/linux/man-pages/man3/pthread_mutex_lock.3p.html
> - `sem_overview(7)` — "an integer whose value is never allowed to fall below zero"·`sem_post`/`sem_wait`·**소유권 서술 없음**(§2-4): https://man7.org/linux/man-pages/man7/sem_overview.7.html
> - `pthread_mutexattr_getprotocol(3p)` — `PTHREAD_PRIO_INHERIT` 우선순위 상속 원문(§2-4): https://man7.org/linux/man-pages/man3/pthread_mutexattr_getprotocol.3p.html
> - `pthread_spin_lock(3)` — "the calling thread spins, testing the lock until it becomes available"(§2-6): https://man7.org/linux/man-pages/man3/pthread_spin_lock.3.html
> - `futex(7)` — 비경쟁 시 유저 공간에서만 처리·커널은 경쟁만 중재·상위 원시타입의 토대(§2-7): https://man7.org/linux/man-pages/man7/futex.7.html
> - 커널 `memory-barriers.txt` — 재정렬 전제·배리어 4종 정의·짝 맞추기 요구·"완료를 보장하지 않는다"(§2-8): https://www.kernel.org/doc/Documentation/memory-barriers.txt
> **미확인**:
> - **상호 배제·진행·유한 대기 세 조건**(§2-3)과 **피터슨 알고리즘** — 교과서(Silberschatz) 개념으로 서술했고 원전 대조는 하지 않았다
> - **`while`로 조건을 감싸야 하는 이유**(§2-5) — 본문에는 확실히 성립하는 이유(깨어난 뒤 다른 소비자가 선점)만 적었다. 표준이 **spurious wakeup**을 허용한다는 서술은 `pthread_cond_wait(3p)`를 대조하지 않아 본문에서 뺐다
> - **단일 코어에서 스핀락이 불리하다**(§2-6) — `pthread_spin_lock(3)`은 이 경고를 담고 있지 않다. [프로세스와 스레드](process-and-scheduling.md) §2-5(선점)의 동작에서 따라 나오는 추론이다
> - **커널 adaptive mutex와 JVM 적응형 스피닝**(§2-6) — 개념만 언급했고 구현 문서는 대조하지 않았다
> - **`synchronized`가 어떤 배리어로 번역되는지**(§2-8) — Java Memory Model 명세를 대조하지 않았다. 락이 배리어를 포함한다는 일반 서술까지만 적었다
> **미작성**: 읽기-쓰기 락(RWLock)과 그 기아 문제 · 배리어(barrier)와 래치 · 데드락 4조건과 회피(→ [트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-7에 DB 관점으로 있다) · 락 프리 자료구조와 ABA 문제 · 트랜잭셔널 메모리
