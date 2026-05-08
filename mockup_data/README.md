# mockup_data — 무안 한옥마을 화재 대응 온톨로지 학습 데이터셋

## 이 폴더의 목적

엔지니어 학습자가 **실제 데이터의 형태와 구조** 위에서 *온톨로지 설계와 매핑*을 직접 연습할 수 있도록 만든 mock 데이터셋입니다.

기준 시나리오는 **ruledata.cloud5.socialbrain.co.kr/#how**의 무안 한옥마을 화재 사례를 그대로 재현 가능하도록 설계되었습니다 — 동일한 건물 ID(#B-3021), 동일한 4개 룰, 동일한 트리거 시각(15:14:23), 동일한 5건 출동 + 6건 통보.

- **컬럼명·구조**: 실제 공공데이터(건축물대장·소방청·KMA·KOSIS) 명세를 참고하되, *룰 시뮬레이션이 가능한 최소 컬럼*으로 재구성
- **행 데이터**: 무안 몽탄면 기반 plausible 가짜 데이터 (실명·실주소 가공)
- **목적**: "raw 데이터가 어떻게 ontology entity로 추상화되고, 그 위에서 룰이 어떻게 작동하는가"의 *전체 추적 가능한 시연*

---

## 파일 목록

| 파일 | 클러스터 | 행 수 | 핵심 컬럼 |
|---|---|---|---|
| `buildings.csv` | Building | 12 | bldg_id, type, year, structure, fire_risk_current, cluster_id |
| `fire_risk_observations.csv` | Observation (시계열) | 10 | obs_id, bldg_id, observed_at, fire_risk, trigger_signal |
| `traditional_clusters.csv` | TraditionalCluster | 1 | cluster_id, member_count, preserved_since, contact |
| `adjacency.csv` | adjacentTo (관계) | 12 | from_bldg, to_bldg, distance_m |
| `fire_stations.csv` | FireStation | 5 | station_id, jurisdiction, *_count, personnel |
| `resources.csv` | Resource (차량·팀) | 8 | resource_id, type, home_station, status, eta_to_b3021_min |
| `incidents.csv` | Incident | 4 | incident_id, building_id, started_at, severity, rules_fired |
| `agencies.csv` | Agency (외부기관) | 4 | agency_id, type, contact, notification_channel |
| `rules.yaml` | Rule definitions | 4룰 | id, when, then/match/collect, metadata |

---

## ruledata #how 시나리오와의 매핑

