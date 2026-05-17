# 예제 2. 가족·친족 관계 모델 — N항 관계와 역할

## 들어가며

분류가 *명사의 도구*였다면, 관계는 *동사의 도구*다. 그리고 — *같은 사람이 한 사건에서는 부모, 다른 사건에서는 자녀, 또 다른 사건에서는 배우자*인 모양은 *역할(role)*이라는 별도의 차원을 필요로 한다.

이 장은 N항 관계와 역할이라는 — *RDF/OWL이 우회로 다루던 자리*가 TypeDB에서는 *일급 시민*이 되는 모양을 보인다. 그리고 이 모양의 *형식 의미론* — Davidson의 *사건 의미론* — 을 짚는다.

---

## 2.1 왜 가족 관계인가

세 가지 이유로 두 번째 예제를 가족·친족 관계에 둔다.

1. **N항 관계의 정수**. 결혼은 *2명+공증인+장소+시점* — 4항 이상. 형제는 *N명*. 부모-자녀는 *부모 1~2명 + 자녀 1명 + 양육 형태*. 단순한 *이진 엣지*로는 표현이 어그러진다.
2. **역할의 정수**. 같은 사람이 *부모이자 자녀이자 배우자*. 사람의 본질이 바뀐 게 아니라 *관계 안의 자리*가 바뀐다 — 이 통찰이 *역할(role)*이라는 도구의 자리.
3. **친숙도**. 모든 독자가 가족 관계를 *직관적으로* 안다. 코드의 모양만 보면 됨.

---

## 2.2 결혼 관계 — N항의 첫 번째 모양

```typeql
transaction schema family
```

```typeql
define
  entity person,
    owns name @card(1..1),
    owns birth_date @card(0..1);
  
  # === 결혼 관계 — 두 배우자 + 자녀들 + 시점 ===
  relation marriage,
    relates spouse @card(2..2),
    relates offspring @card(0..),
    owns start_date @card(1..1),
    owns end_date @card(0..1),
    owns marriage_location @card(0..1);
  
  person plays marriage:spouse;
  person plays marriage:offspring;
  
  attribute name, value string;
  attribute birth_date, value date;
  attribute start_date, value date;
  attribute end_date, value date;
  attribute marriage_location, value string;
commit;
```

**짚어둘 세 자리**:

- `relates spouse @card(2..2)` — 결혼에는 *정확히 두 명의 배우자*. (일부다처제·일처다부제는 도메인 규칙으로 거부됨. 그게 도메인의 약속이라면 — *데이터 모델*에 박힌다.)
- `relates offspring @card(0..)` — 자녀는 *영(0) 이상*. 결혼 직후라면 0, 다섯 자녀라면 5.
- *같은 자리에 두 사람* — 결혼의 두 배우자 모두 `spouse` role을 가짐. 이 모양이 *N항 관계의 다중성*.

---

## 2.3 부모 관계 — 단순 N항

```typeql
define
  # === 부모 관계 — 부모 1~2명 + 자녀 1명 + 관계 종류 ===
  relation parenthood,
    relates parent @card(1..2),
    relates child @card(1..1),
    owns relationship_type @card(1..1);
  
  attribute relationship_type, value string 
    @values("biological", "adoptive", "step", "guardian");
  
  person plays parenthood:parent;
  person plays parenthood:child;
```

**짚어둘 자리**:

- 부모는 1~2명 (한 사건에 친부모 2명 또는 양부모 1명 등)
- 자녀는 정확히 1명 *per 사건* (한 자녀가 *친부모 + 양부모*에게 묶인다면 두 개의 별도 parenthood 매듭)
- `relationship_type`이 `@values`로 강제 — *biological / adoptive / step / guardian* 중 하나

---

## 2.4 ◇ 설계 결정 — N항 vs 이진 관계

본격 데이터 입력 전에 — *왜 결혼을 4자리 N항으로 짰는가*를 깊이 토론.

### 결정 1 — 결혼을 N항으로 vs 이진 관계로

**대안 1: 이진 관계 두 개로**
```typeql
relation married_to,
  relates husband @card(1..1),
  relates wife @card(1..1);
relation has_child,
  relates parent @card(1..1),
  relates child @card(1..1);
```

