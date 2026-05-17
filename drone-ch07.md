# 제7장. 핵심 분석 함수 네 개

## 들어가며

5장의 스키마와 6장의 데이터 위에서 — 이 장에서 *분석 함수 네 개*가 정의된다. 4장에서 짚었던 *Competency Questions 12개*의 절반(CQ 1·2·4·7·8·11)을 이 네 함수가 답한다.

이 장의 이론적 자리: *함수 합성의 형식 의미론*과 *쿼리 최적화의 자리*.

---

## 7.1 함수 1 — 능력 매칭 `drones_with_capability`

**CQ 1·2가 묻는 자리**: *야간 광학 관측이 가능한 드론은 누구인가? 능력 등급 4 이상의 통신 중계 드론은 몇 대인가?*

```typeql
transaction schema drone_swarm
```

```typeql
define
  fun drones_with_capability(
    $required_cap: capability,
    $min_grade: long
  ) -> { drone }:
    match
      (capable_drone: $d, capability_type: $required_cap) isa has_capability,
        has capability_grade $g;
      $g >= $min_grade;
    return { $d };
commit;
```

호출:
```typeql
transaction read drone_swarm
```

```typeql
match
  $night_cap isa night_optical, has capability_name "night_optical";
  let $d in drones_with_capability($night_cap, 3);
  $d has serial_id $sid;
fetch $sid;
```

**예상 답**: DRN-003 ~ 006 (광학 관측 4대 — 등급 4 보유 → 임계 3 충족).

함수의 핵심:
- *능력 entity*와 *최소 등급*을 매개변수로
- *함수의 코드는 한 줄도 변하지 않고* 모든 능력 종류에 적용
- 분석가가 *임계 등급*을 호출 시점에 결정 — 도메인 사고가 *함수의 매개변수*에 들어감

---

## 7.2 함수 2 — 리더 후보 도출 `leader_candidates`

**CQ 11이 묻는 자리**: *현재 리더가 손실되면 누가 대체할 수 있는가?*

조건:
- formation_lead 능력 4등급 이상
- 현재 임무에 이미 *비-리더* 역할로 배치된 드론 (아예 임무 밖의 예비는 제외)

```typeql
define
  fun leader_candidates($current_mission: mission) -> { drone }:
    match
      # formation_lead 능력 4등급 이상
      $cap_lead isa formation_lead;
      (capable_drone: $d, capability_type: $cap_lead) isa has_capability,
        has capability_grade $g;
      $g >= 4;
      
      # 현재 임무에 배치되어 있고
      (assigned_drone: $d, assignment_mission: $current_mission) isa assigned_to;
      
      # 그 역할이 leader가 아님
      not {
        $leader_role isa role_def, has role_type "leader";
        (assigned_drone: $d, assignment_mission: $current_mission, played_role: $leader_role) 
          isa assigned_to;
      };
    return { $d };
commit;
```

호출:

```typeql
match
  $m1 isa mission, has mission_id "MSN-2026-04-01-001";
  let $cand in leader_candidates($m1);
  $cand has serial_id $sid;
fetch $sid;
```

**예상 답**: DRN-002 (formation_lead 5등급, 현재 follower 역할 — 즉시 리더 승격 가능).

**짚어둘 합성**:
- `has_capability` 관계 + `assigned_to` 관계 + `not { ... }` 부정의 결합
- 1부 예제 3에서 익힌 *함수 합성*이 *진짜 도메인*에서 작동
- 같은 함수가 *다른 임무*에도 그대로 적용 — 도메인 어휘의 재사용

---

## 7.3 함수 3 — 통신 경로 재라우팅 `reachable_through_swarm`

**CQ 7·8이 묻는 자리**: *직접 링크가 없는 두 드론이 메시 네트워크로 도달 가능한가?*

