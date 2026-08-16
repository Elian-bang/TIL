# 응용 계층 — HTTP는 왜 세 번 다시 만들어졌나

## 30초 요약

- **429(Too Many Requests)와 503(Service Unavailable)은 발송 도메인의 일상**이다. 채널사 rate limit과 장애. **재시도 판단이 상태코드에서 갈린다** — **4xx = 내 잘못이라 재시도 무의미**, **5xx·타임아웃 = 결과를 모르니 재시도 대상**
- 단 **429는 4xx인데 재시도 대상**이다. 시간이 지나면 풀리기 때문. `Retry-After` 헤더를 봐야 한다
- **HTTP의 진화는 head-of-line blocking을 한 계층씩 밀어낸 역사**다. HTTP/1.1은 **응용 계층**에서 막혔고, HTTP/2가 그걸 풀었지만 **TCP 계층의 막힘은 못 풀었고**, HTTP/3는 **TCP를 버려서** 그걸 푼다
- **TLS 1.3은 협상할 것을 줄여서 핸드셰이크를 줄였다.** 정적 RSA 키 교환을 아예 없애 **전방향 비밀성을 선택이 아닌 전제로** 만들었고, ServerHello 이후는 전부 암호화된다
- **DNS와 CDN은 둘 다 "첫 바이트까지 몇 왕복인가"를 줄이는 장치**다. 그리고 **DNS는 조용한 장애 지점**이다 — 바뀐 IP는 **TTL이 만료돼야** 보인다

---

## 원리 — 왜 그런가

### 2-1. HTTP 상태코드 — 발송 도메인에서 의미 있는 것만

| 코드 | 의미 | 발송 도메인에서 |
|---|---|---|
| 200 / 201 / 204 | 성공 / 생성됨 / 내용 없음 | — |
| **400** | 잘못된 요청 | 템플릿 형식 오류, 번호 형식 오류. **재시도해도 소용없다** |
| 401 / 403 | 인증 실패 / 권한 없음 | 채널사 API 키 만료 ← **알람 대상** |
| 404 | 없음 | — |
| **409** | 충돌 | **중복 요청**을 알릴 때 쓸 수 있다 (멱등 처리 결과) |
| **429** | **Too Many Requests** | **채널사 rate limit 초과.** 백오프 후 재시도 |
| 500 | 서버 오류 | 채널사 내부 오류. 재시도 대상 |
| **502 / 503 / 504** | 게이트웨이 오류 / 서비스 불가 / 게이트웨이 타임아웃 | **채널사 장애.** 서킷 브레이커 대상 |

**★ 재시도 판단이 여기서 갈린다** (→ [대용량 처리](../system-design/high-throughput.md) §2-5)
- **4xx = 내 잘못.** 다시 보내도 같은 결과 → **재시도 금지** (단 **429는 예외** — 시간이 지나면 풀린다)
- **5xx / 타임아웃 = 결과를 모름** → **재시도 대상**, 단 멱등성 필요

**이 구분은 관례가 아니라 규격의 정의다.** RFC 9110은 4xx를 *"the client appears to have erred"*, 5xx를 *"the server is aware that it has erred or is incapable of performing the requested method"*로 정의한다. **4xx는 요청 자체가 틀렸다는 선언**이므로 같은 요청을 다시 보낼 이유가 없고, **5xx는 서버 사정이므로 나중에는 달라질 수 있다**는 것이 재시도 규칙의 근거다.

**429는 예외적인 위치에 있다.** 애초에 RFC 9110이 아니라 **RFC 6585**에서 나중에 추가된 코드이고, 정의가 *"the user has sent too many requests in a given amount of time (rate limiting)"*이다. **요청이 틀린 게 아니라 시점이 틀렸다** — 그래서 4xx인데도 재시도 대상이다.

**429를 받으면 `Retry-After` 헤더를 봐야 한다.** RFC 9110은 이 헤더를 *"how long a user agent ought to wait before making a follow-up request"*로 정의하고 **429·503·3xx·103에 붙을 수 있다**고 규정하며, 값은 **초 단위 정수 또는 HTTP-date** 두 형태다. ⚠️ **두 형태를 다 처리해야 한다** — 정수만 파싱하는 코드는 채널사가 날짜 형식으로 보내는 순간 실패하고, 그 실패는 대개 "헤더가 없다"로 처리돼 **조용히 지수 백오프로 떨어진다**. 알려준 시각을 무시하면 불필요하게 늦어지거나 여전히 튕긴다.

