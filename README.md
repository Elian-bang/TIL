# TIL

백엔드 개발을 하면서 "왜 그런가"를 다시 확인한 것들을 정리해 둔다.

정의를 옮겨 적는 대신 **왜 그렇게 설계됐는지**를 남기는 걸 기준으로 삼았다.
B+Tree가 왜 B-Tree가 아닌지, InnoDB의 락이 왜 인덱스에 걸리는지, 가상 스레드를 늘려도 왜 안 빨라지는지 — 이런 것들이다.

## 이 문서의 구조

각 페이지는 두 부분이다.

| | |
|---|---|
| **30초 요약** | 나중에 다시 볼 때 이것만 읽어도 되게 |
| **원리 — 왜 그런가** | 그 결론이 나온 이유. 본문 |

## 목차

**Database** — `기초`(엔진 무관 원리) · `mysql`(InnoDB 고유) · `postgresql`(PG 고유) 세 갈래로 관리한다.
- [DB는 디스크를 어떻게 다루나](database/basics/storage-and-io.md)
- [B+Tree는 왜 그렇게 생겼나](database/basics/b-tree-index.md)
- [옵티마이저는 왜 그 플랜을 골랐나](database/basics/query-execution.md)
- [트랜잭션 · 락 · 데드락 (+ 커넥션 풀)](database/basics/transaction-and-lock.md)
- [COMMIT은 무엇을 보장하나](database/basics/durability-and-recovery.md)
- [DB 한 대로 안 될 때](database/basics/replication-and-partitioning.md)
- [격리 수준은 어디서 나왔나](database/basics/concurrency-theory.md)
- [키와 함수 종속](database/basics/data-modeling.md)
- [쪼개는 규칙과 되돌리는 판단](database/basics/normalization.md)
- [RDB 전문 검색은 어디까지 되나](database/basics/text-search.md)

**MySQL**
- [엔진을 갈아 끼울 수 있게 만든 대가](database/mysql/innodb-internals.md)
- [기본값이 REPEATABLE READ라서 생기는 일들](database/mysql/lock-and-isolation.md)
- [MySQL이 따로 쥐고 있는 손잡이들](database/mysql/index-and-optimizer.md)
- [binlog 하나가 복제·운영·스키마 변경을 다 정한다](database/mysql/replication-and-ops.md)

**PostgreSQL**
- [청소가 왜 필수 업무인가](postgresql/mvcc-and-vacuum.md)
- [인덱스는 왜 힙을 못 벗어나나](postgresql/heap-and-index.md)
- [갭 락 없이 팬텀을 막는 법](postgresql/lock-and-types.md)
- [WAL 하나로 다 하는 대신 치르는 것](postgresql/replication-and-ops.md)

**Messaging**
- [큐를 왜 쓰고, 무엇을 고를 것인가](messaging/basics.md)
- [Kafka: 큐가 아니라 로그다](messaging/kafka.md)
- [RabbitMQ: 소비하면 사라진다](messaging/rabbitmq.md)

**Datastore**
- [무엇을 어디에 둘 것인가](datastore/selection.md)
- [Elasticsearch: 역색인은 방향을 뒤집는다](datastore/elasticsearch.md)
- [Redis: 싱글 스레드가 왜 설계 선택인가](datastore/redis.md)

**Java / Spring**
- [JVM · GC · 컬렉션 · 동시성](java/jvm-gc-concurrency.md)
- [DI · AOP · 트랜잭션 · JPA · Batch](spring/di-aop-transaction-jpa.md)

**System Design**
- [대용량 처리 · 분산 시스템](system-design/high-throughput.md)

**Fundamentals**
- [자료구조](fundamentals/data-structures.md)
- [복잡도가 아니라 전제가 알고리즘을 고른다](fundamentals/algorithms.md)

**OS**
- [없는 메모리를 있는 척하는 법](os/memory-and-paging.md)
- [컨텍스트 스위칭은 왜 비싼가](os/process-and-scheduling.md)
- [동기/비동기와 블로킹/논블로킹](os/concurrency-primitives.md)
- [리눅스에서 장애 났을 때 뭘 보나](os/linux-troubleshooting.md)

**Infra**
- [컨테이너: 가상머신이 아니라 격리된 프로세스다](infra/containers.md)
- [Kubernetes: 선언한 상태로 계속 되돌리는 기계](infra/kubernetes.md)

**Network**
- [OSI 7계층은 왜 7개인가, 그리고 왜 안 맞는가](network/layered-model.md)
- [얼마나 빨리 보낼지 누가 정하나](network/transport-layer.md)
- [HTTP는 왜 세 번 다시 만들어졌나](network/application-layer.md)
- [타임아웃과 멱등성](network/reliability-patterns.md)

---

## 로드맵

각 문서 하단에는 기준 버전 · 확인한 출처 · 미확인 · 미작성 블록이 있다. **확인하지 못한 것을 감추지 않는 게 이 저장소의 규칙**이다.

39편 전부 공식 문서 대조를 마쳤고 인용은 199건이다. 그래도 각 문서의 `미확인` 칸은 비어 있지 않다. 원논문이 스캔본이라 못 읽은 것, 문서가 접근을 막은 것, 널리 쓰이지만 공식 문서에는 숫자가 없는 것들이 거기 적혀 있다.

### 채울 것

- [x] ~~**전송 계층 보강** — 혼잡 제어·흐름 제어·TIME_WAIT~~ (RFC 5681 대조 완료)
- [x] ~~**프로세스 스케줄링**~~ (man7 sched(7) 등 대조 완료)
- [x] ~~**동기화 원시타입**~~ (pthreads·futex 문서 대조 완료)
- [x] ~~**복제와 파티셔닝** — 복제 지연·읽기 분산·페일오버~~ (PostgreSQL 공식 문서 대조 완료)
- [x] ~~**RDB 전문 검색**~~ (MySQL·PostgreSQL 공식 문서 8개 절 대조 완료)
- [x] ~~**MySQL 4편**~~ (공식 문서 대조 완료)
- [x] ~~**PostgreSQL 4편**~~ (공식 문서 8개 절 대조 완료)
- [x] ~~**응용 계층 보강**~~ (RFC 대조 완료)
- [x] ~~**알고리즘** — 탐색·그래프·DP~~ (별도 편으로 분리)
- [x] ~~**미확인 항목 해소**~~ (39편 전부 대조, 공식 문서 인용 199건)

### 다루지 않기로 한 것

체계적인 CS 커리큘럼(GeeksforGeeks 등)에는 있지만 **의도적으로 넣지 않는다.** 시험용 형식 지식이지 "왜 그렇게 설계됐나"가 아니라서다.

관계대수·튜플 관계 해석 · DKNF · ER 다이어그램 최소화 · Banker's Algorithm · Dekker/Peterson/Bakery 알고리즘 · OSI 세션·표현 계층 프로토콜(AFP·NCP) · Zigbee·5G 슬라이싱 · Data Mining·Hive 아키텍처

---

예제는 대량 발송(메시징) 도메인에서 가져온 게 많다. 그쪽 일을 오래 해서 손에 붙은 예시가 그것뿐이라서고, 원리 자체는 도메인과 무관하다.

틀린 내용이 있으면 이슈로 알려주면 고친다.
