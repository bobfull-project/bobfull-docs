# ChatRoom 생성 실패가 결제·예약 확정까지 롤백시킬 수 있던 문제

> 작성자: 김현승

## 문제 정의

초기 결제 완료 흐름은 다음과 같이 하나의 Transaction이었다.

```text
PortOne 결제 성공
→ Payment PAID
→ Reservation·Participant 생성
→ ChatRoom 생성
→ COMMIT
```

ChatRoom 저장에서 예외가 발생하면 내부 Transaction 전체가 Rollback될 수 있었다. 외부에서는 결제가 성공했지만 내부 Reservation은 없는 상태가 생길 수 있는 구조였다.

부가 기능인 채팅방의 실패가 핵심 결제·예약까지 전파되는 것이 문제였다.

## 가설

핵심 데이터를 먼저 Commit하고 성공한 경우에만 ChatRoom을 별도 처리하면 실패 경계를 나눌 수 있다.

```text
핵심 Transaction 실패
→ ChatRoom 생성 안 함

ChatRoom 실패
→ Payment·Reservation·Participant 유지
```

## V2 해결

- `@TransactionalEventListener(AFTER_COMMIT)`
- `REQUIRES_NEW` 별도 Transaction
- `reservationId` 기준 `createIfAbsent()`
- 구조화 로그 `CHAT_ROOM_CREATION_REQUIRED`
- 조회 시 ChatRoom이 없으면 한 번 복구

```text
Payment PAID
→ Reservation·Participant
→ COMMIT
→ AFTER_COMMIT
→ REQUIRES_NEW ChatRoom
```

## 결과

```text
Before
ChatRoom 실패
→ 결제·예약 전체 Rollback 가능

After
ChatRoom 실패
→ Payment·Reservation·Participant 유지
→ ChatRoom만 복구 대상
```

## 한계와 V3

`AFTER_COMMIT`은 재시도와 이벤트 영속화를 보장하지 않는다. Listener 실행 전 서버가 종료되면 “ChatRoom을 만들어야 한다”는 의도가 사라질 수 있다.

V2에서는 조회 시 `createIfAbsent()` 복구를 두었고, V3에서는 이 한계를 [Transactional Outbox](./after-commit-to-transactional-outbox.md)로 고도화했다.

## 배운 점

> **같이 실행돼야 한다 ≠ 같이 Rollback돼야 한다.**

Transaction은 관련 작업을 모두 묶는 경계가 아니라 **함께 실패해도 되는 작업의 경계**로 봐야 한다.
