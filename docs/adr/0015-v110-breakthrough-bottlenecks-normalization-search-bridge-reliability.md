# 0015: v1.1.0 Breakthrough Bottlenecks (Normalization, Search Scaling, Bridge Discovery, Reliability)

## Status
Proposed

## Context
Atlas는 pgvector 기반 시맨틱 매칭 + 다학제 수집을 통해 "Missing Link"를 제안하는 통찰 엔진을 지향한다.
그러나 데이터/사용량이 증가할수록 다음 병목이 품질과 운영 안정성을 동시에 훼손한다.

1) Entity 중복/표기 불일치로 인한 그래프 오염
- 동일 논문/특허가 소스별로 서로 다른 ID/URL/메타데이터로 들어온다.
- 단순 title/doi 매칭만으로는 중복을 충분히 줄일 수 없고, 누적될수록:
  - 그래프 노드/엣지의 중복 폭발
  - bridge 탐지의 오탐 증가
  - 통찰 요약의 근거 약화

2) 검색 스케일(인덱스/튜닝) 부재
- pgvector `<=>` 검색은 가능하나, 데이터량 증가 시 인덱스 전략이 없으면 성능이 급격히 저하될 수 있다.
- 운영 환경(Neon/서버리스)에서 인덱스 생성/튜닝은 권한/호환성 문제가 있어, "best-effort + env override" 설계가 필요하다.

3) Bridge 탐지의 정교화 필요
- 휴리스틱만으로는 "다른 군집을 잇는 중심 노드"를 안정적으로 찾기 어렵다.
- betweenness centrality는 지식 그래프에서 중개 중심성을 측정하는 표준적 접근이며,
  필요 시 community detection을 더해 "군집 간 연결"을 해석 가능하게 해야 한다.

4) 수집 파이프라인 복구력 부족
- 대량 수집에서 레이트리밋/네트워크 오류가 잦고, 단발성 실패가 전체 분석 실패로 이어진다.
- 배치 수집 + 지수 backoff + jitter가 필요하며, 스크레이퍼 구현체가 pagination을 제공하지 않더라도
  안전하게 "분할 호출 + 조기 종료" 형태를 지원해야 한다.

## Decision
### 0) 기본 정책 (Default Policy; env override 가능)
- Vector index 기본은 IVFFlat, 필요 시 HNSW로 전환 가능:
  - `ATLAS_PGVECTOR_INDEX=ivfflat` (default)
  - `ATLAS_PGVECTOR_INDEX=hnsw` (optional)
- 검색 튜닝은 env로 주입:
  - IVFFlat: `ATLAS_PGVECTOR_PROBES`, `ATLAS_PGVECTOR_IVFFLAT_LISTS`
  - HNSW: `ATLAS_PGVECTOR_EF_SEARCH`, `ATLAS_PGVECTOR_HNSW_M`, `ATLAS_PGVECTOR_HNSW_EF_CONSTRUCTION`
- Community detection은 optional dependency:
  - `ATLAS_COMMUNITY_ALGO=none` (default)
  - `ATLAS_COMMUNITY_ALGO=louvain` 시 python-louvain이 없으면 graceful fallback
- Canonical ID 우선순위:
  - DOI > PMID > arXiv ID > PatentNumber > TitleHash > id
- 수집 복구력:
  - 지수 백오프 + jitter, 기본 max 5회, cap 30s
  - batch size 기본 20

### 1) Entity Normalization: Canonical ID 도입
- 수집 단계에서 각 문서/리소스에 대해 `canonical_id`를 생성한다.
- `canonical_id`는 서로 다른 소스에서 들어온 동일 개체를 단일 노드로 병합하기 위한 “정규화 키”이다.
- 생성 규칙(우선순위):
  - `canon:doi:{doi}` > `canon:pmid:{pmid}` > `canon:arxiv:{arxiv_id}` > `canon:patent:{patent_number}`
  - fallback: `canon:title:{sha1(normalized_title)}` > `canon:id:{id}` > `canon:unknown`

- 정규화와 병합은 "best-effort"로 수행하며, 병합 충돌 시 더 풍부한 summary를 보존한다.

