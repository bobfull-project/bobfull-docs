# ADR 0015: 측정 후 App Auto Scaling 미도입

> BobFull Backend의 전체 ADR 19개 중 포트폴리오에서 보여줄 대표 의사결정을 요약한 문서입니다. 상세 원본은 [Backend ADR](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/adr/0015-no-app-auto-scaling.md)을 기준으로 합니다.

## 문제

다중 EC2를 구성했다고 바로 Auto Scaling까지 도입하면 실제 병목이 App CPU가 아니라 DB Connection Pool·Lock·Query 같은 하위 계층일 때 서버 수만 늘려 병목을 악화시킬 수 있습니다.

## 결정

실제 부하에서 병목을 먼저 측정하고, 현재 프로젝트 범위에서는 App Auto Scaling을 도입하지 않기로 했습니다.

최종 운영 기준:

- Active App EC2 2대 유지
- Hikari `maximumPoolSize=12`
- Inactive Blue-Green EC2는 평상시 STOP
- App CPU·처리량 포화가 실제로 확인될 때 Auto Scaling 재검토

## 측정 근거

Pool 10 환경에서는 App CPU가 약 20~40%, RDS CPU가 약 20%대로 여유가 있었지만 Hikari Active가 10/10에 도달하고 Pending이 약 40~60까지 증가했습니다.

| 지표 | Pool 10 | Pool 12 1차 | Pool 12 재현 |
|---|---:|---:|---:|
| p95 | 35.4ms | 29.8ms | 22.42ms |
| p99 | 358.79ms | 111.85ms | 94.55ms |
| Dropped Iterations | 417 | 18 | 40 |
| Hikari Pending | 약 40~60 | 거의 0 | 거의 0 |

따라서 먼저 관측된 병목은 App CPU가 아니라 Connection Pool 대기였습니다. Pool 12에서 개선 경향이 두 차례 재현됐지만, 전체 개선을 Pool Size 변경 하나의 효과라고 단정하지 않습니다.

## 왜 미도입을 ADR로 남겼나

기술을 많이 넣는 것이 목표가 아니라, **실측한 병목에 맞는 기술만 도입한다는 판단**을 남기기 위해서입니다. 현재 근거만으로는 ASG·Scaling Policy의 복잡도와 비용을 정당화하기 부족했습니다.

## 재검토 조건

App CPU 또는 처리량이 먼저 포화되고, DB Pool·RDS·Redis 등 하위 의존성은 여유가 있는데 p95/p99나 5xx가 증가하는 상황이 반복 측정될 때 다시 검토합니다.
