# 청소가 왜 필수 업무인가

## 30초 요약

- PostgreSQL은 갱신 전 버전을 별도 영역이 아니라 **힙 안에 그대로 둔다**. 그래서 UPDATE만 해도 테이블이 커진다
- 공식 문서가 내건 약속은 *"reading never blocks writing and writing never blocks reading"*이고, **dead tuple은 그 약속의 청구서**다
- VACUUM은 청소부가 아니라 다섯 가지 일을 한다. 그중 하나가 **XID wraparound 방지**인데, 이건 성능이 아니라 데이터 손실 방지다
- **VACUUM은 OS에 공간을 돌려주지 않는다.** 재사용 가능하게 표시할 뿐이다. 돌려받으려면 `VACUUM FULL`인데 그건 테이블을 통째로 잠근다
- 방치하면 DB가 **쓰기를 전부 거부한다.** 느려지는 게 아니라 멈춘다. 남은 트랜잭션 3백만에서 `ERROR: database is not accepting commands`가 뜬다

---

## 원리 — 왜 그런가

> [저장과 I/O](../database/basics/storage-and-io.md) §2-6 비교표의 "갱신 전 버전 보관 — 힙 안에 옛 버전을 그대로 둠" 줄이 이 문서 전체로 펼쳐진다.

### 2-1. 왜 옛 버전을 힙 안에 두나

