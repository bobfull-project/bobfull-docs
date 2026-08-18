# ADR 0013: ALB 기반 Blue-Green 배포 전략

> BobFull Backend의 전체 ADR 19개 중 포트폴리오에서 보여줄 대표 의사결정을 요약한 문서입니다. 상세 원본은 [Backend ADR](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/adr/0013-blue-green-deployment.md)을 기준으로 합니다.

## 문제

단일 EC2에서 기존 컨테이너를 내린 뒤 새 컨테이너를 올리는 방식은 CI/CD 시간을 줄여도 실제 요청을 처리하지 못하는 구간이 남았습니다. 신규 버전을 운영 트래픽에 넣기 전에 검증하고, 실패하면 기존 버전으로 빠르게 되돌릴 수 있는 배포 경계가 필요했습니다.

## 결정

ALB Target Group weight를 이용한 Blue-Green 배포를 채택했습니다.

```text
Active 100 / Inactive 0
→ Inactive EC2 START 및 신규 이미지 배포
→ Readiness + Target Group Healthy
→ ALB Traffic Switch
→ Public readiness/API 검증
→ Prometheus 신규 Active 2대 UP 확인
→ 600초 Rollback Window
→ Listener weight 재확인
→ 기존 Active STOP
```

평상시에는 Active App EC2 2대만 유지하고 Inactive 2대는 STOP합니다.

## 검증 결과

- 정상 Blue-Green 배포 중 public readiness `2,787 / 2,787` HTTP 200
- 의도적 Public API 검증 실패 후 기존 Listener 상태로 자동 Rollback
- Rollback 전체 과정 중 public readiness `2,758 / 2,758` HTTP 200
- App EC2 1대 중지 시 외부 API `10 / 10` HTTP 200

단, App 1대 장애 우회는 Blue-Green 자체가 아니라 **ALB + Active App 2대**의 Runtime HA 결과입니다. 또한 단일 EC2 Before와 Blue-Green After의 측정 위치가 완전히 같지 않아 `40.25초 → 0초`를 동일 조건 개선율로 표현하지 않습니다.

## 트레이드오프

배포 시 최대 4대의 App EC2가 동시에 존재해 비용과 DB Connection Budget을 고려해야 합니다. 또한 Blue/Green이 같은 RDS를 공유하기 때문에 DB Schema 호환 전략이 별도로 필요합니다.
