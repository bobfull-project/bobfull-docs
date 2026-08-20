# 검색 Redis Cache 적용

## 문제

식당 검색 SQL 자체는 이미 개선됐지만, 동일 검색이 반복되면 매 요청마다 content/count 쿼리가 다시 실행됐습니다. 동시 요청에서는 HikariCP 10개가 모두 사용되고 최대 20개 요청이 Connection을 기다렸습니다.

## 개선

정합성 영향이 낮은 **date/time 없는 식당 검색**에만 Redis Cache를 제한적으로 적용했습니다.

- TTL 60초
- Restaurant 변경 시 version namespace 기반 무효화
- Redis 장애 시 DB 조회로 우회하는 Fail-open
- `availableCapacity`처럼 예약·결제 정합성에 직접 영향을 주는 값은 캐시하지 않음

## 결과

동일 조건 동시 반복 요청 기준입니다.

| 지표 | No Cache | Warm Cache Hit |
|---|---:|---:|
| 요청당 DB Query | 2 | **0** |
| Hikari active 최대 | 10/10 | **0** |
| Hikari awaiting 최대 | 20 | **0** |
| p95 | 43ms | **14ms** |

## 판단

캐시를 전 조회 경로에 확대하지 않고 **반복 가능성이 높고 정합성 위험이 낮은 검색 범위에만 적용**했습니다. Redis가 unavailable이어도 검색 API 자체는 동작하도록 했습니다.

실제 운영 트래픽의 Cache hit ratio와 Stampede 효과까지 검증한 결과는 아니므로, 해당 범위는 운영 관측이 필요합니다.

## Evidence

- [Backend Evidence — Issue #62](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/62-search-cache/README.md)
- 선행 SQL 개선: [Issue #61](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/61-search-query/README.md)
