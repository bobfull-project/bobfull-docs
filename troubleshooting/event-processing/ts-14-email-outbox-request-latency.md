# TS-14 — 이메일 Outbox 동기 처리로 인한 요청 지연

> 작성자: 김현승

## 문제 발견

AWS k6 환불 완료 성능 테스트를 준비하며 `POST /api/payments/{id}/complete`를 반복 호출하는 과정에서 Fixture 준비가 비정상적으로 오래 걸렸고 `/actuator/health`가 DOWN되는 상황까지 관측됐다.

당시 CPU·DB Connection·HikariCP는 여유가 있었지만 이메일 SMTP가 결제 완료 요청 스레드에서 직접 실행되고 있었다.

| 관측 항목 | 결과 |
|---|---:|
| SMTP 평균 | 약 0.5~1초 |
| SMTP p99 | 약 1.5초 |
| Fixture 준비 | 시나리오당 약 7~15분 |
| Application | Health DOWN 관측 |

## 원인

Outbox는 이미 사용하고 있었지만 즉시 처리하는 `EmailOutboxProcessor.signal()`이 별도 실행 경계 없이 호출됐다.

```text
HTTP 요청
→ Email Outbox 저장
→ COMMIT
→ Processor.signal()
→ SMTP
→ HTTP 요청 종료
```

즉 Outbox는 작업을 DB에 남겼지만 **SMTP 실행 스레드는 분리하지 않았다.** SMTP Timeout 설정도 없어 외부 메일 서버가 느릴 때 요청 Thread를 오래 점유할 위험이 있었다.

## 해결

이메일 전용 bounded Executor를 추가했다.

```text
HTTP 요청
→ Outbox 저장
→ COMMIT
→ Email Executor에 제출
→ HTTP 반환

[Email Thread]
→ EmailOutboxProcessor
→ SMTP
```

초기 설정:

- Worker 2
- Queue 100
- `AbortPolicy`
- SMTP connection/read/write timeout 각각 5초

Worker 2는 당시 AWS에서 측정한 SMTP `0.5~1초`, p99 `1.5초`를 기준으로 잡은 보수적인 초기값이다.

Executor 제출이 거절되면 이메일을 버리지 않고 Outbox `PENDING`을 유지해 기존 Scheduler가 재처리한다.

## 검증과 한계

느린 Processor를 인위적으로 대기시키는 테스트에서 실제 이메일 처리가 끝나지 않은 상태에서도 Dispatcher가 **500ms 미만에 반환**하는 것을 검증했다. Outbox PENDING 복구와 핵심 결제·예약 Transaction의 실패 격리 계약도 유지했다.

다만 **수정 후 실제 AWS + SMTP 환경에서 k6를 재측정하지 않았다.** 따라서 다음과 같은 측정하지 않은 주장은 하지 않는다.

- `7~15분 → 몇 분`
- `Health DOWN → 0회`

## 배운 점

```text
Outbox
→ 데이터 영속화와 재처리 책임

Executor
→ 실행 스레드 격리

Timeout
→ 외부 장애의 장시간 자원 점유 제한
```

하나의 패턴을 적용했다고 다른 실행 경계까지 자동으로 해결되는 것은 아니다.

## 관련 문서

- [CS-03 — 핵심 거래/후속 작업 Case Study](../../case-studies/cs-03-transaction-and-followup-failure-boundary.md)
- [TS-13 — AFTER_COMMIT → Outbox](./ts-13-after-commit-to-transactional-outbox.md)
