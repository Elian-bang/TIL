# JVM · GC · 컬렉션 · 동시성

## 30초 요약

- **GC가 세대를 나누는 이유**: 대부분의 객체는 금방 죽는다(weak generational hypothesis). 그래서 젊은 영역만 자주, 싸게 치운다
- **Survivor가 둘인 이유**: 살아남은 객체를 한쪽으로 몰아 복사해서 단편화 없이 비우기 위해서다. 항상 하나는 비어 있다
- **메모리 누수 = GC가 회수하지 못하는 참조가 남아 있는 것.** GC의 실패가 아니라 참조를 안 끊은 코드의 문제다
- **HashMap**: 해시로 버킷 결정 → 충돌 시 연결 리스트 → Java 8부터 한 버킷 8개 이상(+테이블 64 이상)이면 트리로 전환
- **Virtual Thread**: I/O 대기 시 carrier 스레드를 반납한다 → I/O 바운드에 강하다. CPU 바운드에는 이득이 없다
- **⚠️ 스레드를 늘려도 커넥션 풀이 10개면 10개다.** 진짜 병목은 대체로 스레드 뒤에 있다

---

## 원리 — 왜 그런가

### 2-1. JVM 메모리 구조는 왜 나뉘어 있나

| 영역 | 무엇이 | 공유 | 특징 |
|---|---|---|---|
| **힙(Heap)** | 모든 객체·배열 | **스레드 간 공유** | GC의 대상. Young + Old |
| **스택(Stack)** | 지역 변수, 메서드 호출 프레임 | **스레드마다 하나** | 메서드가 끝나면 자동 해제. GC 대상 아님 |
| **메타스페이스** | 클래스 메타데이터 | 공유 | **힙 밖(네이티브 메모리)**. Java 8에서 PermGen을 대체 |

왜 이렇게 나누나. **수명이 다르기 때문이다.** 지역 변수는 메서드가 끝나는 순간 확실히 죽는다 → 스택에서 자동 정리하면 되고 GC가 필요 없다. 객체는 누가 언제까지 참조할지 알 수 없다 → 힙에 두고 GC가 판단한다.

> **PermGen이 메타스페이스로 바뀐 이유**: PermGen은 힙 안에 있어 크기가 고정이었다 → 클래스를 동적으로 많이 만드는 애플리케이션에서 `OutOfMemoryError: PermGen space`가 났다. 메타스페이스는 네이티브 메모리라 필요한 만큼 늘어난다.

### 2-2. GC는 왜 세대를 나누고, Survivor는 왜 둘인가

#### (1) 세대를 나누는 근거: 약한 세대 가설

> **"대부분의 객체는 만들어지자마자 죽는다."** (weak generational hypothesis)

실측에서 반복적으로 확인되는 경험칙이다. 메서드 안에서 만든 임시 객체나 반복문의 중간 결과는 대부분 다음 순간이면 쓰레기다.

여기서 최적화가 나온다. 힙 전체를 매번 훑으면 비싸다. 그런데 **쓰레기는 대부분 젊은 영역에 몰려 있다.** 그러니 젊은 영역만 자주 치우면 적은 비용으로 대부분의 쓰레기를 회수할 수 있다.

- **Minor GC** (Young 영역): 자주, 빠르게
- **Major / Full GC** (Old 포함): 드물게, 비싸게 (**여기서 긴 STW가 난다**)

#### (2) Survivor가 둘인 이유

Young = **Eden + Survivor 0 + Survivor 1**. 왜 Survivor가 두 개인가.

Young 영역의 GC는 **복사(copying) 방식**이다. 살아남은 객체를 다른 영역으로 복사하고, 원래 영역은 통째로 비운다.

```
1. 객체는 Eden에 생성된다
2. Minor GC: Eden에서 살아남은 것을 → S0으로 복사. Eden을 통째로 비운다
3. 다음 Minor GC: Eden + S0에서 살아남은 것을 → S1로 복사. Eden과 S0을 통째로 비운다
4. 그 다음: Eden + S1 → S0 ... (S0과 S1이 번갈아 가며 to-space가 된다)
5. 복사 횟수(age)가 임계값을 넘으면 → Old로 승격(promotion)
```

