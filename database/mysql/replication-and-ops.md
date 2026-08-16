# 복제와 운영 — binlog 하나가 복제·운영·스키마 변경을 다 정한다

## 30초 요약

- **binlog 기본 포맷은 ROW다.** 문장을 보내면 `NOW()`·`RAND()`에서 원본과 사본이 갈리기 때문이다 — "복제는 재생"이라는 전제가 깨지는 자리
- **GTID는 "몇 번 파일 몇 바이트" 대신 트랜잭션에 이름을 붙인다.** 그 결과 **같은 트랜잭션이 두 번 적용되지 않고**, 페일오버 때 위치를 손으로 안 찾아도 된다
- **반동기 복제가 기다리는 것은 "받았다"까지지 "적용했다"까지가 아니다.** 그래서 **리플리카에서 최신을 읽는 보장은 여전히 없다**
- **Online DDL의 `INSTANT`는 메타데이터만 고친다.** 되는 연산 목록이 좁고, **타입 변경·PK 삭제는 지금도 `COPY`**다 — 테이블 전체 복제
- **`TIMESTAMP`는 세션 타임존 기준으로 UTC 변환되고 `DATETIME`은 안 된다.** 예약 발송에서 이 차이가 그대로 사고가 된다

---

## 원리 — 왜 그런가

> 복제의 원리(로그 재생·동기/비동기·복제 지연·read-your-writes·페일오버)와 파티셔닝의 프루닝 원리는 [복제와 파티셔닝](../basics/replication-and-partitioning.md)에 있다. redo와 binlog의 2단계 커밋은 [내구성과 복구](../basics/durability-and-recovery.md) §2-8이다. 이 문서는 **MySQL이 그걸 어떻게 구현했고 무엇이 운영 업무가 되는지**다.

### 2-1. binlog 포맷 — ROW가 기본값이 된 이유

MySQL의 복제 로그는 redo와 **별개 파일인 binlog**다([내구성과 복구](../basics/durability-and-recovery.md) §2-8). 그리고 **무엇을 적을지 고를 수 있다.**

| 포맷 | 적는 것 | 문제 |
|---|---|---|
| `STATEMENT` | **SQL 문장 그대로** | 비결정적 문장에서 원본과 사본이 갈린다 |
| **`ROW`** (기본) | **바뀐 행의 값** | 로그가 크다. 1건 UPDATE가 100만 행이면 100만 행이 기록된다 |
| `MIXED` | 기본은 문장, 위험할 때만 행 | 판정을 서버에 맡긴다 |

공식 문서의 표현은 *"In row-based logging (the default), the source writes events to the binary log that indicate how individual table rows are affected."*다. **왜 문장이 위험한가** — [복제와 파티셔닝](../basics/replication-and-partitioning.md) §2-2의 "복제는 재생"이라는 전제 때문이다. 재생이 원본과 같은 결과를 내야 하는데, `NOW()` · `RAND()` · `UUID()` · 순서 의존 `LIMIT` 갱신은 **두 번 실행하면 다른 결과**가 나온다. MySQL 자신도 이런 문장에 *"Statement may not be safe to log in statement format"* 경고를 낸다.

```sql
UPDATE send_reserve SET status='SENDING', picked_at=NOW() WHERE status='READY' LIMIT 1000;
-- STATEMENT: 리플리카가 다시 실행 → picked_at 값이 다르고, LIMIT 1000이 고르는 행도 다를 수 있다
-- ROW      : "이 1000행이 이 값으로 바뀌었다"를 그대로 적용 → 동일
```

> **ROW의 대가는 로그 크기다.** 대량 발송 배치가 1천만 행을 한 번에 갱신하면 binlog가 그만큼 부푼다. 그리고 그게 그대로 **네트워크 전송량**과 **리플리카 재생 시간**이 된다 — [복제와 파티셔닝](../basics/replication-and-partitioning.md) §2-4의 "배치 돌면 지연이 벌어진다"의 물리적 실체가 여기다. **대량 갱신을 청크로 쪼개는 이유가 락뿐만이 아니다.**

### 2-2. GTID — 위치 대신 이름을 붙인다

원래 리플리카는 **"어느 binlog 파일의 몇 바이트까지 읽었다"**로 자기 위치를 기억했다. 이 좌표는 **서버마다 다르다.** 그래서 프라이머리가 죽어 리플리카 B를 승격시키면, 남은 리플리카 C는 **B의 좌표계로 자기 위치를 다시 계산해야** 한다. 사람이 손으로 하는 일이고, 틀리면 트랜잭션이 빠지거나 두 번 적용된다.

