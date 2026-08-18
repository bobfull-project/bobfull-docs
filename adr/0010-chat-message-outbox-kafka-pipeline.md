# ADR 0010: AI 후속 처리에 Outbox + Kafka 적용

> BobFull Backend의 전체 ADR 19개 중 포트폴리오에서 보여줄 대표 의사결정을 요약한 문서입니다. 상세 원본은 [Backend ADR](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/adr/0010-chat-message-outbox-kafka-pipeline.md)을 기준으로 합니다.

## 문제

채팅 저장 경로에서 외부 AI를 동기 호출하면 AI 지연·장애가 실시간 채팅에 전파됩니다. 반대로 Kafka에 직접 발행하면 DB Commit과 Broker publish 사이에 작업 의도를 영속적으로 남기지 못하는 구간이 생깁니다.

## 결정

`ChatMessage`와 `OutboxEvent(CHAT_MESSAGE_CREATED)`를 같은 DB 트랜잭션에 저장하고, Outbox Processor가 Kafka에 발행한 뒤 Broker ACK를 받으면 `COMPLETED`로 전이합니다.

- DB → Broker 전달 의도: Transactional Outbox
- Broker 이후 처리: Kafka Consumer Group
- AI 실패: 최초 처리 포함 최대 3회 재시도 후 DLT 격리
- Spring AI 내부 retry는 `max-attempts=1`로 두어 중첩 재시도 방지
- Moderation partition key는 `messageId`

## 왜 Kafka를 유지했나

#274에서 양쪽 모두 Transactional Outbox를 사용하는 동일 조건으로 다시 비교했습니다.

| 지표 | Outbox + Async | Outbox + Kafka |
|---|---:|---:|
| Drain median | **5.394s** | **7.210s** |
| Throughput median | **5.56 msg/s** | **4.16 msg/s** |
| process crash 후 lost / duplicate | 0 / 0 | 0 / 0 |

따라서 **Kafka가 더 빠르거나 Kafka만이 유실을 막기 때문에 채택한 것이 아닙니다.** 단순 처리 속도는 Async가 더 빨랐습니다.

Kafka를 유지한 이유는 Broker backlog, Consumer Group, Lag 관찰, Retry/DLT, 향후 독립 Worker 확장처럼 **AI 후속 작업을 별도 운영 경계로 관리할 수 있기 때문**입니다.

## 적용 범위

Kafka를 모든 비동기 작업에 적용하지 않습니다.

- ChatRoom: Outbox + 내부 Processor
- Email: Outbox + 내부 Processor
- 실시간 채팅 전파: Redis Pub/Sub
- 결제·환불: 외부 멱등성 + 상태 조회 + Reconciliation
- AI Moderation 및 독립 AI 후속 처리: Outbox + Kafka

## 트레이드오프

Kafka Broker·Topic·Consumer·Retry/DLT 운영 복잡도가 추가되고, 현재 Broker는 단일 EC2의 단일 KRaft 구성이라 메시징 계층 HA까지 보장하지 않습니다.
