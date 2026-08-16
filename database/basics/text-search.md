# RDB 전문 검색은 어디까지 되나

## 30초 요약

- **`LIKE '%키워드%'`가 인덱스를 못 타는 이유는 하나다.** B+Tree는 원본 값의 정렬이라 앞이 와일드카드면 시작점이 없다. 전문 검색 기능은 전부 이 한 문장을 우회하려는 시도다
- **RDB도 역색인을 갖고 있다.** MySQL `FULLTEXT`, PostgreSQL `tsvector`+GIN. ES만의 자료구조가 아니고, 갈리는 건 자료구조가 아니라 **정합과 운영 모델**이다
- **한국어에서는 공백 분리가 무너진다.** 답은 둘이다. 형태소 분석(정확, 사전 필요, 미등록어에 약함)이냐 n-gram(사전 불필요, 색인이 크고 오탐이 난다)이냐. MySQL이 공식 제공하는 형태소 파서는 **일본어(MeCab)뿐**이라 한국어는 사실상 `WITH PARSER ngram`이다
- **`pg_trgm`은 `LIKE '%..%'`에 인덱스를 태운다.** 문자열을 3글자로 쪼개 GIN에 넣으면 *"검색어가 앞에 고정될 필요가 없다"*. 첫 번째 불릿의 원칙에 뚫린 유일한 구멍이다
- **판단선은 "검색이 부가 기능인가, 제품인가"다.** 다른 조건으로 이미 좁혀진 뒤의 키워드 필터 → RDB로 끝난다. 관련도 랭킹·하이라이팅·오타 교정·다중 소스 통합이 요구사항에 들어오면 → ES

---

## 원리 — 왜 그런가

### 2-1. `LIKE '%키워드%'`는 왜 인덱스를 못 타나

발송 이력에서 본문에 "휴진"이 든 건을 찾는다고 하자.

```sql
SELECT * FROM send_message WHERE content LIKE '%휴진%';
```

**인덱스는 `content` 원본 값 기준으로 정렬돼 있다**([DB 인덱스](b-tree-index.md) §2-6). B+Tree 탐색은 루트에서 "찾는 값이 이 구분자보다 큰가 작은가"를 비교하며 내려가는 것인데, `'%휴진%'`은 **첫 글자가 무엇인지 모른다.** 비교 대상이 없으니 어느 자식으로 내려갈지 결정할 수 없다 → 리프를 전부 훑으며 문자열 비교. `LIKE '휴진%'`은 시작점이 있으니 타고, `'%휴진%'`·`'%휴진'`은 못 탄다.

[DB 인덱스](b-tree-index.md) §2-6의 "컬럼에 함수를 씌우면 못 탄다"와 **같은 문제의 다른 얼굴**이다. `content`의 정렬이 `substring(content, ...)`의 정렬을 보장하지 않는다.

그런데 속도만 문제가 아니다. PostgreSQL 공식 문서는 `LIKE`/정규식의 한계를 셋으로 정리한다. *"There is no linguistic support"*(파생어를 못 잡는다) · *"They provide no ordering (ranking) of search results"* · *"They tend to be slow because there is no index support, so they must process all documents for every search"*. 랭킹이 없다는 건 결과가 수천 건일 때 쓸모없다는 뜻이고, 어형을 못 잡는다는 건 "발송"으로 검색해 "발송했습니다"를 놓친다는 뜻이다. **전문 검색은 이 셋을 한꺼번에 푼다.**

### 2-2. 역색인을 RDB 테이블 옆에 놓는다는 것

역색인의 기본 원리(값 → 문서 목록, 애널라이저가 색인 시점에 미리 일한다)는 [Elasticsearch](../../datastore/elasticsearch.md) §2-1에서 다뤘다. **여기서는 그 자료구조가 RDB 안에 있을 때 무엇이 달라지는지만 본다.**

