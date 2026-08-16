# 인덱스와 옵티마이저 — MySQL이 따로 쥐고 있는 손잡이들

## 30초 요약

- **한국어 전문 검색은 `ngram` 파서 없이는 성립하지 않는다.** 기본 파서가 **공백으로 단어를 자르기** 때문이다 — 공백을 안 쓰는 언어에서는 토큰이 아예 안 나온다
- **프리픽스 인덱스는 "짧게 만드는 기능"이 아니라 "길어서 못 걸 때 어쩔 수 없이 쓰는 기능"**이다. 커버링·유니크를 동시에 잃는다
- **인비저블 인덱스는 "지우기 전에 지운 척"이다.** 옵티마이저만 안 보고 **쓰기는 그대로 갱신된다** — 롤백이 즉시라는 게 값어치
- **히스토그램은 인덱스 없는 컬럼용 통계**다. 그리고 **자동 갱신되지 않는다** — 만들어 두고 잊으면 낡은 통계가 오히려 플랜을 망친다
- ICP · MRR · 인덱스 머지는 **전부 "세컨더리 인덱스 → 테이블 재방문"이라는 하나의 비용을 서로 다른 각도에서 깎는 장치**다

---

## 원리 — 왜 그런가

> 인덱스 자료구조 일반론과 `EXPLAIN` 읽는 법은 [DB 인덱스](../basics/b-tree-index.md), 조인 알고리즘과 옵티마이저의 추정 실패는 [쿼리 실행](../basics/query-execution.md)에 있다. 이 문서는 **그 원리 위에 MySQL이 따로 얹어 둔 손잡이들**만 다룬다.

### 2-1. 프리픽스 인덱스 — 자르면 무엇을 잃나

`INDEX(col(10))`은 **컬럼 값의 앞부분만 인덱스에 넣는 것**이다. 공식 문서가 두 가지를 못 박는다.

- **`BLOB`·`TEXT` 컬럼에는 프리픽스가 필수**다(선택이 아니다). `CHAR`·`VARCHAR`·`BINARY`·`VARBINARY`는 선택
- 상한이 있다 — **InnoDB `DYNAMIC`/`COMPRESSED` 행 포맷에서 3072바이트, `REDUNDANT`/`COMPACT`에서는 767바이트** (이 두 숫자가 왜 갈리는지는 [InnoDB 내부 구조](innodb-internals.md) §2-8)
- **길이 단위가 타입마다 다르다.** 비이진 문자열에서 `(10)`은 **10글자**, 이진 문자열에서는 **10바이트**다. utf8mb4는 글자당 최대 4바이트라 3072바이트 상한이 **약 768자**로 환산된다

**왜 필수 기능인가** — 발송 본문(`TEXT`)이나 긴 URL을 인덱스에 그냥 못 넣기 때문이다. 그런데 자르는 순간 세 가지를 같이 잃는다.

```
INDEX(message_body(20))
  ① 커버링 불가        : 잘린 값이라 인덱스만 보고 결과를 못 만든다 → 항상 테이블 재방문
  ② 유니크 불가        : 앞 20자가 같은 서로 다른 값이 존재한다
  ③ ORDER BY 불가      : 앞 20자 순서 ≠ 전체 값 순서
```

**②가 특히 위험하다.** 발송 본문이나 외부 키를 `UNIQUE(col(N))`으로 잡으면 **앞 N자만 같아도 중복으로 거부된다** — 정상 데이터가 들어가지 않는 사고가 된다.

> **자를 길이는 카디널리티로 정한다.** `SELECT COUNT(DISTINCT LEFT(col,N)) / COUNT(*)`를 N을 늘려 가며 재고, **비율이 더 이상 안 오르는 지점**에서 멈춘다. "일단 10"은 근거가 아니다.

### 2-2. Full-Text + ngram — 한국어에서 왜 선택이 아닌가

`FULLTEXT` 인덱스는 InnoDB·MyISAM의 `CHAR`·`VARCHAR`·`TEXT` 컬럼에만 걸 수 있고, 검색 모드가 셋이다(자연어 · `IN BOOLEAN MODE` · 질의 확장).

