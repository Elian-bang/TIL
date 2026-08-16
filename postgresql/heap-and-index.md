# 인덱스는 왜 힙을 못 벗어나나

## 30초 요약

- PostgreSQL에는 클러스터드 인덱스가 없다. **모든 인덱스가 세컨더리**이고 전부 `ctid`(물리 주소)를 가리킨다
- 그래서 행이 이사하면 그 행을 가리키는 모든 인덱스를 고쳐야 한다. **UPDATE 비용에 인덱스 개수가 직접 붙는다**
- **HOT update**가 그걸 피한다. 조건은 둘이다. 인덱스가 참조하는 컬럼을 안 바꿨을 것, 같은 페이지에 공간이 있을 것
- 인덱스만 읽고 끝낼 수 없는 이유는 하나다. *"가시성 정보는 인덱스가 아니라 힙에만 있다."* **visibility map**이 그 예외를 만든다
- 인덱스 타입이 6종인 건 취향이 아니라 **B-tree가 구조적으로 못 하는 질의**가 있어서다. 포함 관계, 겹침, 물리 순서 상관 같은 것들이 여기 걸린다

---

## 원리 — 왜 그런가

> [저장과 I/O](../database/basics/storage-and-io.md) §2-5에서 "힙 vs 클러스터드"의 갈림길을 봤다. 이 문서는 **힙을 택한 쪽에서 파생되는 것들**이다.

### 2-1. 모든 인덱스가 세컨더리다

InnoDB에서 PK 인덱스는 곧 테이블이다. PostgreSQL에는 그런 특별한 인덱스가 없다.

```
힙(테이블)  : 순서 없이 저장. 각 행의 주소 = ctid = (페이지 번호, 페이지 안 슬롯 번호)
모든 인덱스 : "값 → ctid"
```

**PK 인덱스도 이메일 인덱스도 완전히 동등하다.** 둘 다 값을 받아 ctid를 돌려줄 뿐이다.

여기서 두 가지가 갈린다.

**(1) 조회는 한 번에 도달한다.** InnoDB는 세컨더리 인덱스 → PK → 클러스터드 인덱스로 두 번 탐색하는데([DB 인덱스](../database/basics/b-tree-index.md) §2-3), PostgreSQL은 ctid가 물리 주소라 바로 그 페이지로 간다. 커버링 인덱스의 효과가 InnoDB만큼 극적이지 않은 이유다.

**(2) 대신 이사에 취약하다.** ctid는 물리 위치이므로 행이 다른 페이지로 옮겨 가면 모든 인덱스의 포인터가 낡는다. §2-2의 주제다.

**PK 선택 압박도 InnoDB보다 훨씬 약하다.** PK가 물리 순서를 정하지 않으니 UUID를 PK로 써도 페이지 분할이 몰리지 않고, PK가 길어도 다른 인덱스가 뚱뚱해지지 않는다. [데이터 모델링](../database/basics/data-modeling.md) §2-1의 "PK는 짧아야 한다"는 조언은 InnoDB에서 훨씬 무겁게 걸린다.

### 2-2. HOT update가 인덱스 갱신을 피하는 법

§2-1(2)의 문제가 실제로 어떻게 나타나는지부터.

```
발송 이력 테이블에 인덱스 5개.  UPDATE ... SET status='SENT' WHERE id=1;
→ 새 버전이 다른 페이지에 생긴다 (MVCC, → [MVCC와 VACUUM](mvcc-and-vacuum.md) §2-1)
→ ctid가 바뀐다
→ 인덱스 5개를 전부 갱신해야 한다  ← status 하나 바꿨는데
```

**HOT(Heap-Only Tuple)** 이 이 낭비를 없앤다. 공식 문서가 조건을 둘로 명시한다.

> 1. *"The update does not modify any columns referenced by the table's indexes"* (요약 인덱스 제외)
> 2. *"There is sufficient free space on the page containing the old row for the updated row."*

둘 다 만족하면 **새 버전을 같은 페이지 안에 만들고, 옛 튜플에서 새 튜플로 포인터를 건다.** 인덱스는 그대로 둔다. 공식 문서 표현으로 *"인덱스는 항상 원본 행 버전의 page item identifier를 참조"*하므로, 인덱스를 따라가면 옛 자리에 닿고 거기서 체인을 타고 새 버전을 찾는다.

> *"New index entries are not needed to represent updated rows."*

**두 조건에서 실무 규칙이 그대로 유도된다.**