| | ES | RDB 전문 인덱스 |
|---|---|---|
| 자료구조 | 역색인 | **역색인 (같다)** |
| 원본과의 관계 | **별도 저장소로 복제** | **같은 테이블의 인덱스** |
| 갱신 시점 | 비동기 색인 → Near Real-Time(기본 1초) | **같은 트랜잭션 안에서 갱신** |
| 원본과의 정합 | 색인 파이프라인이 깨지면 **틀어진다** | **틀어질 수 없다** |
| 다른 조건과의 결합 | 한 인덱스 안에서 처리 | **인덱스가 분리된다** (§2-8) |

**RDB 전문 검색을 쓰는 진짜 이유는 속도가 아니라 정합과 운영 대상 개수다.** 발송 이력을 ES로 넘기면 색인 파이프라인·클러스터·재색인 절차가 전부 운영 대상이 된다. `FULLTEXT` 인덱스 한 줄이면 `INSERT`가 커밋되는 순간 검색되고, 틀어질 여지가 없다.

**대가는 쓰기다.** 인덱스에도 값이 붙는데([DB 인덱스](b-tree-index.md) §2-8) 전문 인덱스는 행 하나에 토큰 수만큼 항목이 늘어난다. 90자 SMS 본문 하나가 인덱스 항목 수십~수백 개다. **발송 원장처럼 초당 쓰기가 몰리는 테이블에 전문 인덱스를 다는 건 쓰기 처리량을 직접 깎는 결정이다.**

### 2-3. 한국어에서 토크나이징이 무너진다 (형태소 vs n-gram)

MySQL 공식 문서가 문제를 정확히 진술한다:
> *"The built-in MySQL full-text parser uses the white space between words as a delimiter to determine where words begin and end, which is a limitation when working with ideographic languages that do not use word delimiters."*

한국어는 공백을 쓰지만 **어절이 단어가 아니다.** `"휴진 안내를 발송했습니다"`를 공백으로 자르면 `휴진` | `안내를` | `발송했습니다`가 된다. 사용자가 "발송"으로 검색하면 못 찾는다. 색인에 있는 토큰은 `발송했습니다`지 `발송`이 아니다. 조사·어미가 붙는 언어에서 공백 분리는 거의 항상 틀린다.

**해법은 둘뿐이고, 트레이드오프가 정반대다.**

| | **형태소 분석** | **n-gram** |
|---|---|---|
| 방식 | 사전으로 `발송/명사 + 했/선어말 + 습니다/어미` 분해 | 무조건 n글자씩 자름 |
| 사전 | **필요** | **불필요** |
| 색인 크기 | 작다 (형태소 수만큼) | **크다.** 길이 L 텍스트가 대략 L−n+1개 토큰 |
| 오탐 | 거의 없다 | **난다** (§2-5) |
| **미등록어** | **못 쪼갠다** → 검색 누락 | **강하다.** 사전을 안 보므로 |
| 제품 | ES Nori, MySQL MeCab(일본어) | MySQL `ngram`, PG `pg_trgm` |

**미등록어(OOV)가 실무의 갈림길이다.** 대량 발송 본문에는 신규 상호명·약품명·캠페인명·줄임말이 계속 들어온다. 형태소 분석기는 사전에 없는 말을 엉뚱하게 쪼개고, 그러면 검색에서 사라진다. 사전을 관리할 사람이 없으면 n-gram이 실무적으로 더 안전하다. 그리고 ⚠️ **MySQL이 공식 제공하는 형태소 파서는 일본어 MeCab 하나이고 공식 문서 어디에도 한국어 파서가 없다** → MySQL에서 한국어 전문 검색은 사실상 `WITH PARSER ngram` 한 가지 선택지다.

### 2-4. MySQL Full-Text Index의 제약과 세 가지 검색 모드

**먼저 제약부터.** 이걸 모르고 설계하면 나중에 못 붙인다.
> *"Full-text indexes can be used only with `InnoDB` or `MyISAM` tables, and can be created only for `CHAR`, `VARCHAR`, or `TEXT` columns."*

