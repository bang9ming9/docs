# Engineering

특정 언어, 인프라, Kubernetes, 메시징, 아키텍처 범주에 직접 들어가지 않는 실무 개발 메모를 둡니다. 이 디렉토리는 기타 문서 모음이 아니라, 개발 프로세스와 협업 방식, 리뷰 기준, 도구 선택, AI 활용 회고처럼 여러 기술 영역을 가로지르는 판단 기준을 관리하는 공간입니다.

## 분류 기준

- 개발 프로세스, 리뷰 기준, 작업 계획 검토처럼 팀 작업 방식에 가까운 문서
- 특정 제품 사용법보다 실무 판단 기준과 트레이드오프를 정리한 문서
- AI 도구 활용, 모델 선택, 리팩터링 책임 경계처럼 개발 워크플로 전반에 걸친 문서
- Go, Kubernetes, 메시징, 인프라 자동화처럼 더 명확한 상위 범주가 있으면 해당 디렉토리를 우선합니다.

## 문서 목록

- [실무에서 다시 보는 TDD 원칙](./tdd-red-green-refactor-principles.md)
- [코드에는 있지만 아직 켜지지 않은 기능, dormant feature 이해하기](./dormant-feature-understanding-guide.md)
- [실무 리팩터링에서 AI를 깊게 썼을 때 놓치기 쉬운 것들](./ai-driven-refactoring-responsibility-checkpoints.md)
- [작업 계획서를 더 잘 검토하기 위한 type 기반 체크리스트](./type-based-plan-checklist-guide.md)
- [모노레포 vs 일반 단일 프로젝트](./monorepo-vs-single-project-repo-structure-guide.md)
- [Opus와 Sonnet, 실무에서 어떻게 나눠 이해하고 쓸 것인가](./opus-vs-sonnet-practical-usage-guide.md)
- [MongoDB vs MySQL 조회 성능](./mongodb-vs-mysql-read-performance-practical.md)
- [Buf 기반 Protobuf 운영](./buf-config-and-workflow.md)

## 경계 사례

- [MongoDB vs MySQL 조회 성능](./mongodb-vs-mysql-read-performance-practical.md)은 별도 `database/` 디렉토리를 만들기에는 현재 문서 수가 적어, 저장소 선택 기준과 읽기 모델 설계 메모로 이 디렉토리에 둡니다.
- [Buf 기반 Protobuf 운영](./buf-config-and-workflow.md)은 CI/CD 자동화 자체보다 스키마 운영과 개발 워크플로 판단 기준을 다루므로 이 디렉토리에 둡니다.

## 함께 보면 좋은 문서

- [아키텍처 비교 문서](../architecture/clean-hexagonal-layered-architecture-practical-comparison.md): 구조 설계와 개발 프로세스 판단을 함께 볼 때 참고합니다.
- [인프라 자동화 흐름 정리](../infra/ci-cd/github-actions-runner-argocd-kubernetes-terraform-vault.md): 저장소 구조와 배포 자동화 책임을 함께 검토할 때 참고합니다.
