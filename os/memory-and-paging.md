# 없는 메모리를 있는 척하는 법

## 30초 요약

- 프로세스가 보는 주소는 **가짜(가상 주소)**다. 실제 물리 메모리와의 대응은 페이지 테이블이 들고 있고, 변환은 하드웨어(MMU)가 한다
- 그래서 **물리 메모리보다 큰 프로그램이 돌아간다.** 당장 쓰는 페이지만 올려 두고 나머지는 디스크에 둔다(요구 페이징)
- 없는 페이지를 건드리면 페이지 폴트가 나고, 자리가 없으면 **뭔가를 골라 내쫓아야 한다.** 그 선택 규칙이 페이지 교체 알고리즘이다
- **정확한 LRU는 아무도 안 쓴다.** 접근마다 순서를 갱신하는 비용이 커서 근사를 쓴다. 리눅스가 쓰는 근사는 active/inactive 두 리스트다
- **스래싱**: 페이지를 올렸다 내리는 데 시간을 다 쓰는 상태. 무서운 건 이게 스스로 악화된다는 점이다

---

## 원리 — 왜 그런가

> 이 문서는 [저장과 I/O](../database/basics/storage-and-io.md)의 원본이다. DB 버퍼 풀·페이지·LRU·프리페치는 전부 OS가 먼저 푼 문제를 DB가 자기 방식으로 다시 푼 것이다. §2-6에서 그 대응을 정리한다.

### 2-1. 왜 가상 주소인가

프로세스가 `0x00400000` 번지를 읽으면, 그 번지는 **물리 메모리의 0x00400000이 아니다.** 중간에 변환이 한 단계 있다.

```
프로세스가 보는 주소 (가상)  →  [MMU + 페이지 테이블]  →  실제 물리 주소
```

이 한 단계를 끼워 넣어서 얻는 게 셋이다.

**(1) 격리.** 프로세스마다 페이지 테이블이 따로다. A의 가상 주소 100번지와 B의 100번지는 서로 다른 물리 페이지를 가리킨다. A가 잘못된 포인터로 B의 메모리를 건드리는 일이 구조적으로 불가능해진다. [프로세스와 스레드](process-and-scheduling.md) §2-1의 "프로세스는 장애가 격리된다"가 여기서 나온다.

**(2) 연속처럼 보이기.** 프로세스는 주소가 쭉 이어져 있다고 믿지만, 물리 메모리에서는 여기저기 흩어진 페이지들이다. 물리 메모리를 조각내 쓰면서도 프로그램은 단순하게 짤 수 있다.

**(3) 물리 메모리보다 크게 쓰기.** §2-2의 주제다.

변환 단위가 페이지다. 바이트 단위로 대응표를 만들면 표가 메모리보다 커지니, 덩어리로 묶어 표의 크기를 줄인 것이다. 페이지 크기는 고정된 상수가 아니다. 커널 문서는 *"The size of each page is architecture specific"*이라고만 하고, `mmap(2)`도 크기를 못 박는 대신 `sysconf(_SC_PAGE_SIZE)`로 물어보라고 한다. x86-64 리눅스에서 4KB인 것은 그 아키텍처의 기본값이다. [저장과 I/O](../database/basics/storage-and-io.md) §2-1에서 DB가 페이지 단위로 I/O하는 것도 동기가 정확히 같다. 관리 단위를 키워 관리 비용을 줄인다.

> 변환이 매번 페이지 테이블을 뒤지면 메모리 접근이 두 배가 된다. 그래서 최근 변환 결과를 캐시하는 TLB가 CPU 안에 있다. TLB 미스가 잦으면 그것만으로 성능이 무너진다.

### 2-2. 요구 페이징은 필요할 때만 올린다

프로그램 전체를 메모리에 올리지 않는다. 실제로 건드리는 페이지만 올린다.

