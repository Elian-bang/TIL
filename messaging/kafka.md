# 큐가 아니라 로그다

## 30초 요약

- Kafka는 큐라기보다 로그다. 소비해도 사라지지 않고 디스크에 순서대로 쌓인다. **모든 차이가 여기서 파생된다**
- **파티션은 순서 보장의 단위이자 병렬성의 단위다.** 같은 키는 같은 파티션으로 가고, 그 단위의 순서가 보장된다
- 컨슈머는 **파티션 수보다 많으면 논다.** 병렬성의 상한이 파티션 수다
- 유실과 중복은 결국 `acks`와 오프셋 커밋 시점의 문제다. 결론은 **at-least-once + 소비 측 멱등 처리**
- **exactly-once는 Kafka 안에서만 성립한다.** 외부 채널사에 쏘는 순간 보장이 밖으로 나간다

---

## 원리 — 왜 그런가

### 2-1. "로그"에서 전부 파생된다

한 문장으로 줄이면 이렇다. Kafka는 소비해도 사라지지 않는, **디스크에 순서대로 append되는 커밋 로그**다.

이 한 줄만 붙잡으면 나머지는 외울 필요가 없다. 전부 여기서 유도된다.

| Kafka의 특성 | "로그다"에서 어떻게 나오는가 |
|---|---|
| **소비해도 안 사라진다** | 로그는 읽는다고 지워지지 않는다. 지우는 기준은 소비 여부가 아니라 **retention(시간·크기)** |
| **컨슈머가 pull한다** | 브로커가 "어디까지 처리했는지"를 판정하지 않는다. 읽는 쪽이 요청마다 오프셋을 지정한다 |
| **리플레이가 된다** | 로그가 남아 있으니 오프셋을 되감으면 다시 읽힌다 |
| **여러 소비자가 각자 읽는다** | 로그 하나를 여러 그룹이 각자의 오프셋으로 읽는다 → 통계·과금·모니터링이 같은 데이터를 독립적으로 소비 |
| **순서는 파티션 안에서만** | 로그 파일 하나 안에서만 순서가 정의된다. 파일이 여럿(파티션)이면 파일 간 순서는 없다 |
| **병렬성 상한 = 파티션 수** | 로그 파일 하나를 두 명이 동시에 읽으면 순서가 깨진다 → 파일당 컨슈머 1명 |
| **압도적으로 빠르다** | append-only 순차 쓰기 (§2-3) |

> 말로 설명해야 한다면 이렇게 줄인다. "Kafka는 큐라기보다 **로그**입니다. 소비해도 남는다는 점 하나에서 pull 방식, 리플레이, 파티션 단위 순서 보장이 전부 파생됩니다."
> 이 한 문장이면 "책 목차를 외운 사람"과 "구조를 이해한 사람"이 갈린다.

### 2-2. 왜 그렇게 빠른가

"디스크는 느리다"는 통념을 뒤집는 부분이다.

1. **순차 쓰기(append-only)**: 랜덤 쓰기는 느리지만 순차 쓰기는 메모리 랜덤 접근에 필적한다. Kafka는 파일 끝에 덧붙이기만 한다
2. **OS 페이지 캐시에 의존**: JVM 힙에 캐시를 두지 않는다. GC 부담이 없고, 브로커가 재시작해도 캐시가 살아 있다
3. **zero-copy(`sendfile`)**: 디스크 → 페이지 캐시 → 소켓으로 커널 안에서 바로 보낸다. 애플리케이션 메모리로 복사하지 않는다
4. **배치 + 압축**: 프로듀서가 메시지를 모아서(`linger.ms`, `batch.size`) 한 번에 보낸다. 건당 네트워크 왕복을 없앤다

> 지엽적으로 보이지만 **"디스크에 쓰는데 왜 빠른가"** 가 Kafka 설계의 핵심이다. 본질은 1번(순차 쓰기)과 3번(제로 카피)이다.

### 2-3. 파티션에서 순서와 병렬성이 맞물린다

파티션은 토픽을 나눈 물리적 로그 파일이다. 그리고 여기서 **순서와 처리량이 정면으로 충돌**한다.