**GTID는 좌표 대신 트랜잭션마다 전역 고유 이름을 붙인다.**

```
3E11FA47-71CA-11E1-9E33-C80AA9429562:23      (8.4는 source_id:tag:transaction_id 형태도 지원)
└──────────── source_id (서버 UUID) ────────┘ └ transaction_id (커밋 순번)
```

*"This identifier is unique not only to the server on which it originated, but is unique across all servers in a given replication topology."* **이름이 전역이라서 따라오는 성질이 둘이고, 둘 다 운영에서 값어치가 크다.**

- **자동 스킵** — *"a transaction committed on the source can be applied no more than once on the replica."* 이미 적용한 GTID가 또 오면 무시한다. **재적용으로 인한 중복 발송을 구조적으로 막는다**
- **틈 없이 단조 증가** — *"Client transactions are guaranteed to have monotonically increasing GTIDs without gaps."* "어디까지 받았나"가 집합 연산으로 표현되고, 그래서 **자동 위치 지정(auto-positioning)**이 성립한다

켜는 데 조건이 붙는다 — `gtid_mode`(`OFF` → `ON_PERMISSIVE` → `ON`)와 **`enforce_gtid_consistency`**다. 후자는 GTID로 표현할 수 없는 문장(트랜잭션 안의 `CREATE TABLE ... SELECT` 등)을 **금지**한다. **애플리케이션이 그런 문장을 쓰고 있으면 GTID를 켜는 순간 에러가 난다** — 도입 전에 먼저 확인할 항목이다.

### 2-3. 반동기 복제 — "받았다"와 "적용했다" 사이

[복제와 파티셔닝](../basics/replication-and-partitioning.md) §2-3에서 동기/비동기의 트레이드오프를 봤다. MySQL의 절충안이 반동기인데, **정확히 어디까지 기다리는지가 오해를 부른다.**

> *"The replica acknowledges receipt of a transaction's events only after the events have been written to its relay log and flushed to disk."*

```
프라이머리 커밋 → binlog를 디스크에 sync
  → [기다림] 리플리카가 릴레이 로그에 쓰고 flush → ACK   ← 여기까지가 반동기
  → 스토리지 엔진에 커밋 → 클라이언트에 성공 반환
리플리카가 그 이벤트를 실제로 *적용*하는 건 그 다음이다. 프라이머리는 안 기다린다.
```

**그래서 반동기를 켜도 "리플리카에서 방금 쓴 걸 읽을 수 있다"는 보장은 안 생긴다.** 보장되는 건 **"프라이머리가 갑자기 죽어도 그 트랜잭션은 어딘가에 남아 있다"**뿐이다. PostgreSQL의 `remote_apply`가 재생까지 기다리는 것과 다르다.

**나머지 셋도 같이 알아야 한다.**

- **기본 ACK 개수는 1**이다. 리플리카가 10대여도 **한 대만 받으면 진행**한다. 그리고 **대기 지점이 둘**인데, 기본 **`AFTER_SYNC`**(binlog sync 후, 스토리지 엔진 커밋 **전**)가 기본인 이유는 **커밋 전에 기다려야 "프라이머리에서는 보이는데 어디에도 안 남은 트랜잭션"이 안 생기기** 때문이다(대안은 `AFTER_COMMIT`)
- **타임아웃이 나면 비동기로 강등된다** — *"If a timeout occurs without any replica having acknowledged the transaction, the source reverts to asynchronous replication."* **커밋이 멈추는 대신 보장이 조용히 사라진다.** PostgreSQL의 동기 복제가 스탠바이 사망 시 커밋을 못 끝내는 것과 **정반대의 선택**이다(A를 택하고 C를 버린다)

> ⚠️ **"반동기니까 데이터 손실 없음"은 틀린 요약이다.** 타임아웃 강등이 있는 한 **보장은 상시가 아니라 조건부**다. 강등 여부를 모니터링하지 않으면 **비동기로 돌아간 줄 모르고 운영하게 된다.**

### 2-4. 파티셔닝 — 네 종류와 하나의 제약

[복제와 파티셔닝](../basics/replication-and-partitioning.md) §2-7의 프루닝 원리를 MySQL은 네 가지로 제공한다.

