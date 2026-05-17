# 제5장. 스키마 짓기

## 들어가며

4장에서 추출한 *6개 핵심 어휘*가 — 이 장에서 *TypeDB 스키마*로 옮겨진다. 그러나 단순한 *코드 번역*이 아니다. *각 설계 결정의 트레이드오프*를 명시적으로 토론하면서, *Ontology Design Patterns*를 의식적으로 적용하고, *잘못된 스키마*와 *좋은 스키마*의 차이까지 본다.

이 장이 책에서 가장 길고 단단한 자리다.

---

## 5.1 Ontology Design Patterns (ODP)

스키마를 *맨바닥에서 짜지 않는다*. 학계가 정리한 *Ontology Design Patterns*가 있다 (ontologydesignpatterns.org).

### Content ODP — 자주 등장하는 도메인 패턴

**Participant Role Pattern**. 한 사건(event)에 *여러 참여자*가 *각자의 역할*로 참여하는 모양 — Davidson 사건 의미론의 ODP 버전. 5장의 4개 N항 관계가 모두 이 패턴.

**Time-Indexed Situation Pattern**. 같은 사실이 *시점에 따라 다른 값*을 가지는 모양. *드론이 시점에 따라 다른 역할을 한다*는 자리.

**Capability Pattern**. 행위자(agent)가 *능력*을 가지고, 능력에 *등급*이 매겨지는 모양. `has_capability` 관계의 토대.

**Trajectory Pattern**. 객체가 *시공간상의 경로*를 따르는 모양. 이 책은 단순화로 사용하지 않지만 — 본격 시스템에서는 필수.

### Logical ODP — 구조적 패턴

**Tree Hierarchy**. `entity X sub Y`의 계층. 5.2에서 사용.
**N-ary Relation**. `relation R, relates A, relates B, ...` 5.5에서 사용.
**Reified Event**. 사건 자체를 entity로 두는 패턴 (8장 `drone_loss`).
**Identifier Pattern**. `@card(1..1)`과 `@regex`로 *고유 식별자* 강제.

이 ODP들을 *명시적으로 의식하고* 짜는 것이 — 정통 온톨로지 엔지니어링의 핵심.

---

## 5.2 entity 트리

```typeql
transaction schema drone_swarm
```

```typeql
define
  # === 비행체 가지 ===
  entity vehicle, 
    owns model_name @card(1..1), 
    owns manufacturer @card(0..1);
  entity drone sub vehicle, 
    owns serial_id @card(1..1), 
    owns max_flight_time_min @card(0..1);
  entity quadcopter sub drone;
  entity fixed_wing sub drone;
  entity hybrid sub drone;
  
  entity ground_station sub vehicle, owns station_id @card(1..1);
  
  # === 페이로드 가지 ===
  entity payload, owns payload_id @card(1..1);
  entity sensor sub payload;
  entity camera sub sensor, owns resolution @card(0..1);
  entity lidar sub sensor;
  entity thermal_camera sub sensor;
  entity delivery_payload sub payload, owns max_weight_kg @card(0..1);
  
  attribute model_name, value string;
  attribute manufacturer, value string;
  attribute serial_id, value string;
  attribute max_flight_time_min, value long @range(1..600);
  attribute station_id, value string;
  attribute payload_id, value string;
  attribute resolution, value string @values("HD", "4K", "8K");
  attribute max_weight_kg, value double @range(0.0..50.0);
```

### 5.2의 ◇ 설계 결정 — 왜 이 구조인가

**Q1. vehicle 위에 더 상위(예: physical_object)를 두지 않은 이유**
- 도메인이 *드론과 지상 관제소*로 한정 — 더 상위는 *재사용 가능성 없음*
- 상위 온톨로지(SUMO·DOLCE)와 매핑하려면 *external mapping*으로 처리

**Q2. drone과 ground_station이 vehicle을 공유하는 이유**
- 둘 다 *model_name + manufacturer*를 가짐
- 미래에 *공통 관계*(가령 운용자가 어느 vehicle에 책임이 있는가)가 자라날 자리
- 상속의 *재사용*보다 *공통 관계 자리* 확보가 더 큰 가치

