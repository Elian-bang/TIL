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

**Database**
- [DB 인덱스 — B+Tree는 왜 그렇게 생겼나](database/b-tree-index.md)
- [트랜잭션 · 락 · 데드락 (+ 커넥션 풀)](database/transaction-and-lock.md)

**Messaging**
- [메시징 — Kafka · RabbitMQ · ActiveMQ](messaging/kafka-and-rabbitmq.md)

**Search & NoSQL**
- [NoSQL — Elasticsearch · Redis · MongoDB](search-and-nosql/elasticsearch-redis-mongodb.md)

**Java / Spring**
- [Java — JVM · GC · 컬렉션 · 동시성](java/jvm-gc-concurrency.md)
- [Spring — DI · AOP · 트랜잭션 · JPA · Batch](spring/di-aop-transaction-jpa.md)

**System Design**
- [대용량 처리 · 분산 시스템](system-design/high-throughput.md)

**Fundamentals**
- [자료구조 · 알고리즘](fundamentals/data-structures.md)
- [네트워크 · OS](fundamentals/network-and-os.md)

---

예제는 대량 발송(메시징) 도메인에서 가져온 게 많다. 그쪽 일을 오래 해서 손에 붙은 예시가 그것뿐이라서고, 원리 자체는 도메인과 무관하다.

틀린 내용이 있으면 이슈로 알려주면 고친다.