```typeql
define
  fun reachable_through_swarm($from: drone, $excluded: drone) -> { drone }:
    # 기저 — excluded 제외하고 직접 연결된 드론
    match
      ($from, $direct) isa communicates_with;
      not { $direct is $excluded; };
    return { $direct };
    
    or
    
    # 재귀 — 중간 드론을 통한 간접 도달
    match
      ($from, $mid) isa communicates_with;
      not { $mid is $excluded; };
      let $deep in reachable_through_swarm($mid, $excluded);
    return { $deep };
commit;
```

호출 — *DRN-005가 DRN-009와 직접 링크 끊긴 상황에서 (DRN-005 제외) 도달 가능한지*:

```typeql
match
  $d05 isa drone, has serial_id "DRN-005";
  $d09 isa drone, has serial_id "DRN-009";
  
  let $r in reachable_through_swarm($d05, $d05);  # $d05 자기 자신을 excluded로
  $r has serial_id "DRN-009";
fetch $r;
```

답이 있으면 — *DRN-009로 도달 가능*. 6장 데이터에서 d05 ↔ d10 ↔ d09 경로가 있으므로 도달.

**짚어둘 자리**:
- 1부 예제 3의 *all_subordinates_of*와 정확히 같은 패턴
- *기저 + 재귀 + or* + *부정 절(`not`)* — 안전한 그래프 탐색
- *도달 가능한 드론 집합*을 fixpoint로 자동 계산

### 부분 함수 변형 — 링크 품질 임계

CQ 9: *링크 품질 0.7 이상의 경로만으로 도달 가능한가?*

```typeql
define
  fun reachable_with_quality(
    $from: drone, 
    $excluded: drone, 
    $min_quality: double
  ) -> { drone }:
    match
      $link ($from, $direct) isa communicates_with;
      $link has link_quality $q;
      $q >= $min_quality;
      not { $direct is $excluded; };
    return { $direct };
    
    or
    
    match
      $link ($from, $mid) isa communicates_with;
      $link has link_quality $q;
      $q >= $min_quality;
      not { $mid is $excluded; };
      let $deep in reachable_with_quality($mid, $excluded, $min_quality);
    return { $deep };
commit;
```

매개변수에 *최소 품질*을 추가. 같은 패턴 + 한 줄 추가로 *훨씬 엄격한 분석*이 가능.

---

## 7.4 함수 4 — 역할별 동료 도출 `drones_with_role`

**CQ 4·5가 묻는 자리**: *현재 임무의 모든 observer는 누구인가?*

```typeql
define
  fun drones_with_role($mission: mission, $role_type_name: string) -> { drone }:
    match
      $role isa role_def, has role_type $role_type_name;
      (assigned_drone: $d, assignment_mission: $mission, played_role: $role) 
        isa assigned_to;
    return { $d };
commit;
```

호출:

```typeql
match
  $m1 isa mission, has mission_id "MSN-2026-04-01-001";
  let $obs in drones_with_role($m1, "observer");
  $obs has serial_id $sid;
fetch $sid;
```

**예상 답**: DRN-003, DRN-004, DRN-005, DRN-006.

**짚어둘 자리**:
- *문자열 매개변수* + *entity 매개변수*의 결합
- 8장 손실 시나리오에서 *같은 역할의 다른 드론* 찾는 자리에 사용
- 짧고 강력한 함수 — *역할 기반 협조* 패러다임의 일급 도구

---

## 7.5 ◇ 이론 절 — 함수 합성과 쿼리 최적화

### 합성성(Compositionality)

이 장의 네 함수가 *합성될 수 있는* 자리에 본질이 있다. 8장 시나리오에서:

```
대체 후보 = 
  drones_with_capability(formation_lead, 5)  ← 7.1
  ∩ leader_candidates(mission)                ← 7.2
  ∩ {d | d09 ∈ reachable_through_swarm(d, d_lost)}  ← 7.3
```

