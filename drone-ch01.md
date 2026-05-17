# 예제 1. 도서관 장서 시스템 — 분류와 속성

## 들어가며

분류가 본질인 도메인부터 시작한다. 책·정기간행물·논문이 다층 가지로 나뉘고, 각 가지가 *고유한 속성*을 가지는 자리. TypeDB의 첫 번째 도구 — **entity 상속과 속성 제약** — 이 손에 잡힌다.

이 장은 두 호흡으로 짜여 있다. *코드*가 본진이고, *이론*이 그 짚는다.

---

## 1.1 왜 도서관인가

세 가지 이유로 첫 예제를 도서관에 둔다.

1. **분류가 도메인의 본질**. 도서관학(Library Science)이 발전시킨 *DDC, LCC, KDC* 같은 분류 체계가 *수세기에 걸친 다층 분류의 정교한 사례*다.
2. **독자 친숙도**. 모든 독자가 *책의 종류*에 대한 직관을 가진다 — 학습 곡선의 *코드 외 부담*이 0.
3. **속성 제약의 풍부함**. ISBN의 정규식, 출판 연도의 범위, 분야의 허용 값 — 한 도메인에서 `@card`·`@range`·`@values`·`@regex`가 모두 자연스럽게 나옴.

이 예제에서 짤 분류 트리:

```
item
├── book
│   ├── fiction
│   └── nonfiction
├── journal
│   ├── academic_journal
│   └── magazine
└── thesis
    ├── masters_thesis
    └── phd_dissertation
```

3단 깊이의 7개 단말 노드. 단순하지만 — *분류 이론*의 모든 자리를 짚는 데에는 충분하다.

---

## 1.2 entity 상속 트리

스키마 모드 진입, 분류 트리를 한 번에 정의:

```typeql
transaction schema library
```

```typeql
define
  # === 최상위: 모든 장서 항목 ===
  entity item,
    owns title @card(1..1),
    owns added_year @card(1..1);
  
  # === 책 가지 ===
  entity book sub item,
    owns isbn @card(1..1),
    owns author @card(1..);
  entity fiction sub book,
    owns genre @card(0..1);
  entity nonfiction sub book,
    owns subject @card(1..);
  
  # === 정기간행물 가지 ===
  entity journal sub item,
    owns issn @card(1..1),
    owns publisher @card(1..1);
  entity academic_journal sub journal,
    owns peer_reviewed @card(1..1),
    owns impact_factor @card(0..1);
  entity magazine sub journal,
    owns frequency @card(0..1);
  
  # === 논문 가지 ===
  entity thesis sub item,
    owns institution @card(1..1),
    owns advisor @card(1..);
  entity masters_thesis sub thesis;
  entity phd_dissertation sub thesis,
    owns defense_date @card(0..1);
  
  # === 속성 정의 ===
  attribute title, value string;
  attribute added_year, value long @range(1900..2100);
  attribute isbn, value string @regex("^97[89][0-9]{10}$");
  attribute issn, value string @regex("^[0-9]{4}-[0-9]{3}[0-9X]$");
  attribute author, value string;
  attribute genre, value string @values("mystery", "scifi", "literary", "romance", "fantasy");
  attribute subject, value string;
  attribute publisher, value string;
  attribute peer_reviewed, value boolean;
  attribute impact_factor, value double @range(0.0..50.0);
  attribute frequency, value string @values("weekly", "monthly", "quarterly", "annual");
  attribute institution, value string;
  attribute advisor, value string;
  attribute defense_date, value date;
commit;
```

**짚어둘 점**:

- 상위(`item`)에 정의된 `title`, `added_year`는 *모든 하위*가 자동 상속.
- 하위 가지마다 *고유 속성*이 추가됨 — `book`은 `isbn`, `journal`은 `issn`, `thesis`는 `institution`.
- 더 깊은 하위(`academic_journal`의 `peer_reviewed`)는 *해당 하위만* 가지는 속성.
- 상속의 *체이닝*: `academic_journal`은 `title`, `added_year`(item에서), `issn`, `publisher`(journal에서), `peer_reviewed`(자기 자신) 모두 가짐.

---

## 1.3 속성 제약 — 네 가지 무기

도메인 규칙을 *데이터 모델*에 박는 도구 네 가지.