> **멱등성 정의도 같은 RFC에 있다.** *"the intended effect on the server of multiple identical requests with that method is the same as the effect for a single such request"* — GET·HEAD·PUT·DELETE·OPTIONS는 멱등이고 **POST는 아니다**. 발송 요청이 POST라서 생기는 문제와 그 해법은 → [신뢰성 패턴](reliability-patterns.md) §2-2.

### 2-2. HTTPS / TLS — TLS 1.3이 바꾼 것

**변하지 않은 뼈대부터.**
1. 클라이언트가 서버 인증서를 받아 **신뢰할 수 있는지 검증**(CA 체인)
2. **비대칭키 연산으로 대칭키를 안전하게 합의**
3. 이후 통신은 **대칭키로 암호화**

**왜 이 구조인가**: 비대칭키는 안전하지만 느리고, 대칭키는 빠르지만 키를 어떻게 전달할지가 문제다. → **키 합의만 비대칭으로, 본 통신은 대칭으로.**

**TLS 1.3(RFC 8446)이 바꾼 것은 "합의 방식"과 "협상 절차"다.**

| | TLS 1.2 이하 | TLS 1.3 |
|---|---|---|
| 키 교환 | 정적 RSA도 허용 — **서버 개인키로 대칭키를 감싸 전달** | **정적 RSA·DH 제거.** 모든 키 교환이 전방향 비밀성 제공 |
| 핸드셰이크 암호화 | ServerHello 이후도 상당 부분 평문 | **ServerHello 이후 전부 암호화** (`EncryptedExtensions` 도입) |
| 협상 대상 | 알고리즘 조합이 방대 | **레거시 대칭 알고리즘 전부 정리** |
| 상태 기계 | `ChangeCipherSpec` 등 | **불필요한 메시지 제거로 재구성** |

**★ 여기서 중요한 인과가 하나 나온다.** "1.3이 더 빠르다"의 원인은 최적화가 아니라 **고를 것을 없앤 것**이다. 선택지가 많으면 왕복이 필요하다 — 서버가 뭘 고를지 들어봐야 다음 단계를 정할 수 있으니까. **선택지를 규격 차원에서 잘라내니 클라이언트가 첫 메시지에 후보 키를 미리 실어 보낼 수 있게 됐고**, 그래서 왕복이 줄었다.

**정적 RSA 제거가 특히 중요하다.** 정적 RSA에서는 **서버 개인키 하나가 유출되면 과거에 녹음해 둔 트래픽까지 전부 복호화된다**. 1.3은 이 선택지를 남겨두지 않았다 — *"all public-key based key exchange mechanisms now provide forward secrecy"*.

**⚠️ 0-RTT(early data)는 발송 도메인에서 쓰면 안 된다.** TLS 1.3은 재접속 시 **핸드셰이크 완료 전에 데이터를 먼저 보내는** 0-RTT를 제공하는데, RFC 8446이 여기에 명시적 경고를 단다 — *"This data is not forward secret"*, 그리고 *"There are no guarantees of non-replay between connections"*.

**재전송 방지가 없다는 건 공격자가 그 요청을 그대로 복사해 다시 보낼 수 있다는 뜻**이다. 발송 요청은 POST이고 **재실행되면 그대로 중복 발송·중복 과금**이다. 0-RTT에 실어도 되는 건 **멱등한 요청뿐**이다(발송 결과 조회 같은 GET). 이건 [신뢰성 패턴](reliability-patterns.md) §2-2의 멱등성 문제가 **전송 계층 아래에서 다시 나타난 것**이다.

> 이 협상 비용이 [전송 계층](transport-layer.md) §2-2에서 본 왕복 비용에 그대로 얹힌다. **그래서 커넥션 풀이 TLS 시대에 더 중요해졌다** — 재사용하면 이 왕복을 통째로 건너뛴다.

### 2-3. DNS — 이름을 주소로 바꾸는 계층