### 2) Types: 정규화 식별자 보존을 위한 PaperMetadata 확장
- `backend/engine/engine_types.PaperMetadata`에 다음을 포함한다:
  - `canonical_id`
  - `doi`, `pmid`, `arxiv_id`, `patent_number`
  - `external_ids: Dict[str, str]`
  - `source`, `source_id`(원본 레코드 정체성)

- `backend/engine/types.py`는 canonical re-export 유지(단일 스키마 강제).

### 3) Scaling Search: pgvector 인덱스 및 튜닝 파라미터
- DB bootstrap 시점(`backend/db/session.bootstrap_db`)에:
  - 기본적으로 IVFFlat 인덱스(best-effort) 생성
  - env가 HNSW를 지정하면 HNSW 인덱스(best-effort) 생성
- 검색 시점(`src/atlas/search/pgvector_search.PgVectorSearchRepository`)에:
  - session-local 튜닝을 best-effort로 적용
    - ivfflat.probes / hnsw.ef_search

### 4) Vector Dimension Stability: 엄격 가드
- `src/atlas/embeddings/utils.py`에서:
  - NaN/inf 제거(sanitize)
  - 단일 차원 정책 제공(target_vector_dim)
  - DB 경계에서 `require_embedding_dim`로 1536 차원 불일치(미세 오차 포함)를 차단

### 5) Advanced Bridge Discovery: betweenness + optional community detection
- `backend/engine/graph_builder.GraphBuilder.mark_bridge_nodes`에서:
  - paper-paper similarity graph를 구성하고,
  - betweenness centrality(weighted)를 계산해 `bridge_score`로 사용한다.
  - optional로 community id를 부여하여 해석 가능성을 높인다.
- community detection은 의존성이 없을 경우 betweenness-only로 fallback한다.

### 6) Reliability: backoff retry + batch ingest
- `AggregationEngine.collect()`에서:
  - `backoff retry`(지수 백오프 + jitter)��� 적용할 수 있도록 한다.
  - `batch_size` 기반 분할 호출을 지원한다.
  - pagination이 불명확한 scraper에서는 “요청량 대비 반환량이 줄면 조기 종료”로 안전성을 확보한다.

## Consequences
### Positive
- 그래프 품질 향상:
  - 동일 개체 병합으로 중복 노드/엣지 감소 → bridge 탐지 및 reason/evidence 품질 상승
- 검색 성능/운영 유연성:
  - 환경별(Neon/권한/버전) 제약을 고려하면서도 성능 튜닝이 가능
- 통찰 신뢰도 상승:
  - betweenness 기반 bridge_score는 휴리스틱보다 안정적인 “중개 중심성” 신호를 제공
- 장애 복구력 증가:
  - 레이트리밋/네트워크 오류가 “부분 실패”로 국한되고 전체 파이프라인 실패를 줄임

### Negative / Trade-offs
- Canonical ID는 완전한 정답이 아니며, title hash는 오탐/미탐 위험이 있다.
- 인덱스 생성/세션 튜닝은 권한/환경에 따라 실패할 수 있어 best-effort로 설계됨(성능이 항상 보장되진 않음).
- betweenness/community는 데이터가 매우 커질 경우 계산 비용이 증가할 수 있다
  (향후 샘플링/서브그래프/오프라인 배치로 전환 고려).

## Alternatives Considered
- (Alt 1) 소스별 분리 인덱스/분리 테이블:
  - 구현은 명확하지만 cross-source 검색/병합이 약해지고 통찰(bridge) 품질이 떨어질 수 있음.
- (Alt 2) 외부 벡터 DB/그래프 DB 도입:
  - 성능/기능은 강력하나 인프라/비용/��영 복잡도 증가.
- (Alt 3) community detection 강제 의존성:
  - 품질은 좋아질 수 있으나, 배포/운영 복잡도가 커져 서버리스 환경에서 리스크 증가.

## Implementation Notes (Non-normative)
- Canonical ID는 long-term에 DOI/PMID/patent number 등 “표준 식별자” 비중을 높이는 방향으로 개선한다.
- bridge_score는 향후 다음 신호로 확장 가능:
  - 이종 노드 타입(Method/Dataset) 추가 후 meta-path 기반 중심성
  - reason 다양성(여러 근거를 동시에 충족하는 엣지)의 가중치 강화
- 대규모 그래프에서 betweenness 계산이 부담되면:
  - top-k 후보 서브그래프만 구성하여 계산
  - 오프라인 배치 작업으로 전환