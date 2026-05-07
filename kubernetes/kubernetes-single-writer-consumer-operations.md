# Kubernetes 환경에서 단일 writer처럼 동작해야 하는 메시지 소비 모듈 운영 가이드

메시지 소비 모듈 중에는 “논리적으로는 하나만 처리해야” 하는 컴포넌트가 있습니다. 예를 들어, 특정 키 범위의 순서를 엄격히 보장해야 하거나, 동일 집계 레코드에 대해 동시 write가 들어오면 비즈니스 의미가 깨지는 경우입니다.

이때 가장 먼저 떠올리는 방법이 `replica=1` 단일 Pod 운영입니다. 설정은 간단하지만, 운영 현실에서는 이 방식만으로 요구사항을 만족하기 어렵습니다. 이 문서는 **Kubernetes에서 singleton처럼 보이는 소비자를 실무적으로 어떻게 설계/운영할지** 정리합니다.

## 문제 정의: 왜 `replica=1`이 직관적이지만 불완전한가

Deployment를 `replicas: 1`로 두면 평소에는 한 개 Pod만 떠 있으니 singleton처럼 보입니다. 하지만 운영 중에는 아래 이벤트가 반복됩니다.

- 노드 장애/재부팅
- Pod eviction(자원 압박, 유지보수)
- rolling update
- kubelet / runtime 일시 오류
- readiness/liveness 변화에 따른 재시작

즉, **“대부분의 시간에 1개”와 “절대로 1개만”은 다릅니다.**

## 핵심 전제: Kubernetes 기본 설정만으로는 비즈니스 single-writer 보장이 충분하지 않다

Kubernetes의 목표는 워크로드 생존성과 수렴(convergence)입니다. 현재 상태가 기대 상태와 다르면 다시 맞춰 가는 모델이기 때문에, 기본 워크로드 primitives만으로 분산 잠금 수준의 “전역 단일 실행 보증”을 만들기는 어렵습니다.

실무적으로 기억할 점은 두 가지입니다.

1. **공백(window)**: 기존 Pod 종료와 신규 Pod 기동 사이에 소비 공백이 생길 수 있습니다.
2. **중복(window)**: 종료 지연, 네트워크 지연, lease handoff 타이밍 때문에 짧은 중복 처리 구간이 생길 수 있습니다.

그래서 singleton 요구사항은 오케스트레이터 설정만으로 끝내기보다, **애플리케이션/데이터 계층의 안전장치**까지 포함해 설계해야 합니다.

## "DB 적재 후 브로드캐스팅" 순서는 맞다, 하지만 side effect 분리 문제는 남는다

많이 쓰는 흐름은 아래입니다.

1. 메시지 consume
2. DB write
3. 외부 publish/broadcast

순서 자체는 타당합니다. 다만 `DB write`와 `publish`는 서로 다른 시스템에 대한 **분리된 side effect**입니다. 하나의 원자 트랜잭션이 아니기 때문에 중간 장애가 나면 불일치가 생깁니다.

대표적으로:

- DB write 성공 후 publish 직전에 프로세스 크래시 → DB에는 반영됐는데 이벤트는 안 나감
- publish 성공 후 ack 전에 크래시 → 재기동 후 같은 메시지 재처리로 중복 publish 가능

따라서 “순서가 맞다”와 “항상 일관된다”는 별개의 문제입니다.

## 대표 장애 시나리오

운영에서 자주 보는 시나리오를 짧게 정리하면:

1. **노드 장애로 Pod 유실**
   - 새 Pod 스케줄링까지 수 초~수십 초 공백 발생
   - broker 재밸런싱까지 겹치면 지연 체감 증가

2. **rolling update 중 소비자 공백**
   - preStop/terminationGracePeriod 설정이 약하면 in-flight 처리 손실 위험

3. **중복 처리 구간**
   - rolling update, readiness 전환, consumer group rebalance, leader handoff, ack 경계 타이밍이 겹칠 수 있음
   - 그 결과 동일 메시지 중복 소비/중복 write 가능

4. **DB write / publish 불일치**
   - 한쪽만 성공한 상태에서 장애
   - 다운스트림이 상태 재구성 불가

5. **재처리 시 부작용 중복**
   - 외부 API 호출, 알림 발송, 포인트 적립 등 비가역 부작용 중복 발생