**(1) 자주 바뀌는 컬럼에는 인덱스를 걸지 마라.** `status`처럼 상태 전이가 잦은 컬럼에 인덱스를 걸면 그 순간부터 모든 상태 변경이 HOT을 잃는다. 조회 하나 빠르게 하려고 UPDATE 전체를 비싸게 만드는 거래다.

**(2) `fillfactor`를 낮추면 HOT 확률이 올라간다.** 공식 문서도 *"fillfactor를 줄이면 HOT update에 충분한 페이지 공간이 있을 가능성을 높일 수 있다"*고 한다. 페이지를 꽉 채우지 않고 여유를 남겨 두는 것이고, 공간을 팔아 쓰기 성능을 사는 거래다. 갱신이 잦은 테이블에서만 의미가 있다.

> **`fillfactor`는 InnoDB의 페이지 분할 회피와 목적이 다르다.** InnoDB는 삽입 위치가 랜덤일 때 분할을 줄이려 여유를 두지만, PostgreSQL은 같은 페이지 안에 새 버전을 만들려고 여유를 둔다. 비슷해 보이는 손잡이가 서로 다른 문제를 푼다.

### 2-3. Index-Only Scan은 왜 조건부인가

"필요한 컬럼이 전부 인덱스에 있으면 힙에 안 가도 되지 않나." PostgreSQL에서는 **원칙적으로 안 된다.** 이유가 명확하다.

> *"Visibility information is not stored in index entries, only in heap entries; so at first glance it would seem that every row retrieval would require a heap access anyway."*

**인덱스 엔트리는 그 행이 지금 보이는 버전인지 모른다.** §2-1의 MVCC 때문에 힙에는 여러 버전이 있고, 어느 것이 내 스냅샷에 보이는지는 힙의 xmin/xmax를 봐야 안다. 그래서 인덱스만으로는 답을 못 낸다.

**visibility map이 예외를 만든다.**

> *"PostgreSQL tracks, for each page in a table's heap, whether all rows stored in that page are old enough to be visible to all current and future transactions."*

페이지 단위로 "이 페이지는 전부 모두에게 보인다"는 비트 하나를 둔다. 그 비트가 켜져 있으면 **가시성 확인이 필요 없으니 힙에 안 가도 된다.**

```
인덱스에서 후보를 찾음
  → 그 힙 페이지의 visibility map 비트 확인
     켜짐 → 힙 접근 없이 인덱스 값만으로 반환   ← Index-Only Scan
     꺼짐 → 힙에 가서 가시성 확인
```

**비트맵이 작다는 게 핵심이다.** 공식 문서 표현으로 *"visibility map은 그것이 서술하는 힙보다 네 자릿수(10⁴배) 작다."* 확인 비용이 사실상 없다.

여기서 [MVCC와 VACUUM](mvcc-and-vacuum.md)과 연결된다. visibility map 비트를 켜는 게 **VACUUM**이다.

> **VACUUM을 안 돌리면 커버링 인덱스를 만들어도 Index-Only Scan이 안 나온다.** 인덱스 설계 문제로 보이는 것이 사실은 청소 문제인 경우가 있다. `EXPLAIN (ANALYZE, BUFFERS)`에서 `Heap Fetches`가 크면 그 신호다.

### 2-4. INCLUDE가 payload와 search key를 나눈다

커버링 인덱스를 만들 때 예전에는 필요한 컬럼을 그냥 인덱스 컬럼으로 붙였다. `INCLUDE`는 "검색에 안 쓰고 실어만 나르는 컬럼"을 따로 선언한다.

```sql
CREATE INDEX tab_x_y ON tab(x) INCLUDE (y);
--                        ↑ search key   ↑ payload
```

일반 복합 인덱스 `(x, y)`와 무엇이 다른가. 셋이다.

| | `(x, y)` | `(x) INCLUDE (y)` |
|---|---|---|
| `y`의 타입 | **인덱스가 다룰 수 있는 타입이어야** | **아무 타입이나** — 저장만 하고 해석 안 함 |
| UNIQUE 걸면 | `x, y` **조합**이 유니크 | **`x`만** 유니크 |
| B-Tree 상위 레벨 | `y`도 올라갈 수 있음 | **확실히 제외됨** → 상위 노드가 작다 |

**두 번째가 실무에서 결정적이다.** "이메일은 유니크여야 하는데 조회할 때 이름도 같이 필요하다"면서 `(email, name)`으로 만들면 유니크 제약이 조합에 걸려 의도가 깨진다. `(email) INCLUDE (name)`이 정확한 표현이다.

