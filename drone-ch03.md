# 예제 3. 조직도와 의존성 그래프 — 재귀 함수

## 들어가며

세 번째 도구 — *함수*. TypeDB 3.0의 Function이 도입되면서, 데이터베이스가 *저장소*에서 *추론 엔진*으로 변환된다. 데이터에 적힌 *직접 관계*에서 *N단계 그래프*를 자동으로 펼치는 작업 — *재귀*가 그 핵심이다.

이 장의 이론적 자리: *Datalog 전통*. 1970~80년대 데이터베이스 이론의 가장 중요한 가지 중 하나가 — 여기서 TypeDB 함수의 형식 의미론으로 살아 있다.

---

## 3.1 왜 조직도·의존성인가

두 도메인을 함께 다루는 이유:

1. **조직도**: 직관적, 누구나 이해. *한 사람의 직속 부하, 부하의 부하, ...*가 *재귀의 자연스러운 모양*.
2. **소프트웨어 의존성**: 직장인이 일상적으로 다루는 자리. npm·pip·maven이 의존성 그래프를 펼치는 작업이 *정확히 같은 패턴*.

두 도메인이 *같은 함수 패턴으로 풀린다*는 것이 — 이 장의 메타 메시지다. 도메인이 달라도 *그래프 + 재귀*가 같으면 함수는 같다.

이 메시지의 더 큰 자리:
- 가족 관계의 *조상 추적* — 부모-자녀 관계의 transitive closure
- 공급망의 *모든 직간접 공급사* — supply 관계의 transitive closure
- 학계의 *논문 인용 체인* — citation 관계의 transitive closure
- 통신 메시의 *N홉 도달 가능성* — communicates_with의 transitive closure

이 모든 자리가 — *같은 함수 패턴*. 한 도메인에서 익히면 *모든 도메인*에서 작동.

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

### ◇ 설계 결정 — 이진 관계 (subordinate, supervisor)

대안: *N항 관계 (multiple subordinates per supervisor)*
```typeql
relation reporting_unit,
  relates supervisor @card(1..1),
  relates subordinate @card(1..);
```

- 장점: *팀 단위*가 한 매듭. *Bob의 팀원들*이 한 인스턴스
- 단점: *개별 보고 관계*를 별도로 식별하기 어려움
- 단점: 한 부하가 *여러 상사*에게 보고하는 매트릭스 조직 표현 어색

**채택: 이진 관계**
- 각 *보고 관계*가 *개별 매듭*
- *매트릭스 조직*도 자연스럽게 표현 (한 사람이 여러 보고 관계의 subordinate)

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

**짚어둘 점**:
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

## 3.7 재귀 — 본격적 부분

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

## 3.8 ◇ 설계 결정 — Function 설계의 5가지 자리

### 결정 1 — 재귀 vs 반복

*직접 부하부터 깊은 부하까지*를 표현하는 방법:

**반복적 접근** (이론적, TypeDB에서는 직접 표현 어려움):
```pseudocode
visited = {}
queue = [manager]
while queue:
  current = queue.pop()
  for sub in direct_reports_of(current):
    if sub not in visited:
      visited.add(sub)
      queue.push(sub)
return visited
```

**재귀적 접근** (TypeDB의 함수):
```typeql
fun all_subordinates_of($manager: person) -> { person }:
  match (supervisor: $manager, subordinate: $direct) isa reports_to;
  return { $direct };
  or
  match (supervisor: $manager, subordinate: $mid) isa reports_to;
       let $deep in all_subordinates_of($mid);
  return { $deep };
```

**TypeDB가 재귀를 채택한 이유**:
- *선언적* — *무엇을* 답하는가만 적음. *어떻게*는 엔진 책임
- *형식 의미론* 명확 — fixpoint
- *순환 감지·중복 제거*가 엔진에 내장

### 결정 2 — 깊이 제한 (depth-bounded recursion)

기본 재귀는 *fixpoint까지* 완전히 펼침. 그러나 — *깊이 N까지만* 원할 수도.