**문제는 토크나이저다.** 공식 문서의 서술이 그대로 이유다.

> *"The built-in MySQL full-text parser uses the white space between words as a delimiter to determine where words begin and end, which is a limitation when working with ideographic languages that do not use word delimiters. To address this limitation, MySQL provides an ngram full-text parser that supports Chinese, Japanese, and Korean (CJK)."*

```
"광고 대량발송 안내"  기본 파서 → '광고' '대량발송' '안내'
                       "발송"만으로는 못 찾고, 조사가 붙은 "발송을"은 다른 토큰이 된다
ngram(n=2)          → '광고' '대량' '량발' '발송' '송안' '안내' ...  → 부분 문자열 검색이 성립
```

**ngram은 형태소를 이해하는 게 아니라 n글자씩 기계적으로 자른다.** 그래서 조사·어미 문제를 우회한다. 켜는 법은 `FULLTEXT(col) WITH PARSER ngram`이고, **`ngram_token_size` 기본값은 2**(범위 1~10, **서버 기동 시 설정**)다. **토큰 크기보다 짧은 검색어는 매칭되지 않는다** — n=2면 한 글자 검색이 안 된다.

> **트레이드오프가 크다.** n=2면 인덱스 항목 수가 대략 **원문 글자 수만큼** 생긴다. 발송 본문 1억 건에 이걸 걸면 인덱스가 본문보다 커질 수 있다. **[Elasticsearch](../../datastore/elasticsearch.md)로 넘길 판단선이 여기서 결정된다** — "RDB 안에서 되긴 되는데 비용이 어떤가"의 답이 ngram의 팽창률이다.

### 2-3. 함수 기반 인덱스와 내림차순 인덱스 — 정렬 기준을 바꾸는 두 방법

[DB 인덱스](../basics/b-tree-index.md) §2-6의 결론은 "인덱스는 컬럼 **원본 값** 기준으로 정렬돼 있다"였다. **그 정렬 기준 자체를 바꾸는 장치가 둘 있다.**

**(1) 함수 기반 인덱스 (8.0.13+)** — *"MySQL 8.0.13 and higher supports functional key parts that index expression values rather than column or column prefix values."* 예: `CREATE INDEX idx ON send_history ((DATE(sent_at)));`

**구현이 정체를 설명한다** — 함수 키 파트는 **숨은 가상 생성 컬럼(hidden virtual generated column)으로 구현된다.** 즉 "함수를 이해하는 인덱스"가 아니라 **"함수 결과를 담은 보이지 않는 컬럼에 건 평범한 인덱스"**다. 그래서 제약이 따라온다 — 표현식은 **괄호 필수**(`INDEX((col1))`처럼 컬럼 하나만은 불가), **컬럼 프리픽스 참조 불가**, 외래 키 불가, `UNIQUE`는 되지만 **`PRIMARY KEY`·`SPATIAL`·`FULLTEXT`는 불가**.

> ⚠️ **범위 조건으로 이항할 수 있으면 그게 먼저다.** `WHERE DATE(sent_at) = '2026-08-13'`은 `WHERE sent_at >= '2026-08-13' AND sent_at < '2026-08-14'`로 바꾸면 **인덱스를 새로 만들지 않고** 끝난다. 함수 기반 인덱스는 **이항이 안 되는 경우**(대소문자 무시 조회 등)의 도구다.

**(2) 내림차순 인덱스 (8.0)** — *"`DESC` in an index definition is no longer ignored but causes storage of key values in descending order. Previously, indexes could be scanned in reverse order but at a performance penalty."*

**단일 컬럼 `ORDER BY x DESC`에는 쓸 이유가 없다.** 역방향 스캔이 원래 됐다. 값어치는 **정렬 방향이 섞일 때** 나온다.

```sql
-- 캠페인 목록: 최신 우선, 같은 시각이면 우선순위 낮은 번호부터
ORDER BY created_at DESC, priority ASC
→ INDEX(created_at DESC, priority ASC) 가 있어야 filesort 없이 끝난다
```

