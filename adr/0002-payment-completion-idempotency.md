# ADR 0002: 결제 완료 API와 PortOne 웹훅의 멱등성 경계

> BobFull Backend의 전체 ADR 19개 중 포트폴리오에서 보여줄 대표 의사결정을 요약한 문서입니다. 상세 원본은 [Backend ADR](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/adr/0002-payment-completion-idempotency.md)을 기준으로 합니다.

## 문제

사용자의 결제 완료 검증 API와 PortOne 웹훅은 같은 Payment 결과를 동시에 또는 반복해서 반영할 수 있습니다. 두 경로가 독립적으로 상태를 바꾸면 예약·참여·결제 상태가 중복 반영될 수 있습니다.

## 결정

두 진입점은 외부 결제 상태를 검증한 뒤 동일한 내부 Payment 결과 반영 경계로 수렴시킵니다.

- 완료 API: 인증 사용자와 `Payment.memberId` 소유권 검증
- 웹훅: JWT 대신 PortOne 웹훅 서명 검증
- 공통 경계: 외부 상태·금액·통화 재조회 → 내부 Payment 비관적 락 → 상태/만료 재검증
- 이미 `PAID`인 동일 결제 완료 요청은 `200 OK` 멱등 성공으로 기존 결과 반환
- `PAID` 전환과 Reservation/Participant 생성은 하나의 트랜잭션에서 처리

## 왜 이 선택을 했나

외부 결제 결과가 들어오는 두 경로를 하나의 상태 전이 기준으로 수렴시키면 중복 요청 자체를 오류로 취급하지 않으면서도 실제 상태 변경은 한 번만 수행할 수 있습니다.

## 트레이드오프와 검증

내부 Payment가 이미 만료됐는데 외부 PortOne 상태만 PAID인 경우에는 자동 보상까지 수행하지 않고 운영 확인이 필요한 구조화 로그를 남깁니다.

완료 API와 웹훅이 동시에 또는 반복 도착했을 때 예약·참여·결제 결과가 한 번만 반영되는지를 테스트 기준으로 둡니다.
