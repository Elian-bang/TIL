# 싱글 스레드가 왜 설계 선택인가

## 30초 요약

- Redis는 명령을 싱글 스레드로 처리한다. **한계가 아니라 선택이다.** 락이 필요 없고, 모든 명령이 저절로 원자적이 된다
- 그 대가로 **O(n) 명령 하나가 서버 전체를 멈춘다.** `KEYS *`는 운영에서 금지, `SCAN`을 쓴다
- 자료구조가 곧 용도다. **Sorted Set = 예약 발송**, `SET NX EX` = 멱등키, Set = 중복 제거
- 영속성이 완전하지 않다. RDB 스냅샷도 AOF도 유실 구간이 있어서 **과금·계약 데이터를 두면 안 된다**
- 분산락은 성능 최적화지 정합성의 최종 보장은 못 된다. **최종 방어선은 언제나 DB 유니크 제약이다**

---

## 원리 — 왜 그런가

### 2-1. 왜 빠른가, 그리고 왜 싱글 스레드인가

- 인메모리라서 빠르다 (1차 이유)
- **명령 처리는 싱글 스레드**다. 한계가 아니라 설계 선택이고, 그 대가로 얻는 게 있다:
  - 락이 필요 없다 → 락 획득·경합 비용 0
  - 컨텍스트 스위칭 없다
  - 모든 명령이 원자적이다 → `INCR`, `SETNX` 같은 게 그냥 안전하다
  - 병목이 CPU가 아니라 메모리와 네트워크라서 멀티스레드 이득이 작다. 공식 벤치마크 문서도 *"In many real world scenarios, Redis throughput is limited by the network well before being limited by the CPU."* 라고 적는다
  - Redis 6부터는 소켓 읽기·쓰기를 별도 스레드로 넘기는 멀티스레드 I/O가 들어왔다(`io-threads`, 기본값 1이라 꺼져 있다). 명령 실행 자체는 여전히 한 스레드다

**⚠️ 그래서 O(n) 명령 하나가 서버 전체를 멈춘다.**
- `KEYS *`는 전체 키를 훑는다(O(N)). 공식 문서도 *"Don't use `KEYS` in your regular application code."* 라고 못박고 커서 기반으로 나눠 도는 **`SCAN`** 을 권한다
- `FLUSHALL`, 큰 `HGETALL`, 큰 컬렉션 `DEL` 도 같은 위험
- 한 명령이 100ms 걸리면 그동안 모든 클라이언트가 대기한다. **이 지적을 하면 "운영을 아는구나"가 된다**

> [프로세스와 스레드](../os/process-and-scheduling.md) §2-1을 뒤집은 사례다. 거기서는 "스레드를 늘려도 안 빨라진다"였는데, Redis는 아예 **하나로 고정해서 동기화 비용 자체를 없앤다.** 병목이 CPU가 아닐 때 택할 수 있는 수다.

### 2-2. 자료구조가 곧 용도다

| 타입 | 발송 도메인에서 쓸 자리 |
|---|---|
| String | 캐시, 카운터(`INCR`), **멱등키(`SET key NX EX`)** |
| Hash | 객체 캐시 (발신프로필, 템플릿) |
| List | 간단한 큐 (`LPUSH`/`BRPOP`) |
| Set | **중복 수신자 제거** |
| **Sorted Set** | **예약 발송**. score를 발송 예정 시각으로 두고 `ZRANGE key min max BYSCORE`로 도래분만 꺼낸다 |
| **Stream** | 컨슈머 그룹이 있는 로그형 큐 (Kafka 축소판) |
| HyperLogLog | 대략적 유니크 카운트 (표준 오차 0.81%, 키당 최대 12KB) |

> ⚠️ `ZRANGEBYSCORE`는 Redis 6.2.0부터 deprecated다. `ZRANGE`가 `BYSCORE`·`BYLEX`·`REV` 옵션으로 `ZREVRANGE`, `ZRANGEBYSCORE`, `ZREVRANGEBYSCORE`, `ZRANGEBYLEX`, `ZREVRANGEBYLEX`를 모두 흡수했다. 동작은 같으니 새로 쓰는 코드는 `ZRANGE ... BYSCORE`를 쓴다.

### 2-3. 영속성, 왜 원장으로 못 쓰나

