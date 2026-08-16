# DI · AOP · 트랜잭션 · JPA · Batch

## 30초 요약

- **`@Transactional`과 AOP는 프록시가 가로채서 구현된다.** 그래서 프록시를 안 거치는 호출에는 안 걸린다. 함정 3종이 전부 여기서 나온다
- **생성자 주입을 쓰는 이유**: `final` 보장 · 순환참조를 기동 시점에 발견 · 테스트에서 목 주입이 쉬움
- **영속성 컨텍스트**: 1차 캐시 + 더티 체킹. `set`만 해도 커밋 시점에 UPDATE가 나간다
- **N+1**: 지연 로딩 때문에 연관 엔티티를 건당 조회. 해결은 fetch join / `@EntityGraph` / `default_batch_fetch_size` / DTO 직접 조회
- **Spring Batch에서 chunk 크기가 곧 트랜잭션 단위**다 → 메모리·락 점유·재처리 단위가 이 값 하나로 같이 움직인다
- ⚠️ **트랜잭션 안에서 외부 API를 호출하는 것**이 발송 도메인 최대 사고다. 커넥션과 락을 쥔 채 네트워크를 기다린다

---

## 원리 — 왜 그런가

### 2-1. DI / IoC, 왜 생성자 주입인가

**IoC(제어의 역전)**: 객체를 내가 `new` 하지 않고 컨테이너가 만들어 넣어준다.
**DI(의존성 주입)**: 그 넣어주는 행위.

**왜 좋은가**: 의존 대상을 구현체가 아니라 인터페이스로 받게 되므로, 바꿔 끼울 수 있고 테스트에서 가짜를 넣을 수 있다.

**생성자 주입 vs 필드 주입(`@Autowired` 필드)**

| | 생성자 주입 | 필드 주입 |
|---|---|---|
| **불변성** | `final` 선언 가능 → **한 번 정해지면 안 바뀜** | 불가능 |
| **순환참조** | **기동 시점에 즉시 실패** (A가 B를, B가 A를 생성자로 요구 → 만들 수가 없다) | 기동은 되고 **런타임에 터진다** |
| **테스트** | `new Service(mock)` 으로 **Spring 없이** 테스트 가능 | 리플렉션이나 컨테이너 필요 |
| **의존성이 많아지면** | 생성자 파라미터가 길어져 **"이 클래스가 너무 많은 일을 한다"가 눈에 보인다** | 필드만 늘어나 안 보인다 |

마지막 항목이 의외로 중요하다. 생성자 주입은 **설계 냄새를 드러내는 장치**이기도 하다. 파라미터가 계속 늘어나면 그 클래스가 너무 많은 일을 한다는 신호다.

### 2-2. AOP가 프록시로 구현된다는 것의 의미

**AOP**: 로깅·트랜잭션·보안처럼 여러 곳에 흩어지는 공통 관심사를 한 곳으로 모은다.

**Spring이 이걸 구현하는 방식은 프록시다.** 진짜 객체 앞에 대리인을 세우고, 호출이 대리인을 거치게 한다.

```
호출자 → [프록시] → (부가 기능: 트랜잭션 시작) → [진짜 객체] → (트랜잭션 커밋)
```

**두 가지 프록시**
- **JDK 동적 프록시**: 인터페이스 기반. 인터페이스를 구현한 대리 객체를 만든다
- **CGLIB**: 클래스 상속 기반. 대상 클래스를 상속한 자식을 만들어 메서드를 오버라이드한다
- **Spring Boot는 CGLIB이 기본**이다. `AopAutoConfiguration` 문서에 *"The proxyTargetClass attribute will be true, by default, but can be overridden by specifying spring.aop.proxy-target-class=false"*라고 적혀 있다

⚠️ **Spring Framework 자체의 기본은 다르다.** 부트가 값을 뒤집어 놓은 것이지, 프레임워크 규칙이 CGLIB인 게 아니다. 프레임워크 문서는 *"If the target object to be proxied implements at least one interface, a JDK dynamic proxy is used"*이고 *"If the target object does not implement any interfaces, a CGLIB proxy is created"*라고 말한다. 부트 밖에서 같은 코드를 돌리면 프록시 종류가 바뀔 수 있다.