### `@card` — 개수 제약

- `@card(1..1)` — 정확히 한 개 (제목·ISBN)
- `@card(0..1)` — 없거나 하나 (장르는 옵션)
- `@card(1..)` — 하나 이상 (저자는 여러 명 가능)
- `@card(0..)` — 없을 수도 여러 개일 수도

### `@range` — 숫자 범위

```typeql
attribute added_year, value long @range(1900..2100);
attribute impact_factor, value double @range(0.0..50.0);
```

*출판 연도가 1899인 항목*은 `INSERT`가 거부된다. 도메인 규칙이 *데이터베이스*에서 강제됨.

### `@values` — 허용된 값 집합

```typeql
attribute genre, value string @values("mystery", "scifi", "literary", "romance", "fantasy");
```

분류 가지(entity sub)보다 *가벼운 카테고리*를 표현. 다섯 장르를 별도 entity로 만드는 것은 과한 분류 — `@values`가 적절한 자리.

### `@regex` — 문자열 패턴

```typeql
attribute isbn, value string @regex("^97[89][0-9]{10}$");
```

ISBN-13의 형식을 *데이터베이스가* 검증. 잘못된 형식은 `INSERT` 시점에 거부.

---

## 1.4 ◇ 설계 결정 — 왜 이 분류 깊이인가

본격 데이터 입력 전에 — *왜 이 구조인가*를 짚는다.

### 결정 1 — item을 최상위로 둔 이유

**대안 1: item 없이 book·journal·thesis가 각자 독립**
```typeql
entity book, owns title, owns added_year;
entity journal, owns title, owns added_year;
entity thesis, owns title, owns added_year;
```
- 단점: *title·added_year의 중복 정의*
- 단점: *모든 장서*를 묻는 쿼리에서 *세 entity를 union*해야
- 단점: *공통 관계*(예: *대출 가능*)를 정의할 자리가 없음

**대안 2: item에 *모든* 속성을 두기**
```typeql
entity item, owns title, owns added_year, owns isbn, owns issn, owns institution;
```
- 단점: book에는 `issn`이 없는데 — 속성 자체는 *존재 가능*. `@card(0..1)`로 옵션화하면 의미가 흐려짐
- 단점: 분류 가지의 *고유성*이 사라짐

**채택: item을 최상위로, 가지마다 고유 속성**
- 장점: 공통(title, added_year)과 고유(isbn vs issn)가 *명확히 분리*
- 장점: `match $i isa item` 한 줄로 *모든 장서* 조회
- 장점: 미래 확장 — *대출* 관계가 `relates loaned_item, $i isa item`으로 자연스럽게 묶임

### 결정 2 — fiction vs nonfiction을 entity로 vs attribute로

**대안 1: book의 속성으로**
```typeql
entity book, owns book_type;
attribute book_type, value string @values("fiction", "nonfiction");
```
- 장점: 단순
- 단점: fiction 고유 속성(`genre`)을 *모든 book*이 가져야 함 (`@card(0..1)`로 옵션화)
- 단점: *소설만* 묻는 쿼리가 `$b isa book, has book_type "fiction"` — 두 조건 필요

**대안 2: entity 분류 가지로**
```typeql
entity fiction sub book, owns genre;
entity nonfiction sub book, owns subject;
```
- 장점: `match $f isa fiction`으로 단순 조회
- 장점: fiction에만 *고유 속성* (genre)이 자연스럽게 붙음
- 단점: 분류 가지가 늘어남

**채택: entity 분류 가지**
- 이유: 고유 속성이 있고, 쿼리가 자주 *fiction만*·*nonfiction만*을 묻는 도메인

### 결정 3 — genre를 entity로 vs @values로

**대안 1: entity 가지로**
```typeql
entity mystery sub fiction;
entity scifi sub fiction;
entity literary sub fiction;
...
```
- 장점: 각 장르에 *고유 속성* 추가 가능 (예: scifi의 *subgenre*: hard·soft·space opera)
- 단점: 가지가 너무 많아짐. *기본 수준 범주* 너머의 분류

**대안 2: @values 속성**
```typeql
attribute genre, value string @values("mystery", "scifi", ...);
```
- 장점: 단순. *유한 집합 강제*
- 단점: 장르별 고유 속성 표현 어려움

