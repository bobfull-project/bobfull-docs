# TS-11 — 환불 Timeout 이후 PortOne 상태 재조회로 정합성 복구

> 작성자: 김현승

## 문제 정의

PortOne에서는 실제 환불이 성공했지만 응답이 돌아오기 전에 Timeout/Connection Reset이 발생할 수 있다.

```text
BobFull → PortOne 환불 요청
PortOne → 환불 성공 + cancellationId 생성
응답 전달 → Timeout

내부:
Refund = REQUESTED
Payment = PAID
cancellationId = null
```

Timeout은 **업무 실패가 아니라 결과를 모르는 상태**일 수 있다. 이미 성공했을 가능성이 있는 금전 명령을 자동으로 다시 보내는 것은 위험하다.

## 결정

```text
명확한 실패
→ FAILED

Timeout / Connection Reset / 결과 불명확
→ REQUESTED 유지
→ 환불 재요청 X
→ PortOne 상태 재조회 O
```

복구 경로는 PortOne Webhook과 Reconciliation Scheduler 두 개지만 둘 다 최종적으로 동일한 `RefundCompletionService`로 수렴한다.

## 식별자의 역할

```text
paymentId
→ 어떤 결제인지 식별

idempotencyKey
→ 최초 외부 환불 명령의 중복 실행 방어

cancellationId
→ PortOne이 만든 실제 취소/환불 건 식별
```

## 처리 흐름

```text
짧은 Transaction
Payment Lock
→ 기존 Refund 확인
→ Refund REQUESTED + idempotencyKey 저장
→ COMMIT

Transaction 밖
→ PortOne cancel API

결과 명확 성공
→ 공통 완료

결과 불명확
→ REQUESTED 유지
→ Webhook / Scheduler 재조회
```

### Webhook

내부 cancellationId가 아직 없어도 `paymentId`로 대상 Refund를 찾고 PortOne 상태를 다시 확인한 뒤 공통 완료 처리한다.

### Reconciliation Scheduler

기본 정책:

- fixedDelay 5분
- `REQUESTED | PROCESSING`
- 10분 이상 경과
- 한 번에 최대 20건

Scheduler는 환불 요청을 다시 보내지 않고 기존 `paymentId`로 PortOne Payment를 조회한다. 자동 완료는 다음 조건이 충분히 일치할 때만 한다.

- 전액 취소 상태
- cancellation 완료 시각 존재
- 취소 금액 = Refund.amount
- Refund.amount = Payment.amount
- 요청 시각이 내부 요청과 일치
- 필요한 경우 trigger 확인
- 후보가 정확히 하나

`AMBIGUOUS`, `NOT_COMPLETED`, `LOOKUP_FAILED`는 현재 상태를 유지한다. 해결되지 않는 앞쪽 항목만 계속 선택되는 것을 막기 위해 `lastPgCheckedAt`을 상태 변경 시각인 `updatedAt`과 별도로 둔다.

## 검증

- `REQUESTED + cancellationId=null`에서 paymentId 기반 복구
- 금액 불일치/부분 취소/시각 불일치 자동 완료 방지
- 후보 여러 개 → AMBIGUOUS
- PortOne 조회 실패 시 상태 유지
- 한 Refund 실패가 다음 항목 처리를 막지 않음
- Scheduler/Webhook 동시 완료 시 최종 1회 반영
- 이미 완료된 Refund 멱등 종료
- FAILED 자동 재환불 제외

## 성격과 한계

실제 운영 중 중복 환불 사고가 발생한 뒤 수정한 사례가 아니라 **구현 과정에서 결과 불명확 경계를 발견해 복구 경로를 설계·테스트한 신뢰성 보강 사례**다.

PortOne 조회 결과가 계속 애매하면 자동 판단하지 않고 운영자 확인이 필요하다. 금전 처리에서는 추측해서 완료하거나 재요청하기보다 확실한 외부 사실이 확인됐을 때 상태를 바꾸는 방향을 선택했다.

## 관련 문서

- [TS-10 — PG–DB Dual-write](./ts-10-refund-dual-write-consistency.md)
- [TS-07 — READY Payment TOCTOU](./ts-07-ready-payment-toctou.md)
