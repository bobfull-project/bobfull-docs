# [성능] Query·Cache·Hikari 병목과 확장 판단

BobFull에서는 느리다는 느낌만으로 서버나 캐시를 추가하지 않고, Query 수·p95/p99·CPU·HikariCP 상태를 함께 측정한 뒤 적용 여부를 결정했습니다.

## 1. 인기 회차 조회 — 반복 쿼리부터 줄이기

인기 회차 조회에서는 반복 조회로 Query 수가 커지는 문제가 있었습니다.

배치 조회 적용 후 대표 결과는 다음과 같습니다.

- Query `83 → 7`
- Load p95 `802.66ms → 60.27ms`
- Stress p95 `13.14s → 1.34s`

먼저 애플리케이션 내부의 반복 Query 구조를 줄였고, 최고 부하 구간에서는 CPU와 HikariCP 포화가 다시 나타날 수 있다는 한계도 함께 기록했습니다.

→ [인기 회차 조회 상세](../performance/restaurant-view-hotpath.md)

## 2. 정산 조회 — Batch보다 Query / Index 개선

정산 조회는 별도 Batch/Snapshot 구조를 추가하기 전에 조회 경로와 인덱스를 먼저 확인했습니다.

- 결합 p95 `6.5s → 30.32ms`
- Hikari Pending `92 → 0`

현재 요구사항에서는 더 복잡한 Batch 구조보다 Query/Index 개선만 유지하는 쪽을 선택했습니다.

→ [정산 조회 인덱스 상세](../performance/settlement-query-index.md)

## 3. 반복 검색 — 제한적으로 Redis Cache 적용

검색 전체를 무조건 캐싱하지 않고 반복되는 식당 검색처럼 정합성 영향이 상대적으로 낮고 반복도가 높은 범위에 Cache를 적용했습니다.

- 반복 검색 p95 `43ms → 14ms`
- DB Query `2 → 0`

Redis 장애 시에는 DB 조회로 대체할 수 있도록 Cache를 기준 저장소로 만들지 않았습니다.

→ [검색 Redis Cache 상세](../performance/search-cache.md)

## 4. Connection Pool을 늘리면 항상 좋아지는가

인기 회차 고부하 실험에서는 HikariCP가 포화되는 현상을 확인했습니다.

하지만 Pool Size를 크게 늘리는 것만으로 해결되지 않았습니다.

별도 실험에서는 Pool 10 대비 Pool 30에서 p95가 오히려 악화되는 경우도 있었고, CPU 자원을 크게 올렸을 때 처리 지연이 줄었습니다.

따라서 다음을 구분했습니다.

```text
Query 자체가 느림
→ Query / Index 개선

Hikari Pending 증가
→ Connection 사용 시간과 Pool 조건 확인

App CPU가 먼저 포화
→ App Scale-out 후보
```

## 5. Auto Scaling을 바로 넣지 않은 이유

Active App 2대 조건에서 별도로 측정한 Auto Scaling 판단 실험에서는 App CPU와 RDS CPU에 여유가 있었지만 Hikari 대기가 먼저 증가했습니다.

대표 결과:

| 지표 | Pool 10 | Pool 12 재현 |
|---|---:|---:|
| p95 | `35.4ms` | `22.42ms` |
| p99 | `358.79ms` | `94.55ms` |
| Hikari Pending | 약 `40~60` | 거의 `0` |

이 조건에서는 App CPU 포화보다 Connection Pool 대기가 먼저 관측됐기 때문에, ASG와 Scaling Policy를 추가하는 복잡도와 비용을 정당화하기 어렵다고 판단했습니다.

중요한 점은 인기 회차 고부하 실험과 Auto Scaling 판단 실험을 같은 실험으로 섞지 않는 것입니다. 서로 다른 조건에서 다른 병목을 확인했습니다.

## 6. 재검토 조건

Auto Scaling은 영구히 배제한 것이 아니라 다음 조건이 반복 측정될 때 다시 검토합니다.

- App CPU 또는 처리량이 먼저 포화
- DB Pool·RDS·Redis 등 하위 의존성은 여유
- p95/p99 또는 5xx가 증가
- App 인스턴스 추가로 병목이 실제 완화될 근거가 있음

## 관련 문서

- [Performance 요약](../performance/README.md)
- [ADR 0015 - 측정 후 App Auto Scaling 미도입](../adr/0015-no-app-auto-scaling.md)
- [Backend Final Claim Matrix](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/FINAL_CLAIM_MATRIX.md)