세 함수의 *교집합*으로 답이 도출된다. 합성성은 *함수형 프로그래밍의 핵심 원칙* — 작은 함수의 합성으로 큰 시스템을 짓는다.

TypeDB의 함수는 *Datalog 전통*에 있으므로 — 합성이 *형식적으로 보장*된다. 한 함수의 fixpoint가 다른 함수에 입력으로 들어갈 때, 전체 시스템의 fixpoint가 *결정 가능*하다 (단, stratified negation 조건 하에서).

### 쿼리 최적화의 자리

TypeDB는 함수 호출 시 *쿼리 플래너*가 작동한다. 세 가지 최적화 기법:

**1. Predicate Pushdown**
- 부정 절(`not { ... }`)을 *가능한 일찍* 적용해 검색 공간 축소
- `$g >= 4` 같은 비교 술어를 *데이터 스캔 시점*에 평가

**2. Join Order Optimization**
- 다중 매듭 매칭 시 *선택성 높은 매듭 먼저*
- 통계(각 entity의 인스턴스 수, 각 관계의 평균 자리 수)를 활용

**3. Function Memoization**
- 재귀 호출에서 *중복 계산 회피*
- `reachable_through_swarm(d09, d05)`가 여러 경로에서 호출되더라도 *한 번만 계산*

이 최적화들이 — 짜는 사람이 *의식하지 않아도 자동으로 적용*된다. 함수형 패러다임의 또 한 가지 가치.

### Stratification — 부정과 재귀의 안전성

3장에서 짚었던 *stratified negation*이 이 장에서 결정적이다. 함수 7.2와 7.3 모두 *`not { ... }`을 사용*한다. 7.3은 *재귀 함수*이기도 하다.

TypeDB는 *부정이 들어간 자리*가 *낮은 stratum의 함수만 호출*하는지 자동 검증. 위반 시 — 함수 정의 단계에서 거부.

이게 *추론의 안전성*을 보장한다. 모든 함수가 *유한 시간에 종료*하고 *결정 가능한 답*을 준다.

### Materialization vs On-the-fly

TypeDB의 함수는 *기본적으로 on-the-fly* — 호출 시점에 fixpoint 계산. 이게 일관성을 보장하지만 — *큰 그래프*에서는 느릴 수 있다.

대안: *Materialization*. 함수의 결과를 *별도 entity/relation으로 저장*. 데이터 변경 시 *자동 갱신*. 

TypeDB는 현재 자동 materialization을 제공하지 않지만 — *명시적으로 짤 수 있다*:

```typeql
define
  entity precomputed_reachability,
    owns from_drone @card(1..1),
    owns to_drone @card(1..1),
    owns computed_at @card(1..1);
```

쓰기 트랜잭션에서 *재귀 함수를 호출해 결과를 entity로 저장*. 읽기 트랜잭션에서는 *그 entity만 쿼리*. *공간을 쓰고 시간을 산다*는 고전적 트레이드오프.

---

## 7.6 정리

이 장에서 손에 들어온 것:

**네 함수**
| 함수 | 답하는 CQ | 패턴 |
|---|---|---|
| `drones_with_capability` | CQ 1·2 | 관계 매칭 + 비교 |
| `leader_candidates` | CQ 11 | 다중 관계 + 부정 합성 |
| `reachable_through_swarm` | CQ 7·8 | 재귀 + 부정 |
| `drones_with_role` | CQ 4·5 | 문자열 + entity 매개변수 |

**이론적 자리**
- 합성성 — 함수의 교집합으로 답 도출
- 쿼리 최적화 — predicate pushdown, join order, memoization
- Stratified negation — 안전성 보장
- Materialization vs on-the-fly의 트레이드오프

다음 장 — **시나리오 클라이맥스**. 이 네 함수가 *동시에 호출*되어, *한 드론이 손실된 순간 영향이 자동으로 풀려나가는* 자리. Forward chaining과 Truth maintenance의 이론적 자리도 함께.
