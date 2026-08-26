# 핵심 거래와 후속 작업의 실패를 어떻게 나눴는가

> 원본 분류: `[발표]` 트러블슈팅 · 작성자: 김현승

## 1. 문제 — 채팅방 실패가 결제·예약까지 되돌릴 수 있었다

초기에는 PortOne 결제가 성공한 뒤 내부에서 다음을 하나의 트랜잭션으로 처리했다.

```text
PortOne 결제 성공
→ Payment 확정
→ Reservation 생성
→ Participant 생성
→ ChatRoom 생성
→ COMMIT
```

ChatRoom 저장이 실패하면 내부 트랜잭션이 롤백된다. 외부 PortOne에서는 이미 결제가 성공했는데 내부 결제·예약 데이터는 사라질 수 있었다.

여기서 기준을 세웠다.

> **같이 실행되어야 하는 작업과 같이 실패해야 하는 작업은 다르다.**

## 2. AFTER_COMMIT — 실패 범위를 먼저 분리

Payment·Reservation·Participant를 먼저 커밋하고 ChatRoom은 `AFTER_COMMIT` 이후 별도 트랜잭션에서 생성하도록 바꿨다.

```text
핵심 거래 COMMIT
→ AFTER_COMMIT
→ REQUIRES_NEW
→ ChatRoom createIfAbsent()
```

이제 ChatRoom 실패가 핵심 거래를 롤백시키지는 않는다. 그러나 새 문제가 생겼다.

```text
핵심 거래 COMMIT
→ Listener 실행 전 JVM 종료
→ Reservation은 존재
→ ChatRoom 없음
→ 다시 해야 할 작업 기록도 없음
```

`AFTER_COMMIT`은 실행 시점을 분리할 뿐 **해야 할 작업을 영속화하지 않는다.**

## 3. Transactional Outbox — 작업 의도를 같은 트랜잭션에 저장

ChatRoom 자체를 핵심 거래에 다시 넣는 대신 `CHAT_ROOM_CREATION_REQUESTED`라는 작업 의도를 Outbox에 같이 저장했다.

```text
같은 DB Transaction
├─ Payment / Reservation / Participant
└─ OutboxEvent(PENDING)
→ COMMIT

PENDING → PROCESSING → COMPLETED
```

핵심 확정 로직은 반드시 상위 트랜잭션 안에서만 실행되도록 `MANDATORY`를 사용했고, Outbox는 실패 시 `5 → 10 → 20 → 40 → 80초` backoff를 적용해 제한적으로 재시도한다. 반복 실패는 `FAILED`, 5분 이상 `PROCESSING`에 고착된 이벤트는 stale 작업으로 회수한다.

중복 처리는 발생할 수 있다고 가정하고 `createIfAbsent(reservationId)`와 DB UNIQUE 제약으로 Handler를 멱등하게 만들었다.

```text
durable intent
+ at-least-once
+ idempotent handler
```

## 4. Outbox를 넣어도 SMTP는 요청 스레드를 막을 수 있었다

Outbox는 데이터와 재처리 근거를 저장하지만 실행 스레드까지 자동 분리하지 않는다. AWS k6 성능 테스트 중 이메일 SMTP가 요청 스레드에서 동기 처리되는 문제를 확인했다.

당시 SMTP는 평균 약 `0.5~1초`, p99 약 `1.5초`였고 fixture 준비가 시나리오당 `7~15분` 걸리며 Application Health DOWN도 관측됐다.

이메일 전용 bounded Executor(`Worker 2 / Queue 100 / AbortPolicy`)를 추가하고 SMTP connect/read/write timeout을 각각 5초로 제한했다. Executor가 포화되면 Outbox는 `PENDING`으로 남아 Scheduler가 다시 처리한다.

수정 후 느린 Processor를 인위적으로 둔 테스트에서 Dispatcher가 **500ms 미만에 반환**하는 구조적 분리를 검증했다. 다만 **수정 후 실제 AWS+SMTP 환경 k6 재측정은 하지 않았으므로** `7~15분 → 특정 시간`, `Health DOWN → 0회` 같은 수치는 주장하지 않는다.

## 5. 기술의 책임을 분리했다

```text
AFTER_COMMIT
→ 핵심 거래 성공 뒤에만 후속 작업 시작
→ 실패 범위 분리

Transactional Outbox
→ 해야 할 작업을 DB에 남김
→ 재처리 근거 제공

Async Executor
→ 요청 스레드와 느린 외부 I/O 실행 스레드 분리

Kafka
→ 독립 Consumer, 적체·실패 격리·재처리·확장 경계
```

모든 후속 작업을 Kafka로 통일하지 않았다. ChatRoom은 Outbox 내부 처리기, Email은 Outbox + Async, AI는 Outbox + Kafka로 각 요구에 맞춰 나눴다.

## 관련 문서

- [ChatRoom 실패가 핵심 거래까지 롤백시키던 문제](../troubleshooting/event-processing/chatroom-rollback-boundary.md)
- [AFTER_COMMIT에서 Transactional Outbox로](../troubleshooting/event-processing/after-commit-to-transactional-outbox.md)
- [Email Outbox 동기 처리 지연](../troubleshooting/event-processing/email-outbox-request-latency.md)
- [ADR 0008 — ChatRoom Transactional Outbox](../adr/0008-chat-room-transactional-outbox.md)