```typeql
define
  fun subordinates_within_depth(
    $manager: person, 
    $max_depth: long
  ) -> { person }:
    match
      (supervisor: $manager, subordinate: $direct) isa reports_to;
      $max_depth >= 1;
    return { $direct };
    
    or
    
    match
      (supervisor: $manager, subordinate: $mid) isa reports_to;
      $max_depth >= 2;
      let $deep in subordinates_within_depth($mid, $max_depth - 1);
    return { $deep };
```

*깊이 2까지의 부하* = 직속 + 직속의 직속 (손자까지). 깊은 위계에서 *성능 보호* 또는 *부분 분석*에 유용.

### 결정 3 — 집계 함수 (aggregate)

함수가 *집합*이 아니라 *수치*를 반환할 수도.

```typeql
define
  fun subordinate_count($manager: person) -> { long }:
    match
      let $sub in all_subordinates_of($manager);
    return { count($sub) };
```

호출:
```typeql
match
  $alice isa person, has name "Alice";
  let $c in subordinate_count($alice);
fetch $c;
```

답: 4.

TypeDB 3.0은 `count`, `sum`, `min`, `max`, `avg` 같은 집계 함수를 지원. 함수가 *수치 분석 도구*도 됨.

### 결정 4 — 다중 매개변수

함수는 *여러 매개변수*를 받을 수 있다.

```typeql
define
  fun common_supervisor(
    $p1: person, 
    $p2: person
  ) -> { person }:
    match
      (supervisor: $sup, subordinate: $p1) isa reports_to;
      (supervisor: $sup, subordinate: $p2) isa reports_to;
    return { $sup };
```

*David와 Eve의 공통 상사* = Carol.

다중 매개변수가 — *관계 추론*의 표현력을 크게 확장.

### 결정 5 — 부정 (negation)

`not { ... }`를 사용한 함수.

```typeql
define
  fun leaf_nodes() -> { person }:
    match
      $p isa person;
      not { (supervisor: $p) isa reports_to; };
    return { $p };
```

*부하가 없는 사람* — 조직도의 *말단 노드*. David, Eve가 답.

**짚어둘 점**: 부정과 재귀의 결합은 *stratified negation* 조건을 준수해야. TypeDB가 자동 검증.

---

## 3.9 Function의 한계와 자리

도구를 미화하지 않는다. 세 가지 한계를 짚어둔다.

### 한계 1 — 무한 재귀

순환 관계가 데이터에 있으면 — 무한 루프 위험. 가령 *Alice가 Bob에게 보고하고, Bob이 Alice에게 보고*하는 데이터가 들어가면, `all_subordinates_of(Alice)`가 끝나지 않을 수 있다.

TypeDB 엔진은 *순환 감지* 메커니즘을 가지고 있어 안전하게 종료하지만 — *도메인 설계*에서 *순환을 만들지 않는 약속*이 더 안전. 조직도는 위계적이고, 의존성은 *DAG(Directed Acyclic Graph)*여야 한다.

### 한계 2 — 성능

깊은 재귀 + 큰 데이터 = 느려질 수 있다. 1만 명 조직에서 *CEO의 모든 부하*를 묻는 쿼리가 *수십만 매듭*을 펼쳐야 한다면 — 응답 시간이 길어진다.

#### 성능 특성 표

| 그래프 크기 | 재귀 깊이 | 예상 응답 시간 |
|---|---|---|
| 100 노드 | ~3 | < 10ms |
| 1,000 노드 | ~5 | ~50ms |
| 10,000 노드 | ~7 | ~500ms |
| 100,000 노드 | ~10 | ~5초 (depth-bounded 권장) |

해결 방향:
- 깊이 제한 추가 (3.8 결정 2)
- 결과 캐싱 (materialization)
- 인덱스 활용 (TypeDB는 자동 인덱스)

### 한계 3 — 디버깅의 어려움

재귀 함수가 *잘못된 답*을 반환할 때 — *어느 경로*가 잘못됐는지 추적이 일반 query보다 까다롭다. Function 안에서 어떤 경로로 재귀가 펼쳐졌는지를 직접 보기 어렵기 때문이다.