여기서 제약이 나온다. 함정 3종의 근원이다.
- CGLIB은 상속으로 오버라이드한다 → **`private` 메서드는 오버라이드할 수 없다**. 문서 표현으로 *"`private` methods cannot be advised, because they cannot be overridden."*
- 프록시는 바깥에서 들어오는 호출만 가로챈다 → **객체 내부에서 자기 메서드를 부르면 프록시를 안 거친다**. *"self invocation is not going to result in the advice associated with a method invocation getting a chance to run."*
- `final` 클래스·메서드도 오버라이드 불가. *"`final` classes cannot be proxied, because they cannot be extended."*
- 하나 더 있다. 다른 패키지에 있는 부모 클래스의 패키지 전용 메서드는 *"effectively private"*라 어드바이스가 안 걸린다

> 말로 하면 이렇다. "Spring의 트랜잭션은 AOP고, AOP는 프록시로 구현됩니다. **함정들은 전부 '프록시를 안 거치면 안 걸린다'는 한 문장에서 나옵니다.**"
> 셋을 따로 외우는 것보다 하나에서 유도하는 게 낫다.

### 2-3. `@Transactional` 함정

#### (1) self-invocation, 같은 클래스 내부 호출

```java
@Service
public class SendService {
    public void sendAll(List<Target> targets) {
        for (Target t : targets) {
            sendOne(t);          // ❌ 프록시를 안 거친다 → 트랜잭션 안 걸림
        }
    }
    @Transactional
    public void sendOne(Target t) { ... }
}
```
`this.sendOne(t)`는 **진짜 객체가 자기 자신을 직접 부르는 것**이다. 프록시는 그 사이에 없다.
레퍼런스 문서가 이 상황을 그대로 서술한다. *"In proxy mode (which is the default), only external method calls coming in through the proxy are intercepted. This means that self-invocation (in effect, a method within the target object calling another method of the target object) does not lead to an actual transaction at runtime even if the invoked method is marked with `@Transactional`."*
**해법**: 다른 빈으로 분리하거나, 자기 자신을 주입받거나(`ApplicationContext`), `TransactionTemplate`을 쓴다. 가장 깨끗한 건 클래스를 분리하는 것이다.

#### (2) `private` 메서드에는 안 걸린다
CGLIB이 오버라이드할 수 없다. **조용히 안 걸린다.** 에러도 안 나서 더 위험하다.

⚠️ 여기서 "public이 아니면 전부 안 걸린다"로 외우면 지금은 틀린다. **Spring 6.0부터 클래스 기반 프록시에서는 `protected`와 패키지 전용 메서드에도 트랜잭션이 걸린다.** 문서 표현은 이렇다. *"As of 6.0, `protected` or package-visible methods can also be made transactional for class-based proxies by default. Note that transactional methods in interface-based proxies must always be `public` and defined in the proxied interface."* 안 걸리는 건 `private`이고, 인터페이스 기반 프록시는 여전히 `public`만 된다.

#### (3) 기본은 unchecked 예외만 롤백
```java
@Transactional
public void send() throws IOException {   // ❌ checked exception은 롤백 안 됨
    ...
}
```
- 기본 동작: **`RuntimeException`과 `Error`만 롤백**. `Exception`(checked)은 커밋된다
  → 문서 원문. *"In its default configuration, the Spring Framework's transaction infrastructure code marks a transaction for rollback only in the case of runtime, unchecked exceptions. That is, when the thrown exception is an instance or subclass of `RuntimeException`. (`Error` instances also, by default, result in a rollback)."* 그리고 *"Checked exceptions that are thrown from a transactional method do not result in a rollback in the default configuration."*
- **해법**: `@Transactional(rollbackFor = Exception.class)`
- 왜 이렇게 설계됐나: checked 예외는 "호출자가 처리할 수 있는, 예상된 상황"이라는 Java의 관례를 따른 것. **관례일 뿐 정답은 아니라서** 실무에서는 대체로 `rollbackFor`를 명시한다

#### (4) ⚠️⚠️ 추가 함정, 트랜잭션 안에서 외부 API 호출

```java
@Transactional
public void send(Target t) {
    history.save(...);              // DB
    channelApi.send(t);             // ❌ 외부 채널사 API — 200ms? 5초? 타임아웃?
    history.updateStatus(...);      // DB
}
```

**무엇이 나쁜가:**
- **DB 커넥션을 쥔 채로 네트워크를 기다린다** → 커넥션 풀이 빠르게 고갈된다 ([트랜잭션 · 락](../database/basics/transaction-and-lock.md) §2-10)
- **락도 쥐고 있다** → 락 점유 시간이 외부 응답 시간만큼 늘어난다 → 데드락 확률 급증 (§2-7 원칙 1의 정면 위반)
- **외부 호출이 성공한 뒤 DB가 롤백되면** → 문자는 나갔는데 이력은 없다 → 재시도 시 중복 발송

