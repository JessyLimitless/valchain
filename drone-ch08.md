# 제8장. 시나리오 — 한 드론이 손실되었을 때

## 들어가며

이 장이 책의 *클라이맥스*다. 7장에서 짠 네 함수가 *동시에 호출*되어, *한 사건의 영향*이 시스템 안에서 자동으로 펼쳐진다. 사건은 단순하다 — *현재 임무의 리더 드론이 통신 두절로 손실되었다*. 그러나 그 영향은 *역할 공백, 통신 경로 단절, 능력 손실, 편대 와해* — 네 가지가 동시에 일어나는 자리다.

이 장의 이론적 자리: *Forward chaining vs Backward chaining*, *Truth maintenance*, 그리고 *시스템이 답하지 않는 자리*의 정직한 가르기.

---

## 8.1 사건 매듭의 정의

### `drone_loss` 관계 — Reified Event Pattern

5.1에서 ODP로 짚은 *Reified Event Pattern*이 이 자리에서 본격 활용된다. 사건 자체를 entity가 아닌 *N항 관계*로 표현 — 사건이 묶는 *주체, 영향 대상들, 시점, 원인*이 한 매듭에 들어감.

```typeql
transaction schema drone_swarm
```

```typeql
define
  relation drone_loss,
    relates lost_drone @card(1..1),
    relates affected_mission @card(0..),
    relates affected_link @card(0..),
    owns loss_time @card(1..1),
    owns loss_cause @card(0..1),
    owns severity @card(0..1);
  
  drone plays drone_loss:lost_drone;
  mission plays drone_loss:affected_mission;
  communicates_with plays drone_loss:affected_link;
  
  attribute loss_time, value datetime;
  attribute loss_cause, value string 
    @values("communication_failure", "battery_depleted", "mechanical_failure", "weather", "unknown");
  attribute severity, value string 
    @values("minor", "moderate", "severe", "critical");
commit;
```

**짚어둘 자리** — `communicates_with` 관계가 *plays*로 `affected_link` 자리에 들어감. **관계 인스턴스가 다른 관계의 자리에 묶이는 모양**. RDF로는 *reification of reification*이라는 이중 우회가 필요한 자리가 — TypeDB에서는 한 줄.

---

## 8.2 손실 사건 입력

```typeql
transaction write drone_swarm
```

```typeql
match
  $d01 isa drone, has serial_id "DRN-001";
  $m1 isa mission, has mission_id "MSN-2026-04-01-001";

insert
  $event isa drone_loss,
    has loss_time 2026-04-01T17:23:00,
    has loss_cause "communication_failure",
    has severity "severe";
  
  $event links (lost_drone: $d01);
  $event links (affected_mission: $m1);
commit;
```

손실 시점 — 임무 시작 1시간 23분 후. 시간 위에 박힌 한 사건이 — *시스템이 즉시 작동하기 시작하는 자리*다.

---

## 8.3 ◇ 이론 절 — Forward Chaining vs Backward Chaining

본격 영향 도출 전에 — *추론의 두 방향*을 짚는다.

### 두 방향의 추론

추론 시스템은 *어느 방향으로 작동하는가*에 따라 두 가지로 나뉜다.

**Forward Chaining (전진 추론)**
- *데이터에서 결론으로*
- 새 사실이 입력되면 → 적용 가능한 규칙을 모두 발동 → 도출되는 모든 새 사실을 저장
- *Materialization* 전략과 결합
- 장점: 쿼리가 빠름 (이미 도출된 사실을 조회만)
- 단점: 데이터 변경마다 비용, 사용되지 않는 사실도 계산

**Backward Chaining (후진 추론)**
- *질문에서 데이터로*
- 질문이 들어오면 → 답에 필요한 사실을 *역방향으로* 추적
- *On-the-fly* 전략과 결합
- 장점: 질문에 필요한 사실만 계산
- 단점: 같은 질문 반복 시 매번 재계산

### TypeDB의 자리

TypeDB의 함수는 *기본적으로 backward chaining*이다.
- `let $r in leader_candidates($m1)` 호출 → 그 시점에 fixpoint 계산
- 데이터가 변경되면 → 다음 호출에서 자동 반영
- *일관성 보장*

그러나 — *큰 그래프*에서 같은 질문이 자주 호출되면, *명시적 materialization*이 가치 있다 (7.5 참조).

### 이 장의 자리

8.4 이하에서 — 손실 사건의 영향은 *backward chaining*으로 도출된다. *어느 함수가 어느 사실에 의존하는가*를 명시적으로 짚으면서 진행.

---

## 8.4 영향 자동 도출 — 네 함수의 호출

손실 사건 직후 — *시스템이 분석가에게 답해야 할 네 가지 질문*.

### Q1. 어느 역할이 공백이 되는가

