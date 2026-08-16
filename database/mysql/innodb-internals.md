# InnoDB 내부 구조 — 엔진을 갈아 끼울 수 있게 만든 대가

## 30초 요약

- MySQL은 **저장 엔진을 테이블 단위로 갈아 끼울 수 있게** 만들었다. 그래서 서버 층(파서·옵티마이저)은 **트랜잭션도 락도 직접 모른다** — 엔진이 정한다
- 버퍼 풀 위에 얹힌 세 장치(**체인지 버퍼 · AHI · 더블라이트**)는 전부 **"랜덤 I/O가 비싸다"는 전제** 위에 세워졌다
- **MySQL 8.4가 그 전제를 두 개 철회했다** — 체인지 버퍼 기본 `none`, AHI 기본 `OFF`. 8.0까지의 튜닝 지식이 그대로 낡는다
- 더블라이트만 켜진 채 남았다. **torn page는 디스크가 빨라져도 안 사라지는 문제**여서다
- row format 기본이 `DYNAMIC`인 이유는 하나다 — **큰 값의 768바이트 프리픽스를 행 안에 안 남긴다.** 행이 작아야 페이지에 많이 들어간다

---

## 원리 — 왜 그런가

> [저장과 I/O](../basics/storage-and-io.md) §2-2~2-3에서 **버퍼 풀**과 **LRU 미드포인트 삽입**을 이미 다뤘다. 이 문서는 **그 위에 InnoDB가 더 얹어 둔 것들**과, 그것들이 **디스크의 어디에 사는가**다.

### 2-1. 엔진을 갈아 끼운다는 설계

MySQL의 구조적 특이점은 성능이 아니라 **경계선의 위치**다.

> *"MySQL Server uses a pluggable storage engine architecture that enables storage engines to be loaded into and unloaded from a running MySQL server."*

그리고 그 선택이 **서버가 아니라 테이블 단위**다.

> *"You are not restricted to using the same storage engine for an entire server or schema. You can specify the storage engine for any table."*

**여기서 나오는 결과가 중요하다.** 서버 층(SQL 파싱·옵티마이저·복제)과 저장 층이 분리돼 있으므로, **트랜잭션·락·MVCC·크래시 복구가 전부 엔진의 소관**이 된다. 같은 서버, 같은 SQL인데 **테이블마다 보장이 달라진다.**

| 엔진 | 트랜잭션 | 락 단위 | 크래시 복구 |
|---|---|---|---|
| **InnoDB** (8.4 기본) | 있음 | **행** | 있음 |
| MyISAM | 없음 | **테이블** | 없음 |
| MEMORY | 없음 | **테이블** | 해당 없음(휘발) |

공식 문서가 MyISAM을 *"Table-level locking limits the performance in read/write workloads"*라고 적고, MEMORY에 대해서는 *"Its use cases are decreasing; InnoDB with its buffer pool memory area provides a general-purpose and durable way to keep most or all data in memory"*라고 적어 뒀다. **둘 다 "예전에 필요했던 자리를 InnoDB가 이미 덮었다"는 서술**이다.

**그래서 사고가 이런 모양으로 난다.** 발송 이력 테이블 하나가 옛 스크립트 때문에 MyISAM으로 남아 있으면 그 테이블 쓰기는 **롤백이 안 된다.** 애플리케이션은 `@Transactional` 안이라 안전하다고 믿는데 실제로는 아니다. **엔진을 고를 수 있다는 건 잘못 고를 수도 있다는 뜻**이다.

> **[복제와 운영](../../postgresql/replication-and-ops.md) §2-9가 "MySQL은 플러그인이 제한적"이라고 적은 것과 모순이 아니다.** MySQL이 열어 둔 이음매는 **저장 엔진이라는 한 층**이고, PostgreSQL의 extension은 **타입·인덱스·함수 전반**이다. 깊고 좁은 이음매 하나 vs 얕고 넓은 이음매 여럿의 차이다.

### 2-2. 버퍼 풀 위에 무엇이 더 얹혀 있나

