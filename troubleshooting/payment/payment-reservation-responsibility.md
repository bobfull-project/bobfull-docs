# 예약과 결제의 책임을 분리한 이유

> 관련 Issue #91 / PR #98 · 작성자: 김현승

## 배경

이전 프로젝트에서는 OrderService가 Payment Entity 생성과 저장까지 직접 담당했다. 구현은 빠르지만 주문 서비스가 결제 ID·상태·저장 규칙까지 알아 결제 기능이 커질수록 책임이 함께 커졌다.

BobFull에서는 역할을 나눴다.

```text
ReservationService
→ 예약 가능 여부 검증
→ 회원/회차/인원/금액 계산
→ CreateReadyPaymentCommand

PaymentService
→ paymentId 생성
→ READY 상태
→ KRW
→ Clock 기준 10분 만료
→ Payment 생성/저장
→ 결과 반환
```

## 결정 이유

예약은 **예약 가능성과 필요한 입력값**을 알고, 결제는 **결제 생성 규칙과 상태·저장 정책**을 안다.

Payment 도메인은 이후 다음 책임으로 확장된다.

- PortOne 연동
- 결제 완료 검증
- Webhook
- 멱등성
- 임시 선점 만료
- 취소/환불
- 정산/리포트

이 규칙을 ReservationService가 모두 알면 서로 다른 이유로 변경되는 책임이 한 서비스에 모인다.

## Trade-off

Command·Result·Interface가 추가돼 처음에는 코드가 더 많아 보인다. 대신 결제 정책 변경의 영향이 Reservation으로 덜 전파되고 테스트에서 Payment 경계를 대체하기 쉬워진다.

## 배운 점

관심사 분리는 클래스를 많이 나누는 것이 아니라 **서로 다른 이유로 변경되는 책임을 분리하는 것**이다. 단순 기능이라면 직접 생성도 가능하지만 외부 연동과 상태 전이가 많은 결제 도메인에서는 별도 책임 경계가 유리했다.

## 관련 문서

- [결제↔예약 순환 의존 의사결정](../../decisions/payment-reservation-dependency-boundary.md)