| 방식 | 나누는 기준 | 발송 도메인에서 |
|---|---|---|
| **RANGE** | 값의 구간 | **`sent_at` 월별.** 보존 정책의 정석(→ [DB 인덱스](../basics/b-tree-index.md) §2-12) |
| **LIST** | 값의 열거 | 채널별(`SMS`/`LMS`/`MMS`/`ALIMTALK`) — 종류가 고정일 때 |
| **HASH** | **사용자가 쓴 정수 표현식** | 고객사 ID 분산. 표현식이 **정수를 반환해야** 한다 |
| **KEY** | **서버 내장 해시 함수** | 정수가 아닌 컬럼도 가능 |

**HASH와 KEY의 차이가 실무에서 갈린다.** 공식 문서가 KEY를 이렇게 설명한다 — *"These columns can contain other than integer values, since the hashing function supplied by MySQL guarantees an integer result regardless of the column data type."* 즉 `PARTITION BY HASH(YEAR(joined))`처럼 **직접 정수로 바꿔야 하는 것**과, `PARTITION BY KEY(joined)`처럼 **컬럼을 그냥 주는 것**의 차이다. (RANGE·LIST에는 여러 컬럼을 쓸 수 있는 `COLUMNS` 변형이, HASH·KEY에는 재분할 비용을 줄이는 `LINEAR` 변형이 따로 있다.)

**그리고 제약 하나가 도입 자체를 막는다** — *"All columns used in the partitioning expression for a partitioned table must be part of every unique key that the table may have."*

```sql
CREATE TABLE send_history (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,          -- 유니크 키에 sent_at 이 없다
  sent_at DATETIME NOT NULL
) PARTITION BY RANGE (TO_DAYS(sent_at)) (...);   -- ERROR → PK를 (id, sent_at)으로 바꿔야 한다
```

**PK가 `(id, sent_at)`이 되면 §index-and-optimizer 2-1의 "PK는 짧아야 한다"와 정면으로 부딪친다** — 모든 세컨더리 인덱스가 그만큼 뚱뚱해진다([DB 인덱스](../basics/b-tree-index.md) §2-3). **파티셔닝은 설계 단계에서 결정해야 하는 항목**이고, 운영 중 도입은 §2-6의 도구가 필요한 대공사가 된다.

### 2-5. Online DDL — `INSTANT` / `INPLACE` / `COPY`

`ALTER TABLE`에 `ALGORITHM`을 명시할 수 있고, **세 단계의 차이는 "테이블을 다시 쓰느냐"다.**

| | 하는 일 | 동시 DML | 비용 |
|---|---|---|---|
| **`INSTANT`** | **메타데이터만 수정** | 영향 없음 | 거의 0 |
| **`INPLACE`** | 테이블 안에서 처리(재구축할 수도 있다) | 대체로 허용 | 데이터 양에 비례 |
| **`COPY`** | **새 테이블을 만들어 전부 복사** | **불가** | 가장 비쌈 |

- **`INSTANT`가 되는 것**(확인한 것만): 컬럼 추가(**8.0.12+**), 컬럼 삭제(**8.0.29+**), 컬럼 이름 변경, 기본값 설정·삭제, `ENUM`/`SET` 정의 변경, 인덱스 타입 변경
- **`INPLACE` + 동시 DML 허용**: 세컨더리 인덱스 **생성·삭제·이름 변경**, `SPATIAL` 인덱스 추가, `VARCHAR` 길이 확장, `NULL`/`NOT NULL` 전환
- **여전히 `COPY`인 것**: **컬럼 타입 변경**(`INT` → `BIGINT`), **PK 삭제**. PK 추가는 `INPLACE`지만 **테이블을 재구축**한다

**발송 이력 20억 건에서 `id`가 `INT` 상한(21억)에 다가왔다고 하자.** `MODIFY id BIGINT`는 `COPY`뿐이다 — 20억 행 복사에 인덱스 재생성, 그동안 쓰기 불가. **여기가 대량 발송 서비스에서 실제로 터지는 자리**이고, 그래서 §2-6의 외부 도구가 필요해진다.

**"온라인"이어도 메타데이터 락은 필요하다 — 두 번째 함정이다.**

> *"In the commit table definition phase, the metadata lock is upgraded to exclusive to evict the old table definition and commit the new one."* / *"An online DDL operation may have to wait for concurrent transactions that hold metadata locks on the table ... Additionally, a pending exclusive metadata lock requested by an online DDL operation blocks subsequent transactions on the table."*

