# Kubernetes

Kubernetes 환경에서 애플리케이션 워크로드를 운영할 때 생기는 문제를 다룹니다. 리소스 정의 자체보다 Pod 생명주기, failover, 중복 처리, graceful shutdown, 운영 보장 수준처럼 런타임에서 드러나는 판단 기준에 초점을 둡니다.

## 문서 목록

- [Kubernetes 환경에서 단일 writer처럼 동작해야 하는 메시지 소비 모듈 운영 가이드](./kubernetes-single-writer-consumer-operations.md)

## 함께 보면 좋은 문서

- [메시지 큐 핵심 개념 정리](../edd/mq-core-concepts.md): consumer, ack/nack, retry, idempotency 같은 기본 용어를 함께 확인할 때 참고합니다.
- [RabbitMQ, Kafka, Redis Streams 멀티 컨슈머 의미론 비교](../edd/rabbitmq-kafka-redis-streams-multi-consumer-semantics.md): Kubernetes 위에서 소비자 수평 확장과 브로커별 독립성 단위를 함께 볼 때 참고합니다.
- [DLQ 운영 가이드](../edd/dlq-operational-guide.md): 소비 실패가 누적될 때 격리와 재처리 판단 기준을 확인할 때 참고합니다.
