# 예제 3. 조직도와 의존성 그래프 — 재귀 함수

## 들어가며

세 번째 도구 — *함수*. TypeDB 3.0의 Function이 도입되면서, 데이터베이스가 *저장소*에서 *추론 엔진*으로 변환된다. 데이터에 적힌 *직접 관계*에서 *N단계 그래프*를 자동으로 펼치는 작업 — *재귀*가 그 핵심이다.

이 장의 이론적 자리는 *Datalog 전통*. 1970~80년대 데이터베이스 이론의 가장 중요한 가지 중 하나가 — 이 자리에서 TypeDB 함수의 형식 의미론으로 살아 있다.

---

## 3.1 왜 조직도·의존성인가

두 도메인을 함께 다루는 이유:

1. **조직도**: 직관적, 누구나 이해. *한 사람의 직속 부하, 부하의 부하, ...*가 *재귀의 자연스러운 모양*.
2. **소프트웨어 의존성**: 직장인이 일상적으로 다루는 자리. npm·pip·maven이 의존성 그래프를 펼치는 작업이 *정확히 같은 패턴*.

두 도메인이 *같은 함수 패턴으로 풀린다*는 것이 — 이 장의 메타 메시지다. 도메인이 달라도 *그래프 + 재귀*가 같으면 함수는 같다.

---

## 3.2 스키마

```typeql
transaction schema org_and_deps
```

```typeql
define
  # === 조직도 ===
  entity person, owns name @card(1..1), owns title @card(0..1);
  
  relation reports_to,
    relates subordinate @card(1..1),
    relates supervisor @card(1..1);
  
  person plays reports_to:subordinate;
  person plays reports_to:supervisor;
  
  # === 소프트웨어 의존성 ===
  entity package, 
    owns pkg_name @card(1..1), 
    owns version @card(0..1);
  
  relation depends_on,
    relates dependent @card(1..1),
    relates dependency @card(1..1);
  
  package plays depends_on:dependent;
  package plays depends_on:dependency;
  
  attribute name, value string;
  attribute title, value string;
  attribute pkg_name, value string;
  attribute version, value string;
commit;
```

두 도메인이 *같은 모양*의 N항 관계 — *주체-대상* 두 자리 + 카디널리티 1..1. 차이는 *어떤 개체가 어떤 자리에 들어가는가*뿐.

---

## 3.3 데이터 입력

```typeql
transaction write org_and_deps
```

```typeql
insert
  # === 조직 (5명) ===
  $ceo isa person, has name "Alice", has title "CEO";
  $vp isa person, has name "Bob", has title "VP Engineering";
  $mgr isa person, has name "Carol", has title "Engineering Manager";
  $eng1 isa person, has name "David", has title "Senior Engineer";
  $eng2 isa person, has name "Eve", has title "Engineer";
  
  (subordinate: $vp, supervisor: $ceo) isa reports_to;
  (subordinate: $mgr, supervisor: $vp) isa reports_to;
  (subordinate: $eng1, supervisor: $mgr) isa reports_to;
  (subordinate: $eng2, supervisor: $mgr) isa reports_to;
  
  # === 소프트웨어 의존성 (5개 패키지) ===
  $p_app isa package, has pkg_name "my-app", has version "1.0.0";
  $p_lib isa package, has pkg_name "ui-library", has version "2.3.1";
  $p_react isa package, has pkg_name "react", has version "18.2.0";
  $p_lodash isa package, has pkg_name "lodash", has version "4.17.21";
  $p_util isa package, has pkg_name "utility-belt", has version "0.5.2";
  
  (dependent: $p_app, dependency: $p_lib) isa depends_on;
  (dependent: $p_app, dependency: $p_lodash) isa depends_on;
  (dependent: $p_lib, dependency: $p_react) isa depends_on;
  (dependent: $p_lib, dependency: $p_util) isa depends_on;
  (dependent: $p_util, dependency: $p_lodash) isa depends_on;
commit;
```

조직: CEO → VP → Manager → 두 Engineer. 4단 위계.
의존성: my-app → ui-library, lodash | ui-library → react, utility-belt | utility-belt → lodash.

