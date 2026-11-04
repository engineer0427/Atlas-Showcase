# 설계 결정 기록 (ADR)

Atlas 의 설계 결정을 번호 순서대로 기록한다. 공개 저장소에는 27건 중 26건을 싣는다.

- **비공개**: 사업 관련 결정이라 싣지 않은 기록이다. 번호는 비워 두지 않고 표에 남긴다.
- **[비공개: …]**: 본문 일부를 가린 곳이다. 운영 주소, 개발 환경 이름, 현재 보안 상태, 사업 관련 내용을 가렸다.
- 상태는 각 기록에 적힌 그대로다. 그 뒤 결정이나 코드 정리로 바뀐 것은 요약 끝에 "→" 로 적었다.
- 0001~0022 의 효과 서술은 당시의 기대이며 측정으로 확인한 값이 아니다.

| 번호 | 제목 | 상태 | 한 줄 요약 |
|---|---|---|---|
| [0001](0001-provider-pattern.md) | Provider 패턴과 DeepInsight | Proposed / Accepted | 문헌 저장소별 수집을 Provider 로 나누고 군집 단위 인사이트를 만든다 → 해당 코드는 2026-10 정리 때 삭제 |
| [0002](0002-fix-import-and-type-errors.md) | 배포 환경 ImportError·NameError 수정 | Accepted | Provider 패키지화와 타입 명시로 배포 환경 import 오류를 없앤다 → Provider 코드는 2026-10 삭제 |
| 0003 | (비공개) | — | 비공개 |
| [0004](0004-separate-3d-graph-rendering_Version2.md) | 3D 그래프 클라이언트 렌더링 | Accepted | 3D 그래프를 클라이언트 전용 컴포넌트로 분리해 SSR 오류를 없앤다 → 0024 의 2D 지도로 대체 |
| [0005](0005-docker-image-optimization.md) | Docker 이미지 최적화 | Accepted | 배포 플랫폼의 이미지 크기 한도에 맞게 slim 이미지·.dockerignore 를 쓴다 |
| [0006](0006-docker-image-optimization-v2.md) | Alpine·멀티 스테이지 빌드 | Accepted | 이미지를 더 줄이려고 Alpine 과 멀티 스테이지 빌드를 시도한다 → 0009 에서 slim 으로 되돌림 |
| [0007](0007-python-version-and-pip-upgrade.md) | Python 버전과 pip 업그레이드 | Accepted | 패키지 호환을 위해 Python·pip 버전을 올린다 → 지금 운영 이미지는 python:3.10-slim |
| [0008](0008-torch-compatibility-and-python-version.md) | torch 호환성과 Python 버전 | Accepted | sentence-transformers 의 torch 의존성 충돌을 정리한다 → torch 는 이후 운영 의존성에서 뺌 |
| [0009](0009-using-python-slim-for-dependency-compatibility.md) | Python slim 이미지로 전환 | Accepted | Alpine 에서 torch 설치가 실패해 slim 이미지로 바꾼다 |
| [0010](0010-update-torch-version.md) | torch 버전 업그레이드 | Accepted | torch 를 CPU 전용 빌드로 올려 설치 실패를 해결한다 → torch 는 이후 운영 의존성에서 뺌 |
| [0011](0011-cors-and-authentication-settings-update.md) | CORS 와 인증 처리 수정 | 채택됨 | 프런트–백엔드 간 CORS 와 인증 요청 오류를 정리한다 |
| [0012](0012-pgvector-semantic-search-and-aggregation-engine.md) | pgvector 의미 검색과 수집 엔진 | Proposed | PostgreSQL + pgvector 로 1536차원 임베딩 검색을 한다 → 인덱스는 2026-10 HNSW·halfvec 으로, 수집 방식은 0028 로 |
| [0013](0013-atlas-e2e-pipeline-and-types-unification.md) | 엔드투엔드 파이프라인과 타입 통일 | Proposed | 수집·임베딩·검색·그래프를 한 경로로 잇고 논문 타입을 하나로 맞춘다 |
| [0014](0014-evidence-edges-and-bridge-node-detection.md) | 근거 있는 연결과 다리 노드 탐지 | Proposed | 연결마다 이유와 근거를 저장하고, 묶음을 잇는 문서를 표시한다 → 다리 판정은 0025 로 |
| [0015](0015-v110-breakthrough-bottlenecks-normalization-search-bridge-reliability.md) | 정규화·검색 확장·다리 탐지·복구력 | Proposed | 정규화 식별자, 벡터 인덱스 설정, 매개 중심성 기반 다리 점수, 재시도를 도입한다 → 인덱스 기본은 2026-10 HNSW 로 |
| [0016](0016-v120-multidimensional-evidence-and-insight-loop.md) | 다차원 근거와 증분 갱신 | Proposed | 방법·데이터·인용 근거를 구조화하고 바뀐 부분만 다시 계산한다 |
| [0017](0017-v120-cache-package-layout-fix.md) | 캐시 패키지 구조 수정 | Proposed | 캐시 모듈을 패키지로 바꿔 import 충돌을 없앤다 |
| [0018](0018-v121-closing-the-loop-e2e-embeddings-to-graph.md) | 검색 결과 임베딩으로 그래프 연결 | Proposed | 검색 결과에 임베딩을 담아 연결 계산이 실제로 돌게 한다 |
| [0019](0019-v122-deep-evidence-and-hetero-link.md) | 문장 단위 근거와 이종 연결 | Proposed | 근거를 문장 단위로 강화하고 분야가 다른 연결에 가중치를 둔다 |
| [0020](0020-hetero-graph-and-knowledge-transfer-v130.md) | 이종 그래프와 분야 간 전이 | Proposed | 방법·데이터셋·저자·특허 노드와 이종 연결을 그래프에 넣는다 |
| [0021](0021-v120-hetero-evidence-insight-loop.md) | 이종 근거와 증분 갱신 정비 | Proposed | DB 스키마·근거·전이 우선순위·캐시 갱신을 한 흐름으로 맞춘다 |
| [0022](0022-physical-schema-sync-and-paper-references-rename.md) | 물리 스키마 동기화와 컬럼 이름 변경 | Proposed | papers 표에 구조화 필드를 더하고 references 를 paper_references 로 바꾼다 |
| 0023 | (미작성) | 후보 | 로컬 임베더 지원 — 번호만 잡아 두었다 |
| [0024](0024-graph-response-v1.md) | 분석 화면 그래프 응답 형식 graph-v1 | Accepted | 화면이 쓰는 그래프 응답 형식을 정하고 내부 정보는 보내지 않는다 |
| [0025](0025-bridge-candidate-and-edge-reason-v1.md) | 다리 후보 판정 v1 과 연결 이유 원칙 | Accepted | 묶음을 잇는 문서 후보 규칙을 정하고, 실제 공유 근거가 있는 연결 이유만 보인다 (검증 전 휴리스틱) |
| [0026](0026-access-rule-view-anyone-analyze-after-login.md) | 접근 규칙: 보기는 누구나, 분석은 로그인 후 | Accepted | 화면에서 막는 것은 보호가 아니며, 보호는 백엔드에서 한다 |
| [0027](0027-element-comparison-elements-v1.md) | 구성 요소 대조 elements-v1 | Accepted (개발 플래그 안 프로토타입) | 아이디어를 요소로 나눠 문헌과 대조하고, 근거는 초록 원문 인용만 서버가 검사한다 |
| [0028](0028-literature-collection-jit-first.md) | 운영 문헌 수집: 그때그때 모으기 + 참고문헌 따라가기 | Accepted | 150편 정답지로 측정해 미리 쌓기를 보류하고, 질의마다 모으고 참고문헌을 따라가는 방식을 택한다 |

- 다음 번호 후보: 0029 (halfvec·HNSW 전환), 0030 (외부 AI 로 아이디어를 보내는 기준). 아직 쓰지 않았다.