둘인 이유는 이렇다. 복사 방식은 **비어 있는 목적지(to-space)가 필요하다.** 하나뿐이면 살아남은 객체와 새로 복사할 객체가 섞여서 통째로 비울 수가 없다. 둘을 번갈아 쓰면 항상 한쪽은 완전히 비어 있고, 복사가 끝나면 원본을 통째로 비운다.

부수 효과가 더 중요하다. 복사하면서 객체를 한쪽 끝부터 차곡차곡 쌓기 때문에 **단편화가 생기지 않는다.** 그래서 새 객체 할당이 포인터를 앞으로 미는 것만큼 싸다(bump-the-pointer).

> ⚠️ 위 그림은 Eden·S0·S1이 연속된 공간인 모델이다. **기본 GC인 G1에서는 연속 공간이 아니다.** Oracle 문서는 G1의 young 영역을 두고 *"these regions are typically laid out in a noncontiguous pattern in memory"*라고 적는다. 나이에 따라 survivor 또는 old로 복사한다는 원리는 그대로다. *"Objects of the young generation (eden and survivor regions) are copied into survivor or old regions, depending on their age."*


#### (3) GC 알고리즘은 왜 계속 바뀌나

방향은 하나다. **STW(Stop-The-World) 시간을 줄이는 것.**

| GC | 아이디어 | 언제 |
|---|---|---|
| Serial | 한 스레드로 전부 처리 | 단일 프로세서, **작은 데이터셋(대략 100MB 이하)** |
| Parallel | 멈추고 여러 스레드로 빠르게 치운다 | 처리량 우선(throughput collector) |
| **G1** (JDK 21 기본) | 힙을 **리전(region)** 으로 잘게 쪼개고, **쓰레기가 많은 리전부터** 수집 (Garbage First) | 범용. 목표 정지 시간(`MaxGCPauseMillis`)을 걸어 둘 수 있다 |
| **ZGC** | 대부분의 작업을 **애플리케이션과 동시에** 수행 | 지연이 우선일 때. **정지 1ms 미만**, 대신 처리량을 조금 내준다 |

Oracle 문서는 기본값을 *"G1 is selected by default on most hardware and operating system configurations"*라고 적는다. 정지 시간 목표는 보장이 아니라 확률이라는 것도 문서에 그렇게 쓰여 있다. G1은 *"provides the capability to meet a pause-time goal with high probability, while achieving high throughput"*이고, ZGC는 *"provides max pause times under a millisecond, but at the cost of some throughput"*이다.

**G1이 리전을 쓰는 이유**: 기존 방식은 Young/Old가 연속된 큰 덩어리라서, Old를 치우려면 그 전체를 봐야 했다 → 힙이 클수록 STW가 길어진다. 리전으로 쪼개면 "이번엔 이 리전 몇 개만" 처리할 수 있다 → 정지 시간을 예측하고 제어할 수 있다. 문서 표현으로는 *"A region is the unit of memory allocation and memory reclamation"*이고, 이름의 유래는 *"G1 reclaims space in the most efficient areas first (that is the areas that are mostly filled with garbage, therefore the name)"*다.

> Shenandoah은 이 표에서 뺐다. **Oracle JDK 21의 수집기 목록은 Serial · Parallel · G1 · ZGC 넷뿐**이고 Shenandoah은 거기 없다. 배포판에 따라 제공 여부가 갈리므로 "ZGC / Shenandoah"로 묶어 말하면 부정확하다.

### 2-3. GC가 있는데 메모리는 왜 새나

**"메모리 누수 = GC가 회수하지 못하는 참조가 살아 있는 것."**

GC는 GC Root에서 도달 가능한 객체를 살아 있다고 판단한다. 그러니 **쓸모없어졌는데 참조가 남아 있으면 GC는 못 치운다.** 이건 GC의 버그가 아니라 참조를 안 끊은 코드의 문제다.