#### 디버깅 전략

1. **작은 데이터로 격리** — 10개 매듭 이하로 줄이고 *수동으로 fixpoint 계산* 후 비교
2. **기저와 재귀 분리** — 기저만 호출, 재귀를 한 단계만 펼침
3. **중간 결과 검증** — `direct_reports_of` 같은 *비재귀 보조 함수*로 그래프 구조 확인
4. **로그 추가** — TypeDB 3.0의 explain 기능 활용

처음에는 *작은 데이터*로 함수를 검증한 뒤, *큰 데이터*로 옮기는 작업 순서를 권한다.

---

## 3.10 실세계 평행 사례

같은 *재귀 그래프 탐색* 패턴이 다른 도메인에서 어떻게 작동하는가.

### npm dependency resolution

JavaScript 패키지 매니저 npm은 *정확히 이 패턴*을 매일 실행한다.

```bash
npm install my-app
# my-app의 모든 직간접 의존성을 자동 다운로드
```

내부적으로:
1. `package.json`을 읽어 *직접 의존성* 목록 확보
2. 각 직접 의존성의 *직접 의존성*을 재귀적으로 확보
3. *transitive closure*가 fixpoint에 도달할 때까지 반복
4. *순환 감지* — A → B → A는 경고
5. *버전 충돌 해결* — 같은 패키지의 다른 버전 요구 시

`all_dependencies_of` 함수가 — *npm 알고리즘의 일급 형식*이다.

### Git ancestry

`git log` 또는 `git blame` 같은 명령이 *커밋의 조상*을 찾는 작업.

```bash
git log --ancestry-path commit_A..commit_B
# A에서 B까지의 조상 사슬
```

내부:
- 각 커밋이 *parent 커밋들*을 가짐 (merge 커밋은 2명+)
- *재귀적으로 parent를 따라감*
- *graph traversal*

`all_subordinates_of`의 *역방향* — *all_ancestors_of*가 정확히 같은 모양.

### Gene Ontology

생명과학의 *Gene Ontology(GO)*는 약 50,000개의 *생물학적 기능* 분류. 각 분류는 *상위 분류와 is-a / part-of 관계*.

```
metabolism
└── carbohydrate metabolism
    └── glucose metabolism
        └── glucose-6-phosphate metabolism
```

*특정 기능의 모든 하위 기능*을 찾는 쿼리가 — 정확히 `all_subordinates_of` 패턴.

### Wikipedia 카테고리

위키피디아의 *카테고리 트리*. *컴퓨터 과학 > 데이터베이스 > 그래프 데이터베이스 > TypeDB*.

*특정 카테고리의 모든 하위 카테고리*를 찾는 작업이 — 다시 같은 패턴.

### 메시지: 도메인 무관성

`all_X_of(start)` 형식의 함수가 *수많은 도메인에서 동일하게 작동*한다는 자리. 한 도메인에서 익히면 — 다른 도메인의 *같은 모양*을 즉시 알아본다.

---

## 3.11 잘못된 함수 vs 좋은 함수

함수 설계의 *안티패턴*과 *좋은 패턴*.

### 안티패턴 1 — 너무 큰 단일 함수

```typeql
fun analyze_organization($manager: person) -> { person }:
  # 50줄의 복잡한 분석 — 부하 + 직속 상사 + 동료 + 특정 조건...
  match ...; return ...;
```

**왜 나쁜가**:
- *재사용 불가* — 50줄 중 일부만 필요한 부분에서 무용
- *디버깅 불가능*
- *함수의 의미*가 불명확

**좋은 패턴**: 작은 함수 여러 개 + 합성
```typeql
fun direct_reports_of($p: person) -> { person }: ...;
fun all_subordinates_of($p: person) -> { person }: ...;
fun manager_of($p: person) -> { person }: ...;
# 합성으로 큰 분석 짓기
```

### 안티패턴 2 — 매개변수 없는 함수의 남용