- 단점: *한 결혼의 양 면*(husband·wife)이 다른 관계로 흩어짐. *동성 결혼* 표현 어려움
- 단점: *자녀 관계*가 결혼과 분리. *이 부부의 자녀들*을 묻는 쿼리가 복잡

**대안 2: 5자리 N항**
```typeql
relation marriage,
  relates spouse_a @card(1..1),
  relates spouse_b @card(1..1),
  relates child_a @card(0..1),
  relates child_b @card(0..1),
  relates child_c @card(0..1);
```

- 단점: *최대 자녀 수*가 스키마에 고정. 4번째 자녀가 생기면 *스키마 변경*
- 단점: 자녀 자리가 *순서가 있는 듯한* 인상 (실제로는 의미 없음)

**채택: 가변 N항 (spouse 2명, offspring 0+명)**
- 장점: *결혼의 본질적 모양*에 정확히 일치
- 장점: 자녀 수가 *가변* — 스키마 안정성
- 장점: spouse가 *대칭 자리* — 동성 결혼·일반 결혼 통일적 표현

### 결정 2 — 자녀 자리를 *결혼*에 vs *부모-자녀 별도 관계*로

이 책은 *둘 다* 둔다. 결혼에 `offspring` 자리 + 별도 `parenthood` 관계.

**이유 — 두 자리는 다른 정보를 담음**:

- 결혼의 `offspring`: *이 결혼의 산물*. 결혼 사건 안의 *상황적 정보*
- `parenthood`: *친자 관계의 본질*. 생물학적·양육적 *법적 관계*

**예시로 차이 보기**:
- 친부모가 이혼한 후 *입양된 자녀*의 부모: parenthood (양부모)
- 입양 *이전* 친부모의 결혼: marriage (offspring 자리에 동일 자녀)
- 두 사실이 *다른 매듭에 박혀* 양립 가능

**대안: 결혼에서 offspring 빼기**
- 단점: *이 부부의 자녀들*을 알려면 부모-자녀 관계만 봐야
- 단점: *결혼의 사건 정보*(혼인 중 자녀)가 데이터 모델에 없음

**채택: 둘 다 두기**
- 약간의 중복은 있지만 — *두 관점*을 모두 표현
- 분석가가 *어느 관점*을 묻는지에 따라 다른 쿼리

### 결정 3 — relationship_type을 attribute로 vs 분류 가지로

**대안: 분류 가지로**
```typeql
relation biological_parenthood sub parenthood;
relation adoptive_parenthood sub parenthood;
relation step_parenthood sub parenthood;
relation guardian_parenthood sub parenthood;
```

- 장점: 가지별 *고유 속성* 추가 가능 (adoptive에 *입양 일자*, step에 *결혼 일자* 등)
- 단점: 4개의 별도 관계. 쿼리가 복잡

**채택: attribute @values**
- 장점: 단순. 현재 도메인 요구에 충분
- 미래 확장: 가지별 고유 속성이 필요해지면 *분류 가지로 진화 가능*

### 결정 4 — 시간 표현의 모양

결혼에 `start_date`(필수) + `end_date`(옵션). 이혼·사망 시 `end_date` 박음.

**대안: 별도 lifecycle event 관계**
```typeql
relation marriage_event,
  relates the_marriage,
  relates event_type, owns event_date;
```

- 장점: *결혼·이혼·사별*의 *모든 사건*을 통일 표현
- 단점: 단순 *현재 결혼 여부*를 묻는 쿼리가 복잡

**채택: start_date 필수, end_date 옵션**
- 장점: *현재 결혼*은 `not { $m has end_date $e; }` 한 줄
- 장점: 데이터 모델이 간단

---

## 2.5 데이터 입력 — 4세대 가족

```typeql
transaction write family
```