공식 아키텍처 문서가 꼽는 **인메모리 구조는 넷**이다 — **버퍼 풀**([저장과 I/O](../basics/storage-and-io.md) §2-2~2-3), **체인지 버퍼**(§2-3), **Adaptive Hash Index**(§2-4), **로그 버퍼**([내구성과 복구](../basics/durability-and-recovery.md) §2-1의 "커밋 전까지 쌓이는 곳").

**뒤의 셋은 전부 같은 전제 위에 있다** — *디스크 접근, 특히 랜덤 접근이 비싸다*. 그 전제가 얼마나 참인지가 하드웨어에 따라 달라지고, **8.4에서 기본값 두 개가 뒤집혔다.**

### 2-3. 체인지 버퍼 — 랜덤 쓰기를 미루는 도박

**문제부터.** 발송 이력에 행 하나를 INSERT하면 클러스터드 인덱스(=테이블)에는 PK 순서대로 붙으니 순차에 가깝다. 그런데 세컨더리 인덱스는 다르다.

> *"Unlike clustered indexes, secondary indexes are usually nonunique, and inserts into secondary indexes happen in a relatively random order."*

`receiver_phone` 인덱스에 들어갈 자리는 전화번호 순서라 **INSERT마다 전혀 다른 페이지**다. 그 페이지가 버퍼 풀에 없으면 **인덱스 하나당 랜덤 읽기 한 번**이 붙는다. 인덱스 5개면 5번이다.

**체인지 버퍼는 그 페이지를 지금 읽지 않는다.**

> *"The change buffer is a special data structure that caches changes to secondary index pages when those pages are not in the buffer pool. The buffered changes... are merged later when the pages are loaded into the buffer pool by other read operations."*

**핵심은 "다른 읽기 작업이 그 페이지를 올릴 때 합친다"이다.** 나중에 조회 때문에 어차피 올라올 페이지라면, 그때 묻어서 처리하면 **읽기 I/O가 공짜가 된다.**

> *"...avoids substantial random access I/O that would be required to read secondary index pages into the buffer pool from disk."*

**⚠️ 그런데 MySQL 8.4에서 기본값이 꺼졌다.**

| | 8.0까지 | **8.4** |
|---|---|---|
| `innodb_change_buffering` | `all` | **`none`** |

공식 매뉴얼이 *"Before MySQL 8.4, the default value was `all`."*이라고 적어 두었고, 8.4.0 릴리스 노트도 기본값이 바뀐 변수 목록에 이 항목을 올려 뒀다. **다만 두 문서 어디에도 "왜"가 없다.** (아래 `미확인` 참고)

> **이유를 굳이 추정하자면** 이득("랜덤 읽기를 미룬 만큼")이 랜덤 읽기가 싸진 장비에서 줄어들기 때문일 텐데, **이건 내 추정이지 공식 근거가 아니다.** 실무 결론은 추정과 무관하게 하나다 — 8.0에서 8.4로 올리면 이 동작이 **말없이 바뀐다.** 대량 적재 배치의 INSERT 처리량을 8.0 기준으로 잡아 뒀다면 **재측정해야 한다.**

### 2-4. Adaptive Hash Index — B+Tree 위에 얹은 캐시

**같은 값을 계속 조회하면 B+Tree를 매번 루트부터 내려가는 게 낭비**다([DB 인덱스](../basics/b-tree-index.md) §2-1). InnoDB는 그걸 관찰해서 **스스로 해시 인덱스를 만든다.**

- 검색 패턴을 감시하다가, **인덱스 키의 프리픽스**로 해시 인덱스를 만든다
- **자주 접근되는 페이지에 대해서만** 만든다 — 전체가 아니다
- 사용자가 만들 수 없고 지울 수도 없다. **DDL에 안 나타난다**

> **[힙과 인덱스](../../postgresql/heap-and-index.md) §2-8 비교표의 "인덱스 타입 — InnoDB는 B+Tree 중심"과 충돌하지 않는다.** AHI는 사용자가 선언하는 인덱스 타입이 아니라 **B+Tree 위에 자동으로 얹히는 캐시 층**이다.

