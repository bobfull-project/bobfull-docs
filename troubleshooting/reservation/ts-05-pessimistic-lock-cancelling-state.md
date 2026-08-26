# TS-05 — 비관적 락을 걸었는데도 예약이 CANCELLING에 멈춘 이유

> 작성자: 정용태, 배지현

## 문제 정의

여러 참여자의 환불 완료 요청을 동시에 처리했을 때 다음 상태가 발생했다.

```text
Payment      → 모두 REFUNDED
Refund       → 모두 COMPLETED
Participant  → 모두 CANCELLED
Reservation  → CANCELLING ❌
```

순차 처리에서는 정상이고 동시 처리에서만 재현됐다. Reservation에는 이미 `PESSIMISTIC_WRITE`가 있었기 때문에 단순히 “락이 없다”가 원인은 아니었다.

## 원인

MySQL `REPEATABLE READ`에서 트랜잭션 초반 일반 조회가 만든 **이전 시점 Snapshot**을 마지막 참여자 상태 확인에서도 계속 볼 수 있었다.

실제 DB에서는 A/B/C가 모두 CANCELLED여도 현재 트랜잭션의 일반 조회가 일부 `CANCEL_REQUESTED` 상태를 보아 “아직 취소가 덜 끝났다”고 판단할 수 있었다.

## 해결

Reservation 상태를 최종 결정하는 핵심 조회만 일반 SELECT에서 **`PESSIMISTIC_READ` 잠금 조회**로 변경했다.

```text
Reservation PESSIMISTIC_WRITE
→ 현재 Participant CANCELLED
→ 남은 CANCEL_REQUESTED를 PESSIMISTIC_READ로 조회
→ 없음
→ Reservation CANCELLED
```

전체 락 전략을 바꾼 것이 아니라 **최신 커밋 상태가 필요한 최종 판단 지점**을 잠금 읽기로 바꿨다.

## 검증

실제 MySQL + 다중 Thread 통합 테스트로 동일 환불 완료를 동시에 처리하고 최종 상태가 다음으로 수렴하는지 확인했다.

```text
Payment REFUNDED
Refund COMPLETED
Participant CANCELLED
Reservation CANCELLED
```

## 배운 점

비관적 락을 썼다는 사실만으로 동시성 안전성이 보장되지 않는다. MySQL 격리 수준에서는 **어떤 조회가 먼저 실행됐는지, Snapshot이 언제 만들어졌는지, 마지막 판단이 일반 조회인지 잠금 조회인지**까지 함께 봐야 한다.

## 관련 문서

- [TD-02 — 예약 운영 정책 의사결정](../../decisions/td-02-reservation-auth-operational-decisions.md)
- [TS-10 — 환불 Dual-write 정합성](../payment/ts-10-refund-dual-write-consistency.md)
