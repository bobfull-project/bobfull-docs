# ADR 0010: Kafka 적용 범위 결정 — AI 후속 처리에만 Outbox + Kafka 적용

> BobFull Backend의 전체 ADR 19개 중 포트폴리오에서 보여줄 대표 의사결정을 요약한 문서입니다. 상세 원본은 [Backend ADR](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/adr/0010-chat-message-outbox-kafka-pipeline.md)을 기준으로 합니다.

## 문제

채팅 저장 경로에서 외부 AI를 동기 호출하면 AI 지연·장애가 실시간 채팅에 전파됩니다. 반대로 Kafka에 직접 발행하면 DB Commit과 Broker publish 사이에 작업 의도를 영속적으로 남기지 못하는 구간이 생깁니다.

또한 Kafka를 도입하면서 예약·결제·환불·ChatRoom·이메일 같은 다른 후속 작업까지 모두 Kafka로 전환할 필요가 있는지도 함께 판단해야 했습니다.

## 결정

`ChatMessage`와 `OutboxEvent(CHAT_MESSAGE_CREATED)`를 같은 DB 트랜잭션에 저장하고, Outbox Processor가 Kafka에 발행한 뒤 Broker ACK를 받으면 `COMPLETED`로 전이합니다.

- DB → Broker 전달 의도: Transactional Outbox
- Broker 이후 처리: Kafka Consumer Group
- AI 실패: 최초 처리 포함 최대 3회 재시도 후 DLT 격리
- Spring AI 내부 retry는 `max-attempts=1`로 두어 중첩 재시도 방지
- Moderation partition key는 `messageId`
- Moderation은 Kafka 소비 순서에 의존하지 않으며, Split Message 판단에 필요한 이전 메시지는 현재 `messageId`를 기준으로 DB `ChatMessage` 이력을 명시적으로 정렬해 재구성
- 따라서 방 단위 순서를 보장하는 `chatRoomId` 대신 메시지 단위 병렬성을 확보할 수 있는 `messageId`를 partition key로 선택

Kafka는 프로젝트 전체 비동기 처리에 공통 적용하지 않고, **AI 후속 처리에 한정해 사용**합니다.

## 왜 Kafka를 유지했나

#274에서 양쪽 모두 Transactional Outbox를 사용하는 동일 조건으로 다시 비교했습니다.

| 지표 | Outbox + Async | Outbox + Kafka |
|---|---:|---:|
| Drain median | **5.394s** | **7.210s** |
| Throughput median | **5.56 msg/s** | **4.16 msg/s** |
| process crash 후 lost / duplicate | 0 / 0 | 0 / 0 |

따라서 **Kafka가 더 빠르거나 Kafka만이 유실을 막기 때문에 채택한 것이 아닙니다.** 단순 처리 속도는 Async가 더 빨랐습니다.

Kafka를 유지한 이유는 Broker backlog, Consumer Group, Lag 관찰, Retry/DLT, 향후 독립 Worker 확장처럼 **AI 후속 작업을 별도 운영 경계로 관리할 수 있기 때문**입니다.

## 왜 AI 후속 처리에만 Kafka를 적용했나

Kafka 적용 여부는 단순히 `비동기 작업인가`가 아니라 **독립 Consumer, 적체 관리, Retry/DLT, 재처리·확장 경계가 필요한가**를 기준으로 판단했습니다.

- 예약: 요청 시점의 좌석 정합성이 핵심이므로 DB Transaction·Lock으로 처리
- 결제: 외부 결과 검증 후 내부 상태를 즉시 원자적으로 확정해야 하므로 DB Transaction으로 처리
- 환불: timeout 시 외부에서 이미 성공했을 수 있어 단순 Retry 대신 상태 조회·Reconciliation 적용
- ChatRoom: 작업 의도 보존과 재처리는 필요하지만 Outbox + 내부 Processor로 충분
- Email: 유실 방지와 수신자별 재시도가 핵심이므로 Outbox + 내부 Processor 사용
- 실시간 채팅 전파: durable 처리보다 다중 인스턴스 fan-out이 목적이므로 Redis Pub/Sub 사용
- AI 후속 처리: 느린 외부 작업의 적체·실패·재처리·독립 확장이 필요해 Outbox + Kafka 적용

즉 **Kafka를 모든 비동기 처리에 공통 적용하지 않고, 추가 운영 복잡도를 감수할 가치가 명확한 AI 후속 처리에 제한적으로 적용했습니다.**

## 트레이드오프

Kafka Broker·Topic·Consumer·Retry/DLT 운영 복잡도가 추가되고, 현재 Broker는 단일 EC2의 단일 KRaft 구성이라 메시징 계층 HA까지 보장하지 않습니다.

또한 `messageId`를 partition key로 사용하면서 **같은 채팅방 메시지의 Moderation 완료 순서는 보장하지 않습니다.** 대신 현재 Moderation은 Kafka 순서를 정합성 기준으로 사용하지 않고, 필요한 Split Message 이력은 DB를 기준으로 재구성하므로 방 단위 순서 보장보다 Consumer 병렬성을 선택했습니다.
