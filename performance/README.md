# 📈 Performance

BobFull에서 **실제로 측정한** 주요 성능 개선과 기술 비교를 요약합니다.

원본 실험 로그를 복제하지 않습니다. 측정 조건·Raw 결과·재현 방법의 Source of Truth는 `bobfull-backend/docs/evidence/v3`이며, 여기서는 포트폴리오 관점에서 핵심 결과와 판단을 정리합니다.

## 핵심 결과

| ID | 주제 | 핵심 결과 | 판단 |
|---|---|---|---|
| **PF-01** | [인기 회차 조회 Hot-path](./restaurant-view-hotpath.md) | 쿼리 `83 → 7`, Load p95 `802.66ms → 60.27ms` | 반복 쿼리 배치 조회 적용 |
| **PF-02** | [정산 조회 인덱스 개선](./settlement-query-index.md) | 결합 p95 `6.5s → 30.32ms`, Hikari pending `92 → 0` | Batch/Snapshot 대신 Query/Index 개선 유지 |
| **PF-03** | [검색 Redis Cache](./search-cache.md) | 반복 검색 p95 `43ms → 14ms`, DB query `2 → 0` | 정합성 영향이 적은 검색 범위에 제한 적용 |
| **PF-04** | [Outbox + Async vs Kafka](./async-vs-kafka.md) | Async `5.394s / 5.56 msg/s`, Kafka `7.210s / 4.16 msg/s` | 성능 우위가 아닌 독립 Consumer·격리 경계로 Kafka 유지 |

> 기존 `performance/` 파일 경로는 이미 연결된 문서가 있을 수 있어 유지하고, 탐색용 ID만 `PF-xx`로 통일합니다.

## Case Study로 이어서 보기

- [CS-02 — 조회 인덱스 부재에서 정산 조회 병목 해소까지](../case-studies/cs-02-query-index-to-settlement-optimization.md)
- [CS-05 — Outbox + Async면 충분한데 Kafka까지 필요한가?](../case-studies/cs-05-outbox-async-vs-kafka.md)

## 읽는 기준

- 서로 다른 환경의 값을 직접 개선율로 연결하지 않습니다.
- 대표 1회 측정은 평균·안정적인 개선 배수처럼 표현하지 않습니다.
- 실제 측정 범위보다 큰 성능·HA·유실 방지 주장을 하지 않습니다.
- 더 단순한 구조가 충분하면 복잡한 기술을 먼저 추가하지 않습니다.
- 상세 조건과 한계는 Backend Evidence를 우선합니다.

## 원본 Evidence

- [V3 Evidence Standard](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/README.md)
- [Final Claim Matrix](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/FINAL_CLAIM_MATRIX.md)
