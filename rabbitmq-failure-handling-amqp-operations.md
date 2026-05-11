# AMQP 기반 시스템에서 RabbitMQ 장애를 다루는 방법

## 왜 이 주제를 다시 정리하게 되었는가

RabbitMQ를 운영하다 보면 장애 원인을 “MQ가 잠깐 불안정했다” 정도로 단순하게 이해하기 쉽다. 하지만 실제 서비스 장애에서는 RabbitMQ 자체의 일시적인 불안정성보다, 그 상황을 애플리케이션이 어떻게 받아들이고 복구하느냐가 더 큰 차이를 만든다.

예를 들어 RabbitMQ 노드가 재시작되거나 네트워크 연결이 끊겼을 때, 브로커는 다시 살아날 수 있다. 하지만 애플리케이션이 하나의 endpoint만 바라보고 있거나, connection만 다시 열고 channel과 consumer를 복구하지 못하거나, 모든 Pod이 동시에 재시도하면 장애는 쉽게 증폭된다. 메시지를 발행했다고 생각했지만 실제로는 라우팅되지 않았거나, consumer가 처리에 성공했지만 ack 직전에 연결이 끊겨 같은 메시지를 다시 받을 수도 있다.

이 글은 AMQP 문법이나 RabbitMQ 기능 목록을 나열하려는 목적이 아니다. 개발자가 운영 중 겪을 수 있는 RabbitMQ 장애를 이해하기 위해, 어떤 개념을 나눠서 봐야 하는지 정리하는 데 초점을 둔다.

핵심은 다음 네 가지다.

* Queue HA: 큐가 어떤 노드와 복제 구조 위에서 동작하는가
* 재연결: connection, channel, consumer, publisher 상태를 어떻게 복구하는가
* Backoff & Jitter: 장애 중 재시도가 복구를 방해하지 않게 제어하는가
* 메시지 신뢰성: publish, consume, retry, outbox, idempotency를 어떻게 함께 설계하는가

## Queue HA: 클러스터라고 해서 모든 큐가 자동으로 안전한 것은 아니다

RabbitMQ를 클러스터로 운영한다고 해서 모든 큐가 자동으로 장애에 안전해지는 것은 아니다. 메시지는 RabbitMQ 클러스터 안의 추상적인 공간에 균등하게 떠 있는 것이 아니라, 큐 타입과 배치 정책에 따라 특정 노드와 특정 복제 구조 위에서 관리된다.

Classic Queue는 RabbitMQ에서 오래 사용된 기본 큐 타입이다. 단순하고 익숙하지만, 큐의 leader 또는 master 역할을 하는 노드 상태에 영향을 받을 수 있다. 클러스터 노드가 여러 개 있어도 특정 queue가 위치한 노드가 재시작되거나 일시적으로 사용할 수 없게 되면, 해당 queue를 사용하는 producer와 consumer는 영향을 받을 수 있다.

과거에는 Mirrored Queue를 통해 classic queue를 여러 노드에 복제하는 방식도 사용됐다. 하지만 현재 기준에서는 legacy 방식으로 보는 것이 안전하다. 새로운 설계에서는 Mirrored Queue를 적극적인 선택지로 두기보다, 과거에 사용되던 HA 방식으로만 이해하는 편이 좋다.

현재 RabbitMQ에서 HA와 데이터 안전성을 고려할 때 자주 검토되는 큐 타입은 Quorum Queue다. Quorum Queue는 Raft 기반으로 동작하며, leader와 follower replica를 가진다. leader 노드가 내려가면 quorum을 만족하는 조건에서 새 leader를 선출할 수 있다.

다만 Quorum Queue도 모든 상황의 정답은 아니다. 복제를 위해 디스크 I/O와 네트워크 비용이 더 들 수 있고, classic queue와 성능 특성이나 지원 기능이 다를 수 있다. quorum을 만족하지 못하는 상황에서는 큐를 사용할 수 없다는 점도 고려해야 한다.

