# Architecture Decision Records

BobFull의 전체 기술 의사결정 원본은 Backend 저장소에서 관리합니다.

- **Source of Truth:** [Backend ADR 0001~0019](https://github.com/bobfull-project/bobfull-backend/tree/develop/docs/adr)
- **이 저장소의 역할:** 면접관·리뷰어가 핵심 의사결정을 빠르게 볼 수 있도록 대표 ADR만 포트폴리오 형태로 요약합니다.

## 대표 ADR 10선

| ADR | 핵심 의사결정 | 왜 대표로 선정했나 |
|---|---|---|
| [0001](./0001-reservation-seat-consistency.md) | 예약 좌석 정합성 · READY Payment 임시 선점 | 핵심 예약 동시성·락 설계 |
| [0002](./0002-payment-completion-idempotency.md) | 결제 완료 API · PortOne 웹훅 멱등성 | 외부 결제와 내부 상태 정합성 |
| [0005](./0005-domain-boundary-dependency-policy.md) | 도메인 간 의존 경계 | Port/Adapter와 조회 구조의 설계 기준 |
| [0008](./0008-chat-room-transactional-outbox.md) | ChatRoom Transactional Outbox | 커밋 이후 후속 작업의 유실·중복 대응 |
| [0010](./0010-chat-message-outbox-kafka-pipeline.md) | AI 후속 처리 Outbox + Kafka | Kafka를 속도가 아닌 운영 경계로 선택한 근거 |
| [0011](./0011-chat-redis-pubsub.md) | 다중 EC2 채팅 Redis Pub/Sub | 실시간 fan-out과 DB 복구 경계 분리 |
| [0013](./0013-blue-green-deployment.md) | ALB Blue-Green 배포 | 사전 검증·Traffic Switch·Rollback 실측 |
| [0015](./0015-no-app-auto-scaling.md) | 측정 후 Auto Scaling 미도입 | 기술을 추가하지 않은 결정도 실측으로 설명 |
| [0016](./0016-blue-green-db-schema-compatibility.md) | Blue-Green DB Schema 호환 | App Rollback과 같은 RDS의 호환성 문제 해결 |
| [0018](./0018-kafka-dedicated-ec2-over-msk.md) | MSK 대신 Kafka 전용 EC2 | 비용·운영 책임·현재 규모를 비교한 인프라 판단 |

## 읽는 순서

처음 보는 경우에는 아래 순서를 권장합니다.

1. **핵심 정합성:** 0001 → 0002
2. **구조·후속 처리:** 0005 → 0008 → 0010
3. **다중 인스턴스:** 0011
4. **배포·운영 판단:** 0013 → 0015 → 0016 → 0018

## 관리 원칙

이곳의 ADR은 원문 전체 복사본이 아니라 **대표 의사결정의 포트폴리오 요약본**입니다.

세부 구현, 최신 수치, Evidence, 재검토 조건이 충돌할 경우 Backend 저장소의 최신 `develop` ADR과 Evidence를 우선합니다. 전체 19개 ADR을 모두 확인하려면 위 Source of Truth 링크를 이용합니다.