**DNS는 하나의 거대한 서버가 아니라 위임의 연쇄다.** RFC 1034는 이름 공간을 *"a tree structure"*로 정의하고, 그 트리에 **"cut"을 내서 나뉜 조각이 zone**이라고 규정한다. 잘라낸 조각의 관리 권한은 통째로 넘어간다 — 위임받은 조직은 *"unilaterally change the data in the zone"*, 즉 **위쪽에 묻지 않고 자기 zone을 바꿀 수 있다**.

```
        .            (root)
        └── com      ← 이 아래는 com 관리자에게 위임
            └── channel-vendor.com    ← 이 아래는 채널사에게 위임
                └── api.channel-vendor.com
```

**★ 위임이 곧 확장성의 원리다.** 전 세계 도메인을 한 곳에서 관리하면 그 한 곳이 병목이자 단일 장애점이다. **잘라서 넘기면 각자가 각자 것만 책임진다** — [계층 모델](layered-model.md) §2-1의 "변하는 축을 한 층에 몰아넣는다"와 같은 발상이 이름 공간에 적용된 것이다.

**질의 방식이 두 가지다.**

| | 재귀(recursive) | 반복(iterative) |
|---|---|---|
| 서버가 하는 일 | **답을 구해올 때까지 대신 물어본다** | **"저기 가서 물어봐"라고 알려준다**(referral) |
| RFC 1034 표현 | *"never referrals"* | *"an error, the answer, or a referral"* |
| 누가 쓰나 | 우리 애플리케이션 → 리졸버 | 리졸버 → root → TLD → 권한 서버 |

**우리 코드가 보는 건 재귀 질의 하나뿐**이고, 그 뒤에서 리졸버가 반복 질의를 여러 번 돈다. **그래서 DNS 조회 한 번의 실제 비용이 눈에 안 보인다.**

**전송은 기본이 UDP다.** RFC 1035는 UDP 메시지를 **512바이트로 제한**하고, 넘치면 *"Longer messages are truncated and the TC bit is set in the header"* — 클라이언트가 **TCP로 다시 물어야 한다**(포트는 UDP·TCP 모두 53). **왜 UDP인가**: 질의 하나 응답 하나로 끝나는데 [전송 계층](transport-layer.md) §2-1의 3-way handshake를 내면 **연결 비용이 본문보다 크다**. 대신 신뢰성을 포기했으니 **재시도는 리졸버가 직접 한다**.

### 2-4. TTL과 캐싱 — DNS가 조용한 장애 지점이 되는 순간

**캐시가 없으면 DNS는 못 버틴다.** 그래서 모든 응답에 TTL이 붙는다. RFC 1034의 정의는 *"a time limit on how long an RR can be kept in a cache"*이고, 값은 *"assigned by the administrator for the zone where the data originates"* — **캐시하는 쪽이 아니라 데이터 주인이 정한다.**

**★ 여기에 통제 불가능성이 숨어 있다.** 채널사가 자기 도메인의 TTL을 3600으로 잡아 뒀다면, **우리는 그 IP를 최대 1시간 동안 갱신 못 한다.** 우리 코드에는 아무 문제가 없는데 트래픽이 죽은 IP로 계속 간다.

**RFC 1034가 이미 대비책을 적어 뒀다** — *"If a change can be anticipated, the TTL can be reduced prior to the change to minimize inconsistency during the change."* **예정된 변경 전에 TTL을 미리 낮춰 둔다.** 반대로 말하면 **예고 없는 장애에는 이 방법이 안 통한다.** DNS 기반 페일오버가 "빠르지 않다"의 실체가 이것이다.

**발송 도메인에서 실제로 겪는 형태:**

| 증상 | 실제 원인 |
|---|---|
| 채널사가 "IP를 바꿨다"고 공지했는데 여전히 옛 IP로 붙는다 | **캐시 TTL이 안 끝났다** |
| 배포 직후 몇 분간만 실패한다 | 리졸버마다 캐시 만료 시점이 달라 **일부 인스턴스만 옛 IP** |
| DNS는 정상인데 커넥션 풀이 계속 죽은 IP로 간다 | **이미 맺어 둔 연결은 재조회를 안 한다.** DNS 변경과 무관하게 살아 있다 |
| DNS 서버가 죽자 서비스 전체가 멎었다 | 조회 실패에 **타임아웃이 없거나 너무 길다** |

