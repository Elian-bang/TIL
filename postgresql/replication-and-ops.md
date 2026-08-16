# 복제와 운영 — WAL 하나로 다 하는 대신 치르는 것

## 30초 요약

- 복제가 두 종류다 — **물리**는 *"정확한 블록 주소와 바이트 단위"*, **논리**는 *"복제 아이덴티티(보통 PK) 기반"*. 둘을 동시에 쓸 수 있다
- 논리 복제의 킬러 용례는 공식 문서에 그대로 있다 — **메이저 버전 간 복제**. 이게 **무중단 업그레이드 경로**를 만든다
- **복제 슬롯은 안전장치이자 폭탄**이다. 리플리카가 안 돌아오면 WAL이 쌓이고 **동시에 VACUUM이 마비된다**
- `pg_wal/`이 꽉 차면 **PostgreSQL은 PANIC 종료한다.** 공식 문서 표현으로 *"커밋된 트랜잭션은 안 잃지만 공간을 비울 때까지 오프라인"*
- **커넥션 하나가 프로세스 하나**다. 그래서 PgBouncer가 선택이 아니라 사실상 필수 부품이 된다

---

## 원리 — 왜 그런가

> [복제와 파티셔닝](../database/basics/replication-and-partitioning.md)에서 복제의 원리를 봤다. 이 문서는 **PostgreSQL이 그걸 어떻게 구현했고 무엇이 운영 업무가 되는지**다.

### 2-1. 물리 복제와 논리 복제

공식 문서가 둘을 이렇게 대비한다.

> *"Logical replication is a method of replicating data objects and their changes, based upon their replication identity (usually a primary key). We use the term logical in contrast to physical replication, which uses exact block addresses and byte-by-byte replication."*

**물리 복제 = WAL을 그대로 보낸다.** 리플리카는 바이트 단위로 동일한 사본이 된다. 싸고 단순하다. 대신 **전부 아니면 전무**다 — 테이블 하나만 복제할 수 없고, 버전이 달라도 안 되고, 리플리카에 쓰기도 못 한다.

**논리 복제 = "이 행이 이렇게 바뀌었다"를 보낸다.** WAL을 해석해 행 단위 변경으로 바꿔 전송한다. 비싸지만 자유롭다.

> **"replication identity(보통 PK)"라는 단서가 중요하다.** 논리 복제는 "어느 행인지"를 식별해야 하는데, 물리 주소(ctid)를 쓸 수 없으니 **키가 필요하다**. PK 없는 테이블은 논리 복제에서 문제가 된다 — [데이터 모델링](../database/basics/data-modeling.md)의 "PK를 반드시 두라"가 여기서 운영 요구사항이 된다.

**모델은 publish/subscribe이고 구독자가 당겨 간다.**

> *"Subscribers pull data from the publications they subscribe to and may subsequently re-publish data to allow cascading replication."*

그리고 순서 보장이 명시돼 있다 — *"The subscriber applies the data in the same order as the publisher so that transactional consistency is guaranteed."*

### 2-2. 논리 복제의 킬러 용례 — 무중단 메이저 업그레이드

공식 문서가 대표 용례를 나열하는데, 그중 둘이 실무에서 압도적이다.

> - *"Consolidating multiple databases into a single one (for example for analytical purposes)."*
> - **"Replicating between different major versions of PostgreSQL."**
> - *"Replicating between PostgreSQL instances on different platforms (for example Linux to Windows)."*

**두 번째가 업그레이드 문제를 푼다.** 물리 복제는 바이트 단위라 버전이 다르면 성립하지 않는다. 그래서 메이저 업그레이드는 원래 **덤프/복원이나 `pg_upgrade`로 다운타임을 감수**해야 했다.

```
논리 복제로 하는 업그레이드:
① 새 버전 인스턴스를 띄운다
② 논리 복제로 데이터를 따라잡게 한다   ← 서비스는 계속 구버전에서 돈다
③ 지연이 0에 수렴하면 애플리케이션을 새 쪽으로 전환
④ 구버전 정리
```

**다운타임이 ③의 전환 순간으로 줄어든다.**

> ⚠️ **공짜는 아니다.** 논리 복제는 **DDL을 복제하지 않는다**는 제약이 널리 알려져 있고(공식 문서의 별도 Restrictions 절에 정리돼 있으나 이번에 대조하지 않았다), 시퀀스 값도 따로 챙겨야 한다. **전환 직전에 시퀀스를 맞추지 않으면 새 인스턴스에서 PK 충돌이 난다** — 업그레이드 실패담의 단골이다.

