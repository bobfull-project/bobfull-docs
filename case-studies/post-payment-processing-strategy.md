# 결제 확정 이후 후속 기능을 어떻게 처리할 것인가

## AFTER_COMMIT, Transactional Outbox, Async, Kafka를 나눈 기준

> 원본 분류: `[발표]` 기술적 의사결정 · 작성자: 김현승

## 1. 시작점 — ChatRoom 실패가 핵심 거래를 되돌리면 안 된다

초기 구조는 PortOne 결제 성공 이후 Payment·Reservation·Participant·ChatRoom을 하나의 내부 Transaction으로 처리했다.

```text
PortOne 결제 성공
→ Payment PAID
→ Reservation 생성
→ Participant 생성
→ ChatRoom 생성
→ COMMIT
```

외부 결제는 이미 성공했는데 ChatRoom 실패로 내부 거래가 롤백될 수 있었다. 여기서 첫 기준을 세웠다.

> **같이 실행되어야 하는 작업과 같이 실패해야 하는 작업은 다르다.**

## 2. V2 — AFTER_COMMIT으로 실패 경계를 분리

핵심 거래를 먼저 Commit하고 ChatRoom은 `AFTER_COMMIT → REQUIRES_NEW`에서 생성했다.

```text
AFTER_COMMIT
= 핵심 Transaction이 성공한 뒤 후속 작업을 시작

REQUIRES_NEW
= 후속 저장을 별도 Transaction으로 실행
```

ChatRoom 실패가 Payment·Reservation을 되돌리는 문제는 막았지만 Listener 실행 직전 JVM이 종료되면 “ChatRoom을 만들어야 한다”는 작업 정보가 사라질 수 있었다.

## 3. V3 — 반드시 해야 하는 작업은 Outbox에 저장

ChatRoom 자체를 핵심 Transaction에 넣는 대신 작업 의도를 같은 DB Transaction에 저장했다.

```text
핵심 결제·예약 Transaction
├─ Payment PAID
├─ Reservation
├─ Participant
└─ OutboxEvent(CHAT_ROOM_CREATION_REQUESTED, PENDING)
→ COMMIT
```

Outbox Processor는 `PENDING → PROCESSING → COMPLETED`로 처리하고 실패 시 `5 → 10 → 20 → 40 → 80초` backoff 후 제한적으로 재시도한다. `PROCESSING`에 오래 머문 stale 작업도 회수한다. Side Effect는 at-least-once를 전제로 멱등하게 만든다.

## 4. 이메일 — Outbox와 Async는 다른 문제를 푼다

이메일은 SMTP 외부 I/O다. Outbox를 적용해도 Processor를 같은 요청 스레드에서 호출하면 사용자는 SMTP를 기다릴 수 있다.

```text
Outbox
= 해야 할 작업과 재처리 근거를 DB에 남김

Async
= 사용자 요청 스레드와 외부 I/O 실행 스레드를 분리
```

V3에서는 Email Outbox와 별도 bounded Executor를 사용한다. 여러 수신자 중 일부만 성공하는 경우를 위해 수신자별 상태를 유지하고 실패한 수신자만 재처리한다.

## 5. AI — 왜 Outbox + Async가 아니라 Kafka인가

`Outbox + Async`도 작업 영속화·재시도를 구현할 수 있다. 따라서 “Async는 서버가 죽으면 무조건 유실되므로 Kafka를 썼다”는 설명은 정확하지 않다.

AI 작업에서는 하나의 ChatMessage가 여러 독립 후속 기능으로 fan-out되고, 각 Consumer의 적체와 실패를 분리할 요구가 더 중요했다.

```text
Transactional Outbox
= 작업을 DB에 기억

Async
= App 내부 Thread 경계

Kafka
= 독립 Consumer 전달과 적체·실패·재처리·확장 경계
```

실제 `Outbox + Async`와 `Outbox + Kafka` 비교에서는 현재 조건상 Async가 더 빨랐기 때문에 **Kafka를 성능 우위로 선택하지 않았다.** AI Moderation과 Restaurant Insight가 같은 이벤트를 별도 Consumer Group으로 처리하는 운영 경계가 선택 이유였다.

## 6. 최종 선택

| 작업 | 방식 | 핵심 이유 |
|---|---|---|
| ChatRoom | Outbox + 내부 Processor | 핵심 거래와 실패 분리 + 작업 영속화 |
| Email | Outbox + Async | 작업 영속화 + SMTP 스레드 격리 |
| AI 후속 처리 | Outbox + Kafka | 독립 Consumer·적체·실패 격리 |

모든 비동기 작업을 하나의 기술로 통일하지 않고 **각 작업이 필요한 보장에 맞춰 선택**했다.

## 관련 문서

- [핵심 거래와 후속 작업 Case Study](./transaction-and-followup-failure-boundary.md)
- [Outbox + Async vs Kafka Case Study](./outbox-async-vs-kafka.md)
- [AFTER_COMMIT → Transactional Outbox](../troubleshooting/event-processing/after-commit-to-transactional-outbox.md)
- [Email Outbox 요청 지연](../troubleshooting/event-processing/email-outbox-request-latency.md)
