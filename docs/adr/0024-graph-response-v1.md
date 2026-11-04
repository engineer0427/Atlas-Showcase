# 0024: 분석 화면용 그래프 응답 형식 graph-v1

## Status
Accepted (2026-10-24)

> 0023 은 "로컬 임베더(sentence-transformers) 지원" 후보 번호로 남겨 두었다.

## Context
- `/analyze` 화면은 응답 JSON 을 그대로 보여 줄 뿐 그래프를 그리지 않았다 (문제 목록 A-3, `docs/strategy.md` 9절).
- 엔진(`src/atlas/insight_engine.py` `InsightEngine.analyze`, `run_pipeline` 이 그대로 반환)의 응답을 확인한 결과 (2026-10-24, 코드 읽기):
  - `papers[]` 에 문서마다 1536차원 `embedding` 이 그대로 들어 있어 화면으로 보내기에 크다.
  - `graph.nodes` 에는 논문 노드만 있고, 질의와의 유사도(`papers[].score`)는 노드에 붙어 있지 않다.
  - `graph.edges` 에는 논문–논문 엣지와 함께 논문→방법·데이터셋·저자·특허 엣지가 섞여 있는데, 뒤쪽 엣지의 끝점 ID(`paper:…`, `canon:…`)는 `graph.nodes` 에 없다. 그대로 그리면 끊어진 선이 된다.
  - `raw` 에는 캐시 경로 같은 내부 정보가 들어 있다.
- 화면 사용자는 코드를 모르는 사람을 기준으로 한다. 코사인 유사도 값은 관련 있는 문서도 0.4~0.6 이 흔해서, 숫자를 크게 보여 주면 오해하기 쉽다.

## Decision
`POST /api/analyze` 는 `{"graph": <graph-v1>}` 를 돌려준다. 변환은 `backend/routes/graph_view.py` 의 `to_graph_v1(engine_result, query)` 가 맡는다.

```
{
  "version": "graph-v1",
  "query": "<질의>",
  "nodes": [{
    "id", "title", "kind": "paper" | "patent" | "document", "year", "url",
    "snippet": "초록 앞부분 (최대 300자)",
    "similarity": 0~1,          // 코사인 유사도. 화면에는 작게만 표시
    "rank": 1..,                // 이번 결과 안에서 질의와 가까운 순서
    "tier": "closest" | "close" | "related",
    "is_closest": bool,
    "is_bridge": bool,
    "group": 커뮤니티 번호 | null
  }],
  "edges": [{
    "source", "target", "weight": 0~1,
    "reason": "semantic_similarity" | "shared_keywords" | "methodological_overlap"
              | "dataset_protocol_sharing" | "cross_citation_path" | "contradiction_signal",
    "reason_label": "쉬운 한국어 설명",
    "evidence": ["사람이 읽을 수 있는 근거, 최대 3개"]
  }],
  "bridges": ["노드 id"],
  "meta": { "documents": n, "connections": m, "note": "검토를 돕는 자료이며 특허 가능성이나 침해에 대한 법적 판단이 아님" }
}
```

- **노드**: `graph.nodes` 의 논문 노드. `similarity`·`rank` 는 `papers[].score` 를 `source:source_id` 로 맞춰 붙인다.
- **단계(tier)**: 순위 기반. 1위 = closest, 2~5위 = close, 6위 이하 = related. 코사인 절댓값의 고정 임계값을 쓰지 않는다 (값의 수준이 질의·분야마다 달라서). 단계는 **이번 결과 안에서의 상대적인 순서**이며 관련성의 절대적인 정도가 아니다. 화면에도 "이번 결과 안에서" 라고 드러낸다.
- **엣지**: 양 끝이 모두 노드 목록에 있는 엣지만 (= 논문–논문). `reason_label` 은 정해진 표로 바꾸고, `evidence` 는 공유 단어·겹치는 방법·데이터·인용문 중 최대 3개.
- **보내지 않는 것**: `embedding`, `raw`, 가설·전이 엣지(`evidence_based_research_hypothesis`, `transfer_edges_top5`).

## Consequences
- 화면은 graph-v1 하나만 알면 된다. 엔진 내부 구조가 바뀌어도 변환 함수만 고친다.
- 응답 크기가 줄고 내부 정보(캐시 경로 등)가 밖으로 나가지 않는다.
- **이종 연결(논문 ↔ 특허 ↔ 방법·데이터셋) 표시는 graph-v2 과제다.** Atlas 의 차별점이므로 v1 에서 뺀 것이지 포기한 것이 아니다. graph-v2 에서는 엔터티 노드를 응답에 넣고, 엔진의 엣지 ID 공간(`source:source_id` 와 `paper:…`·`canon:…`)을 하나로 맞춘다.
- `snippet`(초록 일부)과 인용문 근거는 출판사 저작권이 있을 수 있다. 개발·로컬에서는 쓰되, 라이브 공개 전에 이용 조건을 다시 확인한다 (`docs/strategy.md` 9절).
- 절대적인 관련성 판단(예: 질의와 코퍼스가 맞지 않을 때 "관련 문서 없음" 표시)은 v1 에 없다. 나중 과제.

