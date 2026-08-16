# 역색인은 방향을 뒤집는다

## 30초 요약

- **역색인**: RDB 인덱스는 *문서 → 값*이고, ES는 *값(단어) → 그 단어가 든 문서 목록*이다. 그래서 전문 검색이 빠르다
- RDB의 `LIKE '%키워드%'`는 인덱스를 못 탄다. **ES를 쓰는 첫 번째 이유가 이것이다**
- **쓰기가 비싸고 읽기가 싸다.** 색인 시점에 애널라이저가 토큰화·정규화를 미리 해 둔다. RDB와는 반대 방향의 거래다
- `text` vs `keyword`: 분석되어 검색용, 분석되지 않아 정확 일치·집계·정렬용. 실무에서는 **멀티 필드로 둘 다 둔다**
- Near Real-Time(기본 1초), 트랜잭션·조인 없음, 업데이트가 사실상 삭제 후 재색인. **전부 세그먼트가 불변이라는 한 가지에서 파생된다**

---

## 원리 — 왜 그런가

**정체**: Lucene 기반 분산 검색·분석 엔진. JSON 문서를 저장하고 역색인으로 검색한다.

### 2-1. 역색인이 왜 빠른가

RDB에서 `WHERE content LIKE '%예약%'`이 왜 느린지부터 보자. 인덱스는 **컬럼 원본 값의 정렬**이다([DB 인덱스](../database/basics/b-tree-index.md) §2-6). 앞이 와일드카드면 시작점을 찾을 수 없으니 전체를 읽으며 문자열을 비교한다. 100만 행이면 100만 번이다.

역색인은 이 방향을 뒤집는다.
```
RDB 인덱스 :  문서1 → "예약 발송이 실패했습니다"
역색인     :  "예약"   → [문서1, 문서7, 문서93]
             "발송"   → [문서1, 문서2, 문서7, ...]
             "실패"   → [문서1, 문서5]
```
검색이 단어로 목록을 바로 꺼내는 일이 된다. 문서가 100만 건이든 1억 건이든 "예약"이라는 키 하나를 찾는 비용만 든다.

대가는 색인 시점에 치른다. 문서를 넣을 때 **애널라이저**가 세 가지를 한다.
1. **토크나이즈**: 문장을 단어로 쪼갠다 (한국어는 형태소 분석기, 예: Nori)
2. **정규화**: 소문자화, 불용어 제거, 어간 추출
3. 그 결과를 역색인에 넣는다

결과적으로 **쓰기가 비싸고 읽기가 싸다.** RDB와는 반대 방향의 거래다.

### 2-2. `text` vs `keyword`

| | `text` | `keyword` |
|---|---|---|
| 애널라이저 | **통과함** (토큰으로 쪼개짐) | **통과 안 함** (통째로 저장) |
| 쓰임 | 전문 검색 (`match`) | 정확 일치(`term`), 집계, 정렬 |
| 예 | 메시지 본문 | 채널 코드, 에러코드, 수신번호 |

왜 나눠야 하나. `"010-1234-5678"`을 `text`로 넣으면 `010`, `1234`, `5678`로 쪼개진다. 그러면 **정확히 그 번호를 찾는 것도, 번호별로 집계하는 것도 안 된다.**
그래서 실무에서는 **멀티 필드**로 둘 다 둔다. `message`(text, 검색용) + `message.keyword`(keyword, 집계용).

### 2-3. 헷갈리기 쉬운 구조 용어

- **인덱스(Index)** = RDB의 테이블에 해당한다. ⚠️ DB에서 말하는 그 인덱스와는 다른 저장 단위다. 이 제품에서 제일 헷갈리는 용어
- **샤드(Shard)** = 인덱스를 나눈 물리 단위. 프라이머리 샤드 수(`index.number_of_shards`)는 인덱스 생성 시점에만 지정할 수 있다. split API로 늘릴 수는 있지만 인덱스를 읽기 전용으로 막아야 하고, 대상 샤드 수가 원본 샤드 수의 배수여야 한다
  → 왜 그런가. 문서가 어느 샤드로 갈지를 라우팅 값의 해시로 정하기 때문이다. 공식 문서에 적힌 식은 `shard_num = (hash(_routing) % num_routing_shards) / routing_factor`이고, `_routing`은 따로 지정하지 않으면 문서의 `_id`다. 샤드 수가 바뀌면 기존 문서의 위치 계산이 틀어진다. Kafka 파티션과 똑같은 이유다([Kafka](../messaging/kafka.md) §2-3)
- **레플리카(Replica)** = 샤드 복제본. 가용성과 읽기 처리량 분산을 담당한다. 이건 나중에 바꿀 수 있다
- **매핑(Mapping)** = 필드 타입 정의. 스키마리스처럼 보이지만 실무에서는 명시적으로 잡는다 (자동 추론이 틀리면 재색인해야 하므로)

### 2-4. Near Real-Time, 색인해도 왜 바로 안 보이나