- 컬럼 타입이 `CHAR`/`VARCHAR`/`TEXT`로 한정된다 → 전문 인덱스에 `client_id BIGINT`나 `created_at`을 **같이 담을 수 없다.** 이게 §2-8의 결정적 제약이다
- 검색어는 *"a string value that is constant during query evaluation"*이라, **다른 컬럼 값으로는 검색할 수 없다**
- InnoDB 불린 검색은 *"require a FULLTEXT index on all columns of the MATCH() expression"* → **`MATCH()`의 컬럼 조합마다 인덱스가 따로 필요하다**

```sql
ALTER TABLE send_message ADD FULLTEXT INDEX ft_content (content) WITH PARSER ngram;
SELECT * FROM send_message WHERE MATCH(content) AGAINST ('휴진' IN BOOLEAN MODE);
```

| 모드 | 구문 | 성격 |
|---|---|---|
| **자연어** (기본) | `AGAINST('휴진 안내')` | 관련도 점수 계산. 연산자 없음(`"`만 예외) |
| **불린** | `AGAINST('+휴진 -취소' IN BOOLEAN MODE)` | **조건을 직접 지정.** `+`필수 `-`제외 `*`접두 `"구문"` `@거리`(InnoDB 전용) `>`/`<`가중치 `~`감점 |
| **쿼리 확장** | `AGAINST('휴진' WITH QUERY EXPANSION)` | 1차 검색 결과의 단어를 추가해 **다시 검색** |

**자연어 모드의 관련도**는 *"the number of words in the row (document), the number of unique words in the row, the total number of words in the collection, and the number of rows that contain a particular word"*로 계산된다. 흔한 단어일수록 가중치가 낮다(TF-IDF 계열).

⚠️ **50% 임계는 MyISAM 이야기다.** MyISAM 자연어 검색은 *"present in at least 50% of the rows"*인 단어를 사실상 불용어 취급한다. 발송 본문에 "고객님"이 90% 들어 있으면 검색이 0건이 된다. 공식 문서가 우회법을 명시한다. *"build search indexes on `InnoDB` tables, or use the boolean search mode."* InnoDB면 애초에 해당 없다. 최소 토큰 길이는 InnoDB `innodb_ft_min_token_size`가 기본 3, MyISAM `ft_min_word_len`이 기본 4이고, 불린 모드에서도 최소 길이와 불용어는 적용되지만 **`*`(절단 연산자)를 붙인 단어는 짧거나 불용어여도 제거되지 않는다.**

### 2-5. 한국어를 쓰려면 `WITH PARSER ngram`

> *"An ngram is a contiguous sequence of `n` characters from a given sequence of text."* · *"The ngram parser has a default ngram token size of 2 (bigram)."*

`ngram_token_size`는 **기본 2, 범위 1~10**이고 읽기 전용 변수다. *"may only be set as part of a startup string or in a configuration file"* 이라 런타임에 못 바꾼다. 바꾸려면 재시작에 전문 인덱스 재생성까지 필요하니 **설계 시점에 정해야 하는 값**이다. `"휴진안내"`(n=2)는 `휴진` | `진안` | `안내`가 되고, 공백은 넘지 않는다(`"ab cd"` → `"ab"`,`"cd"`). 어절 경계를 넘는 오탐은 안 생긴다.

**반드시 알아야 할 네 가지.**

**(1) 최소 토큰 길이 설정이 전부 무시된다.** *"The following minimum and maximum word length configuration options are ignored for `FULLTEXT` indexes that use the ngram parser: `innodb_ft_min_token_size`, `innodb_ft_max_token_size`, `ft_min_word_len`, and `ft_max_word_len`."* 토큰 길이는 오직 `ngram_token_size`가 정한다. §2-4에서 외운 "기본 3자"는 ngram에서는 성립하지 않는다.

**(2) 불용어 처리 규칙이 뒤집힌다.** *"Instead of excluding tokens that are equal to entries in the stopword list, the ngram parser excludes tokens that **contain** stopwords."* 쉼표가 불용어면 `"a,b"`의 토큰 `"a,"`와 `",b"`가 둘 다 버려진다. 기본 불용어 목록을 그대로 두면 예상 못 한 구간이 색인에서 빠진다.