- **RDB 스냅샷**: 주기적으로 메모리 전체를 파일로 내린다. 공식 문서의 표현은 *"you should be prepared to lose the latest minutes of data"* 다. 마지막 스냅샷 이후가 통째로 날아간다
- **AOF(Append Only File)**: 명령을 로그로 기록한다. fsync 정책이 셋 있고(`always` / `everysec` / `no`), *"The suggested (and default) policy is to `fsync` every second."* 다. `everysec`이면 최대 1초치가 날아간다
- 둘 다 완전한 무손실이 아니다. 그래서 **과금·계약 데이터를 Redis에 두면 안 된다.** 캐시와 임시 상태용이다

> AOF는 [내구성과 복구](../database/basics/durability-and-recovery.md) §2-1의 WAL과 같은 구조다. `appendfsync everysec`는 거기서 본 **"내구성 손잡이를 한 칸 내린 상태"** 에 해당한다. 커밋마다 fsync하지 않고 1초에 한 번 몰아서 하는 것이다.

### 2-4. 캐시 전략과 3대 문제

- **Cache-Aside** (가장 흔함): 조회 → 없으면 DB → 캐시에 저장. 갱신할 때는 캐시를 지운다. 값을 새로 넣지 않고 삭제하는 게 정석인데, 경합 시 낡은 값이 남는 걸 막기 위해서다
- **Write-Through**: 쓸 때 캐시와 DB를 같이 쓴다. 일관성이 좋고 쓰기가 느리다
- **Write-Behind**: 캐시에 쓰고 DB는 나중에 반영한다. 빠르지만 유실 위험을 안는다

**3대 문제**

| 문제 | 무엇 | 대응 |
|---|---|---|
| **캐시 스탬피드** | 인기 키가 **동시에 만료**돼 요청이 전부 DB로 몰림 | TTL에 지터 추가, 뮤텍스로 하나만 갱신 |
| **캐시 관통(penetration)** | **없는 키**를 계속 조회 → 매번 DB까지 감 (공격 벡터이기도) | 빈 값도 짧게 캐싱, 블룸 필터 |
| **정합성** | DB는 바뀌었는데 캐시가 낡음 | 갱신 시 삭제, TTL을 짧게, 중요하면 캐시 안 씀 |

> TTL 지터는 [대용량 처리](../system-design/high-throughput.md) §2-5의 재시도 지터와 완전히 같은 장치다. **동시에 몰리는 것을 시간축에 흩뿌린다.**

### 2-5. 분산락으로 중복 발송 막기

```
SET lock:campaign:123 <고유값> NX EX 30    -- 없을 때만 생성 + TTL
```
- NX(없을 때만)와 TTL을 **한 명령으로** 걸어야 한다. 따로 걸면 그 사이에 죽었을 때 락이 영원히 남는다
- 값에는 클라이언트마다 다른 난수를 넣는다. 해제할 때 자기 락인지 확인한 뒤 삭제하기 위해서다(Lua 스크립트로 원자적으로). 그냥 `DEL`하면 TTL로 만료된 뒤 남의 락을 지운다
- 단일 인스턴스 방식은 공식 문서 기준으로 *"a viable solution in applications where a race condition from time to time is acceptable"* 다. 마스터가 죽고 레플리카가 승격되는 사이에 두 클라이언트가 같은 락을 쥘 수 있다
- **Redlock**(독립된 N개 노드에서 과반 획득)은 그 한계를 메우려는 알고리즘이고 redis.io가 직접 제안한다. 다만 같은 문서가 Kleppmann의 비판과 antirez의 반론을 나란히 링크해 둔다. 논쟁이 살아 있다는 뜻이니 단정하지 말 것
- ⚠️ 가장 중요한 인식은 이것이다. **분산락은 성능 최적화이지 정합성의 최종 보장이 아니다.** GC 정지나 네트워크 지연으로 락이 만료된 줄 모르고 진행할 수 있다. 공식 문서도 펜싱 토큰을 구현하라고 권하고, TTL 만료가 monotonic clock이 아니라 벽시계를 쓰기 때문에 시계가 틀어지면 락이 둘에게 잡힐 수 있다고 적는다. 최종 방어선은 DB 유니크 제약이다

---

**다음으로 읽을 것**

