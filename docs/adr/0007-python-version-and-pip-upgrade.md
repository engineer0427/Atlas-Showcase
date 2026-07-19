# ADR-0007: Python 버전 및 pip 업데이트를 통한 빌드 문제 해결

## Status
Accepted

## Context
배포 실패가 발생한 이유는 요구되는 Python 버전과 패키지 호환성 문제 때문입니다.
- `torch==2.1.0`은 Python 3.11 이상을 요구합니다.
- 오래된 pip 버전(23.0.1)이 사용되고 있어 패키지 설치 중 문제가 발생합니다.

## Decision
- Python 이미지를 `python:3.11-alpine`으로 변경합니다.
- `pip install --upgrade pip`로 최신 pip 버전을 사용합니다.
- 호환되지 않는 패키지 버전을 수정하거나 필요하지 않은 경우 제거합니다.

## Consequences
- 빌드 및 배포가 성공 가능성이 높아집니다.
- 최신 Python과 pip로 관리되는 환경에서 호환성 문제가 최소화됩니다.