```
1. 프로세스가 어떤 주소를 읽는다
2. 그 페이지가 물리 메모리에 없다  → 페이지 폴트 (하드웨어 인터럽트)
3. OS가 개입: 디스크에서 그 페이지를 읽어 빈 프레임에 올린다
4. 페이지 테이블을 갱신하고, 방금 실패한 명령을 다시 실행한다
```

**페이지 폴트는 에러가 아니라 정상 동작**이다. 프로그램 시작 직후에는 거의 모든 접근이 폴트다.

여기서 나오는 결과: 물리 메모리 8GB로 합계 20GB짜리 프로세스들을 돌릴 수 있다. 각자 실제로 쓰는 부분만 올라와 있기 때문이다. 안 쓰는 페이지는 디스크(스왑 영역)에 있다.

대가는 폴트 비용이다. 메모리 접근이 나노초인데 디스크 접근은 마이크로초~밀리초다. 이 격차가 §2-5의 스래싱을 만든다.

### 2-3. 페이지 교체는 누구를 내쫓나

프레임이 꽉 찬 상태에서 폴트가 나면 하나를 골라 내보내야 한다. 잘 고르면 폴트가 줄고, 못 고르면 방금 내보낸 걸 바로 다시 불러온다.

| 알고리즘 | 규칙 | 문제 |
|---|---|---|
| **FIFO** | 가장 먼저 들어온 것 | 오래됐다고 안 쓰는 건 아니다. **Belady's Anomaly** |
| **최적(OPT)** | 앞으로 가장 늦게 쓸 것 | **미래를 알아야 한다** → 구현 불가. 다른 알고리즘 평가용 기준선 |
| **LRU** | 가장 오래 안 쓴 것 | 이론상 좋다. **그런데 정확히 구현하면 비싸다** |
| **Clock(2차 기회)** | 참조 비트를 보며 원형으로 훑는다 | LRU 근사. 교과서가 드는 대표적인 근사 |

Belady's Anomaly가 FIFO의 정체를 드러낸다. 프레임을 늘렸는데 페이지 폴트가 오히려 늘어나는 현상이다. 메모리를 더 줬는데 더 느려진다. LRU 같은 스택 알고리즘에서는 이 역전이 생기지 않는다.

정확한 LRU를 왜 안 쓰는지도 여기서 갈린다. "가장 오래 안 쓴 것"을 알려면 메모리 접근이 일어날 때마다 순서를 갱신해야 한다. 접근은 나노초마다 일어나는데 거기에 리스트 조작을 붙이면 배보다 배꼽이 크다.

Clock은 이걸 비트 하나로 근사한다.

```
각 페이지에 참조 비트 1개. 접근되면 하드웨어가 1로 세팅한다.
시곗바늘이 원형으로 돌면서:
   참조 비트가 1이면 → 0으로 바꾸고 지나간다 (한 번 봐준다)
   참조 비트가 0이면 → 이걸 내쫓는다
```

"최근에 쓰였나"만 알면 되고, 그건 비트 하나로 충분하다. **정확한 순서를 포기하는 대신 갱신 비용을 0에 가깝게 만든 거래**다.

다만 리눅스가 이 교과서 Clock을 그대로 돌리는 것은 아니다. 커널은 페이지를 active와 inactive 두 리스트로 나눠 두고, 익명 메모리와 파일 페이지를 다시 갈라 리스트를 넷으로 굴린다. 커널 문서가 *"the LRU-ordered anonymous and file, active and inactive folio lists"*라고 부르는 것이 이 구조다. `/proc/meminfo`가 `Active`·`Inactive`·`Active(anon)`·`Active(file)`을 따로 보고하는 것도 같은 구조가 밖으로 드러난 결과다. `proc_meminfo(5)`는 `Active`를 *"Memory that has been used more recently and usually not reclaimed unless absolutely necessary"*, `Inactive`를 *"Memory which has been less recently used. It is more eligible to be reclaimed for other purposes"*라고 적는다. 내쫓을 후보는 inactive 쪽에서 고른다.

