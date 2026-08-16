# 무엇을 어디에 둘 것인가

## 30초 요약

- "NoSQL이 더 좋다"는 프레임 자체가 틀렸다. **원장은 RDB, 검색·집계는 ES처럼 나눠 쓰는 게 답이다**
- NoSQL이 나온 이유는 셋이다. RDB의 **수평 확장 한계, 스키마 경직성, 특수 접근 패턴**
- 가르는 질문도 하나다. **"조인과 트랜잭션이 필요한가, 아니면 쓰기 처리량과 다양한 조회가 필요한가."**
- CAP에서 실제 선택지는 C냐 A냐뿐이다. 네트워크 분할(P)은 **전제로 깔고 가는 조건**이다
- MongoDB를 한 줄로 줄이면 **조인을 쿼리에서 하는 대신 문서 설계 단계에서 미리 합쳐 두는 모델**이다

---

## 원리 — 왜 그런가

### 2-1. NoSQL이 왜 나왔나 · 4분류

RDB가 감당 못 하는 일이 생겼기 때문이다. 정확히는 세 가지다.

1. 수평 확장: RDB는 조인과 트랜잭션 때문에 여러 장비로 쪼개기 어렵다. 스케일 업(더 좋은 장비)에 의존한다
2. 스키마 유연성: 컬럼을 추가하려면 ALTER TABLE. 대용량 테이블에서는 위험한 작업이다
3. 특수 접근 패턴: 전문 검색, 시계열, 그래프 탐색은 RDB의 B+Tree로는 비효율적이다

| 유형 | 대표 | 데이터 모델 | 쓰는 자리 |
|---|---|---|---|
| Key-Value | [Redis](redis.md) | 키 → 값 | 캐시, 세션, 분산락, rate limit |
| Document | [Elasticsearch](elasticsearch.md), MongoDB | JSON 문서 | 유동적 스키마, **검색·집계** |
| Wide-column | Cassandra, HBase | 행 + 동적 컬럼 | 시계열, 초대용량 쓰기 |
| Graph | Neo4j | 노드 + 엣지 | 관계 탐색 |

> ⚠️ 이 분류에는 한계가 있다. "NoSQL"은 SQL이 아닌 것이라는 **부정으로 정의된 범주**라서, 문제 영역이 전혀 다른 제품들이 한 이름에 묶인다. Redis(캐시·락)와 Elasticsearch(검색)는 같은 자리를 두고 경쟁하지 않는다. 분류는 지도일 뿐이고, 실제 판단은 §2-3의 표로 한다.

**CAP 정리**
> 분산 시스템에서 네트워크 분할(P)은 고를 수 있는 항목이 아니라 전제다. 그래서 실제 선택은 분할이 일어났을 때 **일관성(C)을 지킬 것인가, 가용성(A)을 지킬 것인가** 둘 중 하나다.
- 과금·결제 → **C** (틀린 값을 주느니 에러를 준다)
- 발송 통계 조회 → **A** (조금 늦은 값이라도 응답한다)

**RDB와 NoSQL을 가르는 질문은 하나다**:
> *"조인과 트랜잭션이 필요한가, 아니면 쓰기 처리량과 유연한 스키마·다양한 조회가 필요한가."*
- 돈·과금·계약처럼 정합성이 생명인 원장 → **RDB**
- 로그·이력·검색처럼 양이 많고 조회 패턴이 다양한 것 → **NoSQL**

⚠️ 그래서 "NoSQL이 더 좋다"는 질문 자체가 성립하지 않는다. 실제 설계는 **원장은 RDB에, 검색·집계는 ES에 두고 둘을 나눠 쓰는 쪽**으로 간다.

### 2-2. MongoDB, 개념만

