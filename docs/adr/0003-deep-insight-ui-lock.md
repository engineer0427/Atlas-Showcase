# ADR-0003: Deep Insight UI 잠금(Lock) 정책 및 구현

- Date: 2026-01-03  
- Status: Accepted  
- Authors: engineer0427

## Context
백엔드에서 군집별로 생성되는 구조화된 'Deep Insight' 데이터(technical_gap, critical_limitations, practical_application 등)를 제품의 차별화된 유료 기능으로 제공하기로 했습니다. UI에서 해당 인사이트를 바로 보여줄 수 있으나, 무료 사용자에게는 일부(또는 전체)를 제한하여 Pro 업그레이드를 유도해야 합니다. 또한 LLM 기반 인사이트는 비용과 지연이 발생하므로 관련 UX·캐싱 정책도 함께 고려해야 합니다.

## Decision
1. Deep Insight는 제품의 Pro 전용 기능으로 간주한다.
2. 프론트엔드에서는 각 Deep Insight 항목을 카드 형태로 렌더링하고, `user.plan !== 'pro'` 일 경우 내용은 잠금(Lock) 상태로 표시한다(제목은 노출).
   - 잠금 UI는 `InsightCard` 컴포넌트로 모듈화(components/InsightCard.tsx).
   - 잠긴 카드에는 `Lock` 아이콘과 "Pro 멤버십 전용 분석입니다" 문구, 섹션 단위의 "Upgrade to Pro" CTA(Indigo-600) 노출.
   - 잠금 시 카드 내용은 블러 또는 불투명 처리(opacity)로 가시성 낮춤.
3. 백엔드는 `insights` 필드를 응답에 포함하도록 보장한다(형식: JSON, cluster_id 기준).
4. Deep Insight 생성(LLM)은 비용·지연·신뢰성 이슈가 있으므로:
   - 생성 결과는 캐시/DB에 저장해 재사용한다.
   - LLM 호출 실패 시 휴리스틱 폴백을 제공한다.
   - Pro 사용자 요청에 우선권을 부여하거나 요금제별 세분화 가능.
5. 문서화: 이 결정을 ADR로 기록하여 향후 정책(유료화·캐싱·모니터링) 근거로 사용한다.

## Consequences
- 장점
  - 제품의 명확한 유료 기능(모네타이제이션) 확보.
  - UX상 명확한 업그레이드 유도(CTA).
  - LLM 비용 통제(캐싱) 및 실패 대비.
- 단점
  - 무료 사용자는 기능 체험 제한 → 전환율 설계 필요.
  - LLM·캐시·권한 로직으로 인한 추가 개발·운영 비용.

## Implementation Notes
- 프론트엔드
  - 파일: frontend/app/analyze/AnalyzeClient.tsx (insights 수신 및 상태 저장)
  - 파일: frontend/app/analyze/actions.ts (insights 포함 API 호출)
  - 컴포넌트: frontend/app/analyze/components/InsightCard.tsx (잠금 UI)
- 백엔드
  - EngineFacade/summarizer에서 `insights` 반환 보장(형식: { cluster_summaries: {...}, deep: { cluster_id: { technical_gap, critical_limitations, practical_application, ... } } })
  - Deep Insight 생성 결과 캐싱(예: Redis 또는 DB)
- 운영
  - LLM 사용량/응답시간/비용 모니터링 및 알림
  - ADR 및 관련 문서에 캐싱 TTL, 요금제별 정책(무료/프로) 명시

## Migration / Rollout
1. 백엔드에서 `insights` 필드 포함(스테이징 환경에서 검증).
2. 프론트엔드 `InsightCard` 컴포넌트 추가 및 AnalyzeClient 연동.
3. 캐시·모니터링 구성(우선은 short TTL + 로그 수집).
4. 내부 베타 사용자에게 기능 활성화 후 지표 관찰(전환율/쿼리 패턴 등).