**(1) 순서 보장의 단위**
- 파티션 **안에서는** 쓴 순서 = 읽는 순서
- 파티션 **사이에는** 순서가 없다
- 프로듀서 파티셔닝: 키가 있으면 키의 해시로 파티션을 고른다(자바 클라이언트 구현은 `hash(key) % 파티션수`)
- 키가 없을 때는 라운드로빈이 아니라 **스티키**다. 공식 문서의 표현은 *"choose the sticky partition that changes when at least batch.size bytes are produced to the partition"*, 즉 한 파티션에 `batch.size`만큼 쌓일 때까지 그 파티션에 붙어 있다가 옮긴다
- → 같은 키는 항상 같은 파티션으로 가고, 그래서 같은 키끼리는 순서가 보장된다
- 발송 도메인 적용: 고객사 ID나 수신번호를 키로 잡으면 그 단위의 순서가 지켜진다. 전체 순서까지 필요한 경우는 드물고, 대개 **"이 수신자에게 가는 메시지들의 순서"** 정도면 충분하다

**(2) 병렬성의 단위**
- 같은 컨슈머 그룹 안에서 파티션 하나당 컨슈머 하나
- → 컨슈머를 파티션 수보다 많이 띄우면 남는 컨슈머는 논다
- → 처리량을 늘리려면 파티션을 늘려야 한다

**(3) 그런데 파티션을 늘리면 순서가 깨진다** ← 이 지적을 하면 깊이가 보인다
파티션 수가 바뀌면 `hash(key) % 파티션수`의 결과가 달라진다. 어제까지 3번 파티션에 가던 키가 오늘 7번으로 간다. 그러면 3번에 아직 안 읽힌 메시지와 7번의 새 메시지 사이에 순서가 보장되지 않는다.
→ **파티션 수는 늘릴 수만 있고 줄일 수 없으며, 늘리는 것 자체가 순서 보장을 깨는 이벤트다.** 공식 문서도 *"Kafka does not currently support reducing the number of partitions for a topic"* 이라고 못 박고, 파티션 추가에 대해서는 *"adding partitions doesn't change the partitioning of existing data so this may disturb consumers if they rely on that partition"* 이라고 경고한다. 그래서 초기 설계에서 여유 있게 잡는다.

**(4) 컨슈머 그룹**
- 같은 그룹은 일을 나눠 갖는다 (경쟁 소비)
- 다른 그룹은 같은 데이터를 각자 처음부터 읽는다
- → 발송 이벤트 하나를 통계·과금·모니터링·CRM이 각자 소비하는 구조가 그냥 따라 나온다. RabbitMQ 대비 Kafka의 가장 큰 구조적 이점이다

### 2-4. 리밸런싱, 운영에서 제일 자주 터진다

컨슈머가 들어오거나 나가면(배포·장애·타임아웃) 파티션을 재분배한다. 문제는 **그동안 소비가 멈춘다**는 것이다(eager rebalance = stop-the-world).

**리밸런싱이 자주 일어나는 원인**:
- `max.poll.interval.ms`(기본 300000, 5분) 초과. 공식 설명이 그대로다. *"If poll() is not called before expiration of this timeout, then the consumer is considered failed and the group will rebalance in order to reassign the partitions to another member."* **외부 API 호출처럼 느린 처리에서 흔하다.** 발송 도메인에 직결되는 부분이다
- 배포로 컨슈머가 계속 재시작

**대응**: `max.poll.records`(기본 500)를 줄여 한 번에 가져오는 양을 줄이거나, `max.poll.interval.ms`를 늘리거나, 처리를 비동기로 뺀다.
**개선된 방식**: Cooperative Sticky Assignor는 전체를 멈추지 않고 점진적으로 재분배한다. 최신 클라이언트에서는 `partition.assignment.strategy` 기본값이 `RangeAssignor,CooperativeStickyAssignor`라 이미 목록에 들어 있다.

### 2-5. 프로듀서 쪽 유실과 `acks`

| 값 | 의미 | 트레이드오프 |
|---|---|---|
| `acks=0` | 보내고 안 기다림 | 가장 빠름, **유실 가능** |
| `acks=1` | **리더만** 받으면 OK | 리더가 복제 전에 죽으면 유실 |
| `acks=all` | **ISR 전체**가 받아야 OK | 가장 안전, 느림 |

- ISR(In-Sync Replica)은 리더를 충분히 따라잡은 복제본 집합이다. 리더가 죽으면 여기서 새 리더를 뽑는다. `acks=all`의 공식 정의도 *"the leader will wait for the full set of in-sync replicas to acknowledge the record"* 다
- ⚠️ **`acks=all` 단독으로는 부족하다.** `min.insync.replicas` 기본값이 1이라, ISR이 1개로 줄어든 상태면 "ISR 전체"가 곧 리더 하나다. 리더가 죽으면 그대로 유실이다. 그래서 `min.insync.replicas=2`와 세트로 건다. *이 지적을 하면 확실히 판다*
- `enable.idempotence=true`를 켜면 재전송해도 로그에 중복이 안 남는다. 문서의 설명은 브로커가 프로듀서마다 ID를 부여하고, 메시지에 실려 오는 시퀀스 번호로 중복을 걸러 낸다는 것이다