- **InnoDB의 `BTREE` 인덱스에서만** 지원된다. `HASH`·`FULLTEXT`·`SPATIAL`은 불가
- 세컨더리 인덱스에 내림차순 키 컬럼이 있으면 **체인지 버퍼링이 적용되지 않는다.** 다만 8.4는 체인지 버퍼 자체가 기본 비활성이라 이 대가가 사실상 사라졌다(→ [InnoDB 내부 구조](innodb-internals.md) §2-3)

### 2-4. 인비저블 인덱스 — 지우기 전에 지운 척해 보기

**"이 인덱스 안 쓰는 것 같은데 지워도 되나"**에 답하는 장치다.

> *"Index visibility does not affect index maintenance. For example, an index continues to be updated per changes to table rows, and a unique index prevents insertion of duplicates into a column, regardless of whether the index is visible or invisible."*

**이 한 줄이 전부다** — 옵티마이저만 안 보고, **쓰기 비용과 유니크 제약은 그대로 살아 있다.**

```
ALTER TABLE send_history ALTER INDEX idx_receiver INVISIBLE;   ← 즉시, in-place
  → 며칠 지켜본다 (느려진 쿼리가 있나)
  → 문제 없으면 DROP, 문제 있으면 VISIBLE 로 즉시 복구
```

**값어치는 롤백 속도**다. 공식 문서 표현대로 *"Dropping and re-adding an index can be expensive for a large table, whereas making it invisible and visible are fast, in-place operations."* 1억 건 테이블에서 인덱스를 지웠다가 되살리면 수 시간인데, 가시성 전환은 즉시다.

- **PK는 인비저블로 못 만든다.** 명시적 PK뿐 아니라 **`NOT NULL` 컬럼의 첫 `UNIQUE` 인덱스가 만드는 암묵적 PK도** 마찬가지다
- 검증할 때는 `optimizer_switch`의 **`use_invisible_indexes`**를 세션 단위로 `on`으로 켜면, 인비저블 상태 그대로 **그 세션에서만** 플랜을 확인할 수 있다(기본값 `off`)

### 2-5. 멀티밸류 인덱스 — JSON 배열 하나에 인덱스 여러 개

**8.0.17부터** InnoDB가 지원한다. 보통 인덱스는 행 하나당 인덱스 레코드 하나인데, 이건 **행 하나당 배열 원소 수만큼** 만든다(N:1).

```sql
-- 캠페인의 대상 태그가 JSON 배열이라면
INDEX idx_tags ((CAST(campaign->'$.tags' AS UNSIGNED ARRAY)))
SELECT * FROM campaign WHERE 1004 MEMBER OF(campaign->'$.tags');
```

옵티마이저가 쓰는 함수는 **`MEMBER OF()` · `JSON_CONTAINS()` · `JSON_OVERLAPS()`** 셋뿐이다. 그 외 조건에는 안 탄다.

**제약이 많고, 그 제약이 곧 "정규화 테이블을 이기지 못하는 이유"다.**

- **정렬 개념이 없다** → PK 불가, `ASC`/`DESC` 불가, **범위 스캔 불가**. **커버링도 될 수 없다** → 항상 테이블 재방문
- **온라인 생성이 안 된다** — `ALGORITHM=COPY`가 필요하다(→ [복제와 운영](replication-and-ops.md) §2-5의 그 비용). 문자셋도 `binary` 또는 **`utf8mb4_0900_as_cs`**로 제한된다

> **판단**: 태그·수신 그룹처럼 **포함 여부만 묻는** 조회면 값어치가 있다. 범위·정렬·조인이 필요해지는 순간 **별도 테이블로 정규화하는 쪽이 이긴다.** JSON 컬럼은 "스키마를 안 정해도 되는 것"이지 "인덱스를 다 쓸 수 있는 것"이 아니다.

### 2-6. 히스토그램 — 인덱스 없는 컬럼의 분포

[쿼리 실행](../basics/query-execution.md) §2-4에서 옵티마이저의 오추정을 봤다. 히스토그램은 그중 **"값 분포가 균등하다고 가정해서 틀리는"** 경우를 겨냥한다.