### 2-3. 복제 슬롯 — 안전장치가 폭탄이 되는 지점

리플리카가 잠깐 끊겼다 돌아왔을 때 **필요한 WAL이 이미 지워졌으면** 따라잡을 수 없다. **복제 슬롯**은 그걸 막는다 — 프라이머리가 **"이 슬롯이 아직 안 받아 갔다"**를 기억하고 그 WAL을 **보관**한다.

**안전한 대신 위험하다. 리플리카가 영영 안 돌아오면 슬롯이 계속 붙잡는다.**

```
죽은 리플리카의 슬롯을 방치
  ├→ WAL이 지워지지 않고 pg_wal/ 에 쌓인다        → 디스크 고갈
  └→ 슬롯이 xmin도 붙잡는다                        → VACUUM 마비
                                                    → 블로트 + XID wraparound 시계
```

**두 사고가 동시에 온다.** [MVCC와 VACUUM](mvcc-and-vacuum.md) §2-6에서 봤듯 공식 문서의 wraparound 복구 절차에도 **"오래된 복제 슬롯 제거"**가 들어 있다. **슬롯 하나 방치가 디스크와 wraparound를 동시에 당긴다는 게 PostgreSQL 운영의 대표적 함정**이다.

> `max_slot_wal_keep_size`로 보관량에 상한을 걸 수 있다. 넘으면 **슬롯을 무효화하고 WAL을 지운다** — 그 리플리카는 재구축해야 하지만 **프라이머리는 산다.** "리플리카 하나를 포기하고 클러스터를 살리는" 선택지를 미리 설정해 두는 것이다.

### 2-4. PITR — 백업 하나 + WAL

[내구성과 복구](../database/basics/durability-and-recovery.md) §2-8에서 본 시간여행의 PostgreSQL 구현이다.

> *"we can combine a file-system-level backup with backup of the WAL files. If recovery is needed, we restore the file system backup and then replay from the backed-up WAL files."*

**핵심은 "끝까지 재생할 필요가 없다"는 것이다.**

> *"It is not necessary to replay the WAL entries all the way to the end. We could stop the replay at any point and have a consistent snapshot of the database as it was at that time."*

**멈출 지점(recovery target)을 지정하는 방법이 셋인데 실용성이 다르다.**

> *"either by date/time, named restore point or by completion of a specific transaction ID. As of this writing only the date/time and named restore point options are very usable, since there are no tools to help you identify with any accuracy which transaction ID to use."*

**트랜잭션 ID로 지정하는 건 사실상 못 쓴다**고 문서가 인정한다. **"사고 직전 XID"를 알아낼 방법이 없기 때문**이다. 실무에서는 시각으로 지정하거나, 위험한 작업 전에 **명명된 복구 지점**을 미리 찍어 두는 쪽이 확실하다.

**제약 하나** — *"The stop point must be after the ending time of the base backup."* 백업이 진행 중이던 시점으로는 못 돌아간다.

### 2-5. WAL 아카이빙이 실패하면 — PANIC 종료

PITR을 하려면 WAL을 계속 어딘가로 복사해 둬야 한다(`archive_command`). **이 복사가 실패하기 시작하면 무슨 일이 생기나.**

> *"The `pg_wal/` directory will continue to fill with WAL segment files until the situation is resolved. (If the file system containing `pg_wal/` fills up, PostgreSQL will do a PANIC shutdown. No committed transactions will be lost, but the database will remain offline until you free some space.)"*

**DB가 멈춘다.** 데이터는 안 잃지만 **서비스는 정지**다.

원인이 §2-3과 같은 계열이다 — **"안 지우고 붙잡는" 장치가 둘(복제 슬롯, 아카이빙 대기) 있고 둘 다 방치하면 같은 곳(`pg_wal/`)을 채운다.** PostgreSQL 디스크 알람에서 제일 먼저 볼 곳이 여기인 이유다.

**아카이브 명령 작성에 문서가 두 가지를 못 박는다.**

> *"It is important that the archive command return zero exit status if and only if it succeeds."*

성공하지 않았는데 0을 반환하면 **PostgreSQL은 아카이브됐다고 믿고 그 WAL을 지운다.** 그 순간 백업 체인이 조용히 끊긴다 — 복구를 시도하는 날에야 안다.

> *"Archive commands and libraries should generally be designed to refuse to overwrite any pre-existing archive file. This is an important safety feature."*

