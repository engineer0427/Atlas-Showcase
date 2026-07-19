# ADR-0001: Provider 패턴 도입 & DeepInsight(심층 비판적 인사이트) 도입 

- Date: 2025-12-31  
- Status: Proposed / Accepted  
- Authors: engineer0427

## Context
EngineFacade가 특정 저장소(arXiv)에 직접 의존하고 있고, summarizer는 대표 논문 기반의 단순 요약만 제공합니다. 상용화(프로 서비스)를 위해서는 여러 저장소(bioRxiv, SSRN 등) 확장성, 저장소별 정규화·중복 제거, 그리고 유료 사용자에게 제공할 "심층 비판적 인사이트(DeepInsight)"가 필요합니다.

## Decision
1. Provider 패턴 도입
   - 각 저장소별 로직을 `backend/engine/providers/` 내 Provider 클래스로 분리한다.
   - `BaseProvider`(Protocol)를 정의하고, `ArxivProvider` 등 Provider가 `fetch(query, limit)`를 구현하도록 한다.
   - 활성 Provider 목록은 설정(예: ATLAS_PROVIDERS="arxiv,biorxiv")으로 관리하여 Facade 코드 수정 없이 확장 가능하게 한다.

2. DeepInsightGenerator 도입 (summarizer 확장)
   - 군집(cluster) 단위로 구조화된 JSON 형식의 심층 인사이트 반환:
     - technical_gap: 기존 연구가 미해결한 핵심 지점
     - critical_limitations: 방법론·데이터의 주요 한계
     - practical_application: 즉시 실무/타학문 적용 가능성
     - confidence, evidence, generated_at 등 메타 포함
   - 동작 방식:
     - OPENAI_API_KEY 등 LLM 설정이 있으면 LLM(few-shot, JSON-only)으로 생성
     - LLM 실패 시 휴리스틱 기반 폴백 반환
   - 결과는 프론트엔드에서 필드별로 렌더링 가능하도록 JSON으로 제공

3. EngineFacade 조정
   - pipeline 우선 전략 유지
   - pipeline이나 provider로 얻은 papers에 대해 `summarize_clusters`와 `DeepInsightGenerator.generate_for_clusters`를 호출하여 응답에 `insights` 필드로 포함

4. 프론트엔드 정책
   - `insights.deep`를 렌더링하되 `user.plan !== 'pro'`일 경우 블러 처리 및 Pro 업그레이드 CTA 노출

## Consequences
- 장점: 저장소 추가 시 Facade 수정 불필요, Provider마다 타임아웃·레이트리밋·정규화 적용 가능, 유료용 고급 인사이트 제공으로 제품 차별화
- 단점: LLM 비용·지연(캐싱/비동기 권장), 초기 구현·테스트 비용 증가, LLM 출력 신뢰성 문제(증거·confidence 병기 필요)

## Implementation Notes
- 새 폴더/파일:
  - backend/engine/providers/
    - base.py (BaseProvider)
    - arxiv_provider.py (기존 로직 래핑)
    - __init__.py (registry / get_providers)
- 변경 파일:
  - backend/engine/facade.py (providers 사용하도록 변경)
  - backend/engine/summarizer.py (DeepInsightGenerator 추가)
  - frontend: Analyze 뷰에서 `insights.deep` 렌더링 및 Pro 락 UI 구현
- 운영 권장:
  - DeepInsight는 캐싱/비동기 처리
  - Provider별 telemetry(응답시간·실패율) 도입
  - LLM 호출 비용 모니터링 및 요금제별 제한

## Migration plan (단계 요약)
1. providers 패키지와 BaseProvider 추가, ArxivProvider 구현
2. EngineFacade에서 provider 기반 fetch 사용으로 전환 (ATLAS_PROVIDERS로 제어)
3. summarizer에 DeepInsightGenerator 추가 (먼저 heuristic 폴백 구현)
4. 프론트엔드 DeepInsight 렌더/Pro 잠금 적용
5. 테스트·모니터링·점진 롤아웃