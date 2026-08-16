# 락과 격리 수준 — 기본값이 REPEATABLE READ라서 생기는 일들

## 30초 요약

- InnoDB의 기본은 **REPEATABLE READ**다. 이 한 줄에서 **넥스트키 락**이 나오고, 넥스트키 락에서 **"없는 행까지 잠근다"**가 나온다
- **갭 락은 "억제 전용"(purely inhibitive)이다.** 갭 락끼리는 서로 안 막는다 — **공유/배타 구분조차 없다.** 데드락은 갭 락 혼자서는 안 만든다
- **유니크 인덱스로 유니크한 행 하나를 찾으면 갭 락을 안 건다.** "인덱스 설계가 곧 락 설계"의 가장 구체적인 형태
- **AUTO-INC 기본값은 8.4에서도 `2`(interleaved)** — 채번이 연속이라는 가정은 깨져 있다. 그리고 그 기본값은 **복제 방식이 정했다**
- 진짜 장애를 만드는 건 행 락이 아니라 **메타데이터 락(MDL)**이다. **기본 대기 한도가 1년**이라 알아서 안 끊긴다

---

## 원리 — 왜 그런가

> [트랜잭션 · 락](../basics/transaction-and-lock.md) §2-5가 갭 락을 소개했고, [동시성 제어 이론](../basics/concurrency-theory.md) §2-5가 다중 입도 락과 의도 락의 **이론**을 다뤘다. 이 문서는 그 **InnoDB 구현**과, 앞선 두 편에 없는 **MDL·AUTO-INC**다.

### 2-1. 기본이 REPEATABLE READ라는 것

공식 문서의 한 줄은 짧다 — *"The default isolation level for InnoDB is REPEATABLE READ."* **왜 그런가는 안 적혀 있다.** 다만 그 옆에 **기계적 사실 하나**가 붙어 있다.

> *"Only row-based binary logging is supported with the READ COMMITTED isolation level. If you use READ COMMITTED with `binlog_format=MIXED`, the server automatically uses row-based logging."*

**격리 수준과 복제 방식이 묶여 있다**는 뜻이다. RC에서는 갭 락이 없으니 같은 SQL이 소스와 리플리카에서 다른 행 집합을 건드릴 수 있고, 그래서 **문장을 그대로 복제하는 방식이 성립하지 않는다.** "MySQL의 기본이 RR인 건 옛날 statement-based 복제 때문"이라는 통설([트랜잭션 · 락](../basics/transaction-and-lock.md) §2-2)은 **이 제약과 앞뒤가 맞지만, 공식 문서가 그 인과를 직접 말하지는 않는다.**

**RR에서 팬텀이 어떻게 처리되는지는 읽기 종류에 따라 갈린다.**

| | 무엇을 보나 | 팬텀 |
|---|---|---|
| 일반 `SELECT`(consistent read) | **첫 읽기가 뜬 스냅샷** | 애초에 안 보인다 |
| 잠금 읽기·`UPDATE`·`DELETE`(current read) | 최신 | **넥스트키 락으로 막는다** (§2-2) |

> **PostgreSQL은 두 칸이 다 스냅샷이다**([락과 타입](../../postgresql/lock-and-types.md) §2-2). **InnoDB만 아래 칸을 락으로 푼다.** 이 문서 대부분이 그 선택의 파생물이다.

**RC로 낮추면 무엇이 따라오나** — 갭 락이 사라져 데드락이 줄지만, 공식 문서 표현대로 *"phantom row problems may occur"*이고 **binlog가 ROW로 강제된다.** 격리 수준 하나 내리는 결정이 **복제 설정까지 건드린다.**

### 2-2. 넥스트키 락 — 구간으로 보면 이해가 된다

세 가지를 구분해야 한다. 공식 정의가 짧고 정확하다.

- **레코드 락**: *"a lock on an index record"* — **테이블 행이 아니라 인덱스 레코드**다
- **갭 락**: *"a lock on a gap between index records, or a lock on the gap before the first or after the last index record"*
- **넥스트키 락**: *"a combination of a record lock on the index record and a gap lock on the gap before the index record"*

**구간 표기로 보면 한 번에 들어온다.** 인덱스에 10·11·13·20이 있으면 넥스트키 락이 나눠 갖는 구간이 이렇게 된다.

