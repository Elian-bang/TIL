# 락과 타입 — 갭 락 없이 팬텀을 막는 법

## 30초 요약

- PostgreSQL의 기본 격리 수준은 **READ COMMITTED**다. InnoDB(REPEATABLE READ)와 다르다는 것부터 출발점이 갈린다
- 그런데 PostgreSQL의 REPEATABLE READ는 **표준보다 강하다** — **갭 락 없이 팬텀을 막는다.** 공식 문서가 표에 *"Allowed, but not in PG"*라고 적어 뒀다
- 방식이 정반대다. **InnoDB는 막아서 방지하고(대기), PostgreSQL은 실패시켜서 방지한다(롤백).**
- 그래서 **SERIALIZABLE·REPEATABLE READ에서 재시도 로직은 선택이 아니라 필수**다 — 공식 문서가 `SQLSTATE 40001`을 일반적으로 처리하라고 지시한다
- **advisory lock은 롤백해도 안 풀린다**(세션 수준). 트랜잭션 의미를 안 따르는 게 기능이자 함정이다

---

## 원리 — 왜 그런가

> [동시성 제어 이론](../database/basics/concurrency-theory.md)에서 직렬 가능성과 2PL을 봤다. 이 문서는 **PostgreSQL이 그 목표를 락이 아닌 방법으로 달성하는 이야기**다.

### 2-1. 기본이 READ COMMITTED라는 것

공식 문서가 명시한다 — *"Read Committed is the default isolation level in PostgreSQL."*

