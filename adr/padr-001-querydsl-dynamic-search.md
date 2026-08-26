# P-ADR-001 — 검색 API 동적 조건 처리를 QueryDSL로 구현

> 원본 분류: ADR · 작성자: 김홍기 · 관련 버전: V1 · Issue #35 / PR #106  
> 이 문서는 5분 기록 보드의 Project ADR을 보존한 문서이며, 현재 Backend의 번호형 ADR-0001~ADR-0019와는 별도 기록이다.

## 배경

홈 식당 검색과 모집중 예약 검색은 사용자가 보낸 조건만 선택적으로 적용해야 한다. keyword만 보낼 수도 있고 date/time·category·capacity·잔여석을 조합할 수도 있어 정적 메서드 쿼리만으로 관리하기 어려웠다.

## 요구사항

- 전달된 조건만 쿼리에 반영
- 조건이 없으면 기본 목록 조회
- 날짜·시간·카테고리·키워드·정원·잔여석 조합
- 페이지네이션·정렬
- 모집중 예약은 `RECRUITING` 또는 `CONFIRMED`이면서 `OPEN`
- `currentParticipantCount`, `availableCapacity`, `confirmationThreshold` 등은 조회 시 계산

## 대안

| 방법 | 장점 | 단점 |
|---|---|---|
| Spring Data JPA 메서드 쿼리 | 단순 | 조건 조합이 늘수록 메서드 폭증 |
| `@Query` JPQL | 직접 쿼리 표현 가능 | 동적 조건·정렬·계산이 섞이면 문자열 복잡도 증가 |
| **QueryDSL** | 타입 안정성, 동적 조합, 정렬 표현 용이 | 설정·Repository 구현 코드 증가 |

## 결정

QueryDSL을 사용한다.

- `BooleanBuilder`로 전달된 조건만 누적
- `BooleanExpression`으로 날짜/시간 조건 분리
- `OrderSpecifier`로 정렬 명시
- 조인과 계산값이 필요한 예약 검색을 하나의 Query Repository에서 관리

### 적용 API

`GET /api/restaurants`
- keyword, category, date, time
- 최근순 정렬

`GET /api/reservations/search`
- keyword, date, time, capacity, minimumRemainingSeats
- `RECRUITING | CONFIRMED` + `recruitmentStatus=OPEN`
- 참여 인원·잔여 정원·확정 기준 조회 시 계산

## Trade-off와 후속 과제

QueryDSL 의존성과 annotation processor가 추가되고 단순 Repository보다 구현 코드가 길다. 날짜/시간 조건과 계산식 일부가 중복돼 공통화 후보가 생겼다.

V1에서는 정확한 응답을 우선해 일부 계산을 서브쿼리 기반으로 처리했으므로 데이터가 커질 때는 인덱스·집계 방식·Cache·별도 조회 모델을 실측 후 검토한다.

## 관련 문서

- [CS-02 — 검색/정산 성능 Case Study](../case-studies/cs-02-query-index-to-settlement-optimization.md)
- [PF-03 — 검색 Redis Cache](../performance/search-cache.md)