두 서버의 WAL을 같은 디렉터리로 보내는 실수를 막기 위한 것이다. 문서의 예시도 `test ! -f`로 존재 여부를 먼저 확인한다.

### 2-6. 선언적 파티셔닝

[복제와 파티셔닝](../database/basics/replication-and-partitioning.md) §2-7의 원리를 PostgreSQL은 **선언적 파티셔닝**(10 이상)으로 제공한다. RANGE / LIST / HASH를 지원하고, 그 이전의 상속 기반 방식은 사실상 레거시다.

**MySQL과 다른 점 하나** — PostgreSQL의 각 파티션은 **독립된 테이블**이다. 그래서 파티션마다 다른 인덱스를 걸거나, 오래된 파티션만 다른 테이블스페이스(느린 디스크)로 옮기는 게 자연스럽다.

**주의할 운영 항목 하나**가 [MVCC와 VACUUM](mvcc-and-vacuum.md)과 얽힌다 — 공식 문서가 명시하듯 **autovacuum은 파티션을 개별로만 처리하고 부모 테이블에는 `ANALYZE`를 하지 않는다.** 부모 기준 통계가 낡으면 플래너가 파티션 선택을 잘못할 수 있어 **부모 `ANALYZE`는 수동으로 챙겨야 한다.**

### 2-7. 커넥션 하나가 프로세스 하나 — PgBouncer가 필수인 이유

**PostgreSQL은 커넥션마다 프로세스를 하나 fork한다.** MySQL이 스레드를 쓰는 것과 근본적으로 다르다([프로세스와 스레드](../os/process-and-scheduling.md) §2-1).

```
커넥션 1000개 = 프로세스 1000개
  → 각자 독립 메모리 (work_mem 등이 프로세스별로 붙는다)
  → 컨텍스트 스위칭 비용이 프로세스 단위
```