```
장시간 SELECT 하나가 열려 있다
  → ALTER 가 배타 MDL 을 못 얻고 대기               ("Waiting for table metadata lock")
  → 그 뒤에 들어온 모든 쿼리가 ALTER 뒤에서 대기      ← 여기서 서비스가 멈춘다
```

**멈추는 원인이 `ALTER` 자체가 아니라 그 앞의 오래된 트랜잭션**이다. 대기 중인 배타 MDL이 **뒤따르는 정상 쿼리까지 줄 세우기** 때문에 겉으로는 "DDL 걸었더니 전면 장애"로 보인다. `SHOW FULL PROCESSLIST`로 확인하고, **`lock_wait_timeout`을 짧게 걸어 실패시키는 편**이 무한 대기보다 낫다.

> **나머지 운영 항목 둘.** ① `LOCK` 절(`NONE`/`SHARED`/`EXCLUSIVE`/`DEFAULT`)은 **지원하지 않는 수준을 요구하면 문장이 에러로 실패한다.** `ALGORITHM`도 같다 → **`ALGORITHM=INSTANT, LOCK=NONE`을 명시하면 "조용히 `COPY`로 떨어지는 사고"를 막는다.** ② `INPLACE` 중의 DML은 임시 로그 파일에 쌓이고, `innodb_online_alter_log_max_size`를 넘으면 **`DB_ONLINE_LOG_TOO_BIG`으로 실패·롤백**된다 — 쓰기가 많은 시간대의 큰 `ALTER`는 **몇 시간 돌다가 실패**할 수 있다. 재구축 연산은 **테이블+인덱스 크기만큼의 임시 공간**도 요구한다(`tmpdir` / `innodb_tmpdir`).

### 2-6. gh-ost · pt-osc — 서버 밖에서 스키마를 바꾸는 이유

§2-5의 `COPY`와 MDL 문제를 **애플리케이션 계층에서 우회**하는 도구가 둘이다. 발상은 같고 **동기화 방법이 갈린다.**

```
공통 : ① 새 스키마로 빈 고스트 테이블 생성 → ② 청크 단위로 행 복사(멈추고 재개 가능)
       → ③ 복사 중 들어온 변경을 고스트에 반영  ★ 여기가 갈린다  → ④ RENAME 으로 맞바꾼다
```

| | **pt-online-schema-change** | **gh-ost** |
|---|---|---|
| ③ 변경 반영 | **트리거** | **binlog 구독** (트리거 없음) |
| 원본 테이블 부하 | 쓰기마다 트리거가 **동기 실행** | 원본에 아무것도 안 붙는다 |
| 기존 트리거 | **이미 트리거가 있으면 동작하지 않는다** | 무관 |
| 일시 정지 | 복사만 멈춘다(트리거는 계속 돈다) | **완전히 멈춘다** — 복사도 이벤트 처리도 |
| 사전 검증 | 어려움 | **리플리카에서 먼저 돌려 보고 결과를 비교**할 수 있다 |
| 전환 시점 | RENAME 자동 | **사람이 있을 때로 미룰 수 있다** |

**gh-ost가 트리거를 버린 이유가 핵심이다.** 트리거는 원본 테이블의 **쓰기 트랜잭션 안에서 동기로** 실행된다. 즉 마이그레이션 부하가 **서비스 쓰기 지연에 그대로 더해지고**, 스로틀링을 걸어도 트리거는 계속 돈다. binlog를 읽는 방식은 **비동기**라 원본에 부하를 안 얹고, 그래서 **진짜로 멈출 수 있다**.

둘 다 **복제 지연을 보고 스로틀링**한다(pt-osc의 `--max-lag`) — [복제와 파티셔닝](../basics/replication-and-partitioning.md) §2-4의 지연이 **여기서는 제어 신호**로 쓰인다. 그리고 **외래 키가 있으면 둘 다 어려워진다**(pt-osc는 원자적 `RENAME`이 성립하지 않아 `--alter-foreign-keys-method`가 필요하다).

> **선택 기준**: 8.0에서 **`INSTANT`로 되는 연산이면 도구를 쓸 이유가 없다.** 도구는 §2-5의 **`COPY`가 강제되는 연산**(타입 변경·PK 변경·파티셔닝 도입)에서만 값어치가 있다. 그리고 어느 쪽이든 **원본 테이블 크기만큼의 여유 디스크**가 필요하다.