**마지막 두 줄이 핵심이다.** 커넥션 풀([전송 계층](transport-layer.md) §2-2)은 **DNS를 우회한다** — 좋을 때는 이득이지만 **주소가 바뀐 순간에는 그게 버그**가 된다. `maxLifetime`을 유한하게 잡아야 하는 이유가 하나 더 있는 셈이다.

그리고 **DNS 조회는 대개 타임아웃 설정 목록에서 빠져 있다.** connect/read timeout은 다들 챙기는데([신뢰성 패턴](reliability-patterns.md) §2-1), **이름 해석 단계는 그 타임아웃 바깥이다.** 스레드가 묶이는 경로가 하나 더 있다는 뜻이다.

### 2-5. HTTP/1.1의 한계 — 첫 번째 head-of-line blocking

**HTTP/1.0의 제약이 출발점이다.** RFC 9113의 서술 그대로 *"HTTP/1.0 allowed only one request to be outstanding at a time on a given TCP connection."* **연결 하나에 요청 하나**다.

HTTP/1.1은 연결 재사용(Keep-Alive)과 **파이프라이닝**을 넣었다. 응답을 기다리지 않고 요청을 연달아 보내는 것인데, **응답은 보낸 순서대로만 받을 수 있다.**

```
요청:  A  B  C          연달아 보냄
응답:  A ......         A가 느리면
          B  C          B·C는 다 끝났어도 못 나온다
```

**★ 이게 응용 계층 head-of-line blocking이다.** RFC 9113은 파이프라이닝이 *"only partially addressed request concurrency and still suffers from application-layer head-of-line blocking"*이라고 못 박는다. **앞의 하나가 뒤의 전부를 막는다.**

**그래서 현실의 해법은 프로토콜 밖에 있었다** — **연결을 여러 개 여는 것**. 브라우저가 호스트당 커넥션을 여러 개 열고, 도메인을 쪼개(도메인 샤딩) 그 한도를 늘렸다. **프로토콜이 못 푼 문제를 연결 개수로 때운 것**이고, 그 대가는 [전송 계층](transport-layer.md) §2-1·§2-4의 비용이다 — 연결마다 handshake를 다시 내고, 연결마다 cwnd가 1에서 다시 시작한다.

### 2-6. HTTP/2 — 멀티플렉싱이 푼 것, 그리고 못 푼 것

**HTTP/2는 순서 제약을 없앴다.** 요청·응답을 프레임으로 쪼개 스트림 단위로 붙이고, 한 연결 위에서 섞어 보낸다 — *"A single HTTP/2 connection can contain multiple concurrently open streams, with either endpoint interleaving frames from multiple streams."*

**A가 느려도 B·C의 프레임이 먼저 나갈 수 있다.** §2-5의 응용 계층 HOL blocking이 사라졌다.

**헤더 압축(HPACK)이 같이 들어온 이유도 명확하다.** RFC 9113은 *"HTTP fields are often repetitive and verbose, causing unnecessary network traffic as well as causing the initial TCP congestion window to quickly fill"*이라고 적는다.

> **여기서 계층이 만난다.** 헤더가 크면 [전송 계층](transport-layer.md) §2-4의 **초기 cwnd를 헤더로 다 써 버린다** — Slow Start 중이라 창이 작은데 그 작은 창을 본문이 아니라 반복되는 헤더가 채운다. **응용 계층 낭비가 전송 계층 성능으로 직결되는 구체적인 경로**다.

**★ 그런데 RFC 9113이 스스로 한계를 명시한다** — ***"Note, however, that TCP head-of-line blocking is not addressed by this protocol."***

**왜 못 푸나**: 스트림이 여러 개라는 건 **HTTP/2만 아는 사실**이고, 그 아래 TCP는 **바이트 하나의 연속된 흐름**만 본다. TCP는 순서를 보장하는 게 일이므로, **중간 세그먼트 하나가 유실되면 그 뒤 도착한 데이터를 전부 붙잡아 둔다.** 그 안에 다른 스트림의 프레임이 섞여 있어도 마찬가지다.

```
HTTP/2가 보는 것:  [스트림A] [스트림B] [스트림C]   ← 독립적
TCP가 보는 것   :  ...........바이트 흐름..........  ← 하나
                        ↑ 여기 유실
                   뒤의 B·C 프레임은 다 도착했어도 애플리케이션에 못 올라간다
```