```sql
ANALYZE TABLE send_history UPDATE HISTOGRAM ON send_status WITH 16 BUCKETS;
ANALYZE TABLE send_history DROP HISTOGRAM ON send_status;
```

버킷 수는 **1~1024**이고 **생략하면 100**이다. 고유값 수가 버킷 수보다 **작거나 같으면 싱글턴**(버킷 하나 = 값 하나), **크면 등고**(버킷 하나 = 값 구간)가 만들어진다.

**`send_status`가 정확히 이 대상이다.** `SUCCESS` 99%, `FAIL` 0.9%, `PENDING` 0.1%처럼 **극단적으로 치우친 컬럼**에서, 히스토그램이 없으면 옵티마이저는 "3종류니까 1/3"로 추정한다.

**그런데 두 가지를 반드시 같이 알아야 한다.**

**(1) 자동 갱신되지 않는다.** *"A histogram is created or updated only on demand, so it adds no overhead when table data is modified. On the other hand, the statistics become progressively more out of date when table modifications occur, until the next time they are updated."* → **만들었으면 갱신도 운영 항목이 된다.** (8.4의 `ANALYZE TABLE` 문법에는 `{MANUAL | AUTO} UPDATE` 절이 있다.)

**(2) 인덱스가 있으면 대개 무용지물이다.** *"The optimizer prefers range optimizer row estimates to those obtained from histogram statistics. If the optimizer determines that the range optimizer applies, it does not use histogram statistics."* → **히스토그램은 "인덱스를 걸 만큼은 아닌 컬럼"의 도구**다. 인덱스의 대체재가 아니다.

> 생성 비용도 공짜가 아니다 — **분석 중 테이블에 읽기 락**이 걸리고, `histogram_generation_max_mem_size`를 넘으면 전수 대신 **샘플링**으로 떨어진다. 그리고 [쿼리 실행](../basics/query-execution.md) §2-4(2)의 **컬럼 간 상관관계는 여전히 못 푼다** — MySQL 히스토그램은 단일 컬럼만 다룬다.

### 2-7. ICP · MRR · 인덱스 머지 — 재방문 비용을 깎는 세 각도

셋 다 [DB 인덱스](../basics/b-tree-index.md) §2-3(3)의 **북마크 룩업**을 겨냥한다. 겨냥하는 각도가 다르다.

**(1) ICP — 재방문 *횟수*를 줄인다.** 인덱스에 있는 컬럼으로 판정 가능한 `WHERE` 조각을 **스토리지 엔진까지 내려보낸다.**

```
INDEX(campaign_id, receiver_phone)
WHERE campaign_id = 7 AND receiver_phone LIKE '%1234' AND memo LIKE '%반품%'

ICP 없음 : campaign_id=7 인 100만 건을 전부 테이블에서 읽어 온 뒤 서버가 판정
ICP 있음 : 인덱스 튜플에서 receiver_phone 조건을 먼저 걸러 → 통과한 것만 테이블 읽기
```

- 적용 대상은 **`range` · `ref` · `eq_ref` · `ref_or_null`** 접근 방식, 그리고 **InnoDB에서는 세컨더리 인덱스에만** 쓰인다
- `EXPLAIN`의 `Extra`에 **`Using index condition`**. [DB 인덱스](../basics/b-tree-index.md) §2-4의 `Using index`(커버링)와 **다른 것**이다 — ICP는 테이블을 읽긴 읽는다
- 서브쿼리·저장 함수를 참조하는 조건은 내려보내지 못한다

**(2) MRR — 재방문 *순서*를 바꾼다.** *"MRR enables data rows to be accessed sequentially rather than in random order."* 인덱스를 먼저 훑어 키를 모으고, **PK 순서로 정렬한 뒤** 테이블을 읽는다. 랜덤 I/O를 순차에 가깝게 만드는 것이라 **버퍼 풀에 안 올라간 큰 테이블**에서 효과가 크고, **커버링이면 효과가 0**이다(재방문이 없으니까). `Extra`에 `Using MRR`, 버퍼 크기는 `read_rnd_buffer_size`.

