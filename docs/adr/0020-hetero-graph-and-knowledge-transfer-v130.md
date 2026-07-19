# 0020: Hetero-Graph & Knowledge Transfer (v1.3.0)

## Status
Proposed

## Context
v1.2.x에서 Atlas는 pgvector 후보군에 대해 paper-paper KNN 그래프를 구성하고, evidence/bridge_score를 생성해 통찰 루프를 닫았다.
하지만 (1) hetero node(Method/Dataset/Author/Patent)가 실제 그래프 파이프라인에 주입되지 않아 전이(Transfer)가 점수 휴리스틱에 머물렀고,
(2) references/citations가 placeholder라 cross_citation_path가 실질적으로 작동하지 않았다.

## Decision
1) InsightEngine은 pgvector 결과(Paper dict)에서 method/dataset/author/patent 를 독립 노드로 분리해 GraphStore에 주입한다.
2) Paper-Paper 뿐 아니라 Paper-Method, Method-Dataset, Paper-Patent, Paper-Author 등 이종 엣지를 자동 생성한다.
3) GraphBuilder.mark_bridge_nodes()는 hetero-graph 기반으로 확장하고, cross-type edge에 Transfer Weight를 부여해 “학문 간 장벽을 넘는 브릿지”를 우선 탐지한다.
4) AggregationEngine은 같은 배치 내에서 best-effort citation graph를 매핑하여 references/citations 필드를 채운다.
5) Summarizer는 top_transfer_edges(top5)를 기반으로 Transfer Suggestion(구체적 행동 제안) 섹션을 최종 리포트 포맷에 포함한다.

## Consequences
- Positive:
  - 전이가 점수 조정이 아니라 그래프 구조 자체로 표현된다(hetero nodes + hetero edges).
  - cross_citation_path가 실제 데이터를 기반으로 동작하여 ‘지식의 계보’를 추적 가능해진다.
  - 연구자는 “어떤 방법론을 어떤 분야로 옮겨 실험해야 하는지”를 행동 제안으로 받는다.
- Trade-offs:
  - 노드/엣지 수가 증가하므로 KNN 후보(top_k) 및 hetero 확장 정책(제한/상한)이 중요해진다.
  - best-effort citation 매핑은 false positive 가능성이 있어 추후 외부 citation API(OpenAlex/S2/Crossref 등) 연동이 필요하다.