## A안: "몇 초 failover 공백 허용 가능"할 때의 권장 운영

대부분의 백오피스/일반 도메인은 이 범주에 들어갑니다. 이 경우 목표는 "절대 무중단"이 아니라 **짧은 공백 + 의미 보존**입니다.

### 1) replica 2~3 + leader election

- 소비 인스턴스는 2~3개를 띄우되
- 실제 active writer는 leader 하나만 동작
- 나머지는 standby로 유지

이 구조는 failover 시간을 줄이고, 단일 Pod 장애에 대한 복원력을 높입니다.

### 2) idempotent consumer

중복 소비는 전제하고, 처리 결과가 한 번만 반영되게 만듭니다.

- 메시지 불변 `event_id` 기반 dedupe를 기본으로 적용
- "이미 처리됨" 기록 테이블/캐시
- upsert/conditional update 활용

핵심은 **같은 메시지를 두 번 처리해도 최종 상태가 같아야** 한다는 점입니다.

### 3) outbox pattern

DB 반영과 이벤트 발행을 느슨하게 연결해 불일치를 줄입니다.

- 비즈니스 트랜잭션에서 도메인 변경 + outbox 레코드를 **같은 DB 트랜잭션**으로 저장
- outbox relay가 outbox를 읽어 broker로 publish
- publish 성공 시 outbox 상태 갱신

outbox relay는 별도 프로세스일 수도 있고, 같은 애플리케이션 내부의 분리된 워커일 수도 있습니다.

이렇게 하면 "DB만 성공하고 publish 누락" 문제를, 유실 위험을 재시도 가능한 지연 문제로 전환해 다룰 수 있습니다.

### 4) graceful shutdown

- `preStop` 훅에서 소비 중단 신호 전달
- in-flight 처리 마무리
- commit/ack 타이밍 명확화
- `terminationGracePeriodSeconds`를 실제 처리 시간에 맞게 조정

종료를 잘 다루면 update/eviction 시 손실과 중복이 크게 줄어듭니다.

### 5) PDB / PriorityClass는 보조수단

- PDB는 "동시 축출" 완화용
- PriorityClass는 자원 경합 시 생존성 보강용

둘 다 유용하지만, **singleton 정확성 보장 수단은 아닙니다.**

## B안: "1초 공백도 안 되고, 중복도 절대 불가"일 때

이 요구사항은 일반적인 K8s singleton Pod 패턴으로 풀기 어렵습니다. 사실상 매우 강한 failover 요구와 중복 불가 요구를 동시에 만족해야 하는 영역이기 때문입니다.

현실적인 대안은 아래처럼 **제어 지점을 외부화/분리**하는 것입니다.

- **DB lock 기반 제어**: 행 잠금/advisory lock으로 active writer 단일화
- **external coordinator**: ZooKeeper/etcd/Consul 같은 코디네이터에 리더십 위임
- **dedicated VM/고정 런타임**: 오케스트레이션 변동성을 줄인 전용 실행 환경
- **managed service 활용**: 클라우드 매니지드 큐/스트림의 강한 보장 기능 사용
- **broker-native ordering 활용**: 파티션 키/싱글 파티션 등 브로커 순서 보장 기능에 설계 맞춤
- **아키텍처 분리**: 초강한 보장이 필요한 경로만 별도 파이프라인으로 격리

요점은 Kubernetes 기본 기능만으로 요구 SLO를 충족한다고 가정하면 위험하며, **요구 SLO에 맞는 제어면(control plane)을 별도로 설계해야 한다**는 것입니다.

## Outbox pattern을 조금 더 구체적으로

outbox는 "원자성 없는 두 side effect(DB, publish)"를 다루기 위한 운영 패턴입니다.

구성 요소:

1. `business_table` (실제 도메인 상태)
2. `outbox_table` (발행할 이벤트 payload, 상태, 재시도 횟수, 생성 시각)
3. `relay worker` (outbox polling 또는 CDC 기반 발행기)

동작:

- 애플리케이션 트랜잭션에서 business 변경 + outbox insert 동시 커밋
- relay가 미발행 outbox를 읽고 publish
- 성공 시 `published_at` 기록
- 실패 시 backoff 재시도, DLQ/알람 연계