따라서 Queue HA를 볼 때는 “RabbitMQ 클러스터를 쓰고 있다”에서 멈추면 안 된다. 다음 질문까지 이어져야 한다.

* 이 큐는 어떤 타입인가?
* 이 큐의 leader가 내려가면 어떤 서비스가 영향을 받는가?
* 이 메시지는 잠깐 지연되어도 되는가, 유실되면 안 되는가?
* 처리량이 중요한가, 데이터 안전성이 더 중요한가?
* quorum을 유지할 수 있는 노드 수와 배치 구조가 있는가?

Queue HA는 단순한 인프라 설정이 아니라, 메시지 중요도와 서비스 가용성 요구사항을 함께 판단하는 문제다.

## 재연결: connection만 다시 열면 끝나는 것이 아니다

AMQP 클라이언트에서 자주 놓치는 부분은 connection과 channel의 관계다. 보통 RabbitMQ 클라이언트는 하나의 connection 위에 여러 channel을 만들고, channel을 통해 publish와 consume을 수행한다.

```text
AMQP Connection
  ├── Channel for Publisher
  ├── Channel for Consumer A
  └── Channel for Consumer B
```

문제는 connection이 끊겼을 때다. connection이 끊기면 그 위에 있던 channel도 더 이상 정상 상태라고 보기 어렵다. channel을 통해 등록한 consumer, publisher confirm listener, return handler도 함께 복구 대상이 된다.

그래서 재연결은 단순히 “connection을 다시 만든다”가 아니다. 실제로는 다음 상태를 다시 구성하는 과정에 가깝다.

```text
1. connection drop 감지
2. 기존 channel 사용 중단
3. 새 connection 생성
4. 새 channel 생성
5. 필요한 exchange / queue / binding 재선언
6. publisher confirm 설정 복구
7. return handler 복구
8. consumer 재등록
9. 정상 처리 재개
```

여기서 topology recovery가 중요하다. exchange, queue, binding은 RabbitMQ에 이미 남아 있을 수 있지만, 애플리케이션 입장에서는 필요한 topology가 준비되어 있다고 가정하면 위험할 수 있다. 특히 auto-delete queue, exclusive queue, temporary queue, consumer registration은 연결 상태에 민감하다.

클라이언트 라이브러리가 자동 복구를 지원하더라도, 어떤 상태를 자동으로 복구하고 어떤 상태는 애플리케이션이 직접 복구해야 하는지 확인해야 한다. 자동 복구가 있다는 이유만으로 consumer가 항상 정상적으로 다시 등록된다고 가정하면 운영 중 원인을 찾기 어려운 공백이 생길 수 있다.

정리하면 AMQP 재연결은 네트워크 연결만 다시 여는 작업이 아니다. publish와 consume에 필요한 논리 상태를 다시 구성하는 작업이다.

## Multi-endpoint failover: 클러스터를 쓴다면 클라이언트도 클러스터를 전제로 해야 한다

RabbitMQ 클러스터가 여러 노드로 구성되어 있어도, 애플리케이션이 항상 하나의 endpoint만 바라본다면 여전히 단일 장애점이 남을 수 있다.

```text
amqp://rabbitmq-node-a
```

이 상태에서 node-a가 유지보수나 재시작으로 내려가면, node-b와 node-c가 살아 있어도 애플리케이션은 계속 node-a로만 접속을 시도할 수 있다. RabbitMQ 클러스터 자체는 살아 있어도 애플리케이션 입장에서는 MQ가 죽은 것처럼 보일 수 있다.

이를 줄이기 위해 multi-endpoint 전략을 사용한다.

```text
amqp://rabbitmq-node-a
amqp://rabbitmq-node-b
amqp://rabbitmq-node-c
```

선택 방식은 환경마다 다를 수 있다. 순차 시도, random selection, round-robin, DNS 기반 endpoint, load balancer 기반 endpoint, 명시적 node list 등 여러 방식이 있다. DNS나 load balancer는 설정이 단순하지만 장애 반영 지연이나 연결 유지 방식을 고려해야 하고, 명시적 node list는 장애 대응이 직접적일 수 있지만 설정 관리가 복잡해질 수 있다.