**(3) 불린 모드는 구문 검색으로 바뀐다.** *"For boolean mode search, the search term is converted to an ngram phrase search. For example, the string 'abc' (assuming `ngram_token_size=2`) is converted to '\"ab bc\"'."* `"abc"`를 담은 문서는 매칭되지만 `"ab"`만 담은 문서는 매칭되지 않는다. 토큰이 연속된 위치에 있어야 하므로 **검색어가 n보다 길면 오탐이 상당히 줄어든다.**

**(4) 그럼에도 n자 검색어는 오탐이 난다.** bigram에서 검색어가 2글자면 토큰 하나라 (3)의 보호가 없다.
```text
검색어 "정보"
"고객 정보 변경"  → 토큰 "정보" 포함 → 매칭 (정상)
"수정보다 빠르게"  → "수정"|"정보"|"보다" → 매칭 (오탐)
```
접두가 짧을 때도 필터가 사실상 없다. *"If the prefix term of a wildcard search is shorter than ngram token size, the query returns all indexed rows that contain ngram tokens starting with the prefix term."* (n=2에서 `"a*"`는 "a"로 시작하는 행을 전부 돌려준다.)

**대량 발송 도메인에서 특히 아픈 자리는 수신번호다.** `"01012345678"`을 bigram으로 넣으면 `01`,`10`,`01`,`12`… 로 쪼개지고, `"1234"`로 검색하면 자릿수만 우연히 겹치는 번호가 대량으로 걸린다. 번호는 전문 검색 대상이 아니다. 뒷자리 검색이 요구사항이면 **역순 문자열 컬럼을 따로 두고 접두 인덱스를 태우는 게 정석**이다(`'%1234'` → `reversed_no LIKE '4321%'`). 수신번호가 암호화 컬럼이면 애초에 어떤 인덱스도 무의미하다 → [DB 인덱스](b-tree-index.md) §2-13.

### 2-6. PostgreSQL의 `tsvector`/`tsquery` + GIN

MySQL이 "인덱스 타입" 하나로 푸는 걸 PostgreSQL은 **타입 + 연산자 + 인덱스**로 나눠 푼다.

- **`tsvector`**: *"a sorted array of normalized lexemes"*. 문서를 전처리한 결과
- **`tsquery`**: *"search terms, which must be already-normalized lexemes, and may combine multiple terms using AND, OR, NOT, and FOLLOWED BY operators"*
- **`@@`**: *"returns `true` if a `tsvector` (document) matches a `tsquery` (query)"*

핵심은 **lexeme(어휘소)** 다. 토큰을 정규화한 것으로, *"folding upper-case letters to lower-case, and often involves removal of suffixes"*, 그리고 *"This step typically eliminates stop words"*. 이 정규화를 딕셔너리가 수행한다.

```sql
ALTER TABLE send_message
  ADD COLUMN content_tsv tsvector
  GENERATED ALWAYS AS (to_tsvector('simple', coalesce(content, ''))) STORED;

CREATE INDEX idx_content_tsv ON send_message USING GIN (content_tsv);
```

**두 가지가 설계 결정이다. (1) `to_tsvector`에 설정 이름을 반드시 넣어야 한다.** *"Only text search functions that specify a configuration name can be used in expression indexes. This is because the index contents must be unaffected by `default_text_search_config`."* 설정이 바뀌면 인덱스가 서로 다른 규칙으로 만들어진 `tsvector`들의 뒤죽박죽이 된다. **1인자 `to_tsvector(content)`는 인덱스에 못 쓴다.**

**(2) 표현식 인덱스보다 생성 컬럼.** 공식 문서가 이유 둘을 든다. 쿼리에서 설정 이름을 다시 안 써도 되고, *"Searches will be faster, since it will not be necessary to redo the `to_tsvector` calls to verify index matches."*

