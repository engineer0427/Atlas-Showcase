# 0016: Multidimensional Evidence & Insight Loop (v1.2.0)

## Status
Proposed

## Context
v1.1.0에서 Atlas는 pgvector 기반 검색, canonical id 정규화, evidence edge(기초), bridge 탐지(중개 중심성)를 갖춘 '통찰 인프라'로 정리되었다.
하지만 'Missing Link'를 연구 가설 수준으로 끌어올리려면 단일 축(semantic similarity/shared keywords) 근거만으로는 부족하다.

또한 지식이 기하급수적으로 증가하는 환경에서 통찰은 단발 분석이 아니라, 반복적으로 업데이트되는 '지형도(knowledge landscape)' 형태로 누적되어야 한다.

## Decision
1) Evidence를 다차원으로 구조화한다.
- Edge.evidence에 다음 구조적 근거를 best-effort로 추가한다.
  - methodological_overlap (method_terms, score)
  - dataset_protocol_sharing (dataset_terms, protocol_terms, score)
  - cross_citation_path (shortest_path_len, overlap_score, signals)
  - contradiction_signal (flags, score)

2) Insight Loop(증분 업데이트)를 도입한다.
- 동일 쿼리/동일 canonical entity를 반복 분석할 때:
  - 기존 노드/엣지의 insight_state를 캐시에 저장하고,
  - 새 데이터가 추가/변경된 부분만 fingerprint 기반으로 갱신한다.

3) Hetero-graph를 1st-class로 승격한다.
- Paper 외에 Method/Dataset/Author/Patent 노드 타입을 엔진 계약(types)에서 수용한다.
- GraphStore는 타입별 가중치 + 분야 간 전이(Transfer) 가능성이 높은 엣지를 우선순위화한다.

4) 엔진 진입점을 단일화한다.
- TF-IDF 기반 MVP 파이프라인은 프로덕션 경로에서 제거/대체하고,
  pgvector/Evidence/Bridge 기반 분석이 단일 엔트리로 동작하도록 정리한다.

## Consequences
- Positive:
  - 연결의 근거가 다변화되어 연구자가 "왜 연결인가"를 더 강하게 검증할 수 있다.
  - 증분 업데이트로 폭증 지식 환경에서 지속 가능한 '지형도 갱신'이 가능해진다.
  - hetero-graph로 전이 가능한 연결(방법→분야, 데이터셋→분야)을 더 명확히 포착한다.
- Trade-offs:
  - method/dataset/citation/contradiction 추출은 소스 메타데이터 품질에 의해 제한되며 best-effort로 시작한다.
  - 캐시 무결성/버전 관리가 필요하다(캐시 스키마 버전 도입).

## Implementation Notes
- EvidenceExtractor는 외부 의존성 없이 정규식+사전 기반으로 시작하되, 추후 LLM/NER/코드-인용 연계로 확장 가능.
- InsightCache는 JSON 파일 기반(로컬/서버리스 디스크) + 향후 Redis/S3로 교체 가능하도록 인터페이스로 추상화한다.