```
(-∞, 10]   (10, 11]   (11, 13]   (13, 20]   (20, +∞)
```

**오른쪽이 닫혀 있다**는 게 핵심이다. 각 구간은 "앞의 빈 자리 + 그 레코드 자신"이다. 그리고 **마지막 구간에는 실재하지 않는 레코드가 하나 있다** — 문서 표현으로 *"the 'supremum' pseudo-record having a value higher than any value actually in the index"*. `supremum`은 **"이 인덱스의 끝"이라는 표식**이고, 데드락 덤프에 이 이름이 찍히면 **"인덱스 맨 끝 뒤쪽 전체를 잠갔다"**는 뜻이다.

**발송 도메인에서 이게 나타나는 모양.**

```sql
-- 예약 발송 워커: 지금 보낼 것만 집어 온다
SELECT * FROM send_reserve
 WHERE reserve_at <= NOW() AND status = 'READY' FOR UPDATE;
```

`reserve_at` 인덱스를 타면 **스캔한 구간 전체에 넥스트키 락**이 걸린다. 그 구간에는 **아직 존재하지 않는 예약 시각**도 포함되므로, **같은 시간대로 새 예약을 등록하려는 요청이 막힌다.** 조회 트랜잭션이 길면 등록 API가 통째로 밀린다.

### 2-3. 갭 락은 "억제 전용"이다

여기가 가장 많이 오해되는 지점이고, 공식 문서가 못을 박는다.

> *"Gap locks in InnoDB are 'purely inhibitive', which means that their only purpose is to prevent other transactions from inserting to the gap. Gap locks can co-exist. A gap lock taken by one transaction does not prevent another transaction from taking a gap lock on the same gap. There is no difference between shared and exclusive gap locks."*

**세 가지가 한꺼번에 나온다.** ① 목적은 **INSERT 차단 하나뿐** — 읽기도 수정도 안 막는다. ② **같은 갭에 여러 트랜잭션이 동시에 갭 락을 갖는다.** ③ **공유/배타 구분이 없다** — S 갭 락과 X 갭 락이 서로 안 막는다.

> **그래서 "갭 락 때문에 데드락이 난다"는 절반만 맞다.** 갭 락 혼자서는 아무도 안 막으므로 순환을 못 만든다. 데드락을 만드는 건 **넥스트키 락에 같이 들어 있는 레코드 락**과, 아래의 **insert intention lock**이다.

**INSERT 쪽에도 대칭 장치가 있다** — *"An insert intention lock is a type of gap lock set by INSERT operations prior to row insertion... multiple transactions inserting into the same index gap need not wait for each other if they are not inserting at the same position within the gap."*

**같은 갭에 서로 다른 값을 넣는 INSERT들은 안 기다린다.** 여기까지는 전부 "최대한 안 막게" 설계돼 있다.

**충돌은 두 종류가 만난 순간 생긴다** — 누군가 그 갭에 **갭 락(넥스트키 락의 일부)** 을 쥐고 있는데 다른 쪽이 **insert intention lock**을 요청하면 그때 기다린다. §2-2의 예약 발송 사고가 정확히 이 조합이다.

### 2-4. 유니크 인덱스 등치 검색에는 갭 락을 안 건다

**락 범위를 줄이는 가장 강력한 수단이 인덱스 선택**이라는 [트랜잭션 · 락](../basics/transaction-and-lock.md) §2-6의 결론이, 여기서 문서의 문장으로 확인된다.

> *"Gap locking is not needed for statements that lock rows using a unique index to search for a unique row. ... If `id` is not indexed or has a nonunique index, the statement does lock the preceding gap."*

**조건이 셋 다 맞아야 한다** — 유니크 인덱스이고, 등치이고, **행 하나를 특정**해야 한다. 문서가 예외도 적어 뒀다 — *"(This does not include the case that the search condition includes only some columns of a multiple-column unique index; in that case, gap locking does occur.)"* 즉 `UNIQUE (campaign_id, receiver_phone)`에 `WHERE campaign_id = 7`만 주면 **유니크 인덱스를 탔는데도 갭 락이 붙는다.**

**그리고 더 넓은 함정이 하나 있다.**

> *"A locking read, an UPDATE, or a DELETE generally set record locks on every index record that is scanned in the processing of an SQL statement. It does not matter whether there are WHERE conditions in the statement that would exclude the row."*

