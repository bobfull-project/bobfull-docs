# 🧭 Architecture Decision Records

BobFull의 공식 번호형 ADR은 Backend 저장소에서 관리합니다.

- **Source of Truth:** [Backend ADR 0001~0019](https://github.com/bobfull-project/bobfull-backend/tree/develop/docs/adr)
- **이 저장소의 역할:** 면접관·리뷰어가 핵심 의사결정을 빠르게 읽을 수 있도록 대표 ADR을 요약하고, 5분 기록 보드의 Project ADR도 별도 표기해 보존합니다.

## 대표 Backend ADR

| ADR | 핵심 의사결정 | 포트폴리오 관점 |
|---|---|---|
| [0001](./0001-reservation-seat-consistency.md) | 예약 좌석 정합성 · READY Payment 임시 선점 | 동시성·Lock·결제 경계 |
| [0002](./0002-payment-completion-idempotency.md) | 결제 완료 API · PortOne Webhook 멱등성 | 외부 결제와 내부 상태 정합성 |
| [0005](./0005-domain-boundary-dependency-policy.md) | 도메인 간 의존 경계 | Port/Adapter와 조회 구조 기준 |
| [0007](./0007-s3-presigned-restaurant-image-validation.md) | Presigned URL + Lambda 이미지 검증 | App 부하와 파일 검증 책임 분리 |
| [0008](./0008-chat-room-transactional-outbox.md) | ChatRoom Transactional Outbox | 커밋 이후 작업의 유실·중복 대응 |
| [0010](./0010-chat-message-outbox-kafka-pipeline.md) | AI 후속 처리 Outbox + Kafka | 적체·Retry/DLT·Consumer 확장 경계 |
| [0011](./0011-chat-redis-pubsub.md) | 다중 App 채팅 Redis Pub/Sub | 실시간 fan-out과 공유 상태 |
| [0013](./0013-blue-green-deployment.md) | ALB Blue-Green 배포 | 사전 검증·Traffic Switch·Rollback |
| [0015](./0015-no-app-auto-scaling.md) | 측정 후 Auto Scaling 미도입 | 기술을 추가하지 않은 결정도 실측으로 설명 |
| [0016](./0016-blue-green-db-schema-compatibility.md) | Blue-Green DB Schema 호환 | App Rollback과 공용 RDS 호환성 |
| [0018](./0018-kafka-dedicated-ec2-over-msk.md) | MSK 대신 Kafka 전용 EC2 | 비용·운영 책임·현재 규모 비교 |

## Project ADR

5분 기록 보드에서 ADR로 관리했지만 현재 Backend의 번호형 0001~0019 시리즈에는 별도 번호가 없는 기록입니다. 공식 번호를 임의로 만들지 않고 Project ADR로 구분합니다.

| 문서 | 결정 |
|---|---|
| [검색 API 동적 조건 처리를 QueryDSL로 구현](./project-querydsl-dynamic-search.md) | Optional 검색 조건·조인·정렬을 QueryDSL로 구성 |

`S3 Presigned URL + Lambda` 기록은 Backend 공식 ADR 0007과 같은 결정이므로 신규 번호를 만들지 않고 0007 포트폴리오 요약본으로 연결했습니다.

## 읽는 순서

1. **핵심 정합성:** 0001 → 0002
2. **도메인/조회 설계:** 0005 → Project QueryDSL
3. **파일 처리:** 0007
4. **후속 처리:** 0008 → 0010
5. **다중 App:** 0011
6. **배포·운영:** 0013 → 0015 → 0016 → 0018

## 관리 원칙

- 세부 구현·최신 수치·Evidence가 충돌하면 Backend의 최신 `develop` ADR/Evidence를 우선합니다.
- ADR은 “문제가 있었다”보다 **선택지·결정·Trade-off·재검토 조건**을 중심으로 기록합니다.
- 구체적인 장애·버그는 [Troubleshooting](../troubleshooting/README.md), 최종 대표 흐름은 [Case Studies](../case-studies/README.md)로 연결합니다.
