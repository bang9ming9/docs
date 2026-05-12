# AI Backend 운영 관찰: Metrics, Readiness, Canary

## 서버가 떠 있다는 것과 작업이 처리된다는 것은 다르다

비동기 AI 작업 서버는 여러 구성 요소가 함께 동작해야 한다. API 서버가 Job을 만들고, Worker가 Job을 가져가고, 외부 AI Provider를 호출하고, 결과를 저장하고, Job 상태를 갱신해야 한다.

이 구조에서는 서버 프로세스가 떠 있다는 사실만으로 전체 작업 흐름이 정상이라고 말하기 어렵다.

API는 정상적으로 응답하지만 Worker가 멈춰 있을 수 있다. Worker는 실행 중이지만 Provider 호출이 계속 실패할 수 있다. Provider 호출은 성공하지만 Object Storage 업로드가 실패할 수 있다. Job 상태는 성공으로 바뀌었지만 실제 결과 파일이 없을 수도 있다.

따라서 Async AI Backend의 운영 관찰은 단순 health check만으로 부족하다. “프로세스가 살아 있는가”와 “작업이 끝까지 처리되는가”를 나눠서 봐야 한다.

## Readiness의 범위를 명확히 해야 한다

Kubernetes 같은 운영 환경에서는 readiness probe를 통해 Pod이 트래픽을 받을 준비가 되었는지 판단한다. Readiness는 중요한 장치지만, 이것이 모든 내부 작업 흐름의 정상성을 보장한다고 보면 위험하다.

예를 들어 HTTP 서버는 정상적으로 요청을 받을 수 있지만, 내부 Scheduler loop가 멈춰 있을 수 있다. 또는 Job Store 연결은 정상이지만 Provider 호출이 계속 실패하고 있을 수 있다. 이 경우 readiness는 성공하더라도 실제 AI 작업은 정상적으로 완료되지 않을 수 있다.

Readiness가 확인하는 범위와 확인하지 못하는 범위를 나눠서 봐야 한다.

```text
Readiness가 확인하기 좋은 것:
- HTTP 요청을 받을 수 있는가
- 필수 설정이 로드되었는가
- 기본 의존성 연결이 가능한가
- 최소한의 DB 또는 Job Store 접근이 가능한가

Readiness만으로 부족할 수 있는 것:
- Scheduler loop가 계속 돌고 있는가
- Worker가 실제 Job을 처리하고 있는가
- Provider 호출이 성공하고 있는가
- 결과 저장과 상태 갱신이 끝까지 성공하는가
```

Readiness는 트래픽을 받을 준비를 판단하는 기준이고, 비동기 작업의 품질은 metrics와 alert로 함께 봐야 한다.

## Scheduler와 Worker가 같은 프로세스에 있을 때

초기 구조에서는 Scheduler나 Worker를 별도 프로세스로 분리하지 않고, AI Backend app process 내부 background thread로 실행할 수 있다. 이 방식은 구현과 배포가 단순하다. 별도 Worker Deployment를 만들지 않아도 되고, 설정도 한 프로세스 안에서 관리할 수 있다.

하지만 이 구조에는 한계가 있다. API 서버와 Scheduler가 같은 프로세스에 있으면 fate sharing이 발생한다. 프로세스가 죽으면 API와 Scheduler가 함께 죽는다. 반대로 HTTP 서버는 살아 있지만 background thread만 멈춘 경우에는 외부에서 감지하기 어려울 수 있다.

또한 스케일링 기준도 애매해질 수 있다. API 트래픽을 처리하기 위해 Pod 수를 늘렸는데, 동시에 Scheduler나 Worker도 늘어나 중복 처리 가능성이 커질 수 있다. 반대로 Worker 처리량만 늘리고 싶은데 API 서버까지 함께 늘려야 할 수도 있다.

이 구조가 항상 잘못된 것은 아니다. 작은 규모나 canary 단계에서는 단순성이 장점이 될 수 있다. 다만 운영 단계에서는 다음 질문을 확인해야 한다.

```text
- Scheduler loop가 실제로 돌고 있는지 어떻게 확인할 것인가?
- 여러 Pod에서 Worker가 동시에 돌 때 같은 Job을 중복 처리하지 않는가?
- API scaling과 Worker scaling을 분리해야 하는 시점은 언제인가?
- Worker가 죽은 뒤 processing 상태 Job을 어떻게 회수할 것인가?
```

이 질문에 대한 답이 없으면 서버는 떠 있지만 작업은 멈춘 상태가 생길 수 있다.

## Canary에서 성공률만 보면 부족하다

AI 기능을 canary로 배포할 때 “요청이 성공하는가”만 보면 부족하다. 외부 AI Provider를 호출하는 구조에서는 성공률 외에도 처리 지연, retry, 중복 호출, 비용 증가를 함께 봐야 한다.

특히 Provider 호출에는 비용이 연결될 수 있다. 기능 성공률은 비슷해 보여도 retry나 중복 실행이 늘어나면 billable count가 예상보다 커질 수 있다. 이 경우 사용자는 문제를 느끼지 못하더라도 운영 비용은 증가할 수 있다.

Canary 단계에서 우선 볼 지표는 다음 정도로 잡을 수 있다.

```text
- Job created count
- Queue backlog
- Processing job count
- Retry count
- Stale processing count
- Provider success/failure count
- Provider latency
- Provider billable count
- Result storage failure count
- Time to completion
```

