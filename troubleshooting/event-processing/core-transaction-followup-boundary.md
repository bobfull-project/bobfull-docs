# 핵심 거래와 후속 작업의 실패 경계를 어떻게 나눌 것인가

> 작성자: 김현승 · 5분 기록 보드의 상세 종합 기록

이 문서는 ChatRoom → AFTER_COMMIT → Transactional Outbox → Email Async → Kafka로 이어진 세부 판단을 한 흐름으로 보존한다. 최종 발표용 정리본은 [Case Study](../../case-studies/transaction-and-followup-failure-boundary.md)에서 확인한다.

## 1. 핵심 거래와 부가 기능의 실패 경계

외부 PortOne 결제 이후 Payment·Reservation·Participant·ChatRoom을 하나의 Transaction으로 묶으면 ChatRoom 실패가 이미 성공한 결제·예약까지 되돌릴 수 있었다.

`AFTER_COMMIT`으로 핵심 거래를 먼저 확정해 실패 전파를 막았지만 Listener 실행 전 종료 시 작업 의도가 사라지는 문제가 남았다.

## 2. Outbox — 해야 할 작업 자체를 저장

```text
Reservation / Participant
+ OutboxEvent(PENDING)
→ 같은 Transaction COMMIT
```

Outbox는 `PENDING → PROCESSING → COMPLETED`, 실패 backoff, FAILED, stale PROCESSING 회수, 멱등 Handler를 통해 재처리 가능한 작업 경계를 만든다.

## 3. Async — Outbox와 별개로 실행 Thread를 분리

Email에서 Outbox를 사용하고 있어도 SMTP를 요청 Thread에서 직접 실행할 수 있음을 발견했다. 따라서 Email은 별도 bounded Executor를 사용한다.

```text
Outbox = 작업을 잊지 않음
Async  = 요청 Thread와 실행 Thread 분리
```

## 4. Outbox + Async vs Kafka 직접 비교

AI 후속 처리에서 Kafka가 정말 필요한지 같은 조건으로 비교했다.

```text
메시지 30건
가짜 AI 처리 500ms
동시 처리 3
Kafka Partition 3
Key = messageId
```

| 항목 | Outbox + Async | Outbox + Kafka |
|---|---:|---:|
| 전체 처리 완료 중앙값 | **5.394s** | 7.210s |
| 처리량 | **5.56 msg/s** | 4.16 msg/s |
| 최종 유실 | 0 | 0 |
| 최종 결과 중복 | 0 | 0 |

[실험 Evidence](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/274-outbox-async-vs-kafka/README.md)

현재 조건에서는 Async가 더 빨랐다. 따라서 **성능 때문에 Kafka를 선택한 것이 아니다.** 두 방식 모두 Outbox를 사용했고 최종 중복 0은 Kafka delivery가 exactly-once라는 뜻이 아니라 Handler 결과가 멱등하게 수렴했다는 의미다.

## 5. Kafka — 여러 후속 기능의 독립 처리 경계

하나의 `ChatMessageCreatedEvent`를 AI Moderation과 Restaurant Feedback 분석이 서로 다른 Consumer Group으로 처리한다.

```text
ChatMessageCreatedEvent
→ Kafka
  ├─ Moderation Consumer
  └─ Restaurant Insight Consumer
```

한 Consumer의 실패·적체가 다른 Consumer를 막지 않고, 새 후속 기능을 독립적으로 추가할 수 있는 점이 Kafka 선택 이유였다.

최종 선택:

```text
ChatRoom → Outbox + 내부 Processor
Email    → Outbox + Async
AI       → Outbox + Kafka
```

모든 후속 작업을 한 기술로 통일하지 않고 필요한 보장에 맞춰 경계를 선택했다.

## 관련 문서

- [Outbox + Async vs Kafka Case Study](../../case-studies/outbox-async-vs-kafka.md)
- [AFTER_COMMIT·Outbox·Kafka 선택 기준](../../case-studies/post-payment-processing-strategy.md)
- [Performance 비교](../../performance/async-vs-kafka.md)