**전형적인 패턴**
| 패턴 | 왜 |
|---|---|
| **static 컬렉션에 계속 add** | static은 클래스가 살아 있는 한 GC Root. 앱이 죽을 때까지 안 죽는다 |
| **닫지 않은 리소스** | 스트림·커넥션 |
| **크기 제한 없는 캐시** | 넣기만 하고 안 뺀다. TTL/최대 크기 없이 만든 `Map` 캐시 |
| **리스너·콜백 미해제** | 등록만 하고 해제 안 함 |
| **ThreadLocal 미정리** | **스레드 풀에서 특히 위험.** 스레드가 재사용되므로 값이 계속 남는다 |
| **배치의 전형: 전체 결과를 리스트로** | 한 스텝에서 100만 건을 `List`로 들고 있으면 그대로 힙 |

**진단 흐름**
```
1. 힙 사용량 추이 관측     (jstat / APM / GC 로그)
   → Full GC 후에도 힙이 안 내려가면 누수 신호
2. Heap Dump 확보          (jmap, -XX:+HeapDumpOnOutOfMemoryError)
3. MAT / VisualVM 로 분석
   → Dominator Tree 에서 어떤 객체가 힙을 차지하는지
   → GC Root 까지의 참조 체인을 따라가 "왜 안 죽는지" 특정
4. 그 참조를 끊는다
```

> **핵심은 "어떤 객체가 어떤 참조 체인으로 잡혀 있었나"** 한 줄이다. 도구 이름을 나열하는 것보다, 그 체인을 특정했는지가 누수 분석의 전부다.

### 2-4. HashMap 내부와 컬렉션 선택

#### HashMap이 O(1)인 이유와, O(n)이 되는 조건

```
1. key.hashCode() 로 해시값을 얻는다
2. 해시를 섞어서(spread) 상위 비트도 반영한다   ← 하위 비트만 쓰면 충돌이 몰린다
3. 테이블 크기로 나눈 나머지 → 버킷 인덱스     (크기가 2의 거듭제곱이라 비트 연산으로)
4. 그 버킷에 이미 뭔가 있으면(충돌) → 연결 리스트로 잇는다
5. 조회 시 그 버킷의 리스트를 순회하며 equals() 로 찾는다
```

**충돌이 많으면 리스트가 길어지고 → 최악 O(n)** 이 된다. (해시 충돌을 의도적으로 만드는 DoS 공격도 있었다)

**Java 8의 개선: treeify**
- 한 버킷에 노드가 **8개 이상**인 상태에서 원소를 더 넣으면 → 레드-블랙 트리로 전환(`TREEIFY_THRESHOLD = 8`)
- 단, **테이블 크기가 64 미만이면 트리로 가지 않고 resize로 푼다**(`MIN_TREEIFY_CAPACITY = 64`). 테이블이 작아서 몰린 것과 해시가 나빠서 몰린 것은 처방이 다르기 때문이다
- 최악이 O(n)에서 O(log n)으로 완화된다
- 되돌리는 조건은 "줄어들면"이 아니다. **resize로 버킷을 쪼갤 때** 쪼갠 결과가 6개 이하면 리스트로 되돌린다(`UNTREEIFY_THRESHOLD = 6`)
- 기본 용량 16, load factor 0.75 → 12개가 차면 테이블을 2배로 늘리고 **전부 재배치(resize)** 한다

각 상수에 붙은 주석이 그대로 근거다. 8은 *"Bins are converted to trees when adding an element to a bin with at least this many nodes"*, 64는 *"The smallest table capacity for which bins may be treeified. (Otherwise the table is resized if too many nodes in a bin.)"*, 6은 *"The bin count threshold for untreeifying a (split) bin during a resize operation."*다.

> ⚠️ **이 임계값들은 공개 계약이 아니다.** `HashMap` 자바독에는 트리 전환이 아예 안 나오고, 8·64·6은 OpenJDK 구현 소스의 상수다. 면접에서 말할 때도 "구현 세부"라고 붙이는 편이 정확하다. 자바독이 보장하는 건 여기까지다. *"When the number of entries in the hash table exceeds the product of the load factor and the current capacity, the hash table is rehashed ... so that the hash table has approximately twice the number of buckets."*

> **왜 load factor가 0.75인가**: 자바독은 시간과 공간의 절충이라고만 말한다. *"the default load factor (.75) offers a good tradeoff between time and space costs. Higher values decrease the space overhead but increase the lookup cost"*. 올리면 공간은 아끼고 조회가 비싸진다는 방향만 문서에 있고, 왜 하필 0.75인지의 유도는 없다.