**연결을 하나로 합친 것이 오히려 이 문제를 키웠다.** HTTP/1.1은 연결을 여러 개 열었으니 유실의 영향이 그중 하나에 갇혔는데, **HTTP/2는 전부 한 연결에 태웠으므로 패킷 하나가 모든 요청을 멈춘다.** RFC 9114의 표현이 정확하다 — *"a lost or reordered packet causes all active transactions to experience a stall regardless of whether that transaction was directly impacted by the lost packet."*

**HOL blocking이 사라진 게 아니라 한 층 아래로 내려갔다.**

### 2-7. HTTP/3 — 막힘을 마지막까지 밀어낸다

**TCP를 고쳐서는 풀 수 없다.** 순서 보장은 TCP의 정의이지 버그가 아니고, TCP는 OS 커널과 전 세계 중간 장비에 박혀 있어 **바꿔도 배포가 안 된다**.

**그래서 HTTP/3는 TCP를 버리고 UDP 위에 QUIC(RFC 9000)을 얹었다.** [전송 계층](transport-layer.md) §2-7의 마지막 문단이 말한 그 선택이다 — **신뢰성을 포기한 게 아니라 상위 계층으로 옮긴 것.**

**QUIC은 스트림별로 순서를 보장한다.** RFC 9000은 스트림을 *"a lightweight, ordered byte-stream abstraction"*으로 정의하면서 *"QUIC does not provide any means of ensuring ordering between bytes on different streams"*라고 규정한다. **스트림 사이에 순서 관계가 없으니, 한 스트림의 유실이 다른 스트림을 붙잡을 근거가 없다.**

**★ 세 프로토콜을 한 줄로 세우면 이야기가 보인다.**

| | 응용 계층 HOL | 전송 계층 HOL | 대가 |
|---|---|---|---|
| HTTP/1.1 | **있다**(파이프라이닝 순서 제약) | 있다 | 연결을 여러 개 열어 회피 |
| HTTP/2 | **없다**(멀티플렉싱) | **있다** — 한 연결이라 영향이 더 크다 | 한 연결에 몰아넣은 대가 |
| HTTP/3 | 없다 | **없다**(QUIC 스트림 독립) | 신뢰성·혼잡 제어를 **직접 구현**해야 한다 |

**덤으로 따라온 것 둘.** **(1) 핸드셰이크 통합** — *"The QUIC handshake combines negotiation of cryptographic and transport parameters"*(RFC 9000). TCP 핸드셰이크 위에 TLS 핸드셰이크를 얹는 2단 구조가 아니라 **하나로 합쳐졌다**(*"QUIC also incorporates TLS 1.3 at the transport layer"*, RFC 9114) — §2-2의 TLS 1.3이 여기 들어와 있고, **0-RTT의 재전송 위험도 그대로 따라온다**(*"0-RTT provides no protection against replay attacks"*). **(2) 연결 마이그레이션** — TCP 연결의 정체성은 **IP·포트 4-tuple**이라 주소가 바뀌면 끊기지만, QUIC은 connection ID로 식별한다(*"changes in addressing at lower protocol layers (UDP, IP) do not cause packets for a QUIC connection to be delivered to the wrong endpoint"*). **Wi-Fi에서 LTE로 넘어가도 연결이 유지된다.**

### 2-8. CDN — 왜 가까운 것이 중요한가

**대역폭이 아니라 왕복 수가 문제다.** [전송 계층](transport-layer.md) §2-2에서 본 그대로, 연결 하나를 새로 여는 데 드는 왕복이 이만큼이다.

```
DNS 조회        : 1왕복 (캐시 미스일 때)
TCP handshake  : 1.5왕복
TLS 1.3 협상    : 1왕복
HTTP 요청·응답  : 1왕복
```

**왕복 수는 거리와 무관하게 고정이고, 왕복 한 번의 비용만 거리에 비례한다.** 그래서 **거리가 곱해진다** — RTT가 5ms면 이 전체가 수십 ms지만, RTT가 150ms면 **데이터를 한 바이트도 안 보낸 채 0.5초**가 지난다. 회선을 아무리 굵게 해도 안 줄어든다. **빛의 속도가 상한이라 돈으로 못 산다.**

**★ 그래서 유일한 해법이 "가까이 두는 것"이고, 그게 CDN이다.** 사용자 근처의 엣지가 응답하면 **위 네 줄 전부가 짧은 RTT로 계산된다.** 캐시 히트로 원본 조회를 없애는 것은 그 다음 이야기다.

