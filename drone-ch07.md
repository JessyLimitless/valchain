# 제7장. 핵심 분석 함수 네 개

## 들어가며

5장의 스키마와 6장의 데이터 위에서 — 이 장에서 *분석 함수 네 개*가 정의된다. 4장에서 짚었던 *Competency Questions 12개*의 절반(CQ 1·2·4·7·8·11)을 이 네 함수가 답한다.

이 장은 단순히 *함수를 정의하는 자리*가 아니다. 각 함수의 *3~5가지 사용 사례*, *성능 특성*, *대안 구현*, 그리고 *5가지 합성 패턴*까지 — *진짜 함수를 짜는 사고법*이 본진이다.

이론적 자리: *함수 합성의 형식 의미론*과 *쿼리 최적화*.

---

## 7.1 함수 1 — 능력 매칭 `drones_with_capability`

쿼리를 처음 짜는 사람은 — *match*와 *fetch*로 시작한다. 그것만으로 *대부분의 질문*에 답할 수 있다고 느낀다. 그런데 — *같은 패턴의 쿼리*를 *세 번째 짜는 자리*에서 깨달음이 온다. *이 패턴, 이름을 붙여서 재사용해야 하는 자리다*.

*야간 광학이 가능한 드론*. *통신 중계 5등급 드론*. *장거리 통신 4등급 드론*. 세 쿼리가 모두 *능력 + 등급*이라는 같은 패턴이다. 이걸 *세 번 복사*해서 쓰는 대신 — *능력과 등급을 받아서 드론 리스트를 돌려주는 함수*를 한 번 짜둔다. 그것이 *함수*다. 1부 3장 조직도에서 만난 *재귀 함수*의 친척이지만, 7.1의 이 함수는 *재귀 없는 단순 함수*다 — 함수의 *가장 작은 가족*.

**답하는 CQ 1·2**: *야간 광학 관측이 가능한 드론은 누구인가? 능력 등급 4 이상의 통신 중계 드론은 몇 대인가?*

### 정의

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

### 사용 사례 1 — 임무 시작 전 능력 매칭

```typeql
transaction read drone_swarm
```

*야간 광학 관측이 가능한 드론*:

```typeql
match
  $night_cap isa night_optical, has capability_name "night_optical";
  let $d in drones_with_capability($night_cap, 3);
  $d has serial_id $sid;
fetch $sid;
```

답: DRN-003, DRN-004, DRN-005, DRN-006.

### 사용 사례 2 — 능력 부족 진단

**문제**: 야간 임무인데 *night_optical 4등급 이상이 충분한가?*

```typeql
match
  $night_cap isa night_optical;
  let $d in drones_with_capability($night_cap, 4);
fetch count($d);
```

답: 4대.

*임무 요구사항이 6대*라면 — *부족*. 즉시 분석가에게 알림.

### 사용 사례 3 — 임계 등급 sweep

분석가가 *어느 임계점에서 군집이 충족 가능한가*를 sweep:

```typeql
# 등급 3 이상
match $cap isa night_optical; let $d in drones_with_capability($cap, 3); fetch count($d);
# 답: 5대 (3·4·5등급 모두 포함)

# 등급 4 이상
fetch count($d);  # with $min_grade=4
# 답: 4대

# 등급 5 이상  
fetch count($d);  # with $min_grade=5
# 답: 0대
```

*임계 등급 4가 군집의 최고 보유 수준*임이 한 번에 보임.

### 사용 사례 4 — 다중 능력 교집합

여러 능력을 *동시에 요구*하는 자리:

```typeql
match
  $night isa night_optical;
  $obstacle isa obstacle_avoidance;
  
  let $d1 in drones_with_capability($night, 3);
  let $d2 in drones_with_capability($obstacle, 3);
  $d1 is $d2;
  
  $d1 has serial_id $sid;
fetch $sid;
```

*night_optical 3등급 + obstacle_avoidance 3등급* 둘 다 가진 드론. 답: DRN-003~006.

### 사용 사례 5 — 능력 격차 분석

