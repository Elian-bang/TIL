# 소비하면 사라진다

## 30초 요약

- AMQP 브로커이자 작업 큐(task queue)다. Kafka와 달리 **소비하면 사라진다**
- 그래서 **개별 메시지 ACK·재시도·DLQ가 기본으로 딸려 온다.** Kafka에서는 직접 설계해야 하는 것들이다
- 대신 **처리량은 Kafka가 압도적**이다. 메시지를 남기지 않으니 재처리·재생도 안 된다
- 라우팅이 유연하다. 익스체인지 타입으로 팬아웃·토픽·다이렉트를 골라 쓴다
- 선택 기준은 [메시징 기초](basics.md) §2-2에 있다. 제품의 우열보다 **규모와 요구**가 정한다

---

## 원리 — 왜 그런가

### 2-1. RabbitMQ

**정체**: AMQP 브로커이자 작업 큐(task queue)다. Kafka와 달리 소비하면 사라진다.

**구조**
- Exchange → (binding) → Queue → Consumer. 프로듀서는 큐가 아니라 **exchange에 보낸다**
- exchange 타입: `direct`(라우팅 키 일치) / `topic`(패턴 매칭) / `fanout`(전부 복사) / `headers`
- → 브로커가 라우팅을 해준다. Kafka는 안 해준다(토픽만 있다). **RabbitMQ가 유연한 지점**

**신뢰성 장치 — 실제 설정**
| 장치 | 무엇을 막나 |
|---|---|
| **publisher confirm** | 프로듀서가 보낸 게 브로커에 **정말 도착했는지** 확인. 없으면 보내고 유실돼도 모른다 |
| **durable queue + persistent message** | **브로커가 재시작해도** 메시지가 남는다. 둘 다 켜야 의미가 있다 |
| **수동 ack (`basicAck`)** | 처리 완료 후 ack. ack 전에 채널이나 커넥션이 닫히면 **자동으로 재입큐된다** |
| **`nack` + DLX** | `basic.reject` 또는 `basic.nack`을 `requeue=false`로 주면 **Dead Letter Exchange**로 간다. 개별 메시지 단위로 동작한다 |
| **prefetch (QoS)** | 채널에 **미확인(unacked) 상태로 허용되는 최대 개수**. 이게 백프레셔다 |

durable과 persistent를 둘 다 켜야 한다는 건 관행이 아니라 문서에 적힌 동작이다. *"Durable queues will be recovered on node boot, including messages in them published as persistent. Messages published as transient will be discarded during recovery, even if they were stored in durable queues."* 큐만 durable로 잡고 메시지를 persistent로 안 주면 재시작 때 그냥 사라진다.

DLX로 빠지는 조건은 nack만이 아니다. 문서가 드는 경우는 넷이다. `requeue=false`로 거부, 메시지별 TTL 만료, 큐 길이 제한 초과, 그리고 쿼럼 큐에서 `delivery-limit`을 넘긴 재배달이다. 무한 재시도를 끊는 장치가 브로커 쪽에 이미 있다는 뜻이다.

**prefetch를 왜 조절하나** (자주 나오는 질문이다)
- RabbitMQ는 브로커가 push한다. 컨슈머가 처리 중이든 말든 밀어 넣는다
- 정확히는 미확인 개수의 상한이다. *"When the number reaches the configured count, RabbitMQ will stop delivering more messages on the channel until at least one of the outstanding ones is acknowledged."* 여기서 멈추는 게 백프레셔다
- 한 가지 함정. AMQP 명세는 채널 전체에 걸쳐 공유되는 값으로 정의하지만, RabbitMQ는 채널의 **컨슈머마다 따로** 적용한다. 한 채널에 컨슈머를 여럿 붙였다면 실제 상한은 설정값의 배수가 된다
- prefetch가 크면 한 컨슈머가 메시지를 잔뜩 쥐고 있게 된다. 다른 컨슈머는 놀고, 그 컨슈머가 죽으면 전부 재배달된다
- → **처리 시간이 긴 작업일수록 prefetch를 작게** 잡는다. 발송처럼 외부 API를 기다리는 작업이 여기 해당한다