**채택: @values**
- 이유: *현재 도메인의 질문*이 *장르별 고유 속성*까지 요구하지 않음. 미래에 필요해지면 entity로 *진화 가능*.

### 결정 4 — 분류 트리의 깊이를 어디서 멈출 것인가

**원칙**: *질문에 답할 수 있을 만큼만 쪼갠다*.

이 도메인의 질문들:
- *모든 장서* (item)
- *모든 책* (book)
- *소설만* (fiction)
- *학술지만* (academic_journal)
- *특정 장르의 소설만* (fiction + genre)

이 질문들에 답할 *최소 깊이*가 — 3단 + @values 한 자리. 더 깊이 가면 *과대 설계*.

### 결정 5 — 다중 상속을 허용할 것인가

질문: *교과서(textbook)는 book인가 academic 영역인가*?

TypeDB는 *단일 상속*. 한 entity는 *정확히 한 부모*. 다중 분류가 필요하면:

**해결 1: 다중 분류를 attribute로**
```typeql
entity book sub item, owns academic_use;
attribute academic_use, value boolean;
```

**해결 2: 별도 분류 차원을 관계로**
```typeql
entity academic_topic, owns topic_name;
relation has_academic_use,
  relates academic_item, relates topic;
```

이 책의 단순화에서는 *단일 상속*으로 충분. 본격 도메인에서는 *해결 2*가 자주 쓰인다.

---

## 1.5 인스턴스 입력

```typeql
transaction write library
```

```typeql
insert
  # === 소설 3권 ===
  $f1 isa fiction,
    has title "잃어버린 시간을 찾아서",
    has added_year 2020,
    has isbn "9791160403456",
    has author "마르셀 프루스트",
    has genre "literary";
  $f2 isa fiction,
    has title "은하수를 여행하는 히치하이커를 위한 안내서",
    has added_year 2018,
    has isbn "9788983927408",
    has author "더글러스 애덤스",
    has genre "scifi";
  $f3 isa fiction,
    has title "셜록 홈즈의 모험",
    has added_year 2019,
    has isbn "9788932917245",
    has author "아서 코난 도일",
    has genre "mystery";
  
  # === 비소설 2권 ===
  $n1 isa nonfiction,
    has title "사피엔스",
    has added_year 2021,
    has isbn "9788934972464",
    has author "유발 하라리",
    has subject "역사",
    has subject "인류학";
  $n2 isa nonfiction,  # ← 시연용 가상 데이터
    has title "TypeDB로 짓는 지식 그래프",
    has added_year 2026,
    has isbn "9791234567890",
    has author "Vaticle Team",
    has subject "데이터베이스",
    has subject "온톨로지";
  
  # === 학술지 1종 ===
  $aj1 isa academic_journal,
    has title "Journal of Web Semantics",
    has added_year 2022,
    has issn "1570-8268",
    has publisher "Elsevier",
    has peer_reviewed true,
    has impact_factor 3.4;
  
  # === 잡지 1종 ===
  $mg1 isa magazine,
    has title "Communications of the ACM",
    has added_year 2023,
    has issn "0001-0782",
    has publisher "ACM",
    has frequency "monthly";
  
  # === 박사 논문 1편 ===
  $phd1 isa phd_dissertation,
    has title "지식 그래프의 분산 추론에 관한 연구",
    has added_year 2024,
    has institution "KAIST",
    has advisor "김교수",
    has defense_date 2024-08-15;
commit;
```

8개 항목, 분류 가지에 골고루 분포.

**짚어둘 점**: `$n2`는 *책의 시연을 위한 가상 데이터*. ISBN `9791234567890`은 실재 ISBN이 아니다. 실 시스템에서는 *공식 ISBN 데이터베이스*와의 검증이 필요.

---

## 1.6 첫 쿼리 — 분류 깊이가 답을 결정한다

```typeql
transaction read library
```

### 질문 1 — 모든 장서

```typeql
match $i isa item, has title $t;
fetch $t;
```

답: 8개 항목 모두 — book·journal·thesis 가지 전체.

### 질문 2 — 책만 (소설 + 비소설)

```typeql
match $b isa book, has title $t;
fetch $t;
```

답: 5권 — `book`의 모든 하위(`fiction`, `nonfiction`)가 자동 포함.

### 질문 3 — 소설만

