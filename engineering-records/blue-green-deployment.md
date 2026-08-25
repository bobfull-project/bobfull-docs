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
                AZ 2a             AZ 2c

Blue           Blue #1           Blue #2
Green          Green #1          Green #2

                 총 App EC2 4대
```

Blue와 Green 모두 두 가용 영역에 걸쳐 구성하고, 평시에는 한 환경만 Active로 서비스합니다.

**Blue / Green EC2 구성과 Listener 상태**

![Blue Green App EC2 구성](https://velog.velcdn.com/images/gpekd5/post/27ee7dc5-7a6c-4db8-be79-0c5636bea8e1/image.png)

![ALB Listener Blue 100 Green 0](https://velog.velcdn.com/images/gpekd5/post/ed5e28ec-1084-498e-a69a-237748da29ac/image.png)

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

**GitHub Actions Blue-Green Workflow**

![GitHub Actions Blue Green Workflow](https://velog.velcdn.com/images/gpekd5/post/0509d14f-b94f-4fa0-b17a-7b97761a622b/image.png)

## 4. 실제 검증

Blue-Green 전환 전부터 완료까지 외부 엔드포인트에 요청을 연속으로 보낸 결과:

- 외부 요청 `2,787 / 2,787` HTTP 200
- 실패 `0`
- 성공률 `100%`
- 관측 다운타임 `0초`

![Blue Green 배포 중 HTTP 200 연속 유지](https://velog.velcdn.com/images/gpekd5/post/7143526a-9b5a-49ef-a2f3-976089ff10fc/image.png)

정상 배포만 확인하지 않고, 트래픽 전환 이후 외부 API 검증을 의도적으로 실패시켜 자동 롤백 경로도 확인했습니다.

- 롤백 전체 과정 `2,758 / 2,758` HTTP 200
- 외부 검증 실패 시 기존 Listener 상태로 복귀

![외부 검증 실패 후 자동 Rollback Workflow](https://velog.velcdn.com/images/gpekd5/post/5d6a3be8-11c9-44d5-8229-3123b7f48190/image.png)

App EC2 1대를 중지했을 때 외부 API `10 / 10` HTTP 200도 확인했지만, 이 결과는 Blue-Green 자체가 아니라 **ALB + Active App EC2 2대 구성의 런타임 장애 우회 결과**로 구분합니다.

## 5. 수치를 과장하지 않은 이유

단일 EC2 배포에서 실제 요청 불가 구간을 관측했고, Blue-Green에서는 관측 다운타임 0초를 확인했습니다.

다만 두 측정의 위치와 조건이 완전히 동일하지 않기 때문에 이를 단순한 `기존 다운타임 → 0초` 개선율로 계산하지 않습니다.

이번 검증에서 주장하는 범위는 다음입니다.

> Blue-Green 전환과 롤백 동안 외부 검증 요청에서 실패가 관측되지 않았다.

## 6. 또 다른 문제 — Inactive도 DB Connection을 유지했다

Blue-Green을 처음 구성했을 때는 Blue 2대와 Green 2대, 총 4대의 App EC2를 항상 실행했습니다.

RDS `PROCESSLIST`를 확인하니 실제 트래픽을 받지 않는 Inactive App도 HikariCP Connection을 유지하고 있었습니다.

```text
Active App 2대    → 약 20 Connection
Inactive App 2대  → 약 20 Connection

Threads_connected ≈ 45 / 60
```

![RDS PROCESSLIST App EC2 4대 Connection 유지](https://velog.velcdn.com/images/gpekd5/post/8a15ddca-f638-4580-bcbd-e3851c7d0e00/image.png)

즉 **Inactive는 트래픽만 받지 않을 뿐 Spring Boot와 HikariCP는 계속 실행 중**이었습니다.

두 선택지를 비교했습니다.

| 선택 | 장점 | 단점 |
|---|---|---|
| Inactive 계속 실행 | 즉시 롤백 가능 | EC2 비용 + DB Connection 점유 |
| Inactive EC2 중지 | 비용·Connection 절감 | 이미 중지한 환경으로 즉시 롤백 어려움 |

## 7. 최종 운영 — Inactive STOP + Rollback Window

최종적으로 두 장점을 절충했습니다.

```text
평상시
Active x2   → RUNNING
Inactive x2 → STOPPED

배포
Inactive START
→ 신규 버전 배포·검증
→ 트래픽 전환
→ 기존 Active 600초 유지
→ 문제 없으면 기존 Active STOP
```

배포 직후 `600초` 동안은 이전 환경이 살아 있으므로, 이 구간에서는 Listener만 되돌리는 빠른 롤백 경로를 유지합니다. 이후에는 이전 환경을 중지해 EC2 비용과 불필요한 DB Connection 점유를 줄입니다.

## 8. 트레이드오프

Blue-Green은 배포 안정성을 높였지만 비용이 사라지는 구조는 아닙니다.

- 배포 중 App EC2 최대 4대 동시 기동
- Blue/Green이 동일 RDS를 사용하므로 DB Connection Budget 고려 필요
- 신·구 버전이 잠시 함께 존재하므로 DB Schema 하위 호환 필요
- 애플리케이션 배포가 무중단이어도 RDS·Kafka·Redis 장애까지 해결하는 것은 아님

## 관련 문서

- [ADR 0013 - ALB 기반 Blue-Green 배포](../adr/0013-blue-green-deployment.md)
- [ADR 0016 - Blue-Green DB Schema 호환](../adr/0016-blue-green-db-schema-compatibility.md)
- [System Architecture](../architecture/system-architecture.md)
- [Backend Final Claim Matrix](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/evidence/v3/FINAL_CLAIM_MATRIX.md)
