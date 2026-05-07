# Kubernetes

Kubernetes 환경에서 워크로드 운영 중 드러나는 판단 기준을 다룹니다. 현재는 메시지 소비 모듈 운영 문서의 짧은 인덱스 역할만 합니다.

## 문서 목록

- [Kubernetes 단일 writer 소비자 운영 가이드](./kubernetes-single-writer-consumer-operations.md)

## 관련 문서

- [메시지 큐 핵심 개념 정리](../edd/mq-core-concepts.md): consumer, ack/nack, retry, idempotency 같은 기본 용어를 함께 확인할 때 참고합니다.
- [RabbitMQ, Kafka, Redis Streams 멀티 컨슈머 의미론 비교](../edd/rabbitmq-kafka-redis-streams-multi-consumer-semantics.md): Kubernetes 위에서 소비자 수평 확장과 브로커별 독립성 단위를 함께 볼 때 참고합니다.
- [DLQ 운영 가이드](../edd/dlq-operational-guide.md): 소비 실패가 누적될 때 격리와 재처리 판단 기준을 확인할 때 참고합니다.
