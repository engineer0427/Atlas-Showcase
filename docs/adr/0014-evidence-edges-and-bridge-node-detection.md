# 0014: Evidence Edges and Bridge Node Detection

## Status
Proposed

## Context
Atlas 엔진은 pgvector 기반 시맨틱 매칭을 통해 관련 문서를 빠르게 찾을 수 있지만, 단순히 Top-K 문서 목록(검색 결과)만으로는 다음을 충족하기 어렵다.

- 연구자가 "왜 이 둘이 연결되는가?"를 검증할 수 있는 근거(Explanation/Evidence)가 부족하다.
- 다학제 탐색에서 중요한 것은 유사 문서 나열이 아니라, 서로 다른 지식 군집 사이의 "브릿지(Bridge)"를 찾아
  잠재적 연구 가설(= Missing Link)을 제시하는 것이다.
- 데이터 소스가 확장(bioRxiv, medRxiv, patent 등)될수록 중복/표기 차이/메타데이터 불일치로 그래프가 오염되어,
  연결의 신뢰도가 급격히 떨어질 위험이 있다.

기존 구현의 한계:
- `GraphBuilder.build_knn_edges(...)`는 엣지에 `weight`만 저장하여, 연결이 생성된 이유를 시스템이 보존하지 못한다.
- `DeepInsightGenerator`는 클러스터 단위 요약/휴리스틱을 제공하지만, 그래프 엣지의 근거를 요약 문장에 반영하지 못한다.
- 타입(`PaperMetadata`)이 중복 정의되어(backend/engine/types.py vs engine_types.py) 근거 필드 확장 및 데이터 정규화가 어렵다.
- `AggregationEngine`이 수집 결과를 그대로 반환하여 동일 논문(제목/DOI 중복)이 그래프 노이즈를 만든다.

## Decision
1) Evidence-first Edge Schema를 도입한다.
- 그래프 엣지(dict)에는 최소한 아래 필드를 포함한다.
  - `source`, `target`: paper id
  - `weight`: 0..1
  - `reason`: 연결 근거의 타입(예: `semantic_similarity`, `shared_keywords`, `methodological_overlap`)
  - `evidence`: reason을 뒷받침하는 구조화된 값(예: `semantic_similarity` 수치, `shared_keywords` 리스트)

- 목적:
  - "연결 자체"가 아니라 "연결의 근거"를 1급 데이터로 취급하여,
    엔진을 단순 검색기에서 Evidence-based Insight Tool로 격상한다.

2) Bridge Node(다학제 연결점) 플래그를 도입한다.
- KNN 그래프 기반으로 각 paper node에 다음을 계산해 주입한다.
  - `is_bridge_node`: 브릿지 여부 (bool)
  - `bridge_score`: 브릿지 점수 (float, 0..1)

- 구현 원칙:
  - 초기 버전은 운영/성능/의존성 측면에서 heavy community detection을 피하고,
    degree/이웃 다양성/이웃 간 결속도(낮을수록 bridge-like) 등의 휴리스틱을 사용한다.
  - 알고리즘은 추후 Louvain/Leiden/Infomap 등으로 교체 가능하도록 모듈 경계를 유지한다.

3) 타입을 단일화한다.
- Canonical schema는 `backend/engine/engine_types.PaperMetadata`로 고정한다.
- `backend/engine/types.py`는 canonical을 re-export하는 호환 레이어로 전환하여, 중복 정의를 제거한다.
- 목적:
  - reason/evidence/doi/source 같은 확장 필드를 안정적으로 추가 가능하게 한다.
  - provider/DB/랭킹/그래프/요약이 동일 계약을 공유하게 한다.

4) 수집 단계에서 Deduplication을 수행한다.
- `AggregationEngine.collect()`는 소스별 수집 결과에 대해 중복 제거를 수행한다.
- Dedup key 우선순위:
  1) DOI(존재할 때)
  2) 정규화된 제목(normalized title)
  3) (fallback) id

- 목적:
  - 동일 리소스가 여러 형태로 유입되는 경우 그래프 오염/중복 엣지/잘못된 브릿지 판정을 줄인다.

5) 요약/통찰 생성은 Evidence를 1급 입력으로 취급한다.
- `DeepInsightGenerator`는 (선택적으로) edges를 입력으로 받아,
  연결 근거를 사람에게 읽히는 문장으로 변환해 evidence에 포함한다.
- 출력은 다음을 목표로 한다:
  - "A와 B는 [공통 키워드/유사도]로 연결되므로 기술적 연결점이 있음" 형태의 피드백 제공

## Consequences
### Positive
- 결과물의 “설명가능성(Explainability)”이 증가한다.
  - 연구자는 연결을 검증할 수 있고, 엔진은 근거를 축적/비교/재사용할 수 있다.
- 브릿지 노드 기반으로 다학제 연결점 탐색 UX를 강화할 수 있다.
  - 검색 결과 리스트보다 “지형도 상의 연결점”을 우선 제시 가능.
- 타입 단일화로 소스 확장(bioRxiv 등) 시 필드 충돌/호환성 문제가 줄어든다.
- Deduplication으로 그래프 노이즈가 감소하여, bridge 탐지 품질이 상승한다.

### Negative / Trade-offs
- edge payload가 커진다(reason/evidence 추가) → 응답 크기/저장 비용이 증가할 수 있다.
- bridge node 휴리스틱은 초기에는 오탐/미탐이 존재한다(특히 데이터 규모/분포에 민감).
- DOI가 `Paper` 모델에 명시되지 않은 소스의 경우 title 기반 dedup은 false positive 위험이 있다
  (짧은/일반 제목, 약간의 변형 등).

## Alternatives Considered
- (Alt 1) 엣지 근거를 저장하지 않고 weight만 사용:
  - 운영/구현은 단순하지만, 통찰의 핵심인 “왜 연결인가”를 제공할 수 없다.
- (Alt 2) community detection을 초기부터 도입(Louvain/Leiden):
  - bridge 탐지 품질은 좋아질 수 있으나, 의존성/성능/운영 복잡도가 증가한다.
- (Alt 3) 외부 Graph DB 도입(Neo4j 등):
  - 그래프 질의는 강력하지만, 인프라/운영 비용이 커지고 초기 속도 저하 가능.

## Implementation Notes (Non-normative)
- `reason`은 enum-like 문자열로 관리하고, `evidence`는 reason별로 스키마를 점진 확장한다.
  - 예: `shared_datasets`, `shared_methods`, `citation_path`, `contradiction_signal` 등
- 브릿지 점수는 향후 다음 신호로 보강할 수 있다.
  - 서로 다른 `source`(arxiv/biorxiv/patent) 간 연결 가중치 상향
  - 서로 다른 `categories` 간 연결을 우선적으로 브릿지로 판정
  - edge reason의 다양성(여러 reason을 동시에 만족하면 bridge 신뢰도 상승)
- Dedup은 “완전한 정답”이 아니라 그래프 오염 방지를 위한 안전장치이며,
  장기적으로는 DOI/PMID/arXivID 등 표준 식별자 중심의 정규화를 강화한다.