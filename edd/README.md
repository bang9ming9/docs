# Event-Driven Design

메시지 큐, 브로커, DLQ, 재시도, 소비자 독립성처럼 이벤트 기반 설계와 운영에서 반복해서 등장하는 개념을 정리합니다. 특정 제품 사용법보다 전달 의미론, 실패 처리, 관찰 지표를 구분하는 데 초점을 둡니다.

## 문서 목록

- [메시지 큐 핵심 개념 정리](./mq-core-concepts.md): producer, consumer, broker, ack/nack, retry, DLQ 같은 기본 용어를 정리합니다.
- [DLQ 운영 가이드](./dlq-operational-guide.md): 실패 메시지를 격리하고 재처리/폐기 판단으로 연결하는 운영 흐름을 다룹니다.
- [RabbitMQ, Kafka, Redis Streams 멀티 컨슈머 의미론 비교](./rabbitmq-kafka-redis-streams-multi-consumer-semantics.md): 같은 이벤트를 여러 소비자가 받아야 할 때 시스템별 독립성 단위를 비교합니다.

## 추천 흐름

1. 먼저 [메시지 큐 핵심 개념 정리](./mq-core-concepts.md)로 공통 용어를 맞춥니다.
2. 여러 소비자 또는 브로커별 전달 의미가 헷갈리면 [멀티 컨슈머 의미론 비교](./rabbitmq-kafka-redis-streams-multi-consumer-semantics.md)를 봅니다.
3. 실패 메시지 운영 기준이 필요하면 [DLQ 운영 가이드](./dlq-operational-guide.md)로 이어서 봅니다.

## 연결 문서

- [Kubernetes 단일 writer 소비자 운영 가이드](../kubernetes/kubernetes-single-writer-consumer-operations.md): 메시지 소비자가 Kubernetes에서 공백/중복/side effect 문제를 만날 때 함께 참고합니다.