**Q3. payload를 drone과 별도 트리로 둔 이유**
- payload는 *교환 가능*. 한 드론이 *오늘 카메라, 내일 라이다*를 장착
- entity로 두면 *동적 장착 관계*(`mounts`)를 별도 N항으로 풀 수 있음
- payload를 drone의 *속성*으로 두면 — 교환 시점 추적이 어색해짐

---

## 5.3 capability 분류

```typeql
  entity capability, owns capability_name @card(1..1);
  entity sensing_capability sub capability;
  entity night_optical sub sensing_capability;
  entity thermal_imaging sub sensing_capability;
  entity lidar_mapping sub sensing_capability;
  
  entity communication_capability sub capability;
  entity mesh_relay sub communication_capability;
  entity long_range_comm sub communication_capability;
  
  entity autonomy_capability sub capability;
  entity formation_lead sub autonomy_capability;
  entity obstacle_avoidance sub autonomy_capability;
  
  entity propulsion_capability sub capability;
  entity high_altitude sub propulsion_capability;
  entity hover_long sub propulsion_capability;
  
  attribute capability_name, value string;
```

### 5.3의 ◇ 설계 결정 — capability를 entity로

**대안 1: 속성으로만**
```typeql
drone owns capabilities_string;
# 데이터: "night_optical;mesh_relay;formation_lead"
```
- 단점: 분류 트리 없음. 등급 표현 어려움. 검색 비정상.

**대안 2: 단일 속성에 @values**
```typeql
drone owns capability_value @card(0..);
attribute capability_value, value string @values("night_optical", ...);
```
- 장점: 검색은 가능. 단점: 분류 가지 없음. 새 능력 추가 시 *@values 확장* 필요.

**채택: entity 분류 트리 + has_capability N항 관계**
- 장점:
  - 분류 가지(sensing / communication / autonomy / propulsion) 자유
  - 등급을 *관계 속성*으로 자연스럽게 추가
  - 능력 자체에 *메타데이터*(요구 사양, 인증 정보 등)를 미래에 추가 가능
- 단점: 약간 복잡 (entity + relation 둘 다 정의)

**결정**: 장기 확장성에서 entity 트리가 압도적.

---

## 5.4 mission과 role

```typeql
  entity mission, 
    owns mission_id @card(1..1), 
    owns started_at @card(1..1);
  entity surveillance sub mission;
  entity search_and_rescue sub mission;
  entity delivery_mission sub mission, owns target_location @card(0..1);
  entity formation_demo sub mission;
  
  entity role_def, owns role_type @card(1..1);
  attribute role_type, value string 
    @values("leader", "observer", "relay", "follower", "rescuer");
  
  attribute mission_id, value string;
  attribute started_at, value datetime;
  attribute target_location, value string;
```

### 5.4의 ◇ 설계 결정 — mission은 왜 entity? role은 왜 별도 entity?

#### Mission을 entity로 둔 이유

**대안: relation으로**
```typeql
relation mission,
  relates participating_drone @card(1..),
  relates assigned_role @card(1..),
  ...;
```
- 단점: *임무 자체*에 대한 메타데이터 (시작 시점, 종료, 우선순위, 클라이언트 등)가 어색해짐
- 단점: 분류 가지 (surveillance / delivery / search_and_rescue)를 표현하기 어려움
- 단점: *임무 인스턴스*가 *여러 다른 N항 관계*에 참여하는데 (assigned_to, mission_priority_change 등) — relation의 relation은 다중 reification

**채택: entity로**
- 임무는 *그 자체로 식별 가능한 사건/주체*
- 분류 가지를 자연스럽게 표현
- 다른 관계가 mission을 *참여자*로 다룰 수 있음

#### role_def를 별도 entity로 둔 이유

**대안 1: assigned_to의 속성으로만**
```typeql
relation assigned_to,
  relates drone, relates mission,
  owns role_type;  # 속성으로
```
- 단점: 역할 자체에 *메타데이터*(요구 능력, 책임)를 추가하기 어려움
- 단점: 역할 간 *관계*(예: leader는 follower의 superior)를 표현 불가

**대안 2: role을 entity로 두되 role_type 없이 클래스로**
```typeql
entity role;
entity leader sub role;
entity observer sub role;
...
```
- 단점: 새 역할 추가 시 *스키마 변경* 필요 (entity 추가)
- 단점: 역할 명에 *유한 집합 제약* 없음