```typeql
fun all_engineers() -> { person }:
  match $p isa person, has title "Engineer";
  return { $p };
```

**왜 나쁜가**:
- 도메인이 *Engineer*에 고정 — *Senior Engineer*에는 적용 불가
- 새 직책마다 새 함수 — 함수 수가 폭발

**좋은 패턴**: 매개변수화
```typeql
fun persons_with_title($title: string) -> { person }:
  match $p isa person, has title $title;
  return { $p };
```

### 안티패턴 3 — 재귀 없이 깊이 펼치기

```typeql
fun two_levels_below($manager: person) -> { person }:
  match
    (supervisor: $manager, subordinate: $mid) isa reports_to;
    (supervisor: $mid, subordinate: $deep) isa reports_to;
  return { $deep };
```

**왜 나쁜가**:
- *깊이 2에 고정*. 3단계는?
- *깊이 가변*에 대응 못함

**좋은 패턴**: 재귀 함수 + 깊이 매개변수
```typeql
fun subordinates_within_depth($manager: person, $max_depth: long) -> { person }:
  # 3.8 결정 2와 같은 모양
  ...
```

### 안티패턴 4 — 부정과 재귀의 부주의한 결합

```typeql
fun no_subordinates_recursive($p: person) -> { person }:
  not { let $sub in no_subordinates_recursive($p); };  # 자기 참조 부정!
  return { $p };
```

**왜 나쁜가**:
- *Stratified negation* 위반
- *진리값 모순* — 자기 자신이 답에 있는가?
- TypeDB 엔진이 *함수 정의 거부*

**좋은 패턴**: 부정과 재귀를 *명확히 분리*
```typeql
fun has_subordinate($p: person) -> { boolean }:
  match (supervisor: $p) isa reports_to;
  return { true };

fun leaf_nodes() -> { person }:
  match $p isa person;
  not { let $b in has_subordinate($p); };
  return { $p };
```

부정이 *다른 함수의 결과*에만 적용 — stratification 보장.

### 안티패턴 5 — 부작용 (side effect) 시도

TypeDB의 함수는 *읽기 전용*. 데이터를 변경하지 않음. 만약 *함수 내에서 데이터 변경*을 시도하면 — 함수가 *순수 함수형*이 아니게 됨.

**좋은 패턴**: 함수는 *읽기*, 데이터 변경은 *별도 트랜잭션*
```typeql
# 함수: 어떤 분석을 할 것인가
fun candidates_to_promote() -> { person }: ...;

# 트랜잭션: 분석 결과를 토대로 변경
transaction write ...;
match ... let $c in candidates_to_promote();
insert (subordinate: $c, ...) isa reports_to ...;
```

이 분리가 — *데이터베이스의 무결성*과 *함수의 결정성*을 동시에 보장.

---

## 3.12 ◇ 이론 절 — Datalog 전통과 Fixpoint 의미론

### Datalog의 기원

1970년대 후반, *데이터베이스 + 논리 프로그래밍*을 결합하려는 시도가 있었다. 답은 *Datalog* — Prolog의 부분집합으로, *재귀 쿼리*가 자연스럽고 *결정 가능(decidable)*한 언어.

**쉬운 풀이로 보면** — Datalog는 *"~이라면 ~이다"의 규칙*을 적는 언어다. 자연스러운 규칙 두 가지:

```
규칙 A: 만약 X가 Y의 부모라면 → X는 Y의 조상이다.
규칙 B: 만약 X가 Z의 부모이고 Z가 Y의 조상이라면 → X도 Y의 조상이다.
```

이 두 규칙으로 — *조상이라는 개념*을 정의했다. 데이터에는 *직접 부모 관계*만 적혀 있어도 — 규칙이 *모든 직간접 조상*을 자동으로 도출.

Datalog의 핵심 표현 단위는 *Horn clause*:

```
ancestor(X, Y) :- parent(X, Y).
ancestor(X, Y) :- parent(X, Z), ancestor(Z, Y).
```

`:-`는 *"~이라면"* (영어로 *if*). 위 두 줄이 정확히 위의 *규칙 A·B*다. 형식만 다를 뿐 의미는 같다.

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