군집 전체에서 *어느 능력이 가장 풍부하고 어느 능력이 부족한가*:

```typeql
match
  $cap isa capability, has capability_name $cname;
  let $d in drones_with_capability($cap, 1);
fetch $cname; count($d);
```

각 능력별 *최소 1등급 이상 보유 드론 수* — 군집의 *능력 프로필*이 한 쿼리에.

### 성능 특성

| 데이터 규모 | 예상 응답 시간 | 비고 |
|---|---|---|
| 12대 드론, 30개 has_capability | < 10ms | 이 책의 시연 자리 |
| 100대 드론, 500개 매듭 | ~50ms | 인덱스 활용 |
| 1000대 드론, 5000개 매듭 | ~200ms | 능력 entity의 인덱스 필수 |
| 10,000대 드론 | ~1초 | materialization 고려 시점 |

핵심 인덱스: `(capability_type, capability_grade)` 복합 인덱스가 효과.

### 대안 구현

**대안 1: 능력 등급 변환 후 단일 매칭**
```typeql
fun any_capable_drone($req: capability) -> { drone }:
  match (capable_drone: $d, capability_type: $req) isa has_capability;
  return { $d };
```
- 단점: 등급 정보 손실. 임계 매칭 불가.

**대안 2: 매개변수 없이 *최고 등급만* 반환**
```typeql
fun top_capable_drones($req: capability) -> { drone }:
  match
    (capable_drone: $d, capability_type: $req) isa has_capability, has capability_grade $g;
    $g == 5;
  return { $d };
```
- 단점: 임계점이 고정. 분석가의 *해석 권한* 손실.

**현재 채택**: 매개변수화된 버전이 *가장 유연*하고 *재사용 가능*.

---

## 7.2 함수 2 — 리더 후보 도출 `leader_candidates`

**답하는 CQ 11**: *현재 리더가 손실되면 누가 대체할 수 있는가?*

### 정의

```typeql
define
  fun leader_candidates($current_mission: mission) -> { drone }:
    match
      $cap_lead isa formation_lead;
      (capable_drone: $d, capability_type: $cap_lead) isa has_capability,
        has capability_grade $g;
      $g >= 4;
      
      (assigned_drone: $d, assignment_mission: $current_mission) isa assigned_to;
      
      not {
        $leader_role isa role_def, has role_type "leader";
        (assigned_drone: $d, assignment_mission: $current_mission, played_role: $leader_role) 
          isa assigned_to;
      };
    return { $d };
commit;
```

### 사용 사례 1 — 즉시 대체 후보

```typeql
match
  $m1 isa mission, has mission_id "MSN-2026-04-01-001";
  let $cand in leader_candidates($m1);
  $cand has serial_id $sid;
fetch $sid;
```

답: DRN-002 (현재 follower, formation_lead 5등급 보유).

### 사용 사례 2 — 임무 시작 전 *backup leader* 예약

리더 손실에 *미리 대비*하려면 — 임무 시작 시점에 *backup leader 자격*을 가진 드론을 *별도 표시*. 함수는 그대로 활용:

```typeql
match
  $m1 isa mission, has mission_id "MSN-2026-04-01-001";
  let $cand in leader_candidates($m1);
  # 미래: backup_leader 역할 추가 할당
fetch $cand;
```

### 사용 사례 3 — *리더 후보 깊이* 분석

군집의 *리더 승계 깊이*가 충분한가:

```typeql
match
  $m1 isa mission, has mission_id "MSN-2026-04-01-001";
  let $cand in leader_candidates($m1);
fetch count($cand);
```

답: 1대. *리더 후보가 1대뿐* → 만약 그 1대도 손실되면 *임무 중단 위험*. 분석가에게 *리더 후보 부족* 경고.

### 사용 사례 4 — 다른 임무에 같은 함수

함수가 *어떤 임무에도* 적용 가능. 임무 매개변수만 바꾸면:

```typeql
match
  $m2 isa mission, has mission_id "MSN-2026-04-15-002";  # 다른 임무
  let $cand in leader_candidates($m2);
fetch $cand;
```