MVCC를 구현하려면 한 행의 여러 버전이 동시에 존재해야 한다([트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-3). 그 버전들을 어디에 둘 것인가에서 엔진이 갈린다.

- **InnoDB**: 옛 버전을 undo 세그먼트(별도 영역)로 옮긴다
- **PostgreSQL**: 옛 버전을 있던 자리에 그대로 두고, 새 버전을 새 자리에 쓴다

PostgreSQL 방식에서 UPDATE는 사실상 **삭제 표시 + 삽입**이다.

```
UPDATE send_history SET status='SENT' WHERE id=1;

[ 옛 튜플 ] xmin=100  xmax=200   ← "트랜잭션 200이 이걸 무효화했다"
[ 새 튜플 ] xmin=200  xmax=null  ← 새 버전
```

각 튜플이 **xmin**(나를 만든 트랜잭션)과 **xmax**(나를 무효화한 트랜잭션)를 들고 있다. 읽는 쪽은 자기 스냅샷과 이 둘을 비교해 "내가 볼 수 있는 버전인가"를 판정한다.

이 구조의 이점은 둘이다.
- 읽기가 다른 영역(undo)을 따라갈 필요가 없다. 힙만 보면 된다
- **롤백이 거의 공짜다.** 되돌릴 게 없고 "이 트랜잭션은 실패했다"고만 표시하면 된다([내구성과 복구](../database/basics/durability-and-recovery.md) §2-2)

공식 문서가 내건 약속도 이것이다. *"reading never blocks writing and writing never blocks reading."*

### 2-2. dead tuple과 블로트라는 대가

아무도 볼 수 없게 된 옛 튜플이 힙에 그대로 남는다. 이걸 dead tuple이라 하고, 쌓인 상태를 **블로트**라 한다.

```
1천만 행 테이블에서 전 행을 한 번 UPDATE
→ 살아 있는 튜플 1천만 + dead tuple 1천만
→ 테이블 물리 크기가 두 배
```

**블로트의 진짜 비용은 디스크가 아니다.**

- 읽기가 느려진다. 페이지를 읽었는데 절반이 쓰레기면 [저장과 I/O](../database/basics/storage-and-io.md) §2-1의 "한 번 갈 때 주변까지 들고 오자"는 도박이 실패한다. 같은 행 수를 읽는 데 더 많은 페이지가 필요하다
- 버퍼 풀이 쓰레기로 찬다. 캐시 효율이 떨어진다
- 인덱스도 부푼다. 각 버전이 인덱스 엔트리를 갖는다

> ES의 세그먼트 병합과 구조가 같다([Elasticsearch](../datastore/elasticsearch.md) §2-4). 옛것을 그 자리에서 못 고치니 표시만 남기고, 쌓이면 별도 정리 작업이 돈다. **불변에 가까운 저장을 택하면 청소 담당이 반드시 따라온다.**

### 2-3. VACUUM이 실제로 하는 다섯 가지

"VACUUM = 쓰레기 청소"로 이해하면 절반만 맞다. 공식 문서가 드는 이유는 다섯이다.

| 하는 일 | 왜 |
|---|---|
| **dead tuple 제거** | §2-2의 블로트 회수 |
| **공간을 재사용 가능으로 표시** | 다음 INSERT가 그 자리를 쓴다 |
| **플래너 통계 갱신**(`ANALYZE`) | 옵티마이저 추정의 입력 → [쿼리 실행](../database/basics/query-execution.md) §2-4 |
| **visibility map 갱신** | Index-Only Scan이 가능해진다 |
| **XID wraparound 방지** | §2-5. 이게 진짜 이유다 |

네 번째가 성능에서 중요하다. visibility map은 "이 페이지의 튜플은 전부 모두에게 보인다"를 기록한 비트맵이다. 이게 있으면 인덱스만 읽고 힙에 안 가도 된다. PostgreSQL은 인덱스에 가시성 정보가 없어서 원래는 힙을 확인해야 하는데, visibility map이 그걸 건너뛰게 해 준다. **VACUUM을 안 돌리면 커버링 인덱스를 만들어도 Index-Only Scan이 안 나온다.**

다섯 번째가 왜 필수인지는 §2-5에서 본다.

### 2-4. VACUUM은 공간을 돌려주지 않는다

가장 자주 오해되는 지점이다.

| | VACUUM | VACUUM FULL |
|---|---|---|
| 락 | `SHARE UPDATE EXCLUSIVE` — **읽기·쓰기 계속 가능** | `ACCESS EXCLUSIVE` — **전부 차단** |
| 공간 | **OS에 반환 안 함**(테이블 끝의 완전히 빈 페이지만 예외) | **완전히 압축해 OS에 반환** |
| 속도 | 빠름 | 매우 느림 |
| 추가 디스크 | 불필요 | **테이블 크기만큼 임시 공간 필요** |

**"VACUUM 돌렸는데 디스크가 안 줄어요"의 답이 이것이다.** VACUUM은 공간을 DB 내부에서 재사용 가능하게 만들 뿐이다. 다음 INSERT가 그 자리를 쓰므로 테이블이 더 안 커지는 것이 이득이지, 디스크가 반환되는 게 아니다.

그럼 VACUUM FULL을 쓰면 되지 않나. 공식 문서가 명시적으로 말린다.

> *"Generally, therefore, administrators should strive to use standard VACUUM and avoid VACUUM FULL."*

이유가 실무적이다. `ACCESS EXCLUSIVE` 락은 **그 테이블에 대한 조회조차 막는다.** 1억 행 테이블에 걸면 그동안 서비스가 그 테이블을 못 쓴다. 게다가 원본만큼의 디스크가 더 필요하다. 공간이 부족해서 돌리려는데 공간이 더 필요한 역설이 생긴다.

> 디스크 확보가 목적이라면 파티션을 `DROP`하는 쪽이 압도적으로 싸다([DB 인덱스](../database/basics/b-tree-index.md) §2-12). **"행 단위로 지우지 말고 덩어리째 버려라"가 여기서도 답이다.**

### 2-5. XID wraparound는 느려지는 게 아니라 멈추는 문제다

여기가 PostgreSQL 운영에서 가장 위험한 항목이다.

**트랜잭션 ID가 32비트다.** 약 43억을 쓰면 0으로 돌아간다. 그런데 §2-1에서 봤듯 가시성 판정은 "내 XID와 튜플의 xmin/xmax 중 어느 쪽이 오래됐나"를 따지는 비교다.

```
XID가 한 바퀴 돌면 → 아주 오래된 튜플의 xmin이 "미래"로 보인다
                  → 아직 안 만들어진 행으로 판정된다
                  → 그 데이터가 통째로 사라진다
```

**성능 저하가 아니라 데이터 소실**이다.

**막는 방법은 freeze다.** VACUUM이 충분히 오래된 튜플을 특수 값으로 표시한다. 공식 문서 표현으로는 `FrozenTransactionId`이고, *"정상 XID 비교 규칙을 따르지 않고 항상 모든 정상 XID보다 오래된 것으로 간주"*된다. 비교 대상에서 아예 빼 버리는 것이라 한 바퀴 돌든 말든 무관해진다.

**그래서 VACUUM은 선택이 아니다.** dead tuple이 하나도 없어도, 읽기 전용 테이블이라도, freeze를 위해 주기적으로 돌아야 한다. `autovacuum_freeze_max_age`(기본 2억)를 넘긴 테이블은 autovacuum이 꺼져 있어도 강제로 처리된다.

**한계에 다가가면 DB가 이렇게 반응한다.**

```
남은 트랜잭션 4천만  → WARNING: database "mydb" must be vacuumed within 39985967 transactions
남은 트랜잭션 3백만  → ERROR: database is not accepting commands that assign
                              new transaction IDs to avoid wraparound data loss
```

ERROR 상태에서 무엇이 되고 안 되나.
- ✅ 읽기 전용 트랜잭션: 새로 시작 가능
- ✅ VACUUM: 여전히 실행 가능. **탈출구는 남겨 둔다**
- ❌ INSERT · UPDATE · DELETE · TRUNCATE: 전부 실패

**서비스가 읽기 전용으로 강제 전환된다.** 그리고 이 상태에서 `VACUUM FULL`을 돌리면 안 된다. 그것도 XID를 소비하기 때문이다. 공식 문서의 복구 절차도 일반 `VACUUM`을 지시한다.

### 2-6. autovacuum이 못 따라가는 경우

autovacuum은 기본으로 켜져 있다. 그런데 **켜져 있는데도 블로트가 쌓이는** 상황이 있고, 원인이 두 종류다.

**(A) 임계값을 못 넘긴다**

```
트리거 조건 ≈  dead tuple 수 > 50 + 0.2 × 테이블 행 수
                              (threshold)  (scale_factor)
```

**`scale_factor`가 비율이라는 게 함정이다.** 1억 행 테이블은 dead tuple이 2천만 개 쌓여야 autovacuum이 돈다. 그때는 이미 블로트가 심각하다. 큰 테이블일수록 임계에 늦게 도달하므로, 대형 테이블은 테이블 단위로 `scale_factor`를 낮추는 게 정석이다.

**(B) 청소하려 해도 못 지운다.** 이쪽이 더 자주 사고를 만든다

VACUUM은 **"아직 누군가 볼지도 모르는 튜플"은 못 지운다.** 그래서 가장 오래된 살아 있는 스냅샷이 청소 가능 경계를 정한다. 그 경계를 뒤로 붙잡는 것이 셋이다.

| 붙잡는 것 | 무슨 일이 | 확인 |
|---|---|---|
| **긴 트랜잭션** | 몇 시간 열린 트랜잭션 하나가 **전체 DB의 청소를 막는다** | `pg_stat_activity`의 `age(backend_xmin)` |
| **오래된 복제 슬롯** | 죽은 리플리카의 슬롯이 xmin을 붙잡는다 | `pg_replication_slots` |
| **prepared transaction** | 커밋도 롤백도 안 된 2PC 잔재가 영원히 붙잡는다 | `pg_prepared_xacts` |

**세 항목은 공식 문서의 wraparound 복구 절차 순서와 같다.** VACUUM을 돌리기 전에 이것들을 먼저 치우라고 지시한다. 안 그러면 VACUUM을 돌려도 아무것도 못 지운다.

> [복제와 파티셔닝](../database/basics/replication-and-partitioning.md) §2-8에서 "죽은 리플리카를 방치하면 프라이머리 디스크가 찬다"고 했는데 실은 더 나쁘다. 복제 슬롯은 WAL만 붙잡는 게 아니라 **xmin도 붙잡는다.** 디스크가 차는 동시에 VACUUM이 마비되고 wraparound 시계가 돌기 시작한다. 방치된 슬롯 하나가 두 종류의 사고를 동시에 만든다.

**"긴 트랜잭션이 나쁘다"의 이유가 여기서 하나 더 늘어난다.** [트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-3은 InnoDB에서 undo를 못 지운다고 했는데, PostgreSQL에서는 전체 DB의 VACUUM이 막힌다. 영향 범위가 자기 테이블이 아니라 클러스터 전체다.

### 2-7. 옛 버전을 어디에 두느냐가 갈라 놓은 것

| 항목 | InnoDB | PostgreSQL |
|---|---|---|
| 옛 버전 위치 | undo 세그먼트(별도) | **힙 안에** |
| 청소 주체 | purge 스레드 | **VACUUM / autovacuum** |
| 롤백 비용 | undo 되감기 → **큰 롤백이 비싸다** | 표시만 하면 끝 → **거의 공짜** |
| UPDATE 비용 | 인덱스는 바뀐 컬럼만 | 행 이동 시 **모든 인덱스 갱신**(HOT으로 회피) |
| 테이블 부풀기 | 상대적으로 적음 | **UPDATE만 해도 커진다** |
| 운영 필수 작업 | 특별히 없음 | **VACUUM 관리가 상시 업무** |
| 방치 시 최악 | undo 영역 증가 | **쓰기 전면 거부**(wraparound) |

마지막 줄은 성격이 완전히 다르다. InnoDB의 방치는 성능·용량 문제로 나타나지만, **PostgreSQL의 방치는 서비스 정지로 나타난다.** "PostgreSQL은 운영 손이 더 간다"는 말의 실체가 대부분 이 한 줄이다.

반대로 이득도 분명하다. 롤백이 싸고, 긴 트랜잭션이 undo 영역을 압박하지 않으며, 읽기 경로가 단순하다. **거래이지 우열이 아니다.**

---

**다음으로 읽을 것**

- 힙 구조와 HOT update, 인덱스 종류 → `postgresql/heap-and-index.md` (예정)
- 옛 버전을 undo에 두는 쪽 → [내구성과 복구](../database/basics/durability-and-recovery.md) §2-2
- MVCC 자체의 원리 → [트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-3
- 복제 슬롯이 만드는 다른 사고 → [복제와 파티셔닝](../database/basics/replication-and-partitioning.md) §2-8

---

> **기준 버전**: PostgreSQL 17 기준. §2-6의 autovacuum 트리거 공식은 버전에 따라 항이 추가된다(18에서 INSERT 기반 조건 추가)
> **확인한 출처**:
> - [PostgreSQL 24.1 Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html) — VACUUM의 다섯 가지 역할, VACUUM/VACUUM FULL의 락 수준(`SHARE UPDATE EXCLUSIVE` vs `ACCESS EXCLUSIVE`)과 공간 반환 여부, *"strive to use standard VACUUM and avoid VACUUM FULL"*, XID 32비트와 `FrozenTransactionId`의 정의, `autovacuum_freeze_max_age` 기본 2억, **4천만 WARNING / 3백만 ERROR 문구와 그 상태의 허용 동작**, 복구 절차 순서(prepared transaction → 긴 트랜잭션 → 복제 슬롯 → VACUUM), autovacuum 임계 공식과 기본값(threshold 50, scale_factor 0.2)
> - [PostgreSQL 13.1 MVCC Introduction](https://www.postgresql.org/docs/current/mvcc-intro.html) — *"reading never blocks writing and writing never blocks reading"*
> **미확인**: xmin/xmax를 쓰는 가시성 판정의 정확한 규칙 · HOT update의 동작 조건 · visibility map의 구조 — 다음 편에서 대조 예정
> **미작성**: `vacuum_freeze_min_age` 등 세부 튜닝 파라미터 · aggressive vacuum · 병렬 VACUUM