### Datalog의 의미론적 성질

Datalog가 *수십 년간 살아남은* 이유:

1. **결정 가능 (Decidable)**: 모든 쿼리가 유한 시간에 종료. Prolog의 Turing-complete성을 *제한해서* 얻은 성질.

2. **선언적 (Declarative)**: *무엇을* 답하는가만 적음. *어떻게* 계산하는가는 엔진 책임.

3. **고차원 결합 (Higher-order composition)**: 함수가 함수를 호출 — 합성이 자연스러움.

4. **모델 이론 (Model Theory)**: 모든 Datalog 프로그램은 *minimal model*을 가짐. 그게 *답*이다.

### Fixpoint 의미론

*재귀 함수가 무엇을 의미하는가*에 대한 형식적 답이 *fixpoint semantics*다.

**쉬운 풀이 — 눈사람 굴리기**:

눈사람을 만들 때를 생각해 보자.
- 처음: 작은 눈 뭉치 (한 줌)
- 한 번 굴림: 조금 더 큰 눈 뭉치
- 두 번 굴림: 더 큰 눈 뭉치
- 세 번 굴림: 더 더 큰 눈 뭉치
- ...
- 어느 순간: *더 굴려도 안 커진다* — 주변 눈을 다 묻혀 버렸음

그 *더 이상 안 커지는 순간*이 — *fixpoint*. 형식적으로:

```
F^0 = {} (빈 집합, 시작점)
F^1 = F(F^0) — 한 번 굴린 결과
F^2 = F(F^1) — 두 번 굴린 결과
...
F^∞ = F(F^∞) — 더 굴려도 안 변하는 자리
```

이 *더 이상 변하지 않는 자리*가 *fixpoint*(부동점). 그리고 *가장 작은 fixpoint*(least fixpoint)가 — 재귀 함수의 *정답*이다.

**3장 데이터에 적용해 보자** — `all_subordinates_of(Alice)`의 fixpoint:

```
Step 0: {} (시작)
Step 1: {Bob}              ← Alice의 직속
Step 2: {Bob, Carol}       ← + Bob의 직속
Step 3: {Bob, Carol, David, Eve}  ← + Carol의 직속
Step 4: {Bob, Carol, David, Eve}  ← David·Eve의 직속 없음
        → 변하지 않음 → fixpoint!
```

답: {Bob, Carol, David, Eve}. *눈사람이 더 안 굴러가는 자리*가 답이다.

조직도 예: `all_subordinates_of(CEO)`의 fixpoint 계산
- `step 1`: CEO의 직속 = {VP}
- `step 2`: + VP의 직속 = {VP, Manager}
- `step 3`: + Manager의 직속 = {VP, Manager, Eng1, Eng2}
- `step 4`: Eng1·Eng2는 직속이 없음. 변하지 않음 → fixpoint 도달.

답: {VP, Manager, Eng1, Eng2}.

### Semi-naïve evaluation

TypeDB 엔진은 fixpoint를 *효율적으로* 계산한다. 단순(naïve) 방법은 *매번 처음부터 다시 계산*. Semi-naïve는 *이번 단계에서 새로 추가된 매듭만* 다음 재귀에 사용 — *중복 계산 회피*.

```
naïve:      F^(n+1) = F(F^n)
semi-naïve: F^(n+1) = F^n ∪ F(ΔF^n)
                       where ΔF^n = F^n - F^(n-1)
```

`ΔF^n`만 *다음 재귀의 입력*. 큰 그래프에서 *수백 배 빠름*.

### Stratified Negation의 안전성

*부정(`not { ... }`)*과 *재귀*가 함께 있으면 — 위험한 자리가 있다.

예: *부하가 없는 사람을 찾는 함수*
```
fun has_no_subordinate($p: person) -> { boolean }:
  not { (supervisor: $p) isa reports_to; };
  return { true };
```