**GIN이 기본 선택**이다. *"GIN indexes are the preferred text search index type."* GiST는 *"lossy, meaning that the index might produce false matches, and it is necessary to check the actual table row"* 인데, 문서를 고정 길이 시그니처로 압축하다 보니 해시 충돌이 오탐을 만들고 그만큼 힙을 헛짚는다. GIN도 *"store only the words (lexemes) of `tsvector` values, and not their weight labels"* 라 **가중치를 쓰는 쿼리는 행 재검사가 붙는다.**

⚠️ **한국어에서는 이 정규화가 사실상 작동하지 않는다.** 공식 딕셔너리의 정규화 수단은 Snowball 어간 추출인데, 공식 문서의 딕셔너리 장에 한국어를 포함한 CJK 언급이 없다. `'simple'` 설정은 *"converting the input token to lower case and checking it against a file of stop words"* 뿐이라 어형은 손대지 않는다. §2-3의 공백 분리 문제가 그대로 남는다. **PostgreSQL에서 한국어를 다룰 실질적 수단은 다음 절의 `pg_trgm`이다.**

### 2-7. 트라이그램이 `LIKE '%..%'`를 인덱스로 태운다

[PostgreSQL 복제·운영](../../postgresql/replication-and-ops.md) §2-8에 한 줄로 나온 그 extension이다. **원리를 펼치면 §2-1의 원칙에 어떻게 구멍이 나는지가 보인다.**

> *"A trigram is a group of three consecutive characters taken from a string."* · *"Each word is considered to have two spaces prefixed and one space suffixed when determining the set of trigrams contained in the string."*

`"cat"` → `" c"`, `" ca"`, `"cat"`, `"at "`. 앞에 공백 2개·뒤에 1개를 붙이는 이유는 **단어의 시작과 끝도 트라이그램에 담기 위해서**다. 그래야 3글자 미만 단어도 토큰을 갖는다.

```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_content_trgm ON send_message USING GIN (content gin_trgm_ops);

SELECT * FROM send_message WHERE content LIKE '%휴진안내%';   -- 인덱스를 탄다
```

**어떻게 시작점 없이 인덱스를 타나.** `LIKE`/`ILIKE`는 9.1부터, 정규식 `~`/`~*`는 9.3부터 지원된다.
> *"The index search works by extracting trigrams from the search string and then looking these up in the index... **Unlike B-tree based searches, the search string need not be left-anchored.**"*

**패턴 자체를 트라이그램으로 쪼개서 GIN을 조회하는 것**이다. 검색어를 인덱스 키 순서로 탐색하는 게 아니라 검색어에서 뽑은 토큰들을 키로 직접 꺼낸다. 그래서 앞고정이 필요 없다. §2-1의 "시작점이 없다"가 질문에서 아예 빠지는 것이다.

⚠️ **함정이 하나 있다.** *"a pattern with no extractable trigrams will degenerate to a full-index scan."* `'%가%'`처럼 패턴이 짧으면 뽑을 트라이그램이 없어 인덱스 전체 스캔이 된다. 한국어는 2글자 검색어가 흔해서 이 조건에 잘 걸린다. **`pg_trgm`을 붙였다고 모든 `LIKE`가 빨라지는 게 아니다.**

**보너스로 유사도가 딸려 온다.** `similarity(a, b)`는 0~1을 돌려주고 `%` 연산자는 *"true if its arguments have a similarity that is greater than the current similarity threshold set by `pg_trgm.similarity_threshold`"*. 오타 허용 검색을 extension 하나로 얻는 셈이라, 템플릿 이름처럼 사람이 손으로 치는 짧은 텍스트에 잘 맞는다. (GiST 쪽 `gist_trgm_ops`도 있고 `siglen`(기본 12바이트, 1~2024)으로 시그니처 길이를 조절하지만, §2-6의 lossy 문제가 그대로라 **기본은 `gin_trgm_ops`**다.)

### 2-8. 언제 RDB로 끝내고 언제 ES로 가나

