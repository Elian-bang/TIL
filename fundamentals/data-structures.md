# 자료구조

## 30초 요약

| 구조 | 핵심 | 복잡도 | 실무에서 이것 |
|---|---|---|---|
| 배열 | 연속 메모리, 임의 접근 | 접근 O(1), 중간 삽입 O(n) | `ArrayList` |
| 연결 리스트 | 포인터로 연결 | 접근 O(n), 삽입 O(1) | `LinkedList`, **B+Tree 리프 연결** |
| 해시 테이블 | 해시 함수 + 충돌 처리 | 평균 O(1), 최악 O(n) | `HashMap`, **Redis 멱등키** |
| **B+Tree** | 리프에만 데이터, 리프 연결 | O(log n) | **DB 인덱스** |
| 힙 / 우선순위 큐 | 부모-자식 대소 관계 | 삽입·삭제 O(log n) | **우선순위 발송 큐** |
| 큐 / 스택 | FIFO / LIFO | O(1) | 메시지 큐의 기본 모델 |
| 셋 | 중복 제거 | 평균 O(1) | 발송 대상 중복 제거 |
| 트라이 | 접두사 트리 | O(문자열 길이) | 자동완성 |
| **블룸 필터** | 확률적 존재 판정 | O(k) | **대량 발송 중복 1차 필터** |

- **감각만**: O(1) < O(log n) < O(n) < O(n log n) < O(n²)
- 실무의 진짜 기준은 복잡도가 아니라 **몇 번 디스크/네트워크를 가느냐**다 (→ `[DB 인덱스](../database/basics/b-tree-index.md)` §2-1)

---

## 원리 — 왜 그런가

### 2-1. 배열 vs 연결 리스트, 이론과 실무가 갈리는 지점

교과서는 "중간 삽입은 LinkedList가 O(1)이라 빠르다"고 한다. **실무에서는 대부분 ArrayList가 빠르다.**

이유는 캐시 지역성이다.
- CPU는 메모리를 캐시 라인 단위(보통 64바이트)로 읽어 온다
- 배열은 **연속 메모리**라 하나를 읽으면 옆의 것들이 같이 캐시에 올라온다 → 순회가 매우 빠르다
- 연결 리스트는 노드가 힙 여기저기 흩어져 있어 매번 캐시 미스가 난다. 메모리 접근은 캐시 접근보다 **수십 배 느리다**
- 게다가 LinkedList의 "삽입 O(1)"은 **삽입 위치를 이미 알고 있을 때** 얘기다. 인덱스로 찾아가는 데 O(n)이 든다

> 여기서 얻을 교훈은 이것이다. **점근 복잡도는 상수를 무시한다.** 실무에서는 그 상수가 결정적일 때가 많다.
> 같은 이야기가 `[DB 인덱스](../database/basics/b-tree-index.md)` §2-7에도 있다. 랜덤 I/O 30만 번보다 순차 스캔 1번이 싸다.

### 2-2. 해시 테이블이 O(1)인 조건

**평균 O(1)은 "충돌이 적을 때"의 이야기다.** 충돌이 몰리면 그 버킷이 리스트가 되고 O(n)이 된다.

- 좋은 해시 함수 = **고르게 흩뿌리는 것**
- 부하율(load factor)이 높아지면 충돌이 늘어난다 → 테이블을 키우고 **전부 재배치(rehash)**
- Java 8은 한 버킷이 길어지면 **트리로 전환**해 최악을 O(log n)으로 완화 (→ [Java](../java/jvm-gc-concurrency.md) §2-4)

**해시는 순서를 파괴한다.** 그래서 범위 검색·정렬이 안 된다. DB 인덱스가 해시가 아닌 이유다 (→ `[DB 인덱스](../database/basics/b-tree-index.md)` §2-2)

### 2-3. 트리는 왜 균형이 중요한가

이진 탐색 트리는 정렬된 데이터를 순서대로 넣으면 **한쪽으로만 자라 사실상 연결 리스트**가 된다 → O(n).
그래서 균형을 유지하는 트리들이 나왔다.

- **레드-블랙 트리**: 색깔 규칙으로 높이를 O(log n)으로 유지. `TreeMap`, Java 8 `HashMap`의 treeify
- **B-Tree / B+Tree**: 노드 하나에 키를 여러 개 담아 높이를 극단적으로 낮춘다. 디스크 기반 저장소용 (→ `[DB 인덱스](../database/basics/b-tree-index.md)` §2-1)

> 레드-블랙 트리는 메모리용, B+Tree는 디스크용이다. **비용 모델이 다르면 자료구조가 달라진다.**

### 2-4. 힙 / 우선순위 큐는 발송 도메인에 있다

**완전 이진 트리**에서 부모가 자식보다 항상 크거나(max-heap) 작다(min-heap). 최댓값·최솟값을 O(1)에 보고, 꺼내면 O(log n)에 재정렬한다.

- 전체를 정렬할 필요 없이 **"제일 급한 것 하나"만 필요할 때** 쓴다
- **발송 도메인 적용**: 우선순위 발송 큐. 인증번호는 캠페인보다 먼저 나가야 한다
- **예약 발송**: 발송 예정 시각이 가장 이른 것부터 꺼낸다 → Redis Sorted Set이 같은 역할을 한다 (→ [Redis](../datastore/redis.md) §2-2)

### 2-5. 블룸 필터

**"확실히 없음"은 판정할 수 있고, "있음"은 확률**인 자료구조.