**채택: entity + role_type @values**
- 장점: 역할 자체가 *재사용 가능한 객체*
- 장점: role_type에 `@values` 강제로 *5가지 역할만* 허용
- 장점: 데이터 입력 시 역할 인스턴스를 *공유* (DRN-001과 DRN-002가 같은 leader role 인스턴스에 묶일 수도)

**미래 확장**:
```typeql
# 역할의 요구 능력 (미래 확장)
relation role_requires_capability,
  relates required_role_def @card(1..1),
  relates required_cap @card(1..1),
  owns min_grade @card(0..1);
```
이런 *역할의 요구사항*을 별도 매듭으로 풀 수 있는 자리.

---

## 5.5 핵심 N항 관계 네 개

이 책 2부의 *결정적 자리*. 4개 N항 관계가 *드론 군집비행의 모양*을 데이터 모델로 담는다.

### 5.5.1 `assigned_to` — 드론·임무·역할·시점

*Participant Role Pattern + Time-Indexed*의 결합.

```typeql
  relation assigned_to,
    relates assigned_drone @card(1..1),
    relates assignment_mission @card(1..1),
    relates played_role @card(1..1),
    owns assignment_start @card(1..1),
    owns assignment_end @card(0..1);
  
  drone plays assigned_to:assigned_drone;
  mission plays assigned_to:assignment_mission;
  role_def plays assigned_to:played_role;
  
  attribute assignment_start, value datetime;
  attribute assignment_end, value datetime;
```

#### ◇ 설계 결정 — 시간을 어떻게 다루는가

**대안 1: 시점 없이 현재 상태만**
```typeql
relation assigned_to,
  relates drone, relates mission, relates role;  # 시점 없음
```
- 단점: 역할 변경 이력 손실. 4장 시간축의 *7단계*를 표현 불가.

**대안 2: 별도 history entity**
```typeql
entity assignment_event, owns event_time, owns event_type;
```
- 장점: 모든 변경이 *이벤트로*. 시간 추적 정밀.
- 단점: 단순 *현재 상태* 조회가 복잡. 매번 *최신 이벤트*를 찾아야 함.

**채택: start/end 시점이 박힌 관계**
- 장점: *현재 상태*는 `end가 없는 인스턴스*. *과거 상태*는 `end가 있는 인스턴스*.
- 장점: 시간축의 *연속*이 자연스럽게 표현
- 트레이드오프: 변경 시 *기존 인스턴스에 end 박고 새 인스턴스 insert* — 약간 번잡하지만 명확

### 5.5.2 `communicates_with` — 드론·드론·링크 품질·시점

```typeql
  relation communicates_with,
    relates link_endpoint @card(2..2),
    owns link_quality @card(0..1),
    owns link_established_at @card(1..1);
  
  drone plays communicates_with:link_endpoint;
  
  attribute link_quality, value double @range(0.0..1.0);
  attribute link_established_at, value datetime;
```

#### ◇ 설계 결정 — 대칭적 관계의 모양

**핵심 자리**: *같은 자리에 두 드론*. `relates link_endpoint @card(2..2)`.

**대안: 비대칭 (sender / receiver)**
```typeql
relation communicates_with,
  relates link_sender, relates link_receiver;
```
- 단점: 메시 네트워크의 *대칭 통신*을 비대칭으로 표현 — 부자연스러움
- 단점: A→B와 B→A를 *두 인스턴스*로 적어야

**채택: 대칭 (link_endpoint 자리에 두 드론)**
- 장점: *메시 통신*의 본질에 자연스러움 (어느 쪽도 우월하지 않음)
- 장점: 한 인스턴스로 *양방향* 표현

**비교: 결혼 관계와 같은 모양**
- 2장의 `marriage(spouse, spouse, ...)`와 정확히 같은 패턴
- *N항 관계의 대칭 자리* 패턴이 도메인 간 재사용

### 5.5.3 `has_capability` — 드론·능력·등급

```typeql
  relation has_capability,
    relates capable_drone @card(1..1),
    relates capability_type @card(1..1),
    owns capability_grade @card(0..1);
  
  drone plays has_capability:capable_drone;
  capability plays has_capability:capability_type;
  
  attribute capability_grade, value long @range(1..5);
```