**해법은 트랜잭션 경계를 외부 호출 밖으로 빼는 것이다.**
```
[트랜잭션 1] 발송 이력을 SENDING 상태로 기록 (중복 방지 유니크 키)
    ↓ (트랜잭션 밖)
채널사 API 호출
    ↓
[트랜잭션 2] 결과로 상태 갱신 (REQUIRES_NEW 고려)
```
→ 내가 한 "조회를 트랜잭션 밖으로 분리"와 정확히 같은 원리다. 코드 리뷰 문제로 나오면 **1순위로 지적할 것.**

⚠️ 다만 `REQUIRES_NEW`를 커넥션 고갈의 해법처럼 말하면 안 된다. **이 옵션 자체가 커넥션을 하나 더 쓴다.** 바깥 트랜잭션이 자원을 쥔 채로 안쪽이 새 커넥션을 따로 얻기 때문이다. 레퍼런스 문서가 경고까지 적어 뒀다. *"This may lead to exhaustion of the connection pool and potentially to a deadlock if several threads have an active outer transaction and wait to acquire a new connection for their inner transaction"*, 그래서 *"Do not use `PROPAGATION_REQUIRES_NEW` unless your connection pool is appropriately sized, exceeding the number of concurrent threads by at least 1."* 위 그림에서 트랜잭션 2는 바깥 트랜잭션이 이미 끝난 자리에 놓여야 값어치가 있다.

### 2-4. 빈 스코프와 생명주기

- **기본은 singleton**이다. 컨테이너당 하나뿐이라 빈에 상태를 두면 안 된다(스레드 안전성 문제). 상태는 메서드 지역 변수나 파라미터로 넘긴다
- `prototype`(요청마다 새로), `request`/`session`(웹)
- ⚠️ **singleton이 prototype을 주입받으면** prototype이 한 번만 생성된다 (주입 시점에 한 번). 의도한 동작이 아니면 `ObjectProvider`나 `@Lookup`
- 생명주기: 생성 → 의존성 주입 → `@PostConstruct` → 사용 → `@PreDestroy` → 소멸

### 2-5. JPA 영속성 컨텍스트는 왜 있나

엔티티를 담아 두는 1차 캐시다. 그런데 캐시가 목적이 아니다. **진짜 목적은 세 가지다.**

**(1) 동일성 보장 (1차 캐시)**
같은 트랜잭션 안에서 같은 ID를 두 번 조회하면 **DB에 두 번 안 간다.** 그리고 같은 객체 인스턴스가 반환된다(`==` 성립).

**(2) 더티 체킹(변경 감지)** ← 가장 중요
```java
@Transactional
public void rename(Long id, String name) {
    Member m = repo.findById(id).get();
    m.setName(name);      // save() 를 부르지 않아도
}                          // 커밋 시점에 UPDATE 가 나간다
```
영속성 컨텍스트는 조회 시점의 스냅샷을 갖고 있다가, flush 시점에 **현재 상태와 비교해서 바뀐 필드만** UPDATE한다.
→ ⚠️ **부작용**: 의도치 않게 바꾼 값도 저장된다. 조회 전용이면 `@Transactional(readOnly = true)`를 붙인다 (스냅샷을 안 떠서 메모리도 절약된다)

**(3) 쓰기 지연**
INSERT/UPDATE를 모아 뒀다가 flush 시점에 한 번에 보낸다 → 배치로 묶을 수 있다.

**flush가 일어나는 시점**
1. **트랜잭션 커밋 직전**
2. **JPQL/쿼리 실행 직전.** 아직 반영 안 된 변경이 쿼리 결과에 빠지면 안 되니까
3. `flush()` 직접 호출

### 2-6. N+1은 왜 생기고 어떻게 푸나

```java
List<Campaign> campaigns = repo.findAll();        // 쿼리 1번
for (Campaign c : campaigns) {
    c.getTargets().size();                        // 캠페인마다 쿼리 1번 → N번
}
```

**원인은 지연 로딩(LAZY)이다.** 연관 엔티티를 처음 쓰는 순간 그때 조회한다. 리스트를 돌면 건마다 나간다.
"즉시 로딩(EAGER)으로 바꾸면?" **더 나쁘다.** 안 쓰는 연관까지 항상 가져오고, N+1도 여전히 난다(조회 방식에 따라). 연관관계는 전부 LAZY가 원칙이고, 필요한 곳에서 명시적으로 함께 가져온다.

**해결책 4가지, 상황별로 다르다**