중요한 것은 RabbitMQ를 클러스터로 운영한다면 애플리케이션의 연결 전략도 클러스터를 전제로 해야 한다는 점이다. 브로커는 HA인데 클라이언트가 single endpoint만 바라보면 전체 시스템 관점에서는 여전히 약한 지점이 남는다.

## Consumer 격리: 하나의 구독자 문제가 전체 소비 중단으로 번지지 않게 하기

하나의 서비스가 여러 queue를 구독하는 경우, consumer를 어떻게 묶어서 관리할지 결정해야 한다.

```text
platform-service
  ├── consume user.created
  ├── consume order.completed
  ├── consume payment.failed
  ├── consume settlement.requested
  └── consume notification.requested
```

단순한 구현에서는 하나의 consumer manager가 모든 구독을 관리할 수 있다. 하지만 이 경우 하나의 channel이나 consumer에서 문제가 발생했을 때 전체 consumer manager를 재시작하게 될 수 있다.

```text
payment.failed consumer channel closed
→ 전체 consumer manager 재시작
→ 관련 없는 user.created/order.completed 소비도 함께 중단
```

반대로 subscriber 단위로 격리하면 장애 범위를 줄일 수 있다.

```text
payment.failed consumer만 재연결
나머지 consumer는 유지
```

물론 이 방식도 비용이 있다. connection/channel 수가 늘어날 수 있고, 상태 관리가 복잡해지며, readiness 판단도 어려워진다. 예를 들어 다섯 개 consumer 중 네 개는 정상이고 하나만 복구 중일 때 이 Pod을 ready로 볼 것인지 결정해야 한다.

따라서 중요한 것은 “항상 subscriber 단위로 분리해야 한다”가 아니다. 어떤 consumer가 죽었을 때 전체 기능을 멈춰야 하는지, 아니면 해당 구독만 복구하면 되는지 판단해야 한다.

Consumer 설계는 메시지를 받는 코드의 문제가 아니라, 장애 범위를 어디까지 허용할 것인지에 대한 운영 설계다.

## Backoff와 Jitter: 장애 중 재시도가 복구를 방해하지 않게 하기

RabbitMQ 장애 상황에서 가장 위험한 패턴 중 하나는 즉시 재시도다.

```text
connect failed
retry immediately
connect failed
retry immediately
connect failed
retry immediately
```

장애가 길어질수록 애플리케이션은 계속 RabbitMQ를 두드리고, RabbitMQ가 복구되는 순간 수많은 connection attempt, channel creation, topology declaration, consumer registration, publish retry가 한꺼번에 몰릴 수 있다.

이를 줄이기 위해 backoff가 필요하다.

```text
1s → 2s → 4s → 8s → 16s → max 30s
```

하지만 exponential backoff만으로는 충분하지 않을 수 있다. 여러 Pod이 같은 시점에 실패하면 같은 backoff 스케줄을 타게 되고, 결국 같은 시점에 다시 몰릴 수 있기 때문이다.

```text
Pod A: 1s → 2s → 4s → 8s
Pod B: 1s → 2s → 4s → 8s
Pod C: 1s → 2s → 4s → 8s
```

그래서 jitter가 필요하다. Jitter는 backoff 시간에 랜덤성을 섞어 재시도 시점을 분산한다.

```text
backoff window = 8s

Pod A: 2.1s 뒤 재시도
Pod B: 6.8s 뒤 재시도
Pod C: 4.3s 뒤 재시도
Pod D: 7.5s 뒤 재시도
```

AMQP 기반 시스템에서 jitter를 적용할 지점은 consumer 재연결만이 아니다. RabbitMQ connection retry, channel recovery, publisher retry, outbox relay polling, DLQ 재처리 worker, topology declare retry 모두 재시도 폭주를 만들 수 있다.