## 갱신 (2026-10-25, ADR 0025)
- `is_bridge` 는 "다리 후보"다 (v1 휴리스틱, 정답 기준 검증 전). 판정 기준은 ADR 0025.
- `reason` 에 `contradiction_signal` 은 더 이상 나오지 않는다. 연결 이유는 두 문서가 실제로 함께 가진 근거가 있을 때만 보이고, 없으면 `semantic_similarity` 다.
- `evidence` 에 인용문(한쪽 초록의 문장)은 넣지 않는다. 공유 단어·겹치는 방법·데이터·실험 절차·함께 인용하는 문헌 수만 넣는다.
- `group` 은 이제 커뮤니티 번호가 채워진다.

## 갱신 (2026-10-25, 화면 ui-v1)
- `edges[].shared_terms`: 그 연결의 공유 단어 목록 (엔진 `evidence.shared_keywords` 그대로, 키니스 순·최대 5개, ADR 0025). 화면의 단어 칩·강조에 쓴다.
- `groups[]`: `{ "id": 묶음(커뮤니티) 번호, "size": 문서 수, "top_terms": 대표 단어 최대 2개 }`. 대표 단어는 묶음 안쪽 연결(양 끝이 같은 묶음)의 `shared_terms` 를 모아 많이 나온 순(동점은 알파벳). 안쪽 연결이 없으면 빈 목록, 묶음 번호가 없으면 `groups: []`.
- `meta.groups`: 묶음 수.
- 기존 필드와 이름은 바뀌지 않았다. 화면은 새 필드가 없는 옛 응답도 읽는다.
- (같은 날 보완, 최종) `groups[].top_terms` 규칙:
  1. 후보는 처음처럼 묶음 안쪽 연결(양 끝이 같은 묶음)에서 많이 나온 단어 순 (동점은 알파벳). 단, 안쪽 연결 **2개 이상**에서 나온 단어만.
  2. 여러 묶음이 같은 단어를 고른 경우에만, 그 단어를 더 많이 쓰는 묶음에 주고(동점은 묶음 번호가 작은 쪽) 다른 묶음은 다음 후보로 넘어간다. 겹치지 않으면 1번 결과 그대로.
  3. 후보가 모자라면 1개나 0개 (0개면 화면은 "묶음 N" 만).
  - **이유**: 처음 규칙(빈도 순)은 두 묶음이 같은 대표 단어를 갖는 경우가 질의 8개 중 4개(chen·mee 의 filter, see 의 radar, komorowski 의 learning)였다. 이를 고치려고 잠시 쓴 "다른 묶음 대비 드문 단어" 규칙(키니스)은 겹치지 않던 질의까지 8개 모두의 단어를 바꿨고(예: costa2002 의 heart, rate → analysis, healthy), carlo·providing 같은 약한 단어를 냈다. 그래서 문제가 된 겹침만 고치고 나머지는 처음 규칙을 유지한다. 안쪽 연결 1개에서만 나온 단어는 우연일 수 있어 대표로 쓰지 않는다 (hopfield1984 묶음 2 의 benchmark·choice 가 이 경우라 "묶음 2" 만 보인다).
  - 결과 [확인: 진단 도구, 질의 8개]: 겹치던 4개 질의는 겹친 단어만 바뀌고, 나머지 4개는 처음 규칙과 같다 (hopfield 묶음 2 제외). 두 묶음이 같은 대표 단어를 갖는 경우 0.

## 갱신 (2026-10-31, 부분 실패 표시)
- `meta` 에 필드 두 개를 더했다. 기존 필드와 최상위 형식은 그대로다.
  - `meta.degraded`: 계산에 실패해 비어 있거나 기본값으로 대신한 부분의 이름 목록. 실패가 없으면 `[]`. 같은 이름은 한 번만 나온다.
    - `paper_paper_edges`: 논문–논문 연결 계산 실패 (연결 0개)
    - `bridges`: 다리 후보(매개 중심성) 계산 실패
    - `groups`: 묶음(커뮤니티) 계산 실패
    - `deep_evidence`: 연결 근거의 인용 문장 추출 실패
    - `edge_reasons`: 연결 이유 고르기 실패 (처음 이유를 그대로 둠)
    - `corpus_word_counts`: 공유 단어 순위용 코퍼스 빈도 조회 실패 (결과 안 빈도로 대신)
    - `search_tuning`: 검색 정확도 설정(`SET LOCAL hnsw.ef_search` 등) 실패 (기본값으로 검색)
    - `cache`, `graph_state`: 서버 캐시·그래프 상태 저장 실패 (응답 내용에는 영향 없음)
  - `meta.skipped_documents`: 검색은 됐지만 노드를 만들지 못해 뺀 문서 수. 없으면 `0`.
- **이유**: 엔진이 이런 실패를 조용히 삼켜, 연결 0개가 "연결 없음"인지 "계산 실패"인지 구분할 수 없었다 (`reports/2026-10-31-예외삼킴-정리.md`). 엔진은 경고 로그도 남긴다.
- 화면에서 이 값을 어떻게 보일지는 아직 정하지 않았다. 지금 화면은 이 필드를 읽지 않는다.
- 엔진 결과에서 쓰는 곳이 없던 `transfer_edges_top5` 를 없앴다. graph-v1 에는 원래 들어가지 않던 값이라 응답은 같다.