[저장소 선택](../../datastore/selection.md) §2-3은 *"발송 상세 이력·로그 → Elasticsearch"* 라고 한다. **맞는 기본값이다. 다만 뒤집히는 조건이 있다.**

**RDB로 끝나는 신호. 하나라도 강하면 ES는 과하다**
- **검색이 이미 좁혀진 뒤에 걸린다.** `client_id + 기간`으로 수천~수만 건까지 줄인 다음의 본문 키워드 필터
- **랭킹이 "관련도순"이 아니라 "최신순"이다.** 관련도 점수가 필요 없으면 ES의 핵심 가치 절반이 사라진다
- **검색이 조회 화면의 필터 한 칸**이다. 검색 자체가 제품이 아니다
- **정합이 중요하다.** 색인이 밀리는 동안 "방금 보낸 건이 안 보인다"가 사고가 되는 화면. 그리고 운영 인원이 적을 때. 클러스터 하나를 덜 운영하는 값이 성능보다 크다

**ES로 넘어가야 하는 신호**
- **관련도 랭킹·하이라이팅·자동완성·오타 교정**이 요구사항에 있다 (§2-4의 자연어 모드 점수로는 감당이 안 된다)
- **검색과 집계를 같이 한다.** 결과를 채널별·상태별로 즉시 facet
- **여러 테이블·여러 시스템을 한 검색창에서** 찾아야 한다
- **보존 기간 정책이 있다** → 인덱스 통째 삭제(ILM). [Elasticsearch](../../datastore/elasticsearch.md) §2-6
- **형태소 분석 품질이 필요하다**(Nori). n-gram 오탐을 사용자가 체감하기 시작한 시점이다. 또는 색인 부하가 원장 쓰기를 물고 늘어질 때(§2-2의 대가)

**결정적 제약 하나를 따로 적어 둔다.** §2-4에서 봤듯 MySQL 전문 인덱스는 텍스트 컬럼만 담는다. `client_id`를 같은 인덱스에 넣을 방법이 없다.

```sql
WHERE client_id = 1234 AND MATCH(content) AGAINST ('휴진' IN BOOLEAN MODE)
```
→ 두 인덱스가 별개다. 옵티마이저는 한쪽으로 후보를 뽑고 나머지를 필터링한다([DB 인덱스](b-tree-index.md) §2-7). "휴진"이 전체 발송의 5%면 수백만 행을 매칭한 뒤 그중 한 고객사만 남기는 실행계획이 나올 수 있다. **멀티테넌트 발송 이력에서 RDB 전문 검색이 무너지는 지점은 대개 여기다.** 토큰이 흔할수록, 데이터가 클수록 나빠진다. 완화책은 파티셔닝으로 스캔 대상 자체를 줄이는 것이다([DB 인덱스](b-tree-index.md) §2-12). 그래도 안 되면 그때가 ES다.

### 2-9. 닫아 둔 MySQL, 조립하게 둔 PostgreSQL

| | **MySQL 8.0** | **PostgreSQL** |
|---|---|---|
| 진입점 | `FULLTEXT` 인덱스 + `MATCH ... AGAINST` (역색인) | `tsvector` 컬럼 + `@@` + **GIN** (역색인) |
| **대상 타입 제약** | **`CHAR`/`VARCHAR`/`TEXT`만**, 엔진도 InnoDB/MyISAM만 | **제약 없음.** `tsvector`로 변환만 되면 된다 |
| 정규화 위치 | 파서가 색인 시점에 (서버 설정으로) | **`to_tsvector` 호출에 설정 이름 명시** (인덱스에서 필수) |
| 한국어 | **`WITH PARSER ngram`** (형태소 파서는 일본어 MeCab뿐) | 내장 딕셔너리로는 사실상 불가 → **`pg_trgm`** |
| 토큰 크기 | `ngram_token_size` 기본 2, **읽기 전용**(재시작 필요) | 트라이그램 **3 고정** |
| 최소 단어 길이 | InnoDB 3 / MyISAM 4. **ngram에서는 무시됨** | 딕셔너리·불용어가 담당 |
| 검색 모드 | **3종**(자연어 / 불린 / 쿼리 확장) | `tsquery` 연산자(AND·OR·NOT·FOLLOWED BY) |
| **`LIKE '%..%'` 가속** | **없다** (ngram + 불린 구문 검색으로 우회) | **`pg_trgm`**. *"need not be left-anchored"* |
| 오타 허용 · 확장 방식 | 없음 · 파서 플러그인(MeCab 등) | **`similarity()`/`%`** · **extension** ([복제·운영](../../postgresql/replication-and-ops.md) §2-8) |