커널이 이 일을 부르는 이름도 '교체'가 아니라 회수(reclaim)다. *"The process of freeing the reclaimable physical memory pages and repurposing them is called (surprise!) reclaim."* 여유 페이지가 low watermark 아래로 내려가면 `kswapd`가 깨어나 비동기로 회수하고, min watermark까지 내려가면 할당하려던 그 자리에서 직접 회수(direct reclaim)가 일어난다. 회수 대상의 큰 갈래는 페이지 캐시와 익명 메모리 둘이다.

> 커널 6.x에는 MGLRU(multi-gen LRU)라는 대안 구현도 있다. 문서는 이것을 *"an alternative LRU implementation that optimizes page reclaim"*이라 하고, 세대(generation)의 젊음과 늙음이 *"the counterparts to the active and the inactive"*라고 설명한다. `CONFIG_LRU_GEN`으로 빌드하고 `/sys/kernel/mm/lru_gen/enabled`로 켜는 선택 사항이라, 기본 동작을 이야기할 때의 기준은 여전히 active/inactive 쪽이다.

### 2-4. 지역성이 무너지면 다 무너진다

요구 페이징도 페이지 교체도 하나의 가정 위에 서 있다.

- **시간 지역성**: 방금 쓴 건 곧 또 쓴다
- **공간 지역성**: 방금 쓴 것 옆을 곧 쓴다

이 가정이 맞으면 **소수의 페이지만 올려 둬도 대부분의 접근이 적중**한다. 틀리면 전부 무너진다.

가정이 깨지는 대표적 상황은 "전체를 한 번씩만 훑는" 작업이다. 대용량 배치나 전수 스캔이 그렇다. 방금 읽은 페이지를 다시 볼 일이 없으니 시간 지역성이 성립하지 않는다. 그런데 교체 알고리즘은 그 페이지들을 "최근에 썼다"고 판단해 원래 잘 쓰이던 페이지들을 밀어낸다.

→ 이것이 [저장과 I/O](../database/basics/storage-and-io.md) §2-3에서 본 버퍼 풀 오염과 정확히 같은 문제다. OS와 DB가 같은 함정을 각자 만난다.

### 2-5. 스래싱은 스스로 악화된다

**스래싱**: 페이지를 올리고 내리는 데 시간을 대부분 쓰고, 실제 일은 거의 못 하는 상태.

무서운 건 악순환 구조다.

```
① 실행 중인 프로세스들의 활성 페이지 합계 > 물리 프레임 수
② 페이지 폴트가 급증한다
③ 프로세스들이 디스크를 기다리느라 CPU가 논다
④ 스케줄러: "CPU가 놀고 있네? 프로세스를 더 올리자"      ← 여기가 함정
⑤ 메모리 경쟁이 더 심해진다 → ②로 돌아간다
```

④가 핵심이다. CPU 이용률이 낮다는 신호를 스케줄러가 "일이 부족하다"로 읽는데, 실제 원인은 정반대다. 좋은 뜻으로 한 조치가 상황을 악화시킨다.

증상은 뚜렷하다. **CPU 사용률은 낮은데 디스크는 100%이고 시스템 전체가 느리다.** CPU만 보고 있으면 "한가한데 왜 느리지?"가 된다.

④는 중기 스케줄러가 다중 프로그래밍 정도를 자동으로 올리던 고전적 설계를 전제로 한 서술이다. 리눅스에는 그 되먹임 고리가 없고, 회수로도 감당이 안 되면 OOM 킬러가 *"selects a task to sacrifice for the sake of the overall system health"*한다. 대신 압박 자체를 재는 창구가 따로 생겼다. PSI(Pressure Stall Information)가 `/proc/pressure/memory`로 값을 내보내는데, 커널 문서는 그 배경을 *"When CPU, memory or IO devices are contended, workloads experience latency spikes, throughput losses, and run the risk of OOM kills."*라고 적는다. `some`은 일부 태스크가 멈춰 있던 시간의 비율, `full`은 idle이 아닌 태스크 전부가 동시에 멈춰 있던 시간의 비율이다. CPU 사용률만 보면 놓치는 신호가 여기 잡힌다.