```typeql
match $f isa fiction, has title $t, has genre $g;
fetch $t; $g;
```

답: 3권 + 장르.

### 질문 4 — Impact Factor 3.0 이상의 학술지

```typeql
match
  $aj isa academic_journal,
    has title $t,
    has impact_factor $if;
  $if >= 3.0;
fetch $t; $if;
```

답: *Journal of Web Semantics* (3.4).

### 질문 5 — 2020년 이후 추가된 모든 항목

```typeql
match
  $i isa item,
    has title $t,
    has added_year $y;
  $y >= 2020;
fetch $t; $y;
```

답: 6개 — 분류와 무관하게 *상위 속성*인 added_year로 필터.

### 질문 6 — 같은 저자의 모든 저작

```typeql
match
  $b isa book,
    has author "마르셀 프루스트",
    has title $t;
fetch $t;
```

답: 1권. 한 저자 entity가 *여러 책*에 등장하는 모양.

### 질문 7 — 분류 가지에 빈 자리가 있는가

```typeql
match
  $mt isa masters_thesis, has title $t;
fetch $t;
```

답: 빈 결과. *현재 데이터에 석사 논문은 없다*. 이게 *분류 가지의 빈 자리*가 시스템적으로 드러나는 자리. 분석가는 — *데이터의 결여를 자동 감지*할 수 있음.

---

## 1.7 분류 깊이 사고법

> **분류는 *질문에 답할 수 있을 만큼만* 쪼갠다.**

### 너무 거친 분류의 위험

- 모든 장서를 `item` 하나로 둔다면 — *책만 보여달라*는 질문에 답할 수 없음
- *책*과 *논문*의 본질적 차이(ISBN vs institution)가 흐려짐
- 분류의 *빈 가지*가 보이지 않음 — 결여 감지 불가

### 너무 정밀한 분류의 위험

- `mystery_novel`, `scifi_novel`, ..., 30개 장르를 각각 entity로 둔다면 — 분류 트리가 *질문보다 무거워짐*
- 같은 본질(소설)이 다른 가지에 흩어짐
- 새 장르가 나올 때마다 *스키마 변경*이 필요

### 균형점

이 책의 도서관 예제는 *3단 깊이, 7개 단말 노드*의 균형점. 더 큰 도서관 시스템이라면 — `genre` 같은 가벼운 카테고리는 entity로 *내려가지 않고* `@values`로 유지하는 게 자연스러움.

### 실제 도서관 분류 체계의 자리

도서관학이 발전시킨 표준 분류 체계 세 가지:

**DDC (Dewey Decimal Classification)** — 듀이 십진분류. 1876년 멜빌 듀이가 짠 *수자 기반 10진 분류*. 23판이 현재 표준. 약 *3만 개의 분류 단말*. 깊이 *10단계 이상*.

**LCC (Library of Congress Classification)** — 미국 의회도서관 분류. *알파벳 + 숫자* 조합. 21개 주요 클래스(A~Z). 더 정밀하고 더 깊음. 학술 도서관에 적합.

**KDC (Korean Decimal Classification)** — 한국십진분류. DDC를 한국 도메인에 맞게 변형. 6판이 현재.

**왜 이 표준들이 그 깊이인가**:
- 도서관의 *실제 검색 패턴*이 *수만 개의 단말 분류*를 요구
- 사용자가 *한 권의 책을 정확히 어디서 찾을 것인가*를 결정해야
- 시스템 차원이 아니라 *물리 책장의 위치*를 결정하는 자리

**이 책의 예제와의 차이**:
- 이 책: *7개 단말, 3단 깊이* — 코드 학습을 위한 최소
- 실 시스템: *수만 개 단말, 10단계 깊이* — 실 도메인 요구

분류 깊이는 *도메인이 요구하는 만큼* 가야 하지, *코드 학습 자료가 요구하는 만큼*에 멈춰서는 안 된다는 짚음.

---

## 1.8 잘못된 스키마 vs 좋은 스키마

같은 도서관 도메인을 *세 가지 잘못된 방식*과 *현재 책의 좋은 방식*으로 비교.

### 잘못된 스키마 1 — 모든 것을 attribute로