```typeql
insert
  # === 조부모 세대 ===
  $grandpa isa person, has name "김덕수", has birth_date 1940-03-15;
  $grandma isa person, has name "이정희", has birth_date 1942-07-20;
  
  # === 부모 세대 ===
  $father isa person, has name "김민수", has birth_date 1965-04-10;
  $mother isa person, has name "박지영", has birth_date 1968-11-05;
  $uncle isa person, has name "김민철", has birth_date 1970-09-22;
  
  # === 자녀 세대 ===
  $son isa person, has name "김지훈", has birth_date 1995-02-14;
  $daughter isa person, has name "김지수", has birth_date 1998-12-25;
  
  # === 결혼 1: 조부모 ===
  $m1 (spouse: $grandpa, spouse: $grandma, 
       offspring: $father, offspring: $uncle) 
    isa marriage,
    has start_date 1964-04-15,
    has marriage_location "서울";
  
  # === 결혼 2: 부모 ===
  $m2 (spouse: $father, spouse: $mother, 
       offspring: $son, offspring: $daughter) 
    isa marriage,
    has start_date 1993-10-20,
    has marriage_location "부산";
  
  # === 부모-자녀 관계 (생물학적) ===
  $p1 (parent: $grandpa, parent: $grandma, child: $father) 
    isa parenthood, has relationship_type "biological";
  $p2 (parent: $grandpa, parent: $grandma, child: $uncle) 
    isa parenthood, has relationship_type "biological";
  $p3 (parent: $father, parent: $mother, child: $son) 
    isa parenthood, has relationship_type "biological";
  $p4 (parent: $father, parent: $mother, child: $daughter) 
    isa parenthood, has relationship_type "biological";
commit;
```

7명, 2개 결혼, 4개 부모-자녀 매듭.

---

## 2.6 같은 사람의 여러 자리 — `plays`의 진짜 가치

`김민수(아버지)` 한 사람을 보자. 데이터 안에서 이 사람은:

- *결혼 m2의 spouse* — 배우자 자리
- *결혼 m1의 offspring* — 조부모의 자녀 자리
- *parenthood p1의 child* — 부모-자녀의 자녀 자리
- *parenthood p3의 parent* — 다른 부모-자녀의 부모 자리

같은 한 사람이 *네 가지 자리*에 동시에 있다.

### 통합 쿼리

한 사람의 *모든 자리*를 한 번에 조회:

```typeql
transaction read family
```

```typeql
match
  $father isa person, has name "김민수";
  
  # spouse 자리
  $m (spouse: $father, spouse: $other_spouse) isa marriage;
  $other_spouse has name $spouse_name;
  
  # child 자리 (parenthood에서)
  $p_as_child (parent: $parent, child: $father) isa parenthood;
  $parent has name $parent_name;
  
  # parent 자리 (parenthood에서)
  $p_as_parent (parent: $father, child: $kid) isa parenthood;
  $kid has name $child_name;

fetch
  $spouse_name; $parent_name; $child_name;
```

답:

```json
{ "spouse_name": "박지영", "parent_name": "김덕수", "child_name": "김지훈" }
{ "spouse_name": "박지영", "parent_name": "김덕수", "child_name": "김지수" }
{ "spouse_name": "박지영", "parent_name": "이정희", "child_name": "김지훈" }
{ "spouse_name": "박지영", "parent_name": "이정희", "child_name": "김지수" }
```

네 행이 나오는 이유 — *2명의 부모 × 2명의 자녀*의 카르테시안 곱. 한 사람이 *여러 관계의 여러 자리*에 있는 모양이 한 쿼리에 모두 잡힘.

### 다양한 자리별 조회

**김민수가 spouse인 결혼만**:
```typeql
match
  $father isa person, has name "김민수";
  $m (spouse: $father) isa marriage;
  $m has start_date $sd;
fetch $sd;
```
답: 1993-10-20.

**김민수가 child인 부모-자녀 관계만**:
```typeql
match
  $father isa person, has name "김민수";
  $p (parent: $par, child: $father) isa parenthood;
  $par has name $pn;
fetch $pn;
```
답: 김덕수, 이정희.

**김민수의 친자 자녀들**:
```typeql
match
  $father isa person, has name "김민수";
  $p (parent: $father, child: $kid) isa parenthood;
  $p has relationship_type "biological";
  $kid has name $kn;
fetch $kn;
```
답: 김지훈, 김지수.

각 쿼리가 *같은 인물*을 *다른 role*로 조회. 데이터 모델의 *역할 자리*가 명시적이라 — 쿼리도 명시적.

