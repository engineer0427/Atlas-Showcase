# 0011: CORS 및 인증 처리 방식 수정

## 상태
채택됨

## 맥락 (Context)
- 클라이언트(`https://atlaslab.app`)와 서버(`https://atlas-backend.koyeb.app`) 간 CORS 이슈가 발생.
- 기존 FastAPI 엔드포인트에서 `/me` URL이 404 오류를 반환.
- 프론트엔드의 Axios 요청에 대해 401 인증 실패 및 네트워크 에러 현상 지속.

## 결단 (Decision)
1. **CORS 허용**:
   - 환경 변수(`CORS_ORIGINS`) 및 백엔드 설정을 검토.
   - `https://atlaslab.app`와 다른 Origin들을 안전하게 허용하기 위해 `MERGED_CORS_ORIGINS` 값을 확인 및 보완.
   - 필요 시 Koyeb 배포 환경의 절대경로를 명확히 저장.

2. **엔드포인트 보완 작업**:
   - `/me`와 `/auth/login` 엔드포인트에서 잘못된 종속성을 확인하고 수정.
   - 현재 앱에 `get_current_user` 로직 업그레이드 및 기본 데이터 예제 배치.
   - 401 에러와 잘못된 요청을 명확히 로그로 남기기.

3. **프론트엔드 요청 검증**:
   - `frontend/utils/api.js`의 기본 API Base URL 확인(이 문제 중 환경 설정과의 일치도 고려 필요). 기본값: `https://atlas-backend.koyeb.app`.
   - 각 프론트엔드 Axios 함수에서 전달하는 헤더(예: Content-Type, Authorization)가 올바르게 적용되었는지 검토.

## 결과 (Consequences)
- 프론트엔드와 백엔드 간 요청 실패(CORS 및 네트워크) 문제 해결.
- 클라이언트에서 기존 Axios 요청 실패 건수가 대폭 감소.
- 잘못된 환경 변수로 인한 배포 문제의 예방 가능성 확보.