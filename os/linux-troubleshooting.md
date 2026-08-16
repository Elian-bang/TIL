# 리눅스에서 장애 났을 때 뭘 보나

> ⚠️ **이 문서만 성격이 다르다.** 나머지가 "왜 그렇게 설계됐나"라면 이건 운영 치트시트다. 원리가 아니라 순서를 남긴다.

## 30초 요약

- 어떤 순서로 좁혀 들어가는지가 실제로 쓰이는 부분이다. 명령어를 몇 개 아느냐는 그다음이다
- 순서는 대체로 **자원(CPU·메모리·디스크) → 연결 상태 → 프로세스 내부** 순으로 좁혀진다
- 디스크 full이 의외로 잦은 장애 원인이다. `df -h`를 먼저 본다
- `ss -tanp`의 **TIME_WAIT / CLOSE_WAIT 적체**는 커넥션 관리 문제를 가리킨다. `-a`를 빼면 TIME_WAIT가 안 보인다
- 응답이 멈췄으면 **스레드 덤프(`jstack`)**를 뜬다. 어디서 블로킹돼 있는지가 거기 있다

---

## 조사 순서

| 상황 | 명령 |
|---|---|
| CPU·메모리 확인 | `top`, `htop` |
| 디스크 용량 | `df -h`, `du -sh *` ← **디스크 full이 의외로 잦은 장애 원인** |
| 포트·연결 상태 | `ss -tanp` ← **TIME_WAIT / CLOSE_WAIT 적체 확인** |
| 로그 실시간 | `tail -f`, `grep`, `awk` |
| 프로세스 | `ps -ef`, `jps` (Java) |
| JVM 상태 | `jstat -gc`, `jmap`, `jstack` ← **스레드 덤프로 어디서 막혔는지** |

`ss`에 `-a`를 붙인 이유가 있다. man page는 아무 옵션 없이 쓰면 *"a list of open non-listening sockets (e.g. TCP/UNIX/UDP) that have established connections"*를 보여 준다고 적고, `-l` 설명에는 listening 소켓이 *"(these are omitted by default)"*라고 못 박아 뒀다. TIME-WAIT와 SYN-RECV는 STATE-FILTER에서 *"states, which are maintained as minisockets"*로 따로 묶여 있다. TIME_WAIT를 세려면 `ss -tanp`처럼 `-a`를 붙이거나 `ss -tn state time-wait`으로 상태를 명시하는 편이 안전하다. `netstat`은 이제 쓰지 않는다. man page가 *"This program is mostly obsolete. Replacement for netstat is ss."*라고 적어 뒀다.

두 상태가 가리키는 방향이 서로 다르다. RFC 9293은 CLOSE-WAIT을 *"represents waiting for a connection termination request from the local user"*라고 정의한다. 상대가 FIN을 보냈는데 우리 쪽 애플리케이션이 `close()`를 안 부른 상태다. CLOSE_WAIT이 쌓이면 원인은 상대가 아니라 우리 코드에 있다. TIME-WAIT은 *"represents waiting for enough time to pass to be sure the remote TCP peer received the acknowledgment of its connection termination request and to avoid new connections being impacted by delayed segments from previous connections"*이고, 먼저 끊은 쪽이 *"MUST linger in the TIME-WAIT state for a time 2xMSL"* 하도록 규정돼 있다. 정상 동작이라 없앨 대상이 아니고, 많다는 것은 연결을 너무 자주 맺고 끊는다는 신호다.

**`jstack`을 언급하면 좋다**: "응답이 멈췄을 때 스레드 덤프를 떠서 어디서 블로킹돼 있는지 봅니다." [신뢰성 패턴](../network/reliability-patterns.md) §2-1의 무한 대기를 실제로 잡는 방법이다.

> `jmap`은 JDK 문서가 *"This command is experimental and unsupported"*라고 표시해 둔 도구다. 운영 절차에 고정으로 박아 두기 전에 알고는 있어야 한다.

---

**다음으로 읽을 것**

- CLOSE_WAIT가 왜 쌓이는가 → [전송 계층](../network/transport-layer.md)
- 스레드가 왜 묶이는가 → [신뢰성 패턴](../network/reliability-patterns.md) §2-1
- 메모리 부족의 원인 판별 → [메모리와 페이징](memory-and-paging.md)

---

> **기준 버전**: Linux man-pages · JDK 21 도구 문서 · RFC 9293 기준. 배포판별 차이는 다루지 않는다
> **확인한 출처**:
> - `top(1)` — "display Linux processes", "provides a dynamic real-time view of a running system": https://man7.org/linux/man-pages/man1/top.1.html
> - `df(1)` — "report file system space usage", `-h`는 "print sizes in powers of 1024": https://man7.org/linux/man-pages/man1/df.1.html
> - `du(1)` — "estimate file space usage", `-s`는 "display only a total for each argument": https://man7.org/linux/man-pages/man1/du.1.html
> - `ss(8)` — 기본 출력이 established 연결이라는 서술, `-l`의 "(these are omitted by default)", STATE-FILTER의 minisocket 분류(time-wait·syn-recv), `-t`/`-n`/`-p`/`-a` 설명: https://man7.org/linux/man-pages/man8/ss.8.html
> - `netstat(8)` — "This program is mostly obsolete. Replacement for netstat is ss.": https://man7.org/linux/man-pages/man8/netstat.8.html
> - `ps(1)` — "report a snapshot of the current processes", `-e`("Select all processes")·`-f`("Do full-format listing"): https://man7.org/linux/man-pages/man1/ps.1.html
> - [RFC 9293](https://www.rfc-editor.org/rfc/rfc9293.html) — CLOSE-WAIT·TIME-WAIT의 정의 원문과 "MUST linger in the TIME-WAIT state for a time 2xMSL"
> - JDK 21 `jstack` — "print Java stack traces of Java threads for a specified Java process": https://docs.oracle.com/en/java/javase/21/docs/specs/man/jstack.html
> - JDK 21 `jstat` — "monitor JVM statistics", `-gc`가 "Garbage collected heap statistics"를 낸다는 것: https://docs.oracle.com/en/java/javase/21/docs/specs/man/jstat.html
> - JDK 21 `jmap` — "print details of a specified process", "This command is experimental and unsupported": https://docs.oracle.com/en/java/javase/21/docs/specs/man/jmap.html
> **미확인**:
> - **`ss`가 기본 출력에서 TIME-WAIT을 뺀다는 것** — `-l`처럼 "omitted by default"라고 명시한 문장은 없다. 기본이 established 연결이라는 서술과 TIME-WAIT이 minisocket으로 분류된다는 점에서 따라 나오는 추론이라, 본문에는 "상태를 명시하는 편이 안전하다"까지만 적었다
> - **`htop`·`jps`·`tail`·`grep`·`awk`** — 표에는 있으나 man page를 대조하지 않았다
> - **`-p`가 다른 사용자의 프로세스를 보려면 권한이 필요한지** — `ss(8)`에서 확인하지 못해 본문에 쓰지 않았다
> - **조사 순서 자체**(자원 → 연결 → 프로세스 내부) — 실무에서 쓰던 순서이고 근거가 되는 문서는 없다
> **미작성**: `/proc/pressure`(PSI)로 자원 압박 재기 · `dmesg`와 OOM 킬러 로그 확인 · `strace`/`perf` · 컨테이너 환경에서 달라지는 것(cgroup 한도와 `top`이 보는 값의 불일치)
