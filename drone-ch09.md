# 제9장. 정리 — 네 도구의 회수, 그리고 다음 자리

## 들어가며

8장 시나리오에서 한 사건이 자동으로 풀려나갔다. 그 자동화의 안쪽에는 — 1부에서 손에 익힌 *네 가지 도구*가 있었다. 이 장은 그 네 도구가 *드론 도메인에서 어떻게 회수되었는가*를 짚고, *Competency Questions 12개의 답*을 정리하고, 자매 책과의 비교, *더 깊은 자리*까지 다룬다.

---

## 9.1 네 도구의 회수

### 분류 (entity 상속)

- 1부 예제 1에서 *도서관*에서 처음 익힘
- 2부에서 *vehicle → drone → quadcopter/fixed_wing/hybrid*로 확장
- *capability* 트리, *mission* 트리, *payload* 트리, *role_def* — 네 개의 분류 트리가 동시에
- 8.5 대체 드론 도출에서 — *formation_lead 능력 가지*가 검색 단위가 됨
- **이론적 자리**: Subtyping의 LSP, 분류학 vs 분류 이론

### N항 관계 (relation)

- 1부 예제 2에서 *결혼·부모*에서 처음 익힘
- 2부에서 *4개의 핵심 N항 관계*로 확장
  - `assigned_to` (4자리)
  - `communicates_with` (드론 두 자리 + 품질·시점)
  - `has_capability` (드론·능력·등급)
  - `flies_in_formation` (드론들 + 편대·시점)
- 8.1에서 *drone_loss* 관계가 *다른 관계 인스턴스*를 자기 자리에 받음 — 메타적 N항
- **이론적 자리**: Davidson의 사건 의미론

### 역할 (role, plays)

- 1부 예제 2에서 *같은 사람의 여러 자리*로 익힘
- 2부에서 *같은 드론이 leader / observer / relay / rescuer 자리*에 시점별로 다르게 묶임
- 8.6 임무 재할당에서 — DRN-002가 *follower 자리에서 leader 자리로* 이동, *이력은 보존*
- **이론적 자리**: 역할 vs 속성의 결정 기준

### 함수 (Function)

- 1부 예제 3에서 *재귀*가 손에 잡힘
- 2부 7장에서 *네 함수*로 펼쳐짐
- 8.5에서 *세 함수의 합성*으로 *대체 드론 도출*이 완성됨
- 같은 *기저 + 재귀 + or* 패턴이 *공급망 추적*(자매 책)과 *통신 메시 탐색*(이 책)에 동일하게 작동
- **이론적 자리**: Datalog, Fixpoint 의미론, Stratified negation

---

## 9.2 Competency Questions 12개의 회수

4장에서 던진 12개 CQ — *온톨로지가 답할 수 있어야 하는 질문들* — 이 어디서 답해졌는가:

| CQ # | 질문 | 답한 자리 |
|---|---|---|
| 1 | 야간 광학 가능 드론은? | 7.1 `drones_with_capability` |
| 2 | 등급 4+ 통신 중계 드론 수는? | 7.1 + `count` 집계 |
| 3 | 임무에 부족한 능력은? | 7.1 보완 + 임무 요구 명세 (미완) |
| 4 | 현재 임무 leader는? | 7.4 `drones_with_role(m, "leader")` |
| 5 | 한 드론이 동시에 여러 역할? | 2.5 통합 쿼리 패턴 |
| 6 | 단계별 역할 변경 이력? | 8.6 — assignment_start/end 시간축 |
| 7 | 두 드론 직접 통신 링크? | 6.7 Check 4 |
| 8 | 메시 네트워크 도달 가능? | 7.3 `reachable_through_swarm` |
| 9 | 품질 0.7+ 경로만? | 7.3 변형 `reachable_with_quality` |
| 10 | X 손실 시 영향 임무? | 8.4 Q1 |
| 11 | X 역할 대체 후보? | 7.2 + 8.5 |
| 12 | 대체 후 능력 합 충족? | 8.4 Q4 + 임무 요구 매칭 |

**12개 중 10개가 명시적으로 답해졌다.** CQ 3과 12는 *임무 요구사항 명세*가 추가 필요한 자리 — 이 책의 단순화 범위 밖. 실전 시스템에서는 *임무 entity가 owns required_capabilities @card(1..)*  같은 자리에서 보강.

10/12 = 83% 커버리지. 정직한 짚음:
- CQ 기반 검증은 *온톨로지 품질의 정량 지표*
- 100% 답하지 못한 자리가 *시스템의 한계의 명시*
- 다음 버전의 *명확한 작업 목록*