| #how 시나리오 요소 | 데이터 위치 |
|---|---|
| 트리거 건물 #B-3021 (fireRisk = 0.85) | `buildings.csv` 1행 + `fire_risk_observations.csv` OBS-9004 |
| 위험도 진행 14:00→15:14:23 | `fire_risk_observations.csv` OBS-9001~9004 |
| 4개 룰 정의 | `rules.yaml` RULE-001~004 |
| RULE-002 자원 매칭 (폼차×2 + 전통건축팀, 화학소방차 제외) | `resources.csv` (VEH-117·VEH-203·TEAM-T1) + `rules.yaml` RULE-002 |
| RULE-003 100m 이내 인접 (3건) | `adjacency.csv` (#B-3021의 distance < 100인 행: #B-3019·#B-3022·#B-3025) |
| RULE-004 외부 통보 (소방서·국가유산청·보존회장) | `agencies.csv` AG-01·AG-02·AG-03 + `traditional_clusters.csv` |
| 5 dispatches | `resources.csv` (VEH-117 4min, VEH-203 6min, AMB-09 2min, CMD-01 5min, TEAM-T1 8min) |
| 6 notifications | adjacency 3건 + agencies 3건 |

→ 학습자는 `rules.yaml`의 RULE-001~004를 순차 실행해보면 ruledata #how의 200ms 시퀀스를 그대로 재현할 수 있습니다.

---

## 인코딩·줄바꿈

- **본 mock**: UTF-8 (BOM 없음), LF
- **실제 공공데이터 다수**: EUC-KR/CP949, CRLF — 특히 건축물대장·통계청 KOSIS Excel/CSV
- **변환**: `iconv -f euc-kr -t utf-8 input.csv > output.csv` 또는 `pd.read_csv(path, encoding='cp949')`

---

## NULL / 결측 처리 컨벤션

- **빈 셀** (`,,`): 미수집·미등록·결측 (예: `incidents.csv`의 affected_count, closed_at)
- 실데이터에선 `NULL`, `-`, `미상` 같은 문자열도 자주 등장 — 매핑 단계에서 *어떤 셀이 ontology의 누락/unknown으로 가야 할지* 결정하는 것이 핵심 토픽

---

## 주요 cross-cluster 관계 (학습 포인트)

| 관계 | 연결 | 의미 |
|---|---|---|
| Building → TraditionalCluster | `cluster_id` | 클러스터 멤버십 (RULE-002·004 발동 조건) |
| Building ↔ Building | `adjacency.csv` | 100m 이내 인접 (RULE-003) |
| Building → FireStation | `location_addr` ↔ `jurisdiction` (이름 기반!) | 관할 소방서 |
| Resource → FireStation | `home_station`, `deployed_at` | 정주지 ≠ 현재 위치 가능 |
| Incident → Building | `building_id` | 사건 발생 위치 |
| Incident → Rule | `rules_fired` (`\|` 구분 다중값) | 어느 룰이 발동했나 |
| TraditionalCluster → Agency | `heritage_authority` (이름 기반!) | 통보 대상 기관 |

→ 이름 기반 join 후보(`jurisdiction`, `heritage_authority`)는 *행정개편·기관명 변경 시 깨짐*. 온톨로지에서는 URI로 정규화하는 이유가 여기서 시연됨 (예: `agencies.csv`의 AG-02 note: "2024년 문화재청 → 국가유산청 개편").

---

## 일부러 둔 함정 / 학습 포인트

엔지니어가 "real data가 이런 거구나"를 체감하도록 의도적으로 심은 것들:

1. **`buildings.csv` `type=TraditionalCluster`** — Building의 attribute로 cluster 타입을 표시했지만, 동시에 `traditional_clusters.csv`에 별도 entity가 있음. 온톨로지에서 `:Building`의 *서브클래스*로 표현할지, *멤버십 관계*로 표현할지가 schema_decisions의 첫 토픽

2. **`resources.csv`의 `eta_to_b3021_min` 컬럼** — 자원의 *고유 속성*이 아닌 *#B-3021 기준 derived value*. 정석은 current_location_lat_lng + on-the-fly 계산. 학습자에게 "어느 컬럼은 stored fact, 어느 컬럼은 derived인가" 토론 포인트

3. **`agencies.csv` AG-02 note** — `문화재청 → 국가유산청` (2024 개편). `rules.yaml` RULE-004의 `naming_history` 메타가 같은 사실을 명시. *명칭 변경*이 온톨로지 evolution 이슈의 대표 사례 — URI 기반 식별이 왜 필수인가의 답

4. **`fire_stations.csv` ST-201의 location** — "광주광역시 본부 / 무안 임시거점 운영" — 텍스트에 두 위치가 섞여 있음. 정규화 필요. 이런 *비표준 텍스트 필드*는 실데이터에 흔함

5. **`adjacency.csv`의 `via_type`** — "건물간직선" vs "골목경유" — 100m라는 거리 임계가 *직선거리*인지 *실 도보거리*인지 모호. RULE-003의 `distance < 100` 의미가 무엇인지 명세 필요 (현재 룰 메타엔 명시 안 됨 → 학습자가 발견할 갭)

6. **`fire_risk_current` (현재) vs `fire_risk_observations` (시계열)** — Building 테이블에 *현재값*이 denormalize되어 있음. 시계열과 동기화는 누가 책임지나? 룰 발동은 어느 쪽을 보나? — 데이터 일관성 설계 토픽

---

## 룰 작동 추적 예시

`#B-3021`의 fire_risk가 0.85가 된 순간(`OBS-9004`) 어느 룰이 어떤 순서로 발동하는지:

```
T+0ms     RULE-001 평가
          → fire_risk_current(#B-3021) = 0.85 > 0.70 ✓
          → emit_trigger(severity=HIGH)
          → create Incident(INC-882, #B-3021, NOW, HIGH)

T+45ms    그래프 traversal
          → #B-3021.cluster_id = CL-01 (TraditionalCluster)
          → #B-3021.adjacentTo from adjacency.csv
              [#B-3019(42m), #B-3022(68m), #B-3025(93m), #B-3018(135m), #B-3027(210m)]

T+78ms    RULE-002 평가
          → INC-882.building.type == TraditionalCluster ✓
          → match resources where type in [폼차, 전통건축팀]
            and exclude type == 화학소방차
          → 후보: VEH-117(4min), VEH-203(6min), TEAM-T1(8min)

T+112ms   RULE-003 평가
          → INC-882.severity == HIGH ✓
          → collect adjacentTo where distance < 100
          → 결과: [#B-3019, #B-3022, #B-3025]  (3건, #B-3018·#B-3027 제외)
          → notify(3건, "선제대피 권고")

T+156ms   RULE-004 평가
          → INC-882.building.type == TraditionalCluster ✓
          → notify [관할소방서=무안소방서(AG-01),
                    국가유산청(AG-02),
                    보존회장(AG-03 via cluster CL-01.contact)]

T+201ms   dispatch 실행
          → VEH-117, VEH-203, TEAM-T1 + AMB-09(보충), CMD-01(보충) → #B-3021
          → 5 dispatches + 6 notifications 완료
```

→ 이 시퀀스는 ruledata #how의 200ms 타임라인과 *정확히* 대응.

---

## 다음 산출물 (예고)

이 데이터셋 위에 다음 자료를 차례로 쌓을 예정:

1. **`mapping_diagram.html`** — *raw CSV 컬럼* → *Entity 속성* → *Rule 조건* → *Trigger·Action*의 4단 매핑을 시각화. 페어 뷰의 좌측 매크로 클러스터 맵 + 우측 마이크로 패널
2. **`schema_decisions.md`** — 5개 핵심 설계 결정 일지 (Building 클래스 계층 / URI 명명 / OWL vs SHACL vs 룰엔진 분업 / 시간 모델링 / NULL 처리)
3. **`ontology.ttl`** — Turtle 형식 온톨로지 정의 (RDFS·OWL 어휘 사용, *추론은 룰엔진에 위임*)
4. **`mapping_rules.yaml`** — declarative 매핑 (CSV 컬럼 → ontology property). 마지막에 R2RML 변환 사례 추가
5. **patos_demo 안 새 뷰 또는 별도 페이지** — 페어 뷰 HTML 구현

---

## 라이선스 / 면책

- 이 mock 데이터의 행은 *학습 목적의 가공된 가짜 데이터*입니다. 실명·실주소·실연락처를 동일하게 사용하지 마세요.
- 컬럼·구조는 공공데이터포털·MOLIT·KMA·소방청 공개 명세를 *참고*했으나, 학습 단순화를 위해 일부 변형되었습니다. 실데이터로 갈아끼울 때 명세 차이 확인 필요.
- "무안 한옥마을"·"몽탄면 한옥마을길"·"보존회"·"이○○" 등은 *시나리오 재현용 가공명*입니다.
