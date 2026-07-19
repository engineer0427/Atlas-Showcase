# ADR-0004: 3D Graph Client-Side Rendering Strategy

## Status
Accepted

## Context
Next.js 15 환경의 `Workbench.tsx` 페이지 진입 시, `react-force-graph-3d` 라이브러리 내부에서 사용하는 `AFRAME`이 서버 사이드 렌더링(SSR) 과정에서 `window` 객체에 접근하려다 `TypeError: AFRAME.registerComponent is not a function` 오류를 발생시키며 애플리케이션이 비정상 종료되는 현상이 확인됨.

## Decision
우리는 3D 그래프 렌더링 로직을 다음과 같이 재구성하기로 결정함:

1.  **컴포넌트 분리**: `react-force-graph-3d`를 사용하는 로직을 `ForceGraphWrapper.tsx`라는 별도의 클라이언트 컴포넌트로 완전히 격리한다.
2.  **SSR 비활성화**: 상위 컴포넌트(`Workbench.tsx`)에서 `next/dynamic`의 `{ ssr: false }` 옵션을 사용하여 해당 래퍼 컴포넌트를 동적으로 로드한다. 이는 3D 엔진이 브라우저 환경(Client-Side)에서만 초기화되도록 보장한다.
3.  **물리 기반 UX**: 노드 간의 링크 길이를 `similarity_score` 역수에 비례하여 설정하고, 초기 로딩 완료 후 `zoomToFit`을 호출하여 사용자 경험을 개선한다.

## Consequences
- **Positive**: 초기 페이지 진입 시 발생하던 500 에러 및 Application Crash가 완전히 해결됨.
- **Positive**: 3D 라이브러리의 무거운 로딩 과정을 스켈레톤 UI로 대체하여 체감 성능이 향상됨.
- **Negative**: 해당 그래프 영역은 서버 사이드에서 렌더링되지 않으므로, 초기 HTML에 포함되지 않아 SEO에는 불리할 수 있음 (단, 분석 툴 특성상 큰 문제는 아님).