같은 코드, 다른 답. 함수의 *재사용성*.

### ◇ 설계 결정 — 왜 "비-리더" 조건이 있는가

```typeql
not {
  (... played_role: $leader_role ...) isa assigned_to;
};
```

이 부정 절이 *없다면* — 현재 리더 본인이 후보에 포함됨. 즉 *손실된 본인을 대체 후보로*. 무의미.

이 부정 절은 — *손실 시나리오*를 가정한 설계다. 만약 *지원 리더 후보 명단*(손실 전 미리 알기)을 원한다면 — 별도 함수 `potential_leader_candidates`가 적절.

---

## 7.3 함수 3 — 통신 경로 재라우팅 `reachable_through_swarm`

**답하는 CQ 7·8**: *직접 링크가 없는 두 드론이 메시 네트워크로 도달 가능한가?*

### 정의

```typeql
define
  fun reachable_through_swarm($from: drone, $excluded: drone) -> { drone }:
    match
      ($from, $direct) isa communicates_with;
      not { $direct is $excluded; };
    return { $direct };
    
    or
    
    match
      ($from, $mid) isa communicates_with;
      not { $mid is $excluded; };
      let $deep in reachable_through_swarm($mid, $excluded);
    return { $deep };
commit;
```

### 사용 사례 1 — 직접 단절 시 우회 경로

DRN-005가 DRN-009와 직접 링크 끊긴 상황. 그래도 도달 가능한가:

```typeql
match
  $d05 isa drone, has serial_id "DRN-005";
  $d09 isa drone, has serial_id "DRN-009";
  $excluded isa drone, has serial_id "DRN-005";
  
  let $r in reachable_through_swarm($d05, $excluded);
  $r has serial_id "DRN-009";
fetch $r;
```

답이 있으면 — *우회 경로 존재* (예: DRN-005 → DRN-010 → DRN-009).

### 사용 사례 2 — 메시 견고성 분석

*어떤 드론을 제거해도 메시가 견고한가*를 sweep:

```typeql
# 모든 드론 d에 대해, d 제외 시 다른 드론들이 여전히 연결되는가
match
  $excluded isa drone, has serial_id $excluded_sid;
  $d01 isa drone, has serial_id "DRN-001";
  $d02 isa drone, has serial_id "DRN-002";
  not { $excluded is $d01; not { $excluded is $d02; }; };  # 단순화
  
  let $r in reachable_through_swarm($d01, $excluded);
  $r is $d02;
fetch $excluded_sid;
```

각 드론 제거 시나리오마다 — *연결성 유지 여부*. 결과로 *cut vertex*(제거 시 메시 분리되는 드론) 식별.

이 분석이 *예방적 군집 설계*에 활용: cut vertex가 있다면 — *해당 드론에 백업 추가* 또는 *대체 링크 구축*.

### 사용 사례 3 — 통신 지연 추정

재귀 깊이가 *통신 hop 수*에 대응. 깊이 제한 버전:

```typeql
define
  fun reachable_within_hops(
    $from: drone, 
    $excluded: drone,
    $max_hops: long
  ) -> { drone }:
    match
      ($from, $direct) isa communicates_with;
      not { $direct is $excluded; };
      $max_hops >= 1;
    return { $direct };
    
    or
    
    match
      ($from, $mid) isa communicates_with;
      not { $mid is $excluded; };
      $max_hops >= 2;
      let $deep in reachable_within_hops($mid, $excluded, $max_hops - 1);
    return { $deep };
```

*2 hops 이내*만 도달 가능한 드론 — 지연 민감 임무에 활용.