**(3) 인덱스 머지 — 인덱스를 여러 개 동시에 쓴다.** 한 테이블에 대한 여러 범위 스캔 결과를 합친다. 알고리즘이 셋 — **intersection**(`AND`) · **union**(`OR`) · **sort-union**(`OR`인데 union이 안 될 때, **행 ID를 전부 모아 정렬한 뒤** 반환). `type: index_merge`, `Extra`에 `Using intersect(...)` / `union(...)` / `sort_union(...)`.

> ⚠️ **인덱스 머지는 대개 "복합 인덱스가 없다"는 신고다.** `WHERE campaign_id = 7 AND send_status = 'FAIL'`에 intersection이 뜨면, 답은 머지를 잘 쓰는 게 아니라 **`INDEX(campaign_id, send_status)`를 만드는 것**이다. 그리고 **한 테이블 안에서만 동작하고, 전문 검색 인덱스에는 적용되지 않는다.**

### 2-8. Hash Join과 힌트 — `BNL` 힌트가 해시 조인을 켜는 이유

[쿼리 실행](../basics/query-execution.md) §2-2에서 알고리즘 자체는 봤다. **MySQL에서 중요한 건 도입 시점과 그것이 남긴 흔적**이다.

> *"Beginning with MySQL 8.0.18, MySQL employs a hash join for any query for which each join has an equi-join condition, and in which there are no indexes that can be applied to any join conditions"*
>
> *"Beginning with MySQL 8.0.20, support for block nested loop is removed, and the server employs a hash join wherever a block nested loop would have been used previously."*

```
~ 8.0.17 : Block Nested Loop 만  → 인덱스 없는 큰 조인은 사실상 불가
8.0.18   : 등치 조인 + 적용 가능한 인덱스 없음 → 해시 조인
8.0.20   : BNL 제거. 비등치·아우터·세미·안티 조인까지 해시 조인
```

**힌트 이름이 그 역사의 화석이다.** 해시 조인을 켜고 끄는 힌트가 `HASH_JOIN`이 아니라 **`BNL` / `NO_BNL`**이고, `optimizer_switch`의 플래그도 여전히 `block_nested_loop`다. **없어진 알고리즘의 이름이 손잡이로 남아 있다** — 문서를 안 보면 절대 못 맞히는 부분이다.

**메모리 한계도 그대로 물려받았다.** 해시 조인은 `join_buffer_size`를 넘을 수 없고, 넘으면 **디스크 파일로 떨어진다.** 이때 파일 수가 `open_files_limit`를 넘으면 **조인이 실패할 수도 있다** — 느려지는 게 아니라 에러다.

**힌트는 `/*+ ... */` 주석으로 `SELECT`·`UPDATE`·`INSERT`·`REPLACE`·`DELETE`의 첫 키워드 뒤에 붙인다.**

| 힌트 | 무엇을 강제하나 |
|---|---|
| `JOIN_FIXED_ORDER` | `FROM` 절에 적힌 순서로 조인. *"This is the same as specifying `SELECT STRAIGHT_JOIN`"* |
| `JOIN_ORDER` / `JOIN_PREFIX` / `JOIN_SUFFIX` | 조인 순서를 부분적으로 고정 |
| `INDEX` / `NO_INDEX`, `JOIN_INDEX` · `GROUP_INDEX` · `ORDER_INDEX` | 인덱스 사용 강제·금지 (용도별로 나뉜다) |
| `BNL` / `NO_BNL` | **해시 조인** 켜기·끄기 |
| `MRR` / `NO_MRR`, `NO_ICP`, `BKA` / `NO_BKA` | §2-7의 최적화 개별 제어 |

> **둘이 같은 일을 하지만 `STRAIGHT_JOIN`은 SQL 문법이고 힌트는 주석**이다. 주석은 다른 엔진에서 그냥 무시되므로, 새로 쓴다면 힌트 쪽이 낫다.
>
> ⚠️ [쿼리 실행](../basics/query-execution.md) §2-3의 경고가 그대로 적용된다 — **힌트는 통계가 개선되면 오히려 방해가 된다.** 먼저 `ANALYZE TABLE`과 히스토그램(§2-6)을 의심하고, 힌트는 마지막에 쓴다.

