# [배포] Blue-Green 무중단 배포와 롤백

## 1. 문제

초기 배포는 단일 App EC2에서 기존 컨테이너를 내리고 새 이미지를 실행하는 방식이었습니다.

CI/CD 자체가 자동화돼 있어도 컨테이너 교체 동안 실제 요청을 처리하지 못하는 구간이 남았습니다. 또한 새 버전에 문제가 있어도 이미 기존 컨테이너를 내린 뒤라 빠른 복구가 어려웠습니다.

필요한 것은 단순한 배포 자동화가 아니라 다음 두 가지였습니다.

- 새 버전을 운영 트래픽에 넣기 전에 검증
- 검증 실패 시 기존 정상 환경으로 즉시 복귀

## 2. 선택한 구조

ALB 뒤에 Blue / Green Target Group을 두고, 각 환경을 App EC2 2대로 구성했습니다.

```text
평시
ALB
└─ Active 환경 App EC2 x2

배포 시
Active 환경 x2  ← 기존 운영
Inactive 환경 x2 ← 신규 버전 배포·검증
```

비활성 환경에 새 이미지를 먼저 배포한 뒤 Readiness와 Target Group Health Check를 확인합니다.

## 3. 배포 흐름

```text
GitHub Actions
→ ECR
→ SSM
→ 비활성 App EC2 2대 기동·배포
→ Readiness / Target Group Health Check
→ ALB 트래픽 전환
→ 외부 요청 검증
  ├─ 성공 → 모니터링 대상 갱신 → Prometheus 신규 Active 2대 UP 확인
  └─ 실패 → 기존 Listener 상태로 롤백
```

평시에는 비용과 DB Connection 점유를 줄이기 위해 Inactive 환경을 중지하고, 배포가 필요할 때만 기동합니다.

## 4. 실제 검증

Blue-Green 전환 중 외부 요청을 연속으로 보낸 결과:

- 외부 요청 검증 `2,787 / 2,787` HTTP 200
- 실패 `0`
- 관측 다운타임 `0초`

의도적으로 신규 환경 검증을 실패시켜 롤백한 테스트에서도:

- 전체 과정 `2,758 / 2,758` HTTP 200
- 기존 Listener 상태로 복귀

을 확인했습니다.

App EC2 1대를 중지했을 때 외부 API `10 / 10` HTTP 200도 확인했지만, 이 결과는 Blue-Green 자체가 아니라 **ALB + Active App EC2 2대 구성의 런타임 장애 우회 결과**로 구분합니다.

## 5. 수치를 과장하지 않은 이유

단일 EC2 배포에서 실제 요청 불가 구간을 관측했고, Blue-Green에서는 관측 다운타임 0초를 확인했습니다.

다만 두 측정의 위치와 조건이 완전히 동일하지 않기 때문에 이를 단순한 `기존 다운타임 → 0초` 개선율로 계산하지 않습니다.

이번 검증에서 주장하는 범위는 다음입니다.

> Blue-Green 전환과 롤백 동안 외부 검증 요청에서 실패가 관측되지 않았다.

## 6. 트레이드오프

Blue-Green은 배포 안정성을 높였지만 비용이 사라지는 구조는 아닙니다.

- 배포 중 App EC2 최대 4대 동시 기동
- Blue/Green이 동일 RDS를 사용하므로 DB Connection Budget 고려 필요
- 신·구 버전이 잠시 함께 존재하므로 DB Schema 하위 호환 필요
- 애플리케이션 배포가 무중단이어도 RDS·Kafka·Redis 장애까지 해결하는 것은 아님

따라서 Inactive 환경은 평시 중지하고, 배포 성공 후 일정 롤백 구간 동안만 이전 환경을 유지하는 방향으로 운영했습니다.

## 관련 문서

- [ADR 0013 - ALB 기반 Blue-Green 배포](../adr/0013-blue-green-deployment.md)
- [ADR 0016 - Blue-Green DB Schema 호환](../adr/0016-blue-green-db-schema-compatibility.md)
- [System Architecture](../architecture/system-architecture.md)
- [Backend Final Claim Matrix](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/FINAL_CLAIM_MATRIX.md)