**기본값은 버전마다 다르다. 이건 확인하고 답해야 한다.**

| | Kafka 2.8 | Kafka 4.1 |
|---|---|---|
| `acks` | `1` | `all` |
| `enable.idempotence` | `false` | `true` |
| `linger.ms` | `0` | `5` |

"기본이 `acks=1`이라 위험하다"는 설명은 2.x 시절 기준이다. 지금은 `acks`가 `all`이고 멱등 프로듀서가 기본으로 켜져 있으며, 둘은 묶여 있다. 문서 표현으로 *"Note that enabling idempotence requires this config value to be 'all'."*

### 2-6. 컨슈머 쪽 중복/유실과 오프셋 커밋 시점

**갈림길은 하나다: 처리와 커밋 중 뭐가 먼저인가.**

```
처리 전에 커밋 → 처리 중 죽으면 그 메시지는 영영 안 읽힌다   = 유실 (at-most-once)
처리 후에 커밋 → 커밋 전에 죽으면 다음에 다시 읽는다        = 중복 (at-least-once)
```

둘 중 하나를 반드시 골라야 한다. **중간은 없다.** 그리고 발송 도메인에서 유실은 절대 안 되므로 답은 at-least-once다.

- `enable.auto.commit`은 기본이 `true`이고, `auto.commit.interval.ms` 기본값인 5초마다 백그라운드에서 커밋한다. 이 주기는 "어디까지 처리했는지"와 무관하게 흐르므로, 발송처럼 사고가 나면 안 되는 도메인은 수동 커밋을 쓴다
- 커밋한 오프셋 자체는 컨슈머가 들고 있는 게 아니라 Kafka로 커밋된다. 읽는 위치를 정하는 주체가 컨슈머라는 것이지, 저장이 로컬이라는 뜻은 아니다
- **결론은 RabbitMQ 때와 똑같다. at-least-once로 받고, 소비 측에서 멱등하게 처리한다**

**Exactly-once는 왜 답이 아닌가**
- Kafka는 트랜잭션 API로 EOS를 제공하지만, Kafka 안에서 끝나는 경우(read-process-write)에 한정된다. 문서가 드는 예도 *"When consuming from a Kafka topic and producing to another topic (as in a Kafka Streams application)"* 이고, 원자적으로 묶이는 대상은 *"all the records it produces, and any offsets it updates on behalf of the consumer"* 다
- 외부 채널사에 실제로 문자를 쏘는 순간 **그 보장은 시스템 밖으로 나간다.** 채널사 API 호출은 Kafka 트랜잭션에 못 넣는다. 공식 문서도 외부 시스템에 쓸 때 남는 문제를 *"the need to coordinate the consumer's position with what is actually stored as output"* 이라고 적고, 고전적 해법으로 2단계 커밋을 든다
- → 결국 발송 이력 기준의 멱등 처리가 답이다. 발송 전에 (고객사, 캠페인, 수신번호) 유니크 키로 이력을 남기고, 이미 있으면 건너뛴다
- 중복 발송은 곧 고객사 과금 사고다. 이 도메인에서 가장 비싼 실수라는 걸 아는 사람으로 보여야 한다

### 2-7. Head-of-line blocking, DLT가 왜 필수인가

Kafka는 파티션 단위로 순차 처리한다. 그래서 **한 건이 계속 실패하면 그 뒤가 전부 막힌다.**

RabbitMQ는 이 문제가 약하다. 개별 메시지를 `nack`하면 그것만 DLX로 빠지고 나머지는 계속 흐른다. Kafka에는 그 장치가 기본으로 없다.

**대응 (Spring Kafka)**:
- `DefaultErrorHandler` + `DeadLetterPublishingRecoverer` → DLT(Dead Letter Topic)로 보낸다
- 재시도 정책도 직접 잡는다: `FixedBackOff` / `ExponentialBackOff`
- **"N번 실패하면 DLT로 넘기고 다음으로 진행"이 필수 설계**다

> 여기까지 봐야 설계가 끝난다. "Kafka는 DLQ가 없어서 직접 만든다"까지는 누구나 말한다. **"안 만들면 뒤가 다 막힌다"** 까지 말하는 게 차이다.

---

**다음으로 읽을 것**

- 큐를 왜 쓰는지, 무엇을 고를지 → [메시징 기초](basics.md)
- 개별 재시도·DLQ가 기본으로 오는 쪽 → [RabbitMQ](rabbitmq.md)
- 멱등 처리 설계 → [대용량 처리](../system-design/high-throughput.md) §2-6

