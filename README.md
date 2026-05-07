# 개발 노트와 기술 메모

이 저장소는 개인 개발 과정에서 겪은 문제와 기술 개념을 실무 관점에서 정리해 둔 문서 저장소입니다. 특정 기술을 정답처럼 제시하기보다, 나중에 다시 읽었을 때 판단 맥락과 트레이드오프를 빠르게 복원하는 것을 목표로 합니다.

주로 다음 성격의 문서를 다룹니다.

- 개인 개발 과정에서 만난 문제와 운영 판단
- 기술 개념을 실무 적용 관점에서 다시 정리한 메모
- 이후 비슷한 상황에서 참고하기 위한 기술 블로그형 노트

## 디렉토리 구조

| 경로 | 기준 |
|---|---|
| [architecture/](./architecture/) | 레이어드, 헥사고날, 클린 아키텍처처럼 구조 설계와 의존성 경계를 다루는 문서 |
| [edd/](./edd/) | 메시지 큐, DLQ, 브로커, 이벤트 전달 의미론처럼 이벤트 기반 설계와 운영을 다루는 문서 |
| [editor/](./editor/) | Neovim 설정과 실제 편집 워크플로처럼 개발 환경 사용법을 다루는 문서 |
| [engineering/](./engineering/) | 특정 기술 범주에 묶기 어려운 개발 프로세스, 협업 방식, 리뷰 기준, 도구 선택, AI 활용 회고 문서 |
| [go/](./go/) | Go 언어 또는 Go 생태계 동작 방식에 직접 종속된 문서 |
| [infra/](./infra/) | CI/CD, Terraform, Argo CD, Vault, Runner 등 인프라 자동화와 운영 흐름을 다루는 문서 |
| [kubernetes/](./kubernetes/) | Kubernetes 환경에서의 워크로드 운영 이슈와 운영 패턴을 다루는 문서 |

## 주요 문서

- [클린 아키텍처, 헥사고날, 레이어드 아키텍처를 어떻게 구분할까](./architecture/clean-hexagonal-layered-architecture-practical-comparison.md)
- [메시지 큐 핵심 개념 정리](./edd/mq-core-concepts.md)
- [DLQ(Dead Letter Queue), 왜 필요하고 어떻게 운영해야 할까](./edd/dlq-operational-guide.md)
- [같은 이벤트를 여러 소비자가 받아야 할 때](./edd/rabbitmq-kafka-redis-streams-multi-consumer-semantics.md)
- [Kubernetes 환경에서 단일 writer처럼 동작해야 하는 메시지 소비 모듈 운영 가이드](./kubernetes/kubernetes-single-writer-consumer-operations.md)
- [Go에서 JSON marshal한 `[]byte`가 DB에 base64처럼 저장되는 이유](./go/go-json-byte-base64-in-json-column.md)
- [GitHub Actions · Runner · Docker · Registry · Argo CD · Kubernetes · Terraform · Vault를 실무 흐름으로 구분해보기](./infra/ci-cd/github-actions-runner-argocd-kubernetes-terraform-vault.md)
- [실무 리팩터링에서 AI를 깊게 썼을 때 놓치기 쉬운 것들](./engineering/ai-driven-refactoring-responsibility-checkpoints.md)

## 처음 읽는 순서

1. 저장소의 범위를 잡고 싶다면 이 README와 각 디렉토리 README를 먼저 봅니다.
2. 메시징/이벤트 기반 설계가 궁금하다면 [MQ 핵심 개념](./edd/mq-core-concepts.md) → [멀티 컨슈머 의미론](./edd/rabbitmq-kafka-redis-streams-multi-consumer-semantics.md) → [DLQ 운영](./edd/dlq-operational-guide.md) 순서로 읽습니다.
3. Kubernetes에서 소비자 운영 이슈를 다룬다면 [Kubernetes single-writer consumer 운영 가이드](./kubernetes/kubernetes-single-writer-consumer-operations.md)를 함께 봅니다.
4. 서비스 구조 설계 판단이 필요하다면 [아키텍처 비교 문서](./architecture/clean-hexagonal-layered-architecture-practical-comparison.md)를 읽고, 구현/검토 방식은 [engineering/](./engineering/) 문서로 이어서 확인합니다.
5. Go 관련 문제를 찾는다면 [go/](./go/)에서 언어 동작이나 생태계에 직접 연결된 문서를 확인합니다.

## 문서 작성/검토 원칙

- 관찰한 사례를 보편적 사실처럼 단정하지 않습니다.
- 특정 사례를 일반론으로 과하게 확장하지 않습니다.
- 실무 판단 기준과 트레이드오프를 함께 드러냅니다.
- 특정 기술 선택지를 정답처럼 말하기보다 조건과 제약을 함께 적습니다.
- 과장된 표현을 피하고 담백한 기술 메모 톤을 유지합니다.