#### `equals()` / `hashCode()` 계약, 왜 같이 재정의해야 하나

> **equals가 true면 hashCode도 반드시 같아야 한다.**

hashCode로 버킷을 먼저 찾고, 그 안에서 equals로 비교하기 때문이다. hashCode를 재정의하지 않으면 **같은 객체인데 다른 버킷을 보게 되어** 넣은 값을 못 찾는다.
(역은 성립하지 않아도 된다. hashCode가 같아도 equals는 다를 수 있고, 그게 충돌이다.)

#### 그 외

- **HashMap vs ConcurrentHashMap vs Hashtable**
  동기화 없음 / **CAS + 버킷 단위 부분 잠금**(전체를 안 막아 동시성이 좋다) / 메서드 전체 `synchronized`(레거시, 느림)
- **ArrayList vs LinkedList**: 인덱스 접근 O(1) vs O(n). 중간 삽입은 이론상 LinkedList가 유리하지만, **실무에서는 캐시 지역성 때문에 ArrayList가 대부분 빠르다.** 연속 메모리라 CPU 캐시에 잘 올라간다
- **String vs StringBuilder**: String은 불변 → 반복 연결하면 매번 새 객체. 루프 안 문자열 연결은 StringBuilder

### 2-5. 동시성 기초, 가시성과 원자성은 다르다

동시성 버그는 대부분 이 둘 중 하나다. **구분해서 봐야 한다.**

- **가시성(visibility)**: 한 스레드가 바꾼 값이 다른 스레드에 보이지 않는 문제. CPU 캐시·컴파일러 최적화 때문
- **원자성(atomicity)**: `count++` 는 **읽기 → 더하기 → 쓰기** 세 단계다. 중간에 끼어들면 값이 유실된다

| 도구 | 가시성 | 원자성 |
|---|---|---|
| `volatile` | **O** | **X** ← 가장 흔한 오해 |
| `synchronized` / `ReentrantLock` | O | O |
| `AtomicInteger` (CAS) | O | O |

- **`volatile`은 원자성을 보장하지 않는다.** `volatile int count; count++;` 는 여전히 안전하지 않다
- **CAS(Compare-And-Swap)**: "값이 아직 A이면 B로 바꿔라"를 CPU 명령 하나로 한다. 실패하면 재시도(스핀). 락 없이 원자성을 얻으니 경합이 적을 때 락보다 빠르다
- **`synchronized` vs `ReentrantLock`**: 후자는 타임아웃, 인터럽트, 공정성(fairness), 조건 변수를 지원한다. 전자는 문법이 간단하고 JVM이 최적화한다

**`ThreadPoolExecutor`의 동작 순서** ← 직관에 반해서 자주 틀린다
```
1. 현재 스레드 < corePoolSize        → 새 스레드를 만든다
2. core가 찼다                       → 큐에 넣는다
3. 큐도 찼다                         → maximumPoolSize 까지 새 스레드
4. 그것도 찼다                       → RejectedExecutionHandler
```
**⚠️ 큐가 먼저다.** 그래서 용량을 지정하지 않은 `LinkedBlockingQueue`처럼 무제한 큐를 쓰면 3번이 영원히 안 온다 → `maximumPoolSize`가 아무 의미가 없어지고, 대신 큐가 무한정 쌓여 OOM으로 간다.

자바독이 이 순서와 결과를 그대로 적어 뒀다. *"If corePoolSize or more threads are running, the Executor always prefers queuing a request rather than adding a new thread. If a request cannot be queued, a new thread is created unless this would exceed maximumPoolSize, in which case, the task will be rejected."* 무제한 큐 항목은 더 노골적이다. *"Using an unbounded queue (for example a LinkedBlockingQueue without a predefined capacity) ... Thus, no more than corePoolSize threads will ever be created. (And the value of the maximumPoolSize therefore doesn't have any effect.)"*

### 2-6. Virtual Thread

#### (1) 왜 필요했나

기존 모델은 **요청 하나당 플랫폼 스레드 하나**(thread-per-request)다. 문제는 두 가지.