```typeql
transaction read drone_swarm
```

```typeql
match
  $d01 isa drone, has serial_id "DRN-001";
  $role isa role_def;
  (assigned_drone: $d01, played_role: $role) isa assigned_to;
  $role has role_type $role_type;
fetch $role_type;
```

답: `leader`. *임무의 리더 자리가 공백*.

### Q2. 어느 통신 링크가 끊기는가

```typeql
match
  $d01 isa drone, has serial_id "DRN-001";
  ($d01, $other) isa communicates_with;
  $other has serial_id $sid;
fetch $sid;
```

답: DRN-002, DRN-009, DRN-010, DRN-011 — 리더가 *4개 링크의 한 끝*이었음.

### Q3. 통신 경로가 여전히 도달 가능한가 (7.3 호출)

각 *남은 드론 쌍*에 대해 — DRN-001을 제외하고도 *재귀 경로*가 있는가:

```typeql
match
  $d05 isa drone, has serial_id "DRN-005";
  $d09 isa drone, has serial_id "DRN-009";
  $d01 isa drone, has serial_id "DRN-001";
  
  let $r in reachable_through_swarm($d05, $d01);
  $r has serial_id "DRN-009";
fetch $r;
```

답이 있으면 — *DRN-005 → DRN-010 → DRN-009* 같은 우회 경로 존재. 답이 없으면 — 고립.

### Q4. 능력의 부분 손실은 (7.1 호출)

DRN-001이 가졌던 *formation_lead 5등급*과 *obstacle_avoidance 4등급*이 — 군집 전체에서 어느 정도 보존되는가:

```typeql
match
  $cap_lead isa formation_lead;
  let $d_remaining in drones_with_capability($cap_lead, 5);
  $d01 isa drone, has serial_id "DRN-001";
  not { $d_remaining is $d01; };
  $d_remaining has serial_id $sid;
fetch $sid;
```

답: DRN-002 (남은 formation_lead 5등급 보유자). *군집의 최고 등급 리더 능력은 유지됨*.

---

## 8.5 대체 드론 도출 — 세 함수의 합성

7장 함수들의 *교집합*으로 최종 답.

```typeql
match
  $m1 isa mission, has mission_id "MSN-2026-04-01-001";
  $d01 isa drone, has serial_id "DRN-001";
  
  # 7.2의 리더 후보
  let $cand in leader_candidates($m1);
  
  # 7.1의 능력 매칭 — formation_lead 5등급
  $cap_lead isa formation_lead;
  let $capable in drones_with_capability($cap_lead, 5);
  $cand is $capable;
  
  # 7.3의 통신 가능성 — DRN-009와의 통신 보장
  $d09 isa drone, has serial_id "DRN-009";
  let $reach in reachable_through_swarm($cand, $d01);
  $reach is $d09;
  
  $cand has serial_id $sid;
fetch $sid;
```

**세 함수의 교집합**:
- 리더 후보 (능력 4+ 비-리더 역할) ∩
- formation_lead 5등급 보유자 ∩
- 손실 드론 제외하고도 DRN-009와 통신 가능

답: **DRN-002**.

이게 — *한 사건이 입력된 직후*, *시스템이 분석가에게 던지는* 답이다. 분석가가 12대 드론·능력·통신을 *머릿속에 들고 있지 않아도* — 시스템이 즉시 후보를 짚어준다.

---

## 8.6 임무 재할당

분석가가 *DRN-002를 새 리더로 결정*했다고 하자. 데이터 갱신:

```typeql
transaction write drone_swarm
```

```typeql
match
  $d02 isa drone, has serial_id "DRN-002";
  $m1 isa mission, has mission_id "MSN-2026-04-01-001";
  $role_leader isa role_def, has role_type "leader";
  
  # DRN-002의 기존 역할(follower) 종료
  $old (assigned_drone: $d02, assignment_mission: $m1) isa assigned_to;

insert
  $old has assignment_end 2026-04-01T17:25:00;
  
  # DRN-002의 새 leader 역할
  (assigned_drone: $d02, assignment_mission: $m1, played_role: $role_leader)
    isa assigned_to,
    has assignment_start 2026-04-01T17:25:00;
commit;
```

**짚어둘 자리** — *기존 매듭을 삭제하지 않고 종료 시점만 박는다*. *역할 변경의 이력*이 데이터에 보존됨. 추후 *어느 드론이 언제부터 언제까지 어느 역할*이었는지를 *시간축으로 재구성*할 수 있음.

---

## 8.7 ◇ 이론 절 — Truth Maintenance

### 데이터 변경 시 추론의 일관성

새 데이터가 입력되면 — *기존 추론 결과*는 어떻게 되는가? 이게 *Truth Maintenance*의 영역.

