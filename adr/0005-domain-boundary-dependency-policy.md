# ADR 0005: 도메인 간 의존 경계와 조회 조합 원칙

> BobFull Backend의 전체 ADR 19개 중 포트폴리오에서 보여줄 대표 의사결정을 요약한 문서입니다. 상세 원본은 [Backend ADR](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/adr/0005-domain-boundary-dependency-policy.md)을 기준으로 합니다.

## 문제

예약·회차·테이블 등 여러 도메인이 서로의 Repository, Entity, 상태 Enum을 직접 참조하면 저장 구조와 정책 변경이 다른 도메인으로 전파됩니다. 반대로 모든 조회까지 Port/Adapter로 분리하면 호출과 코드가 과도하게 늘어날 수 있습니다.

## 결정

쓰기·상태 변경·정책 검증에는 최소한의 Port/Adapter 경계를 사용하고, 여러 도메인을 조합하는 읽기 기능은 전용 QueryRepository·Projection·Read Model을 허용합니다.

- 소비 도메인이 필요한 최소 Port와 불변 DTO/record를 정의
- Adapter가 외부 도메인의 Repository와 내부 연결 구조를 캡슐화
- 복합 조회는 서비스 체인을 늘리지 않고 조회 전용 구조 사용 가능
- 같은 Aggregate나 명확한 부모-하위 관계의 단순 조회까지 Port를 강제하지 않음

## 왜 이 선택을 했나

도메인의 저장 구조·상태 정책 소유권을 분리하면서도 조회 성능과 구현 복잡도를 함께 관리하기 위해서입니다.

## 트레이드오프와 검증

Port/Adapter를 과도하게 적용하면 오히려 코드가 읽기 어려워질 수 있으므로 쓰기/정책과 복합 조회의 기준을 구분합니다. 대상 Application Service가 외부 도메인 Repository·Entity·상태 Enum에 직접 의존하지 않는지와 기존 락·트랜잭션·예외 계약이 유지되는지를 회귀 테스트합니다.