---

## 3.4 단일 반환 Function — 첫 함수

가장 단순한 모양 — *한 사람의 직속 상사*.

```typeql
transaction schema org_and_deps
```

```typeql
define
  fun manager_of($p: person) -> { person }:
    match
      (subordinate: $p, supervisor: $m) isa reports_to;
    return { $m };
commit;
```

호출:

```typeql
transaction read org_and_deps
```

```typeql
match
  $carol isa person, has name "Carol";
  let $m in manager_of($carol);
  $m has name $manager_name;
fetch $manager_name;
```

답: `{ "manager_name": "Bob" }`.

**짚어둘 자리**:
- 함수의 *반환 타입* `{ person }`은 *person의 집합*. 한 사람만 답해도 *집합 형식*.
- `let $m in manager_of($carol)` — 함수의 *반환 집합에서 한 원소를 변수에 묶음*.
- *match*로도 풀 수 있는 작업이지만 — 함수로 묶으면 *재사용 가능한 도메인 어휘*가 됨.

---

## 3.5 집합 반환 Function

*한 사람의 모든 직속 부하* — 여러 결과가 자연스러운 자리.

```typeql
define
  fun direct_reports_of($manager: person) -> { person }:
    match
      (supervisor: $manager, subordinate: $sub) isa reports_to;
    return { $sub };

  fun direct_dependencies_of($p: package) -> { package }:
    match
      (dependent: $p, dependency: $d) isa depends_on;
    return { $d };
commit;
```

호출:

```typeql
match
  $carol isa person, has name "Carol";
  let $sub in direct_reports_of($carol);
  $sub has name $sub_name;
fetch $sub_name;
```

답: David, Eve.

같은 패턴이 *다른 도메인*에:

```typeql
match
  $app isa package, has pkg_name "my-app";
  let $dep in direct_dependencies_of($app);
  $dep has pkg_name $dep_name;
fetch $dep_name;
```

답: ui-library, lodash.

---

## 3.6 함수 합성 — query에 들어가는 자리

한 함수의 출력이 다음 query의 입력이 되는 모양. *합성(composition)*이 함수의 진짜 가치.

```typeql
match
  $bob isa person, has name "Bob";
  let $sub in direct_reports_of($bob);
  $sub has title $t;
  $t == "Engineering Manager";
fetch $sub;
```

*Bob의 직속 부하 중 직책이 Engineering Manager인 사람만*. 함수 출력에 *추가 조건*을 자연스럽게 결합.

---

## 3.7 재귀 — 본격 자리

*모든 직간접 부하* — 직속 + 그 직속의 부하 + ... N단계.

```typeql
define
  fun all_subordinates_of($manager: person) -> { person }:
    # 기저 — 직속 부하
    match
      (supervisor: $manager, subordinate: $direct) isa reports_to;
    return { $direct };
    
    or
    
    # 재귀 — 중간 부하의 부하
    match
      (supervisor: $manager, subordinate: $mid) isa reports_to;
      let $deep in all_subordinates_of($mid);
    return { $deep };
commit;
```

호출:

```typeql
match
  $ceo isa person, has name "Alice";
  let $sub in all_subordinates_of($ceo);
  $sub has name $sub_name;
fetch $sub_name;
```

답: Bob, Carol, David, Eve — *전 직원 (CEO 제외)*.

CEO의 직속은 VP뿐인데, 재귀가 VP→Manager→Engineers까지 자동으로 펼침.

같은 패턴이 의존성 그래프에:

```typeql
define
  fun all_dependencies_of($p: package) -> { package }:
    match
      (dependent: $p, dependency: $d) isa depends_on;
    return { $d };
    
    or
    
    match
      (dependent: $p, dependency: $mid) isa depends_on;
      let $deep in all_dependencies_of($mid);
    return { $deep };
commit;
```

`my-app`의 모든 의존성: ui-library, lodash, react, utility-belt — *transitive closure*.

---

## 3.8 Function의 한계와 자리

