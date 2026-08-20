# 정산 조회 Query / Index 개선

## 문제

정산 조회는 이미 배치·집계 쿼리로 구성돼 있어 N+1은 아니었지만, 데이터가 누적되자 조인 컬럼의 인덱스 부재 때문에 반복 조회에서 HikariCP Pool이 포화됐습니다.

## 개선

실행 계획과 데이터 규모별 지연을 확인한 뒤 다음 인덱스를 추가했습니다.

- `payment.reservation_id`
- `reservation.time_slot_id`
- `payment.time_slot_id`

별도 Snapshot이나 Spring Batch를 도입하지 않고 현재 Query-time 계산 구조를 유지했습니다.

## 결과

| 지표 | Before | After |
|---|---:|---:|
| 결합 p95 | 6.5s | **30.32ms** |
| 결합 p99 | 9.12s | **91.69ms** |
| dropped iterations | 1,365 | **0** |
| Hikari active 최대 | 10/10 | **1** |
| Hikari pending 최대 | 92 | **0** |
| checks succeeded | 100% | **100%** |

`getExpectedSettlement` 단독 p95도 `7.51s → 91.02ms`로 개선됐습니다.

## 판단

문제는 계산 방식을 Snapshot으로 바꿔야 해서가 아니라 **기존 조회 경로의 인덱스가 부족했던 것**이었습니다. 따라서 운영 복잡도가 더 큰 Batch/Snapshot은 도입하지 않고 Query/Index 개선안을 채택했습니다.

## Evidence

- [Backend Evidence — Issue #65](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/65-settlement/README.md)