해법의 방향은 하나다. 동시에 돌리는 일의 양을 줄인다. 프로세스를 일부 통째로 중단(swap out)시켜 나머지가 필요한 페이지를 확보하게 한다.

Working Set 모델이 이걸 정량화한다. "최근 Δ 시간 동안 참조한 페이지 집합"을 그 프로세스의 working set이라 하고, 모든 프로세스의 working set 합이 물리 프레임을 넘으면 스래싱이라고 본다. 넘지 않게 다중 프로그래밍 정도를 조절하는 것이 처방이다.

> 같은 이야기가 이 저장소 곳곳에 있다. [트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-10의 "커넥션 풀을 크게 잡으면 오히려 느려진다", [대용량 처리](../system-design/high-throughput.md) §2-2의 백프레셔가 전부 같은 부등호다. 처리 자원이 안 늘었는데 동시 요청만 늘리면 대기와 전환 비용만 커진다.

### 2-6. 그래서 DB는 왜 이걸 또 만드나

여기까지 오면 질문이 생긴다. OS가 페이지 캐시를 이미 해 주는데 DB는 왜 버퍼 풀을 따로 두나?

| | OS 페이지 캐시 | DB 버퍼 풀 |
|---|---|---|
| 접근 패턴 | **모른다.** 그냥 파일 오프셋 | **안다.** 인덱스 탐색인지 전수 스캔인지 |
| 교체 정책 | active/inactive 두 리스트 근사 LRU, 일괄 적용 | **스캔 오염을 아는 LRU**(InnoDB midpoint 삽입) |
| 쓰기 순서 | OS가 알아서 | **WAL이 먼저 나가야 한다**는 제약이 있다 |
| 무엇을 아나 | 페이지는 다 같은 페이지 | 이건 인덱스 루트, 저건 한 번 쓰고 버릴 리프 |

**핵심은 세 번째 줄이다.** 내구성을 지키려면 "로그가 데이터보다 먼저 디스크에 닿는다"는 순서를 강제해야 하는데([내구성과 복구](../database/basics/durability-and-recovery.md) §2-1), OS 페이지 캐시에 맡기면 그 순서를 통제할 수 없다. DB가 버퍼 풀을 갖는 이유는 캐시가 아쉬워서가 아니다. 순서와 시점을 자기가 정해야 하기 때문이다.

첫 줄도 그렇다. DB는 자기 접근 패턴을 안다. OS는 "이 페이지는 전수 스캔의 일부라 다시 안 쓸 것"을 알 수 없지만 DB는 안다. 그래서 §2-4의 함정을 DB는 피할 수 있고 OS는 못 피한다.

> 다만 PostgreSQL은 `shared_buffers`를 작게 잡고 OS 페이지 캐시를 적극 활용하는 쪽을 택했다. 이중 캐싱을 감수하는 대신 OS의 알고리즘을 공짜로 쓰는 선택이다(→ [저장과 I/O](../database/basics/storage-and-io.md) §2-2). 어느 쪽도 정답은 아니다. 서로 다른 거래일 뿐이다.

---

**다음으로 읽을 것**

- DB가 같은 문제를 어떻게 다시 푸는가 → [저장과 I/O](../database/basics/storage-and-io.md) §2-2, §2-3
- 컨텍스트 스위칭이 캐시를 무효화하는 이야기 → [프로세스와 스레드](process-and-scheduling.md) §2-1
- 메모리 부족을 실제로 어떻게 확인하나 → [리눅스 장애 조사](linux-troubleshooting.md)

---

> **기준 버전**: Linux man-pages 및 커널 문서 기준(리눅스 6.x). 알고리즘 표(§2-3)는 특정 OS에 매이지 않는 개념 수준 서술
> **확인한 출처**:
> - 커널 `admin-guide/mm/concepts` — 페이지 테이블이 가상 주소를 물리 주소로 옮긴다는 서술, "The size of each page is architecture specific", 요구 페이징(demand paging), reclaim 정의 문장, low/min watermark와 `kswapd`·direct reclaim, 회수 대상이 페이지 캐시와 익명 메모리라는 것, OOM 킬러 문장(§2-1·§2-2·§2-3·§2-5): https://docs.kernel.org/admin-guide/mm/concepts.html
> - 커널 `mm/unevictable-lru` — "the LRU-ordered anonymous and file, active and inactive folio lists"(§2-3): https://docs.kernel.org/mm/unevictable-lru.html
> - 커널 `mm/multigen_lru` — MGLRU가 "an alternative LRU implementation"이고 세대가 "the counterparts to the active and the inactive"라는 서술(§2-3): https://docs.kernel.org/mm/multigen_lru.html
> - 커널 `admin-guide/mm/multigen_lru` — `CONFIG_LRU_GEN`·`/sys/kernel/mm/lru_gen/enabled`로 켜는 선택 사항이라는 것(§2-3): https://docs.kernel.org/admin-guide/mm/multigen_lru.html
> - `proc_meminfo(5)` — `Active`/`Inactive`의 정의 원문과 `Active(anon)`·`Inactive(file)` 같은 분리 필드의 존재(§2-3): https://man7.org/linux/man-pages/man5/proc_meminfo.5.html
> - `mmap(2)` — 페이지 크기를 `sysconf(_SC_PAGE_SIZE)`로 얻는다는 규정, `MAP_NORESERVE`의 스왑 예약 서술(§2-1·§2-2): https://man7.org/linux/man-pages/man2/mmap.2.html
> - 커널 `accounting/psi` — "When CPU, memory or IO devices are contended…" 문장, `/proc/pressure/`, `some`/`full`의 정의(§2-5): https://docs.kernel.org/accounting/psi.html
> **미확인**:
> - **Belady's Anomaly가 스택 알고리즘에서 발생하지 않는다는 성질**(§2-3) — 원전은 Mattson·Gecsei·Slutz·Traiger, "Evaluation techniques for storage hierarchies", IBM Systems Journal 9(2), 1970(doi:10.1147/sj.92.0078)이다. 서지 정보만 확인했고 본문을 읽지 못해 증명을 대조하지 못했다
> - **Working Set 모델의 정의**(§2-5) — 원전은 Denning, "The working set model for program behavior", CACM 11(5), 1968이다. `denninginstitute.com`의 PDF는 텍스트 추출이 안 되는 스캔본이라 정의 문장을 원문으로 확인하지 못했다. 본문의 "최근 Δ 시간 동안 참조한 페이지 집합"은 2차 자료와 대조해 어긋나지 않는 것까지만 확인했다
> - **FIFO·OPT·LRU·Clock이라는 알고리즘 이름과 비교표**(§2-3) — 교과서 개념으로 서술했고 원전 대조는 하지 않았다. 커널 문서에는 이 이름들이 없다
> - **x86-64 기본 페이지 크기가 4KB**(§2-1) — 커널 문서는 아키텍처별이라고만 하고 숫자를 적지 않는다. 4KB는 통용되는 값이고 문서 문장으로 확인하지 못했다
> - **스래싱의 ①~⑤ 되먹임 고리**(§2-5) — 교과서 서술이다. 리눅스에 그 중기 스케줄러가 없다는 점은 본문에 적었지만, 고전 설계가 실제로 그렇게 동작했다는 서술은 원전으로 대조하지 않았다
> - **TLB**(§2-1) — 개념만 언급했고 문서를 대조하지 않았다
> **미작성**: 세그먼테이션 · 다단계 페이지 테이블 · 내부/외부 단편화 · 스왑 영역 운영 · `vm.swappiness`와 회수 튜닝