#### ◇ 설계 결정 — 등급을 어떻게 표현하는가

- *없다/있다* (0..1 bool 등급)
- *정수 등급* (1~5) ← **채택**
- *연속 점수* (0.0~1.0 double)
- *복합 점수* (여러 차원의 점수 벡터)

**정수 등급의 선택 이유**:
- 임계 매칭 자연스러움 (`$g >= 4`)
- 인증·테스트 결과와 자연스러운 매핑 (Level 1~5)
- 연속 점수의 *false precision* 문제 회피

### 5.5.4 `flies_in_formation` — 드론들·편대 형태·시점

```typeql
  relation flies_in_formation,
    relates formation_member @card(2..),
    owns formation_type @card(1..1),
    owns formation_started_at @card(1..1);
  
  drone plays flies_in_formation:formation_member;
  
  attribute formation_type, value string 
    @values("v_shape", "circle", "line", "diamond", "echelon");
  attribute formation_started_at, value datetime;

commit;
```

#### ◇ 설계 결정 — formation을 attribute로

**대안: formation을 entity로**
```typeql
entity formation_def, owns formation_type;
entity v_shape sub formation_def;
entity circle sub formation_def;
...
relation flies_in_formation,
  relates formation_member,
  relates formation_used;  # entity 자리
```
- 장점: 편대 자체에 *메타데이터*(권장 드론 수, 권장 역할 배치) 추가 가능
- 단점: 5가지 편대 외 *확장 가능성*이 좁다면 과대 설계

**채택: attribute @values**
- 장점: 단순. 5가지 편대만 허용됨이 명시적
- 단점: 편대별 메타데이터 표현 어려움
- 미래 확장: 필요 시 entity로 *진화 가능* (5.10의 schema evolution)

---

## 5.6 4장 6개 축의 스키마 매핑

| 4장의 어휘 | 5장 스키마의 자리 |
|---|---|
| drone | entity 트리 (`vehicle → drone → quadcopter/fixed_wing/hybrid`) |
| mission | entity 트리 (`mission → search_and_rescue / delivery / ...`) |
| capability | entity 트리 + `has_capability` 관계 |
| role | `role_def` entity + `assigned_to` 관계의 `played_role` 자리 |
| link | `communicates_with` 관계 |
| formation | `flies_in_formation` 관계의 `@values` 속성 |

*같은 6개 어휘가 entity·관계·속성의 세 자리로 분산*. 도메인의 *의미*가 *데이터 모델의 어느 자리에서 표현되는가*는 — *작동 방식*이 결정한다.

- *재사용·분류 가지·메타데이터가 필요하면* → entity
- *N개 개체를 묶고 사건의 속성이 있으면* → relation
- *유한 집합의 라벨이면* → @values attribute

---

## 5.7 카디널리티·범위·값 강제 — 정리

5장에서 박은 강제들:

| 강제 | 어디 | 도메인 규칙 |
|---|---|---|
| `@card(2..2)` | link_endpoint | 통신 링크는 정확히 두 드론 |
| `@card(2..)` | formation_member | 편대는 두 드론 이상 |
| `@range(0.0..1.0)` | link_quality | 링크 품질은 0~1 |
| `@range(1..5)` | capability_grade | 능력 등급은 1~5 |
| `@range(1..600)` | max_flight_time_min | 비행시간 1~600분 |
| `@values("leader", ...)` | role_type | 5가지 역할만 |
| `@values("v_shape", ...)` | formation_type | 5가지 편대만 |
| `@values("HD","4K","8K")` | resolution | 카메라 해상도 3종 |

도메인 사고가 *코드의 if*가 아니라 *스키마*에 박혔다.

---

## 5.8 잘못된 스키마 vs 좋은 스키마

같은 도메인을 *세 가지 잘못된 방식*과 *현재 책의 좋은 방식*으로 비교.

### 잘못된 스키마 1 — 모든 것을 attribute로

```typeql
define
  entity drone,
    owns serial,
    owns capabilities,        # "night_optical;mesh_relay" 같은 문자열
    owns current_mission,
    owns current_role,
    owns linked_drones;       # "DRN-002,DRN-009" 같은 문자열
```

