# ADR-0005: Docker 이미지 최적화 및 배포 실패 해결

## Status
Accepted

## Context
현재 Koyeb 배포 로그에서 도커 이미지의 압축 크기가 Koyeb의 허용 크기(2000 MiB)를 초과하여 배포가 실패했습니다. 이는 불필요한 파일과 데이터가 Docker 이미지에 포함되어 있기 때문입니다.

## Decision
Dockerfile 및 `.dockerignore` 파일을 수정하여 이미지 크기를 2000 MiB 이내로 최적화합니다.

- **Dockerfile**: Slim base 이미지를 사용하여 크기 감소.
- **.dockerignore**: 불필요한 캐시, 로그 파일, 테스트 데이터를 빌드 제외.
- 추가적으로, pip 캐시를 제거하여 이미지 크기를 더 축소.

## Consequences
- 성공적으로 배포될 가능성이 높아짐.
- 비용 없는 최적화를 통해 Koyeb 인스턴스 업그레이드 대신 해결.