예: `leader_candidates(m1)`을 *손실 직전*에 호출했다면 — DRN-002가 답에 있었을 것. 손실 *직후*에 호출해도 — 여전히 DRN-002. 그러나 *DRN-002에게 리더 역할을 할당한 후*에 다시 호출하면 — DRN-002가 *현재 리더이므로* 답에서 빠짐.

같은 함수가 *데이터 상태에 따라 다른 답*을 준다. TypeDB의 backward chaining + on-the-fly가 — 이 일관성을 자동으로 보장한다.

### Truth Maintenance Systems (TMS)

학계에서 *Truth Maintenance Systems*는 *추론된 사실의 의존성을 명시적으로 추적*하는 시스템.

- 사실 A로부터 B가 추론됨 → *B는 A에 의존*
- A가 부정되면 → B도 자동 부정 (또는 재계산)

TypeDB의 함수는 *명시적 TMS가 아니다*. 그러나 — *backward chaining이라서 매번 재계산*이므로, 사실상 *자동 truth maintenance*가 일어남.

명시적 TMS가 필요한 자리:
- 추론 비용이 매우 큰 경우 (재계산이 비실용적)
- 부분 갱신만 필요한 경우
- 추론 사슬을 *추적·설명*해야 할 경우 (XAI)

이 책의 8장 사례는 — *데이터 규모가 작아* on-the-fly로 충분.

### Materialization과의 트레이드오프

8.5의 합성 쿼리는 *세 함수를 호출*. 각각 fixpoint 계산. 12대 드론에서는 즉시지만 — *수백 대*로 확장되면 *materialization*이 필요해진다.

운영 시스템의 자리:
- *자주 묻는 함수*는 materialize
- *드물게 묻는 함수*는 on-the-fly
- *데이터 갱신 빈도*가 materialization의 trigger
- *consistency 요구*가 strategy 결정

---

## 8.8 시스템이 답하지 않는 자리 — 정직한 한계

도구를 미화하지 않는 약속이 이 장에서 가장 중요하다.

### 시스템이 답하는 것

- 어느 역할이 공백이 됐는가
- 어느 통신 링크가 끊겼는가
- 능력의 군집 차원 손실이 있는가
- 대체 후보는 누구인가
- 대체 후 데이터 일관성을 어떻게 유지하는가

### 시스템이 답하지 않는 것

**1. 실시간 비행 제약**
- 배터리 잔량 (DRN-002가 현재 30% 남았다면?)
- 풍속·기상 (산악 강풍에 견디는가?)
- 충돌 회피 (재배치 경로의 안전)
- 통신 지연 (재할당 명령이 도달할 시간)

**2. 안전 마진**
- 새 리더가 자기 자리에 도달할 시간 동안 군집이 견디는가
- 임무 중단 vs 재할당 결정의 *위험 평가*
- 조난자의 시간적 긴급성

**3. 인간 관제사의 판단**
- 시스템이 *후보*를 답하지만 — *최종 결정*은 사람의 자리
- 분석가의 *도메인 지식*과 *맥락 이해*가 시스템 위에 있음

**4. 법적·운용적 제약**
- 야간 비행 인가 (시각이 일몰에 가까운 자리)
- 비행 금지 구역
- 통신 주파수 규제
- 군 운용 시 *교전 규칙(ROE)*

**5. 신뢰도와 검증**
- 시스템의 답이 *원리적으로 옳다*는 것과 *실제 작동한다*는 것은 다르다
- *시뮬레이션 검증* + *제한된 환경에서의 시범* + *실전 적용*의 단계가 필요

### 시스템과 분석가의 협업

시스템은 *분석가·관제사의 결정을 대체하는 도구가 아니다*. 분석가가 *놓치고 있던 자리*를 자동으로 보여주는 도구다.

- 12대 드론·30개 능력 매듭·19개 링크를 *머릿속에 들고 있을 필요가 없다*
- *함수가 답한 후보*에서 *최종 판단*은 사람이 한다
- 시스템과 사람의 *분업*이 명확해질 때 — 둘 다 강점이 살아남

---

## 8.9 정리

이 장에서 손에 들어온 것:

**시나리오**
- `drone_loss` 관계 (Reified Event Pattern)
- 네 함수의 동시 호출로 영향 자동 도출
- 세 함수 합성으로 대체 후보 도출
- 시간축이 보존된 임무 재할당

**이론적 자리**
- Forward chaining vs Backward chaining
- Truth maintenance — 데이터 변경 시 추론 일관성
- Materialization 트레이드오프
- 시스템과 인간의 협업 자리

다음 장 — *9장 정리와 다음 발걸음*. 책 전체의 *네 가지 도구가 어떻게 회수되었는가*, 자매 책과의 비교, 그리고 *OWL·SHACL·Description Logic*으로의 다음 자리.
