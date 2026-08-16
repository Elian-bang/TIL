# 큐를 왜 쓰고, 무엇을 고를 것인가

## 30초 요약

- 큐가 하는 일은 넷이다. **결합 분리, 피크 흡수, 재시도 지점, 독립 확장**
- 대신 값을 치른다. 비동기가 되는 순간 **"이 요청이 지금 어디까지 갔나"를 추적하는 설계가 반드시 따라온다**
- 브로커 논쟁은 대개 질문이 틀렸다. **Kafka와 RabbitMQ는 애초에 다른 물건이다**
- 고르는 기준은 제품의 우열에 있지 않다. **규모와 요구**가 정한다
- 한 줄만 확실히 하면 나머지는 유도된다. **소비하면 사라지느냐, 남느냐**

---

## 원리 — 왜 그런가

### 2-1. 메시지 큐를 왜 쓰나 (기본기 확인용)

발송 시스템에서 큐가 하는 일은 네 가지다. 설계 질문의 출발점이라 정리해 둔다.

1. 결합 분리: API 서버는 "발송 요청을 접수했다"까지만 책임지고 즉시 응답한다. 실제 발송은 컨슈머가 맡는다
2. 피크 흡수(버퍼): 캠페인 발송은 순간에 수백만 건이 몰린다. 큐가 없으면 그 부하가 그대로 DB와 채널사로 간다. 큐가 완충 장치다
3. 재시도 지점: 채널사가 죽었을 때 다시 시도할 자리가 필요하다
4. 독립 확장: 발송이 느리면 컨슈머만 늘린다

> 역으로, 큐를 쓰면 생기는 비용도 있다. 비동기가 되면서 "지금 이 요청이 어디까지 갔나"를 추적하기 어려워진다. **상태 관리와 결과 조회 설계가 반드시 따라온다.** 이 지적을 하면 설계 질문에서 앞서간다.


### 2-2. 브로커 선택은 우열이 아니라 규모의 문제다

브로커 논쟁은 대개 질문이 잘못됐다. Kafka와 RabbitMQ는 같은 자리를 놓고 겨루는 제품이 아니고, 선택을 가르는 것도 **규모와 요구**다.

일 1만 건 규모의 발송 큐라면 **ACK·개별 재시도·DLQ가 기본으로 오는 RabbitMQ**가 운영 비용 대비 합리적이다. 같은 자리에 Kafka를 얹으면 그 셋을 전부 직접 설계해야 한다([Kafka](kafka.md) §2-7). 어떤 규모에서도 Kafka가 과하다는 뜻은 아니고, 그 규모에서의 판단이라는 게 요점이다.

반대로 아래 조건이 붙기 시작하면 그때부터 Kafka가 맞는 선택이 된다.

**Kafka로 기우는 조건**:
- 캠페인 단위로 순간에 수백만 건이 몰린다 → 순차 쓰기·배치 기반 처리량
- 발송 이벤트 하나를 통계·과금·모니터링·CRM이 각자 소비한다 → 컨슈머 그룹 분리([Kafka](kafka.md) §2-3)
- 장애 후 특정 구간을 다시 처리해야 한다 → 오프셋 되감기
- 고객사·채널 단위 순서 보장이 필요하다 → 파티션 키

> 한계도 같이 적어 둔다. 개별 메시지 재시도·지연 큐·우선순위는 RabbitMQ처럼 기본으로 딸려 오지 않는다. 에러 핸들러와 DLT를 따로 설계해야 한다([Kafka](kafka.md) §2-7).
> **장점만 나열하면 외운 것, 한계를 붙이면 판단한 것이다.**


### 2-3. 브로커 비교표

외울 필요 없다. **"소비하면 사라지느냐 남느냐"** 한 줄만 확실히 해 두면 나머지는 거기서 유도된다([Kafka](kafka.md) §2-1).

| | RabbitMQ (실경험) | Kafka | ActiveMQ |
|---|---|---|---|
| 정체 | AMQP 브로커 (작업 큐) | 분산 커밋 로그 | JMS 구현체 |
| 소비 후 | **제거** | **남음** (retention) | 제거 |
| 소비 방식 | 브로커가 **push** (prefetch로 제어) | 컨슈머가 **pull** | push/pull |
| 리플레이 | 불가 | **가능** | 불가 |
| 순서 보장 | 큐 단위 | **파티션 단위** | 큐 단위 |
| 라우팅 | **Exchange** (direct/topic/fanout) | 토픽만 | 셀렉터 |
| 개별 재시도·DLQ | **기본 제공** (nack, DLX) | 직접 설계 (DLT) | 기본 제공 |
| 병렬성 상한 | 컨슈머 수 자유 | **파티션 수** | 컨슈머 수 자유 |
| 처리량 | 중간 | **최상** | 중간 |
| 쓰는 자리 | 작업 분배, 복잡한 라우팅 | 대용량 스트림, 다중 소비 | 레거시 JMS 자산 |