- 비트 배열 + 해시 함수 여러 개. 넣을 때 해당 비트들을 켠다
- 조회 시 비트 중 하나라도 0이면 → **확실히 없다**
- 전부 1이면 → **아마 있다** (다른 값들이 우연히 그 비트를 켰을 수 있다 = false positive)
- **삭제가 안 된다** (비트를 끄면 다른 값이 영향받는다)

**왜 쓰나**: 메모리가 극단적으로 적게 든다. 1억 개를 몇십 MB로 다룰 수 있다.

**발송 도메인 적용**: 1천만 건 캠페인에서 중복 수신자를 1차로 거른다. 블룸 필터가 "없다"고 하면 확실히 없으니 바로 통과, "있다"고 하면 DB를 확인한다. → **DB 조회 횟수를 크게 줄인다**
(→ [Redis](../datastore/redis.md) §2-4 캐시 관통 방어에도 같은 원리로 쓰인다)

### 2-6. 정렬, 감각만 잡는다

- **퀵**: 평균 O(n log n), 최악 O(n²). 제자리(in-place), 캐시 친화적이라 실측이 가장 빠른 경우가 많다
- **병합**: 항상 O(n log n), 안정 정렬(같은 값의 순서 유지). 추가 메모리 필요
- **힙**: 항상 O(n log n), 제자리. 상수가 커서 실측은 퀵보다 느린 편
- **Java `Arrays.sort`**: 원시 타입은 듀얼 피벗 퀵정렬, 객체는 TimSort 계열 병합 정렬
  → 왜 다른가. 객체는 **안정성이 필요하다.** 정렬 기준이 같은 원소의 순서가 뒤바뀌면 안 되는 경우가 있기 때문이다. 원시 타입은 값이 같으면 구분이 없으니 안정성이 무의미하다
  → 이 "왜"가 핵심이다

자바독 표현을 그대로 옮기면 이렇다. 원시 타입 배열은 *"The sorting algorithm is a Dual-Pivot Quicksort by Vladimir Yaroslavskiy, Jon Bentley, and Joshua Bloch."*, 객체 배열은 *"This implementation is a stable, adaptive, iterative mergesort"*이고 *"The implementation was adapted from Tim Peters's list sort for Python (TimSort)."*다. 안정성은 추측이 아니라 계약으로 적혀 있다. *"This sort is guaranteed to be stable: equal elements will not be reordered as a result of the sort."*

> 한 가지 주의. 위 불릿의 "퀵정렬 최악 O(n²)"은 교과서 퀵정렬 이야기다. JDK의 듀얼 피벗 구현에는 *"This algorithm offers O(n log(n)) performance on all data sets, and is typically faster than traditional (one-pivot) Quicksort implementations."*라고 적혀 있다. 둘을 같은 것으로 놓고 "자바 원시 타입 정렬은 최악 O(n²)"이라고 말하면 틀린다.

---

> **기준 버전**: JDK 21 (§2-2 treeify, §2-6 `Arrays.sort`). 두 항목 모두 Java 8에 도입된 동작이고 21에서도 그대로다
> **확인한 출처**:
> - [`java.util.Arrays` (JDK 21 javadoc)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Arrays.html) — §2-6 원시타입/객체 분기의 근거. 원시 타입 *"Dual-Pivot Quicksort by Vladimir Yaroslavskiy, Jon Bentley, and Joshua Bloch"*·*"offers O(n log(n)) performance on all data sets"*, 객체 *"a stable, adaptive, iterative mergesort"*·*"adapted from Tim Peters's list sort for Python (TimSort)"*·*"This sort is guaranteed to be stable"*
> - [`java.util.HashMap` (JDK 21 javadoc)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashMap.html) — §2-2 부하율과 재해시. 기본 용량 16·기본 부하율 0.75, *"the default load factor (.75) offers a good tradeoff between time and space costs"*, *"the hash table is rehashed ... so that the hash table has approximately twice the number of buckets"*
> - [OpenJDK `HashMap.java` (jdk-21+35)](https://github.com/openjdk/jdk/blob/jdk-21%2B35/src/java.base/share/classes/java/util/HashMap.java) — §2-2·§2-3의 트리 전환. *"when bins get too large, they are transformed into bins of TreeNodes, each structured similarly to those in java.util.TreeMap"*. 임계값과 그 해석은 [Java](../java/jvm-gc-concurrency.md) §2-4에 정리했다
> **미확인**:
> - **캐시 라인 64바이트와 "메모리 접근이 캐시보다 수십 배 느리다"**(§2-1) — CPU 벤더 문서로 대조하지 않았다. 자릿수 감각으로 읽어야 하고, 아키텍처마다 다르다
> - **`ArrayList`가 `LinkedList`보다 실측이 빠르다**(§2-1, 30초 요약) — 원리는 자바독이 아니라 캐시 지역성 논증이다. JDK 문서에는 이런 비교가 없고, 이 문서에서도 실측하지 않았다
> - **B+Tree·블룸 필터·트라이**(§2-3, §2-5) — 저장소 제품 문서가 아니라 일반 교과서 개념으로 서술했다. B+Tree의 실물 대조는 `[DB 인덱스](../database/basics/b-tree-index.md)` 쪽에 있다
> - **블룸 필터로 1억 개를 몇십 MB로 다룬다**(§2-5) — 파라미터(비트 수·해시 개수·오탐률)에 따라 달라지는 값인데 계산 근거를 적지 않았다. 자릿수 감각용이다
> **미작성**: 알고리즘은 [알고리즘](algorithms.md)으로 분리했다. 탐색 · 그래프 · DP · 그리디는 그쪽에 있다. 제목을 "자료구조"로 좁힌 것도 그 때문이며, **정렬(§2-6)만 자료구조와 붙어 있어 여기 남겼다**
