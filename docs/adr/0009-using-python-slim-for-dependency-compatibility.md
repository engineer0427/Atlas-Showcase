# ADR-0009: Python Slim 이미지로 전환 및 종속성 문제 해결

## Status
Accepted

## Context
Alpine Linux 이미지를 사용하��� 도커 환경에서 `torch` 패키지 (버전 1.11.0)가 시스템 라이브러리와의 호환성 문제로 설치에 실패했습니다.  
Slim 이미지를 사용하면 더 많은 시스템 의존성을 기본적으로 포함하고 있어, 빌드 과정에서 필수적인 라이브러리를 쉽게 처리할 수 있습니다.

## Decision
Python Slim (`python:3.11-slim`) 이미지를 기반으로 환경을 전환합니다.  
이전 Alpine 이미지 기반의 Dockerfile을 Slim 이미지로 수정하여 빌드 및 종속성 문제를 해결합니다.

## Consequences
- `torch`와 같은 라이브러리 빌드가 성공적으로 진행됩니다.
- 이미지 크기가 약간 증가하지만, 설정의 안정성이 크게 향상됩니다.
- 추가적으로, 최적화된 이미지 제작을 위해 불필요한 패키지 제거 (`--no-install-recommends`) 및 캐시 클리어를 수행했습니다.