---

## 2.7 잘못된 데이터 거부 — @card의 강제

스키마의 `@card`가 *도메인 규칙*을 어떻게 강제하는가.

### 시연 1 — 세 명이 한 결혼에

```typeql
insert
  $bad_marriage (spouse: $a, spouse: $b, spouse: $c) isa marriage,
    has start_date 2020-01-01;
```

오류:
```
Error: Cardinality violation. Relation 'marriage' has role 'spouse' 
with cardinality 3, exceeds maximum 2.
```

### 시연 2 — 친부모가 세 명인 데이터

```typeql
insert
  $bad_parenthood (parent: $a, parent: $b, parent: $c, child: $kid) 
    isa parenthood, has relationship_type "biological";
```

오류:
```
Error: Cardinality violation. Relation 'parenthood' has role 'parent' 
with cardinality 3, exceeds maximum 2.
```

### 시연 3 — 종료 시점이 시작보다 빠른 결혼

```typeql
insert
  $bad_marriage (spouse: $a, spouse: $b) isa marriage,
    has start_date 2020-01-01,
    has end_date 2015-06-15;
```

이건 *카디널리티*가 아니라 *논리적 일관성* 문제. TypeDB 스키마만으로는 *값 사이의 관계*를 강제할 수 없음.

**해결책 — 검증 함수**:
```typeql
fun marriages_with_invalid_dates() -> { marriage }:
  match
    $m isa marriage, has start_date $s, has end_date $e;
    $e <= $s;
  return { $m };
```

이 함수를 *주기적으로 호출*해 *논리적 일관성 위반*을 감지. 함수 자체가 *데이터 품질 모니터링 도구*가 됨.

도메인 사고가 *스키마로 충분히 표현되는 자리*와 *함수가 필요한 자리*가 갈라지는 첫 자리.

---

## 2.8 시간을 포함한 관계 — 이혼·재혼 시나리오

가족 관계는 *시간에 따라 변한다*. 결혼·이혼·재혼·사별 같은 사건들.

### 이혼 처리

```typeql
match
  $father isa person, has name "김민수";
  $mother isa person, has name "박지영";
  $m (spouse: $father, spouse: $mother) isa marriage;
insert
  $m has end_date 2010-08-15;
```

기존 결혼에 *end_date 박음*. 결혼 매듭은 *보존*. *과거 사실*로 기록됨.

### 재혼

```typeql
insert
  $new_partner isa person, has name "이수진", has birth_date 1972-05-30;
  
  # 김민수의 두 번째 결혼
  $m3 (spouse: $father, spouse: $new_partner) isa marriage,
    has start_date 2012-11-20,
    has marriage_location "제주";
```

같은 person 인스턴스(`$father`)가 *두 결혼*에 spouse 자리로 등장. 데이터는 *과거*와 *현재*를 모두 보존.

### "현재 결혼 중인 사람" 쿼리

```typeql
match
  $p isa person, has name $n;
  $m (spouse: $p) isa marriage;
  not { $m has end_date $e; };
fetch $n;
```

*end_date가 없는* 결혼의 spouse만. 현재 결혼 중인 모든 사람.

### "이혼했지만 자녀가 있는 사람" 쿼리

```typeql
match
  $p isa person, has name $n;
  $m (spouse: $p, offspring: $kid) isa marriage;
  $m has end_date $e;  # 이혼한 결혼
fetch $n;
```

### 가족 관계의 시간축

```
1964-04-15  김덕수 + 이정희 결혼  (m1)
1965-04-10  김민수 출생  (p1)
1970-09-22  김민철 출생  (p2)
1993-10-20  김민수 + 박지영 결혼  (m2)
1995-02-14  김지훈 출생  (p3)
1998-12-25  김지수 출생  (p4)
2010-08-15  김민수 + 박지영 이혼  (m2.end_date)
2012-11-20  김민수 + 이수진 결혼  (m3)
```

이 시간축 전체가 *데이터에 있다*. 어느 시점에 어느 관계가 활성이었는지 — *모든 시점*에 대해 답 가능.

---

## 2.9 RDF/OWL과의 비교 — 같은 도메인, 다른 모양

