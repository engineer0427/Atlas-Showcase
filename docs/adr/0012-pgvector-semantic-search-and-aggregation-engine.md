# 0012: pgvector 기반 시맨틱 검색 및 멀티 소스 Aggregation Engine 도입

## Status
Proposed

## Context
Atlas 엔진은 다양한 학술/기술 데이터 소스(arXiv, bioRxiv, medRxiv, chemRxiv, SSRN, IEEE Xplore Open Access, Patent 등)를 통합 수집하고,
사용자 질문에 대해 의미 기반(semantic)으로 가장 관련도 높은 결과를 빠르게 반환해야 한다.

기존 구현은:
- 소스별 수집 로직이 분산/단발성으로 존재하거나(arXiv 중심),
- 유사도 계산이 TF-IDF 기반(로컬/실험)으로 되어 있어,
- 데이터가 축적되는 구조(영속 DB + 벡터 검색)로 확장하기 어렵다.

또한, 운영 환경(서버리스/Koyeb/Neon)에서는 “수동 DB 작업”이 배포 안정성을 저해한다.
따라서 애플리케이션 startup 단계에서 DB 확장/테이블 준비를 자동화하여, 배포 시점의 불확실성을 최소화할 필요가 있다.

## Decision
1) Vector DB 계층으로 PostgreSQL + pgvector를 채택한다.
- 논문/특허 등 텍스트 리소스를 1536차원 임베딩으로 저장한다.
- 유사도 검색은 pgvector cosine distance 연산자를 사용한다.
- 임베딩 모델은 OpenAI `text-embedding-3-small`(1536 dims)을 기준으로 한다.

2) 엔진 수집 파이프라인은 Strategy Pattern을 사용한다.
- `BaseScraper` 추상 클래스를 정의한다.
- 플랫폼별 수집기는 `BaseScraper`를 상속하여 독립 모듈로 관리한다.
- `AggregationEngine`이 `asyncio.gather`로 모든 수집기를 병렬 실행한다.

3) 시맨틱 검색 API 엔드포인트를 백엔드에 제공한다.
- 사용자 질문을 임베딩한 뒤, DB에 저장된 리소스(논문+특허 등)와의 유사도를 계산해 Top-K(기본 5개)를 반환한다.
- 서로 다른 소스가 결과에서 자연스럽게 섞이도록, 단일 테이블(papers) 내 `source` 필드로 타입을 구분하고 동일한 검색 경로를 탄다.

4) 운영 편의성을 위해 DB 초기화의 자동화를 목표로 한다.
- 서버 startup 시점에 `CREATE EXTENSION IF NOT EXISTS vector;` 를 실행한다.
- (선택) 마이그레이션 체계(Alembic)가 없을 경우, 초기 1회에 한해 테이블 생성까지 자동화(`metadata.create_all`)하는 방향을 허용한다.
  단, 이는 “스키마 변경 관리”의 엄격성이 약해질 수 있으므로 이후 Alembic 도입 시 단계적으로 제거/전환한다.

## Consequences
### Positive
- 의미 검색의 핵심을 DB 레이어(pgvector)로 내리면서, 데이터 규모가 커져도 성능/확장성이 유지된다.
- 소스 추가(bioRxiv 등)가 `BaseScraper` 구현 추가로 수렴되어 유지보수성이 증가한다.
- `AggregationEngine`의 병렬 수집으로 전체 수집 시간이 단축된다.
- 배포 환경에서 “수동 DB 준비 작업”을 최소화하여 운영 안정성이 향상된다.

### Negative / Trade-offs
- pgvector 의존성(확장/패키지/인덱스 튜닝)이 추가되어 운영 복잡도가 증가한다.
- startup 단계에서 extension/table 생성을 수행할 경우, 권한/락/레이턴시 이슈가 발생할 수 있다.
- OpenAI 임베딩 사용 시 비용/레이트리밋/키 관리가 필요하다.

## Alternatives Considered
- 외부 벡터 DB(Pinecone, Weaviate 등):
  - 운영 분리 및 성능 장점이 있으나, 추가 인프라/비용/네트워크 홉이 증가.
- 로컬 임베딩 모델(sentence-transformers 등):
  - 비용 절감 가능하나, 서버리스 환경에서 모델 로딩/메모리/콜드스타트가 부담.
- TF-IDF 기반 유사도(기존 방식 유지):
  - 구현 단순하지만, 의미 검색 품질/확장성 측면에서 한계가 명확.

## Implementation Notes
- `papers.embedding`은 `Vector(1536)`로 정의한다.
- 소스 구분은 `papers.source`(arxiv/biorxiv/medrxiv/chemrxiv/ssrn/ieee/patent 등)로 한다.
- 검색은 cosine distance 기반으로 Top-K를 반환한다.
- DB 초기화는 startup hook에서 수행한다(확장/테이블/인덱스 자동화 포함 여부는 운영 안정성에 따라 단계적으로 조정).

## Follow-ups
- Alembic 마이그레이션 도입 및 `vector` 인덱스(ivfflat/hnsw) 튜닝.
- 수집 파이프라인의 실 구현(IEEE Open Access/Patent API)과 에러/레이트리밋/캐싱 정책 확정.
- 임베딩 배치 처리 및 재시도(큐/백그라운드 작업) 도입.