# ADR-0002: 배포 환경 ImportError / NameError 수정 (패키지화 및 typing 명시)

- Date: 2025-12-31  
- Status: Accepted  
- Authors: engineer0427

## 결정 배경 (Context)
Koyeb 등 클라우드 배포 환경에서 Python 패키지/모듈 임포트 과정과 타입 힌트(typing) 관련으로 ImportError 또는 NameError가 발생하는 문제가 관찰되었습니다. 원인으로는:
- providers 디렉터리의 모듈/패키지화(__init__.py 부재 또는 불완전)로 인한 상대 import 실패
- BaseProvider 인터페이스(또는 typing이 명확히 선언되지 않아) 이름 인식 문제

이 문제는 로컬 개발 환경에서는 드러나지 않지만, 배포(가상환경·패키지 경로)에서는 import 룰이 엄격하게 적용되어 런타임 에러로 이어집니다.

## 조치 내용 (Decision / Actions)
1. typing 명시
   - backend/engine/providers/base.py 파일 상단에 `from typing import Optional, List, Any` 를 명시적으로 추가하여 타입 참조 누락으로 인한 이름 인식 문제를 방지합니다.
2. BaseProvider 추상화 구현
   - BaseProvider를 추상 클래스(ABC)로 구현하여 모든 Provider가 일관된 인터페이스를 사용하도록 강제합니다.
3. 패키지화 및 registry 추가
   - backend/engine/providers/__init__.py에 provider registry(get_providers)를 구현하여 Facade에서 dynamic하게 provider를 로드할 수 있게 합니다. 이를 통해 facade.py 수정 없이 설정(ATLAS_PROVIDERS)만으로 provider 확장이 가능합니다.
4. arXiv Provider 정합성 확보
   - backend/engine/providers/arxiv_provider.py에서 `.base`로부터 BaseProvider 를 가져와 상속 구조를 보장하도록 수정했습니다.

## 결과 및 기대효과 (Consequences)
- 배포 환경에서의 ImportError / NameError가 해소됩니다.
- Provider 추가/관리(예: bioRxiv, SSRN) 시 패키지 등록만으로 확장 가능해 유지보수성이 향상됩니다.
- 타입 명시 및 추상화로 코드의 명확성과 안정성이 증가합니다.

## 상태
- 승인됨 (Accepted)

## 참고 사항 / 추가 권장 작업
- providers 폴더에 새 provider를 추가할 때는 반드시 `providers/__init__.py`의 `_PROVIDER_MAP`에 등록하세요.
- Provider 구현 시 반환 타입을 프로젝트 공통 PaperMetadata로 정규화하는 로직을 포함하면 downstream 처리(embedding, graph build 등)에서 호환성 문제가 줄어듭니다.
- CI 단계에 `python -m pip install -e .` 및 `python -c "import backend.engine.providers"` 형식의 간단한 import smoke-test를 추가하여 배포 전 검증을 권장합니다.