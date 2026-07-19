# ADR-0010: torch 버전 업그레이드 및 빌드 실패 해결

## Status
Accepted

## Context
Docker 빌드 과정에서 `torch==1.11.0` 버전을 찾을 수 없어 설치가 실패했습니다. 이는 현재 Python 3.11 환경에서 `torch==1.11.0`이 지원되지 않거나, 더 이상 유지관리되지 않기 때문입니다.

## Decision
`torch` 버전을 `1.13.1+cpu`로 업그레이드하여, 최신 환경 및 의존성에 맞게 빌드 구성 파일을 수정합니다.

- **requirements.txt**:
  - `torch==1.11.0` -> `torch==1.13.1+cpu` 로 수정.
  - CPU 전용 빌드를 명시적으로 사용.

- **Dockerfile**:
  - Python Slim 이미지를 유지하면서, 의존성 및 설치 환경을 최적화.

## Consequences
- Docker 빌드가 성공적으로 완료될 가능성이 높아짐.
- 더 최신 패키지로 전환하여 보안성과 성능 개선.