### 2-7. utf8mb4와 콜레이션 — 기본값이 바뀌었다

**MySQL 8.4의 서버 기본값은 `utf8mb4` / `utf8mb4_0900_ai_ci`다.** 그 이전 세대(`utf8` = 3바이트, `utf8mb4_general_ci`)에서 옮겨 온 스키마와 **기본값이 다르다**는 게 사고의 출발점이다.

| 콜레이션 | 기반 | 특징 | 패딩 |
|---|---|---|---|
| **`utf8mb4_0900_ai_ci`** (8.x 기본) | **UCA 9.0.0** | 악센트·대소문자 무시. *"faster than collations based on UCA versions prior to 9.0.0"* | **`NO PAD`** |
| `utf8mb4_unicode_ci` | UCA 4.0.0 | 확장·축약·무시 문자 지원(독일어 `ß = ss`) | `PAD SPACE` |
| `utf8mb4_general_ci` | UCA 아님(레거시) | *"only one-to-one comparisons between characters"* — `ß = s` | `PAD SPACE` |

**`NO PAD` 차이가 조용한 사고를 만든다.** `PAD SPACE` 계열은 비교할 때 **끝의 공백을 무시**해서 `'홍길동'`과 `'홍길동 '`을 같다고 보고, `NO PAD`는 **다르다고 본다.** 수신자명에 `UNIQUE`가 걸려 있으면 `'홍길동 '` 삽입이 **전자에서는 중복 에러, 후자에서는 성공**이다 — 같은 스키마가 이사 후에 다르게 동작한다.

**진짜 문제는 섞였을 때다.** 테이블마다 콜레이션이 다르면 **조인 조건에서 `Illegal mix of collations` 에러**가 나거나, 암묵적 변환이 일어나 **인덱스를 못 타게 된다** — [DB 인덱스](../basics/b-tree-index.md) §2-6의 암묵적 형변환과 **같은 함정의 문자열 판**이다.

> **판단**: 신규는 기본값(`0900_ai_ci`)을 쓴다. **기존 스키마에 테이블을 추가할 때는 기본값을 따르지 말고 기존 콜레이션에 맞춘다** — 한 스키마 안에서 섞는 것이 어느 한쪽을 쓰는 것보다 나쁘다. 이모지가 들어오는 발송 본문에는 **`utf8mb4`가 아니라 3바이트 `utf8`이면 저장 자체가 실패**한다는 점도 같이 확인할 것.

### 2-8. `TIMESTAMP` vs `DATETIME` — 예약 발송이 걸리는 자리

> *"MySQL converts `TIMESTAMP` values from the current time zone to UTC for storage, and back from UTC to the current time zone for retrieval. (This does not occur for other types such as `DATETIME`.)"*

| | `TIMESTAMP` | `DATETIME` |
|---|---|---|
| 범위 | **1970-01-01 00:00:01 UTC ~ 2038-01-19 03:14:07 UTC** | 1000-01-01 ~ 9999-12-31 |
| 타임존 변환 | **한다**(세션 타임존 ↔ UTC) | **안 한다**. 넣은 문자열 그대로 |
| 의미 | **시점**(어느 지역에서 보든 같은 순간) | **달력의 벽시계 값** |

**이 차이가 두 방향의 사고를 만든다.** ① `DATETIME`에 UTC를 넣고 KST로 읽으면 변환이 없으니 **9시간 어긋난 값이 조용히 나온다** — 에러가 안 나므로 발송이 엉뚱한 시각에 나가기 전까지 모른다. ② `TIMESTAMP`인데 애플리케이션 서버 A는 UTC, B는 `Asia/Seoul`로 커넥션을 열면 **같은 컬럼을 읽고 다른 값을 본다** — 배치 서버와 API 서버가 다르게 떠 있는 흔한 구성에서 나온다.

**예약 발송에서는 "무엇을 저장할 것인가"부터 갈린다.**

```
"2026-08-20 09:00 에 보내 달라"  ← 고객이 말한 것은 *벽시계 시각*이다
  UTC 시점으로 저장            → 서머타임/타임존 정책이 바뀌면 의도와 어긋난다
  벽시계 + 타임존을 따로 저장   → DATETIME(예약 시각) + VARCHAR(타임존 ID)
```