소비자는 조심스럽게 backoff하는데 publisher가 즉시 재시도하거나, outbox relay가 MQ 장애 중에도 짧은 주기로 tight polling을 계속하면 전체적으로는 여전히 RabbitMQ에 부하를 줄 수 있다. 따라서 backoff와 jitter는 RabbitMQ를 호출하는 모든 경로에 적용해야 하는 복구 제어 장치로 보는 편이 안전하다.

## 메시지 신뢰성: 보냈다, 받았다, 처리했다를 구분하기

RabbitMQ를 사용할 때 “메시지를 보냈다”와 “메시지가 안전하게 큐에 들어갔다”는 같은 말이 아니다. 마찬가지로 “메시지를 받았다”와 “처리를 완료했다”도 같은 말이 아니다.

Publisher Confirm은 RabbitMQ가 publisher에게 메시지 처리 결과를 알려주는 메커니즘이다. 이를 사용하면 애플리케이션은 `Publish()` 함수 호출 성공이 아니라 RabbitMQ가 메시지를 수락했는지를 기준으로 삼을 수 있다.

```text
불안정한 기준:
Publish() 함수를 호출했으니 보낸 것이다.

더 안전한 기준:
RabbitMQ로부터 confirm을 받았으니 브로커가 수락한 것이다.
```

하지만 confirm만으로는 충분하지 않다. 메시지가 exchange에는 도달했지만 어떤 queue에도 라우팅되지 못할 수 있기 때문이다. 이때 mandatory 옵션과 return notification을 사용하면 라우팅되지 않은 메시지를 publisher가 감지할 수 있다.

```text
mandatory=false
  → 라우팅 실패 메시지가 조용히 버려질 수 있음

mandatory=true
  → 라우팅 가능한 큐가 없으면 publisher에게 반환됨
```

Consumer 쪽에서는 manual ACK가 중요하다. 자동 ACK를 사용하면 consumer가 메시지를 받는 순간 RabbitMQ는 그 메시지를 처리된 것으로 볼 수 있다. 이 경우 consumer가 실제 처리 중에 죽으면 메시지가 유실된 것처럼 보일 수 있다.

```text
1. consumer가 메시지를 받음
2. RabbitMQ는 처리 완료로 간주
3. consumer가 실제 처리 중 crash
4. 메시지는 이미 ack된 것으로 처리되어 재전달되지 않음
```

중요한 작업에서는 보통 처리 완료 후 ACK를 보낸다.

```text
1. 메시지 수신
2. 비즈니스 로직 처리
3. DB 저장 또는 외부 작업 완료
4. 성공하면 ACK
5. 실패하면 NACK / reject
```

manual ACK를 사용하면 unacked message가 생긴다. 이때 prefetch는 consumer가 동시에 들고 있을 수 있는 unacked message 수를 제한한다.

```text
prefetch = 10
→ consumer는 ack하지 않은 메시지를 최대 10개까지만 받음
```

Prefetch는 단순한 성능 튜닝 값이 아니다. 장애 상황에서 재전달될 수 있는 메시지의 폭을 제한하는 안정성 장치이기도 하다.

결국 메시지 신뢰성은 다음 기준을 분리해서 보는 문제다.

* publish 호출이 성공했는가
* RabbitMQ가 메시지를 수락했는가
* 메시지가 실제 queue로 라우팅되었는가
* consumer가 메시지를 받았는가
* consumer가 처리를 완료했는가
* ACK가 RabbitMQ에 도달했는가

이 기준을 섞어버리면 장애 상황에서 메시지가 어디까지 처리됐는지 판단하기 어려워진다.

## Requeue와 DLQ: 실패한 메시지를 무조건 다시 넣지 않기

Consumer 처리 실패는 크게 일시적 실패와 영구적 실패로 나누어야 한다.

```text
일시적 실패:
DB timeout, 외부 API timeout, RabbitMQ 일시 장애

영구적 실패:
잘못된 payload, schema 불일치, 검증 실패, 존재하지 않는 참조 데이터
```