- **문서 지향**: BSON 문서. 컬렉션 = 테이블, 도큐먼트 = 행. 스키마는 고정이 아니다
- **임베딩 vs 레퍼런스**: 같이 읽는 데이터는 문서 안에 넣어 조인을 피하고, 독립적으로 크는 데이터는 참조로 분리한다. 한 줄로 줄이면 조인을 쿼리에서 하는 대신 문서 설계 단계에서 미리 합쳐 두는 모델이다
- **인덱스**: B-Tree 기반. 복합·부분·TTL 인덱스(자동 만료)를 지원한다. TTL 인덱스는 발송 이력 보존 기간에 쓸 수 있는데, 만료 즉시 지워지는 게 아니라 `mongod`의 백그라운드 스레드가 60초마다 돌면서 걷어간다. 부하가 높으면 그보다 더 오래 남는다. "만료 시각이 지나면 조회되지 않는다"를 보장해야 하면 쿼리 조건으로도 걸러야 한다
- **레플리카 셋**: Primary 1 + Secondary N, 자동 페일오버. 읽기를 세컨더리로 분산할 수 있고, 대신 복제 지연을 감수한다
- **샤딩**: 샤드 키 선택이 전부다. 단조 증가 키(타임스탬프, ObjectId)를 쓰면 항상 마지막 샤드로만 쓰기가 몰린다(hot shard). 해시 샤드 키나 복합 키로 푼다
- **트랜잭션**: 다중 문서 트랜잭션은 레플리카 셋에서 4.0, 샤디드 클러스터에서 4.2가 필요하다. 단 비용이 크므로 문서 설계로 피하는 게 정석이고, 이건 공식 문서의 입장이기도 하다. *"the availability of distributed transactions should not be a replacement for effective schema design"*

> 샤드 키의 hot shard 문제는 [대용량 처리](../system-design/high-throughput.md) §2-7의 파티션 키 선택, Kafka 파티션([Kafka](../messaging/kafka.md) §2-3), InnoDB의 단조 증가 PK([저장과 I/O](../database/basics/storage-and-io.md) §2-6)와 전부 같은 문제다. **키가 한쪽으로 쏠리면 분산 자체가 무의미해진다.**

### 2-3. 데이터별 저장소 판단표

설계할 때 그대로 꺼내 쓰라고 만든 표다.

| 데이터 | 저장소 | 이유 |
|---|---|---|
| 고객사·계약·과금 원장 | **RDB** | 정합성이 생명. 트랜잭션 필요 |
| 발송 요청·발송 상태 | **RDB** | 상태 전이와 중복 방지(유니크 제약) |
| 발송 상세 이력·로그 | [Elasticsearch](elasticsearch.md) | 쓰기 많음 + 조회 조건 다양 + 보존 기간(ILM) |
| 발송 통계·리포트 | RDB 집계 테이블 또는 ES 집계 | 원장에는 **집계만** 남긴다 |
| 템플릿·발신프로필 캐시 | [Redis](redis.md) | 자주 읽고 거의 안 바뀜 |
| 멱등키(중복 발송 차단 1차) | [Redis](redis.md) (`SET NX EX`) | 빠른 사전 차단. 최종 방어는 DB 유니크 |
| 채널사 rate limit 토큰 | [Redis](redis.md) | 원자적 카운터 |
| 예약 발송 대기열 | [Redis](redis.md) Sorted Set 또는 DB | score = 발송 예정 시각 |

---

**다음으로 읽을 것**

- 역색인이 왜 빠른가 → [Elasticsearch](elasticsearch.md) §2-1
- 왜 원장을 Redis에 두면 안 되는가 → [Redis](redis.md) §2-3
- RDB 쪽의 근거 → [트랜잭션 · 락](../database/basics/transaction-and-lock.md)

---