- 무엇을 Redis에 두고 무엇을 RDB에 둘 것인가 → [저장소 선택](selection.md)
- DB 락과의 비교 → [트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-8

---

> **기준 버전**: Redis 6 이상 가정(§2-1의 I/O 멀티스레드). 대조는 redis.io 현행 문서(`/docs/latest/`)로 했다
>
> **확인한 출처**:
> - [Redis benchmark](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/) — *"Redis is, mostly, a single-threaded server from the POV of commands execution (actually modern versions of Redis use threads for different things). It is not designed to benefit from multiple CPU cores."*, *"Being single-threaded, Redis favors fast CPUs with large caches and not many cores."*, *"In many real world scenarios, Redis throughput is limited by the network well before being limited by the CPU."*
> - [Talking to Redis](https://redis.io/tutorials/operate/redis-at-scale/talking-to-redis/) — §2-1의 **Redis 6 멀티스레드 I/O 범위**. *"in Redis version 6.0 multi-threaded I/O was introduced. When this feature is enabled, Redis can delegate the time spent reading and writing to I/O sockets over to other threads"* 즉 소켓 읽기·쓰기만 스레드로 넘어가고 명령 처리는 그대로다. `io-threads` 기본값이 1이라 기본 상태에서는 꺼져 있다
> - [`KEYS`](https://redis.io/docs/latest/commands/keys/) — 복잡도 O(N), *"Use extreme care when using this command in production environments."*, *"Don't use `KEYS` in your regular application code."*, `SCAN` 권고
> - [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/) — **`appendfsync` 기본값**이 *"The suggested (and default) policy is to `fsync` every second."* 라는 것, 세 정책(`always`/`everysec`/`no`)과 각각의 유실 폭, RDB의 *"you should be prepared to lose the latest minutes of data"*, *"Snapshotting is not very durable."*
> - [HyperLogLog](https://redis.io/docs/latest/develop/data-types/probabilistic/hyperloglogs/) — **오차율**. *"The Redis implementation uses up to 12 KB of memory and provides a standard error rate of 0.81%."* 원래 본문의 "~0.8%"를 0.81%로 고쳤다
> - [`SET`](https://redis.io/docs/latest/commands/set/) — `NX`/`XX`/`EX`/`PX`/`KEEPTTL` 옵션, `SET`이 `SETNX`·`SETEX`를 대체할 수 있다는 노트, 그리고 §2-5의 락 패턴 원문(*"The command `SET resource-name anystring NX EX max-lock-time` is a simple way to implement a locking system with Redis."*)과 값 비교 후 삭제하는 Lua 해제 스크립트
> - [Distributed Locks with Redis](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/) — 단일 인스턴스 방식이 *"a viable solution in applications where a race condition from time to time is acceptable"* 라는 것, 마스터 죽고 레플리카 승격 시의 SAFETY VIOLATION 시나리오, Redlock 알고리즘, "Disclaimer about consistency"의 **펜싱 토큰 권고**와 *"Redis is not using monotonic clock for TTL expiration mechanism."*, 그리고 Kleppmann 분석과 antirez 반론을 나란히 링크한다는 것
> - [`ZRANGEBYSCORE`](https://redis.io/docs/latest/commands/zrangebyscore/) — *"Deprecated as of Redis v6.2.0."* · [`ZRANGE`](https://redis.io/docs/latest/commands/zrange/) — *"Starting with Redis 6.2.0, this command can replace the following commands: `ZREVRANGE`, `ZRANGEBYSCORE`, `ZREVRANGEBYSCORE`, `ZRANGEBYLEX` and `ZREVRANGEBYLEX`."* §2-2의 명령을 이에 맞춰 고쳤다
>
> **미확인**:
> - **§2-1의 "싱글 스레드라서 락이 필요 없다 / 컨텍스트 스위칭이 없다"** — 공식 문서는 싱글 스레드라는 사실과 *"single-threading avoids race conditions and CPU-heavy context switching associated with threads"* 까지만 말한다. "그래서 얻는 게 있다"는 인과 서술은 이 문장에서 추론한 것이고, 락 획득 비용이 0이라는 정량 진술의 근거는 확인하지 못했다
> - **§2-1의 "한 명령이 100ms 걸리면 그동안 모든 클라이언트가 대기한다"** — 싱글 스레드 실행 모델에서 따라 나오는 결론이지만, 100ms라는 수치 자체는 예시일 뿐 공식 문서에서 가져온 값이 아니다
> - **§2-4의 캐시 3대 문제(스탬피드·관통·정합성)** — Redis 공식 문서의 개념이 아니라 캐시 일반론이다. redis.io에서 대응하는 페이지를 찾지 못했다. 블룸 필터는 Redis 모듈로 존재하지만 이 문서에서는 확인하지 않았다
> - **§2-2의 Stream이 "Kafka 축소판"이라는 비유** — 컨슈머 그룹이 있다는 것 외에 두 제품을 비교한 공식 서술은 확인하지 않았다
>
> **미작성**: `SCAN` 커서 동작과 보장 범위 · Redis Cluster의 해시 슬롯과 리샤딩 · Sentinel 페일오버 · `maxmemory-policy` 선택 기준 · Lua/Function 스크립팅 · Streams 컨슈머 그룹 실사용