이 함수 안에서 *다시 자기 자신*을 재귀로 부르면 — *진리값이 결정되지 않는* 상황이 생길 수 있다 (paradox of self-reference). *Liar paradox*의 데이터베이스 버전.

해결책: **Stratified negation**. 부정과 재귀를 *층(stratum)*으로 분리. 한 층의 함수는 *낮은 층의 함수만* 부정 절 안에서 호출 가능. TypeDB는 이 조건을 자동 검증.

### Closed-World Assumption과의 관계

3.8에서 짚은 부정 함수는 *CWA*를 전제로 한다. 데이터에 *Alice 밑에 부하가 적혀 있지 않다*면 — *Alice는 부하가 없다*고 결론. OWA에서는 *모른다*가 답.

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
| 매개변수 | 없음 | 있음 (재사용성 큼) |
| 반환 타입 | 데이터 변형 | 명시적 타입 |

함수의 *합성성(compositionality)*이 — 추론 도구를 *프로그래밍의 일급 시민*으로 만들었다.

### Datalog의 산업적 자리

Datalog는 *학술의 자리*만이 아니다. 산업에서도 살아 있다:

- **Datomic** (Rich Hickey의 데이터베이스) — Datalog 쿼리
- **LogicBlox** — 비즈니스 분석에서 Datalog
- **Yedalog** (Google) — 대규모 데이터 분석
- **Soufflé** — 정적 분석에서 Datalog
- **TypeDB** — 함수가 Datalog의 직계 후손

40년 전의 *학술 전통*이 *2020년대 산업*에 살아 있는 자리.

---

## 3.13 정리 — 1부 마무리

이 장에서 손에 들어온 것:

**Syntax 어휘**
- `fun X(...) -> { Y }: match ... return { ... };`로 함수 정의
- `let $x in func($arg)`로 함수 호출
- `or`로 분기되는 재귀
- 함수 합성 — 한 함수의 출력에 추가 조건
- 깊이 제한, 집계 함수, 다중 매개변수, 부정

**설계 결정의 트레이드오프** (이 장의 결정적 추가)
- 재귀 vs 반복
- 깊이 제한의 의미와 적용
- 집계 함수의 자리
- 다중 매개변수의 표현력
- 부정과 stratification

**잘못된 함수 vs 좋은 함수**
- 너무 큰 단일 함수의 위험
- 매개변수화의 가치
- 재귀의 올바른 사용
- 부정과 재귀의 안전한 결합
- 함수의 *읽기 전용* 성질

**실세계 평행**
- npm dependency resolution
- Git ancestry
- Gene Ontology
- Wikipedia 카테고리

**이론적 토대**
- *Datalog 전통* — TypeDB 함수가 Horn clause에 대응
- *Fixpoint 의미론* — 재귀 함수의 정답이 least fixpoint
- *Semi-naïve evaluation*의 효율성
- *Stratified negation*의 안전성
- *Function vs Rule*의 설계 결정
- Datalog의 *산업적 자리*

---

## 1부 마무리 — 세 가지 도구가 모인 자리

1부 세 장에서 손에 들어온 도구:

| 장 | 도구 | 이론적 자리 | 잘못된 사용 |
|---|---|---|---|
| 1장 | 분류 (entity 상속) | Subtyping, LSP, Rosch | 평면 모델, 너무 얕음 |
| 2장 | N항 관계와 역할 (relation, plays) | Davidson 사건 의미론 | 이진 분해, attribute 일변도 |
| 3장 | 함수와 재귀 (Function) | Datalog, Fixpoint | 큰 단일 함수, stratification 위반 |

이 세 도구가 *동시에* 작동해야 풀리는 도메인이 — 2부의 드론 군집비행이다. 친숙한 예제에서 익힌 도구가 *진짜 실전*에서 어떻게 결합하는가 — 다음 부에서 본다.

> *손이 익은 도구는 — 도메인이 달라져도 같은 모양으로 작동한다.*

다음 장 — **드론 군집비행의 도메인 분석**. *온톨로지 엔지니어링 방법론*과 *Competency Questions*가 들어온다. 본격 도메인 모델링의 첫 자리.