---

## 9.3 자매 책과의 비교 — 같은 도구, 다른 도메인

자매 책 *광전자에서 시작된 한 권의 책*과 이 책이 *같은 네 가지 도구*를 *다른 두 도메인*에 적용한 자리를 표로 정리.

| 자리 | 자매 책 (NVIDIA 반도체) | 이 책 (드론 군집) |
|---|---|---|
| **분류 트리** | 6개 층 (L1 AI칩~L6 인접영역) | 비행체·페이로드·능력·임무·역할 |
| **N항 관계의 정수** | `supplies` (공급자·수요자·제품·시점) | `assigned_to` (드론·임무·역할·시점) |
| **다중 자리** | `technology_endorsement` (announcer·tech·beneficiary들) | `flies_in_formation` (편대 + 멤버들) |
| **재귀 그래프** | `all_suppliers_of` (공급망 N단계) | `reachable_through_swarm` (메시 N단계) |
| **부정 신호 전파** | `supply_disruption` (공급 차질 → 고객 영향) | `drone_loss` (드론 손실 → 임무·링크 영향) |
| **합성 분석** | 시차 알파 (가중치+시점+후속노드) | 대체 드론 도출 (능력+후보+통신) |
| **클라이맥스** | 광반도체 발언 → 후속 노드 자동 도출 | 드론 손실 → 영향 자동 도출 |
| **호흡** | 회상 에세이 | 정통 기술서 |

이 표가 말하는 한 가지:

> **도구는 도메인에 종속되지 않는다.** 같은 *네 가지 도구*가 반도체 밸류체인에서도, 드론 군집비행에서도, 그리고 — 의료·법률·생명과학에서도 — 같은 모양으로 작동한다.

도메인이 *분류·매듭·자리·시점*을 동시에 요구하는 자리라면 — *TypeDB가 자연스러운 자리*다.

---

## 9.4 ◇ 이론 절 — 다음 자리

이 책 뒤에서 갈 수 있는 깊은 자리들.

### OWL과 Description Logic

이 책은 *TypeDB라는 한 도구*를 집중적으로 다뤘다. 그러나 — *온톨로지의 형식적 토대*는 더 넓다.

**OWL (Web Ontology Language)**. W3C의 표준. *Description Logic*에 기반. 표현력이 매우 높지만 — *결정 가능성(decidability)*과의 트레이드오프가 항상 작동.

OWL 2의 세 가지 profile:
- **OWL 2 EL** — Existential Language. 계산 효율적, 의료 온톨로지(SNOMED CT)에 사용
- **OWL 2 QL** — Query Language. 관계형 DB 매핑에 친화적
- **OWL 2 RL** — Rule Language. 규칙 기반 추론에 친화적

TypeDB와의 비교:
- OWL은 *추론 풍부*하지만 *학습 곡선 가파름*
- TypeDB는 *추론을 일급 도구화*하면서 *데이터베이스 운영 직관*에 가까움
- 둘 다 *온톨로지의 다른 표현*

깊이 들어가려면: Baader, Calvanese, McGuinness, Nardi, Patel-Schneider의 *Description Logic Handbook*이 표준.

### SHACL — 데이터 형태 제약

*Shapes Constraint Language*. RDF 데이터의 *형태 제약*을 표현하는 W3C 표준.

OWL이 *추론*을 위해 *Open-World*를 채택했다면, SHACL은 *데이터 검증*을 위해 *Closed-World*를 채택. TypeDB의 `@card`·`@range`·`@values`가 — SHACL의 *core constraints*와 같은 자리.

### 추론기(Reasoner)

OWL 추론기(HermiT, Pellet, ELK)는 *데이터 + 스키마*에서 *모든 도출 가능한 사실*을 자동으로 계산. TypeDB의 *backward chaining*이 같은 일을 *호출 시점에* 한다.

연구 흐름:
- 분산 추론 (대규모 그래프)
- 점진적 추론 (incremental reasoning)
- 설명 가능한 추론 (XAI)

### Modal Logic — 시점과 가능성

이 책은 *시점*을 *attribute*로 다뤘다 (`assignment_start`, `assignment_end`). 더 형식적인 자리에는 *시제 논리(temporal logic)*, *모달 논리(modal logic)*가 있다.

- LTL (Linear Temporal Logic) — 시간의 흐름에 대한 명제
- CTL (Computation Tree Logic) — 가능한 미래들에 대한 명제
- Epistemic Logic — *누가 무엇을 아는가*에 대한 논리