---

> **기준 버전**: Apache Kafka 4.1 문서로 대조. 기본값이 바뀐 항목은 2.8 문서와 나란히 적었다
> **확인한 출처** (전부 kafka.apache.org):
> - [Producer Configs (4.1)](https://kafka.apache.org/41/configuration/producer-configs/) — **`acks` 기본 `all`**, **`enable.idempotence` 기본 `true`**, `linger.ms` 기본 5, `batch.size` 16384, `retries` 2147483647, `max.in.flight.requests.per.connection` 5, *"Note that enabling idempotence requires this config value to be 'all'."*, `acks=all`의 정의 *"the leader will wait for the full set of in-sync replicas to acknowledge the record"*, 기본 파티셔너의 **스티키 동작** 원문 *"If no partition or key is present, choose the sticky partition that changes when at least batch.size bytes are produced to the partition."* / [같은 표의 2.8판](https://kafka.apache.org/28/generated/producer_config.html) — **`acks` `1`**, **`enable.idempotence` `false`**, `linger.ms` `0`
> - [Consumer Configs (4.1)](https://kafka.apache.org/41/configuration/consumer-configs/) — **`enable.auto.commit` 기본 `true`**, **`auto.commit.interval.ms` 기본 5000**, `max.poll.records` 500, `max.poll.interval.ms` 300000, `session.timeout.ms` 45000, `heartbeat.interval.ms` 3000, `isolation.level` 기본 `read_uncommitted`, `partition.assignment.strategy` 기본 `RangeAssignor,CooperativeStickyAssignor`, `max.poll.interval.ms` 초과 시 *"the consumer is considered failed and the group will rebalance"*
> - [Broker Configs (4.1)](https://kafka.apache.org/41/configuration/broker-configs/) — **`min.insync.replicas` 기본 1**, ISR 미달 시 `NotEnoughReplicas` / `NotEnoughReplicasAfterAppend`, `unclean.leader.election.enable` 기본 `false`
> - [Design (4.1)](https://kafka.apache.org/41/design/design/) — 순차 쓰기 대 랜덤 쓰기 수치(*"600MB/sec ... about 100k/sec–a difference of over 6000X"*), 페이지 캐시 우선 설계(*"All data is immediately written to a persistent log on the filesystem without necessarily flushing to disk."*), 힙 캐시를 안 쓰는 이유(*"Java garbage collection becomes increasingly fiddly and slow as the in-heap data increases."*)와 재시작 후에도 캐시가 남는다는 서술, zero-copy(*"in Linux this is done with the sendfile system call"*), 배치·압축, 컨슈머가 요청마다 오프셋을 지정한다는 서술, 파티션당 그룹 내 컨슈머 1명(*"each partition is consumed by exactly one consumer within each subscribing consumer group at any given time"*), 멱등 프로듀서의 프로듀서 ID·시퀀스 번호, **EOS 범위**와 외부 시스템의 2단계 커밋 원문
> - [Introduction (4.1)](https://kafka.apache.org/41/getting-started/introduction/) — 같은 키가 같은 파티션으로 가고 *"Kafka guarantees that any consumer of a given topic-partition will always read that partition's events in exactly the same order as they were written"*, retention(*"events are not deleted after consumption"*)
> - [Basic Kafka Operations (4.1)](https://kafka.apache.org/41/operations/basic-kafka-operations/) — **파티션 축소 미지원** 원문과 파티션 추가 시 기존 데이터가 재분배되지 않는다는 경고
> **미확인**: Cooperative Sticky Assignor의 **도입 릴리스**(2.4로 알려져 있으나 4.1 문서에서 확인하지 못했고, 기본 전략 목록에 들어 있다는 것만 확인했다) · 리밸런싱이 stop-the-world라는 §2-4의 서술은 **eager 방식 설명이며 4.1 문서에서 원문을 찾지 못했다** · 멱등 프로듀서의 중복 제거가 **파티션 단위·세션 단위**라는 점(널리 알려져 있으나 문서가 명시하지 않아 본문에서 뺐다) · `hash(key) % 파티션수`라는 **구체적 계산식**(문서는 *"a hash of the key"* 까지만 말한다) · §2-7의 `DefaultErrorHandler`·`DeadLetterPublishingRecoverer`는 **Spring Kafka 쪽 API로, spring.io 문서를 대조하지 않았다**
> **미작성**: KRaft(ZooKeeper 제거)와 컨트롤러 쿼럼 · 로그 컴팩션 · Kafka Streams · 티어드 스토리지 · `transactional.id`와 좀비 펜싱의 실제 절차