**"조건에 맞는 행"이 아니라 "스캔하다 지나간 레코드"에 락이 걸린다.** 인덱스로 걸러지지 않고 서버 층에서 걸러지는 조건은 **락 범위를 못 줄인다.** [트랜잭션 · 락](../basics/transaction-and-lock.md) §2-6이 "인덱스가 없으면 사실상 테이블 락"이라 한 것의 정확한 근거가 이 문장이다.

```sql
UPDATE send_history SET status='FAILED' WHERE campaign_id=7 AND error_code='TIMEOUT';
-- campaign_id 에만 인덱스가 있으면 → 캠페인 7의 발송건 전부(수십만 건)에 넥스트키 락
--                                  → error_code 로 걸러지는 건 락을 잡은 다음이다
```

**해법은 락을 줄이는 옵션을 찾는 게 아니라 `(campaign_id, error_code)` 인덱스를 만드는 것**이다. 인덱스 튜닝과 락 튜닝이 같은 작업인 이유가 이것이다.

### 2-5. 의도 락 — 이론의 InnoDB 구현

[동시성 제어 이론](../basics/concurrency-theory.md) §2-5에서 **왜 의도 락이 필요한지**를 봤다. InnoDB의 구현은 단출하다 — **테이블 수준에 IS와 IX 두 종뿐**이고, *"SELECT ... FOR SHARE sets an IS lock, and SELECT ... FOR UPDATE sets an IX lock."*

| | X | IX | S | IS |
|---|---|---|---|---|
| **X** | 충돌 | 충돌 | 충돌 | 충돌 |
| **IX** | 충돌 | **호환** | 충돌 | **호환** |
| **S** | 충돌 | 충돌 | 호환 | 호환 |
| **IS** | 충돌 | **호환** | 호환 | 호환 |

**표에서 읽어야 할 것은 IX-IX가 호환이라는 칸 하나다.** 서로 다른 행을 잠그는 트랜잭션들은 테이블 수준에서 **전혀 안 부딪힌다.** 의도 락은 실무에서 **거의 보이지 않는 게 정상**이고, 걸리는 순간은 누군가 **테이블 전체 락**(`LOCK TABLES ... WRITE` 등)을 요청했을 때다. 그리고 ⚠️ **"ALTER가 막혔다"를 의도 락으로 설명하면 틀린다** — DDL을 막는 건 이 테이블 락 계열이 아니라 **완전히 다른 층에 있는 메타데이터 락**이다(§2-7).

### 2-6. AUTO-INC 락 — 채번은 연속이 아니다

`AUTO_INCREMENT` 값을 나눠 주려면 **누군가는 줄을 세워야 한다.** 그 방식이 세 가지고, **기본값이 바뀐 이유가 명시돼 있는 드문 사례**다.

| 모드 | 이름 | AUTO-INC 테이블 락 | 값의 연속성 |
|---|---|---|---|
| 0 | traditional | 항상 | 보장 |
| 1 | consecutive | bulk insert에만 | 보장 |
| **2** (**8.4 기본**) | **interleaved** | **안 쓴다** | **보장 안 함** |

> *"The default setting of interleaved lock mode in MySQL 8.4 reflects the change from statement-based replication to row based replication as the default replication type. Statement-based replication requires the consecutive auto-increment lock mode... whereas row-based replication is not sensitive to the execution order of SQL statements."*

**복제 방식이 락 기본값을 정했다.** §2-1에서 격리 수준과 복제가 묶여 있는 걸 봤는데, **여기서는 그 인과가 문서에 그대로 적혀 있다.** 락의 수명도 특이하다 — *"This lock is normally held to the end of the statement (not to the end of the transaction)"*. **트랜잭션이 아니라 문장 단위**라, 롤백해도 이미 소비한 번호는 안 돌아온다.

**실무에서 물리는 곳 셋.**

- **ID 순서로 시간 순서를 추론하면 안 된다.** interleaved 모드에서 동시 INSERT의 번호는 뒤섞인다. 발송 이력을 시간순으로 보려면 `sent_at` 인덱스를 써야 한다
- **ID 구간으로 집계하면 틀린다.** "이 배치가 넣은 건 id 1000~2000"은 성립하지 않는다. 배치 식별자 컬럼을 따로 둬야 한다
- **statement-based 복제를 쓴다면** 문서가 `0` 또는 `1`로 내리고 **소스와 리플리카에 같은 값**을 쓰라고 명시한다. 다르면 리플리카의 채번이 어긋난다