이 지표들은 각각 다른 질문에 답한다.

```text
Job created count:
- 요청이 정상적으로 접수되고 있는가?

Queue backlog:
- Worker 처리량이 요청량을 따라가고 있는가?

Processing job count:
- 현재 처리 중인 작업이 비정상적으로 쌓이고 있지 않은가?

Retry count:
- 일시 실패가 증가하고 있지 않은가?

Stale processing count:
- 처리 중 상태로 멈춘 Job이 생기고 있지 않은가?

Provider billable count:
- 실제 비용이 예상한 작업 수와 크게 벌어지지 않는가?

Result storage failure count:
- Provider 결과를 서비스 저장소에 보관하는 단계에서 실패하지 않는가?

Time to completion:
- 사용자가 결과를 받기까지 시간이 예상 범위 안에 있는가?
```

이런 지표를 봐야 기능이 단순히 동작하는지뿐 아니라, 운영 가능한 방식으로 동작하는지 판단할 수 있다.

## Backlog와 Stale Processing은 다른 신호다

비동기 Job 처리에서 backlog와 stale processing은 둘 다 “작업이 잘 끝나지 않는다”는 신호처럼 보일 수 있다. 하지만 의미는 다르다.

Backlog는 아직 처리되지 않은 작업이 쌓이는 상태다. 요청량이 Worker 처리량보다 많거나, Provider 응답이 느리거나, Worker 수가 부족할 때 증가할 수 있다.

```text
Backlog 증가가 의미할 수 있는 것:
- 요청량 증가
- Worker 처리량 부족
- Provider 지연
- Queue 소비 지연
```

반면 stale processing은 이미 처리 중으로 표시된 작업이 오래 끝나지 않는 상태다.

```text
Stale processing 증가가 의미할 수 있는 것:
- Worker crash
- Provider 응답 누락
- 상태 업데이트 실패
- lock 또는 lease 회수 실패
- 결과 저장 이후 상태 갱신 누락
```

Backlog가 높으면 처리량 확장이나 요청 제한을 검토할 수 있다. Stale processing이 높으면 상태 전이와 복구 로직을 먼저 확인해야 한다. 둘을 같은 문제로 보면 원인 분석이 흐려진다.

## Readiness와 Metrics의 역할을 분리하기

Readiness와 metrics는 역할이 다르다.

Readiness는 보통 트래픽을 받을 수 있는지 판단하는 장치다. 반면 metrics는 실제 작업 흐름이 정상인지 관찰하는 장치다.

예를 들어 Provider 실패율이 높다고 해서 API 서버 전체를 unready로 만들면, 사용자는 기존 Job 상태 조회도 못 하게 될 수 있다. 반대로 Scheduler가 완전히 멈췄는데 readiness가 계속 성공하면, 새 Job은 계속 접수되지만 처리되지 않는 상황이 생길 수 있다.

따라서 무엇을 readiness에 포함하고, 무엇을 metric과 alert로 볼지 구분해야 한다. 시스템 요구사항에 따라 Scheduler 상태를 readiness에 강하게 연결할 수도 있지만, 그 경우 어떤 조건에서 트래픽을 차단할지 신중하게 정해야 한다.

중요한 것은 readiness가 무엇을 보장하고 무엇을 보장하지 않는지 명확히 남기는 것이다. 그래야 장애 상황에서 “ready인데 왜 작업이 안 도는가” 같은 혼란을 줄일 수 있다.

## Canary에서 확인할 질문

Async AI Backend를 canary로 올릴 때는 다음 질문을 기준으로 확인하면 좋다.

```text
- Job이 정상적으로 생성되는가?
- Worker가 Job을 실제로 가져가는가?
- Processing 상태가 succeeded 또는 failed로 전이되는가?
- Retry가 과도하게 증가하지 않는가?
- Stale processing Job이 남지 않는가?
- Provider 실패율이 예상 범위 안에 있는가?
- Provider billable count가 Job 수와 비정상적으로 벌어지지 않는가?
- 결과 파일이 Object Storage에 저장되는가?
- 사용자가 완료된 결과를 조회할 수 있는가?
- 완료까지 걸리는 시간이 예상 범위 안에 있는가?
```

이 질문들은 완전한 운영 매뉴얼은 아니지만, canary 단계에서 최소한 어떤 관점으로 봐야 하는지 정리하는 기준이 된다.

## 정리

Async AI Backend의 안정성은 서버가 살아 있는지만으로 판단하기 어렵다. API 요청 접수, Scheduler loop, Worker 처리, Provider 호출, 결과 저장, 상태 업데이트가 모두 이어져야 사용자가 결과를 받을 수 있다.

Readiness는 필요하지만 전체 비동기 작업 흐름을 보장하지 못할 수 있다. 특히 Scheduler나 Worker가 app process 내부에서 함께 동작하는 구조라면, HTTP readiness와 작업 처리 상태를 분리해서 봐야 한다.

Canary 단계에서는 성공률뿐 아니라 backlog, retry count, stale processing, provider billable count, result storage failure, time to completion을 함께 봐야 한다.

핵심은 다음과 같다.

> Async AI Backend의 안정성은 프로세스 생존 여부가 아니라, 작업이 접수되고 처리되고 저장되고 관찰 가능한 상태로 끝나는지로 판단해야 한다.
