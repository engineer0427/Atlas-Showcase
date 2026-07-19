# 0017: Closing the Loop (E2E embeddings-to-graph integration)

## Status
Proposed

## Context
v1.2.0에서 Atlas는 pgvector 검색, evidence edge(다차원), bridge discovery(centrality), cache 기반 insight loop를 도입했다.
하지만 실 프로덕션 경로(InsightEngine)에서 pgvector 검색 결과에 임베딩이 포함되지 않아 KNN edge 생성이 수행되지 않았고,
그 결과 bridge_score가 대부분 0으로 귀결되어 'Missing Link' 탐지 루프가 닫히지 않았다.

또한 임베딩 생성 로직이 backend/engine(동기)과 src/atlas(비동기)로 이원화되어 모델/차원 정책이 분산되어 있었다.

## Decision
1) pgvector 검색 결과에 임베딩을 포함하여 InsightEngine이 KNN edge를 생성할 수 있게 한다.
2) backend/engine의 임베딩은 src/atlas EmbeddingClient를 표준으로 참조하는 래퍼로 통합한다.
3) 수집 데이터의 초록/요약에서 Method/Dataset을 best-effort로 추출하여 hetero-graph 노드를 생성한다.
4) cache 레이어에 graph_state 저장/로드를 추가하여 증분 업데이트(Insight Loop)의 기반을 둔다.

## Consequences
- Missing Link(브리지/전이/커뮤니티 경계 연결)의 계산이 실제로 동작한다.
- 모델/차원 정책이 단일화되어 pgvector(Vector(1536))와의 불일치 리스크가 감소한다.
- Method/Dataset 노드가 추가되며 "동일 방법론/데이터셋 공유" 근거를 edge.reason/evidence에 반영할 수 있다.