갈라지는 지점은 하나다. MySQL은 **전문 검색을 인덱스 종류 하나로 닫아 두고** 파서 플러그인으로만 넓힌다. PostgreSQL은 타입·연산자·인덱스·extension을 분리해 조립하게 한다. 그래서 `pg_trgm` 같은 "전문 검색이 아닌 방식의 문자열 검색"이 같은 자리에 끼어들 수 있고, **`LIKE '%..%'`를 인덱스로 태우는 길이 PostgreSQL에만 있다.**

---

**다음으로 읽을 것**

- 왜 앞고정 `LIKE`만 인덱스를 타는가 → [DB 인덱스](b-tree-index.md) §2-6, 자료구조 지도는 §2-2
- 역색인의 기본 원리와 애널라이저 → [Elasticsearch](../../datastore/elasticsearch.md) §2-1
- GIN이 왜 "여러 값을 담은 컬럼"용인가 → [힙과 인덱스](../../postgresql/heap-and-index.md) §2-5, extension이라는 발상 → [PostgreSQL 복제·운영](../../postgresql/replication-and-ops.md) §2-8
- 무엇을 어디에 둘 것인가의 전체 판단 프레임 → [저장소 선택](../../datastore/selection.md) §2-3

---

> **기준 버전**: MySQL 8.0 · PostgreSQL 17 (`pg_trgm` 서술은 9.3 이상 전제)
>
> **확인한 출처**:
> - [MySQL 14.9 Full-Text Search Functions](https://dev.mysql.com/doc/refman/8.0/en/fulltext-search.html) — 지원 타입 `CHAR`/`VARCHAR`/`TEXT`·엔진 InnoDB/MyISAM 한정, `MATCH(...) AGAINST(...)` 구문, 검색어가 *"constant during query evaluation"* 이어야 한다는 제약, 검색 3종
> - [MySQL 14.9.1 Natural Language](https://dev.mysql.com/doc/refman/8.0/en/fulltext-natural-language.html) — 관련도 계산 4요소 원문, **50% 임계가 MyISAM 대상**이라는 서술과 우회법(*"build search indexes on InnoDB tables, or use the boolean search mode"*), **최소 단어 길이 InnoDB 3 / MyISAM 4** · [14.9.2 Boolean](https://dev.mysql.com/doc/refman/8.0/en/fulltext-boolean.html) — 연산자 전체(`+ - @거리 > < ~ * "구문"`), `@거리`가 InnoDB 전용, InnoDB는 *"require a FULLTEXT index on all columns of the MATCH() expression"*, 불린 모드는 50% 임계를 쓰지 않음, 최소 길이·불용어는 적용되나 `*` 붙은 단어는 예외
> - [MySQL 14.9.8 ngram Full-Text Parser](https://dev.mysql.com/doc/refman/8.0/en/fulltext-search-ngram.html) — **기본 토큰 크기 2(bigram), 범위 1~10**, `ngram_token_size`가 **읽기 전용**, 토큰화 예시와 `"ab cd"` → `"ab"`,`"cd"`, **불용어를 *포함*하는 토큰을 제외**한다는 규칙, **`innodb_ft_min_token_size`·`innodb_ft_max_token_size`·`ft_min_word_len`·`ft_max_word_len`이 무시된다**는 목록, 불린 모드의 ngram 구문 변환(`'abc'` → `'"ab bc"'`), 접두가 토큰 크기보다 짧은 와일드카드 동작
> - [MySQL 14.9.9 MeCab Full-Text Parser](https://dev.mysql.com/doc/refman/8.0/en/fulltext-search-mecab.html) — 기본 파서가 공백 구분자에 의존한다는 한계 진술, **MeCab은 일본어용**이며 InnoDB/MyISAM 지원. **한국어 언급 없음**
> - [PostgreSQL 12.1 Introduction](https://www.postgresql.org/docs/17/textsearch-intro.html) — `LIKE`/정규식의 한계 3가지 원문, `tsvector`가 *"a sorted array of normalized lexemes"*, `tsquery`의 AND/OR/NOT/FOLLOWED BY, `@@` 연산자, lexeme 정규화(소문자화·접미사 제거·불용어 제거)와 딕셔너리
> - [PostgreSQL 12.2 Tables and Indexes](https://www.postgresql.org/docs/17/textsearch-tables.html) — `USING GIN (to_tsvector('english', body))`, **2인자 형태만 표현식 인덱스에 쓸 수 있는 이유**(*"index contents must be unaffected by default_text_search_config"*), `GENERATED ALWAYS AS ... STORED` 예시와 장점 2가지
> - [PostgreSQL 12.9 GiST/GIN Index Types](https://www.postgresql.org/docs/17/textsearch-indexes.html) — *"GIN indexes are the preferred text search index type"*, GiST가 lossy이며 고정 길이 시그니처의 해시 충돌로 오탐→행 재검사가 필요하다는 서술, GIN이 weight label을 저장하지 않는다는 서술 · [12.6 Dictionaries](https://www.postgresql.org/docs/17/textsearch-dictionaries.html) — `simple` 딕셔너리가 소문자화 + 불용어 대조만 한다는 원문, 정규화 수단이 Snowball 어간 추출이라는 서술. **이 페이지에 한국어·CJK 언급 없음**
> - [PostgreSQL F.35 pg_trgm](https://www.postgresql.org/docs/17/pgtrgm.html) — 트라이그램 정의, **앞 공백 2 + 뒤 공백 1** 규칙과 `"cat"` 예시, `gin_trgm_ops`/`gist_trgm_ops`(`siglen` 기본 12바이트, 1~2024), `LIKE`/`ILIKE` 9.1+·정규식 9.3+ 지원과 ***"the search string need not be left-anchored"***, *"a pattern with no extractable trigrams will degenerate to a full-index scan"*, `similarity()`·`%`·`pg_trgm.similarity_threshold`
>
> **미확인**:
> - **PostgreSQL 한국어 전문 검색 수단** — 공식 딕셔너리 장에 한국어가 없다는 것까지만 확인했다. `pg_bigm` 등 서드파티 extension의 존재·성숙도는 확인하지 않았다. **`btree_gin`으로 스칼라 컬럼과 `tsvector`를 한 GIN 인덱스에 담을 수 있는지**(§2-8의 MySQL 제약을 PG가 우회하는 길로 자주 언급된다)도 공식 문서로 확인하지 않았다
> - **ngram 색인이 원본 대비 몇 배로 커지는지** — 토큰 수가 대략 `L−n+1`인 건 정의에서 나오지만 **공식 문서에 실제 크기 배수는 없다.** §2-3의 "크다"는 정성 서술이다. **InnoDB 자연어 모드에 50% 임계에 준하는 컷오프가 있는지**도 미확인 — 공식 문서는 MyISAM 대상으로만 진술한다
> - **§2-8의 실행계획 서술** — "전문 인덱스에 스칼라 컬럼을 담을 수 없다"는 확인된 제약에서 **추론**한 것이고, `EXPLAIN`으로 검증하지 않았다
>
> **미작성**: ES Nori 형태소 분석기 설정 · `INFORMATION_SCHEMA.INNODB_FT_*`를 통한 색인 진단 · `ts_rank`/`setweight` 랭킹 · 하이라이팅(`ts_headline`) · 자동완성·오타 교정 구현
