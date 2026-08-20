# 인기 회차 조회 Hot-path 개선

## 문제

인기 회차 조회 부하에서 CPU와 DB Connection Pool이 함께 포화됐습니다. 식당 상세와 회차 조회를 분리 측정한 결과, `dining-sessions` 조회가 주 병목이었고 회차마다 4종의 쿼리를 반복 실행하고 있었습니다.

## 개선

회차 ID 목록을 기준으로 활성 예약, 참여 인원, CLOSED 여부, READY Payment 선점 합계를 각각 한 번에 조회하도록 변경했습니다. 계산 정책은 유지하고 데이터를 가져오는 방식만 바꿨습니다.

## 결과

| 지표 | Before | After |
|---|---:|---:|
| TimeSlot 20건 SQL | 83 | **7** |
| Load p95 | 802.66ms | **60.27ms** |
| Load p99 | 1.706s | **265.54ms** |
| 평균 응답시간 | 299.22ms | **35.41ms** |
| CPU 최대 | 91.7% | **21.2%** |
| 오류율 | 0% | **0%** |

동일 Stress 조건에서도 p95는 `13.14s → 1.34s`, HTTP RPS는 `51.4 → 195.3 req/s`로 개선됐습니다.

## 판단

Pool 크기를 먼저 늘리기보다 **요청당 DB 비용을 줄이는 것이 우선**이라고 판단했습니다. 이번 측정 범위에서는 Hikari Pool 10을 유지했습니다.

최고 Stress 단계에서는 CPU와 Pool 포화가 다시 발생하므로, 모든 부하 수준에서 병목이 완전히 사라졌다고 주장하지 않습니다.

## Evidence

- [Backend Evidence — Issue #235](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/restaurant-view-hotpath/README.md)
- 선행 측정: [Issue #142 Reservation Peak](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/142-reservation-peak/README.md)