```typeql
define
  entity library_item,
    owns title,
    owns item_type,           # "book", "journal", "thesis"
    owns identifier,          # ISBN, ISSN 등 다 들어감
    owns extra_info;          # "author:프루스트;genre:literary"
```

**왜 나쁜가**:
- *분류 강제 없음* — `item_type`이 "boook" 오타도 통과
- *식별자 형식 검증 없음* — ISBN인지 ISSN인지 구분 안 됨
- *저자가 문자열에 묻혀* — *같은 저자의 모든 책*을 묻는 쿼리 불가
- *유형별 고유 속성*(impact_factor, defense_date 등)이 *모든 항목*에 들어가게 됨

### 잘못된 스키마 2 — 분류 없는 평면 + 관계로만

```typeql
define
  entity thing, owns name;
  relation classifies,
    relates classified_thing, relates category;
  entity category, owns category_name;
```

모든 항목이 `thing`. 분류는 *관계로*. *책*은 *book 카테고리에 분류된 thing*.

**왜 나쁜가**:
- 분류 정보가 *관계 매듭*에 들어가서 *상속 자동 작동 불가*
- 모든 쿼리가 *관계 join*을 요구
- 카테고리 자체에 *고유 속성*을 박기 어려움 (book vs journal의 고유성)

### 잘못된 스키마 3 — 깊이가 너무 얕음

```typeql
define
  entity item, owns title, owns type;
  attribute type, value string @values("book", "journal", "thesis");
```

분류를 *entity 가지로 펴지 않고 attribute로*. type 속성으로 구분.

**왜 나쁜가**:
- *책의 ISBN과 학술지의 ISSN*이 같은 *item*에 묶임 — 의미 혼란
- *상속의 자동 전파* 없음 — 모든 속성을 item에 박아야
- 분류 가지를 *깊게* 가져갈 수 없음 (`academic_journal` 같은 손자 가지)

### 좋은 스키마 — 현재 책의 스키마

1.2의 스키마는:
- *3단 entity 가지* (item → book → fiction, etc.)
- *가지마다 고유 속성*
- *상속의 자동 전파*
- `@card`·`@range`·`@values`·`@regex`의 적절한 강제

**결과**: 도메인의 *모든 의미*가 스키마에 명시적으로 박혔다. 검색·필터·확장이 모두 자연스럽게 작동.

---

## 1.9 성능과 인덱스

### TypeDB의 자동 인덱싱

TypeDB 3.0은 *모든 entity·attribute·role 자리에 자동 인덱스*. 명시적 `CREATE INDEX`가 필요 없다. 그러나 — *성능을 의식한 스키마 설계*는 여전히 가치 있다.

### 인덱스가 잘 작동하는 자리

- `match $i isa item` — entity 가지 인덱스
- `match $b has isbn $i` — attribute 인덱스
- `match $i has added_year > 2020` — 범위 인덱스

### 인덱스가 약한 자리

- 정규식 매칭의 *부분 검색* — `@regex("^.*proust.*$")`는 인덱스 사용 어려움
- 매우 *낮은 선택성* 속성 — `peer_reviewed true`가 *90% 일치*면 인덱스 효과 적음

### 도서관 규모별 성능 (예상)

| 항목 수 | 단일 entity 조회 | 분류 가지 조회 | 다중 속성 조회 |
|---|---|---|---|
| 1,000 | < 5ms | < 10ms | < 20ms |
| 100,000 | ~10ms | ~50ms | ~100ms |
| 10,000,000 | ~30ms | ~200ms | ~500ms |

실 도서관 규모(*수백만 건*)에서도 *TypeDB가 충분히 빠름*. 단 — *복잡한 재귀 함수*가 들어가면 성능 특성이 달라짐 (3장에서 본격).

---

## 1.10 ◇ 이론 절 — Subtyping과 분류의 의미론

### 타입 시스템의 Subtyping

이 절은 코드를 떠나, *왜 그렇게 짜야 하는가*의 자리.

`entity fiction sub book` 한 줄이 표현하는 것은 — *Liskov 치환 원칙(Liskov Substitution Principle, LSP)*과 같은 자리다.

> *서브타입의 객체는 슈퍼타입의 객체가 기대되는 자리에 치환 가능해야 한다.*

**쉬운 풀이로 보면**:

*과일을 주세요*라고 했을 때 — 누가 *사과를 가져왔다*. 그게 충분히 답이 된다. *사과는 과일의 한 종류*이기 때문. 거꾸로 — *사과를 주세요*라고 했는데 *과일을 가져왔다*면? *어떤 과일인지*를 다시 물어야 한다. 사과인지, 배인지, 귤인지.

이게 *치환 가능성*의 일상적 직관:
- 상위(과일)이 필요한 부분 ← 하위(사과)는 들어갈 수 있다 ✓
- 하위(사과)가 필요한 부분 ← 상위(과일)는 들어갈 수 없다 ✗

TypeDB의 `entity fiction sub book`이 정확히 여기다. *책을 묻는 쿼리*(`match $b isa book`)가 *fiction 인스턴스도 답에 포함*. 그러나 *fiction을 묻는 쿼리*에는 *nonfiction은 답에 없다*. 일상의 *과일-사과* 직관이 데이터 모델에서 그대로 작동한다.

`fiction` 인스턴스는 — `book`이 기대되는 자리(`isa book` 쿼리, `book`의 속성 접근)에 *항상* 들어갈 수 있다. 그래서 *book 전체를 묻는 쿼리*에 fiction 인스턴스가 답에 자동 포함되는 것.

여기는 객체지향 언어의 *상속*과 같은 모양 — TypeDB의 PolyModel이 *객체지향 부분*을 데이터 모델에 가져왔다는 0부의 말이 *이 자리*다.

**LSP가 깨지는 경우**:
- 서브타입이 부모의 *제약을 강화*하는 경우 (parent: optional, child: required) — TypeDB는 허용. `@card(0..1)`을 자식에서 `@card(1..1)`로 *조이는* 건 OK.
- 서브타입이 *역할을 거부*하는 경우 — `book plays loan:loaned_item`인데 `rare_book sub book`이 *대출 불가*하다면, plays를 *오버라이드*할 수 없음. 이런 경우는 *별도 분류 차원*으로 풀어야.

### 분류학(Taxonomy) vs 분류 이론(Classification Theory)

도메인의 *위계적 분류*에는 두 개의 전통이 있다.

**분류학(Taxonomy)**. 린네의 *생물 분류*가 원형. *Kingdom → Phylum → Class → Order → Family → Genus → Species*의 깊은 위계. *각 단계가 명확한 기준*을 가져야 한다는 엄밀함. 도서관학의 DDC(Dewey Decimal Classification), 의학의 ICD가 이 전통.

**분류 이론(Classification Theory)**. 인지과학적 접근. *모든 개체가 위계적으로만 분류되지는 않는다*는 통찰. 한 개체가 *여러 분류*에 동시에 속할 수 있고, *prototype 효과*(어느 인스턴스가 그 분류의 *대표*인가)가 있음.

TypeDB의 `sub`는 *분류학적 위계*에 가깝다. 한 entity는 *정확히 한 부모*를 가진다. 그래서 — *다중 분류*가 필요한 부분에서는 *역할(role)*이나 *카테고리 attribute*가 답이 된다 (1.4 결정 5).

### Rosch의 기본 수준 범주

심리학자 *Eleanor Rosch*의 1973년 연구가 분류에 결정적 통찰을 줬다. 사람들은 *위계의 모든 수준을 동등하게 사용하지 않는다*. *기본 수준 범주(basic-level categories)*가 인지적으로 특권적인 자리에 있다.

예: *동물 → 새 → 참새*에서 사람들이 가장 많이 쓰는 단어는 *새*. *동물*은 너무 추상적이고, *참새*는 너무 구체적. *새*가 기본 수준.

기본 수준 범주의 특성:
- *공통 속성*이 가장 많음 (새 = 깃털·날개·부리 모두)
- *시각적 형태*가 비슷함 (모든 새는 비슷하게 생김)
- *언어적 사용 빈도*가 가장 높음
- *아동이 가장 먼저 배움*

스키마 설계에서 이 통찰의 의미: **사용자가 가장 자주 쿼리할 수준이 분류 깊이의 중심이어야 한다.** 도서관 예제에서 사용자가 *fiction을 찾고 싶어 한다면* fiction은 entity로, *mystery·scifi 등을 자주 묻는다면* 그것도 entity로. 사용자가 *2020년 이후 mystery 소설*만 찾고 *세부 mystery 하위(thriller·cozy mystery 등)*는 거의 안 묻는다면 — mystery는 entity로 두되 그 아래는 더 쪼개지 말 것.