| 방법 | 어떻게 | 언제 |
|---|---|---|
| **fetch join** | `JOIN FETCH`로 한 번에 | 연관을 확실히 쓸 때 |
| **`@EntityGraph`** | 어노테이션으로 fetch join | 리포지토리 메서드 단위 |
| **`default_batch_fetch_size`** | 지연 로딩을 **`IN (...)` 하나로** 묶어 조회 | **컬렉션 + 페이징** ★ |
| **DTO 직접 조회** | 필요한 컬럼만 select | 조회 전용 화면 |

**⚠️ fetch join + 페이징 함정**
컬렉션을 fetch join하면 **결과 행이 뻥튀기된다**(캠페인 1건 × 대상 100건 = 100행). 그러면 DB에서 `LIMIT`을 걸 수가 없다. 100행을 자르면 캠페인이 잘리니까.
→ **하이버네이트가 전부 가져와서 메모리에서 페이징한다.** 경고 로그(`firstResult/maxResults specified with collection fetch; applying in memory`)가 뜨고, 데이터가 크면 그대로 OOM이다.
→ **정석은 `default_batch_fetch_size`로 푸는 것.** 부모를 페이징으로 가져오고, 자식은 `IN` 쿼리 하나로 묶어 온다.

하이버네이트도 이걸 알고 있고, 문서에 같은 말을 적어 뒀다. *"When pagination is used in combination with a `fetch join` applied to a collection or many-valued association, the limit must be applied in-memory instead of on the database. This typically has terrible performance characteristics, and should be avoided."* 그런데 **막아 주지는 않는다.** `hibernate.query.fail_on_pagination_over_collection_fetch`를 켜면 예외로 바꿀 수 있지만 기본값은 꺼짐이고, 설명이 노골적이다. *"By default, the exception is disabled, and the possibility of terrible performance is left as a problem for the client to avoid."* 팀 차원에서 이 옵션을 켜 두는 것이 로그를 눈으로 찾는 것보다 낫다.

`default_batch_fetch_size`는 개별 `@BatchSize`를 전역 기본값으로 올리는 스위치다. *"Specifies the default value for batch fetching. By default, Hibernate only uses batch fetching for entities and collections explicitly annotated `@BatchSize`."*

### 2-7. Spring Batch는 chunk가 곧 트랜잭션 단위다

**구조**: `Job` → `Step` → chunk (`ItemReader` → `ItemProcessor` → `ItemWriter`)

```
reader 로 chunk 크기만큼 읽고 → processor 로 가공 → writer 로 한 번에 쓰고 → 커밋
                                                                    ↑
                                                   여기가 트랜잭션 경계
```

레퍼런스 문서의 한 문장이 이 구조를 그대로 말한다. *"Chunk oriented processing refers to reading the data one at a time and creating 'chunks' that are written out within a transaction boundary. Once the number of items read equals the commit interval, the entire chunk is written out by the `ItemWriter`, and then the transaction is committed."*

**★ chunk 크기가 곧 트랜잭션 크기다.** 그래서 하나의 손잡이가 세 가지에 동시에 영향을 준다:

| chunk를 키우면 | chunk를 줄이면 |
|---|---|
| 커밋 횟수 ↓ → **처리 속도 ↑** | 커밋 횟수 ↑ → 처리 속도 ↓ |
| 트랜잭션 길어짐 → **락 점유 ↑ → 데드락 ↑** | 락 점유 ↓ → **데드락 ↓** |
| 한 번에 들고 있는 객체 ↑ → **메모리 ↑** | 메모리 ↓ |
| 실패 시 롤백 범위 ↑ | 롤백 범위 ↓ |

**→ 그래서 chunk 크기는 성능 손잡이이자 안정성 손잡이다.** OOM([Java](../java/jvm-gc-concurrency.md))과 락 경합([트랜잭션 · 락](../database/basics/transaction-and-lock.md))이 같은 값 하나로 동시에 움직인다는 뜻이기도 하다.

**그 외 알아둘 것**
- **재시작(restart)**: `JobRepository`가 실행 이력을 저장해서, 실패한 지점부터 다시 시작할 수 있다
- **skip / retry**: 특정 예외는 건너뛰거나 재시도하도록 선언
- **`JpaPagingItemReader`의 함정**: 페이징으로 읽으면서 읽는 대상의 상태를 바꾸면 페이지가 밀린다(2페이지를 읽을 때 1페이지가 조건에서 빠져나감) → 커서 기반이나 상태 컬럼 설계로 회피. **발송 배치에서 흔한 버그**

---

