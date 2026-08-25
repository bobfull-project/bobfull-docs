# [인프라] 최종 운영 구조와 남은 고가용성 과제

이 문서는 BobFull 인프라 관련 Velog 시리즈의 마지막 기록을 기준으로, 실제 최종 운영 범위와 남은 한계를 한 번에 정리한 문서입니다.

원본 기록:
- [단일 EC2 SPOF에서 Multi-AZ Blue-Green까지](https://velog.io/@gpekd5/%EC%B5%9C%EC%A2%85-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8%ED%8A%B8%EB%9F%AC%EB%B8%94%EC%8A%88%ED%8C%85-%EB%8B%A8%EC%9D%BC-EC2-SPOF%EC%97%90%EC%84%9C-Multi-AZ-Blue-Green%EA%B9%8C%EC%A7%80)

## 1. 출발점 — 단일 EC2가 곧 전체 서비스 장애 지점

초기에는 App EC2 한 대가 백엔드 실행의 중심이었습니다.

```text
Client
→ App EC2 1대
→ RDS
```

같은 EC2에서 Spring Boot, Redis, Kafka를 함께 실행하던 시점에 실제 메모리 경쟁 장애가 발생했습니다.

![장애 당시 Grafana Memory 92.76%](https://velog.velcdn.com/images/gpekd5/post/1507c5e1-9366-4ee0-8f44-5a760c0b7ec7/image.png)

![ALB Health Check Timeout 또는 Slack 장애 알림](https://velog.velcdn.com/images/gpekd5/post/9038065c-38d6-46d1-b052-842ee7bbdeb9/image.png)

배포도 단일 App 컨테이너를 직접 교체해 대표적으로 약 `40.25초`의 서비스 접근 불가 구간을 관측했습니다.

![단일 EC2 배포 접근 불가 구간 측정](https://velog.velcdn.com/images/gpekd5/post/189f510b-00d8-4d42-9d38-9dbbddcba728/image.png)

> 이후 Blue-Green의 관측 다운타임 0초와는 측정 위치·조건이 완전히 같지 않으므로 단순 개선율로 계산하지 않습니다.

## 2. App 확장 전에 공유 자원부터 분리

App EC2를 그대로 복제하면 내부 Redis와 Kafka도 서버별로 나뉘어 상태 공유 문제가 생깁니다.

그래서 먼저 다음을 분리했습니다.

- Redis → ElastiCache for Valkey
- Kafka → 전용 Kafka EC2
- Monitoring → 별도 Monitoring EC2

![단일 App EC2에서 공유 자원 분리와 Multi-AZ App 구성으로 변경](https://velog.velcdn.com/images/gpekd5/post/b9144db7-d58e-4263-aada-1f4cd053e4b7/image.png)

이를 통해 App 인스턴스 자체를 복제할 수 있는 기반을 만들었습니다.

## 3. 애플리케이션 계층 Multi-AZ

Active App EC2 2대를 서로 다른 AZ에 배치했습니다.

```text
ALB
├─ App #1 / ap-northeast-2a
└─ App #2 / ap-northeast-2c
```

두 Target이 Healthy인 상태를 확인한 뒤 한 App의 Spring Boot 컨테이너를 직접 중지했습니다.

![Target Group App EC2 2대 Healthy](https://velog.velcdn.com/images/gpekd5/post/a73b70c4-4a56-48b5-bc0c-03e25d2111b8/image.png)

![App #1 중지 후 Unhealthy Healthy 상태](https://velog.velcdn.com/images/gpekd5/post/40920431-d7fe-462a-bce2-f3bf4a2e68fb/image.png)

이 상태에서 외부 API `10 / 10` HTTP 200을 확인했습니다.

![App 한 대 장애 상태의 외부 API 10회 HTTP 200](https://velog.velcdn.com/images/gpekd5/post/262d1090-bc1f-47e7-b85d-174d75fa6f9b/image.png)

이 결과는 **애플리케이션 서버 한 대 장애 시 ALB가 정상 App으로 요청을 우회할 수 있음**을 확인한 범위입니다.

## 4. App이 두 대가 되자 실시간 채팅도 다시 검증

WebSocket Session은 각 App 메모리에 있기 때문에 서로 다른 App에 연결된 사용자끼리도 Redis Pub/Sub을 통해 메시지를 주고받는지 확인했습니다.

![다중 App 환경 실제 채팅 화면](https://velog.velcdn.com/images/gpekd5/post/62157471-0805-4cb4-8b41-510097f648f0/image.png)

한 App에서 `messageId=29`를 Publish했을 때 다른 App에서 같은 메시지를 Subscribe하는 로그를 확인했습니다.

![EC2 1 PUBLISHED messageId 29](https://velog.velcdn.com/images/gpekd5/post/c09c88c5-1124-4ba0-88f5-4285de2d99a9/image.png)

![EC2 2 SUBSCRIBED messageId 29](https://velog.velcdn.com/images/gpekd5/post/5bbf0321-9696-4495-b066-9a4736dbc5f4/image.png)

즉 서버 수만 늘린 것이 아니라 기존 기능이 다중 App 환경에서도 동작하는지 함께 검증했습니다.

## 5. Blue-Green으로 배포 장애도 분리

애플리케이션 장애 우회와 별개로 배포 중단 문제를 해결하기 위해 Blue/Green 환경을 구성했습니다.

```text
                AZ 2a             AZ 2c

Blue           Blue #1           Blue #2
Green          Green #1          Green #2
```

![Blue Green App EC2 구성](https://velog.velcdn.com/images/gpekd5/post/27ee7dc5-7a6c-4db8-be79-0c5636bea8e1/image.png)

![ALB Listener Blue 100 Green 0](https://velog.velcdn.com/images/gpekd5/post/ed5e28ec-1084-498e-a69a-237748da29ac/image.png)

배포 시에는 Inactive 환경을 기동해 새 버전을 배포·검증한 뒤 ALB 트래픽을 전환합니다.

![GitHub Actions Blue Green Workflow](https://velog.velcdn.com/images/gpekd5/post/0509d14f-b94f-4fa0-b17a-7b97761a622b/image.png)

검증 결과:

- 정상 전환: `2,787 / 2,787` HTTP 200
- 실패 `0`
- 관측 다운타임 `0초`

![Blue Green 배포 중 HTTP 200 연속 유지](https://velog.velcdn.com/images/gpekd5/post/7143526a-9b5a-49ef-a2f3-976089ff10fc/image.png)

정상 배포뿐 아니라 외부 API 검증을 의도적으로 실패시켜 기존 Listener 상태로 돌아가는 롤백 경로도 확인했습니다.

- 롤백 전체 과정: `2,758 / 2,758` HTTP 200

![외부 검증 실패 후 자동 Rollback Workflow](https://velog.velcdn.com/images/gpekd5/post/5d6a3be8-11c9-44d5-8229-3123b7f48190/image.png)

## 6. Inactive App의 DB Connection 점유 발견

Blue-Green 초기에는 Blue 2대와 Green 2대, 총 4대의 App EC2를 항상 실행했습니다.

RDS `PROCESSLIST`를 확인하니 실제 트래픽을 받지 않는 Inactive App도 각각 HikariCP Connection을 유지하고 있었습니다.

```text
Active App 2대    → 약 20 Connection
Inactive App 2대  → 약 20 Connection
Threads_connected ≈ 45 / 60
```

![RDS PROCESSLIST App EC2 4대 Connection 유지](https://velog.velcdn.com/images/gpekd5/post/8a15ddca-f638-4580-bcbd-e3851c7d0e00/image.png)

최종적으로는 평시에 Inactive App을 중지하고, 배포 시 기동한 뒤 트래픽 전환 후 이전 Active 환경을 `600초` 동안 유지하는 Rollback Window를 두었습니다.

```text
평상시
Active x2   RUNNING
Inactive x2 STOPPED

배포
Inactive START
→ 신규 버전 배포·검증
→ 트래픽 전환
→ 기존 Active 600초 유지
→ 문제 없으면 기존 Active STOP
```

이렇게 DB Connection과 EC2 비용을 줄이면서도 배포 직후의 빠른 롤백 경로는 유지했습니다.

## 7. Auto Scaling은 적용하지 않음

다중 App을 구성했다고 트래픽 Auto Scaling까지 바로 적용하지 않았습니다.

Active App 2대 조건의 Auto Scaling 판단 실험에서는 CPU보다 HikariCP 대기가 먼저 나타나는 구간을 확인했고, Pool 조건을 조정했을 때 Pending이 거의 0으로 감소하는 경향을 확인했습니다.

현재 측정만으로는 ASG와 Scaling Policy의 복잡도와 비용을 정당화하기 부족해 Auto Scaling은 미도입으로 남겼습니다.

> 인기 회차 최고 Stress에서 CPU/HikariCP 포화를 본 실험과 Auto Scaling 적용 여부를 판단한 실험은 서로 다른 조건의 실험이며, 같은 결과로 섞어 해석하지 않습니다.

## 8. 최종 운영 구조

```text
Frontend
Route 53 → CloudFront → S3

Backend
Route 53 → ALB
→ Blue / Green Target Group
→ Active App EC2 x2

Data
RDS MySQL
ElastiCache for Valkey

Messaging
Kafka EC2 / Single KRaft Broker

Monitoring
Monitoring EC2
→ Prometheus
→ Grafana Alerting
→ Slack

CI/CD
GitHub Actions
→ ECR
→ SSM
→ Inactive App 배포
→ Health Check
→ 트래픽 전환 / 롤백
```

## 9. 이번 프로젝트에서 실제로 주장할 수 있는 HA 범위

### 검증한 범위

- Active App EC2 2대
- App 1대 장애 시 ALB 우회
- Blue-Green 트래픽 전환
- 신규 환경 검증 실패 시 롤백
- 다중 App 채팅 Redis Pub/Sub 전달

### 아직 남은 장애 지점

#### RDS

현재 Single-AZ입니다.

향후 운영 단계에서는 RDS Multi-AZ를 적용하고 Failover 시 애플리케이션 연결 복구를 검증할 필요가 있습니다.

#### ElastiCache

현재 단일 노드입니다.

향후 Replica와 Automatic Failover를 구성하고, 장애 전환 시 인증·Cache·Pub/Sub과 Readiness 영향까지 확인해야 합니다.

#### Kafka

현재 Single KRaft Broker입니다.

향후 Multi-Broker 또는 MSK를 검토하고, Broker 장애와 적체 재처리까지 검증해야 합니다.

## 10. 최종 판단

이번 인프라 고도화의 핵심은 AWS 서비스를 많이 추가하는 것이 아니었습니다.

```text
단일 EC2 장애 확인
→ 자원 경쟁 원인 분석
→ 공유 자원 분리
→ App 이중화
→ 다중 App 기능 검증
→ Blue-Green 배포
→ 실제 부하 측정
→ Auto Scaling 필요성 판단
```

측정으로 필요성이 확인된 영역만 적용하고, RDS·Redis·Kafka처럼 아직 검증하지 않은 계층은 TODO로 명확히 남겼습니다.

## 관련 문서

- [[인프라] AWS 인프라 발전 과정](./infrastructure-evolution.md)
- [[인프라] 단일 EC2 메모리 장애와 자원 분리](./resource-separation.md)
- [[실시간] 다중 App 채팅과 Redis Pub/Sub](./realtime-multi-app-chat.md)
- [[배포] Blue-Green 무중단 배포와 롤백](./blue-green-deployment.md)
- [[성능] Query·Cache·Hikari 병목과 확장 판단](./performance-and-scaling.md)
- [System Architecture](../architecture/system-architecture.md)