### Closed-World Assumption

TypeDB는 *Closed-World Assumption(CWA)*을 따른다. *데이터에 적혀 있지 않은 것은 거짓*으로 간주.

예: *Journal of Web Semantics의 peer_reviewed가 true로 적혀 있다*. 그런데 *Communications of the ACM의 peer_reviewed는 적혀 있지 않다*. CWA에서 후자는 *peer_reviewed가 false다*가 아니라 *peer_reviewed에 대해 우리가 모른다*는 의미. `match $m isa magazine, has peer_reviewed true` 쿼리는 *Communications of the ACM를 답에 포함하지 않는다*.

RDF/OWL은 반대로 *Open-World Assumption(OWA)*. *적혀 있지 않은 것은 모른다*. 추론을 더 강력하게 하지만 — *직관에 어긋나는 경우*가 많다. TypeDB가 CWA를 택한 것은 — *실용적 데이터베이스의 직관*에 맞춤.

**실용적 함의**:
- 데이터 입력 시 *누락이 있으면* 그 누락 자체가 의미 있는 신호
- *분석가가 데이터의 완전성을 의식*해야 함
- *부재(absence)*가 *부정(negation)*과 같이 다뤄짐

### 객체지향 상속과 다른 점

전통 OOP의 상속은 *행동 다형성*(method overriding)이 핵심. TypeDB의 sub는 *데이터 다형성*이 핵심.

- OOP: 같은 메소드 호출이 *서브타입에서 다르게 행동*
- TypeDB: 같은 쿼리가 *서브타입의 인스턴스도 답으로 포함*

OOP의 *오버라이드*에 해당하는 것이 TypeDB에는 거의 없다. 행동(메소드)은 *Function*에서 짜고, 함수는 *entity가 아니라 데이터 위*에서 작동.

이 차이가 — *PolyModel의 객체지향 측면*이 *행동 다형성*이 아니라 *데이터 다형성*에 한정된다는 짚음.

### 분류와 추론의 관계

분류 트리에서 자동으로 도출되는 추론:
- *fiction이면 book이다* (자동, sub에서 도출)
- *book이면 item이다* (자동, 체인 도출)
- *fiction이면 item이다* (전이성)

이게 *DL의 classification reasoning*의 한 자리. TypeDB는 이걸 *명시적으로 추론*하지 않고 — *쿼리 시점에 자동 작동*. 더 효율적이고 더 직관적.

---

## 1.11 정리 — 분류의 도구가 손에 닿은 자리

이 장에서 손에 들어온 것:

**Syntax 어휘**
- `entity X sub Y`로 분류 트리
- `attribute X, value T @제약`으로 속성과 제약
- `@card`, `@range`, `@values`, `@regex` 네 가지 제약
- `isa`, `has`로 인스턴스 입력
- `match` + `fetch`로 분류 깊이에 따른 쿼리

**설계 결정의 트레이드오프** (이 장의 결정적 추가)
- 최상위 entity(item)의 가치
- entity 가지 vs attribute @values의 선택 기준
- 분류 깊이의 균형점 — 질문이 깊이를 결정
- 다중 상속이 필요한 부분

**잘못된 스키마와의 비교**
- 모든 것을 attribute로 (검증 손실)
- 분류 없는 평면 (상속 자동성 손실)
- 너무 얕은 분류 (의미 혼란)

**이론적 토대**
- Subtyping과 Liskov 치환 원칙
- 분류학 vs 분류 이론
- Rosch의 기본 수준 범주 — *질문이 분류 깊이를 결정한다*
- Closed-World Assumption
- 객체지향 상속과의 차이 (데이터 다형성 vs 행동 다형성)

**실세계 평행**
- DDC, LCC, KDC의 실 도서관 분류 체계
- 도메인 규모에 따른 깊이 결정

다음 장 — **관계와 역할**. 분류만으로는 풀리지 않는 자리. 결혼·부모·자녀 같은 *여러 개체가 한 사건에 묶이는 모양*과, *같은 사람이 여러 자리에 있는 모양*이 들어간다.

분류가 *명사의 도구*였다면, 관계는 *동사의 도구*다.