색인 직후 검색하면 안 나온다. `index.refresh_interval` 기본값인 **1초** 뒤에야 보인다. 이유는 Lucene의 구조에 있다.

- Lucene은 데이터를 **세그먼트(segment)** 라는 불변(immutable) 파일에 쓴다
- 새 문서는 일단 메모리 색인 버퍼에 쌓이고, refresh 시점에 새 세그먼트로 만들어져야 검색 대상이 된다
- 문서마다 즉시 세그먼트를 만들면 파일이 폭증한다. 그래서 모아서 1초에 한 번 처리한다

조건이 하나 더 붙는다. 공식 문서는 *"By default, Elasticsearch periodically refreshes indices every second, but only on indices that have received one search request or more in the last 30 seconds."* 라고 적는다. 아무도 검색하지 않는 인덱스는 1초마다 refresh하지 않는다는 뜻이다. Serverless의 기본값은 1초가 아니라 5초다.

**여기서 파생되는 것들:**
- **업데이트가 사실상 삭제 후 재색인이다.** 세그먼트가 불변이라 수정 자체가 불가능하다. 기존 문서를 "삭제됨"으로 표시만 하고 새 문서를 쓴다. 잦은 업데이트에 불리한 이유다
- 삭제 표시가 쌓이면 **세그먼트 병합(merge)** 이 일어나 실제로 정리된다. 작은 세그먼트를 큰 세그먼트로 합치면서 삭제분을 걷어내는(*"expunge deletes"*) 작업이고, ES는 이 I/O를 색인 처리량에 영향이 가지 않도록 스로틀링한다
- 대량 색인 시 `refresh_interval`을 늘리거나(`30s` 등) `-1`로 꺼 두는 것이 공식 문서가 권하는 튜닝이다

> PostgreSQL의 dead tuple과 구조가 같다. 옛 버전을 그 자리에서 못 고치니 삭제 표시만 남기고, 쌓이면 별도 정리 작업이 돈다(ES는 세그먼트 병합, PG는 VACUUM). **불변 저장을 택하면 청소 담당이 반드시 따라온다.** 자세한 건 [저장과 I/O](../database/basics/storage-and-io.md) §2-6에 있다.

### 2-5. RDB와 다른 점 정리

- 트랜잭션 없음. 여러 문서를 묶어 롤백할 수 없어서, 원장으로 쓰면 안 된다
- SQL식 조인 없음. 공식 문서의 표현은 *"Performing full SQL-style joins in a distributed system like Elasticsearch is prohibitively expensive"* 이고, 대신 한 인덱스 안에서 쓰는 `nested`와 `join` 필드(`has_child`/`has_parent`)를 제공한다. 실무에서는 색인 시점에 비정규화해서 합쳐 두는 쪽을 먼저 고른다
- 집계(Aggregation)가 강하다. 발송 성공/실패율, 채널별·시간대별 추이. RDB에서 `GROUP BY`로 수백만 건을 훑는 것보다 압도적으로 빠르다
- `from`/`size` 딥 페이징이 위험하다. 애초에 기본 상한이 있어서 *"By default, you cannot use `from` and `size` to page through more than 10,000 hits."* 이고, 이유는 *"Each shard must load its requested hits and the hits for any previous pages into memory."* 다. 대안은 `search_after`이며, 페이지를 넘기는 동안 인덱스 상태를 고정하려면 PIT(point in time)와 함께 쓴다

### 2-6. ILM과 보존 기간 정책

- **ILM(Index Lifecycle Management)**: 인덱스는 정책에 따라 hot → warm → cold → frozen → delete 다섯 단계를 지나간다. hot은 활발히 쓰고 읽는 구간, warm은 읽기 전용으로 돌리고 샤드를 줄이는 구간, cold는 더 싼 하드웨어로 옮기는 구간이다
- **시계열 인덱스 전략**: `send-log-2026.08` 처럼 날짜 단위로 인덱스를 쪼갠다
  → 오래된 것을 인덱스 통째로 삭제할 수 있다. 행 단위 DELETE보다 압도적으로 싸고, 세그먼트 병합도 안 생긴다
- **제품 정책으로 자주 보이는 형태**: *"발송 상세 내역은 최대 3개월까지만 조회, 이후엔 성공/실패 건수만"*
  → 이런 문구는 대개 상세 로그를 보존 기간이 있는 별도 스토어에 두고, 집계만 원장에 남기는 구조라는 신호다
  → RDB로 같은 정책을 구현한 것이 [DB 인덱스](../database/basics/b-tree-index.md) §2-12의 파티셔닝이다. **발상은 같다. 행 단위로 지우지 말고 덩어리째 버린다**

---

**다음으로 읽을 것**

- 무엇을 ES에 두고 무엇을 RDB에 둘 것인가 → [저장소 선택](selection.md)
- RDB 안에서의 전문 검색 → `database/basics/text-search.md` (예정)

---

