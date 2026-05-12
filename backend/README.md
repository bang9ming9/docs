# Backend

외부 API 호출, 긴 작업 처리, 비동기 Job, 상태 관리, 운영 관찰처럼 백엔드 서버를 구현하면서 반복해서 마주치는 설계 판단을 정리합니다.

이 디렉토리는 특정 프레임워크 사용법보다, 서비스 요청을 안정적으로 처리하기 위해 서버가 어떤 책임을 가져야 하는지에 초점을 둡니다.

## 문서 목록

- [외부 AI 모델 호출을 백엔드에서 다룰 때의 기본 책임](./external-ai-model-call-backend.md)
- [AI 모델 호출을 동기 API에서 비동기 Job으로 분리하는 기준](./sync-to-async-ai-job-processing.md)
- [AI Job 상태, 결과 저장, Retry를 함께 설계해야 하는 이유](./ai-job-state-result-retry.md)
- [AI Backend 운영 관찰: Metrics, Readiness, Canary](./ai-backend-observability-canary.md)
