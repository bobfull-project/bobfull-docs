# TS-13 — AFTER_COMMIT 후속 작업 유실 가능성과 Transactional Outbox 전환

> 작성자: 김현승

## 문제 정의

ChatRoom을 핵심 결제·예약 Transaction에서 `AFTER_COMMIT`으로 분리해 실패 전파는 막았지만 새로운 유실 구간이 남았다.

```text
Payment / Reservation / Participant COMMIT
→ AFTER_COMMIT Listener 실행 전 App 종료
→ 핵심 데이터는 존재
→ ChatRoom 없음
→ 다시 해야 할 작업 기록도 없음
```

`AFTER_COMMIT`은 **실패 경계와 실행 시점**을 바꾸지만 작업 의도를 DB에 저장하지 않는다.

## 해결 — Transactional Outbox

ChatRoom 자체가 아니라 “이 Reservation의 ChatRoom을 만들어야 한다”는 의도를 핵심 데이터와 **같은 DB Transaction**에 저장한다.

```text
같은 Transaction
├─ Payment / Reservation / Participant
└─ OutboxEvent
   ├─ CHAT_ROOM_CREATION_REQUESTED
   ├─ reservationId
   └─ PENDING
→ COMMIT
```

Reservation이 남으면 Outbox도 남고, Transaction이 Rollback되면 둘 다 남지 않는다.

## Processor 상태와 복구

```text
PENDING
→ PROCESSING
→ COMPLETED
```

실패는 무한 재시도하지 않고 `5 → 10 → 20 → 40 → 80초` backoff 후 제한적으로 재시도하고 계속 실패하면 `FAILED`로 남긴다.

Processor가 `PROCESSING`으로 바꾼 직후 죽는 경우를 위해 5분 이상 처리 중인 이벤트를 stale로 보고 다시 PENDING으로 회수할 수 있게 했다.

## 중복 처리 전략

Outbox를 썼다고 exactly-once가 되는 것은 아니다. Side Effect 수행 후 Outbox 완료 상태 저장 전에 장애가 나면 같은 작업이 다시 실행될 수 있다.

따라서 at-least-once를 허용하고 Handler를 멱등하게 만들었다.

```text
createIfAbsent(reservationId)
+ Reservation 기준 UNIQUE 제약
```

최종 설계 기준은 다음이다.

```text
durable intent
+ at-least-once
+ idempotent handler
```

## 검증

- 핵심 Transaction Rollback → Outbox도 저장되지 않음
- 핵심 Commit → Outbox PENDING 존재
- 다음 처리 주기에서 ChatRoom 생성 → COMPLETED
- 같은 이벤트 중복 처리 → ChatRoom 최종 1건
- backoff `5/10/20/40/80초`
- 반복 실패 → FAILED
- 동시 Claim → 하나의 Processor만 선점
- 5분 이상 stale PROCESSING 회수

이 사례는 실제 운영 ChatRoom 유실 사고가 난 뒤 수정한 것이 아니라 **설계상 유실 구간을 발견하고 결정론적인 실패 시나리오와 자동 테스트로 검증한 사례**다. 당시 JVM kill/restart를 반복해 유실률·복구시간을 측정한 것은 아니므로 그런 수치는 주장하지 않는다.

## 관련 문서

- [ADR-0008 — ChatRoom Transactional Outbox](../../adr/0008-chat-room-transactional-outbox.md)
- [CS-03 — 핵심 거래/후속 작업 Case Study](../../case-studies/cs-03-transaction-and-followup-failure-boundary.md)