**[트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-10의 "커넥션 풀을 크게 잡으면 안 된다"가 PostgreSQL에서 훨씬 더 무겁다.** 애플리케이션이 여러 대이고 각자 풀을 50씩 잡으면 금방 수백 프로세스가 된다.

**그래서 PgBouncer 같은 커넥션 풀러를 DB 앞에 둔다.** 애플리케이션 커넥션 수천 개를 받아 **실제 DB 커넥션 수십 개로 다중화**한다.

**풀링 모드가 셋이고 트레이드오프가 명확하다.**

| 모드 | 언제 반납 | 문제 |
|---|---|---|
| session | 클라이언트가 끊을 때 | 다중화 효과가 거의 없다 |
| **transaction** | **트랜잭션이 끝날 때** | 가장 많이 쓴다. **세션 상태가 깨진다** |
| statement | 문장마다 | 트랜잭션을 못 쓴다 |

**transaction 모드의 함정이 이 시리즈와 직접 이어진다.** 커넥션이 트랜잭션마다 다른 클라이언트에게 넘어가므로 **세션에 붙는 것들이 전부 위험해진다** — 임시 테이블, 세션 변수, prepared statement, 그리고 **[락과 타입](lock-and-types.md) §2-6의 세션 수준 advisory lock**.

> **세션 advisory lock + transaction 풀링 = 최악의 조합**이다. 락은 롤백해도 안 풀리는데(§lock-and-types 2-6), 그 커넥션은 곧바로 다른 요청에게 넘어간다. **누가 잡았는지도 모르는 락이 남고, 다음 사용자가 그 커넥션을 물려받는다.** 풀러를 쓸 거면 `pg_advisory_xact_lock`(트랜잭션 수준)만 쓰는 게 안전하다.

### 2-8. extension — 부족한 걸 표준 방식으로 붙인다

PostgreSQL의 특징 중 실무 체감이 가장 큰 항목이다. **핵심을 건드리지 않고 기능을 추가하는 공식 경로**가 있다.

| extension | 무엇 |
|---|---|
| **`pg_stat_statements`** | 쿼리별 누적 실행 통계. **총 실행시간 상위 쿼리**를 찾는 표준 도구 |
| **`pg_trgm`** | 3글자 단위 유사도. **`LIKE '%...%'`에 인덱스를 태운다** |
| `pgvector` | 벡터 유사도 검색 |
| `PostGIS` | 공간 데이터 |
| `pg_partman` | 파티션 자동 생성·삭제 |

**`pg_stat_statements`는 사실상 필수**다. [DB 인덱스](../database/basics/b-tree-index.md) §2-10에서 "슬로우 로그는 한 방에 느린 쿼리를 잡고, 총합이 큰 쿼리는 다른 도구로 잡는다"고 했는데, PostgreSQL에서 그 후자가 이것이다.

**`pg_trgm`은 [DB 인덱스](../database/basics/b-tree-index.md) §2-2의 예외를 만든다.** "앞이 와일드카드면 인덱스를 못 탄다"가 원칙인데, 문자열을 3글자씩 쪼개 GIN 인덱스에 넣으면 **중간 검색도 인덱스를 탄다.** 전문 검색 엔진을 도입하기 전에 **RDB 안에서 해결되는 구간**을 넓혀 준다([Elasticsearch](../datastore/elasticsearch.md)를 언제 도입할지의 판단선이 뒤로 밀린다).

> **extension이 있다는 게 "무엇이든 된다"는 뜻은 아니다.** 관리형 서비스(RDS 등)는 **허용 목록이 정해져 있다.** 설계 단계에서 "우리 환경에서 이 extension을 쓸 수 있나"를 먼저 확인해야 한다.

### 2-9. 그래서 MySQL과 무엇이 다른가

| 항목 | MySQL | PostgreSQL |
|---|---|---|
| 복제 로그 | **binlog**(redo와 별개) | **WAL 하나** + 논리 복제 |
| 커밋 시 2단계 커밋 | 필요(redo ↔ binlog) | **불필요** |
| 메이저 버전 간 복제 | 가능(binlog가 논리 계열) | **논리 복제로 가능** |
| 밀린 로그 보관 | 없음 | **복제 슬롯** — 안전하지만 방치 시 폭탄 |
| 로그 디스크 고갈 시 | 쓰기 스톨 | **PANIC 종료**(오프라인) |
| 커넥션 모델 | 스레드 | **프로세스** → 풀러 사실상 필수 |
| 기능 확장 | 플러그인 제한적 | **extension 생태계** |

**"WAL 하나로 복구·복제·PITR을 다 한다"는 단순함의 대가가 복제 슬롯과 아카이빙**이다([내구성과 복구](../database/basics/durability-and-recovery.md) §2-9). MySQL은 로그가 둘이라 2단계 커밋이 필요한 대신, **한쪽이 밀려도 다른 쪽이 안 막힌다.** 어느 쪽도 공짜가 아니다.

---

**다음으로 읽을 것**

- 복제 슬롯이 VACUUM을 어떻게 막나 → [MVCC와 VACUUM](mvcc-and-vacuum.md) §2-6
- advisory lock과 풀링의 조합 위험 → [락과 타입](lock-and-types.md) §2-6
- 복제 일반 원리와 페일오버 → [복제와 파티셔닝](../database/basics/replication-and-partitioning.md)
- 로그가 복구·복제·PITR에 같이 쓰이는 이유 → [내구성과 복구](../database/basics/durability-and-recovery.md) §2-8

---

> **기준 버전**: PostgreSQL 17
> **확인한 출처**:
> - [PostgreSQL 29 Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html) — 논리/물리 대비 원문(*"based upon their replication identity (usually a primary key)"* vs *"exact block addresses and byte-by-byte"*), publish/subscribe와 구독자 pull·캐스케이딩, 순서 보장, **대표 용례 목록(메이저 버전 간·플랫폼 간·다중 DB 통합)**
> - [PostgreSQL 25.3 Continuous Archiving and PITR](https://www.postgresql.org/docs/current/continuous-archiving.html) — base backup + WAL 재생 원리, *"not necessary to replay the WAL entries all the way to the end"*, 복구 목표 3종과 **트랜잭션 ID 방식이 실용적이지 않다는 인정**, 정지 지점이 백업 종료 이후여야 한다는 제약, **`pg_wal/` 고갈 시 PANIC 종료** 원문, `archive_command`가 성공할 때만 0을 반환해야 한다는 요구와 기존 파일 덮어쓰기 금지
> **미확인**: §2-2의 논리 복제 제약(DDL·시퀀스 미복제)은 별도 Restrictions 절 대조하지 않았다 · §2-3 `max_slot_wal_keep_size` 동작 · §2-6 선언적 파티셔닝 세부 · §2-7 PgBouncer 풀링 모드와 세션 상태 제약 · §2-8 각 extension의 동작 — 전부 대조 필요
> **미작성**: 스트리밍 복제 설정 절차 · 동기 스탠바이 이름 지정 · 논리 복제 충돌 해결 · pgBackRest 등 백업 도구
