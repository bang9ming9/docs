# Infra

빌드, 배포, 운영 자동화 도구의 책임 경계를 정리하는 문서를 둡니다. Kubernetes 자체의 워크로드 운영 이슈는 [kubernetes/](../kubernetes/)를 우선합니다.

## 하위 구조

- [ci-cd/](./ci-cd/): GitHub Actions, Runner, Registry, Argo CD, Terraform, Vault처럼 배포 자동화 흐름 안에서 함께 쓰이는 도구의 책임 경계를 정리합니다.

## 문서 목록

- [CI/CD 도구 책임 경계 정리](./ci-cd/github-actions-runner-argocd-kubernetes-terraform-vault.md)

## 관련 문서

- [Kubernetes 단일 writer 소비자 운영 가이드](../kubernetes/kubernetes-single-writer-consumer-operations.md): 배포 플랫폼 위에서 실제 워크로드 운영 의미를 검토할 때 참고합니다.
- [모노레포 vs 일반 단일 프로젝트](../engineering/monorepo-vs-single-project-repo-structure-guide.md): 저장소 구조와 자동화 파이프라인 책임을 함께 판단할 때 참고합니다.
