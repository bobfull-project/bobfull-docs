# TS-07 — READY Payment 만료 검증과 PortOne 호출 사이의 경쟁 조건 (TOCTOU)

> 작성자: 김현승

## 문제 정의

결제 준비 시 READY Payment로 좌석을 10분 임시 선점한다. 사용자가 만료 직전에 결제 완료 API를 호출하면 최초 검증과 실제 상태 변경 사이에 PortOne 네트워크 I/O가 끼어 조건이 달라질 수 있다.

```text
12:10:00  READY 만료
12:09:59  완료 API 진입 → 아직 유효
           ↓ PortOne 단건 조회
12:10:01  PAID 응답 수신 → 실제 변경 시점에는 만료
```

처음 확인한 시점에는 참이던 조건이 사용 시점에는 바뀌는 **TOCTOU(Time Of Check To Time Of Use)** 문제였다.

만료된 선점이 가용 좌석 계산에서 빠진 뒤 다른 사용자가 좌석을 사용했는데, 이전 사용자의 결제를 뒤늦게 확정하면 좌석 정합성이 깨질 수 있다.

## 검토한 방법

PortOne 조회 전에 Payment `PESSIMISTIC_WRITE`를 잡고 외부 응답까지 기다리는 방법도 가능하지만 외부 API가 느려질수록 DB Lock과 Connection을 오래 점유한다.

```text
Lock
→ PortOne 네트워크 대기
→ 상태 변경
→ Unlock
```

이를 피하기 위해 외부 I/O와 내부 상태 변경 Transaction을 분리했다.

## 해결

```text
Payment 일반 조회 / 소유권 확인
→ Transaction 밖에서 PortOne 단건 조회
→ 짧은 Transaction 시작
→ Payment PESSIMISTIC_WRITE
→ 최신 상태·expiresAt 재검증
→ 유효 READY만 PAID + Reservation 확정
```

Lock을 얻은 뒤 현재 시간을 다시 가져와 만료 여부를 확인한다. PortOne은 PAID지만 내부 READY가 이미 만료됐으면 다음을 수행하지 않는다.

- Payment PAID 전환 X
- Reservation 확정 X
- Participant 생성 X

대신 `PAYMENT_COMPENSATION_REQUIRED`로 기록해 외부 결제 성공과 내부 만료가 갈린 상황을 관찰한다.

결제 완료 Transaction 안에서 `READY → EXPIRED`로 바꾼 뒤 예외를 던지는 것도 피했다. 예외 Rollback으로 EXPIRED 변경까지 사라질 수 있기 때문이다. 상태 정리는 만료 Scheduler가 담당한다.

## 검증

고정 `Clock`으로 Lock 획득 시점에는 이미 `expiresAt`이 지난 상황을 구성해 다음을 확인했다.

- 만료 Payment → `PAYMENT_EXPIRED`
- 늦은 `READY → PAID` 전환 방지
- ReservationConfirmationPort 미호출
- 뒤늦은 예약 확정 방지
- 이미 PAID인 Payment는 기존 결과를 반환해 중복 확정 방지

## 배운 점

외부 API 호출 전의 검증을 상태 변경 시점까지 신뢰하면 안 된다. **외부 I/O는 Transaction 밖에서, 실제 변경은 짧은 Lock 안에서 최신 상태를 다시 확인**하는 방식이 현재 요구에 적합했다.

## 관련 문서

- [ADR-0001 — 좌석 정합성](../../adr/0001-reservation-seat-consistency.md)
- [TS-11 — 환불 Timeout Reconciliation](./ts-11-refund-timeout-reconciliation.md)