### 2-9. 그래서 갈라지는 것들

| 항목 | MySQL 8.4 | PostgreSQL 17 |
|---|---|---|
| 프리픽스 인덱스 | **`col(N)`** — `TEXT`/`BLOB`엔 필수 | 없음(대신 **표현식 인덱스**로 `left(col,N)`) |
| CJK 전문 검색 | **`WITH PARSER ngram`** (기본 파서는 공백 기준) | `pg_trgm`(3-gram) · `tsvector` + 외부 사전 |
| 함수 기반 인덱스 | 8.0.13+, **숨은 가상 컬럼으로 구현** | 오래전부터 **표현식 인덱스**로 지원 |
| 인덱스 숨기기 | **인비저블 인덱스**(즉시 전환) | 대응물 없음(드롭을 트랜잭션 안에서 롤백) |
| JSON 배열 인덱스 | **멀티밸류 인덱스**(8.0.17+, 제약 많음) | **GIN** — 범용적이고 제약이 적다 |
| 분포 통계 | **히스토그램**(수동 갱신, 단일 컬럼) | autovacuum이 갱신 + **`CREATE STATISTICS`**(다중 컬럼) |
| 재방문 최적화 | **ICP · MRR** | **Bitmap Heap Scan** 하나로 통합(→ [힙과 인덱스](../../postgresql/heap-and-index.md) §2-6) |
| 여러 인덱스 동시 사용 | 인덱스 머지(제한적) | **비트맵 AND/OR**(일반적) |
| 힌트 | **옵티마이저 힌트 내장** | 기본 제공 없음, `pg_hint_plan` |

**가운데 두 줄이 성격을 요약한다.** MySQL은 **부족한 자리마다 전용 손잡이를 하나씩 붙이는 방식**이고(멀티밸류·히스토그램·ICP·MRR), PostgreSQL은 **범용 장치 하나가 여러 문제를 덮는 방식**이다(GIN·비트맵 스캔). 전자는 손잡이를 아는 만큼 쓸 수 있고, 후자는 몰라도 어느 정도 굴러간다.

---

**다음으로 읽을 것**

- 인덱스 자료구조와 `EXPLAIN` 읽는 법 → [DB 인덱스](../basics/b-tree-index.md) §2-2, §2-9
- 조인 알고리즘과 옵티마이저의 추정 실패 → [쿼리 실행](../basics/query-execution.md) §2-2, §2-4
- 인덱스를 추가·변경할 때 서비스가 멈추지 않게 하는 법 → [복제와 운영](replication-and-ops.md) §2-5
- 재방문 비용을 아예 없애는 PostgreSQL 쪽 접근 → [힙과 인덱스](../../postgresql/heap-and-index.md) §2-6

---