세 번째는 [DB 인덱스](../database/basics/b-tree-index.md) §2-1의 팬아웃 이야기다. 상위 노드가 작아야 한 페이지에 더 많은 자식이 들어가고 트리가 얕아진다. 공식 문서도 payload를 명시적으로 non-key로 선언하면 *"상위 레벨의 튜플을 작게 유지하는 것이 확실해진다"*고 한다.

### 2-5. 인덱스 타입이 6종인 이유

[DB 인덱스](../database/basics/b-tree-index.md) §2-2에서 "인덱스가 전부 B+Tree는 아니다"를 봤다. PostgreSQL은 그걸 **제품 안에서 6종으로 구현해 둔 흔치 않은 사례**다.

| 타입 | 잘하는 질의 | 쓰는 자리 |
|---|---|---|
| **B-tree** (기본) | `<` `<=` `=` `>=` `>` `BETWEEN` `IN` `IS NULL`, **앞고정** `LIKE` | 대부분 |
| **Hash** | `=` 만 | 등치만 하고 인덱스를 작게 |
| **GiST** | 기하·범위의 **겹침·포함** | 좌표, 기간 범위 |
| **SP-GiST** | 공간 분할이 유리한 구조 | 비균등 분포 공간 데이터 |
| **GIN** | **여러 값을 담은 컬럼 안의 원소** 검색 | `jsonb`, 배열, 전문 검색 |
| **BRIN** | 범위. **물리 순서와 상관된 컬럼**에서만 | 시계열 대용량 |

GIN을 이해하는 열쇠는 공식 문서의 표현이다. *"배열처럼 여러 컴포넌트 값을 포함하는 데이터에 적합"*하며 각 컴포넌트 값마다 별도 항목을 저장한다. 즉 **역색인**이고([Elasticsearch](../datastore/elasticsearch.md) §2-1과 같은 구조), 그래서 `jsonb`의 특정 키 포함 여부나 배열 원소 검색이 빠르다.

**BRIN은 발상이 완전히 다르다.** 인덱스가 값마다 항목을 갖는 게 아니라 연속된 블록 범위마다 최소·최대만 요약한다.

```
블록 1~128   : sent_at 이 2026-08-01 ~ 2026-08-03
블록 129~256 : sent_at 이 2026-08-03 ~ 2026-08-05
...
```

조건이 `sent_at >= '2026-08-04'`면 **범위가 안 겹치는 블록 뭉치를 통째로 건너뛴다.** 인덱스 크기도 극단적으로 작다. 수억 행 테이블에 수십 KB 수준이다.

**대신 전제가 강하다.** 공식 문서가 *"열의 값이 테이블 행의 물리적 순서와 잘 상관될 때 가장 효과적"*이라고 못 박는다. 시간순으로 append되는 로그라면 완벽하지만, 값이 흩어져 들어오면 모든 블록의 범위가 겹쳐서 아무것도 못 건너뛴다. 인덱스가 있으나 마나가 된다.

> **발송 이력이 BRIN의 교과서적 대상이다.** `sent_at`으로 append만 되므로 물리 순서와 시간 순서가 일치한다. B-tree로 같은 걸 하면 인덱스가 수 GB인데 BRIN이면 수십 KB다. 단 `UPDATE`가 잦으면 행이 이사하면서 상관이 깨지므로(§2-2), 이력처럼 불변인 데이터라야 성립한다.

### 2-6. 힙이 무순서라서 생긴 Bitmap Heap Scan

실행 계획에서 자주 보이는데 InnoDB에는 대응물이 없다.

**문제**: 인덱스가 돌려주는 ctid는 인덱스 순서다. 힙에서는 그게 랜덤한 위치다. 1만 건을 가져오면 랜덤 I/O 1만 번이다([저장과 I/O](../database/basics/storage-and-io.md) §2-7의 그 부등호).

**해법**: 한 번에 다 모아서 정렬한 뒤 순서대로 읽는다.

```
① Bitmap Index Scan : 인덱스를 훑어 ctid를 비트맵에 모은다 (아직 힙 접근 없음)
② (여러 인덱스면 비트맵끼리 AND / OR 로 합친다)
③ Bitmap Heap Scan  : 비트맵을 페이지 번호 순으로 훑으며 힙을 읽는다  ← 순차에 가까워진다
```

**②가 부수 효과로 강력하다.** 조건이 여러 개고 각각 인덱스가 있으면 인덱스를 여러 개 동시에 쓸 수 있다. InnoDB에도 인덱스 머지가 있지만 PostgreSQL 쪽이 훨씬 일반적으로 쓰인다.

> [DB 인덱스](../database/basics/b-tree-index.md) §2-2의 용어 주의를 여기서 다시 확인하게 된다. **`Bitmap Index Scan`은 저장된 비트맵 인덱스가 아니라 실행 중에 만드는 임시 비트맵이다.** PostgreSQL에 비트맵 *인덱스*는 없다.

