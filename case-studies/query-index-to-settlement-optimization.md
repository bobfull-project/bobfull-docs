# 조회 인덱스 부재에서 정산 조회 병목 해소까지

> 원본 분류: `[발표]` 트러블슈팅 · 작성자: 정용태

## 전체 흐름

```text
검색 API 풀스캔
→ 조인 키 인덱스 부재
→ 인덱스 추가
→ 반복 조회 시 Pool 포화
→ No Cache 기준선
→ 제한적 Redis Cache
→ 인기 회차 예약 CPU/Pool 병목
→ Pool 확장 후 오히려 악화
→ CPU가 실제 병목임을 확인
→ 정산 조회에서 유사 패턴 재현
→ 가설 배제와 원인 특정
→ 인덱스 추가로 병목 해소
→ TODO: CPU 증설·Batch/Snapshot 재검토
```

관련 Issue는 `#61`, `#62`, `#142`, `#65`다.

## 1. date/time 조건만 유독 느렸다

식당 검색 API에서 필터별 실행 계획을 확인했을 때 date/time 조건만 크게 느렸다.

측정 조건은 MySQL 8.4.10 Docker, Restaurant 5,000건, SharedTable 5,000건, TimeSlot 10,000건이며 로컬 단일 요청에서 `EXPLAIN ANALYZE`를 대표 1회 실행한 값이다.

| 조건 | 관측 응답시간 |
|---|---:|
| 필터 없음 | 1.21ms |
| keyword | 1.82ms |
| category | 0.075ms |
| date | **11.5ms** |
| time | **12.1ms** |

이 값은 반복 평균이 아니므로 안정적인 개선율이나 배수로 일반화하지 않는다.

코드만 보고 추측하지 않고 `EXPLAIN`을 확인하자 `shared_table`을 포함한 3-way join이 인덱스 없이 `type=ALL`로 테이블을 훑고 있었다. 조인 키 인덱스를 추가해 실행 계획을 `range/ref` 중심으로 바꿨다.

이후 팀의 공통 습관도 하나 생겼다.

> **인덱스가 있을 것이라고 추측하지 않고 실행 계획으로 확인한다.**

## 2. 반복 조회에서는 DB Connection Pool이 다음 병목이었다

단건 SQL을 줄인 뒤 같은 검색이 반복되면 Hikari Connection Pool이 포화되는 문제가 보였다. 캐시부터 넣지 않고 No Cache 기준선을 먼저 측정한 뒤, 정합성 영향이 적고 반복성이 높은 검색 범위에만 Redis Cache를 제한적으로 적용했다.

기존 Performance 문서의 재현 조건에서는 반복 검색 p95가 `43ms → 14ms`, DB query가 `2 → 0`으로 줄었다. 이 결과는 해당 캐시 시나리오의 실측이며 모든 검색 요청의 보편적인 개선율을 의미하지 않는다.

## 3. Pool을 늘리면 항상 빨라지는 것이 아니었다

인기 회차 예약 부하에서는 Hikari Pool이 10/10에 도달해 처음에는 Pool 확장을 해결책으로 생각했다. 그러나 Pool을 늘리자 DB로 동시에 더 많은 작업이 밀리면서 CPU 경합이 커지고 오히려 결과가 악화되는 구간이 나타났다.

이 과정에서 **대기열이 보인다는 이유만으로 Pool 크기를 원인으로 단정하지 않고 CPU·DB·Lock·Query를 함께 봐야 한다**는 기준을 세웠다.

## 4. 정산 조회에서 같은 방식으로 원인을 좁혔다

정산 조회 p95가 크게 증가했을 때 Batch/Snapshot 같은 큰 구조 변경부터 적용하지 않고 다음 가설을 차례로 배제했다.

- 애플리케이션 CPU가 먼저 포화됐는가
- Hikari Pool 자체가 너무 작은가
- 집계 계산량이 핵심인가
- SQL 실행 계획과 인덱스가 문제인가

최종적으로 쿼리/인덱스 쪽이 핵심 병목임을 확인했고, 기존 Performance 기록 기준 결합 p95가 **6.5s → 30.32ms**, Hikari pending이 **92 → 0**으로 내려갔다.

따라서 현재 데이터 규모에서는 Batch/Snapshot을 추가하지 않고 **Query/Index 개선을 유지**했다.

## 배운 점

성능 개선은 기술 목록을 추가하는 일이 아니라 병목을 하나씩 분리하는 과정이었다.

```text
SQL이 느리다 → EXPLAIN
반복 요청이 많다 → Pool/Cache
Pool이 찬다 → CPU·DB·Lock을 함께 확인
큰 구조 변경이 필요해 보인다 → Query/Index부터 배제
```

## 관련 문서

- [정산 조회 인덱스 개선](../performance/settlement-query-index.md)
- [검색 Redis Cache](../performance/search-cache.md)
- [인기 회차 조회 Hot-path](../performance/restaurant-view-hotpath.md)
- [Query·Cache·Hikari 병목과 확장 판단](../engineering-records/performance-and-scaling.md)