1. **플랫폼 스레드는 OS 스레드**라 비싸다. 스택에 수백 KB~1MB. 수천 개를 넘기기 어렵다
2. **I/O를 기다리는 동안 스레드가 통째로 놀고 있다.** 외부 채널사 API가 200ms 걸리면, 그 200ms 동안 OS 스레드 하나가 아무것도 안 하고 묶여 있다

기존 해법은 **비동기/리액티브**였다. 그런데 코드가 콜백·체인으로 어려워지고 디버깅과 스택 트레이스가 망가진다.

**Virtual Thread의 제안**: *"코드는 동기식 그대로 두고, 런타임이 알아서 스레드를 반납하게 하자."*

#### (2) 어떻게 동작하나

- **JVM이 스케줄링하는 경량 스레드.** 스택이 힙에 있고, 수십만 개를 만들 수 있다
- **carrier(플랫폼 스레드)에 마운트**되어 실행된다
- **블로킹 I/O를 만나면 언마운트**한다. carrier를 반납하고, 다른 가상 스레드가 그 carrier를 쓴다
- I/O가 끝나면 다시 아무 carrier에나 마운트되어 이어서 실행된다

→ **적은 수의 OS 스레드로 매우 높은 동시성.** 코드는 여전히 평범한 동기 코드다.

#### (3) 적합 / 부적합

- **적합: I/O 바운드.** 외부 API 호출, DB 쿼리처럼 대기가 긴 작업
- **부적합: CPU 바운드.** 실제 연산은 코어 수만큼만 병렬이다. 가상 스레드를 늘려도 이득이 없다

#### (4) ⚠️ pinning이라는 함정

**`synchronized` 블록 안에서 블로킹하면 언마운트가 안 된다.** carrier를 붙잡고(pinning) 놓지 않는다. 가상 스레드를 아무리 많이 만들어도 carrier 수만큼만 동시 실행되고, 최악의 경우 전부 묶여 멈춘다.

핀 되는 조건은 하나가 아니라 둘이다. Oracle 문서는 이렇게 적는다. *"The virtual thread runs code inside a `synchronized` block or method"*, *"The virtual thread runs a `native` method or a foreign function"*. **네이티브 호출과 FFI도 같은 함정이다.**

- **회피책: `ReentrantLock`으로 바꾼다.** 문서의 권고도 같다. *"Try avoiding frequent and long-lived pinning by revising `synchronized` blocks or methods that run frequently and guarding potentially long I/O operations with java.util.concurrent.locks.ReentrantLock."*
- JDK 21 기준의 제약이다. **"21 기준으로는 그랬다"**고 말하는 게 정확하다

#### (5) ⚠️⚠️ 진짜 병목은 스레드가 아니라 그 뒤다

가상 스레드를 1만 개 띄워도:
- **DB 커넥션 풀이 10개면 동시에 DB에 가는 건 10개**다. 나머지는 `connectionTimeout`을 기다리다 예외로 터진다
- **채널사 rate limit이 초당 100건이면 100건**이다. 더 많이 보내면 429를 받는다

**동시성을 늘리는 것과 처리량을 늘리는 것은 다르다.** 가상 스레드는 대기하는 비용을 싸게 만들 뿐, 뒤쪽의 처리 능력을 늘려주지 않는다.

Oracle 문서가 이 문장을 직접 적어 뒀다. *"Virtual threads are not faster threads; they do not run code any faster than platform threads. They exist to provide scale (higher throughput), not speed (lower latency)."* 적합/부적합의 근거도 같은 문서에 있다. *"Virtual threads are suitable for running tasks that spend most of the time blocked, often waiting for I/O operations to complete. However, they aren't intended for long-running CPU-intensive operations."*

