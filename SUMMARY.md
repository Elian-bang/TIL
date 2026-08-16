# Table of contents

* [소개](README.md)

## Database — 기초

* [저장과 I/O — DB는 디스크를 어떻게 다루나](database/basics/storage-and-io.md)
* [DB 인덱스 — B+Tree는 왜 그렇게 생겼나](database/basics/b-tree-index.md)
* [쿼리 실행 — 옵티마이저는 왜 그 플랜을 골랐나](database/basics/query-execution.md)
* [트랜잭션 · 락 · 데드락 (+ 커넥션 풀)](database/basics/transaction-and-lock.md)
* [내구성과 복구 — COMMIT은 무엇을 보장하나](database/basics/durability-and-recovery.md)
* [복제와 파티셔닝 — 한 대로 안 될 때](database/basics/replication-and-partitioning.md)
* [동시성 제어 이론 — 격리 수준은 어디서 나왔나](database/basics/concurrency-theory.md)
* [데이터 모델링 — 키와 함수 종속](database/basics/data-modeling.md)
* [정규화 — 쪼개는 규칙과 되돌리는 판단](database/basics/normalization.md)

## Database — MySQL

## Database — PostgreSQL

## Messaging

* [메시징 기초 — 큐를 왜 쓰고, 무엇을 고를 것인가](messaging/basics.md)
* [Kafka — 큐가 아니라 로그다](messaging/kafka.md)
* [RabbitMQ — 소비하면 사라진다](messaging/rabbitmq.md)

## Datastore

* [저장소 선택 — 무엇을 어디에 둘 것인가](datastore/selection.md)
* [Elasticsearch — 역색인은 방향을 뒤집는다](datastore/elasticsearch.md)
* [Redis — 싱글 스레드가 왜 설계 선택인가](datastore/redis.md)

## Java

* [Java — JVM · GC · 컬렉션 · 동시성](java/jvm-gc-concurrency.md)

## Spring

* [Spring — DI · AOP · 트랜잭션 · JPA · Batch](spring/di-aop-transaction-jpa.md)

## System Design

* [대용량 처리 · 분산 시스템](system-design/high-throughput.md)

## Fundamentals

* [자료구조](fundamentals/data-structures.md)

## OS

* [메모리와 페이징 — 없는 메모리를 있는 척하는 법](os/memory-and-paging.md)
* [프로세스와 스레드 — 컨텍스트 스위칭은 왜 비싼가](os/process-and-scheduling.md)
* [동시성 원시타입 — 동기/비동기와 블로킹/논블로킹](os/concurrency-primitives.md)
* [리눅스 — 장애 났을 때 뭘 보나](os/linux-troubleshooting.md)

## Infra

* [컨테이너 — 가상머신이 아니라 격리된 프로세스다](infra/containers.md)
* [Kubernetes — 선언한 상태로 계속 되돌리는 기계](infra/kubernetes.md)

## Network

* [계층 모델 — OSI 7계층은 왜 7개인가](network/layered-model.md)
* [전송 계층 — 연결은 왜 비싼가](network/transport-layer.md)
* [응용 계층 — HTTP 상태코드와 TLS](network/application-layer.md)
* [신뢰성 패턴 — 타임아웃과 멱등성](network/reliability-patterns.md)