### 사용 사례 4 — 링크 품질 필터링

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
```

저품질 링크가 *대안에서 제외*됨. 신뢰 통신만으로의 경로 분석.

### 성능 특성

| 그래프 크기 | 예상 응답 시간 |
|---|---|
| 12 노드, 19 엣지 (이 책) | < 5ms |
| 100 노드, 500 엣지 | ~30ms (재귀 깊이 ~5) |
| 1000 노드, 5000 엣지 | ~200ms (depth-bounded 권장) |
| 10,000 노드 | ~수초 (materialization 또는 별도 그래프 DB 고려) |

재귀 성능의 핵심: *fixpoint 도달 속도*. TypeDB는 semi-naïve evaluation을 내장하므로 — 중복 계산이 자동 회피.

---

## 7.4 함수 4 — 역할별 동료 도출 `drones_with_role`

**답하는 CQ 4·5**: *현재 임무의 모든 observer는 누구인가?*

### 정의

```typeql
define
  fun drones_with_role(
    $mission: mission, 
    $role_type_name: string
  ) -> { drone }:
    match
      $role isa role_def, has role_type $role_type_name;
      (assigned_drone: $d, assignment_mission: $mission, played_role: $role) 
        isa assigned_to;
    return { $d };
commit;
```

### 사용 사례 1 — 현재 역할별 멤버

```typeql
match
  $m1 isa mission, has mission_id "MSN-2026-04-01-001";
  let $obs in drones_with_role($m1, "observer");
  $obs has serial_id $sid;
fetch $sid;
```

답: DRN-003, 004, 005, 006.

### 사용 사례 2 — 역할 분포 진단

군집의 *역할별 인원*이 임무 요구사항과 맞는가:

```typeql
match
  $m1 isa mission, has mission_id "MSN-2026-04-01-001";
  let $leader in drones_with_role($m1, "leader");
  let $observer in drones_with_role($m1, "observer");
  let $rescuer in drones_with_role($m1, "rescuer");
  let $relay in drones_with_role($m1, "relay");
fetch count($leader); count($observer); count($rescuer); count($relay);
```

답: leader 1, observer 4, rescuer 2, relay 3.

### 사용 사례 3 — 손실 시 같은 역할 동료 찾기

DRN-003(observer)이 손실되면 — 같은 observer 역할의 다른 드론들이 *작업을 분담*할 수 있는가:

```typeql
match
  $m1 isa mission, has mission_id "MSN-2026-04-01-001";
  $lost isa drone, has serial_id "DRN-003";
  
  let $peer in drones_with_role($m1, "observer");
  not { $peer is $lost; };
  
  $peer has serial_id $sid;
fetch $sid;
```

답: DRN-004, 005, 006. 세 명이 잔존 — *분담 가능*.

### 사용 사례 4 — 시간축 적용 (이력 추적)

`assigned_to`에 *start/end* 시점이 있으므로 — 특정 시점의 역할 분포 조회 가능:

```typeql
match
  $m1 isa mission;
  $role isa role_def, has role_type "leader";
  $a (assigned_drone: $d, assignment_mission: $m1, played_role: $role) isa assigned_to;
  $a has assignment_start $start;
  $start <= 2026-04-01T17:23:00;
  not { 
    $a has assignment_end $end; 
    $end <= 2026-04-01T17:23:00;
  };
  $d has serial_id $sid;
fetch $sid;
```

*손실 시점(17:23) 직전*의 리더 — DRN-001.

이 패턴이 *임무 회고 분석*(post-mission analysis)에 활용.

---

## 7.5 함수 합성의 다섯 가지 패턴

함수 4개가 *어떻게 합성되는가*. 5가지 패턴이 있다.

### 패턴 1 — 교집합 (intersection)

여러 함수의 *공통 답*만:

```typeql
match
  let $a in func1(...);
  let $b in func2(...);
  $a is $b;
fetch $a;
```

**예**: 7.1 (능력) ∩ 7.2 (리더 후보) ∩ 7.3 (통신 도달) — 8장 5절의 *대체 드론 도출*.

### 패턴 2 — 차집합 (difference)

한 함수의 답 *중 다른 함수에 없는 것*:

```typeql
match
  let $a in func_all(...);
  not { let $b in func_exclude(...); $a is $b; };
fetch $a;
```

**예**: *전체 드론 중 임무에 배치되지 않은 예비 드론*:
```typeql
match
  $d isa drone;
  not { 
    $m1 isa mission, has mission_id "MSN-001";
    (assigned_drone: $d, assignment_mission: $m1) isa assigned_to;
  };
