# 0022: Physical Schema Sync & paper_references Rename

## Status
Proposed

## Context
Atlas 엔진은 v1.2.0~v1.4.x 과정에서 아래 기능을 중심으로 고도화되었다.

- pgvector 기반 검색 및 임베딩 기반 KNN 그래프(edge) 구성
- 다차원 Evidence(방법론/데이터셋·프로토콜/상호인용/모순 신호) 추출 및 edge.evidence 구조화
- Insight Loop(증분 업데이트)를 통해 지식 지형도를 반복적으로 갱신

그러나 운영 DB의 `papers` 테이블이 base 컬럼만 존재하는 경우(예: source/source_id/title/abstract/url/published/embedding 등),
엔진이 기대하는 구조화 필드가 물리적으로 저장되지 못해 다음 문제가 발생한다.

- 수집/업서트 후에도 hetero/evidence-loop 필드가 DB에 누적되지 않아 재분석 시 근거가 축적되지 않는다.
- DB 기반 검색 결과를 사용할 때 구조화 필드가 없어(또는 컬럼 자체가 없어) fallback 경로로만 동작하여,
  “지속 가능한 통찰 루프”의 완성도가 떨어진다.

또한 `references`라는 컬럼명은 SQL 문맥 및 일부 ORM/툴링에서 예약어/키워드 충돌(가독성 저하 및 유지보수 리스크)을 유발할 수 있다.
장기 운영 및 소스 확장(bioRxiv 등)을 고려하면 충돌 가능성이 있는 명칭을 회피하는 편이 안전하다.

## Decision
1) `papers` 테이블의 물리 스키마를 엔진 계약에 맞게 확장한다.
- 아래 컬럼을 `papers`에 물리적으로 추가하여, 수집→DB 업서트→검색→그래프 구축 흐름에서 구조화된 근거가 누적되도록 한다.
  - `method_terms` (TEXT, NOT NULL, default '[]')
  - `dataset_terms` (TEXT, NOT NULL, default '[]')
  - `paper_references` (TEXT, NOT NULL, default '[]')
  - `citations` (TEXT, NOT NULL, default '[]')
  - `authors` (TEXT, NOT NULL, default '[]')
  - `patent_number` (VARCHAR, NULL)

2) 예약어 충돌 방지를 위해 `references` 명칭을 `paper_references`로 변경한다.
- DB 컬럼명과 코드 내 key/필드명을 모두 `paper_references`로 정렬한다.
- 호환성을 위해(점진 배포/부분 반영 환경) 런타임에서는 일정 기간 schema-guard(try/except) 및 구명(alias) 로직을 유지할 수 있다.

3) 저장 포맷은 1차적으로 TEXT(JSON string)로 유지한다.
- 마이그레이션 프레임워크 의존 없이 즉시 적용 가능하도록 리스트형 필드들을 TEXT에 JSON 문자열로 저장한다.
- 추후 질의성/성능을 위해 JSONB로의 마이그레이션을 고려한다.

## Consequences
- Positive
  - 엔진이 생성하는 구조화 근거가 DB에 누적되어, 반복 분석 시 Insight Loop가 현실적으로 동작한다.
  - `references` 명칭 충돌 리스크를 제거하여 유지보수성과 툴링 호환성이 개선된다.
  - bioRxiv 등 신규 소스 추가 시에도 동일 스키마/필드 계약으로 확장 가능하다.

- Trade-offs
  - TEXT(JSON string) 저장은 JSONB 대비 서버 사이드 질의/인덱싱이 불리하다.
  - 컬럼 rename/추가를 포함하므로 배포 시점에 DB 스키마 반영 작업이 반드시 필요하다.

## Implementation Notes (Non-normative)
- DB 스키마 반영은 `ALTER TABLE papers ...`로 수행한다.
- 기존 `references` 컬럼이 이미 존재한다면, 데이터 보존을 위해 `paper_references`로 rename하는 방식을 우선한다.