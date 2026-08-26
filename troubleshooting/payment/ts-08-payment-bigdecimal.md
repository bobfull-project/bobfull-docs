# TS-08 — 결제 금액 타입을 Long에서 BigDecimal로 변경한 이유

> 작성자: 김현승

## 문제 상황

초기에는 KRW 정수 금액만 고려해 `Long amount`를 사용했다. 원화 정수만 다룬다면 Long도 정확한 타입이지만 PortOne 검증과 환불·정산을 확장하면서 금액의 책임이 커졌다.

필요해진 표현은 다음과 같다.

- 소수 단위를 쓰는 통화
- 수수료율·세율
- 부분 환불
- 반올림 정책
- 외부 결제 시스템과 일관된 십진수 비교

## 해결

금액 타입을 `BigDecimal`로 변경했다.

```java
private BigDecimal amount;
```

외부 금액과 비교할 때는 `equals()`가 아니라 수치적 동일성을 보는 `compareTo()`를 사용한다.

```java
new BigDecimal("1000.0").equals(new BigDecimal("1000.00"));
// false: scale도 비교

new BigDecimal("1000.0").compareTo(new BigDecimal("1000.00"));
// 0: 수치적으로 동일
```

결제 검증에서는 표현 scale보다 실제 승인 금액이 같은지가 중요하므로:

```java
payment.getAmount().compareTo(external.amount()) == 0
```

을 기준으로 비교한다.

## PortOne 검증 경계

프론트의 결제 성공 응답을 그대로 신뢰하지 않고 서버가 `paymentId`로 PortOne 결제를 다시 조회해 다음을 비교한다.

- paymentId
- 실제 PAID 상태
- 승인 금액
- 통화

현재는 KRW를 사용하지만 타입을 현재 UI 요구에만 묶지 않고 환불·정산으로 이어지는 도메인 연산까지 고려했다.

## 배운 점

`Long`이 틀린 타입이어서가 아니라 **결제 도메인이 앞으로 수행할 연산과 외부 계약을 더 명확하게 표현하기 위해 BigDecimal을 선택**했다. BigDecimal을 선택한 뒤에는 scale/rounding/comparison 정책도 함께 관리해야 한다.