일시적 실패는 재시도할 가치가 있다. 이 경우 requeue, retry queue, delayed retry 같은 방식을 고려할 수 있다. 반대로 영구적 실패를 계속 requeue하면 poison message가 된다.

```text
잘못된 메시지 수신
→ 처리 실패
→ requeue
→ 다시 수신
→ 처리 실패
→ requeue
→ 무한 반복
```

이런 메시지는 DLQ로 격리해야 한다. DLQ는 실패 메시지를 버려두는 공간이 아니라, 어떤 메시지가 왜 실패했는지 확인하고 재처리 또는 폐기 판단을 하기 위한 운영 장치다.

따라서 실패 처리의 핵심은 “실패하면 다시 넣는다”가 아니다. 일시 오류와 영구 오류를 구분하고, 재시도할 수 없는 메시지는 메인 처리 흐름에서 분리하는 것이다.

## At-least-once와 멱등성: 같은 메시지는 두 번 올 수 있다

RabbitMQ를 신뢰성 있게 사용하면 보통 at-least-once 처리 모델에 가까워진다. 이는 메시지를 최소 한 번 처리하도록 노력하지만, 중복 처리가 발생할 수 있음을 의미한다.

예를 들어 다음 상황을 생각할 수 있다.

```text
1. consumer가 메시지를 받음
2. 비즈니스 로직 처리 성공
3. ACK 보내기 직전에 connection drop
4. RabbitMQ는 ACK를 받지 못함
5. 메시지를 다시 전달
```

애플리케이션 입장에서는 이미 처리했지만 RabbitMQ 입장에서는 ACK를 받지 못했기 때문에 메시지를 다시 보낼 수 있다. 따라서 consumer는 기본적으로 같은 메시지가 두 번 올 수 있다고 가정해야 한다.

이때 필요한 것이 멱등성이다. 멱등성은 같은 메시지가 여러 번 처리되어도 최종 결과가 의도와 크게 달라지지 않게 만드는 성질이다.

```text
message_id = tx-submit:order-123:attempt-1

이미 처리된 key인가?
  yes → 중복 처리하지 않음
  no  → 처리 진행 후 처리 완료 기록
```

멱등성은 코드에서 `if` 문 하나를 추가하는 문제가 아니다. DB unique constraint, 처리 이력 테이블, outbox 상태, 외부 API idempotency key, business key 설계와 함께 봐야 한다.

특히 결제, 정산, 블록체인 트랜잭션 제출처럼 외부 side effect가 큰 작업에서는 중복 메시지를 어떻게 처리할지 먼저 설계해야 한다.

## Outbox: AMQP만으로 해결되지 않는 dual write 문제

RabbitMQ를 잘 사용해도 해결되지 않는 문제가 있다. 바로 DB 상태 변경과 메시지 publish를 하나의 원자적 작업으로 묶기 어렵다는 점이다.

```text
1. DB에 주문 상태를 PAID로 변경
2. payment.completed 이벤트 publish
```

이 두 작업 사이에서 장애가 나면 문제가 생길 수 있다.

```text
DB 변경 성공
publish 실패
→ 시스템 내부 상태는 PAID인데, 후속 서비스는 이벤트를 받지 못함
```

반대도 가능하다.

```text
publish 성공
DB commit 실패
→ 이벤트는 나갔는데 실제 상태는 반영되지 않음
```

이것이 dual write problem이다.

Transactional Outbox는 이 문제를 줄이기 위한 패턴이다. 핵심은 비즈니스 상태 변경과 “발행해야 할 메시지”를 같은 DB 트랜잭션에 기록하는 것이다.

```text
하나의 DB transaction 안에서:
1. business_table update
2. outbox_table insert

별도 relay가:
3. unpublished outbox row 조회
4. RabbitMQ에 publish
5. confirm 수신 후 published 처리
```

이렇게 하면 DB 상태 변경과 메시지를 발행해야 한다는 사실은 함께 남길 수 있다. RabbitMQ가 잠시 장애 상태여도 outbox row는 DB에 남아 있고, relay가 나중에 다시 publish할 수 있다.

