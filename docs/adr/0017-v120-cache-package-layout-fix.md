# 0017: Cache package layout fix for InsightCache imports

## Status
Proposed

## Context
v1.2.0에서 InsightCache가 `src.atlas.cache.insight_cache` 경로로 import 되도록 설계되었다.
하지만 기존 코드베이스에는 `src/atlas/cache.py` 단일 모듈이 존재하여,
Python import 시스템이 `src.atlas.cache`를 "package"가 아닌 "module"로 해석한다.
이로 인해 `src.atlas.cache.insight_cache`가 import 불가능해져 프로덕션 부팅 단계에서 즉시 크래시한다.

## Decision
- `src/atlas/cache.py`를 패키지로 승격한다.
  - 기존 cache 유틸(get/set/delete/clear)은 `src/atlas/cache/__init__.py`로 이동하여 호환을 유지한다.
  - v1.2.0 `InsightCache`는 `src/atlas/cache/insight_cache.py`에 둔다.

## Consequences
- `src.atlas.cache.*` 하위 모듈 확장이 가능해진다.
- 기존 `from src.atlas import cache` 또는 `from src.atlas.cache import get` 사용도 유지된다.