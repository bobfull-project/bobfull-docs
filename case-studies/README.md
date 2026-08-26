# ⭐ Representative Case Studies

BobFull 프로젝트의 여러 실험·장애 대응·기술 판단을 하나의 흐름으로 정리한 최종 사례입니다.

`5분 기록 보드`에서 `[발표]`로 정리된 문서를 기준으로 하며, 개별 구현 근거는 ADR·Troubleshooting·Performance·Backend Evidence로 연결합니다.

## Case Studies

| 주제 | 사례 | 핵심 질문 |
|---|---|---|
| 고가용성 | [단일 EC2 SPOF에서 Multi-AZ Blue-Green까지](./spof-to-multi-az-blue-green.md) | App 한 대의 장애·배포가 전체 중단으로 이어지는 구조를 어떻게 바꿨는가 |
| 성능 | [조회 인덱스 부재에서 정산 조회 병목 해소까지](./query-index-to-settlement-optimization.md) | Query·Index·Cache·Pool·CPU 병목을 어떤 순서로 구분했는가 |
| 거래/후속 작업 | [핵심 거래와 후속 작업의 실패를 어떻게 나눴는가](./transaction-and-followup-failure-boundary.md) | 함께 실행되는 작업을 어디까지 같이 실패시켜야 하는가 |
| AI | [AI 검수 호출 최적화와 우회 검수 보완](./ai-moderation-optimization.md) | AI 호출을 줄이면서 Split/Prompt Injection 우회를 어떻게 다뤘는가 |
| Kafka | [Outbox + Async면 충분한데 Kafka까지 필요한가](./outbox-async-vs-kafka.md) | 성능이 아닌 어떤 요구 때문에 Kafka를 유지했는가 |
| 이벤트 설계 | [AFTER_COMMIT·Outbox·Kafka를 나눈 기준](./post-payment-processing-strategy.md) | AFTER_COMMIT, Outbox, Async, Kafka의 책임을 어떻게 구분했는가 |

## 읽는 기준

- 이 디렉터리는 발표용으로 최종 정제한 **대표 사례**입니다.
- 같은 내용의 세부 사건은 [Troubleshooting](../troubleshooting/README.md), 수치와 실험은 [Performance](../performance/README.md), 공식 결정은 [ADR](../adr/README.md)에서 확인합니다.
- 측정하지 않은 개선 효과나 현재 구성보다 넓은 HA 범위를 주장하지 않습니다.