**기본형은 UTC 시점으로 통일하고 표시에서만 변환하는 것**이고, 위 예약 발송처럼 **"벽시계 약속"이 원본 의도**인 경우가 예외다. 그리고 **2038 문제가 실제 제약**이라 장기 보관 이력이나 만료일에는 `TIMESTAMP`를 못 쓴다. 둘 다 **마이크로초(6자리)**까지 지원하니, 같은 초에 수천 건이 쌓이는 발송 로그는 정밀도를 명시해야 정렬이 안정된다.

### 2-9. 그래서 갈라지는 것들

| 항목 | MySQL 8.4 | PostgreSQL 17 |
|---|---|---|
| 복제 로그 | **binlog**(redo와 별개, 논리 계열) | **WAL 하나** + 논리 복제 별도 |
| 포맷 선택 | **ROW**(기본) / STATEMENT / MIXED | 물리 복제는 선택 없음 |
| 트랜잭션 식별 | **GTID** — 전역 고유 이름, 자동 스킵 | LSN(위치) + 복제 슬롯 |
| 절충 동기화 | **반동기** — **수신(릴레이 로그)까지만** 대기 | `synchronous_commit` 4단계, **`remote_apply`는 재생까지** |
| 동기 대상이 죽으면 | **타임아웃 → 비동기로 강등**(A를 택함) | **커밋이 안 끝날 수 있다**(C를 택함) |
| 밀린 로그 보관 | 없음(`binlog_expire_logs_seconds`로 만료) | **복제 슬롯** — 안전장치이자 폭탄 |
| 파티셔닝 | RANGE / LIST / HASH / **KEY** | 선언적 파티셔닝, RANGE / LIST / HASH |
| 파티션의 정체 | 테이블 내부 구조 | **각 파티션이 독립 테이블** |
| 스키마 변경 | **`INSTANT`/`INPLACE`/`COPY`** 명시 가능 | 대부분 카탈로그 수정(예: 컬럼 추가) |
| 문자열 비교 | **콜레이션이 컬럼마다** 붙는다 | DB 생성 시 로케일이 대체로 고정 |

**네 번째·다섯 번째 줄이 성격을 가른다.** 같은 "절충"인데 **MySQL은 못 기다리면 보장을 버리고 계속 가고, PostgreSQL은 보장을 지키려 멈춘다.** 어느 쪽이 옳은 게 아니라 **기본값이 반대**인 것이고, 그래서 **한쪽 감각으로 다른 쪽을 운영하면 놀란다** — MySQL 쪽은 "손실 없는 줄 알았는데 강등돼 있었다", PostgreSQL 쪽은 "리플리카 하나 죽었는데 쓰기가 전부 멈췄다"가 된다.

---

**다음으로 읽을 것**

- 복제 원리·복제 지연·read-your-writes·페일오버 → [복제와 파티셔닝](../basics/replication-and-partitioning.md) §2-4, §2-5, §2-6
- binlog와 redo의 2단계 커밋 → [내구성과 복구](../basics/durability-and-recovery.md) §2-8
- 보존 정책으로서의 파티셔닝과 `DROP PARTITION` → [DB 인덱스](../basics/b-tree-index.md) §2-12
- `ALTER`로 추가할 인덱스를 고르는 쪽 → [인덱스와 옵티마이저](index-and-optimizer.md) §2-4
- 같은 자리에서 PostgreSQL이 치르는 비용 → [복제와 운영](../../postgresql/replication-and-ops.md) §2-3, §2-9

---

