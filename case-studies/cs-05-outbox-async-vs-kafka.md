# CS-05 — Outbox + Async면 충분한데 Kafka까지 필요한가?

> 원본 분류: `[발표]` 기술적 의사결정 · 작성자: 김현승

## 배경

결제 후 이메일에는 `Outbox + Async`를 사용했다. Outbox가 해야 할 작업을 DB에 남기고, Async Executor가 느린 SMTP를 요청 스레드에서 분리한다.

AI 후속 처리도 같은 방식으로 구현할 수 있었기 때문에 단순히 “비동기니까 Kafka”라고 결정하지 않고 두 방식을 같은 조건에서 비교했다.

## 요구사항

1. ChatMessage 저장과 AI 처리의 실패가 서로 영향을 주지 않을 것
2. 하나의 채팅 이벤트를 여러 후속 기능이 각각 처리할 수 있을 것
3. 한 기능의 실패와 적체가 다른 기능의 처리까지 막지 않을 것

## 비교 실험

```text
메시지 수          30건
가짜 AI 처리 시간   500ms
동시 처리 수        3
Kafka Partition     3
Kafka Key           messageId
```

| 측정 항목 | Outbox + Async | Outbox + Kafka |
|---|---:|---:|
| 전체 처리 완료 시간(중앙값) | **5.394초** | 7.210초 |
| 처리량 | **5.56 msg/s** | 4.16 msg/s |
| 최종 유실 | 0 | 0 |
| 최종 결과 중복 | 0 | 0 |

- [실험 Evidence](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/274-outbox-async-vs-kafka/README.md)

현재 실험 조건에서는 **Async가 더 빨랐다.** 이 실험은 Kafka 최대 성능 벤치마크가 아니라 현재 프로젝트 규모에서 “성능 때문에 Kafka가 필요한가”를 확인하기 위한 비교다.

두 방식 모두 Outbox를 사용했기 때문에 장애 시 작업 근거가 DB에 남았다. `최종 결과 중복 0`도 Kafka가 중복 전달을 하지 않는다는 뜻이 아니라 **재처리되더라도 최종 결과를 멱등하게 만들었고 실험에서 중복 결과가 없었다**는 의미다.

## 결정 — 속도가 아니라 처리 경계 때문에 Kafka를 유지

ChatMessage 하나는 AI Moderation과 Restaurant Feedback 분석이 각각 소비한다.

```text
ChatMessageCreatedEvent
          ↓
        Kafka
       ↙     ↘
Moderation   Restaurant Insight
Consumer     Consumer
```

같은 이벤트를 서로 다른 Consumer Group이 소비하게 하면 기능별 적체·실패·재시도를 독립적으로 관리할 수 있다. 한 Consumer의 실패가 다른 Consumer 결과를 막지 않도록 분리할 수 있고, 후속 기능이 늘어날 때 Producer 계약을 바꾸지 않고 새로운 Consumer를 추가할 수 있다.

Consumer 실패는 1초 간격으로 재시도하며 최초 처리를 포함해 최대 3회 시도한 뒤 DLT로 보낸다. 재시도해도 의미가 없는 잘못된 이벤트나 일부 비재시도 예외는 바로 DLT로 보낸다.

- [같은 이벤트 재사용·Consumer Group 분리 Evidence](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/277-restaurant-feedback-event-reuse/README.md)

## 기술 역할

```text
Transactional Outbox
= 작업 유실 방지를 위한 영속 의도

Async Executor
= 같은 애플리케이션 안의 실행 스레드 분리

Kafka
= 독립 Consumer 전달, 적체·실패 격리, 재처리·확장 경계
```

따라서 모든 비동기 작업을 Kafka로 통일하지 않았다.

- ChatRoom → Outbox + 내부 Processor
- Email → Outbox + Async
- AI 후속 작업 → Outbox + Kafka

## 관련 문서

- [PF-04 — Outbox + Async vs Kafka](../performance/async-vs-kafka.md)
- [ADR-0010 — Chat Message Outbox Kafka Pipeline](../adr/0010-chat-message-outbox-kafka-pipeline.md)
- [CS-06 — AFTER_COMMIT·Outbox·Kafka 선택 기준](./cs-06-post-payment-processing-strategy.md)
