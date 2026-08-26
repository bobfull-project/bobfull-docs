# 🔧 Troubleshooting

BobFull에서 발견한 개별 문제를 **문제 정의 → 원인/가설 → 해결 → 검증 → 한계** 흐름으로 정리합니다.

`CS`가 여러 기록을 묶은 대표 사례라면 `TS`는 하나의 구체적인 실패·버그·병목을 다루는 상세 기록입니다.

## TS-01 ~ TS-02 · Infrastructure

| ID | 문서 | 관점 |
|---|---|---|
| **TS-01** | [CI/CD Pipeline 및 Docker Layer 구조 개선](./infrastructure/ts-01-cicd-docker-layer-optimization.md) | 배포 성능·Image Layer |
| **TS-02** | [ALB Health Check로 발생한 p95 알람 오탐](./infrastructure/ts-02-alb-healthcheck-p95-false-alarm.md) | 관측성·Metric 설계 |

## TS-03 · Auth

| ID | 문서 | 관점 |
|---|---|---|
| **TS-03** | [Access Token jti 하위 호환 버그](./auth/ts-03-access-token-jti-compatibility.md) | 인증 + 배포 마이그레이션 |

## TS-04 · AI

| ID | 문서 | 관점 |
|---|---|---|
| **TS-04** | [Prompt Injection·Split Message 우회 대응](./ai/ts-04-prompt-injection-message-splitting.md) | AI Moderation·보안·Kafka Context |

## TS-05 · Reservation

| ID | 문서 | 관점 |
|---|---|---|
| **TS-05** | [비관적 락 이후 Reservation이 CANCELLING에 멈춘 문제](./reservation/ts-05-pessimistic-lock-cancelling-state.md) | MySQL Repeatable Read·동시성 |

## TS-06 ~ TS-11 · Payment

| ID | 문서 | 관점 |
|---|---|---|
| **TS-06** | [PortOne SDK 동기 대기와 외부 API 예외 변환](./payment/ts-06-portone-sdk-blocking.md) | 외부 API·Timeout·예외 계약 |
| **TS-07** | [READY Payment TOCTOU](./payment/ts-07-ready-payment-toctou.md) | 외부 I/O와 짧은 Lock |
| **TS-08** | [결제 금액 Long → BigDecimal](./payment/ts-08-payment-bigdecimal.md) | 금액 모델링·외부 검증 |
| **TS-09** | [예약과 결제의 책임 분리](./payment/ts-09-payment-reservation-responsibility.md) | 도메인 책임 경계 |
| **TS-10** | [외부 PG–DB Dual-write 환불 정합성](./payment/ts-10-refund-dual-write-consistency.md) | 부분 성공·상태 수렴 |
| **TS-11** | [환불 Timeout Reconciliation](./payment/ts-11-refund-timeout-reconciliation.md) | 결과 불명확·Webhook·재조회 |

## TS-12 ~ TS-15 · Event Processing

| ID | 문서 | 관점 |
|---|---|---|
| **TS-12** | [ChatRoom 실패가 핵심 거래까지 Rollback](./event-processing/ts-12-chatroom-rollback-boundary.md) | Transaction 실패 경계 |
| **TS-13** | [AFTER_COMMIT → Transactional Outbox](./event-processing/ts-13-after-commit-to-transactional-outbox.md) | Durable Intent·At-least-once |
| **TS-14** | [Email Outbox 동기 처리 지연](./event-processing/ts-14-email-outbox-request-latency.md) | Outbox vs Async·SMTP |
| **TS-15** | [핵심 거래와 후속 작업 실패 경계](./event-processing/ts-15-core-transaction-followup-boundary.md) | Outbox/Async/Kafka 종합 상세 기록 |

## 여러 관점으로 찾기

한 문서는 한 위치에만 두고 중복 복사하지 않습니다. 대신 ID를 기준으로 여러 관점에서 연결합니다.

- **배포 하위 호환:** TS-03
- **외부 시스템 정합성:** TS-07, TS-10, TS-11
- **동시성:** TS-05, TS-07
- **이벤트 신뢰성:** TS-12, TS-13, TS-14, TS-15
- **관측성:** TS-02, TS-14

## Source of Truth

구현 코드·공식 ADR·Raw Evidence가 충돌하면 [bobfull-backend](https://github.com/bobfull-project/bobfull-backend)의 최신 `develop` 문서를 우선합니다.