### 2-7. TOAST는 큰 값을 행 밖으로 뺀다

페이지가 8KB인데 그보다 큰 값을 넣으면 어떻게 되나. **행을 페이지에 걸쳐 저장하지 않는다.** 대신 TOAST가 처리한다.

```
값이 크다 → ① 압축을 먼저 시도
          → ② 그래도 크면 조각내서 별도 TOAST 테이블에 저장
          → ③ 원래 행에는 포인터만 남는다
```

결과가 §2-1의 페이지 효율과 이어진다. **큰 텍스트 컬럼이 있어도 그 컬럼을 안 읽는 쿼리는 비용을 안 낸다.** 행 자체는 작게 유지되므로 페이지 하나에 많은 행이 들어간다.

> [데이터 모델링](../database/basics/data-modeling.md) §2-4에서 "1:1 관계는 대개 합쳐도 된다"고 했는데 PostgreSQL에서는 더 그렇다. 큰 텍스트를 따로 테이블로 빼는 수직 분할을 **TOAST가 자동으로 해 주기 때문이다.** 직접 나누기 전에 이미 해결돼 있는지 확인하는 게 낫다.

### 2-8. 인덱스가 무엇을 가리키느냐에서 시작된 차이

| 항목 | InnoDB | PostgreSQL |
|---|---|---|
| 인덱스가 가리키는 것 | **PK 값** | **ctid**(물리 주소) |
| 세컨더리 조회 | 두 번 탐색 | **한 번에 도달** |
| PK 선택의 무게 | **매우 무겁다**(물리 순서·모든 인덱스에 복제) | 가볍다 |
| UPDATE와 인덱스 | 바뀐 컬럼의 인덱스만 | **행 이사 시 전부** → HOT으로 회피 |
| 커버링 인덱스 효과 | **극적**(북마크 룩업 제거) | 있지만 덜 극적 + **VACUUM 의존** |
| 인덱스 타입 | B+Tree 중심(+ FULLTEXT, R-Tree) | **6종** |
| 큰 값 | row format에 따라 오버플로 페이지 | **TOAST**(압축 → 분리) |

"커버링 인덱스 효과가 VACUUM에 의존한다"는 줄이 PostgreSQL 특유의 함정이다. 인덱스를 아무리 잘 설계해도 청소가 안 돌면 Index-Only Scan이 안 나온다. **인덱스 튜닝과 VACUUM 관리가 분리된 작업이 아니다.**

---

**다음으로 읽을 것**

- dead tuple과 VACUUM이 이 문서 곳곳에 걸리는 이유 → [MVCC와 VACUUM](mvcc-and-vacuum.md)
- 힙 vs 클러스터드의 갈림길 → [저장과 I/O](../database/basics/storage-and-io.md) §2-5
- 인덱스 자료구조 일반론 → [DB 인덱스](../database/basics/b-tree-index.md) §2-2

---

> **기준 버전**: PostgreSQL 17
> **확인한 출처**:
> - [PostgreSQL 73.7 Heap-Only Tuples](https://www.postgresql.org/docs/current/storage-hot.html) — HOT 성립 조건 두 가지 원문, *"New index entries are not needed to represent updated rows"*, *"Indexes always refer to the page item identifier of the original row version"*, fillfactor를 낮추면 HOT 가능성이 올라간다는 서술
> - [PostgreSQL 11.9 Index-Only Scans and Covering Indexes](https://www.postgresql.org/docs/current/indexes-index-only-scans.html) — *"Visibility information is not stored in index entries, only in heap entries"*, visibility map의 정의와 *"four orders of magnitude smaller than the heap"*, `INCLUDE`의 payload 성격·UNIQUE가 search key에만 적용된다는 점·suffix truncation
> - [PostgreSQL 11.2 Index Types](https://www.postgresql.org/docs/current/indexes-types.html) — 타입 6종과 각 지원 연산자, GIN이 "여러 컴포넌트 값을 포함하는 데이터"용이라는 서술, BRIN이 "물리적 순서와 상관될 때 가장 효과적"이라는 서술
> **미확인**: §2-6 Bitmap Heap Scan의 동작 서술 · §2-7 TOAST의 임계값과 압축 전략 · `CREATE INDEX CONCURRENTLY`의 제약 — 공식 문서 대조하지 않았다
> **미작성**: 부분 인덱스(partial index) · 표현식 인덱스 · 연산자 클래스 · 인덱스 팽창과 REINDEX