**어느 엣지로 보낼지는 대개 DNS가 정한다** — 같은 이름을 물어도 **묻는 위치에 따라 다른 IP를 돌려준다**. §2-3의 위임 구조가 그대로 로드밸런싱 장치로 쓰이는 것이고, **§2-4의 TTL 문제도 그대로 물려받는다**(엣지를 빼도 캐시된 IP로 계속 간다).

**발송 도메인에서는 대상이 갈린다.** **MMS·RCS 첨부 이미지**는 수신자 단말이 URL을 직접 당기고 대량·읽기 전용이라 CDN이 정확히 맞는다. 반대로 **채널사 API 호출은 쓰기 요청이라 캐시할 수 없다** — 여기서 왕복을 줄이는 수단은 **커넥션 풀 재사용**뿐이다.

### 2-9. 쿠키와 세션 — 상태 없는 프로토콜에 상태를 얹기

**HTTP는 요청 간에 아무것도 기억하지 않는다.** [계층 모델](layered-model.md) §2-4에서 본 대로 **OSI가 예상한 세션 계층이 실현되지 않았고**, HTTP가 그 일을 자기 안에서 처리했다. 그 결과물이 쿠키(RFC 6265)다.

```
서버 → 클라이언트 : Set-Cookie: SESSIONID=abc; HttpOnly; Secure
클라이언트 → 서버 : Cookie: SESSIONID=abc      ← 이후 요청마다 자동으로
```

**상태를 클라이언트에 맡겨 두고 매 요청에 다시 받는 구조**다. `Expires`/`Max-Age`(만료, 둘 다 있으면 `Max-Age` 우선), `Domain`(하위 도메인까지 포함), `Path`, `Secure`(보안 채널일 때만 전송), `HttpOnly`(스크립트 접근 불가) 같은 속성으로 범위를 좁힌다.

**★ 그런데 RFC 6265가 직접 경고하는 비대칭이 있다.** 서버는 *"cannot determine from the Cookie header alone ... whether the cookie was set with the Secure or HttpOnly attributes."* **속성은 보낼 때만 붙고 돌아올 때는 값만 온다** — 자기가 건 조건이 지켜졌는지 확인할 방법이 없다. **쿠키 값은 검증 대상이지 신뢰 대상이 아니다.**

**그리고 브라우저는 요청을 누가 유발했든 쿠키를 붙인다**(ambient authority). 남의 사이트가 우리 서버로 요청을 만들어도 **쿠키가 실려 간다** — CSRF의 뿌리가 이 자동 첨부다.

**발송 도메인에서는 대부분 쿠키를 안 쓴다.**

| | 쿠키 세션 | 토큰(API 키·Bearer) |
|---|---|---|
| 자동 첨부 | 브라우저가 알아서 | **직접 헤더에 넣는다** |
| 상태 | **서버가 세션을 들고 있다** | 서버가 안 들고 있어도 된다 |
| 스케일아웃 | 세션 공유 필요 → [Redis](../datastore/redis.md) 같은 외부 저장소 | 인스턴스가 늘어도 그대로 |

**채널사 API 호출은 서버 대 서버이고 쿠키는 브라우저 규약**이라, 자동 첨부가 이득이 아니라 위험이다. 그래서 **명시적으로 붙이는 토큰**을 쓴다. 반대로 **어드민 화면은 브라우저가 상대**라 쿠키 세션이 자연스럽다 — **같은 서비스 안에서도 상대가 누구냐에 따라 갈린다.**

---

**다음으로 읽을 것**

- 왕복 비용과 커넥션 풀, HTTP/3가 피하려는 그 HOL blocking → [전송 계층](transport-layer.md) §2-2, §2-7
- TLS가 몇 계층인지에 답이 없는 이유 → [계층 모델](layered-model.md) §2-4
- 재시도할 때 중복을 어떻게 막는가(0-RTT 문제와 같은 뿌리) → [신뢰성 패턴](reliability-patterns.md) §2-2
- 재시도 전략과 서킷 브레이커 → [대용량 처리](../system-design/high-throughput.md) §2-4, §2-5

---