fetch $d;
```

답: DRN-012 (예비).

### 패턴 3 — 매개변수 전파

한 함수의 출력이 *다른 함수의 입력*으로:

```typeql
match
  let $first in func1(...);
  let $second in func2($first);
fetch $second;
```

**예**: *직속 부하의 부하* (3장 조직도):
```typeql
match
  let $direct in direct_reports_of($manager);
  let $deep in direct_reports_of($direct);
fetch $deep;
```

이게 *재귀의 한 단계 펼침*. 무한 깊이라면 *재귀 함수* 자체로.

### 패턴 4 — 단계별 필터링 (pipeline)

여러 조건을 *순차*로 좁히기:

```typeql
match
  let $stage1 in capability_filter(...);
  $stage1 has some_attr $a;
  $a > threshold;
  let $stage2 in mission_filter($stage1);
  ... ;
fetch $stage2;
```

각 단계가 *답 집합을 좁힘*. 7.1 → 추가 속성 조건 → 7.4 같은 흐름.

### 패턴 5 — 임계점 sweep

매개변수를 *변경하며 반복 호출*:

```typeql
# 등급 3 이상
fetch count(drones_with_capability($cap, 3));
# 등급 4 이상  
fetch count(drones_with_capability($cap, 4));
# 등급 5 이상
fetch count(drones_with_capability($cap, 5));
```

각 임계점의 *답 크기 분포*. 분석가가 *적절한 임계*를 시각화·결정.

### 패턴 활용의 자리

이 5가지 패턴을 *명시적으로 의식하면* — 새 분석 요청이 들어와도 *어느 패턴인지* 빠르게 분류 가능. 그리고 — *기존 함수 4개*의 합성으로 답이 풀리는 경우가 대부분.

8장 시나리오에서 — 이 패턴들이 *실제로 어떻게 적용*되는가 본격적으로.

---

## 7.6 ◇ 이론 절 — 합성성과 쿼리 최적화

### 합성성(Compositionality)

이 장의 네 함수가 *합성될 수 있는* 자리에 본질이 있다. 

**쉬운 풀이 — 레고 블록의 비유**:

레고 블록은 *각각이 단순한 모양*이다. 그러나 *조합하면* 자동차, 우주선, 도시 어느 것이든 짓는다. 핵심은 *블록의 인터페이스가 일관*돼서 — *어느 두 블록도 결합 가능*하다는 점.

함수도 같은 자리에 있다. 각 함수가 *작고 단순한 도메인 작업*. 그러나 — *한 함수의 출력이 다른 함수의 입력*이 될 수 있어서 *합성으로 큰 분석*을 짓는다. 

8장 시나리오에서:

```
대체 후보 = 
  drones_with_capability(formation_lead, 5)  ← 7.1
  ∩ leader_candidates(mission)                ← 7.2
  ∩ {d | d09 ∈ reachable_through_swarm(d, d_lost)}  ← 7.3