다만 Outbox가 exactly-once를 보장하는 것은 아니다. relay가 publish에 성공한 뒤 published 표시를 하기 전에 죽으면 같은 outbox row가 다시 발행될 수 있다. 따라서 consumer 멱등성은 여전히 필요하다.

정리하면 Outbox는 메시지 유실 가능성을 줄이지만, 중복 가능성을 제거하지는 않는다.

## 실무에서 점검할 질문

RabbitMQ 장애를 분석하거나 설계를 검토할 때는 다음 순서로 질문해보면 좋다.

### 브로커 구조

* queue type은 무엇인가?
* classic queue인가 quorum queue인가?
* cluster node 수는 몇 개인가?
* queue leader가 특정 노드에 몰려 있지 않은가?
* 유지보수 중 어떤 큐가 영향을 받는가?

### 클라이언트 복구

* multi-endpoint를 사용하는가?
* connection drop을 감지하는가?
* channel을 재생성하는가?
* topology를 다시 선언하는가?
* publisher confirm listener와 return handler를 복구하는가?
* consumer를 재등록하는가?

### 재시도 제어

* retry에 최대 지연 시간이 있는가?
* jitter가 있는가?
* 모든 Pod이 같은 타이밍에 재시도하지 않는가?
* publisher와 consumer 모두 backoff를 적용하는가?
* outbox relay가 MQ 장애 중 tight polling하지 않는가?

### 메시지 신뢰성

* publisher confirm을 사용하는가?
* mandatory return을 처리하는가?
* manual ACK를 사용하는가?
* prefetch를 제한하는가?
* 실패 유형별 requeue/DLQ 정책이 있는가?
* consumer가 멱등적인가?

## 마무리

AMQP 기반 시스템에서 RabbitMQ 장애를 다룬다는 것은 RabbitMQ 설정 몇 개를 조정하는 문제만은 아니다.

Queue HA는 브로커의 가용성을 높이는 문제이고, 재연결과 backoff는 클라이언트의 복구성을 높이는 문제다. Publisher Confirm, mandatory return, manual ACK, prefetch, DLQ, outbox, idempotency는 메시지 처리의 신뢰성을 높이기 위한 장치다.

이 관점들이 함께 설계되지 않으면 RabbitMQ의 일시적인 유지보수나 노드 재시작도 애플리케이션 장애로 증폭될 수 있다. 반대로 각 계층의 책임을 분리해서 이해하면, 어디까지는 RabbitMQ가 보장하고 어디서부터는 애플리케이션이 책임져야 하는지 더 명확하게 판단할 수 있다.

핵심은 다음 문장으로 정리할 수 있다.

> RabbitMQ는 메시지 전달의 신뢰성을 높이는 기능을 제공하지만, 장애 상황에서 애플리케이션이 어떻게 재연결하고, 재시도하고, 중복을 감당하고, 유실을 복구할지는 별도로 설계해야 한다.

## 함께 보면 좋은 문서

* [메시지 큐 핵심 개념 정리](./mq-core-concepts.md): producer, consumer, broker, queue, ack/nack 같은 기본 용어를 먼저 확인할 때 참고한다.
* [DLQ 운영 가이드](./dlq-operational-guide.md): 실패 메시지를 어떻게 격리하고 재처리/폐기 판단으로 연결할지 다룬다.
* [RabbitMQ, Kafka, Redis Streams 멀티 컨슈머 의미론 비교](./rabbitmq-kafka-redis-streams-multi-consumer-semantics.md): 같은 이벤트를 여러 소비자가 받아야 할 때 독립 소비 단위를 어떻게 나눌지 함께 볼 수 있다.
* [Kubernetes 단일 writer 소비자 운영 가이드](../kubernetes/kubernetes-single-writer-consumer-operations.md): 메시지 소비자가 Kubernetes에서 공백/중복 문제를 만날 때 함께 참고한다.

