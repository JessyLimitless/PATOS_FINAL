# abstraction_steps.md — 한 행이 ontology entity가 되는 9단계

이 문서는 `buildings.csv`의 한 행(#B-3021, 한옥마을 21호)이 어떻게 RDF triple → 룰 평가 가능한 entity로 *추상화*되는지를 단계별로 보여줍니다. 엔지니어 학습자가 자기 데이터에 동일한 절차를 적용할 수 있도록, **모든 결정에 "다른 선택지 + 왜 이걸 골랐나"** 를 함께 기록했습니다.

기준 시나리오: `ruledata.cloud5.socialbrain.co.kr/#how` · 무안 한옥마을 #B-3021

---

## 시작점 — Raw 한 행

`buildings.csv`의 첫 데이터 행:

```csv
bldg_id,name,type,sub_type,location_addr,year,structure,roof,total_area,floor_count,fire_risk_current,status,cluster_id,owner_role,registered
#B-3021,한옥마을 21호 (김씨고가),TraditionalCluster,한옥,전남 무안군 몽탄면 한옥마을길 21,1923,일반목구조,한식기와,158.4,1,0.85,경보,CL-01,보존회 위탁운영,Y
```

목표: 이 14컬럼 한 줄을 *룰 엔진이 추론할 수 있는 entity*로 변환.

---

## Step 1 — 소스 식별 (Provenance)

가장 먼저 답해야 할 질문: **이 데이터는 어디서 왔는가?**

```turtle
:Building/B-3021
    prov:wasDerivedFrom :Source/buildings.csv ;
    prov:generatedAtTime "2024-03-15T08:00:00+09:00"^^xsd:dateTime ;
    prov:wasAttributedTo :DataSource/MOLIT-건축물대장 .
```

**왜 중요한가**: 같은 건물 정보가 *건축물대장(MOLIT)*에서 올 수도, *지자체 자체 조사*에서 올 수도, *보존회 등록부*에서 올 수도 있음. 출처에 따라 신뢰도·갱신주기·법적 구속력이 다름.

**다른 선택지**: provenance 무시하고 plain entity만. → 향후 "이 fireRisk 0.85는 누가 어떻게 계산?" 추적 불가. 운영 단계에서 반드시 후회함.

**결정**: PROV-O 어휘로 최소한의 출처는 항상 기록.

---

## Step 2 — URI 명명 규칙 결정

CSV의 `bldg_id = #B-3021`을 어떻게 URI로 만들 것인가?

| 방안 | 예시 | 장점 | 단점 |
|---|---|---|---|
| (a) HTTP URI | `http://patos.example/ontology/Building/B-3021` | 표준·dereferenceable | 길고 도메인 정해야 함 |
| (b) URN | `urn:patos:building:muan:B-3021` | 도메인 독립 | dereferenceable 아님 |
| (c) BNode | `_:b3021` | 간단 | 식별 불안정 — 외부 참조 불가능 |
| (d) Compact prefix | `:Building/B-3021` (turtle prefix) | 가독성 | prefix 매핑 필요 |

**결정**: **(d) compact prefix** 사용 (`@prefix : <http://patos.example/ontology#>`). 내부 일관성 + Turtle 가독성. 외부 발행 시 `(a)`로 자동 확장.

**핵심 원칙 — 이름 절대 금지**:
```
❌ :Building/한옥마을21호김씨고가     (한국어 + 공백 깨질 수 있음)
❌ :Building/Kim-Family-Old-House    (영문 번역 표류)
✅ :Building/B-3021                  (불투명 식별자)
```

`rdfs:label`에 사람-읽기용 이름을 별도로 둠. **URI는 결코 이름이 아니어야 함** — 이름은 바뀌고, 합쳐지고, 분리됨. URI는 영원해야 함.

---

## Step 3 — 클래스 결정 (가장 어려운 단계)

`type=TraditionalCluster` 이 컬럼을 어떻게 ontology로 옮길까?

### 옵션 A: 단순 literal value
```turtle
:Building/B-3021 a :Building ;
                 :type "TraditionalCluster" .
```
→ RULE-002가 `?b :type "TraditionalCluster"`로 평가. 작동은 함.

### 옵션 B: 클래스 멤버십으로 끌어올림
```turtle
:TraditionalClusterMember rdfs:subClassOf :Building .
:Building/B-3021 a :TraditionalClusterMember .
```
→ RULE-002가 `?b a :TraditionalClusterMember`로 평가.

### 옵션 C: 더 세분화된 계층
```turtle
:Building rdfs:subClassOf owl:Thing .
:TraditionalClusterMember rdfs:subClassOf :Building .
:Hanok                    rdfs:subClassOf :TraditionalClusterMember .
:Yangban-Hanok            rdfs:subClassOf :Hanok .

:Building/B-3021 a :Hanok .
```
→ subsumption으로 자동 추론: `:B-3021 a :Hanok ⊑ :TraditionalClusterMember ⊑ :Building`.

**결정**: **(C) 다단계 계층** 채택.

**왜 (C)인가**:
- 한옥은 "전통건축물 클러스터의 한 종류"이자 "건물의 한 종류". 두 진리가 동시에 필요
- RULE-002는 `:TraditionalClusterMember`만 알면 됨 — `:Hanok`이든 `:Banga`든 묻지 않음
- 새로운 전통건축 종류(예: `:Banga`, `:Seowon`)가 추가될 때 *룰 수정 불필요* — subClassOf만 선언
- **이게 OWL의 진짜 가치**: open-world assumption + subsumption이 *룰 안정성*을 만든다

**(A) 안 한 이유**: literal "TraditionalCluster"는 오타·번역에 취약. 한 곳에서 "Traditional Cluster"(공백)로 적히면 매칭 실패. 클래스 URI는 단 한 군데(scheme 정의)에서만 정의되므로 그런 류의 silent failure 없음.

---

## Step 4 — Atomic 컬럼 → Datatype Property

스칼라 값들을 RDF datatype property로 매핑:

```turtle
:Building/B-3021
    rdfs:label    "한옥마을 21호 (김씨고가)" ;            # name
    :builtYear    "1923"^^xsd:gYear ;                     # year
    :totalArea    "158.4"^^xsd:decimal ;                  # total_area (m²)
    :floorCount   "1"^^xsd:integer ;                      # floor_count
    :fireRisk     "0.85"^^xsd:decimal ;                   # fire_risk_current
    :ownerRole    "보존회 위탁운영" ;                       # owner_role
    :registered   "true"^^xsd:boolean .                    # registered "Y" → boolean
```

**xsd 타입 선택의 의미**:
- `xsd:gYear`로 `"1923"`: 연도임을 명시 → 산술 비교 가능 (`?y > 1900`)
- `xsd:decimal`로 `"0.85"`: 부동소수점 비교
- `xsd:boolean`으로 `"Y"→"true"`: 룰에서 if/else 평가

**다른 선택지**: 모든 값을 `xsd:string`으로. → 산술·비교가 안 되거나 매번 cast 필요. **타입은 데이터의 *의도*를 표현하는 가장 저렴한 방법** — 무조건 명시.

---

## Step 5 — Coded Value → URI (정규화)

`structure="일반목구조"`, `roof="한식기와"`, `status="경보"` 같은 *분류 코드*는 literal로 두지 말고 URI로 정규화:

```turtle
# 코드 vocabulary 정의 (한 곳에서 정의)
:StructureCode/일반목구조  a :StructureCode ;
                          rdfs:label "일반목구조" ;
                          skos:prefLabel "Wood Frame, General"@en .
:StructureCode/철근콘크리트구조 a :StructureCode ;
                          rdfs:label "철근콘크리트구조" .

# 인스턴스
:Building/B-3021
    :structure :StructureCode/일반목구조 ;
    :roof      :RoofCode/한식기와 ;
    :status    :Status/경보 .
```

**왜 URI인가**:
- 오타 방지: literal "일반 목구조"(공백 추가)는 다른 값. URI는 한 곳에서만 정의
- 다국어: `skos:prefLabel`로 한/영/일 라벨 동시 보유
- 위계: `:Hanok-Roof rdfs:subClassOf :RoofCode` 같은 분류 가능
- **국제 표준 매핑**: 향후 ISO/W3C 표준 코드와 `owl:sameAs`로 연결 가능

이 단계가 ruledata #why가 말한 **"Shared Semantic Substrate"**의 본체입니다.

---

## Step 6 — FK·다중값 관계 처리

CSV는 평면이라 관계가 *컬럼 안에* 숨어 있음:

### `cluster_id="CL-01"` (단일 FK)
```turtle
:Building/B-3021 :memberOf :Cluster/CL-01 .
```
간단. 1:N 관계 (Cluster → 여러 Building).

### `adjacency.csv` (외부 다중 관계)
```csv
from_bldg,to_bldg,distance_m,via_type
#B-3021,#B-3019,42,건물간직선
#B-3021,#B-3022,68,건물간직선
...
```

이 관계는 단순한 owl:ObjectProperty로는 부족 — **거리·경로 종류**라는 *attribute*가 관계에 붙어 있음. RDF 표준 패턴 두 가지:

#### 옵션 A: 단순 ObjectProperty (정보 손실)
```turtle
:Building/B-3021 :adjacentTo :Building/B-3019 .
```
→ 거리 정보 사라짐. RULE-003의 `distance < 100` 평가 불가.

#### 옵션 B: Reified Statement / N-ary
```turtle
:Building/B-3021 :hasAdjacency :Adjacency/B3021-B3019 .
:Adjacency/B3021-B3019
    a :Adjacency ;
    :to        :Building/B-3019 ;
    :distance  "42"^^xsd:decimal ;
    :viaType   :ViaType/건물간직선 .
```

**결정**: **(B) N-ary**. 거리 임계 룰을 위해 필수.

**대안**: RDF-star (`<<:B-3021 :adjacentTo :B-3019>> :distance "42"`) — 더 간결하지만 도구 지원 아직 약함.

---

## Step 7 — 결측값 처리

`buildings.csv` 마지막 행에는 `bldNm`, `structure`, `useAprDay`가 비어 있음 (실제 미등록 건물). 이걸 ontology에서 어떻게?

### 옵션 1: triple 자체를 생략 (Open World)
```turtle
:Building/B-something
    rdfs:label "..." ;
    # :structure 없음
    :totalArea "72.5"^^xsd:decimal .
```
→ "구조 정보가 *없다*" 가 아니라 "*아직 알지 못한다*". OWL의 open-world에 부합.

### 옵션 2: 명시적 unknown
```turtle
:Building/B-something :structure :Unknown .
```
→ "*조사했지만 식별 불가*"라는 *지식*을 표현. SHACL constraint 검사 시 통과시킬 수 있음.

### 옵션 3: nil literal
```turtle
:Building/B-something :structure rdf:nil .
```
→ 잘 안 씀. 비표준.

**결정**: 데이터가 *수집되지 않은* 결측 → **옵션 1 (생략)**. 데이터가 *수집됐지만 미상* → **옵션 2 (`:Unknown`)**. 둘은 구분되어야 함.

매핑 단계에서 이 구분을 *명시적으로* 하지 않으면 나중에 "모르는 건물 N채" vs "조사된 미상 N채" 같은 운영 보고가 불가능.

---

## Step 8 — SHACL 검증 (게이트)

ontology에 entity를 *받아들이기 전에* 검증:

```turtle
@prefix sh: <http://www.w3.org/ns/shacl#> .

:BuildingShape
    a sh:NodeShape ;
    sh:targetClass :Building ;
    sh:property [
        sh:path :builtYear ;
        sh:datatype xsd:gYear ;
        sh:minInclusive "1800"^^xsd:gYear ;
        sh:maxInclusive "2030"^^xsd:gYear ;
        sh:message "건축연도가 1800~2030 범위를 벗어남"
    ] ;
    sh:property [
        sh:path :fireRisk ;
        sh:datatype xsd:decimal ;
        sh:minInclusive "0.0"^^xsd:decimal ;
        sh:maxInclusive "1.0"^^xsd:decimal ;
    ] ;
    sh:property [
        sh:path :memberOf ;
        sh:class :Cluster ;
        sh:maxCount 1 ;        # 한 클러스터에만 속할 수 있음
    ] .
```

**왜 OWL이 아니라 SHACL인가** (이게 사용자의 OWL 질문 핵심):

| 작업 | 도구 | 이유 |
|---|---|---|
| 클래스 계층, 어휘 정의 | **OWL/RDFS** | 표현력 적합 |
| 필수값·범위·카디널리티 검증 | **SHACL** | OWL의 open-world와 정반대 — 검증은 closed-world |
| 임계·이벤트·시간 룰 | **외부 룰엔진** | OWL DL에서 표현 불가 |

OWL DL reasoner(HermiT, Pellet)가 *production*에서 도는 곳이 드문 이유: open-world 가정 때문에 "fireRisk 누락 = 위반"을 표현 못 함. SHACL은 closed-world라 이 검증이 자연스러움.

---

## Step 9 — 룰 평가 가능 상태로 활성화

이제 entity가 룰 엔진의 working memory에 들어감. RULE-001이 평가:

```python
# 의사코드
for building in graph.instances_of(:Building):
    if building.fireRisk > 0.70:
        emit_trigger(severity=HIGH)
        create(Incident, building=building, time=now, severity=HIGH)
```

`:Building/B-3021.fireRisk = 0.85` → 트리거 발동. **여기까지가 추상화 사다리의 정점**.

ruledata #how 사이트에서 보여주는 `200ms 시퀀스`가 정확히 이 단계 — 9단계의 추상화가 완료된 entity 위에서 4개 룰이 connection chain을 이루며 발동.

---

## 통합본: #B-3021의 최종 RDF

```turtle
@prefix :     <http://patos.example/ontology#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix xsd:  <http://www.w3.org/2001/XMLSchema#> .
@prefix prov: <http://www.w3.org/ns/prov#> .

# === 클래스 계층 (Step 3) ===
:Building                 a owl:Class .
:TraditionalClusterMember rdfs:subClassOf :Building .
:Hanok                    rdfs:subClassOf :TraditionalClusterMember .

# === 인스턴스 (Steps 1, 2, 4-7) ===
:Building/B-3021
    a                  :Hanok ;                                # Step 3
    rdfs:label         "한옥마을 21호 (김씨고가)" ;              # Step 4
    :builtYear         "1923"^^xsd:gYear ;
    :totalArea         "158.4"^^xsd:decimal ;
    :floorCount        "1"^^xsd:integer ;
    :fireRisk          "0.85"^^xsd:decimal ;
    :registered        "true"^^xsd:boolean ;
    :structure         :StructureCode/일반목구조 ;              # Step 5
    :roof              :RoofCode/한식기와 ;
    :status            :Status/경보 ;
    :ownerRole         "보존회 위탁운영" ;
    :memberOf          :Cluster/CL-01 ;                         # Step 6
    :hasAdjacency      :Adjacency/B3021-B3019 ,
                       :Adjacency/B3021-B3022 ,
                       :Adjacency/B3021-B3025 ,
                       :Adjacency/B3021-B3018 ,
                       :Adjacency/B3021-B3027 ;
    :address           [ :sido        :Region/전라남도 ;
                         :sigungu     :Region/무안군 ;
                         :eupmyundong :Region/몽탄면 ;
                         :roadName    "한옥마을길" ;
                         :buildingNo  "21" ] ;
    prov:wasDerivedFrom :Source/buildings.csv ;                 # Step 1
    prov:generatedAtTime "2024-03-15T08:00:00+09:00"^^xsd:dateTime .

:Adjacency/B3021-B3019
    a         :Adjacency ;
    :to       :Building/B-3019 ;
    :distance "42"^^xsd:decimal ;
    :viaType  :ViaType/건물간직선 .
# (다른 4개 adjacency도 동일 패턴)
```

---

## OWL · SHACL · 룰엔진의 *경계* (한 화면 요약)

```
┌─────────────────────────────────────────────────────────────────┐
│  OWL/RDFS (정적 어휘)                                           │
│  - 클래스 계층 (Hanok ⊑ TraditionalClusterMember ⊑ Building)   │
│  - Property 정의 (:fireRisk a owl:DatatypeProperty)             │
│  - 동치성 (owl:sameAs, owl:equivalentClass)                     │
│  → reasoner 거의 안 돌림. Subsumption 정도만.                    │
├─────────────────────────────────────────────────────────────────┤
│  SHACL (Validation Gate)                                        │
│  - 필수값, 타입, 범위, 카디널리티                                 │
│  - "fireRisk는 0~1 사이여야 함"                                 │
│  - "Building은 cluster를 1개만 가짐"                            │
│  → 데이터 받아들이기 전 게이트. 위반은 reject 또는 quarantine.   │
├─────────────────────────────────────────────────────────────────┤
│  외부 룰엔진 (Action Logic)                                     │
│  - 임계 (fireRisk > 0.70)                                       │
│  - 그래프 traversal (adjacentTo where distance < 100)           │
│  - 외부 호출 (notify, dispatch)                                 │
│  - 시간 (since, until, every)                                   │
│  → 여기에 비즈니스 로직이 응축. ruledata #how의 4개 룰이 이 영역. │
└─────────────────────────────────────────────────────────────────┘
```

**한 줄 요약**: OWL은 *세계의 구조*, SHACL은 *세계의 게이트*, 룰엔진은 *세계의 행동*.

---

## 흔히 하는 실수 (Anti-patterns)

1. **이름을 URI에 박기** — `:Building/한옥마을21호` → 이름 변경 시 모든 참조 깨짐
2. **모든 컬럼을 `:` literal로** — 코드값(`"일반목구조"`)을 URI 정규화 안 함 → 오타·번역 표류
3. **OWL DL reasoner로 모든 추론 처리 시도** — 임계·시간·외부호출 표현 불가. 룰엔진에 위임해야 함
4. **결측을 0/빈문자열로 채워 넣음** — "정보 없음"과 "값이 0"이 구분 안 됨. 트리플 *생략*이 정답
5. **provenance 안 적기** — 운영 단계에서 "이 값은 어디서 왔지?" 추적 불가
6. **n-ary 관계를 단순 ObjectProperty로** — `:adjacentTo` 만 두면 거리·경로종류 정보 사라짐. RULE-003 평가 불가
7. **literal에 한국어/영어 혼재** — `:status "경보"` vs `:status "alert"` 가 동시에 존재. SKOS multilingual label 또는 URI 정규화로 해결

---

## 다음 단계

1. **`schema_decisions.md`** — 위 9단계의 결정 일지를 ADR(Architecture Decision Record) 포맷으로 정리
2. **`ontology.ttl`** — 클래스 계층·property 정의 전체 (`:Building`, `:TraditionalClusterMember`, `:Hanok`, `:Adjacency`, `:Cluster` 등)
3. **`shapes.ttl`** — SHACL 검증 규칙 전체
4. **`mapping_rules.yaml`** — CSV 컬럼 → ontology property 자동 매핑 (이 yaml을 마지막에 R2RML로 변환하는 사례 추가)
5. **R2RML 등가 변환** — 학습자가 *표준*과 *DSL*의 차이를 체감하도록

---

## 참고

- W3C OWL 2 Primer: https://www.w3.org/TR/owl2-primer/
- W3C SHACL: https://www.w3.org/TR/shacl/
- W3C PROV-O: https://www.w3.org/TR/prov-o/
- W3C R2RML: https://www.w3.org/TR/r2rml/
- ruledata #how: https://ruledata.cloud5.socialbrain.co.kr/#how