> **기준 버전**: Spring Framework 6.x · Spring Boot 3.x · Hibernate ORM 6.4. §2-3 (2)의 가시성 규칙과 §2-2의 프록시 기본값이 이 버전에 묶인다
> **확인한 출처**:
> - [Using @Transactional (Spring Framework)](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html) — §2-3 (1) self-invocation 원문 전체, §2-3 (2)의 가시성 규칙(**6.0부터 `protected`·패키지 전용도 클래스 기반 프록시에서 트랜잭션 대상**, 인터페이스 기반은 `public`만)
> - [Rolling Back a Declarative Transaction](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html) — §2-3 (3). *"marks a transaction for rollback only in the case of runtime, unchecked exceptions"*, *"(`Error` instances also, by default, result in a rollback)"*, checked 예외는 롤백하지 않는다는 문장
> - [Transaction Propagation](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html) — §2-3 (4)에 추가한 경고. `REQUIRES_NEW`가 *"always uses an independent physical transaction"*이고 *"acquires its own resources such as a new database connection"*이라는 것, 커넥션 풀 고갈·데드락 경고와 *"exceeding the number of concurrent threads by at least 1"* 권고. `REQUIRED`의 rollback-only 전파와 `UnexpectedRollbackException`도 같은 문서에서 확인했다
> - [Proxying Mechanisms (Spring Framework AOP)](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html) — §2-2. **프레임워크 기본은 "인터페이스가 있으면 JDK 프록시"**라는 것, CGLIB 제약 원문(`final` 클래스·`final` 메서드·`private` 메서드·다른 패키지의 패키지 전용 메서드), self-invocation이 어드바이스를 못 태운다는 문장
> - [`AopAutoConfiguration` (Spring Boot javadoc)](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/autoconfigure/aop/AopAutoConfiguration.html) — §2-2의 **부트 기본값이 CGLIB**이라는 근거. *"The proxyTargetClass attribute will be true, by default"*
> - [Chunk-oriented Processing (Spring Batch)](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing.html) — §2-7의 *"written out within a transaction boundary"*와 commit interval 원문
> - [`QuerySettings` (Hibernate ORM 6.4 javadoc)](https://docs.hibernate.org/orm/6.4/javadocs/org/hibernate/cfg/QuerySettings.html) — §2-6의 fetch join + 페이징. *"the limit must be applied in-memory instead of on the database"*, `fail_on_pagination_over_collection_fetch`의 **기본값이 꺼짐**이라는 것과 *"left as a problem for the client to avoid"* / [`FetchSettings`](https://docs.hibernate.org/orm/6.4/javadocs/org/hibernate/cfg/FetchSettings.html) — `default_batch_fetch_size`가 `@BatchSize`의 전역 기본값이라는 설명
> **미확인**:
> - **영속성 컨텍스트의 1차 캐시·더티 체킹·쓰기 지연·flush 시점 3종**(§2-5) — Hibernate User Guide가 단일 대용량 페이지라 해당 절을 끝까지 읽어 내지 못했다. 서술은 통설을 옮긴 것이고 원문 대조는 못 했다
> - **`readOnly = true`가 스냅샷을 뜨지 않아 메모리를 절약한다**(§2-5) — 읽기 전용이라는 것까지는 알려져 있으나 스냅샷 생략 여부를 문서로 확인하지 않았다
> - **N+1의 원인이 지연 로딩이고 EAGER가 더 나쁘다**(§2-6) — 근거를 문서에서 확인하지 못했다. `fetch join`·`@EntityGraph`·DTO 조회의 선택 기준도 마찬가지다
> - **경고 로그 문자열**(§2-6) — 메시지 본문은 그대로지만 하이버네이트 5와 6의 로그 코드가 다르다. 본문에 코드를 쓰지 않은 건 그 때문이고, 실제 코드는 대조하지 않았다
> - **생성자 주입의 세 가지 이점**(§2-1) — 순환참조가 기동 시점에 실패한다는 것을 포함해 레퍼런스 문서로 대조하지 않았다
> - **빈 스코프와 생명주기**(§2-4) — singleton 기본·prototype 주입 함정·`ObjectProvider`/`@Lookup`을 문서로 확인하지 않았다
> - **Spring Batch의 재시작·skip/retry·`JpaPagingItemReader` 페이지 밀림**(§2-7) — chunk 트랜잭션 경계만 대조했고 나머지는 실무 경험 서술이다
> **미작성**: `@Async`와 트랜잭션의 상호작용 · 이벤트(`@TransactionalEventListener`)와 커밋 후 처리 · 2차 캐시 · `OSIV`(`spring.jpa.open-in-view`) · 테스트에서의 트랜잭션 롤백(`@Transactional` 테스트) · Spring Batch 파티셔닝과 병렬 스텝