> **`INSERT ... ON DUPLICATE KEY UPDATE`와 mixed-mode insert는 값을 건너뛴다.** 번호에 구멍이 나는 걸 버그로 보면 안 된다 — **연속성은 애초에 보장 대상이 아니다.**

### 2-7. 메타데이터 락 — 아무도 안 잠갔는데 ALTER가 막힌다

여기가 실제 장애를 만드는 자리이고, **행 락과 완전히 다른 층**에 있다.

> *"To ensure transaction serializability, the server must not permit one session to perform a data definition language (DDL) statement on a table that is used in an uncompleted explicitly or implicitly started transaction in another session."*

**적용 범위가 테이블만이 아니다** — 스키마, 스토어드 프로그램, 테이블스페이스, 그리고 **`GET_LOCK()`으로 잡은 사용자 락**까지 같은 MDL 체계 안에 있다.

**가장 중요한 한 줄이 수명이다** — *"The server holds metadata locks on tables used within a transaction and deferring release of those locks until the transaction ends."* **`SELECT` 하나만 해도 그 테이블의 MDL을 트랜잭션 끝까지 쥔다.** 커밋을 안 하면 안 놓는다.

**그래서 사고가 이렇게 번진다.**

```
① 세션 A: BEGIN; SELECT ... FROM send_history;   -- 그리고 커밋을 안 한다
                                                    (또는 외부 API를 기다린다)
② 세션 B: ALTER TABLE send_history ADD COLUMN ...;  -- A의 MDL 뒤에서 대기
③ 세션 C·D·E…: SELECT ... FROM send_history;        -- B 뒤에서 줄줄이 대기
```

**③이 진짜 장애다.** ②만 막히면 DDL 하나 실패로 끝나는데, **대기 중인 DDL이 뒤따르는 평범한 조회까지 전부 막는다.** 조회 하나가 커넥션 풀을 소진시키고 애플리케이션 전체가 멈춘다. `SHOW PROCESSLIST`에 **`Waiting for table metadata lock`**이 줄줄이 찍히면 이 상황이고, 범인은 그 목록 맨 아래 어딘가에서 조용히 열려 있는 트랜잭션이다.

**⚠️ 그리고 알아서 안 끊긴다.**

| 변수 | 기본값 | 무엇의 한도 |
|---|---|---|
| `lock_wait_timeout` | **31536000초 (= 365일)** | **MDL** 획득 대기 |
| `innodb_lock_wait_timeout` | (아래 `미확인` 참고) | **InnoDB 행 락** 대기 |

**둘이 다른 변수라는 걸 놓치면 진단이 어긋난다.** 행 락 타임아웃을 아무리 짧게 잡아도 **MDL 대기는 그 설정과 무관**하고, MDL 쪽 기본값은 **사실상 무한**이다. 게다가 타임아웃은 **락 하나마다 따로** 적용되므로 문장 하나가 그 값보다 오래 막힐 수도 있다. (문서에 *"Statements acquire metadata locks one by one... and perform deadlock detection in the process"*도 있다 — **MDL에도 데드락 감지가 있다.**)

> **대책은 셋이고 순서가 있다.** ① **DDL 전에 열린 트랜잭션이 없는지 확인한다**(`performance_schema.metadata_locks`). ② **DDL 세션에서만 `lock_wait_timeout`을 짧게**(수 초) 잡는다 — 못 잡으면 빨리 실패하는 게 낫다. ③ 근본은 [트랜잭션 · 락](../basics/transaction-and-lock.md) §2-7의 그 조언이다 — **트랜잭션 안에서 외부 API를 기다리지 마라.** 그 습관이 행 락 사고와 MDL 사고를 동시에 만든다.
>
> **온라인 DDL이 이 MDL을 어느 구간에서 어떻게 요구하는지**는 [복제와 운영](replication-and-ops.md) §2-5가 정본이다.

### 2-8. 중복 키 에러가 만드는 데드락

발송의 멱등키(`UNIQUE (dedup_key)`)에 동시 INSERT가 들어오는 상황은 흔한데, **여기에 데드락 함정이 문서에 명시돼 있다.**

