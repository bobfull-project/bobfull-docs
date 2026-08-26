# 🔧 Troubleshooting

BobFull에서 발견한 개별 문제를 **문제 정의 → 원인/가설 → 해결 → 검증 → 한계** 흐름으로 정리합니다.

`[발표]`로 최종 정제한 종합 사례는 [Representative Case Studies](../case-studies/README.md)에서 먼저 볼 수 있고, 이 디렉터리는 그 사례를 구성하는 상세 사건과 버그를 보존합니다.

## Infrastructure

| 문서 | 관점 |
|---|---|
| [CI/CD Pipeline 및 Docker Layer 구조 개선](./infrastructure/cicd-docker-layer-optimization.md) | 배포 성능·Image Layer |
| [ALB Health Check로 발생한 p95 알람 오탐](./infrastructure/alb-healthcheck-p95-false-alarm.md) | 관측성·Metric 설계 |

## Auth

| 문서 | 관점 |
|---|---|
| [Access Token jti 하위 호환 버그](./auth/access-token-jti-compatibility.md) | 인증 + 배포 마이그레이션 |

## AI

| 문서 | 관점 |
|---|---|
| [Prompt Injection·Split Message 우회 대응](./ai/prompt-injection-message-splitting.md) | AI Moderation·보안·Kafka Context |

## Reservation

| 문서 | 관점 |
|---|---|
| [비관적 락 이후 Reservation이 CANCELLING에 멈춘 문제](./reservation/pessimistic-lock-cancelling-state.md) | MySQL Repeatable Read·동시성 |

## Payment

| 문서 | 관점 |
|---|---|
| [PortOne SDK 동기 대기와 외부 API 예외 변환](./payment/portone-sdk-blocking.md) | 외부 API·Timeout·예외 계약 |
| [READY Payment TOCTOU](./payment/ready-payment-toctou.md) | 외부 I/O와 짧은 Lock |
| [결제 금액 Long → BigDecimal](./payment/payment-bigdecimal.md) | 금액 모델링·외부 검증 |
| [예약과 결제의 책임 분리](./payment/payment-reservation-responsibility.md) | 도메인 책임 경계 |
| [외부 PG–DB Dual-write 환불 정합성](./payment/refund-dual-write-consistency.md) | 부분 성공·상태 수렴 |
| [환불 Timeout Reconciliation](./payment/refund-timeout-reconciliation.md) | 결과 불명확·Webhook·재조회 |

## Event Processing

| 문서 | 관점 |
|---|---|
| [AFTER_COMMIT → Transactional Outbox](./event-processing/after-commit-to-transactional-outbox.md) | Durable Intent·At-least-once |
| [ChatRoom 실패가 핵심 거래까지 Rollback](./event-processing/chatroom-rollback-boundary.md) | Transaction 실패 경계 |
| [Email Outbox 동기 처리 지연](./event-processing/email-outbox-request-latency.md) | Outbox vs Async·SMTP |
| [핵심 거래와 후속 작업 실패 경계](./event-processing/core-transaction-followup-boundary.md) | Outbox/Async/Kafka 종합 상세 기록 |

## 여러 관점으로 찾기

한 문서는 한 위치에만 두고 중복 복사하지 않습니다. 대신 인덱스에서 여러 관점으로 연결합니다.

- **배포 하위 호환:** Access Token jti
- **외부 시스템 정합성:** READY Payment TOCTOU, PG–DB Dual-write, Refund Reconciliation
- **동시성:** READY Payment TOCTOU, Reservation CANCELLING
- **이벤트 신뢰성:** ChatRoom AFTER_COMMIT, Transactional Outbox, Email Executor
- **관측성:** ALB p95 false alarm, Email Health DOWN 발견 과정

## Source of Truth

구현 코드·공식 ADR·Raw Evidence가 충돌하면 [bobfull-backend](https://github.com/bobfull-project/bobfull-backend)의 최신 `develop` 문서를 우선합니다.
