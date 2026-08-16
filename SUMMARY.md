# Table of contents

* [소개](README.md)

## Database — 기초

* [DB는 디스크를 어떻게 다루나](database/basics/storage-and-io.md)
* [B+Tree는 왜 그렇게 생겼나](database/basics/b-tree-index.md)
* [옵티마이저는 왜 그 플랜을 골랐나](database/basics/query-execution.md)
* [트랜잭션 · 락 · 데드락 (+ 커넥션 풀)](database/basics/transaction-and-lock.md)
* [COMMIT은 무엇을 보장하나](database/basics/durability-and-recovery.md)
* [DB 한 대로 안 될 때](database/basics/replication-and-partitioning.md)
* [격리 수준은 어디서 나왔나](database/basics/concurrency-theory.md)
* [키와 함수 종속](database/basics/data-modeling.md)
* [쪼개는 규칙과 되돌리는 판단](database/basics/normalization.md)
* [RDB 전문 검색은 어디까지 되나](database/basics/text-search.md)

## Database — MySQL

* [엔진을 갈아 끼울 수 있게 만든 대가](database/mysql/innodb-internals.md)
* [기본값이 REPEATABLE READ라서 생기는 일들](database/mysql/lock-and-isolation.md)
* [MySQL이 따로 쥐고 있는 손잡이들](database/mysql/index-and-optimizer.md)
* [binlog 하나가 복제·운영·스키마 변경을 다 정한다](database/mysql/replication-and-ops.md)

## Database — PostgreSQL

* [청소가 왜 필수 업무인가](postgresql/mvcc-and-vacuum.md)
* [인덱스는 왜 힙을 못 벗어나나](postgresql/heap-and-index.md)
* [갭 락 없이 팬텀을 막는 법](postgresql/lock-and-types.md)
* [WAL 하나로 다 하는 대신 치르는 것](postgresql/replication-and-ops.md)

## Messaging

* [큐를 왜 쓰고, 무엇을 고를 것인가](messaging/basics.md)
* [Kafka: 큐가 아니라 로그다](messaging/kafka.md)
* [RabbitMQ: 소비하면 사라진다](messaging/rabbitmq.md)

## Datastore

* [무엇을 어디에 둘 것인가](datastore/selection.md)
* [Elasticsearch: 역색인은 방향을 뒤집는다](datastore/elasticsearch.md)
* [Redis: 싱글 스레드가 왜 설계 선택인가](datastore/redis.md)

## Java

* [JVM · GC · 컬렉션 · 동시성](java/jvm-gc-concurrency.md)

## Spring

* [DI · AOP · 트랜잭션 · JPA · Batch](spring/di-aop-transaction-jpa.md)

## System Design

* [대용량 처리 · 분산 시스템](system-design/high-throughput.md)

## Fundamentals

* [자료구조](fundamentals/data-structures.md)
* [복잡도가 아니라 전제가 알고리즘을 고른다](fundamentals/algorithms.md)

## OS

* [없는 메모리를 있는 척하는 법](os/memory-and-paging.md)
* [컨텍스트 스위칭은 왜 비싼가](os/process-and-scheduling.md)
* [동기/비동기와 블로킹/논블로킹](os/concurrency-primitives.md)
* [리눅스에서 장애 났을 때 뭘 보나](os/linux-troubleshooting.md)

## Infra

* [컨테이너: 가상머신이 아니라 격리된 프로세스다](infra/containers.md)
* [Kubernetes: 선언한 상태로 계속 되돌리는 기계](infra/kubernetes.md)

## Network

* [OSI 7계층은 왜 7개인가, 그리고 왜 안 맞는가](network/layered-model.md)
* [얼마나 빨리 보낼지 누가 정하나](network/transport-layer.md)
* [HTTP는 왜 세 번 다시 만들어졌나](network/application-layer.md)
* [타임아웃과 멱등성](network/reliability-patterns.md)