> *"If a duplicate-key error occurs, a shared lock on the duplicate index record is set. This use of a shared lock can result in deadlock should there be multiple sessions trying to insert the same row if another session already has an exclusive lock."*

**실패한 INSERT가 락을 놓는 게 아니라 공유 락을 잡는다**는 게 반직관적이다.

```
세션 1: INSERT dedup_key='A'  → 배타 락 획득
세션 2: INSERT dedup_key='A'  → 중복 키 에러. 그 레코드에 공유 락 요청 (대기)
세션 3: INSERT dedup_key='A'  → 같은 것. 공유 락 요청 (대기)
세션 1: ROLLBACK              → 2와 3이 각각 공유 락을 얻고, 둘 다 배타 락을 요청한다
                              → 서로가 쥔 공유 락 때문에 못 올라간다 → 데드락
```

**멱등 처리를 "일단 INSERT하고 중복이면 무시"로 짜면 이 경로를 매번 밟는다.** 재시도 폭풍이 겹치는 순간(발송 결과 콜백이 몰릴 때) 데드락이 무더기로 난다.

> **[대용량 처리](../../system-design/high-throughput.md) §2-6의 멱등성이 여기서 구현 방식까지 요구한다.** 유니크 제약은 **최종 방어선으로 남기고**([데이터 모델링](../basics/data-modeling.md) §2-5), 정상 경로는 **동시 요청이 같은 키로 몰리지 않게** 설계하는 쪽이 낫다. 그리고 어느 쪽이든 **재시도 로직은 필수다** — 공식 문서도 *"even if your application logic is correct, you must still handle the case where a transaction must be retried"*라고 한다.
>
> 외래 키도 같은 계열이다 — 제약 검사가 *"sets shared record-level locks on the records that it looks at"*이고, **검사에 실패하는 경우에도** 락을 잡는다.

### 2-9. 그래서 PostgreSQL과 무엇이 다른가

| 항목 | InnoDB | PostgreSQL |
|---|---|---|
| 기본 격리 수준 | **REPEATABLE READ** | READ COMMITTED |
| 팬텀 차단 방식 | 잠금 읽기는 **넥스트키 락**, 일반 읽기는 스냅샷 | **전부 스냅샷** |
| 갭 락 | 있다. **억제 전용·서로 공존·S/X 구분 없음** | 없다 |
| 격리 수준 ↔ 복제 | **묶여 있다** (RC는 ROW binlog 강제) | 무관 |
| 의도 락 | **IS·IX 두 종**(테이블 수준) | 테이블 락 8종 체계 안에 흡수 |
| DDL이 막히는 원인 | **MDL** — 트랜잭션 끝까지, 대기 기본 **365일** | `ACCESS EXCLUSIVE` 락([MVCC와 VACUUM](../../postgresql/mvcc-and-vacuum.md) §2-4) |
| 채번 | **AUTO-INC** (기본 interleaved, 연속 보장 없음) | 시퀀스 객체([복제와 운영](../../postgresql/replication-and-ops.md) §2-2) |
| 임의 이름 락 | **`GET_LOCK()`** (세션 수준, MDL 체계 안) | **advisory lock**([락과 타입](../../postgresql/lock-and-types.md) §2-6) |
| 재시도 로직 | **필수** — 데드락은 없앨 수 없다 | **필수** — `SQLSTATE 40001` |

**마지막 두 줄이 [락과 타입](../../postgresql/lock-and-types.md) §2-8의 표와 다르게 읽힐 수 있다.** 그쪽 표는 임의 락을 "InnoDB에는 없음(테이블로 흉내)"으로, 재시도를 "InnoDB는 권장"으로 적었는데, **MySQL 문서에는 `GET_LOCK()`이 있고 재시도도 `must`로 쓰여 있다.** 두 엔진이 갈리는 건 **재시도의 필요 여부가 아니라 실패의 이름**이다 — 한쪽은 데드락·락 타임아웃으로, 다른 쪽은 직렬화 실패로 온다.

---

**다음으로 읽을 것**

- 갭 락의 기본 개념과 데드락 3원칙 → [트랜잭션 · 락](../basics/transaction-and-lock.md) §2-5, §2-7
- 의도 락이 왜 필요한가(이론) → [동시성 제어 이론](../basics/concurrency-theory.md) §2-5
- 갭 락 없이 팬텀을 막는 쪽 → [락과 타입](../../postgresql/lock-and-types.md) §2-2
- 이 락들이 걸리는 물리 구조 → [InnoDB 내부 구조](innodb-internals.md) §2-6