> **기준 버전**: MySQL 8.4 · PostgreSQL 17
> **확인한 출처** (별도 표기가 없으면 dev.mysql.com 8.4):
> - [Replication Formats](https://dev.mysql.com/doc/refman/8.4/en/binary-log-formats.html) — **ROW가 기본값**(*"In row-based logging (the default)"*), 세 포맷의 동작, *"Statement may not be safe to log in statement format"* 경고
> - [GTID Concepts](https://dev.mysql.com/doc/refman/8.4/en/replication-gtids-concepts.html) — `source_id:transaction_id`와 **8.4의 태그 형식**, *"unique across all servers in a given replication topology"*, **자동 스킵**(*"applied no more than once on the replica"*), *"monotonically increasing GTIDs without gaps"*, `gtid_mode` 3단계와 `enforce_gtid_consistency`
> - [Semisynchronous Replication](https://dev.mysql.com/doc/refman/8.4/en/replication-semisync.html) — **릴레이 로그 기록·flush 후 ACK**라는 명시, 기본 ACK 1개, **타임아웃 시 비동기 강등**, `rpl_semi_sync_source_wait_point`의 `AFTER_SYNC`(기본, 스토리지 엔진 커밋 전) vs `AFTER_COMMIT`
> - [Partitioning Types](https://dev.mysql.com/doc/refman/8.4/en/partitioning-types.html) — RANGE/LIST/HASH/KEY와 COLUMNS·LINEAR 변형, **KEY는 서버 내장 해시라 비정수 컬럼 가능** / [Partitioning Keys and Unique Keys](https://dev.mysql.com/doc/refman/8.4/en/partitioning-limitations-partitioning-keys-unique-keys.html) — *"All columns used in the partitioning expression ... must be part of every unique key"* 원문과 위반 예시
> - [Online DDL Operations](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html) — `INSTANT`/`INPLACE`+동시 DML 지원 연산 목록, **컬럼 타입 변경·PK 삭제는 `COPY`**, PK 추가는 `INPLACE`이나 재구축 / [What Is New in 8.0](https://dev.mysql.com/doc/refman/8.0/en/mysql-nutshell.html) — **`INSTANT` 컬럼 추가 8.0.12 · 삭제 8.0.29**
> - [Online DDL Performance and Concurrency](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-performance.html) — `LOCK` 4단계와 **지원 수준보다 느슨하게 요구하면 에러로 실패**, 커밋 단계의 **배타 MDL**, 장시간 트랜잭션이 `ALTER`를 막고 **대기 중인 배타 MDL이 후속 트랜잭션까지 막는다**는 원문 / [Space Requirements](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-space-requirements.html) — `innodb_online_alter_log_max_size` 초과 시 **`DB_ONLINE_LOG_TOO_BIG`으로 실패·롤백**, 정렬·중간 테이블 파일의 공간 요구
> - [Server Character Set and Collation](https://dev.mysql.com/doc/refman/8.4/en/charset-server.html) — **기본값 `utf8mb4` / `utf8mb4_0900_ai_ci`** / [Unicode Character Sets](https://dev.mysql.com/doc/refman/8.4/en/charset-unicode-sets.html) — `0900_ai_ci`(UCA 9.0.0, `NO PAD`, 더 빠름) vs `unicode_ci`(UCA 4.0.0, `PAD SPACE`) vs `general_ci`(레거시, *"only one-to-one comparisons"*)
> - [DATE, DATETIME, TIMESTAMP](https://dev.mysql.com/doc/refman/8.4/en/datetime.html) — 타임존 변환 원문(*"This does not occur for other types such as DATETIME"*), 두 타입의 범위와 **2038 상한**, 마이크로초 지원
> - [gh-ost](https://github.com/github/gh-ost) — 트리거리스·binlog 구독, 완전한 일시 정지, 리플리카 사전 검증, 전환 시점 지연 / [pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html) — 청크 복사 + **트리거 동기화** → 원자적 `RENAME TABLE`, **기존 트리거가 있으면 동작 불가**, `--alter-foreign-keys-method`, `--max-lag` 스로틀링
> **미확인**: MySQL 8.4에서 반동기가 **플러그인인지 컴포넌트인지**와 변수 이름의 정확한 세대(`rpl_semi_sync_source_*`는 문서에서 확인했으나 패키징 형태는 대조하지 않았다) · `binlog_format`이 8.x 후반에 **deprecated 됐는지**(대조한 8.4 페이지에는 관련 서술이 없었다) · `ALGORITHM=INSTANT`의 **누적 행 버전 상한**(널리 알려진 제약이나 이번에 원문을 못 찾았다) · `FULLTEXT` 인덱스 추가 시 동시 DML 허용 여부 · MySQL의 **메이저 버전 간 복제 지원 범위**(업그레이드 방향만 지원된다고 알려져 있으나 정책 문서를 대조하지 않았다) · 병렬 재생(`replica_parallel_workers`) 설정과 순서 보장 · §2-1의 binlog 크기·§2-6의 디스크 여유 서술은 **자릿수 감각용**이며 실측이 아니다
> **미작성**: 그룹 복제(InnoDB Cluster) · 다중 소스 복제 · 지연 리플리카 · `mysqlbinlog`를 이용한 PITR 절차 · 백업 도구(`mysqldump` / Percona XtraBackup) · `binlog_expire_logs_seconds` 운영 기준
