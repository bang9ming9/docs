# 같은 이벤트를 여러 소비자가 받아야 할 때: RabbitMQ fanout, Kafka consumer group, Redis Streams consumer group 비교

## 1) 왜 이 주제가 헷갈렸는가
실무에서 가장 자주 헷갈린 지점은 “브로커를 쓰고 있다”는 공통점 때문에, 서로 다른 시스템의 **소비 독립성 단위**를 같은 것으로 착각하기 쉽다는 점이었다.

- RabbitMQ에서도 여러 소비자가 붙을 수 있고,
- Kafka에서도 여러 소비자가 같은 토픽을 읽을 수 있고,
- Redis Streams도 consumer group을 제공한다.

하지만 “같은 메시지를 여러 주체가 **각각** 받아야 한다”는 요구를 구현할 때, 실제로 독립성을 만드는 단위는 시스템마다 다르다. 이 차이를 놓치면, 새 소비자를 붙였을 때 기존 처리 흐름에 영향을 주거나, 장애 시 어디에 적체가 생기는지 잘못 해석하게 된다.

---

## 2) RabbitMQ에서 exchange와 queue의 역할을 분리해서 보기
RabbitMQ에서 먼저 분리해서 봐야 할 것은 다음 두 가지다.

- **Exchange**: 들어온 메시지를 어떤 큐로 보낼지 결정하는 라우팅 계층
- **Queue**: 실제 메시지가 대기/적체되는 저장 지점

즉, “메시지가 쌓이는 곳”은 exchange가 아니라 queue다. exchange는 라우팅 규칙(fanout, direct, topic 등)에 따라 메시지를 하나 이상의 queue로 전달한다.

이 구분이 중요한 이유는 운영 관찰 지표도 queue 기준으로 해석해야 하기 때문이다. 소비자 지연, 적체, 재처리 이슈는 대체로 queue 레벨에서 드러난다.

---

## 3) RabbitMQ에서 같은 메시지를 여러 소비자가 각각 받아야 할 때
핵심은 **소비자 수를 늘리는 것**과 **소비 단위를 분리하는 것**이 다르다는 점이다.

### 같은 queue에 여러 consumer를 붙인 경우
- 메시지는 보통 경쟁 소비(competitive consumption)된다.
- 한 메시지를 여러 consumer가 각각 받는 것이 아니라, 여러 consumer 중 하나가 가져가 처리한다.
- 처리량 확장(워커 수평 확장)에는 유용하지만, “모든 소비자가 동일 이벤트를 각자 처리” 요구에는 맞지 않는다.

### 여러 queue로 분리한 경우 (fanout/binding 설계)
- fanout exchange(또는 목적에 맞는 binding 설계)로, 동일 메시지를 여러 queue에 전달한다.
- 각 queue는 독립된 소비 파이프라인이 된다.
- 한 queue의 consumer가 멈추면 적체는 해당 queue에 국소화되고, 다른 queue의 소비 진행은 별도로 관찰된다.

실무적으로는 “새 기능을 기존 흐름에 영향 없이 붙이기”가 자주 필요하다. RabbitMQ에서는 보통 **새 queue를 추가하고 exchange에 바인딩**하는 방식이 이에 해당한다.

---

## 4) Kafka consumer group에서 같은 요구를 푸는 방식
Kafka에서는 같은 topic을 여러 consumer group이 독립적으로 읽을 수 있다. RabbitMQ의 “queue 분리”와 대응되는 감각은 Kafka에서 “**group 분리**”에 가깝다.

- 같은 group 내부 소비자는 파티션을 나눠 읽는 협력 관계다.
- 서로 다른 group은 같은 topic 로그를 각자 offset으로 추적하며 독립 소비한다.
- 따라서 새 consumer group을 추가하면, 기존 group의 **offset 진행과는 독립적으로** 같은 메시지를 별도로 읽을 수 있다.

다만 운영 해석은 queue 모델과 다르게 해야 한다.

- 어떤 group이 멈추면 그 group의 offset 기준으로 **lag**가 증가한다.
- 이는 “해당 group이 아직 읽지 못한 기록”이 늘어나는 의미이지, RabbitMQ의 특정 queue 적체와 완전히 같은 개념은 아니다.
- Kafka에서는 topic retention 정책에 따라 로그 보관 기간/용량이 관리되며, group이 늦게 읽으면 retention 경계와 충돌할 수 있다.