**왜 나쁜가**:
- *분류 없음* — 능력의 다층 구조 불가능
- *N항 관계 없음* — 시점·등급·품질 같은 매듭 속성 손실
- *역할 자리 없음* — 같은 드론의 *여러 역할*을 표현 못함
- *링크된 드론*이 문자열 — 데이터베이스가 *그 문자열이 실제 드론인지* 검증 못함
- 검색은 가능하나 *의미가 코드와 데이터에 흩어짐*

### 잘못된 스키마 2 — 이진 관계로 쪼개기

```typeql
define
  entity drone, owns serial;
  entity mission, owns mission_id;
  
  relation drone_in_mission,
    relates the_drone, relates the_mission;
  
  relation drone_has_role,
    relates the_drone, relates the_role;
  
  entity role, owns role_name;
```

**왜 나쁜가**:
- *드론 + 임무 + 역할*이 *세 개의 이진 관계*로 흩어짐
- 한 사건 (DRN-001이 mission M1의 leader)을 표현하려면 *두 관계*에 데이터를 입력 — *원자성*이 깨짐
- *시점*을 박을 자리가 어색 (어느 관계에?)
- 데이터 무결성 — drone_in_mission과 drone_has_role의 정합성을 *외부 검증*해야

### 잘못된 스키마 3 — 분류 없는 평면

```typeql
define
  entity thing, owns type, owns name;
  # 모든 것이 thing — drone도, capability도, mission도
  
  relation related_to,
    relates source, relates target,
    owns relation_type;
  # 모든 관계가 related_to
```

**왜 나쁜가**:
- *RDF/OWL의 단순화*를 잘못 적용한 모양
- 스키마가 *도메인 의미*를 표현하지 않음
- 모든 쿼리가 `type == "drone"` 같은 *런타임 필터*에 의존
- 카디널리티·범위 강제 불가능 (모든 관계가 same type)

### 좋은 스키마 — 현재 책의 스키마

5.2~5.5에서 짠 스키마는:
- *4단 entity 트리* (vehicle, payload, capability, mission, role_def)
- *4개 N항 관계* (assigned_to, communicates_with, has_capability, flies_in_formation)
- *8개 강제* (@card, @range, @values)
- *6개 도메인 어휘*가 *entity·관계·속성*의 세 자리로 적절히 분산

**결과**: 도메인의 *모든 의미*가 스키마에 명시적으로 박혔다. 6개월 뒤 다른 분석가가 스키마를 처음 봐도 — *이 도메인이 어떻게 작동하는가*가 보인다.

---

## 5.9 스키마 검증 시연

세 가지 시연으로 *스키마가 자기 자신을 어떻게 지키는가*를 본다.

### 시연 1 — 링크 품질 1.5

```typeql
insert
  $bad_link (link_endpoint: $d1, link_endpoint: $d2) isa communicates_with,
    has link_quality 1.5,
    has link_established_at 2026-04-01T16:00:00;
```

오류: `Range violation. Attribute 'link_quality' value 1.5 outside range [0.0, 1.0].`

### 시연 2 — 세 드론이 한 link에

```typeql
insert
  $bad_link (link_endpoint: $d1, link_endpoint: $d2, link_endpoint: $d3) 
    isa communicates_with, ...;
```

오류: `Cardinality violation. Relation 'communicates_with' has role 'link_endpoint' with cardinality 3, exceeds maximum 2.`

### 시연 3 — 역할이 "captain"

```typeql
insert
  $bad_role isa role_def, has role_type "captain";
```

오류: `Values violation. Attribute 'role_type' value 'captain' not in allowed set.`

---

## 5.10 ◇ 이론 절 — 스키마 진화와 모듈화

### 운영 중 스키마 변경

온톨로지는 *살아 있는 문서*다. 도메인이 변하면 스키마도 변한다. *스키마 진화(schema evolution)*는 — 운영 중 데이터를 보존하면서 스키마를 변경하는 작업.

세 가지 변경 유형:

**비파괴적 변경 (Non-breaking)**
- 새 entity 가지 추가
- 새 optional 속성 추가 (`@card(0..1)`)
- 새 relation 추가
- → 기존 데이터·쿼리에 영향 없음