도구를 미화하지 않는다. 세 가지 한계를 짚어둔다.

### 한계 1 — 무한 재귀

순환 관계가 데이터에 있으면 — 무한 루프 위험. 가령 *Alice가 Bob에게 보고하고, Bob이 Alice에게 보고*하는 데이터가 들어가면, `all_subordinates_of(Alice)`가 끝나지 않을 수 있다.

TypeDB 엔진은 *순환 감지* 메커니즘을 가지고 있어 안전하게 종료하지만 — *도메인 설계*에서 *순환을 만들지 않는 약속*이 더 안전. 조직도는 위계적이고, 의존성은 *DAG(Directed Acyclic Graph)*여야 한다.

### 한계 2 — 성능

깊은 재귀 + 큰 데이터 = 느려질 수 있다. 1만 명 조직에서 *CEO의 모든 부하*를 묻는 쿼리가 *수십만 매듭*을 펼쳐야 한다면 — 응답 시간이 길어진다.

해결 방향:
- 깊이 제한 추가 (depth-bounded recursion)
- 결과 캐싱 (materialization)
- 인덱스 활용 (TypeDB는 자동 인덱스)

### 한계 3 — 디버깅의 어려움

재귀 함수가 *잘못된 답*을 반환할 때 — *어느 경로*가 잘못됐는지 추적이 일반 query보다 까다롭다. 처음에는 *작은 데이터*로 함수를 검증한 뒤 *큰 데이터*로 옮기는 작업 순서를 권한다.

---

## 3.9 ◇ 이론 절 — Datalog 전통과 Fixpoint 의미론

### Datalog의 기원

1970년대 후반, *데이터베이스 + 논리 프로그래밍*을 결합하려는 시도가 있었다. 답은 *Datalog* — Prolog의 부분집합으로, *재귀 쿼리*가 자연스럽고 *결정 가능(decidable)*한 언어.

Datalog의 핵심 표현 단위는 *Horn clause*:

```
ancestor(X, Y) :- parent(X, Y).
ancestor(X, Y) :- parent(X, Z), ancestor(Z, Y).
```

*X가 Y의 조상인 경우: (1) X가 Y의 부모이거나, (2) X가 Z의 부모이고 Z가 Y의 조상.*

이 모양 — *기저 + 재귀 + 분기(or)* — 가 TypeDB의 Function과 *형식적으로 동일*하다.

TypeDB의 함수:
```typeql
fun ancestor_of($x: person) -> { person }:
  match (parent: $x, child: $y) isa parenthood;
  return { $y };
  or
  match
    (parent: $x, child: $z) isa parenthood;
    let $y in ancestor_of($z);
  return { $y };
```

Datalog의 *Horn clause 두 개*가 TypeDB의 *재귀 함수 한 개*에 정확히 대응한다.

### Fixpoint 의미론

*재귀 함수가 무엇을 의미하는가*에 대한 형식적 답이 *fixpoint semantics*다.

재귀 함수 `F`를 *데이터의 변환*으로 본다:
- `F^0 = {}` — 빈 집합
- `F^1 = F(F^0)` — 기저 규칙만 적용한 결과
- `F^2 = F(F^1)` — 한 번 더 적용
- ...
- `F^∞ = F(F^∞)` — *더 이상 변하지 않는 자리*

이 *더 이상 변하지 않는 자리*가 *fixpoint*(부동점). 그리고 *가장 작은 fixpoint*(least fixpoint)가 — 재귀 함수의 *정답*이다.

조직도 예: `all_subordinates_of(CEO)`의 fixpoint 계산
- `step 1`: CEO의 직속 = {VP}
- `step 2`: + VP의 직속 = {VP, Manager}
- `step 3`: + Manager의 직속 = {VP, Manager, Eng1, Eng2}
- `step 4`: Eng1·Eng2는 직속이 없음. 변하지 않음 → fixpoint 도달.

답: {VP, Manager, Eng1, Eng2}.

TypeDB 엔진은 이 fixpoint를 *효율적으로* 계산하는 알고리즘(semi-naïve evaluation 등)을 내장. 짜는 사람은 *기저와 재귀만 적으면* — 엔진이 fixpoint를 보장한다.