요약하면 Kafka는 “로그 + group별 offset” 관점으로 봐야 하며, queue 길이와 1:1로 대응해 이해하면 오해가 생긴다.

---

## 5) Redis Streams consumer group은 어디에 위치하는가 (비교 관점)
Redis Streams도 consumer group을 통해 여러 소비자 협력/복구를 지원하지만, 실무 감각은 RabbitMQ queue 모델이나 Kafka log 모델과 완전히 동일하지 않다.

비교에 필요한 최소 포인트만 정리하면:

- 데이터는 stream에 append되고,
- group은 전달 진행 기준과
- 소비자별 pending entries(미확인 처리 목록)를 관리한다.

특히 pending 상태는 장애 복구나 재할당(claim) 시 운영적으로 중요하다.

즉 “어디에 상태가 남는가”를 볼 때,
- RabbitMQ는 queue 적체,
- Kafka는 topic log + group offset/lag,
- Redis Streams는 stream 항목 + group/consumer의 pending 상태
를 구분해서 보는 것이 안전하다.

---

## 6) 소비자가 자주 멈추거나 붙었다 떨어질 때 운영상 주의점
이 상황에서는 “시스템별 독립 단위”를 기준으로 경계 조건을 점검해야 한다.

### RabbitMQ
- queue 단위로 적체를 모니터링한다.
- 특정 소비자 장애가 특정 queue backlog로 국소화되는지 확인한다.
- 재시도/재큐잉 정책을 과도하게 두면 queue 체류 시간이 급격히 증가할 수 있다.
- 여러 기능이 하나의 queue를 공유하면 적체 원인과 장애 영향 범위를 분리해서 보기 어려워질 수 있다.

### Kafka
- group별 lag 추이를 본다.
- 재시작이 잦은 소비자는 rebalance 비용과 처리 지연을 유발할 수 있다.
- “나중에 읽으면 되지”라고 단순화하지 말고 retention과 최대 복구 시간 목표를 함께 맞춘다.

### Redis Streams
- pending entries 누적 및 idle 시간(오랫동안 ack되지 않은 항목)을 점검한다.
- 장애 복구 시 pending 재할당/claim 전략을 운영 절차로 명확히 둔다.

---

## 7) 설계 판단을 위한 실무 기준
“전역적으로 발생하는 모든 이벤트를 기존 소비자에 영향 없이 받고 싶다”는 요구는, 시스템이 달라도 본질적으로 **독립된 소비 단위를 추가**하는 문제다.

- RabbitMQ: queue를 분리하고 exchange 바인딩으로 동일 이벤트를 복제 전달
- Kafka: 새 consumer group을 추가해 같은 topic을 독립 offset으로 소비
- Redis Streams: 별도 group/소비 전략으로 독립 처리 경로를 구성

실무에서는 독립 소비 단위만 맞추는 것으로 끝나지 않는다. 장애 시 어디에 상태가 쌓이고, 복구 시 어떤 관찰 포인트를 봐야 하는지까지 함께 설계해야 운영 문제가 줄어든다.

설계 시에는 아래를 함께 본다.

1. **독립성 단위**: queue / group / stream-group 중 무엇을 분리할 것인가
2. **장애 시 적체 위치**: queue backlog / group lag / pending entries
3. **신규 소비자 추가 비용**: 라우팅 변경, group 생성, 운영 모니터링 포인트 증가
4. **복구 한계**: 보관(retention)과 재처리 정책, 소비 지연 허용 범위

같은 요구라도 브로커마다 관찰 포인트와 복구 방식이 다르기 때문에, 신규 소비자 추가 설계와 운영 기준을 한 세트로 정해두는 편이 좋다.

결론적으로, “같은 메시지를 여러 소비자가 받아야 한다”는 문장을 만나면 먼저 시스템 이름보다 **독립 소비 단위를 어디에 둘지**부터 결정하는 편이 안전하다. 이 판단은 아키텍처 유연성과 장애 격리 수준에 큰 영향을 준다.