**InnoDB의 기본은 REPEATABLE READ**다([트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-2). 이 차이가 이식할 때 사고를 만든다.

```
InnoDB(RR)   : 트랜잭션 시작 시 스냅샷 한 번 → 몇 번을 읽어도 같은 값
PostgreSQL(RC): 쿼리마다 스냅샷 새로 → 같은 트랜잭션 안에서도 값이 달라질 수 있다
```

**MySQL에서 "트랜잭션 안이니 값이 안 변한다"를 전제로 짠 코드가 PostgreSQL에서 깨진다.** 잔액을 읽고 계산한 뒤 다시 읽었더니 다른 값인 상황이 정상 동작이 된다.

> 기본값을 바꾸기보다 **필요한 트랜잭션만 `BEGIN ISOLATION LEVEL REPEATABLE READ`로 올리는 것**이 정석이다. 올리면 §2-3의 재시도 의무가 따라온다는 걸 알고 올려야 한다.

### 2-2. RR이 표준보다 강하다 — 갭 락이 없는 이유

[트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-5에서 InnoDB가 팬텀을 막으려고 **갭 락**을 도입한 걸 봤다. "앞으로 들어올 자리"까지 잠그는 방식이다.

**PostgreSQL에는 갭 락이 없다.** 그런데도 팬텀이 안 생긴다. 공식 문서 표에 그렇게 적혀 있다.

| 격리 수준 | 팬텀 읽기 |
|---|---|
| Repeatable read | *"Allowed, but not in PG"* |

> *"PostgreSQL's Repeatable Read implementation does not allow phantom reads. This is acceptable under the SQL standard because the standard specifies which anomalies must not occur at certain isolation levels; higher guarantees are acceptable."*

**어떻게 막나 — 잠그는 대신 실패시킨다.**

RR 트랜잭션은 **시작 시점의 스냅샷 하나로 끝까지 간다.** 읽기는 그 스냅샷만 보므로 팬텀이 애초에 안 보인다. 문제는 [트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-4의 **current read** — 쓰기는 최신을 봐야 한다는 것이었다. PostgreSQL은 이 충돌을 **대기가 아니라 에러**로 처리한다.

> *"a repeatable read transaction cannot modify or lock rows changed by other transactions after the repeatable read transaction began."*
>
> ```
> ERROR: could not serialize access due to concurrent update
> ```

**여기가 두 엔진의 철학이 갈리는 지점이다.**

| | InnoDB | PostgreSQL |
|---|---|---|
| 팬텀 방지 | **갭 락으로 미리 막는다** | **스냅샷 + 충돌 시 롤백** |
| 충돌 시 | **기다린다** (그러다 데드락) | **즉시 에러** |
| 대가 | 락 범위가 넓어져 **데드락 증가** | **재시도 로직이 필수** |

**어느 쪽도 공짜가 아니다.** InnoDB는 "대기와 데드락"을 내고, PostgreSQL은 "롤백과 재시도"를 낸다. **MySQL 감각으로 PostgreSQL을 쓰면 "왜 자꾸 실패하지"가 되고, 반대면 "왜 이렇게 데드락이 많지"가 된다.**

### 2-3. SERIALIZABLE = SSI — 기다리지 않고 실패시킨다

PostgreSQL의 SERIALIZABLE은 **락으로 줄을 세우지 않는다.** 일단 다 돌게 두고 **커밋 시점에 "이 실행이 직렬 가능했나"를 판정**한다 — [동시성 제어 이론](../database/basics/concurrency-theory.md) §2-6의 낙관적 계열이다.

> *"This level emulates serial transaction execution for all committed transactions; as if transactions had been executed one after another, serially, rather than concurrently."*

**핵심은 "커밋된 트랜잭션들에 대해서"라는 단서다.** 직렬 실행처럼 보이게 만드는 방법이 **어긋난 트랜잭션을 커밋시키지 않는 것**이다.

```
ERROR: could not serialize access due to read/write dependencies among transactions
```

**그래서 재시도가 의무가 된다.** 공식 문서가 두 번 강조한다.

> *"applications using this level must be prepared to retry transactions due to serialization failures."*
>
> *"It is important that an environment which uses this technique have a generalized way of handling serialization failures (which always return with an SQLSTATE value of '40001'), because it will be very hard to predict exactly which transactions might contribute to the read/write dependencies."*

**"어느 트랜잭션이 희생될지 예측하기 매우 어렵다"**는 게 중요하다. 특정 쿼리만 방어하는 식으로는 안 되고, **`40001`을 잡아 트랜잭션 전체를 처음부터 재시도하는 공통 장치**가 있어야 한다.

**그리고 읽은 값을 커밋 전에 믿으면 안 된다.**

> *"any data read from a permanent user table not be considered valid until the transaction which read it has successfully committed... applications must not depend on results read during a transaction that later aborted."*

**트랜잭션 중간에 읽은 값으로 외부에 무언가를 하면 안 된다는 뜻이다.** 그 트랜잭션이 나중에 직렬화 실패로 죽으면 그 값은 없던 일이 되는데, 이미 나간 발송은 되돌릴 수 없다. → [대용량 처리](../system-design/high-throughput.md) §2-6의 멱등성이 여기서도 최종 방어선이다.

> **재시도 구현의 함정**: 재시도할 때 **트랜잭션을 처음부터** 다시 해야 한다. 공식 문서 표현도 *"abort the current transaction and retry the whole transaction from the beginning"*이다. 실패한 쿼리만 다시 던지면 스냅샷이 그대로라 또 실패한다.

### 2-4. 행 락이 4단계인 이유 — FK

InnoDB의 행 락은 사실상 공유/배타 둘이다. PostgreSQL은 **넷**이고, 강도 순으로 이렇다.

| 모드 | 무엇을 막나 |
|---|---|
| `FOR UPDATE` | 가장 강함. 다른 모든 행 락과 수정·삭제를 막는다 |
| `FOR NO KEY UPDATE` | `FOR UPDATE`보다 약함 — **`FOR KEY SHARE`는 통과시킨다** |
| `FOR SHARE` | 공유 락. `FOR SHARE`·`FOR KEY SHARE`끼리는 공존 |
| `FOR KEY SHARE` | 가장 약함. **키 값을 바꾸는 UPDATE와 DELETE만** 막는다 |

**왜 이렇게 잘게 나눴나 — 외래 키 때문이다.**

자식 행을 INSERT하면 **부모가 존재하는지 확인**하고, 확인하는 동안 **부모가 사라지면 안 된다.** 그래서 부모 행에 락을 건다. 그런데 이때 `FOR UPDATE` 같은 강한 락을 걸면:

```
캠페인(부모) 1건에 발송건(자식) 1만 건을 동시에 INSERT
→ 전부 같은 부모 행에 락을 요청
→ 직렬화된다. 동시성이 죽는다
```

**`FOR KEY SHARE`가 이걸 푼다.** 자식이 부모에 거는 건 **"키 값만 안 바뀌면 된다"**는 최소한의 락이다. 부모의 이름이나 설명이 바뀌는 건 상관없으니 `FOR NO KEY UPDATE`와 공존한다. **부모를 수정하는 작업과 자식을 추가하는 작업이 서로 안 막힌다.**

> **`SELECT ... FOR UPDATE`를 습관적으로 쓰면 이 설계가 무의미해진다.** 정말 그 행을 바꿀 게 아니라 "사라지지만 않으면 된다"면 `FOR KEY SHARE`나 `FOR NO KEY UPDATE`가 맞다. 락 모드를 고르는 것만으로 경합이 크게 줄어드는 자리다.

### 2-5. `SKIP LOCKED` — DB를 큐로 쓰기

[트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-8에서 본 패턴의 PostgreSQL판이다.

```sql
SELECT * FROM send_queue
 WHERE status = 'READY'
 ORDER BY reserved_at
 LIMIT 100
 FOR UPDATE SKIP LOCKED;
```

**다른 워커가 이미 잡은 행을 기다리지 않고 건너뛴다.** 워커 N대가 같은 테이블을 경합 없이 나눠 소비한다.

**이게 없으면** 워커들이 전부 같은 상위 100건에 몰려 **한 대만 일하고 나머지는 대기**한다. 있으면 각자 다른 100건을 가져간다.

> **"큐가 필요하면 Kafka를 써야 하나"에 대한 실무적 답**이기도 하다. 초당 수백 건 규모라면 **이미 있는 DB로 충분**하고, 브로커를 하나 더 운영하는 비용을 안 내도 된다([메시징 기초](../messaging/basics.md) §2-2의 "규모로 판단하라"와 같은 이야기). 상태 전이와 이력이 같은 트랜잭션 안에 들어간다는 이점도 크다.
>
> 대신 **처리량 한계가 명확하다** — 폴링이고, 테이블이 커지면 dead tuple이 쌓여([MVCC와 VACUUM](mvcc-and-vacuum.md) §2-2) VACUUM 부담이 된다. 완료된 행을 제때 다른 테이블로 옮기거나 파티션째 버리는 설계가 같이 가야 한다.

### 2-6. advisory lock — MVCC에 안 맞는 것을 위한 탈출구

행이나 테이블이 아니라 **애플리케이션이 정한 임의의 숫자**에 거는 락이다.

> *"PostgreSQL provides a means for creating locks that have application-defined meanings. These are called advisory locks, because the system does not enforce their use — it is up to the application to use them correctly."*

**"시스템이 강제하지 않는다"가 핵심**이다. 락을 잡든 말든 데이터는 그냥 바뀐다. **모두가 규칙을 지킬 때만 동작한다.**

**언제 쓰나** — 공식 문서는 *"MVCC 모델에 어색하게 들어맞는 잠금 전략"*에 유용하다고 한다. 즉 **잠글 행 자체가 없는 경우**다.

```
"이 배치는 전체 클러스터에서 한 번에 하나만 돌아야 한다"
→ 잠글 행이 없다. 테이블에 플래그 컬럼을 만들 수도 있지만…
```

**플래그 컬럼보다 나은 이유도 문서에 있다** — *"advisory locks are faster, avoid table bloat, and are automatically cleaned up by the server at the end of the session."* 특히 **테이블 블로트를 피한다**는 게 PostgreSQL다운 근거다. 플래그를 UPDATE로 켰다 껐다 하면 그때마다 dead tuple이 쌓인다(§[MVCC와 VACUUM](mvcc-and-vacuum.md) §2-2).

**⚠️ 가장 중요한 함정 — 세션 수준 advisory lock은 트랜잭션 의미를 안 따른다.**

> *"a lock acquired during a transaction that is later rolled back will still be held following the rollback"*

**롤백해도 락이 안 풀린다.** 트랜잭션이 실패했으니 당연히 정리됐겠거니 하면 그 락은 세션이 끝날 때까지 남는다. **커넥션 풀 환경에서는 그 커넥션이 반납된 뒤 다른 요청이 받아 가므로, 원인을 찾기가 매우 어려운 교착이 된다.**

→ 대부분의 경우 **트랜잭션 수준 advisory lock**(`pg_advisory_xact_lock`)을 쓰는 게 안전하다. 트랜잭션이 끝나면 자동으로 풀린다.

### 2-7. 타입 — 모델링을 DB에서 끝내는 것들

PostgreSQL의 타입은 편의 기능이 아니라 **[데이터 모델링](../database/basics/data-modeling.md) §2-5의 "제약을 DB에 새긴다"를 더 멀리 밀어붙인 것**이다.

**`jsonb` vs `json`**

| | `json` | `jsonb` |
|---|---|---|
| 저장 | 텍스트 그대로 | **파싱된 이진 형태** |
| 입력 순서·공백 | 보존 | 버림 |
| 인덱스 | 사실상 불가 | **GIN 인덱스 가능** |

**실무에서는 사실상 `jsonb`만 쓴다.** `json`은 원본 텍스트를 그대로 보존해야 하는 드문 경우용이다. `jsonb` + GIN이면 **"이 키를 포함하는 문서"를 인덱스로 찾을 수 있다**([힙과 인덱스](heap-and-index.md) §2-5).

> **다만 [정규화](../database/basics/normalization.md) §2-1의 판단 기준이 그대로 적용된다** — 그 안의 값으로 **검색·조인·집계를 하면 컬럼으로 빼고**, 통째로 읽고 쓰기만 하면 `jsonb`가 낫다. "스키마를 안 정해도 되니 편하다"로 쓰기 시작하면 나중에 그 안에서 검색하게 되고, 그때 이미 늦다.

**range 타입**

`tstzrange` 같은 타입이 **"시작~끝"을 한 값으로** 다룬다. 진짜 값어치는 **겹침을 DB가 막을 수 있다**는 것이다.

```
발송 예약 시간대가 겹치면 안 된다
→ 애플리케이션에서 검사하면 동시 요청에서 뚫린다 (전형적 경쟁 조건)
→ 배타 제약(EXCLUDE)으로 DB에 새기면 구조적으로 못 뚫린다
```

**[데이터 모델링](../database/basics/data-modeling.md) §2-5에서 "최종 방어선은 DB 제약"이라고 했는데, 유니크로는 "겹침"을 표현할 수 없다.** range + 배타 제약이 그 빈자리를 채운다.

**`TIMESTAMPTZ`**

이름과 달리 **타임존을 저장하지 않는다.** 입력받은 시각을 UTC로 변환해 저장하고, 읽을 때 세션 타임존으로 변환해 보여준다. **`TIMESTAMP`(타임존 없음)는 "2026-08-16 14:00"이라는 문자열에 가깝고, `TIMESTAMPTZ`는 "지구의 그 순간"이다.**

**발송 예약처럼 절대 시점이 중요한 데이터는 `TIMESTAMPTZ`가 맞다.** 서버 타임존이 바뀌거나 서머타임이 끼면 `TIMESTAMP`는 의미가 달라진다.

### 2-8. 그래서 InnoDB와 무엇이 다른가

| 항목 | InnoDB | PostgreSQL |
|---|---|---|
| 기본 격리 수준 | REPEATABLE READ | **READ COMMITTED** |
| RR에서 팬텀 | **갭 락으로 차단** | **스냅샷으로 안 보임** (표준보다 강함) |
| 쓰기 충돌 시 | **대기** → 데드락 감지 | **즉시 에러**(`40001`) |
| SERIALIZABLE | 사실상 모든 읽기를 잠금 읽기로 | **SSI** — 낙관적 검증 |
| 재시도 로직 | **필수** — 데드락·락 타임아웃 | **필수** — `SQLSTATE 40001` |
| 행 락 모드 | 공유/배타 | **4단계** (FK 경합 완화) |
| 큐 패턴 | `FOR UPDATE SKIP LOCKED` (8.0+) | `FOR UPDATE SKIP LOCKED` |
| 임의 이름 락 | **`GET_LOCK()`**(세션 수준, MDL 체계 안) | **advisory lock** |

**세 번째 줄이 이식할 때 가장 크게 물린다.** MySQL에서 잘 돌던 코드를 PostgreSQL에 올리고 격리 수준을 RR로 맞추면, **데드락 대신 `40001`이 쏟아진다.** 같은 문제를 다르게 신고하는 것인데, 재시도 장치가 없으면 그냥 실패로 보인다.

> **재시도가 갈림길이 아니다.** 초판에서 이 표는 재시도를 "InnoDB는 권장 / PostgreSQL은 필수"로, 임의 락을 "InnoDB에는 없음"으로 적었는데 **둘 다 틀렸다.** MySQL 문서도 재시도를 요구하고 `GET_LOCK()`이라는 사용자 수준 락이 존재한다. **두 엔진이 실제로 갈리는 건 재시도의 필요 여부가 아니라 실패가 도착하는 이름**이다 — 한쪽은 데드락·락 타임아웃으로, 다른 쪽은 직렬화 실패로 온다. → [MySQL 락과 격리](../database/mysql/lock-and-isolation.md) §2-9

---

**다음으로 읽을 것**

- 직렬 가능성과 2PL의 원리 → [동시성 제어 이론](../database/basics/concurrency-theory.md)
- 갭 락으로 푸는 쪽 → [트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-5
- 플래그 컬럼이 왜 블로트를 만드나 → [MVCC와 VACUUM](mvcc-and-vacuum.md) §2-2
- `jsonb`에 인덱스를 거는 방법 → [힙과 인덱스](heap-and-index.md) §2-5

---

> **기준 버전**: PostgreSQL 17
> **확인한 출처**:
> - [PostgreSQL 13.2 Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html) — *"Read Committed is the default isolation level"*, RR이 팬텀을 허용하지 않는다는 표(*"Allowed, but not in PG"*)와 그것이 표준상 허용되는 이유, SERIALIZABLE의 *"emulates serial transaction execution"*, **`SQLSTATE 40001`을 일반적으로 처리하라는 지시**, `could not serialize access due to concurrent update` / `due to read/write dependencies` 두 에러 문구, *"retry the whole transaction from the beginning"*, 커밋 전 읽은 값을 신뢰하지 말라는 서술
> - [PostgreSQL 13.3 Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html) — 행 락 4모드의 정의와 충돌 매트릭스, advisory lock의 정의(*"the system does not enforce their use"*), **세션 수준 락이 롤백 후에도 유지된다는 서술**, 플래그 컬럼 대비 이점(*"faster, avoid table bloat, automatically cleaned up"*)
> **미확인**: §2-5 `SKIP LOCKED`·`NOWAIT`의 동작은 참조한 Explicit Locking 장에 없었다(SELECT 레퍼런스에 있을 것으로 보이나 대조하지 않았다) · §2-4에서 FK가 `FOR KEY SHARE`를 사용한다는 서술 · §2-7의 `jsonb` 내부 표현, EXCLUDE 제약 문법, `TIMESTAMPTZ`의 UTC 저장 방식 — 전부 대조 필요
> **미작성**: 테이블 수준 락 8종 · 데드락 감지 동작 · `pg_locks` 조회 · 예측 가능한 락 순서 설계
