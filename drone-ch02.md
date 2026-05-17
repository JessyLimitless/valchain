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
commit;
```

**짚어둘 자리**:

- 부모는 1~2명 (한 사건에 친부모 2명 또는 양부모 1명 등)
- 자녀는 정확히 1명 *per 사건* (한 자녀가 *친부모 + 양부모*에게 묶인다면 두 개의 별도 parenthood 매듭)
- `relationship_type`이 `@values`로 강제 — *biological / adoptive / step / guardian* 중 하나

---

## 2.4 데이터 입력 — 4세대 가족

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

## 2.5 같은 사람의 여러 자리 — `plays`의 진짜 가치

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

fetch $spouse_name; $parent_name; $child_name;
```

답:

```json
{ "spouse_name": "박지영", "parent_name": "김덕수", "child_name": "김지훈" }
{ "spouse_name": "박지영", "parent_name": "김덕수", "child_name": "김지수" }
{ "spouse_name": "박지영", "parent_name": "이정희", "child_name": "김지훈" }
{ "spouse_name": "박지영", "parent_name": "이정희", "child_name": "김지수" }
```

네 행이 나오는 이유 — *2명의 부모 × 2명의 자녀*의 카르테시안 곱. 한 사람이 *여러 관계의 여러 자리*에 있는 모양이 한 쿼리에 모두 잡힘.

---

## 2.6 잘못된 데이터 거부 — @card의 강제

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

이건 *카디널리티*가 아니라 *논리적 일관성* 문제. TypeDB 스키마만으로는 *값 사이의 관계*를 강제할 수 없음. **이 자리에서 책의 후반에 *검증 함수* 또는 *Constraint* 패턴이 들어옴**. 도메인 사고가 *스키마로 충분히 표현되는 자리*와 *함수가 필요한 자리*가 갈라지는 첫 자리.

---

## 2.7 ◇ 이론 절 — Davidson의 사건 의미론

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

RDF는 *주어-술어-목적어*의 triple 세 자리만 가진다. 그래서 *사건의 추가 자리*를 담으려면:

```
event_42 type Marriage
event_42 spouse John
event_42 spouse Mary
event_42 location Paris
event_42 time "1990-06-15"
```

*이벤트 노드를 따로 만들고 그 노드에 여러 triple을 다는* 패턴 — 이것이 *reification*. 작동은 하나, *왜 사건을 노드로 만들어야 하는가*가 데이터 모델에서 정당화되지 않는다. RDF의 *세 자리 제약*을 우회하기 위한 패턴일 뿐.

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

---

## 2.8 정리 — 매듭과 자리의 도구

이 장에서 손에 들어온 것:

**Syntax 어휘**
- `relation`, `relates`로 N항 관계
- `plays`로 역할 자리 강제
- N항 인스턴스 입력 `(role1: $x, role2: $y, ...)`
- 같은 자리에 여러 개체 (`spouse: $a, spouse: $b`)
- `@card`로 카디널리티 강제

**이론적 자리**
- Davidson의 *사건 의미론* — 관계를 사건으로 보는 형식
- TypeDB의 `relation`이 사건의 일급 표현
- RDF의 *reification* 문제와 TypeDB의 자연스러움
- 역할(role) vs 속성(attribute)의 결정 기준

다음 장 — **재귀 함수와 그래프 탐색**. 데이터에 적힌 *직접 관계*에서 *N단계 그래프*를 자동으로 펼치는 도구. *Datalog 전통*과 *fixpoint 의미론*이 깔린 자리.