> **기준 버전**: Elasticsearch 8.x 계열 가정. 대조는 elastic.co 현행 문서(`/docs/`)로 했다
>
> **확인한 출처**:
> - [General index settings](https://www.elastic.co/docs/reference/elasticsearch/index-settings/index-modules) — `index.refresh_interval` 기본값이 *"1s in Elastic Stack and 5s in Serverless"* 이고 `-1`로 끌 수 있다는 것, `index.number_of_shards`가 *"can only be set at index creation time"* 이라는 것
> - [Near real-time search](https://www.elastic.co/docs/manage-data/data-store/near-real-time-search) — 세그먼트 정의(*"A segment is similar to an inverted index, but the word index in Lucene means 'a collection of segments plus a commit point'"*), refresh 정의(*"this process of writing and opening a new segment is called a refresh"*), 메모리 색인 버퍼 → 파일시스템 캐시 → 디스크 순서, **그리고 §2-4에 새로 넣은 조건** *"only on indices that have received one search request or more in the last 30 seconds"*
> - [Merge](https://www.elastic.co/docs/reference/elasticsearch/index-settings/merge) — 세그먼트가 *"immutable"* 이라는 것, 병합이 *"expunge deletes"* 를 겸한다는 것, 병합 I/O가 스로틀링된다는 것
> - [Tune for indexing speed](https://www.elastic.co/docs/deploy-manage/production-guidance/optimize-performance/indexing-speed) — 대량 색인 시 `refresh_interval`을 `-1`로 끄거나 `30s` 같은 큰 값으로 올리라는 권고
> - [`_routing` field](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/mapping-routing-field) — 라우팅 식 `routing_factor = num_routing_shards / num_primary_shards`, `shard_num = (hash(_routing) % num_routing_shards) / routing_factor`, `_routing` 기본값이 문서 `_id`라는 것
> - [Split index API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-split) — 인덱스가 읽기 전용이어야 하고, 대상 프라이머리 샤드 수가 원본의 배수여야 한다는 제약
> - [Text analysis](https://www.elastic.co/docs/manage-data/data-store/text-analysis) — 분석이 `text` 필드에만 일어난다는 것, 토큰화와 정규화(소문자화·어간 추출·불용어)의 정의 · [`text` 필드](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/text) — *"These fields are `analyzed`"*, *"Text fields are not used for sorting and seldom used for aggregations"* · [`keyword` 필드](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/keyword) — 정형 데이터(ID·이메일·상태코드)용이며 *"often used in sorting, aggregations, and term-level queries"*, *"Avoid using keyword fields for full-text search"* · [multi-fields](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/multi-fields) — 같은 문자열을 *"a `text` field for full-text search, and as a `keyword` field for sorting or aggregations"* 로 동시에 색인하는 §2-2의 멀티 필드 관행
> - [Paginate search results](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/paginate-search-results) — `from`/`size` 기본 상한 10,000, 각 샤드가 이전 페이지분까지 메모리에 올린다는 이유, `search_after` + PIT 대안
> - [Joining queries](https://www.elastic.co/docs/reference/query-languages/query-dsl/joining-queries) — *"Performing full SQL-style joins in a distributed system like Elasticsearch is prohibitively expensive"*, 대안으로 `nested`와 `join` 필드(`has_child`/`has_parent`)를 제공한다는 것
> - [ILM](https://www.elastic.co/docs/manage-data/lifecycle/index-lifecycle-management) — 단계가 *"(`hot`, `warm`, `cold`, `frozen`, `delete`)"* 다섯이라는 것과 예시(warm에서 읽기 전용 + shrink, cold에서 저가 하드웨어로 이동, delete에서 보존 기간 만료 시 삭제)
>
> **미확인**:
> - **§2-5의 "트랜잭션 없음"** — 여러 문서를 묶는 트랜잭션이 없다는 건 조인 문서와 낙관적 동시성 제어 문서에서 간접적으로만 확인된다. **"문서 단위 원자성은 보장된다"** 는 원래 문장은 공식 문서에서 그렇게 명시한 곳을 찾지 못해 본문에서 뺐다
> - **한국어 형태소 분석기 Nori** — §2-1의 예시로 이름만 언급한다. 플러그인 문서를 열어 설치·설정·동작을 대조하지 않았다
> - **§2-6의 시계열 인덱스 삭제가 행 단위 DELETE보다 싸다는 비교** — ILM/rollover 문서에서 인덱스 통째 삭제가 정책 수단이라는 건 확인했지만, 비용 배수를 수치로 진술한 공식 문서는 확인하지 못했다. 본문의 "압도적으로 싸다"는 정성 서술이다
> - **§2-5의 집계 성능이 RDB `GROUP BY`보다 빠르다는 비교** — 벤치마크 근거를 확인하지 않았다
>
> **미작성**: Nori 애널라이저 설정 · `search_after` + PIT 실제 사용법 · 데이터 스트림과 rollover · 샤드 크기 산정 기준 · `_source` 비활성화와 저장 공간 튜닝