같은 *결혼 관계*를 RDF로 표현하면 어떻게 보이는가.

### RDF의 reification 패턴

```turtle
# RDF/Turtle 표기
:m_001 a :Marriage .
:m_001 :spouse :김민수 .
:m_001 :spouse :박지영 .
:m_001 :offspring :김지훈 .
:m_001 :offspring :김지수 .
:m_001 :startDate "1993-10-20"^^xsd:date .
:m_001 :location "부산" .
```

**짚어둘 자리**:
- 결혼을 *별도 entity(m_001)*로 만들어야 — *reification*
- 각 triple이 *별개의 사실*
- *결혼의 두 spouse 자리가 동일하다*는 사실은 *암묵적* (role 이름이 같으니)
- 데이터 모델 차원에서 *spouse가 두 명*이라는 제약 강제 어려움

### OWL로 확장 (제약 추가)

```turtle
# OWL — 제약 추가
:Marriage rdfs:subClassOf owl:Thing .
:hasSpouse a owl:ObjectProperty ;
  rdfs:domain :Marriage ;
  rdfs:range :Person .

# 정확히 2명의 spouse 강제
:Marriage rdfs:subClassOf [
  a owl:Restriction ;
  owl:onProperty :hasSpouse ;
  owl:qualifiedCardinality "2"^^xsd:nonNegativeInteger
] .
```

- 가능하지만 — *훨씬 장황*
- *qualified cardinality*가 OWL 2의 확장 (OWL 1에서는 어려움)
- *추론기*가 카디널리티 위반을 감지하기까지 *별도 작업*

### TypeDB의 같은 자리

```typeql
relation marriage,
  relates spouse @card(2..2),
  relates offspring @card(0..),
  owns start_date @card(1..1);
```

- *한 줄로 같은 의미*
- *카디널리티*가 데이터 모델에 박힘 — INSERT 시점에 자동 강제
- *reification 우회 없음* — 결혼이 *직접 N항 관계*

### 비교 정리

| 자리 | RDF/OWL | TypeDB |
|---|---|---|
| N항 관계 | reification 우회 필요 | 일급 |
| 카디널리티 | OWL 2 qualified restriction | `@card` |
| 역할 (role) | property 이름 (암묵) | `relates` (명시) |
| 추론 | 추론기 (외부) | Function (내부) |
| 데이터 모델 강제 | 약함 (validation은 SHACL 추가) | 강함 (INSERT 시점) |
| 학습 곡선 | 가파름 | 완만 |
| 표준 호환성 | W3C 표준 | 단일 벤더 |

각자의 강점이 있다. TypeDB는 *실용적 데이터베이스 운영*에 강점. RDF/OWL은 *표준 호환성·시맨틱 웹 통합*에 강점.

---

## 2.10 잘못된 스키마 vs 좋은 스키마

가족 도메인의 *세 가지 잘못된 방식*과 *현재 책의 좋은 방식*.

### 잘못된 스키마 1 — 모든 관계를 이진으로

```typeql
define
  entity person, owns name;
  
  relation married_to,
    relates first_spouse, relates second_spouse;
  
  relation child_of,
    relates the_child, relates the_parent;
  
  relation sibling_of,
    relates first_sib, relates second_sib;
```

**왜 나쁜가**:
- *결혼의 자녀들*을 표현하려면 *married_to + child_of*의 두 관계를 *조합 추론*해야
- *first_spouse vs second_spouse*가 대칭성을 깨뜨림 (남편이 first?)
- *결혼 시점*을 어디에 박을지 어색

### 잘못된 스키마 2 — 모든 관계를 한 관계로

```typeql
define
  entity person, owns name;
  
  relation family_relation,
    relates from, relates to,
    owns relation_kind;  # "spouse", "parent", "sibling"
  
  attribute relation_kind, value string;
```

**왜 나쁜가**:
- *카디널리티 강제 불가* — 결혼의 *두 명* 강제 못함
- *각 관계 종류별 고유 속성*(결혼 일자 vs 양육 형태) 어색
- 모든 쿼리가 `relation_kind == "spouse"` 같은 *런타임 필터*에 의존

