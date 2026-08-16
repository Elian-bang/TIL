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
- [저장과 I/O — DB는 디스크를 어떻게 다루나](database/basics/storage-and-io.md)
- [DB 인덱스 — B+Tree는 왜 그렇게 생겼나](database/basics/b-tree-index.md)
- [쿼리 실행 — 옵티마이저는 왜 그 플랜을 골랐나](database/basics/query-execution.md)
- [트랜잭션 · 락 · 데드락 (+ 커넥션 풀)](database/basics/transaction-and-lock.md)
- [내구성과 복구 — COMMIT은 무엇을 보장하나](database/basics/durability-and-recovery.md)
- [동시성 제어 이론 — 격리 수준은 어디서 나왔나](database/basics/concurrency-theory.md)
- [데이터 모델링 — 키와 함수 종속](database/basics/data-modeling.md)
- [정규화 — 쪼개는 규칙과 되돌리는 판단](database/basics/normalization.md)

**Messaging**
- [메시징 기초 — 큐를 왜 쓰고, 무엇을 고를 것인가](messaging/basics.md)
- [Kafka — 큐가 아니라 로그다](messaging/kafka.md)
- [RabbitMQ — 소비하면 사라진다](messaging/rabbitmq.md)

**Datastore**
- [저장소 선택 — 무엇을 어디에 둘 것인가](datastore/selection.md)
- [Elasticsearch — 역색인은 방향을 뒤집는다](datastore/elasticsearch.md)
- [Redis — 싱글 스레드가 왜 설계 선택인가](datastore/redis.md)

**Java / Spring**
- [Java — JVM · GC · 컬렉션 · 동시성](java/jvm-gc-concurrency.md)
- [Spring — DI · AOP · 트랜잭션 · JPA · Batch](spring/di-aop-transaction-jpa.md)

**System Design**
- [대용량 처리 · 분산 시스템](system-design/high-throughput.md)

**Fundamentals**
- [자료구조](fundamentals/data-structures.md)

**OS**
- [메모리와 페이징 — 없는 메모리를 있는 척하는 법](os/memory-and-paging.md)
- [프로세스와 스레드 — 컨텍스트 스위칭은 왜 비싼가](os/process-and-scheduling.md)
- [동시성 원시타입 — 동기/비동기와 블로킹/논블로킹](os/concurrency-primitives.md)
- [리눅스 — 장애 났을 때 뭘 보나](os/linux-troubleshooting.md)

**Infra**
- [컨테이너 — 가상머신이 아니라 격리된 프로세스다](infra/containers.md)
- [Kubernetes — 선언한 상태로 계속 되돌리는 기계](infra/kubernetes.md)

**Network**
- [계층 모델 — OSI 7계층은 왜 7개인가](network/layered-model.md)
- [전송 계층 — 연결은 왜 비싼가](network/transport-layer.md)
- [응용 계층 — HTTP 상태코드와 TLS](network/application-layer.md)
- [신뢰성 패턴 — 타임아웃과 멱등성](network/reliability-patterns.md)

---

## 로드맵

각 문서 하단에는 **기준 버전 · 확인한 출처 · 미확인 · 미작성** 블록이 있다. **확인하지 못한 것을 감추지 않는 게 이 저장소의 규칙**이다. 아래는 그걸 저장소 단위로 모은 것이다.

### 채울 것

- [x] ~~**전송 계층 보강** — 혼잡 제어·흐름 제어·TIME_WAIT~~ (RFC 5681 대조 완료)
- [ ] **프로세스 스케줄링** — CPU 스케줄링 알고리즘·선점/비선점. *"컨텍스트 스위칭이 비싸다"를 전제로 쓰는데 스케줄러가 언제 왜 전환하는지가 없다*
- [ ] **동기화 원시타입** — 임계 구역·뮤텍스 vs 세마포어·모니터. *DB 락을 다루면서 그 원형인 OS 동기화가 없다*
- [ ] **복제와 파티셔닝** — 복제 지연·읽기 분산·페일오버·샤드 키. *[내구성과 복구](database/basics/durability-and-recovery.md) §2-8이 "복제는 다른 장비에서 계속되는 크래시 복구"까지만 걸어 두고 끊긴다*
- [ ] **RDB 전문 검색** — `LIKE '%..%'`가 인덱스를 못 타는 이유·n-gram·형태소
- [ ] **MySQL 4편** — InnoDB 내부 / 갭 락·MDL / Full-Text·옵티마이저 / binlog·Online DDL
- [ ] **PostgreSQL 4편** — MVCC와 VACUUM / 힙과 인덱스(GIN·GiST·BRIN) / SSI·jsonb / 논리 복제·PgBouncer
- [ ] **응용 계층 보강** — DNS·HTTP/2·HTTP/3·CDN
- [ ] **알고리즘** — 탐색·그래프·DP. *현재 `자료구조` 문서에 정렬 한 절뿐이라 제목에서 "알고리즘"을 뺐다*
- [ ] **미확인 항목 해소** — 각 문서의 `미확인` 칸을 공식 문서로 대조

### 다루지 않기로 한 것

체계적인 CS 커리큘럼(GeeksforGeeks 등)에는 있지만 **의도적으로 넣지 않는다.** 시험용 형식 지식이지 "왜 그렇게 설계됐나"가 아니라서다.

관계대수·튜플 관계 해석 · DKNF · ER 다이어그램 최소화 · Banker's Algorithm · Dekker/Peterson/Bakery 알고리즘 · OSI 세션·표현 계층 프로토콜(AFP·NCP) · Zigbee·5G 슬라이싱 · Data Mining·Hive 아키텍처

---

예제는 대량 발송(메시징) 도메인에서 가져온 게 많다. 그쪽 일을 오래 해서 손에 붙은 예시가 그것뿐이라서고, 원리 자체는 도메인과 무관하다.

틀린 내용이 있으면 이슈로 알려주면 고친다.