**언제 손해인가**가 이 기능의 성격을 보여준다. 공식 문서가 두 경우를 든다 — **동시 조인이 많은 워크로드**에서는 AHI 접근 자체가 경합 지점이 되고, **`LIKE`와 `%` 와일드카드** 질의는 애초에 이득이 없다. 관찰하고 유지하는 비용은 계속 내는데 이득이 없는 것이다.

**여기도 8.4에서 기본값이 뒤집혔다.**

| | 8.0까지 | **8.4** |
|---|---|---|
| `innodb_adaptive_hash_index` | `ON` | **`OFF`** |

> ⚠️ **§2-3과 §2-4에서 같은 함정을 조심해야 한다.** [Adaptive Hash Index 매뉴얼 페이지](https://dev.mysql.com/doc/refman/8.4/en/innodb-adaptive-hash.html) 본문은 여전히 *"enabled by default"* 취지로 읽히는데, **같은 8.4 매뉴얼의 파라미터 표와 릴리스 노트는 `OFF`라고 한다.** 기본값은 서술 문단이 아니라 **파라미터 표에서 확인해야 한다.** 더 확실한 건 운영 중인 인스턴스에서 `SELECT @@innodb_adaptive_hash_index`를 직접 찍어 보는 것이다.

**경합 진단 방법은 문서에 있다.** `SHOW ENGINE INNODB STATUS`의 `SEMAPHORES` 절에서 `btr0sea.c`에 생성된 rw-latch를 기다리는 스레드가 많으면 AHI 경합이다. 그때는 파티션 수(`innodb_adaptive_hash_index_parts`, 기본 8)를 늘리거나 꺼야 한다.

### 2-5. 더블라이트 버퍼 — 왜 이것만 살아남았나

[내구성과 복구](../basics/durability-and-recovery.md) §2-7에서 **torn page**와 그 존재 이유를 이미 다뤘다. 여기서는 **물리 구현**만 본다.

> *"the doublewrite buffer does not require twice as much I/O overhead or twice as many I/O operations. Data is written to the doublewrite buffer in a large sequential chunk, with a single `fsync()` call to the operating system."*

**"쓰기가 두 배인데 비용은 두 배가 아니다"의 근거가 이 한 문장**이다. 흩어진 페이지들을 **한 덩어리 순차 쓰기 + fsync 한 번**으로 처리한다. [저장과 I/O](../basics/storage-and-io.md) §2-7의 부등호가 여기서 또 답이다 — **랜덤 N회를 순차 1회로 바꿨기 때문에** 두 배가 안 된다.

**8.0.20부터 전용 파일로 나왔다.** 시스템 테이블스페이스 안이 아니라 `#ib_16384_0.dblwr` 같은 이름의 별도 파일이고, 8.4 기준 `innodb_doublewrite_files` 기본값은 **2**다(플러시 리스트용 하나, LRU 리스트용 하나).

**모드가 넷이라는 게 설계 의도를 보여준다** — `ON`(=`DETECT_AND_RECOVER`), `OFF`, `DETECT_ONLY`, `DETECT_AND_RECOVER`.

- `DETECT_ONLY`: **메타데이터만 쓴다.** 페이지 내용은 안 쓰므로 복구는 못 하지만 **찢어졌다는 사실은 알아챈다**
- 원자적 쓰기를 지원하는 하드웨어(Fusion-io NVMFS)에서는 **자동으로 꺼진다** — 하드웨어가 이미 보장하니까

> **`DETECT_ONLY`가 있다는 게 중요하다.** 탐지와 복구를 분리해 둔 것은, **복구 비용은 못 내지만 탐지는 포기 못 하는 자리**가 실제로 있다는 뜻이다. "조용히 깨진 데이터를 모르고 서비스하는 것"과 "깨진 걸 알고 멈추는 것"은 다르다.

**§2-3·§2-4와 대비된다.** 체인지 버퍼와 AHI는 *디스크가 느리다*는 전제 위에 있어서 하드웨어가 바뀌자 기본값이 뒤집혔다. **더블라이트의 전제는 다르다** — 페이지 크기(16KB)와 디스크의 원자적 쓰기 단위(보통 4KB)가 어긋난다는 사실은 장비가 빨라져도 그대로다.

### 2-6. undo와 redo가 사는 곳

[내구성과 복구](../basics/durability-and-recovery.md) §2-2에서 **redo와 undo의 역할 분담**을 다뤘다. 여기서는 **물리적으로 어디에 있고 무엇이 커지는가**다.

**undo — 전용 테이블스페이스 두 개로 시작한다.**

> *"Two default undo tablespaces are created when the MySQL instance is initialized... Default undo tablespace data files are named `undo_001` and `undo_002`."*

**undo가 시스템 테이블스페이스 밖으로 나온 게 핵심**이다. 안에 있던 시절에는 **한번 부풀면 줄일 방법이 없었다.** 지금은 테이블스페이스 단위로 **잘라낼(truncate) 수 있다.**

| 변수 | 기본값 | 뜻 |
|---|---|---|
| `innodb_undo_log_truncate` | **`OFF`** | 자동 잘라내기 |
| `innodb_max_undo_log_size` | 1073741824 (1 GiB) | 이 크기를 넘으면 잘라내기 후보 |
| `innodb_purge_rseg_truncate_frequency` | 128 | purge가 후보를 확인하는 주기 |

**⚠️ 자동 잘라내기가 기본 `OFF`다** — "undo가 알아서 줄어들겠지"는 기본 구성에서 사실이 아니다. **그리고 켜도 못 자르는 조건이 있다.** 해당 테이블스페이스의 롤백 세그먼트를 쓰는 트랜잭션과, **그것들이 끝나기 전에 시작된 트랜잭션**이 전부 끝나야 purge가 세그먼트를 놓는다.

> **[트랜잭션 · 락](../basics/transaction-and-lock.md) §2-3의 "긴 트랜잭션이 undo를 못 지우게 한다"가 여기서 파일 크기로 나타난다.** 발송 배치가 트랜잭션 하나로 30분을 돌면 그동안 undo 테이블스페이스는 계속 자란다. **원인은 디스크가 아니라 트랜잭션 길이**다.

**redo — 32개 파일로 관리된다.**

`#innodb_redo` 디렉터리에 **총 32개**를 유지하려 하고, 각 파일은 `innodb_redo_log_capacity`의 1/32이다. **ordinary(사용 중)와 spare(대기 중)** 두 종류로 나뉘고, spare는 `_tmp` 접미사를 단다.

용량이 모자라면 어떻게 되는지도 문서에 있다 — *"dirty pages are flushed more aggressively"*. [내구성과 복구](../basics/durability-and-recovery.md) §2-5의 "로그 공간이 꽉 차면 쓰기가 스톨된다"가 **먼저 이 단계를 거친다.** 즉 **급정지 전에 "플러시가 갑자기 공격적으로 변하는" 구간**이 있고, 대량 적재 중 응답이 튀는 현상의 흔한 정체가 그것이다.

### 2-7. 테이블스페이스와 `.ibd` — 파일이 테이블마다 하나인 이유

`innodb_file_per_table`은 **기본 `ON`**이고, 테이블 하나가 스키마 디렉터리 아래 **`테이블명.ibd` 파일 하나**가 된다.

**왜 기본값이 이쪽인가** — 문서가 든 이점 중 첫 줄이 답이다.

> *"Disk space is returned to the operating system after truncating or dropping a table. In shared tablespaces, this creates only free space within the file that cannot be returned to the OS."*

**공유 테이블스페이스에서는 지워도 OS가 공간을 못 돌려받는다.** 파일 안의 빈자리가 될 뿐이다. 이건 [MVCC와 VACUUM](../../postgresql/mvcc-and-vacuum.md) §2-4의 "VACUUM은 공간을 OS에 반환하지 않는다"와 **정확히 같은 성격의 문제**이고, InnoDB는 **파일을 쪼개는 것으로** 푼다.

```
발송 이력의 오래된 월 파티션을 DROP
  file-per-table → 그 파티션의 .ibd 가 통째로 사라진다 → 디스크가 실제로 빈다
  공유 파일     → 파일 안에 빈 구멍만 남는다          → df 로는 아무 변화가 없다
```

> [DB 인덱스](../basics/b-tree-index.md) §2-12와 [MVCC와 VACUUM](../../postgresql/mvcc-and-vacuum.md) §2-4가 각각 "행 단위로 지우지 말고 덩어리째 버려라"에 도달했는데, **file-per-table은 그 조언이 성립하기 위한 물리적 전제**다.

**공짜는 아니다.** 문서가 단점도 나열한다 — **fsync가 파일별로 흩어져 총 횟수가 늘고**(공유 파일이면 묶을 수 있었다), **파일 디스크립터를 테이블 수만큼 열어 둬야 하며**, 테이블을 DROP할 때 **버퍼 풀 전체를 스캔**하느라 버퍼 풀이 크면 수 초가 걸린다.

> **마지막 항목이 실무에서 물린다.** "임시 테이블 DROP 한 번"이 버퍼 풀 스캔을 부르고 그동안 다른 쿼리가 밀린다. 버퍼 풀이 큰 인스턴스에서 DDL을 한가한 시간에 몰아 하라는 조언의 근거 중 하나다.

### 2-8. row format — 768바이트 프리픽스의 유산

기본은 **`DYNAMIC`**(`innodb_default_row_format`)이다. **왜 `COMPACT`가 아니라 `DYNAMIC`인가**가 이 절의 전부다.

**차이는 "긴 값을 어디까지 행 안에 남기는가" 하나다.**

| | COMPACT / REDUNDANT | **DYNAMIC** |
|---|---|---|
| 긴 가변 길이 값 | **앞 768바이트를 행 안에** + 나머지는 오버플로 페이지 | **20바이트 포인터만** + 값 전체는 오버플로 페이지 |
| 인덱스 키 프리픽스 한도 | 767바이트 | **3072바이트** |
| 압축 | 없음 | 없음 (`COMPRESSED`가 담당) |

**768바이트가 왜 나쁜가.** [저장과 I/O](../basics/storage-and-io.md) §2-1의 "행이 작을수록 페이지에 많이 들어간다"가 그대로 적용된다.

```
발송 이력에 message_body TEXT 컬럼(평균 2KB)이 있을 때
  COMPACT : 행마다 768바이트가 페이지를 먹는다 → 16KB 페이지에 20행 남짓
  DYNAMIC : 행마다 20바이트                    → 같은 페이지에 훨씬 많은 행
→ SELECT id, status ... 처럼 본문을 안 읽는 쿼리의 페이지 수가 달라진다
```

> **[힙과 인덱스](../../postgresql/heap-and-index.md) §2-7의 TOAST와 목적이 같다** — 큰 값을 행 밖으로 빼서 **그 컬럼을 안 읽는 쿼리가 비용을 안 내게** 한다. 다만 **TOAST는 압축을 먼저 시도**하고, InnoDB의 `DYNAMIC`은 **압축 없이 분리만** 한다. 압축은 별도 row format인 `COMPRESSED`의 일이다.

**`COMPRESSED`는 기본이 될 수 없게 막혀 있다.** `innodb_default_row_format`에 넣으면 에러가 나고, `CREATE TABLE`/`ALTER TABLE`에서 명시해야만 쓸 수 있다. 그리고 **file-per-table 또는 general 테이블스페이스에서만** 만들 수 있다(시스템 테이블스페이스 불가).

> **"기본값으로 못 만들게 막아 뒀다"를 신호로 읽어야 한다.** 압축은 CPU와 버퍼 풀을 더 쓰는 거래이고(압축본과 압축 해제본이 버퍼 풀에 같이 올라온다), 전 테이블에 일괄로 걸 성질이 아니다. **읽기가 드물고 부피가 큰 이력 테이블 같은 특정 대상에만** 붙이는 선택지로 남겨 둔 것이다.

### 2-9. 그래서 PostgreSQL과 무엇이 다른가

| 항목 | InnoDB | PostgreSQL |
|---|---|---|
| 저장 엔진 교체 | **테이블 단위로 가능** → 보장이 테이블마다 다를 수 있다 | 고르는 개념이 없다 |
| 세컨더리 인덱스 쓰기 지연 | **체인지 버퍼** (8.4 기본 `none`) | 없음 |
| B+Tree 위 해시 캐시 | **AHI** (8.4 기본 `OFF`) | 없음 |
| torn page 대응 | **더블라이트 전용 파일**(순차 + fsync 1회) | `full_page_writes` — **WAL에 페이지 전체** |
| 옛 버전 저장소 | **undo 테이블스페이스**(별도 파일) | 힙 안 |
| 옛 버전 청소 | **purge 스레드** — 운영 업무가 아니다 | **VACUUM** — 상시 운영 업무 |
| 테이블 = 파일 | **`.ibd` 하나** (file-per-table 기본 `ON`) | 관계마다 파일 + visibility map 등 부속 |
| 큰 값 | **row format이 결정** (`DYNAMIC`=20바이트 포인터) | **TOAST**(압축 → 분리) |
| 압축 | `COMPRESSED` row format (기본 지정 불가) | TOAST의 압축 단계 |

**"청소 주체" 줄이 운영 부담의 방향을 정한다.** InnoDB는 purge가 알아서 돌아 **평소에 신경 쓸 일이 없는 대신**, 긴 트랜잭션 때문에 undo가 부푸는 문제가 **눈에 안 띈다.** PostgreSQL은 VACUUM이 상시 업무라 **번거로운 대신 계기판이 있다**([MVCC와 VACUUM](../../postgresql/mvcc-and-vacuum.md) §2-6).

**그리고 이 문서를 관통하는 한 줄이 있다.** 체인지 버퍼도 AHI도 더블라이트도 전부 *"디스크가 느리다"*에 대한 답이었는데 **8.4에서 앞의 둘만 기본값이 뒤집혔다.** 성능 최적화는 하드웨어와 함께 낡고, **정합성 장치는 안 낡는다.**

---

**다음으로 읽을 것**

- 이 구조 위에서 락이 어떻게 걸리는가 → [락과 격리 수준](lock-and-isolation.md) §2-2
- 버퍼 풀과 LRU 미드포인트 삽입 → [저장과 I/O](../basics/storage-and-io.md) §2-2~2-3
- redo·undo의 역할 분담과 torn page의 존재 이유 → [내구성과 복구](../basics/durability-and-recovery.md) §2-2, §2-7
- 큰 값을 행 밖으로 빼는 다른 방식 → [힙과 인덱스](../../postgresql/heap-and-index.md) §2-7

---

> **기준 버전**: MySQL 8.4 · PostgreSQL 17. **§2-3·§2-4의 기본값은 8.4에서 바뀐 것이라 8.0 문서·블로그와 어긋난다** — 버전을 반드시 같이 보라
> **확인한 출처**:
> - [1.3 What Is New in MySQL 8.4](https://dev.mysql.com/doc/refman/8.4/en/mysql-nutshell.html) · [8.4.0 릴리스 노트](https://dev.mysql.com/doc/relnotes/mysql/8.4/en/news-8-4-0.html) · [17.14 InnoDB 시스템 변수](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html) — **`innodb_change_buffering` `all` → `none`**, **`innodb_adaptive_hash_index` `ON` → `OFF`**(*"Before MySQL 8.4, ..."* 단서 포함), `innodb_doublewrite_files` 기본 2, `innodb_doublewrite_pages` 기본 128, `innodb_change_buffer_max_size` 기본 25
> - [17.5.2 Change Buffer](https://dev.mysql.com/doc/refman/8.4/en/innodb-change-buffer.html) · [17.5.3 Adaptive Hash Index](https://dev.mysql.com/doc/refman/8.4/en/innodb-adaptive-hash.html) — 체인지 버퍼 정의 원문과 세컨더리 인덱스가 비유니크·랜덤 순서라는 서술·시스템 테이블스페이스 거주 / AHI가 **키 프리픽스 기반·자주 쓰는 페이지에만** 생성된다는 점, 동시 조인 경합과 `LIKE`·`%`가 이득 없다는 서술, `innodb_adaptive_hash_index_parts` 기본 8, `SEMAPHORES`·`btr0sea.c` 진단법
> - [17.6.2 Doublewrite Buffer](https://dev.mysql.com/doc/refman/8.4/en/innodb-doublewrite-buffer.html) — *"large sequential chunk, with a single fsync()"* 원문, 4개 모드와 `DETECT_ONLY`의 의미, `#ib_16384_0.dblwr` 명명, Fusion-io 원자적 쓰기 시 자동 비활성
> - [17.6.3.4 Undo Tablespaces](https://dev.mysql.com/doc/refman/8.4/en/innodb-undo-tablespaces.html) · [17.6.5 Redo Log](https://dev.mysql.com/doc/refman/8.4/en/innodb-redo-log.html) — `undo_001`·`undo_002` 2개, **`innodb_undo_log_truncate` 기본 `OFF`**, `innodb_max_undo_log_size` 1 GiB, `innodb_purge_rseg_truncate_frequency` 128, 진행 중 트랜잭션이 끝나야 세그먼트가 풀린다는 서술 / redo 32개 파일·`#innodb_redo`·ordinary·spare(`_tmp`), 용량 초과 시 *"dirty pages are flushed more aggressively"*
> - [17.6.3.2 File-Per-Table Tablespaces](https://dev.mysql.com/doc/refman/8.4/en/innodb-file-per-table-tablespaces.html) · [17.10 InnoDB Row Formats](https://dev.mysql.com/doc/refman/8.4/en/innodb-row-format.html) — 기본 `ON`·`.ibd` 명명·**OS에 공간을 반환한다는 이점 원문**·단점(fsync 분산·파일 디스크립터·DROP 시 버퍼 풀 스캔) / 기본 `DYNAMIC`, **768바이트 프리픽스** vs **20바이트 포인터**, 키 프리픽스 767 vs 3072, `COMPRESSED`를 기본값으로 못 넣는다는 에러
> - [16장 Alternative Storage Engines](https://dev.mysql.com/doc/refman/8.4/en/storage-engines.html) · [17.4 InnoDB Architecture](https://dev.mysql.com/doc/refman/8.4/en/innodb-architecture.html) — 플러거블 아키텍처와 테이블 단위 엔진 지정 원문, MyISAM 테이블 락, MEMORY의 *"use cases are decreasing"*, 인메모리 4종·온디스크 구조 목록
> **미확인**:
> - **§2-3·§2-4에서 기본값이 뒤집힌 "이유"** — 릴리스 노트·매뉴얼 어디에도 근거가 없다. 본문의 "랜덤 I/O가 싸져서"는 **내 추정이며 그렇게 표시했다**. 체인지 버퍼 머지 지연이 크래시 복구를 늘린다는 이야기도 근거를 못 찾아 단정하지 않았다
> - [AHI 페이지](https://dev.mysql.com/doc/refman/8.4/en/innodb-adaptive-hash.html) 본문과 파라미터 표·릴리스 노트가 **기본값에 대해 다르게 읽힌다.** 후자(=`OFF`)를 채택했으나 인스턴스에서 `SELECT @@innodb_adaptive_hash_index`로 재확인 필요
> - `innodb_redo_log_capacity` 기본값(문서가 수치를 안 밝혀 본문에서 뺐다) · `innodb_file_per_table` 기본 `ON`(본문 서술로만 확인, 파라미터 표 미대조) · PostgreSQL 12 이상의 table access method 확장점(§2-9의 "고르는 개념이 없다"는 **사용자 관점 서술**이다)
> **미작성**: 버퍼 풀 인스턴스 분할과 8.4의 새 기본 계산식 · 온라인 DDL의 내부 동작(INPLACE/COPY) · 전문 검색(FULLTEXT) 인덱스 · 파티셔닝의 InnoDB 구현 · `innodb_flush_neighbors` 등 플러시 튜닝
