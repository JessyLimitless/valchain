# 예제 1. 도서관 장서 시스템 — 분류와 속성

## 들어가며

분류가 본질인 도메인부터 시작한다. 책·정기간행물·논문이 다층 가지로 나뉘고, 각 가지가 *고유한 속성*을 가지는 자리. TypeDB의 첫 번째 도구 — **entity 상속과 속성 제약** — 이 손에 잡힌다.

이 장은 두 호흡으로 짜여 있다. *코드*가 본진이고, *이론*이 그 자리를 짚는다.

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

**짚어둘 자리**:

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

## 1.4 인스턴스 입력

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
  $n2 isa nonfiction,
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
    has title "ACM Communications",
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

---

## 1.5 첫 쿼리 — 분류 깊이가 답을 결정한다

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

---

## 1.6 분류 깊이 사고법

> **분류는 *질문에 답할 수 있을 만큼만* 쪼갠다.**

### 너무 거친 분류의 위험

- 모든 장서를 `item` 하나로 둔다면 — *책만 보여달라*는 질문에 답할 수 없음
- *책*과 *논문*의 본질적 차이(ISBN vs institution)가 흐려짐

### 너무 정밀한 분류의 위험

- `mystery_novel`, `scifi_novel`, ..., 30개 장르를 각각 entity로 둔다면 — 분류 트리가 *질문보다 무거워짐*
- 같은 본질(소설)이 다른 가지에 흩어짐
- 새 장르가 나올 때마다 *스키마 변경*이 필요

### 균형점

이 책의 도서관 예제는 *3단 깊이, 7개 단말 노드*의 균형점. 더 큰 도서관 시스템이라면 — `genre` 같은 가벼운 카테고리는 entity로 *내려가지 않고* `@values`로 유지하는 게 자연스러움.

---

## 1.7 ◇ 이론 절 — Subtyping과 분류의 의미론

### 타입 시스템의 Subtyping

이 절은 코드를 떠나, *왜 그렇게 짜야 하는가*의 자리.

`entity fiction sub book` 한 줄이 표현하는 것은 — *Liskov 치환 원칙(Liskov Substitution Principle, LSP)*과 같은 자리다.

> *서브타입의 객체는 슈퍼타입의 객체가 기대되는 자리에 치환 가능해야 한다.*

`fiction` 인스턴스는 — `book`이 기대되는 자리(`isa book` 쿼리, `book`의 속성 접근)에 *항상* 들어갈 수 있다. 그래서 *book 전체를 묻는 쿼리*에 fiction 인스턴스가 답에 자동 포함되는 것.

이 자리는 객체지향 언어의 *상속*과 같은 모양 — TypeDB의 PolyModel이 *객체지향 부분*을 데이터 모델에 가져왔다는 0부의 말이 *이 자리*다.

### 분류학(Taxonomy) vs 분류 이론(Classification Theory)

도메인의 *위계적 분류*에는 두 개의 전통이 있다.

**분류학(Taxonomy)**. 린네의 *생물 분류*가 원형. *Kingdom → Phylum → Class → Order → Family → Genus → Species*의 깊은 위계. *각 단계가 명확한 기준*을 가져야 한다는 엄밀함. 도서관학의 DDC(Dewey Decimal Classification), 의학의 ICD가 이 전통.

**분류 이론(Classification Theory)**. 인지과학적 접근. *모든 개체가 위계적으로만 분류되지는 않는다*는 통찰. 한 개체가 *여러 분류*에 동시에 속할 수 있고, *prototype 효과*(어느 인스턴스가 그 분류의 *대표*인가)가 있음.

TypeDB의 `sub`는 *분류학적 위계*에 가깝다. 한 entity는 *정확히 한 부모*를 가진다. 그래서 — *다중 분류*가 필요한 자리에서는 *역할(role)*이나 *카테고리 attribute*가 답이 된다 (1.6에서 본 `@values`의 자리).

### Rosch의 기본 수준 범주

심리학자 *Eleanor Rosch*의 1973년 연구가 분류에 결정적 통찰을 줬다. 사람들은 *위계의 모든 수준을 동등하게 사용하지 않는다*. *기본 수준 범주(basic-level categories)*가 인지적으로 특권적인 자리에 있다.

예: *동물 → 새 → 참새*에서 사람들이 가장 많이 쓰는 단어는 *새*. *동물*은 너무 추상적이고, *참새*는 너무 구체적. *새*가 기본 수준.

스키마 설계에서 이 통찰의 의미: **사용자가 가장 자주 쿼리할 수준이 분류 깊이의 중심이어야 한다.** 도서관 예제에서 사용자가 *fiction을 찾고 싶어 한다면* fiction은 entity로, *mystery·scifi 등을 자주 묻는다면* 그것도 entity로. 사용자가 *2020년 이후 mystery 소설*만 찾고 *세부 mystery 하위(thriller·cozy mystery 등)*는 거의 안 묻는다면 — mystery는 entity로 두되 그 아래는 더 쪼개지 말 것.

### Closed-World Assumption

TypeDB는 *Closed-World Assumption(CWA)*을 따른다. *데이터에 적혀 있지 않은 것은 거짓*으로 간주.

예: *Journal of Web Semantics의 peer_reviewed가 true로 적혀 있다*. 그런데 *ACM Communications의 peer_reviewed는 적혀 있지 않다*. CWA에서 후자는 *peer_reviewed가 false다*가 아니라 *peer_reviewed에 대해 우리가 모른다*는 의미. `match $m isa magazine, has peer_reviewed true` 쿼리는 *ACM Communications를 답에 포함하지 않는다*.

RDF/OWL은 반대로 *Open-World Assumption(OWA)*. *적혀 있지 않은 것은 모른다*. 추론을 더 강력하게 하지만 — *직관에 어긋나는 경우*가 많다. TypeDB가 CWA를 택한 것은 — *실용적 데이터베이스의 직관*에 맞춤.

---

## 1.8 정리 — 분류의 도구가 손에 닿은 자리

이 장에서 손에 들어온 것:

**Syntax 어휘**
- `entity X sub Y`로 분류 트리
- `attribute X, value T @제약`으로 속성과 제약
- `@card`, `@range`, `@values`, `@regex` 네 가지 제약
- `isa`, `has`로 인스턴스 입력
- `match` + `fetch`로 분류 깊이에 따른 쿼리

**이론적 자리**
- Subtyping과 Liskov 치환 원칙
- 분류학 vs 분류 이론
- Rosch의 기본 수준 범주 — *질문이 분류 깊이를 결정한다*
- Closed-World Assumption

다음 장 — **관계와 역할**. 분류만으로는 풀리지 않는 자리. 결혼·부모·자녀 같은 *여러 개체가 한 사건에 묶이는 모양*과, *같은 사람이 여러 자리에 있는 모양*이 들어간다.

분류가 *명사의 도구*였다면, 관계는 *동사의 도구*다.
