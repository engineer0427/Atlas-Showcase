# ADR-0021: Hetero Evidence & Insight Loop (v1.2.0 → v1.4.0 Hardening)

## Status
Proposed

## Context
Atlas는 v1.1~v1.3에서 pgvector 기반 검색, 다차원 Evidence, Bridge 탐지, Hetero-graph/Transfer prioritization의 기반을 구축했다.
하지만 상용 환경에서는 다음이 결정적으로 필요하다.

1) DB 스키마가 코드/수집 로직과 물리적으로 동기화되어야 한다.
- `papers` 테이블에 hetero 필드(method_terms, dataset_terms, references, citations, authors, patent_number)가 존재해야 한다.
- ingest 단계에서 생성된 hetero/citation 정보가 DB에 저장되어 검색/그래프/인사이트 루프에 재사용되어야 한다.

2) Hetero-graph 가중치/전이(transfer) 우선순위가 엔드투엔드로 반영되어야 한다.
- Evidence 4종(Method/Dataset/Citation/Contradiction)이 edge에 구조화되어 저장되어야 한다.
- GraphStore가 타입별/관계별 가중치로 edge를 우선순위화하고, 상단에 transfer edges를 노출할 수 있어야 한다.

3) Insight 상태(Bridge Score/Community 등)가 누적/부분 갱신되어야 한다.
- 같은 스코프(쿼리/프로젝트)에서 반복 실행 시, fingerprint 기반으로 바뀐 노드/엣지만 업데이트해야 한다.
- 캐시는 파일 기반을 유지하되, 향후 Redis/S3로 대체 가능한 인터페이스 형태를 유지한다.

4) 프로덕션 엔진 경계(단일 진입점)를 명확히 해야 한다.
- TF-IDF 등 레거시 경로는 제거하고 pgvector + hetero graph 기반 분석이 단일 경로로 동작한다.

## Decision
1) DB 스키마(physical)를 명시적으로 반영한다.
- ALTER TABLE 구문을 제공하고, ingest/upsert 경계를 `PgVectorRepository`로 통일한다.
- JSON 계열 필드는 DB에서는 JSONB가 이상적이나, 현재 프로젝트 제약(마이그레이션 프레임워크 부재/호환성)을 고려하여 기본은 TEXT(JSON-string)로 저장하되, 읽기 시 항상 list 형태로 복원(best-effort)한다.

2) PgVectorRepository를 단일 DB 경계로 정의한다.
- search (KNN) + upsert (paper row 저장/업데이트) + schema bootstrap(ALTER SQL 제공/선택 실행)을 한 곳에 둔다.

3) Evidence와 Transfer 우선순위는 GraphBuilder → GraphStore로 단일 흐름으로 전달한다.
- GraphBuilder는 evidence 4종을 edge.evidence에 저장하고, GraphStore.edge_priority_score에 필요한 필드를 제공한다.
- Transfer는 (a) cross-type, (b) cross-source, (c) evidence strength로 점수화한다.

4) InsightCache는 node/edge fingerprint 기반 부분 갱신을 제공한다.
- bridge/community 같은 derived state는 fingerprint 변화가 없으면 재계산/저장을 피한다.

## Consequences
- Positive:
  - DB/ingest/search/graph가 물리적으로 결합되어 “빈 hetero 필드” 문제를 제거한다.
  - transfer edges를 안정적으로 상단 노출 가능.
  - 폭증 데이터 환경에서 캐시 기반 부분 갱신으로 비용/메모리를 절감한다.
- Trade-offs:
  - TEXT(JSON-string) 저장은 JSONB 대비 쿼리성이 낮다. 추후 마이그레이션으로 JSONB 전환을 권장한다.
  - best-effort 추출(방법/데이터셋/인용)은 소스 품질에 영향을 받는다.