> **"가상 스레드로 바꿨더니 빨라졌다"로 끝내면 안 되는 이유다.** 무엇이 얼마나 좋아지는지는 대체로 스레드가 아니라 그 뒤의 자원이 정한다.
> → [트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-10 커넥션 풀과 같은 이야기다

---

> **기준 버전**: JDK 21 (LTS). §2-2의 GC 목록과 기본값, §2-6의 pinning 조건이 여기에 묶인다
> **확인한 출처**:
> - [Available Collectors (JDK 21 GC Tuning Guide)](https://docs.oracle.com/en/java/javase/21/gctuning/available-collectors.html) — §2-2 표 전체. 수집기 **4종(Serial · Parallel · G1 · ZGC)**, *"G1 is selected by default on most hardware and operating system configurations"*, Serial의 *"up to approximately 100 MB)"*, Parallel의 *"also known as throughput collector"*, ZGC의 *"provides max pause times under a millisecond, but at the cost of some throughput"*
> - [Garbage-First Garbage Collector (JDK 21)](https://docs.oracle.com/en/java/javase/21/gctuning/garbage-first-g1-garbage-collector1.html) — §2-2 리전과 승격. *"A region is the unit of memory allocation and memory reclamation"*, *"reclaims space in the most efficient areas first ... therefore the name"*, *"these regions are typically laid out in a noncontiguous pattern in memory"*, *"copied into survivor or old regions, depending on their age"*
> - [`java.util.HashMap` (JDK 21 javadoc)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashMap.html) — §2-4의 기본 용량 **16** · 부하율 **0.75**와 재해시 규칙, 0.75의 근거 문장. **트리 전환은 이 자바독에 없다는 사실도 여기서 확인했다**
> - [OpenJDK `HashMap.java` (jdk-21+35)](https://github.com/openjdk/jdk/blob/jdk-21%2B35/src/java.base/share/classes/java/util/HashMap.java) — §2-4의 `TREEIFY_THRESHOLD` **8** · `MIN_TREEIFY_CAPACITY` **64** · `UNTREEIFY_THRESHOLD` **6**과 각 주석 원문, 해시 섞기의 *"we just XOR some shifted bits ... to incorporate impact of the highest bits"*
> - [`ThreadPoolExecutor` (JDK 21 javadoc)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html) — §2-5의 처리 순서 3줄과 무제한 큐 문단 원문, 거절 핸들러 4종(`AbortPolicy`가 기본)
> - [Virtual Threads (JDK 21 Core Libraries)](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html) — §2-6 전체. 마운트/언마운트, **pinning 조건 2종(`synchronized`, 네이티브·foreign function)**, `ReentrantLock` 권고, I/O 대기에 적합·CPU 집약에 부적합, *"not faster threads ... provide scale (higher throughput), not speed (lower latency)"*
> **미확인**:
> - **G1이 Java 9부터 기본**(§2-2) — JDK 21 문서로 "지금 기본"인 것만 확인했다. 도입 릴리스는 JEP 248을 열어야 하는데 openjdk.org가 접근을 막아 대조하지 못했다
> - **`MaxGCPauseMillis`의 기본값**(§2-2) — 이 옵션이 정지 시간 목표를 조절한다는 것까지는 튜닝 가이드에 있으나 **기본값은 문서에서 찾지 못했다.** 그래서 본문에 숫자를 쓰지 않았다
> - **약한 세대 가설·Eden/S0/S1 복사·승격 age 임계**(§2-2) — G1의 리전 서술은 대조했지만, 세대 가설 자체와 age 기반 승격의 구체 수치는 GC 튜닝 가이드에서 확인하지 못했다
> - **PermGen → 메타스페이스 전환 이유**(§2-1) — JEP 122가 원전인데 openjdk.org 403으로 열지 못했다
> - **`volatile`의 가시성/원자성 구분, CAS, `synchronized` vs `ReentrantLock` 차이**(§2-5) — JLS와 `java.util.concurrent` 패키지 문서로 대조하지 않았다. 표의 O/X는 통설을 옮긴 것이다
> - **`ConcurrentHashMap`이 CAS + 버킷 단위 부분 잠금**(§2-4) — 자바독으로 확인하지 않았다
> - **메모리 누수 패턴과 진단 흐름**(§2-3) — 도구(jstat·jmap·MAT) 사용법과 `-XX:+HeapDumpOnOutOfMemoryError` 동작을 문서로 대조하지 않았다
> - **JDK 21 이후 pinning 개선 여부**(§2-6) — 이전 판은 "이후 버전에서 개선되었다"고 단정했으나 근거 문서(JEP)를 열지 못해 그 문장을 뺐다
> **미작성**: JIT 컴파일과 JVM 워밍업 · 클래스 로더 계층과 위임 모델 · `CompletableFuture`와 리액티브 · JMM의 happens-before 규칙 · 힙 외 메모리(다이렉트 버퍼·네이티브 누수)
