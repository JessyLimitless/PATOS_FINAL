# PATOS_FINAL

무안 한옥마을 화재 대응 온톨로지 학습 데모.

## 빠른 시작

📱 **모바일 최적화 학습 hub**: [`mockup_data/index.html`](mockup_data/index.html)

세 가지 대화형 데모가 hub에서 바로 열립니다:

- 🗺️ **매핑 설계도** ([mapping_diagram.html](mockup_data/mapping_diagram.html)) — Raw CSV → Entity → Rule → Action 4단 추적
- 🦉 **OWL 자동 추론** ([owl_reasoner_demo.html](mockup_data/owl_reasoner_demo.html)) — 7개 사실 → 14개 자동 도출
- ⚖️ **변화 비교** ([change_comparison.html](mockup_data/change_comparison.html)) — 시나리오 vs OWL 추론

## 무엇을 배우는가

엔지니어 학습자가 **실제 데이터셋 위에서 온톨로지를 어떻게 설계하고 구성하는지** 따라할 수 있도록 만든 데모. 무안 한옥마을 화재 대응 시나리오 ([ruledata #how](https://ruledata.cloud5.socialbrain.co.kr/#how) 호환)를 anchor로 사용.

핵심 세 단계:
1. **묶기 (bundling)** — OWL 클래스 계층 (`ontology.ttl`)
2. **매핑 (mapping)** — CSV 컬럼 → RDF property 선언형 규칙 (`mapping_rules.yaml`)
3. **결과 (instances)** — 매핑된 사실 + 자동 추론 (`instances.ttl`)

## 폴더 구조

```
.
├── README.md                       ← 이 파일
├── mockup_data/                    ← 학습 데모 본체
│   ├── index.html                  ← 모바일 최적화 hub (시작점)
│   ├── mapping_diagram.html        ← 4단 추상화 시각화
│   ├── owl_reasoner_demo.html      ← OWL 자동 추론 시연
│   ├── change_comparison.html      ← 시나리오 vs OWL 비교
│   ├── ontology.ttl                ← OWL 클래스 계층
│   ├── mapping_rules.yaml          ← CSV→RDF 매핑 규칙
│   ├── instances.ttl               ← 변환 결과 + 추론
│   ├── rules.yaml                  ← RULE-001~004 (선언형)
│   ├── *.csv                       ← 8개 데이터셋
│   ├── README.md                   ← 상세 가이드
│   └── abstraction_steps.md        ← 9단계 추상화 walkthrough
├── index.html                      ← 원본 PATOS 대시보드 (legacy)
└── ...
```

## 면책

학습 목적 데모. CSV 행 데이터는 **plausible mock**이며 실명·실주소·실연락처가 아닙니다. 컬럼 명세는 공공데이터포털·MOLIT·KMA·소방청 공개 명세를 참고했으나 학습 단순화를 위해 일부 변형되었습니다.
