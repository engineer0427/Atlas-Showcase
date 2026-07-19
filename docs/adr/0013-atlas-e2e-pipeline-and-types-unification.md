# 0013: Atlas E2E Pipeline and Types Unification

## Status
Proposed

## Context
Atlas 코드베이스에는 다음 구성요소가 이미 존재한다.

- 데이터 수집:
  - `src/atlas/data_sources/base.py`의 `BaseScraper` (async `fetch`)
  - `src/atlas/ingest/aggregation_engine.py`의 `AggregationEngine.collect()` (scraper 병렬 실행)
  - `src/atlas/data_sources/arxiv_client.py` (arXiv 수집 구현)

- 임베딩:
  - `src/atlas/embeddings/client.py`의 `EmbeddingClient`(async) + 로컬(sentence-transformers)/OpenAI 선택
  - `backend/engine/embedder.py`는 동기 임베딩 생성 로직을 별도로 가짐

- 검색/저장:
  - `backend/db/models.py`의 `papers` 테이블은 pgvector `Vector(1536)` 컬럼을 보유
  - `src/atlas/search/pgvector_search.py`는 pgvector cosine distance `<=>` 기반 검색을 제공
  - `backend/db/session.py`의 `bootstrap_db()`는 pgvector extension/인덱스 생성을 best-effort로 수행

- 그래프:
  - `src/atlas/graph_store.py` (GraphStore)
  - `backend/engine/graph_builder.py` (임베딩 KNN edge 생성)

그러나 현재 실제 엔드포인트(`/analyze`)에서 호출되는 `src/atlas/ai/pipeline.run_pipeline()`은 `papers=[]` Stub를 반환하여,
수집/DB/검색/그래프 생성이 실제로 연결되지 않는다.

또한 `PaperMetadata`가 아래 두 위치에 중복 정의되어 데이터 계약이 분리되어 있다.

- `backend/engine/types.py` : `PaperMetadata(title, authors, abstract)` (간단 class)
- `backend/engine/engine_types.py` : `PaperMetadata(id, title, summary, authors, published, url, categories, keywords)` (dataclass)

이로 인해 Provider/Ranker/Facade/DB 저장 스키마가 일관되지 않고, 새로운 데이터 소스(bioRxiv 등) 확장 시 타입/필드 충돌 가능성이 크다.

## Decision
1) "논문 단위"의 공통 스키마를 하나로 통일한다.
- `backend/engine/engine_types.PaperMetadata` 형태(식별자/요약/url/published/categories/keywords 포함)를 Canonical로 채택한다.
- `backend/engine/types.py`의 `PaperMetadata`는 폐기(또는 alias/compat)하여 단일 타입 계약을 유지한다.
- Provider가 반환하는 Paper는 Canonical `PaperMetadata`로 표준화한다.

2) 데이터 소스 확장(bioRxiv 등)을 고려한 플러그인 구조를 고정한다.
- Provider 레이어: `backend/engine/providers/*.py` (sync fetch) 를 "API boundary"로 유지한다.
- Scraper 레이어: `src/atlas/data_sources/*` (async fetch) 는 "수집 구현"을 담당한다.
- 향후 bioRxiv 등은 `BaseScraper` 구현을 추가하고, 필요 시 Provider에서 해당 Scraper를 호출하도록 브릿지(동기 래핑)를 추가한다.

3) End-to-End 파이프라인을 실제로 연결한다.
- `run_pipeline(query)`는 다음 단계 중 최소 1개 이상의 실행 경로를 제공해야 한다.
  - (A) providers 기반 수집→임베딩→GraphBuilder(KNN) 그래프 생성
  - (B) DB(pgvector) 기반 검색 결과→랭킹→그래프 생성
- 결과 포맷은 기존 API가 기대하는 `{ papers, graph, raw }`를 유지한다.

4) pgvector 차원(1536) 고정 정책을 코드 레벨에서 강제한다.
- 임베딩 저장/검색 시 `pad_embedding(..., target_dim=1536)`를 공통 적용한다.
- 임베딩 모델이 변경되더라도 DB schema와 검색 품질의 일관성을 유지한다.

## Consequences
Positive:
- `/analyze` 호출 시 Stub가 아닌 실제 데이터/그래프를 반환하는 경로가 확보된다.
- 데이터 소스 확장(bioRxiv, Semantic Scholar, PubMed 등)이 `BaseScraper` 구현 추가로 확장 가능해진다.
- Provider/Ranker/DB 저장 구조가 공통 Paper 스키마에 의해 안정화된다.

Negative / Trade-offs:
- 타입 통일 과정에서 기존 `backend/engine/types.PaperMetadata` 사용자 코드는 수정이 필요하다.
- sync/async 경계(backend Provider vs src Scraper)에서 브릿지 계층이 필요해질 수 있다(추가 유지보수 비용).
- embedding 모델 차원 고정(1536)은 다른 모델 차원의 잠재 이점을 제한할 수 있다(대신 운영 안정성/인덱스 유지가 쉬움).

## Alternatives Considered
- (Alt 1) Stub 유지 + UI 레벨에서만 기능 구현:
  - 서버/DB/그래프 레벨 확장이 불가능해져 장기적으로 불리.
- (Alt 2) DB embedding_dim을 가변으로 운영:
  - 인덱스/검색 연산/마이그레이션 복잡도가 급증.
- (Alt 3) `backend/engine/types.py`를 Canonical로 유지:
  - 최소 필드만 있어 bioRxiv 포함 다양한 소스 통합 시 메타데이터 손실이 커짐.

## Implementation Notes (Non-normative)
- Canonical PaperMetadata 변환 유틸(예: `to_paper_metadata(dict|model)`)를 추가하면 레거시/외부 소스 적응이 쉬워진다.
- 그래프 edge에 `why_link`, `similarity`, `source`(knn/keyword/citation 등) 같은 근거 필드를 attrs로 저장하면 "Missing Link" 설명 가능성이 커진다.