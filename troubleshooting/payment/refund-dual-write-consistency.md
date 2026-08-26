# 외부 PG–DB Dual-write로 발생한 환불 정합성 문제

> 작성자: 김현승

## 문제 정의

예약 전체 취소에서 여러 참여자를 환불할 때 A의 PortOne 환불은 성공했지만 B 처리 중 예외가 발생해 바깥 DB Transaction이 Rollback되면 외부와 내부 상태가 갈릴 수 있었다.

```text
PortOne A 환불 성공
Refund A = COMPLETED
Payment A = REFUNDED

하지만 바깥 Transaction Rollback
Participant A = RESERVED
Reservation = 기존 상태
```

외부 PG 결과는 Spring Transaction Rollback으로 되돌릴 수 없다.

## 첫 시도 — REQUIRES_NEW

참여자별 환불 DB 처리를 `REQUIRES_NEW`로 분리하면 A의 성공 결과를 B 실패와 독립적으로 Commit할 수 있다. 그러나 Participant·Reservation 변경이 바깥 Transaction에 남아 있으면 외부/내부 전체 정합성은 여전히 깨질 수 있다.

따라서 `REQUIRES_NEW`는 Dual-write 자체의 해결책이 아니라 **부분 성공을 독립적으로 보존하는 도구**로 한정했다.

## 해결 — 상태를 한 번에 원자화하려 하지 않고 수렴시킨다

취소를 세 단계로 나눴다.

```text
1. 취소 접수
Reservation  → CANCELLING
Participant  → CANCEL_REQUESTED
COMMIT

2. Transaction 밖에서 PortOne 환불

3. 확인된 성공만 완료 Transaction
Refund       → COMPLETED
Payment      → REFUNDED
Participant  → CANCELLED
Reservation  → 마지막 참여자 완료 후 CANCELLED
```

PortOne 호출 전 Refund `REQUESTED`도 별도 Transaction으로 저장해 외부 명령의 근거를 남긴다.

## 결과 불명확 처리

Timeout·Connection Reset은 실제 환불 실패를 뜻하지 않는다. 이런 경우 즉시 FAILED나 재환불로 가지 않고 `REQUESTED/PROCESSING`을 유지한다.

- Webhook 또는 Reconciliation 대상
- Scheduler 기본 주기 5분
- 10분 이상 지난 미완료 Refund 재조회
- 자동 Reconciliation 최대 24시간
- 명확한 FAILED는 자동 재환불하지 않고 운영 확인

Webhook과 Scheduler 모두 환불 명령을 다시 보내는 것이 아니라 **PortOne의 실제 상태를 조회해 같은 완료 Service로 수렴**한다.

## 검증

- 앞 참여자 성공 후 다음 참여자 실패
- 참여자별 성공 결과 독립 보존
- Timeout / Connection Reset
- 즉시 응답과 Webhook 동시 완료
- 마지막 환불 완료 후 Reservation 재계산

## 배운 점

Dual-write에서는 거대한 Transaction으로 원자성을 흉내 내기보다 **중간 상태를 명시하고 외부 사실을 확인해 최종 상태로 수렴**시키는 구조가 중요했다.

## 관련 문서

- [환불 Timeout Reconciliation](./refund-timeout-reconciliation.md)
- [Reservation CANCELLING 동시성](../reservation/pessimistic-lock-cancelling-state.md)
- [결제↔예약 의존 경계](../../decisions/payment-reservation-dependency-boundary.md)