### Stratified Negation의 안전성

*부정(`not { ... }`)*과 *재귀*가 함께 있으면 — 위험한 자리가 있다.

예: *부하가 없는 사람을 찾는 함수*
```
fun has_no_subordinate($p: person) -> { boolean }:
  not { (supervisor: $p) isa reports_to; };
  return { true };
```

이 함수 안에서 *다시 자기 자신*을 재귀로 부르면 — *진리값이 결정되지 않는* 상황이 생길 수 있다 (paradox of self-reference).

해결책: **Stratified negation**. 부정과 재귀를 *층(stratum)*으로 분리. 한 층의 함수는 *낮은 층의 함수만* 부정 절 안에서 호출 가능. TypeDB는 이 조건을 자동 검증.

### Closed-World Assumption과의 관계

3.8에서 짚은 `has_no_subordinate`는 *CWA*를 전제로 한다. 데이터에 *Alice 밑에 부하가 적혀 있지 않다*면 — *Alice는 부하가 없다*고 결론. OWA에서는 *모른다*가 답.

TypeDB의 CWA는 *데이터베이스의 일반적 직관*에 맞춤. *부하가 적혀 있지 않으면 없다*가 — 실용적으로는 옳다.

### Function vs Rule — TypeDB 3.0의 설계 결정

TypeDB 2.x에서는 *Rule*이 추론 도구였다.

```
# TypeDB 2.x (3.0 이후 deprecated)
rule manager_is_supervisor:
  when {
    (subordinate: $s, supervisor: $m) isa reports_to;
  } then {
    $m has title $t;
  };
```

3.0에서 *Rule을 폐기*하고 *Function*으로 통합한 결정의 핵심:

| 특징 | Rule | Function |
|---|---|---|
| 패러다임 | 선언적 (when/then) | 함수형 (input/output) |
| 합성 | 어려움 (규칙 사이의 명시적 호출 없음) | 자연스러움 (`let ... in`) |
| 디버깅 | 어려움 (어느 규칙이 적용됐는지 추적 어려움) | 일반 함수처럼 추적 |
| 형식 의미론 | 복잡함 (모든 규칙의 fixpoint) | 명확함 (호출된 함수의 fixpoint) |

함수의 *합성성(compositionality)*이 — 추론 도구를 *프로그래밍의 일급 시민*으로 만들었다.

---

## 3.10 정리 — 1부 마무리

이 장에서 손에 들어온 것:

**Syntax 어휘**
- `fun X(...) -> { Y }: match ... return { ... };`로 함수 정의
- `let $x in func($arg)`로 함수 호출
- `or`로 분기되는 재귀
- 함수 합성 — 한 함수의 출력에 추가 조건

**이론적 자리**
- *Datalog 전통* — TypeDB 함수가 Horn clause에 대응
- *Fixpoint 의미론* — 재귀 함수의 정답이 least fixpoint
- *Stratified negation*의 안전성
- *Function vs Rule*의 설계 결정

---

## 1부 마무리 — 세 가지 도구가 모인 자리

1부 세 장에서 손에 들어온 도구:

| 장 | 도구 | 이론적 자리 |
|---|---|---|
| 1장 | 분류 (entity 상속) | Subtyping, LSP |
| 2장 | N항 관계와 역할 (relation, plays) | Davidson 사건 의미론 |
| 3장 | 함수와 재귀 (Function) | Datalog, Fixpoint |

이 세 도구가 *동시에* 작동해야 풀리는 도메인이 — 2부의 드론 군집비행이다. 친숙한 예제에서 익힌 도구가 *진짜 실전*에서 어떻게 결합하는가 — 다음 부에서 본다.

> *손이 익은 도구는 — 도메인이 달라져도 같은 모양으로 작동한다.*

다음 장 — **드론 군집비행의 도메인 분석**. *온톨로지 엔지니어링 방법론*과 *Competency Questions*가 들어온다. 본격 도메인 모델링의 첫 자리.
