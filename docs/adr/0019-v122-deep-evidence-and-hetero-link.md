# 0019: Deep Evidence & Hetero-Link (The Final Inch)

## Status
Proposed

## Context
v1.2.1에서 pgvector 검색 결과에서 embedding을 회수하여 KNN edge와 bridge_score가 실제로 계산되면서 E2E 루프가 닫혔다.
하지만 EvidenceExtractor는 아직 term-bank substring 매칭 중심이라, 연구자가 검증 가능한 “근거 문장(evidence quotes)”이 부족하고,
분야 간 전이(Transfer)를 우선 노출시키는 가중치 정책도 edge 생성에 직접 반영되어 있지 않았다.

## Decision
1) EvidenceExtractor를 graph_builder에서 분리하여 backend/engine/evidence_extractor.py로 승격한다.
2) evidence를 “문장/구문 기반”으로 강화한다:
   - methodology (method phrases + supporting sentences)
   - dataset (dataset/benchmark mentions + supporting sentences)
   - contradiction/gap (conflict/uncertainty sentences)
3) KNN edge 생성에서 cross-domain(edge 양끝 paper.source 상이) bump를 적용하여 transfer를 가속한다.
4) cache/__init__.py에 fingerprint diff 유틸을 추가해 증분 병합 경로를 표준화한다.
5) InsightEngine 출력에 Evidence-based Research Hypothesis를 포함하여 프로덕션 포맷을 확정한다.

## Consequences
- Missing Link 후보는 “왜 연결인지”를 문장 단위로 제시할 수 있다.
- 단순 허브가 아니라 분야 장벽을 넘는 링크가 상위로 노출될 확률이 증가한다.
- 대규모 수집 환경에서 변경된 노드/엣지 중심의 증분 업데이트가 가능해진다.