---

> **기준 버전**: MySQL 8.4 · PostgreSQL 17
> **확인한 출처**:
> - [MySQL 17.7.1 InnoDB Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html) — S·X·IS·IX 정의와 **호환 매트릭스**, `FOR SHARE`→IS / `FOR UPDATE`→IX, 레코드·갭·넥스트키 락 정의 원문, **10·11·13·20 구간 예시와 supremum 의사 레코드**, **유니크 인덱스 등치 검색에 갭 락이 불필요하다는 원문과 복합 유니크 부분 사용 예외**, *"purely inhibitive"* 문단 전체, insert intention lock 정의, RC에서 갭 락이 FK·중복 키 검사에만 남는다는 서술
> - [MySQL 17.7.2.1 Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html) — **"The default isolation level for InnoDB is REPEATABLE READ"**, RR의 스냅샷·잠금 읽기 구분, RC에서 *"phantom row problems may occur"*, **RC는 row-based binary logging만 지원**, SERIALIZABLE이 `autocommit` 비활성 시 `SELECT`를 `FOR SHARE`로 바꾼다는 서술
> - [MySQL 17.7.3 Locks Set by Different SQL Statements](https://dev.mysql.com/doc/refman/8.4/en/innodb-locks-set.html) — *"every index record that is scanned... It does not matter whether there are WHERE conditions"*, INSERT가 갭 락이 아닌 인덱스 레코드 락을 건다는 점, **중복 키 에러 시 공유 락과 3세션 데드락 시나리오**, FK 검사가 공유 레코드 락을 걸며 **실패 시에도** 건다는 서술
> - [MySQL 10.11.4 Metadata Locking](https://dev.mysql.com/doc/refman/8.4/en/metadata-locking.html) — MDL의 목적 원문, 적용 대상(스키마·스토어드 프로그램·테이블스페이스·`GET_LOCK()`), **트랜잭션 끝까지 유지된다는 원문**과 DDL 차단 예시, *"acquire metadata locks one by one... and perform deadlock detection"*
> - [MySQL 17.6.1.6 AUTO_INCREMENT Handling](https://dev.mysql.com/doc/refman/8.4/en/innodb-auto-increment-handling.html) — 세 모드, **8.4 기본이 2(interleaved)이고 그 이유가 행 기반 복제 전환이라는 원문**, AUTO-INC 락이 **문장 끝까지**(트랜잭션 아님) 유지된다는 서술, simple/bulk/mixed-mode 분류, SBR에서는 0·1을 쓰고 소스·리플리카를 맞추라는 지시
> - [MySQL 17.7.5 Deadlocks in InnoDB](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlocks.html) — *"you must still handle the case where a transaction must be retried"* / `lock_wait_timeout` 기본 **31536000초(1년)** 와 적용 범위는 매뉴얼의 `server-system-variables` 항목으로 확인
> **미확인**:
> - **§2-1의 "기본이 RR인 이유"** — 공식 문서에 인과가 없다. 본문은 *RC가 ROW binlog를 강제한다*는 확인된 사실만 제시하고 통설은 통설로 표시했다
> - `innodb_lock_wait_timeout` 기본값 — Deadlocks·InnoDB 파라미터 페이지에서 수치를 확인하지 못해 §2-7 표에서 뺐다([트랜잭션 · 락](../basics/transaction-and-lock.md) §2-7이 50초로 적었으나 그 문서도 출처 대조 전이다). `lock_wait_timeout` 쪽도 **8.4 매뉴얼 본문 표를 직접 열어 대조하지는 못했다**(해당 페이지가 잘려 L 항목에 도달하지 못함)
> - §2-7의 `performance_schema.metadata_locks` 사용법 · MDL의 락 종류(SHARED_READ 등) 체계 — 대조하지 않았다
> **미작성**: `SHOW ENGINE INNODB STATUS` 데드락 덤프 읽는 법 · `performance_schema.data_locks` 조회 · 온라인 DDL의 MDL 승격 구간 · `FOR UPDATE NOWAIT`·`SKIP LOCKED`([트랜잭션 · 락](../basics/transaction-and-lock.md) §2-8이 정본) · 외래 키 락의 상세 동작