### 잘못된 스키마 3 — 자녀를 결혼의 *속성*으로

```typeql
define
  relation marriage,
    relates spouse @card(2..2),
    owns child_names;  # 문자열로 자녀 이름들
  
  attribute child_names, value string @card(0..);
```

**왜 나쁜가**:
- *자녀가 person entity가 아니라 문자열*
- *자녀의 다른 속성*(birth_date, 다른 결혼) 표현 불가
- *동명이인* 구분 불가

### 좋은 스키마 — 현재 책의 스키마

2.2~2.3의 스키마는:
- *N항 관계*로 결혼·부모-자녀
- *역할(role)*이 명시적
- *@card*로 카디널리티 강제
- *@values*로 관계 종류 강제
- *시점*이 attribute로 박힘

**결과**: 도메인의 *모든 의미*가 명시적. 카르테시안 곱·시간축·자리별 조회가 모두 자연스럽게.

---

## 2.11 ◇ 이론 절 — Davidson의 사건 의미론

### 동사의 표현 문제

언어학·철학에서 오래된 질문이 있었다. *동사를 어떻게 형식화할 것인가*.

전통 술어논리에서는 동사를 *함수 술어*로 다룬다.
- *John married Mary* → `Marry(John, Mary)`
- 두 자리 술어.

문제: *John married Mary in Paris on June 15, 1990* — 추가 정보를 어떻게 담는가? 매번 함수의 자리 수를 늘려야 하나? *Marry(John, Mary, Paris, 1990-06-15)*? 그러면 *John married Mary in Paris*(시점 없음)는 *Marry(John, Mary, Paris, ?)*? 어색하다.

### Davidson의 해결책 (1967)

도널드 데이비드슨(Donald Davidson)의 논문 *The Logical Form of Action Sentences*가 이 자리를 풀었다.

**핵심 아이디어**: *사건(event)을 일급 개체로 도입*하라.

- *John married Mary in Paris on June 15* → 
  - `∃e [Marry(e) ∧ Spouse(e, John) ∧ Spouse(e, Mary) ∧ Location(e, Paris) ∧ Time(e, June 15)]`

*결혼이라는 사건 e가 존재하고, John이 그 사건의 spouse 자리에, Mary가 다른 spouse 자리에, Paris가 location 자리에, June 15가 time 자리에 있다.*

이 형식의 우아함:
- 추가 정보(증인·날씨·관습 등)는 *같은 사건 e에 또 다른 자리*를 더하면 됨. 함수 시그니처를 바꾸지 않아도 됨.
- *spouse라는 자리(role)*가 *John이나 Mary라는 개체*보다 *근원적*.

### TypeDB와의 정확한 대응

TypeDB의 `relation`이 정확히 Davidson의 *사건*에 대응한다.

| Davidson 사건 의미론 | TypeDB |
|---|---|
| 사건(event) e | relation 인스턴스 |
| 자리(role) | relates X |
| 자리의 참여자(participant) | plays X |
| 사건의 속성 | relation owns attribute |

`$m (spouse: $father, spouse: $mother) isa marriage, has start_date 1993-10-20` 한 줄이 — 데이비드슨이 1967년에 도입한 *사건 형식*을 *데이터베이스에 박는* 자리다.

### Reification — RDF의 우회

RDF는 *주어-술어-목적어*의 triple 세 자리만 가진다. 그래서 *사건의 추가 자리*를 담으려면 *이벤트 노드를 따로 만들고 그 노드에 여러 triple을 다는* 패턴 — 이것이 *reification*. 작동은 하나, *왜 사건을 노드로 만들어야 하는가*가 데이터 모델에서 정당화되지 않는다. RDF의 *세 자리 제약*을 우회하기 위한 패턴일 뿐.

TypeDB의 `relation`은 *처음부터 N항*. Reification이 필요 없다. *데이비드슨의 사건이 일급으로 존재*하는 데이터 모델.

### 역할(role) vs 속성(attribute)의 구분

또 한 가지 이론적 자리 — *언제 역할이고 언제 속성인가*.

- **역할(role)**: 관계의 *참여자가 가지는 자리*. 그 자리에 *다른 개체*가 들어옴.
- **속성(attribute)**: 관계 자체의 *값*. 그 자리에 *원자값(문자·숫자·날짜)*이 들어옴.