**호환 변경 (Compatible)**
- `@card` 완화 (1..1 → 0..1, 1..1 → 1..)
- `@range` 확장 (0..100 → 0..1000)
- `@values` 확장 (값 추가)
- → 기존 데이터는 그대로, 새 데이터에만 영향

**파괴적 변경 (Breaking)**
- entity 삭제 또는 이름 변경
- relation role 이름 변경
- `@card` 강화 (1.. → 1..1)
- `@range` 축소
- `@values` 축소
- → *기존 데이터 마이그레이션* 필요

TypeDB는 *비파괴적 변경*은 안전하게 처리. *파괴적 변경*은 *명시적 마이그레이션 스크립트*가 필요.

이 책의 5장 스키마는 — *비파괴 확장 가능*하게 짜였다. 새 드론 유형(`octocopter`), 새 능력(`underwater_operation`), 새 역할(`spotter`) 등이 *기존 코드를 깨지 않고* 추가 가능.

### 모듈화

큰 온톨로지는 *여러 모듈*로 나뉜다. 이 책 스키마의 자연스러운 모듈:

```
core/        — vehicle, payload, attribute 기본
capability/  — capability 트리
mission/     — mission, role_def
relations/   — assigned_to, communicates_with, has_capability, flies_in_formation
events/      — drone_loss (8장에서 추가)
```

TypeDB는 *물리적 모듈 분리*는 지원하지 않지만 — *논리적 모듈*은 `define` 블록의 구조로 표현. 큰 프로젝트에서는 *모듈별 define 파일*을 두고 빌드 시 합치는 패턴.

### Schema-as-Code

스키마 자체가 *코드*이므로 — 일반 코드와 같은 도구를 적용:
- 버전 관리 (Git)
- 코드 리뷰
- CI/CD (스키마 변경의 자동 테스트)
- 문서화 (스키마에서 ERD 자동 생성)

운영 환경에서 이 자리가 *온톨로지 거버넌스*의 본질.

### 스키마 진화의 실전 예시

가령 *능력 시스템에 인증 정보*를 추가하고 싶다고 하자.

**비파괴 추가**:
```typeql
define
  has_capability,
    owns certification_date @card(0..1),
    owns certified_by @card(0..1);
  
  attribute certification_date, value date;
  attribute certified_by, value string;
```

기존 `has_capability` 데이터는 *그대로* — 새 속성이 *옵션*이라 영향 없음. 새 데이터부터 *인증 정보 포함 가능*.

**파괴적 변경 예시**:
```typeql
# capability_grade를 long(1~5)에서 double(0~1)로 변경
# → 모든 기존 데이터를 마이그레이션해야
```

마이그레이션 스크립트:
```typeql
# 1. 기존 grade를 임시 attribute로 백업
# 2. 새 attribute로 변환 (grade/5.0 → 0~1)
# 3. 기존 attribute 삭제
# 4. 새 attribute로 데이터 재입력
```

이런 운영 부담 때문에 — *처음 스키마를 짤 때* 미래 확장을 고려한 *비파괴 확장 가능 설계*가 가치 있다.

---

## 5.11 정리

이 장에서 손에 들어온 것:

**스키마 코드**
- 비행체·페이로드·능력·임무·역할의 entity 트리
- 4개 핵심 N항 관계
- 8개의 카디널리티·범위·값 강제

**설계 결정의 토론** (이 장의 결정적 추가)
- vehicle 트리 구조, capability를 entity로 둔 이유
- mission entity vs relation의 선택
- role_def 별도 entity의 가치
- 시간축의 표현 (start/end 시점)
- 대칭 관계(link_endpoint)의 모양
- 등급의 정수 vs 연속의 선택
- formation의 attribute @values 채택

**잘못된 스키마와의 비교**
- 모든 것을 attribute로 (의미 손실)
- 이진 관계로 쪼개기 (원자성 깨짐)
- 분류 없는 평면 (모든 검색이 런타임 필터)

**이론적 자리**
- Ontology Design Patterns (4가지)
- 스키마 진화 — 비파괴·호환·파괴 변경
- 모듈화와 Schema-as-Code

다음 장 — **데이터 채우기**. 5장의 빈 스키마에 *12대 드론 + 1개 임무 + 19개 통신 링크*가 들어온다. *A-Box vs T-Box*의 자리.