### 2-4. ActiveMQ는 한 문단이면 충분하다

- **JMS 표준 구현체**다. Queue(1:1) / Topic(pub-sub), 메시지 셀렉터(헤더 조건으로 골라 받기), durable subscriber, JMS 트랜잭션을 제공한다. 공식 문서 표기로는 *"fully supports JMS 1.1 and J2EE 1.4+"* 이고 JMS 2.0과 Jakarta Messaging 3.1은 부분 지원이다
- 프로토콜로 갈라진다고 외우면 틀린다. ActiveMQ Classic은 OpenWire 외에 **AMQP 1.0·MQTT·STOMP를 함께 지원**한다. RabbitMQ와의 차이는 프로토콜 계열이 아니라 JMS를 중심에 두느냐다. 큐·토픽·DLQ·재배달이라는 뼈대는 어차피 같다
- **Artemis**가 별도 계열로 갈라져 있다 (ActiveMQ Classic 5.x / ActiveMQ Artemis)
- 현업에서 마주치는 자리는 이렇다. 업력이 긴 조직에 ActiveMQ가 남아 있다면 대개 **기존 JMS 자산을 유지하기 위해서**다. Kafka와 병존한다면 "새 스트림은 Kafka, 레거시 연동은 JMS" 같은 분담인 경우가 많다

---

**다음으로 읽을 것**

- 로그에서 전부 파생되는 쪽 → [Kafka](kafka.md)
- 소비하면 사라지는 쪽 → [RabbitMQ](rabbitmq.md)
- 큐를 앞에 두는 설계 전반 → [대용량 처리](../system-design/high-throughput.md)

---

> **기준 버전**: 제품 버전에 의존하지 않는 판단 프레임. 표의 근거만 각 제품 최신 문서(Kafka 4.1 · RabbitMQ 4.3 · ActiveMQ Classic)로 대조했다
> **확인한 출처**:
> - [Kafka Introduction (4.1)](https://kafka.apache.org/41/getting-started/introduction/) — **소비해도 안 지워진다**는 원문(*"events are not deleted after consumption"*)과 retention 설정, 같은 키가 같은 파티션으로 가며 파티션 단위 순서가 보장된다는 원문 / [Kafka Design (4.1)](https://kafka.apache.org/41/design/design/) — **pull 방식**, 오프셋 되감기, **파티션당 그룹 내 컨슈머 1명**(*"each partition is consumed by exactly one consumer within each subscribing consumer group at any given time"*) → 표의 "병렬성 상한 = 파티션 수"
> - [RabbitMQ AMQP 0-9-1 Model](https://www.rabbitmq.com/tutorials/amqp-concepts) — 익스체인지 4종(direct/topic/fanout/headers) → 표의 "라우팅: Exchange", 브로커가 push하고 ack 없으면 재배달한다는 서술 / [Dead Letter Exchanges](https://www.rabbitmq.com/docs/dlx) — **DLQ가 브로커 기본 기능**이라는 근거 → 표의 "개별 재시도·DLQ 기본 제공"
> - [ActiveMQ Classic](https://activemq.apache.org/components/classic/) — *"fully supports JMS 1.1 and J2EE 1.4+"*, Jakarta Messaging 3.1 · JMS 2.0 부분 지원, **OpenWire·STOMP·AMQP v1.0·MQTT v3.1 동시 지원**(§2-4의 "프로토콜 계열이 다르다"는 서술을 이 근거로 고쳤다)
> **미확인**: §2-3 표의 **처리량 등급**(중간/최상)은 벤치마크가 아니라 감각 표기다 · ActiveMQ의 "소비 후 제거" · "리플레이 불가" · "push/pull" 행은 JMS 일반 모델에서 유도한 것이고 **activemq.apache.org 원문으로 대조하지 않았다** · **Artemis가 Classic의 후속이자 성능 개선판**이라는 통설은 도달 가능한 공식 페이지에서 확인하지 못해 본문에서 "별도 계열"로만 적었다 · §2-2의 "일 1만 건 규모면 RabbitMQ" 같은 규모 판단은 **경험 기반이고 출처가 없다**
> **미작성**: Redis Streams · AWS SQS/SNS · Google Pub/Sub 같은 관리형 선택지 · 아웃박스 패턴과 트랜잭셔널 메시징
