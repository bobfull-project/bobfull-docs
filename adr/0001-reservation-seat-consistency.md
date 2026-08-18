# ADR 0001: 예약 좌석 정합성과 임시 선점 전략

> BobFull Backend의 전체 ADR 19개 중 포트폴리오에서 보여줄 대표 의사결정을 요약한 문서입니다. 상세 원본은 [Backend ADR](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/adr/0001-reservation-seat-consistency.md)을 기준으로 합니다.

## 문제

최초 예약과 추가 참여는 결제 완료 전에도 남은 좌석을 고려해야 합니다. 결제 대기 요청을 좌석 계산에서 제외하면 같은 회차에 동시 요청이 몰릴 때 정원을 초과할 수 있습니다.

## 결정

별도 `SeatHold` 엔티티를 추가하지 않고, 만료되지 않은 `PaymentStatus.READY`를 10분 임시 선점으로 사용합니다.

- 좌석 계산: `테이블 정원 - PAID 유효 참여 인원 - 만료되지 않은 READY 선점`
- 최초 예약: `TimeSlot` 행 비관적 락으로 직렬화
- 복수 락 획득 순서: `Payment → Reservation → TimeSlot`
- JOIN 준비 흐름은 MySQL `REPEATABLE_READ`에서 오래된 스냅샷을 피하기 위해 첫 조회부터 Reservation 잠금 조회 사용
- 예약 취소는 Reservation 락을 잡은 트랜잭션 안에서 외부 환불을 실행하지 않고, 취소 접수 → 외부 환불 → 완료 확정으로 분리

## 왜 이 선택을 했나

현재 Payment 상태를 이용해 별도 선점 저장소 없이 임시 선점을 표현하면서, 결제 성공 전 예약 데이터 생성을 피하고 동시 최초 예약의 초과 생성을 막을 수 있기 때문입니다.

## 트레이드오프와 검증

비관적 락은 인기 회차에 요청이 몰리면 대기 시간이 늘어날 수 있습니다. 실제 MySQL 동시성 테스트로 같은 TimeSlot에 대한 경쟁 요청에서 하나만 성공하고 나머지는 정상 대기·실패하는지 검증했습니다.

재검토 조건은 READY 선점 관리가 복잡해지거나, TimeSlot 락 병목이 실제로 확인되거나, 별도 선점 저장소가 필요한 수준으로 확장될 때입니다.
