# ADR-0008: Torch 호환성과 Python 버전 문제 해결

## Status
Accepted

## Context
배포 실패가 발생한 주요 원인은 다음과 같습니다:
1. `sentence-transformers`의 종속성인 `torch>=1.11.0` 패키지를 설치할 수 없는 문제.
2. 일부 패키지가 Python 버전에 맞지 않아 설치 충돌 발생.

## Decision
- `sentence-transformers` 종속성을 충족하도록 필요 시 Python 버전을 조정하거나 종속성 설치 단계를 세분화합니다.
- `torch`의 호환 가능한 버전을 명시적으로 설정하여 충돌을 최소화합니다.
- `requirements.txt`에서 호환되지 않는 버전을 제거하거나 다른 패키지로 대체합니다.

## Consequences
- Python 및 종속성 충돌 문제 해결로 빌드와 배포가 성공 가능성을 높임.
- 패키지 호환성을 명확히 관리하여 추후 배포 안정성을 강화함.