장점은 "유실"보다 "지연"으로 문제를 변환한다는 점입니다. 즉시성은 조금 희생되지만 복구 가능성이 높아집니다.

## Idempotency를 조금 더 구체적으로

idempotency는 중복 처리 현실을 흡수하는 핵심 장치입니다.

실무 팁:

- 메시지에 불변 `event_id`를 넣고 unique 제약으로 중복 차단
- 상태 전이를 `현재 상태 + 이벤트 버전`으로 조건부 갱신
- 외부 호출은 idempotency key 지원 API를 우선 사용
- "처리 성공 후 ack" 원칙을 유지하되, 재시도 시 동일 결과 보장

주의할 점은, business key만 dedupe 키로 쓰면 정상적인 재이벤트까지 막을 수 있으므로 `event_id`를 기본 키로 두고 business key는 보조 조건으로 쓰는 편이 안전합니다.

idempotency는 "중복을 허용"하는 게 아니라 **중복이 발생해도 의미가 변하지 않게 하는 설계**라는 점을 기억해야 합니다.

## StatefulSet, PDB 등의 한계

singleton 운영에서 자주 오해되는 부분을 정리하면:

- **StatefulSet**: stable identity/순차 배포에는 유리하지만, "절대 1개만 실행" 보장은 아님
- **PDB**: 자발적 중단의 동시성 제한일 뿐, 장애/강제 종료를 막지 못함
- **Anti-affinity / topology spread**: 가용성 분산 도구이지 단일성 보장 도구가 아님

즉, K8s 오브젝트는 **가용성과 운영 편의성**을 높이는 도구이고, 비즈니스 의미의 exactly-one writer를 완전히 대체하지는 못합니다.

## 결론

운영 관점에서 더 안전한 질문은 "파드를 안 움직이게 할 수 있나?"가 아닙니다.

> **파드가 움직여도, 재시작돼도, 중복이 잠깐 생겨도 비즈니스 의미가 깨지지 않게 만들 수 있는가?**

정리하면:

- `replica=1`은 출발점일 뿐 최종 해답이 아니다.
- Kubernetes 기본 설정만으로는 비즈니스 의미의 single-writer 보장을 충분히 만들기 어렵다.
- DB write와 publish는 분리된 side effect라 outbox/idempotency가 필수다.
- 수 초 failover 허용이면 "replica 2~3 + leader election + idempotent consumer + outbox + graceful shutdown" 조합이 실무적으로 가장 균형적이다.
- 1초 공백도/중복도 허용 불가면 K8s 기본 패턴 밖의 제어 장치를 포함한 별도 아키텍처가 필요하다.

결국 핵심은 하나입니다. **파드를 고정하려고 애쓰기보다, 파드가 움직여도 시스템 의미가 보존되게 설계해야 한다.**

---

## 짧은 요약

Kubernetes에서 `replica=1` singleton Pod 운영은 간단하지만 공백/중복 가능성을 제거하지 못한다. DB write 후 publish 순서는 타당해도 두 작업은 분리된 side effect라 장애 시 불일치가 생긴다. 실무적으로는 failover를 몇 초 허용할 수 있다면 replica 2~3, leader election, idempotent consumer, outbox pattern, graceful shutdown 조합이 권장된다. 반대로 1초 공백도 없고 중복도 불가한 요구는 일반 K8s singleton 방식만으로 해결하기 어렵고, DB lock·외부 코디네이터·전용 런타임·managed service·broker-native ordering·아키텍처 분리 같은 추가 제어가 필요하다.

## 함께 보면 좋은 문서

- [메시지 큐 핵심 개념 정리](../edd/mq-core-concepts.md): consumer, ack/nack, retry, idempotency의 기본 용어를 확인할 때 참고한다.
- [RabbitMQ, Kafka, Redis Streams 멀티 컨슈머 의미론 비교](../edd/rabbitmq-kafka-redis-streams-multi-consumer-semantics.md): 브로커별 소비자 독립성 단위와 장애 시 적체 위치를 함께 볼 수 있다.
- [DLQ 운영 가이드](../edd/dlq-operational-guide.md): 소비 실패가 누적될 때 운영 판단과 재처리 흐름을 함께 검토할 수 있다.