예: *marriage*에서
- `spouse` — 역할 (사람 개체가 들어감)
- `offspring` — 역할 (사람 개체가 들어감)
- `start_date` — 속성 (날짜 값이 들어감)
- `marriage_location` — 속성 (문자열 값)

만약 *결혼 장소를 entity로 둬야 한다면* (다른 결혼이 같은 장소에서 일어났는지를 추적하고 싶다면), `location`을 *역할*로 바꾸고 `place` entity를 도입. 도메인의 *질문*이 결정한다.

### 의미론적 깊이 — Modal Logic의 자리

이 책의 시간 처리(start/end date)는 *비교적 단순한 시간 모델*. 더 형식적인 자리에는 *시제 논리(temporal logic)*가 있다.

**LTL (Linear Temporal Logic)**:
- *언젠가 결혼한 적이 있다*: `◇married`
- *항상 같은 사람과 결혼*: `□(married → same_spouse)`
- *결혼했다면 그 이후 어느 시점*: `married → ○ever_married`

**CTL (Computation Tree Logic)**:
- *가능한 모든 미래*에 대한 명제
- *어떤 미래에서는 이혼하고 어떤 미래에서는 안 함*

**Epistemic Logic** (인식 논리):
- *누가 무엇을 아는가*
- *부모는 자녀의 출생 정보를 안다*

이런 논리들이 *현재 TypeDB에 직접 지원되지 않지만* — 함수와 시간 attribute의 조합으로 *부분적 표현* 가능.

### 사건 의미론의 다른 자리

Davidson 1967이 시작이고 — 그 후 여러 확장:

- **Parsons (1990)** — *Events in the Semantics of English*. Davidson의 형식을 영어 문법 전체로 확장.
- **Kim (1993)** — *Supervenience and Mind*. 사건의 *동일성 기준* 토론.
- **Davidson 본인 (2003년 사망)** — 1980·90년대에 사건 의미론을 *언어철학·심리철학*까지 확장.

이 학술 전통이 *2020년대 데이터베이스 설계*에 영향을 주는 자리가 — 흥미롭다. *철학의 어휘*가 *실용적 도구*가 된 자리.

---

## 2.12 정리 — 매듭과 자리의 도구

이 장에서 손에 들어온 것:

**Syntax 어휘**
- `relation`, `relates`로 N항 관계
- `plays`로 역할 자리 강제
- N항 인스턴스 입력 `(role1: $x, role2: $y, ...)`
- 같은 자리에 여러 개체 (`spouse: $a, spouse: $b`)
- `@card`로 카디널리티 강제

**설계 결정의 트레이드오프** (이 장의 결정적 추가)
- N항 vs 이진 관계의 선택 기준
- 결혼의 offspring vs 별도 parenthood 관계 (둘 다 두기)
- relationship_type을 attribute로 vs 분류 가지로
- 시간 표현의 모양 (start/end vs lifecycle event)

**시간 관계의 시연**
- 이혼·재혼 시나리오
- "현재 결혼 중", "이혼했지만 자녀 있는" 같은 쿼리

**RDF/OWL과의 비교 코드**
- 같은 도메인을 두 도구로 표현 — 표현력·강제·학습곡선 차이

**잘못된 스키마와의 비교**
- 모든 이진 관계 (조합 추론 필요)
- 한 관계 + relation_kind 속성 (강제 손실)
- 자녀를 결혼의 속성으로 (entity 손실)

**이론적 자리**
- Davidson의 *사건 의미론* — 관계를 사건으로 보는 형식
- TypeDB의 `relation`이 사건의 일급 표현
- RDF의 *reification* 문제와 TypeDB의 자연스러움
- 역할(role) vs 속성(attribute)의 결정 기준
- Modal Logic의 자리 (LTL, CTL, Epistemic)

다음 장 — **재귀 함수와 그래프 탐색**. 데이터에 적힌 *직접 관계*에서 *N단계 그래프*를 자동으로 펼치는 도구. *Datalog 전통*과 *fixpoint 의미론*이 깔린 자리.