> **기준 버전**: RFC 9110(HTTP Semantics) · RFC 9113(HTTP/2) · RFC 9114(HTTP/3) · RFC 9000(QUIC) · RFC 8446(TLS 1.3) · RFC 1034/1035(DNS) · RFC 6265(Cookies) 기준
> **확인한 출처**: [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html) — 4xx/5xx 클래스 정의 문장, `Retry-After`의 정의·적용 상태코드(103·429·503·3xx)와 두 가지 syntax, 멱등성 정의와 GET/HEAD/PUT/DELETE/OPTIONS 목록(POST 제외) · [RFC 6585](https://www.rfc-editor.org/rfc/rfc6585.html) — **429가 RFC 9110이 아니라 여기서 정의된다는 사실**과 정의 문장, `Retry-After`를 MAY로 포함한다는 규정
> [RFC 8446](https://www.rfc-editor.org/rfc/rfc8446.html) — §1.2 변경 목록(정적 RSA·DH 제거와 전방향 비밀성 필수화, ServerHello 이후 전면 암호화와 `EncryptedExtensions`, 레거시 알고리즘 정리, `ChangeCipherSpec` 제거), §2.3의 0-RTT 경고 문장 두 개
> [RFC 9113](https://www.rfc-editor.org/rfc/rfc9113.html) — HTTP/1.0의 요청 하나 제약, 파이프라이닝이 남긴 응용 계층 HOL blocking, 헤더 반복이 초기 cwnd를 채운다는 서술, 스트림 인터리빙, **"TCP head-of-line blocking is not addressed by this protocol"** · [RFC 9114](https://www.rfc-editor.org/rfc/rfc9114.html) — 유실 패킷 하나가 무관한 트랜잭션까지 stall시킨다는 서술, QUIC이 TLS 1.3을 전송 계층에 포함, QUIC = RFC 9000 · [RFC 9000](https://www.rfc-editor.org/rfc/rfc9000.html) — 스트림 정의와 스트림 간 순서 미보장, connection ID의 목적과 연결 마이그레이션, 암호·전송 핸드셰이크 통합, 0-RTT 재전송 무방비
> [RFC 1034](https://www.rfc-editor.org/rfc/rfc1034.html) — 트리 구조와 zone 위임(단독 변경 권한), 재귀/반복 질의 정의, TTL 정의와 zone 관리자가 정한다는 규정, **변경 전 TTL을 미리 낮추라는 권고** · [RFC 1035](https://www.rfc-editor.org/rfc/rfc1035.html) — UDP 512바이트 제한과 TC 비트·TCP 재질의, 포트 53 · [RFC 6265](https://www.rfc-editor.org/rfc/rfc6265.html) — Set-Cookie/Cookie 왕복 구조와 각 속성의 의미(`Max-Age` 우선), **서버가 Cookie 헤더만으로는 Secure/HttpOnly 설정 여부를 알 수 없다는 경고**, ambient authority 경고
> **미확인**: **§2-2** — "TLS 1.3 full handshake가 1왕복"은 RFC 8446 §2의 흐름도에서 읽은 것이고 인용 가능한 규정 문장으로는 확인하지 못했다. **TLS 1.2가 2왕복이라는 통설도 RFC 5246으로 대조하지 않았다** — 본문 표에서 왕복 수를 단정하지 않은 이유다 · **§2-4** — JVM의 DNS 캐시 기본값(`networkaddress.cache.ttl`)이 리졸버 TTL을 무시한다는 함정을 확인하지 못해 본문에서 뺐다
> **§2-5** — "브라우저가 호스트당 커넥션 6개"는 구현 관례이고 RFC 규정이 아니라 숫자를 본문에 쓰지 않았다 · **§2-8** — **CDN은 RFC가 없다.** 왕복 수 분해는 §2-2·§2-3과 [전송 계층](transport-layer.md)에서 확인한 값을 더한 것이고, **RTT 5ms/150ms는 예시 수치이지 측정값이 아니다** · **§2-9** — `SameSite`는 RFC 6265에 없어(6265bis 계열) 뺐다 · 409의 정확한 규정 문장은 전문을 확보하지 못해 서술을 약하게 두었다
> **미작성**: HTTP 캐시 헤더(`Cache-Control`·`ETag`) 상세 · HPACK/QPACK 동작 · WebSocket · gRPC · CORS · DoH/DoT · 인증서 체인 검증과 OCSP