> **기준 버전**: MySQL 8.4 · PostgreSQL 17
> **확인한 출처** (전부 dev.mysql.com):
> - [CREATE INDEX (8.4)](https://dev.mysql.com/doc/refman/8.4/en/create-index.html) — 프리픽스 필수 대상(`BLOB`/`TEXT`)과 상한(**3072 / 767바이트**), 비이진은 **글자 수**·이진은 **바이트 수**, 함수 키 파트 제약(괄호 필수·PK 불가·**숨은 가상 생성 컬럼으로 구현**), 멀티밸류 인덱스의 `CAST(... AS ... ARRAY)`·사용 함수 3종·**정렬/커버링/범위 스캔 불가**·`ALGORITHM=COPY`·문자셋 제한 / [같은 문서 8.0판](https://dev.mysql.com/doc/refman/8.0/en/create-index.html) — **함수 키 파트 8.0.13**, **멀티밸류 8.0.17** 원문
> - [Descending Indexes](https://dev.mysql.com/doc/refman/8.4/en/descending-indexes.html) — *"DESC ... is no longer ignored"*, 혼합 정렬에서의 효용, **InnoDB `BTREE` 한정**, 체인지 버퍼링 미지원 / [Invisible Indexes](https://dev.mysql.com/doc/refman/8.4/en/invisible-indexes.html) — *"Index visibility does not affect index maintenance"*, **PK(암묵적 포함) 불가**, `use_invisible_indexes` 기본 `off`, *"fast, in-place operations"*
> - [ngram Full-Text Parser](https://dev.mysql.com/doc/refman/8.4/en/fulltext-search-ngram.html) — 기본 파서의 **공백 구분자 한계**와 CJK 대응 원문, `WITH PARSER ngram`, **`ngram_token_size` 기본 2**(1~10) / [Full-Text Search Functions](https://dev.mysql.com/doc/refman/8.4/en/fulltext-search.html) — 대상 엔진·컬럼 타입, 검색 모드 3종
> - [Optimizer Statistics](https://dev.mysql.com/doc/refman/8.4/en/optimizer-statistics.html) — 싱글턴/등고 분기 조건, *"created or updated only on demand"*, *"The optimizer prefers range optimizer row estimates"* / [ANALYZE TABLE](https://dev.mysql.com/doc/refman/8.4/en/analyze-table.html) — `UPDATE/DROP HISTOGRAM` 문법과 `{MANUAL | AUTO} UPDATE` 절, **버킷 1~1024·생략 시 100**, 분석 중 **읽기 락**, `histogram_generation_max_mem_size` 초과 시 샘플링
> - [Index Condition Pushdown](https://dev.mysql.com/doc/refman/8.4/en/index-condition-pushdown-optimization.html) — 동작 원리, 접근 방식 4종, **InnoDB는 세컨더리 인덱스에만**, `Using index condition` ≠ `Using index` / [Multi-Range Read](https://dev.mysql.com/doc/refman/8.4/en/mrr-optimization.html) — *"sequentially rather than in random order"*, **커버링이면 효용 없음**, `mrr`/`mrr_cost_based` 기본 `on` / [Index Merge](https://dev.mysql.com/doc/refman/8.4/en/index-merge-optimization.html) — 3종 알고리즘과 `Extra` 표기, **단일 테이블 한정**, *"not applicable to full-text indexes"*
> - [Hash Join (8.0)](https://dev.mysql.com/doc/refman/8.0/en/hash-joins.html) — **8.0.18 도입**, **8.0.20에서 BNL 제거**·비등치/아우터 확대, `join_buffer_size` 상한과 디스크 스필·`open_files_limit` 초과 시 **실패** / [Optimizer Hints](https://dev.mysql.com/doc/refman/8.4/en/optimizer-hints.html) — `/*+ ... */` 위치, **`BNL`/`NO_BNL`이 해시 조인을 제어**, *"JOIN_FIXED_ORDER ... the same as specifying SELECT STRAIGHT_JOIN"*
> **미확인**: 인비저블 인덱스·내림차순 인덱스의 **정확한 도입 포인트 릴리스**(8.0 계열인 것은 확인했으나 8.0.x가 문서에 명시돼 있지 않았다) · `innodb_ft_min_token_size` 기본값과 자연어 검색의 50% 임계 규칙(InnoDB 적용 여부) · §2-2의 ngram 인덱스 팽창률은 **자릿수 감각용 추정**이며 실측이 아니다 · §2-1의 프리픽스 길이 산정 절차는 관행이지 공식 권고문이 아니다 · PostgreSQL 쪽 비교 항목(표현식 인덱스·GIN·`pg_trgm`)은 이번에 PostgreSQL 문서로 재대조하지 않았고 [힙과 인덱스](../../postgresql/heap-and-index.md)·[복제와 운영](../../postgresql/replication-and-ops.md)의 기존 서술을 따랐다
> **미작성**: `innodb_adaptive_hash_index` · 공간 인덱스(`SPATIAL`) · `optimizer_switch` 전체 플래그 · 조인 순서 탐색 비용(`optimizer_search_depth`) · `EXPLAIN FORMAT=JSON`의 비용 항목 읽는 법
