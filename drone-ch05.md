# 제5장. 스키마 짓기

## 들어가며

4장에서 추출한 *6개 핵심 어휘*가 — 이 장에서 *TypeDB 스키마*로 옮겨진다. 그러나 단순한 *코드 번역*이 아니다. *Ontology Design Patterns*가 함께 들어오고, *모듈화*와 *스키마 진화*까지 짚는다.

이 장이 책에서 가장 길고 단단한 자리다.

---

## 5.1 Ontology Design Patterns (ODP)

스키마를 *맨바닥에서 짜지 않는다*. 학계가 정리한 *Ontology Design Patterns*가 있다.

### Content ODP — 자주 등장하는 도메인 패턴

**Participant Role Pattern**. 한 사건(event)에 *여러 참여자*가 *각자의 역할*로 참여하는 모양. — Davidson 사건 의미론의 ODP 버전. 5장의 4개 N항 관계가 모두 이 패턴.

**Time-Indexed Situation Pattern**. 같은 사실이 *시점에 따라 다른 값*을 가지는 모양. *드론이 시점에 따라 다른 역할을 한다*는 자리.

**Capability Pattern**. 행위자(agent)가 *능력*을 가지고, 능력에 *등급*이 매겨지는 모양. `has_capability` 관계의 토대.

### Logical ODP — 구조적 패턴

**Tree Hierarchy**. `entity X sub Y`의 계층. 5.2에서 사용.
**N-ary Relation**. `relation R, relates A, relates B, ...` 5.4에서 사용.
**Reified Event**. 사건 자체를 entity로 두는 패턴 (8장 `drone_loss`).

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

### 5.3 capability 분류

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

### 5.4 mission과 role

```typeql
  entity mission, 
    owns mission_id @card(1..1), 
    owns started_at @card(1..1);
  entity surveillance sub mission;
  entity search_and_rescue sub mission;
  entity delivery_mission sub mission, owns target_location @card(0..1);
  entity formation_demo sub mission;
  
  # 역할은 별도 entity로 — 데이터에서 다른 사건 사이에 공유 가능
  entity role_def, owns role_type @card(1..1);
  attribute role_type, value string 
    @values("leader", "observer", "relay", "follower", "rescuer");
  
  attribute mission_id, value string;
  attribute started_at, value datetime;
  attribute target_location, value string;
```

**짚어둘 자리** — *역할을 entity로 둔 결정*. 대안은 *attribute로만 두는 것* (`assigned_to`의 속성으로). 왜 entity로 했는가:

1. *같은 역할 정의*가 *여러 임무에서 재사용*됨
2. *역할 자체에 메타데이터*(요구 능력, 책임 범위)를 부여할 수 있음
3. *역할의 분류 가지*가 미래에 자라날 수 있음 (`leader` → `formation_leader` / `mission_leader`)

이게 *3.10 정리*에서 짚은 *분류·관계·역할의 균형*이 도메인에서 결정되는 모양.

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

- 4자리 + 시점 — *한 드론이 한 임무의 한 역할에 언제부터 언제까지*
- 역할 이름(`played_role`)이 entity 이름(`role_def`)과 분리됨 — 가독성 + 명명 충돌 회피
- 같은 드론이 *다른 시점에 다른 역할*에 묶일 수 있음

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

- *같은 자리에 두 드론* — 결혼 관계와 같은 모양 (2장 참조)
- 링크 품질 0~1
- 시점 박힘 — *언제 형성된 링크*가 분석 가능

### 5.5.3 `has_capability` — 드론·능력·등급

Capability Pattern의 직접 구현.

```typeql
  relation has_capability,
    relates capable_drone @card(1..1),
    relates capability_type @card(1..1),
    owns capability_grade @card(0..1);
  
  drone plays has_capability:capable_drone;
  capability plays has_capability:capability_type;
  
  attribute capability_grade, value long @range(1..5);
```

- *능력 여부*만이 아니라 *등급(1~5)*까지
- 임무가 요구하는 *최소 등급*과 매칭 가능

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

- *같은 자리에 N명* — 편대의 모든 드론이 member role
- 편대 형태가 `@values`로 강제
- formation 자체는 *entity가 아니라 attribute value* — 4.5에서 짚은 *어휘가 attribute 자리에 떨어진 경우*

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

## 5.8 스키마 검증 — 잘못된 데이터 거부 시연

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

## 5.9 ◇ 이론 절 — 스키마 진화와 모듈화

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

---

## 5.10 정리

이 장에서 손에 들어온 것:

**스키마 코드**
- 비행체·페이로드·능력·임무·역할의 entity 트리
- 4개 핵심 N항 관계
- 8개의 카디널리티·범위·값 강제

**이론적 자리**
- Ontology Design Patterns — Participant Role, Time-Indexed, Capability
- 6개 어휘의 *entity·관계·속성 자리* 결정 기준
- 스키마 진화 — 비파괴·호환·파괴 변경
- 모듈화와 Schema-as-Code

다음 장 — **데이터 채우기**. 5장의 빈 스키마에 *12대 드론 + 1개 임무 + 18개 통신 링크*가 들어온다. *A-Box vs T-Box*의 자리.