다중 UAV에서는 — *각 드론이 자기 시점의 상태*만 알고 *전역적 상태*를 모를 수 있는 자리에 *분산 인식*이 필요. 학술 깊이가 매우 깊은 자리.

### 학습 자료

이 책 뒤로 갈 수 있는 자료들:

**TypeDB**
- 공식 문서: typedb.com/docs
- TypeDB 학습 자료: typedb.com/learn

**온톨로지 일반**
- Tom Gruber, *What is an Ontology?* (1993)
- Nicola Guarino, *Formal Ontology in Information Systems* (1998)
- Asunción Gómez-Pérez 외, *Ontological Engineering* (2004)

**Description Logic**
- Baader 외, *The Description Logic Handbook* (2003)
- Krötzsch, Simancik, Horrocks, *A Description Logic Primer* (2012)

**Datalog와 재귀 쿼리**
- Abiteboul, Hull, Vianu, *Foundations of Databases* (1995) — 5장이 Datalog
- Green 외, *Datalog and Recursive Query Processing* (2013, Foundations and Trends)

**Knowledge Graphs**
- Hogan 외, *Knowledge Graphs* (2021, ACM Computing Surveys)
- 산업 사례: Google Knowledge Graph, Amazon Product Graph, Facebook Social Graph

---

## 9.5 다음 발걸음 — 실전 확장

### 데이터 규모

- 12대 → 250대 (DARPA OFFSET 규모) → 수천 대 (UTM 도시 규모)
- 정적 데이터 → 실시간 스트림 (Kafka·MQTT)
- 단일 데이터베이스 → 분산 클러스터 (TypeDB Cloud)

### 다른 도구와의 결합

- **ROS·PX4** — 실제 비행 제어
- **Gazebo·AirSim** — 물리 시뮬레이션
- **Grafana** — 실시간 시각화
- **Apache Kafka** — 이벤트 스트리밍

### LLM과의 결합 — 뉴로심볼릭

자매 책 *6편 뉴로심볼릭*에서 본 패턴이 — 드론 도메인에도 적용 가능.

- 자연어 임무 지시(*"이 지역을 1시간 안에 수색하라"*)를 LLM이 *온톨로지 매듭*으로 자동 변환
- 임무 중 사건 발생 시 — LLM이 *사건의 의미*를 추출, 시스템이 *영향과 대응*을 도출
- 답변의 *근거를 데이터에서 추적*하는 *설명 가능한 AI*

이 결합이 — *다음 책*의 자리다.

### 다른 도메인으로의 이식

같은 네 도구가 자연스러운 다른 도메인:

- **의료** — 환자·질병·치료·약물 N항, 진료 이력 시점, 치료 결과 재귀 분석
- **법률** — 판례·법조문·당사자 N항, 인용 그래프 재귀
- **생명과학** — 단백질·유전자·경로·약물 분류, 상호작용 그래프
- **공급망 일반** — 자매 책의 반도체 사례를 다른 산업(자동차·항공·식품)으로

각 도메인에서 — 이 책의 *4장(도메인) → 5장(스키마) → 6장(데이터) → 7장(함수) → 8장(시나리오)*의 패턴이 동일하게 작동.

---

## 9.6 책을 닫으며

이 책이 다룬 것:

- TypeDB가 국내에 정식 소개되는 첫 자리
- *네 가지 도구*(분류·N항·역할·함수)의 이론적 자리
- *드론 군집비행*이라는 본격 도메인의 온톨로지 짓기
- *한 사건의 영향이 자동으로 풀려나가는* 시스템 시연
- 자매 책과의 *같은 도구·다른 도메인* 회수

이 책의 진짜 자리는 — *TypeDB의 매뉴얼*이 아니라 *온톨로지 사고의 한 사례집*이었다. 분류·관계·역할·시점이 *동시에 의미를 가지는 자리*를 어떻게 *데이터로 옮기는가*에 대한 한 권의 답이었다.

자매 책 *광전자에서 시작된 한 권의 책*과 함께 읽힐 때 — 두 책이 *같은 도구의 두 도메인*에서 일하는 모양이 가장 또렷이 드러난다. 하나는 회상의 호흡으로, 하나는 코드와 이론의 호흡으로. 두 책이 *온톨로지 입문 코스 1세트*가 된다.

이 책의 약속이 진짜로 작동하는 자리는 — 이 책의 마지막 페이지가 아니라, *이 책을 덮은 다음의 독자가 자기 도메인에서 짜는 첫 스키마*다.

그 자리가 잘 짜이기를.

— 끝.
