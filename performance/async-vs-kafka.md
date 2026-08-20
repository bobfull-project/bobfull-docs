# Outbox + Async vs Outbox + Kafka

## 질문

Transactional Outbox를 공통으로 둔 상태에서 Local Async와 Kafka가 실제로 어떻게 다른지 비교했습니다.

단순히 `Memory Async`와 `Outbox + Kafka`를 비교하면 Outbox가 제공하는 내구성과 Kafka의 효과가 섞이므로, **두 방식 모두 동일한 Outbox를 사용**하도록 조건을 맞췄습니다.

## 성능 결과

30 messages, Fake AI 500ms, concurrency 3 조건의 반복 측정 중앙값입니다.

| 지표 | Outbox + Async | Outbox + Kafka |
|---|---:|---:|
| Drain | **5.394s** | 7.210s |
| Throughput | **5.56 msg/s** | 4.16 msg/s |
| 정상 처리 lost | 0 | 0 |
| 정상 처리 duplicate | 0 | 0 |

이 workload에서는 Async가 더 빨랐습니다. 따라서 **Kafka를 처리 속도 때문에 선택했다는 주장은 사용하지 않습니다.**

## 프로세스 종료·복구

실제 child JVM을 강제 종료한 뒤 재기동했습니다.

| 지표 | Outbox + Async | Outbox + Kafka |
|---|---:|---:|
| restart → 처리 재개 | 296.825s | **40.614s** |
| restart → 전체 완료 | 301.041s | **47.035s** |
| crash lost | 0 | 0 |
| crash duplicate | 0 | 0 |

Async도 DB Outbox의 stale reclaim을 통해 복구됐습니다. Kafka만이 작업 내구성을 제공하는 것은 아닙니다.

## 판단

Kafka 유지 근거는 성능이나 유일한 유실 방지가 아니라 다음과 같은 **운영 경계**입니다.

- Broker backlog로 적체 분리
- Consumer Group 단위 독립 처리
- Consumer lag 관측
- Worker 확장과 장애 격리
- Retry/DLT 구조로 확장 가능한 실행 경계

즉, 현재 규모에서 더 빠른 구조는 Async였지만 AI 후속 작업이 늘어날 때 적체·복구·관측·확장을 애플리케이션 내부 큐가 아닌 별도 Consumer 경계로 관리하기 위해 Kafka를 유지했습니다.

## 검증 범위

- 성능: H2(MySQL mode) 기반 테스트 환경
- 복구: Testcontainers + child JVM
- 실제 AWS multi-EC2 Kafka 복구시간을 측정한 값은 아님
- 이 실험에서는 Retry/DLT failure injection을 별도로 수행하지 않음

## Evidence / ADR

- [Backend Evidence — Issue #274](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/274-outbox-async-vs-kafka/README.md)
- [ADR — Chat Message Outbox Kafka Pipeline](../adr/0010-chat-message-outbox-kafka-pipeline.md)