```

세 함수의 *교집합*으로 답이 도출된다 — 세 블록을 조합한 결과. 합성성은 *함수형 프로그래밍의 핵심 원칙* — 작은 함수의 합성으로 큰 시스템을 짓는다.

TypeDB의 함수는 *Datalog 전통*에 있으므로 — 합성이 *형식적으로 보장*된다. 한 함수의 fixpoint가 다른 함수에 입력으로 들어갈 때, 전체 시스템의 fixpoint가 *결정 가능*하다 (단, stratified negation 조건 하에서).

### 쿼리 최적화의 자리

TypeDB는 함수 호출 시 *쿼리 플래너*가 작동한다. 세 가지 최적화 기법:

**1. Predicate Pushdown**
- 부정 절(`not { ... }`)을 *가능한 일찍* 적용해 검색 공간 축소
- `$g >= 4` 같은 비교 술어를 *데이터 스캔 시점*에 평가
- 결과: 메모리에 적재되는 매듭 수 감소

**2. Join Order Optimization**
- 다중 매듭 매칭 시 *선택성 높은 매듭 먼저*
- 통계(각 entity의 인스턴스 수, 각 관계의 평균 자리 수)를 활용
- 결과: 중간 결과 집합 크기 최소화

**3. Function Memoization**
- 재귀 호출에서 *중복 계산 회피*
- `reachable_through_swarm(d09, d05)`가 여러 경로에서 호출되더라도 *한 번만 계산*
- 결과: 재귀 깊이가 깊을수록 효과 큼

이 최적화들이 — 짜는 사람이 *의식하지 않아도 자동으로 적용*된다. 함수형 패러다임의 또 한 가지 가치.

### Stratification — 부정과 재귀의 안전성

3장에서 짚었던 *stratified negation*이 이 장에서 결정적이다. 함수 7.2와 7.3 모두 *`not { ... }`을 사용*한다. 7.3은 *재귀 함수*이기도 하다.

TypeDB는 *부정이 들어간 자리*가 *낮은 stratum의 함수만 호출*하는지 자동 검증. 위반 시 — 함수 정의 단계에서 거부.

이게 *추론의 안전성*을 보장한다. 모든 함수가 *유한 시간에 종료*하고 *결정 가능한 답*을 준다.

### Materialization vs On-the-fly

TypeDB의 함수는 *기본적으로 on-the-fly* — 호출 시점에 fixpoint 계산. 이게 일관성을 보장하지만 — *큰 그래프*에서는 느릴 수 있다.

대안: *Materialization*. 함수의 결과를 *별도 entity/relation으로 저장*. 데이터 변경 시 *자동 갱신*.

운영 시스템의 *결정 기준*:
- *자주 묻는 함수* → materialize
- *드물게 묻는 함수* → on-the-fly
- *데이터 갱신 빈도*가 materialization의 trigger
- *consistency 요구*가 strategy 결정

### 함수의 디버깅

재귀 함수가 *잘못된 답*을 반환할 때 — 디버깅 전략:

1. **작은 데이터로 격리** — 데이터를 10개 매듭 이하로 줄이고 *수동으로 fixpoint 계산* 후 비교
2. **기저와 재귀를 분리** — 기저만 호출해 보고, 재귀를 한 단계만 펼쳐 보고
3. **`fetch all` 사용** — 중간 결과를 *모두* 받아서 확인
4. **로그 추가** — 함수 안에 임시 `print` (TypeDB 3.0 디버그 모드)

진짜 시스템에서는 — *함수 호출 트레이스*를 자동 수집하는 도구가 필요.

---

## 7.7 정리

이 장에서 손에 들어온 것:

**네 함수**
| 함수 | 답하는 CQ | 패턴 |
|---|---|---|
| `drones_with_capability` | CQ 1·2·3 | 관계 매칭 + 비교 |
| `leader_candidates` | CQ 11 | 다중 관계 + 부정 합성 |
| `reachable_through_swarm` | CQ 7·8·9 | 재귀 + 부정 |
| `drones_with_role` | CQ 4·5·6 | 문자열 + entity 매개변수 |

**각 함수의 사용 사례 (이 장의 결정적 확장)**
- 사용 사례 4~5개씩 — 단순 호출부터 복잡 합성까지
- 성능 특성 표 — 데이터 규모별 응답 시간
- 대안 구현 — 왜 현재 모양이 최선인가

**합성의 5가지 패턴**
- 교집합 / 차집합 / 매개변수 전파 / 단계별 필터링 / 임계점 sweep
- 8장 시나리오에서 *실제 적용*

**이론적 자리**
- 합성성 — 함수의 교집합으로 답 도출
- 쿼리 최적화 — predicate pushdown, join order, memoization
- Stratified negation — 안전성 보장
- Materialization vs on-the-fly의 트레이드오프

다음 장 — **시나리오 클라이맥스**. 이 네 함수가 *동시에 호출*되어, *한 드론이 손실된 순간 영향이 자동으로 풀려나가는* 자리. 그리고 — *다양한 사건 시나리오*까지 본격 시연.