---

**다음으로 읽을 것**

- 어느 것을 고를 것인가 → [메시징 기초](basics.md) §2-2, §2-3
- 소비해도 사라지지 않는 쪽 → [Kafka](kafka.md)

---

> **기준 버전**: rabbitmq.com 4.3 문서로 대조. 본문의 AMQP 0-9-1 모델은 3.x와 4.x가 같다
> **확인한 출처** (전부 rabbitmq.com):
> - [AMQP 0-9-1 Model Explained](https://www.rabbitmq.com/tutorials/amqp-concepts) — 프로듀서가 큐가 아니라 익스체인지로 보낸다는 원문(*"messages are published to exchanges ... Exchanges then distribute message copies to queues using rules called bindings."*), **익스체인지 4종**의 정의(direct는 라우팅 키 일치, **fanout은 라우팅 키를 무시**, topic은 패턴 매칭, headers는 헤더 속성), ack 없이 컨슈머가 죽으면 재배달된다는 서술, prefetch의 목적, durable 큐 메타데이터가 디스크에 있다는 서술
> - [Consumer Acknowledgements and Publisher Confirms](https://www.rabbitmq.com/docs/confirms) — 순정 AMQP 0-9-1에서는 트랜잭션 말고 유실 방지 수단이 없다는 서술(publisher confirm이 존재하는 이유), automatic 대 manual ack, **미확인 상태로 채널·커넥션이 닫히면 자동 재입큐**, 재배달에 `redeliver` 플래그가 붙는다는 점, prefetch 정의 원문, `basic.nack`/`basic.reject` + `requeue=false` → DLX 없으면 폐기
> - [Consumer Prefetch](https://www.rabbitmq.com/docs/consumer-prefetch) — prefetch가 **채널의 미확인 배달 수 상한**이라는 점, AMQP 명세와 달리 RabbitMQ는 **채널의 컨슈머마다 따로 적용**한다는 차이, `rabbit.default_consumer_prefetch` 설정 존재
> - [Dead Letter Exchanges](https://www.rabbitmq.com/docs/dlx) — **데드레터 조건 4종**(`requeue=false` 거부 / 메시지별 TTL 만료 / 큐 길이 제한 초과 / 쿼럼 큐 `delivery-limit` 초과), `x-dead-letter-exchange` 인자와 정책 설정, 하드코딩된 `x-arguments`를 권장하지 않는다는 서술
> - [Queues](https://www.rabbitmq.com/docs/queues) — **durable 큐 + persistent 메시지를 둘 다 켜야 한다**는 원문(transient로 발행한 메시지는 durable 큐에 있어도 복구 시 폐기), 클래식 큐의 미러링이 **4.x에서 제거**됐다는 서술
> **미확인**: "prefetch가 1이면 균등 분배지만 왕복이 잦아 처리량이 떨어진다"는 **통용되는 설명이나 Consumer Prefetch 문서에서 원문을 못 찾아 본문에서 뺐다** · prefetch의 **시스템 기본값**(문서는 `default_consumer_prefetch` 설정이 있다는 것만 말하고 값을 밝히지 않는다) · §2-1 표의 "소비하면 사라진다"는 ack 이후 큐에서 제거된다는 뜻으로 쓴 것이고, **제거 시점을 명시한 원문은 대조하지 못했다** · [메시징 기초](basics.md) §2-3 비교표의 RabbitMQ 처리량 등급은 **실측이 아니라 감각 표기**다
> **미작성**: 쿼럼 큐와 스트림(4.x에서 사실상 표준이 된 큐 타입) · 클러스터링과 파티션 내성 · lazy queue · 지연 메시지 플러그인 · 관리 UI와 모니터링 지표