> **기준 버전**: 제품 버전에 의존하지 않는 판단 프레임. MongoDB 서술은 레플리카 셋 4.0 / 샤디드 클러스터 4.2 이상 가정이고, 대조는 mongodb.com 현행 매뉴얼로 했다
>
> **확인한 출처**:
> - [MongoDB Transactions](https://www.mongodb.com/docs/manual/core/transactions/) — **다중 문서 트랜잭션 도입 버전**. FCV 요구가 레플리카 셋 `4.0`, 샤디드 클러스터 `4.2`다. 본문의 "4.0부터"를 두 값으로 갈라 적었다. 그리고 문서 설계로 피하라는 권고 원문 *"In most cases, a distributed transaction incurs a greater performance cost over single document writes, and the availability of distributed transactions should not be a replacement for effective schema design."*, 임베딩 권고 *"Because you can use embedded documents and arrays to capture relationships between data in a single document structure instead of normalizing across multiple documents and collections, multi-document transactions are not necessary for many practical use cases."*
> - [TTL Indexes](https://www.mongodb.com/docs/manual/core/index-ttl/) — **TTL 인덱스 동작**. *"A background thread in `mongod` reads the values in the index and removes expired documents from the collection."*, *"The background task that removes expired documents runs every 60 seconds."*, *"The TTL index does not guarantee that expired data is deleted immediately upon expiration."*, 그리고 부하에 따라 *"expired data may exist for some time beyond the 60 second period"*. §2-2에 이 지연을 새로 적었다
> - [Choose a Shard Key](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/) — §2-2의 hot shard 서술. 단조 증가 키면 `maxKey`를 상한으로 갖는 청크로 전부 몰리고 *"The shard containing that chunk becomes the bottleneck for write operations."*, 해법으로 *"consider using Hashed Sharding"*. 카디널리티가 낮으면 *"reduces the effectiveness of horizontal scaling in the cluster"*
> - [Replication](https://www.mongodb.com/docs/manual/replication/) — Primary 1 + Secondary N 구조, `electionTimeoutMillis` 기본 10초 후 선거, 세컨더리 읽기의 대가 *"Asynchronous replication to secondaries means that reads from secondaries may return data that does not reflect the state of the data on the primary."*
> - [Indexes](https://www.mongodb.com/docs/manual/indexes/) — *"MongoDB indexes use a B-tree data structure."*
> - [Elasticsearch: Joining queries](https://www.elastic.co/docs/reference/query-languages/query-dsl/joining-queries) — §2-1이 ES를 "조인 없음" 쪽에 두는 근거. *"Performing full SQL-style joins in a distributed system like Elasticsearch is prohibitively expensive."*
> - Eric Brewer, [CAP Twelve Years Later: How the "Rules" Have Changed](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/) — §2-1의 CAP 서술. "3 중 2" 프레임이 *"tended to oversimplify the tensions among properties"* 이고 *"was always misleading"* 이라는 것, 그리고 C/A 선택이 분할이 있을 때 생긴다는 것(*"Although designers still need to choose between consistency and availability when partitions are present"*). 본문의 "P는 전제고 실제 선택은 C냐 A냐"는 이 논지와 맞는다
>
> **미확인**:
> - **CAP는 벤더 공식 문서가 있는 주제가 아니다.** 위 Brewer 글은 IEEE Computer 게재본의 InfoQ 재수록이고, 인용한 것은 문장 단위 발췌다. Gilbert–Lynch의 원 증명 논문(2002)은 PDF라 본문을 열지 못했다. 본문의 "과금·결제 → C, 발송 통계 → A" 대응은 이 저장소의 판단이지 논문의 진술이 아니다
> - **§2-1의 NoSQL 4분류 표** — Cassandra·HBase·Neo4j 항목은 각 제품 공식 문서로 대조하지 않았다. Redis와 Elasticsearch 항목만 해당 문서(`redis.md`, `elasticsearch.md`)에서 확인한 범위다
> - **§2-1의 "NoSQL이 나온 이유 셋"** — 기술사 서술이라 대조할 1차 문서가 없다. `ALTER TABLE`이 대용량에서 위험하다는 부분은 [MySQL 온라인 DDL](../database/mysql/index-and-optimizer.md) 쪽에서 다룰 주제이고 여기서는 확인하지 않았다
> - **§2-2의 BSON·컬렉션/도큐먼트 용어 대응** — MongoDB 매뉴얼의 데이터 모델 장을 직접 열어 대조하지는 않았다
>
> **미작성**: MongoDB 집계 파이프라인 · 청크 밸런서와 리샤딩 · 읽기/쓰기 concern 조합 · Cassandra의 파티션 키와 클러스터링 키 · 폴리글랏 저장소 사이의 데이터 동기화(CDC, outbox)
