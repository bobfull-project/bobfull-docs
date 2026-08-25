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

이 구조에서는 App EC2 장애나 배포 중 컨테이너 교체가 곧 사용자 요청 실패로 연결될 수 있었습니다.

운영 중 실제 메모리 경쟁 장애를 경험하면서 단순한 이론적 SPOF가 아니라 실제 장애 가능성을 확인했습니다.

## 2. App 확장 전에 공유 자원부터 분리

App EC2를 그대로 복제하면 내부 Redis와 Kafka도 서버별로 나뉘어 상태 공유 문제가 생깁니다.

그래서 먼저 다음을 분리했습니다.

- Redis → ElastiCache for Valkey
- Kafka → 전용 Kafka EC2
- Monitoring → 별도 Monitoring EC2

이를 통해 App 인스턴스 자체를 복제할 수 있는 기반을 만들었습니다.

## 3. Application Layer Multi-AZ

Active App EC2 2대를 서로 다른 AZ에 배치했습니다.

```text
ALB
├─ App #1 / ap-northeast-2a
└─ App #2 / ap-northeast-2c
```

App 한 대를 중지한 장애 검증에서는 외부 API `10 / 10` HTTP 200을 확인했습니다.

이 결과는 **애플리케이션 서버 한 대 장애 시 ALB가 정상 App으로 요청을 우회할 수 있음**을 확인한 범위입니다.

## 4. Blue-Green으로 배포 장애도 분리

애플리케이션 장애 우회와 별개로 배포 중단 문제를 해결하기 위해 Blue/Green 환경을 구성했습니다.

```text
Blue  App x2
Green App x2

평시 → 한 환경만 Active
배포 → Inactive 기동·검증 → ALB 트래픽 전환
```

검증 결과:

- 정상 전환: `2,787 / 2,787` HTTP 200
- 실패 `0`
- 관측 다운타임 `0초`
- 롤백 검증: `2,758 / 2,758` HTTP 200

평시에는 Inactive App을 중지해 불필요한 EC2 비용과 DB Connection 점유를 줄였습니다.

## 5. Auto Scaling은 적용하지 않음

다중 App을 구성했다고 트래픽 Auto Scaling까지 바로 적용하지 않았습니다.

Active App 2대 부하 조건에서 CPU보다 HikariCP 대기가 먼저 나타나는 구간을 확인했고, Pool 조건을 조정했을 때 Pending이 거의 0으로 감소하는 경향을 확인했습니다.

현재 측정만으로는 ASG와 Scaling Policy의 복잡도와 비용을 정당화하기 부족해 Auto Scaling은 미도입으로 남겼습니다.

## 6. 최종 운영 구조

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

## 7. 이번 프로젝트에서 실제로 주장할 수 있는 HA 범위

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

## 8. 최종 판단

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
- [[배포] Blue-Green 무중단 배포와 롤백](./blue-green-deployment.md)
- [[성능] Query·Cache·Hikari 병목과 확장 판단](./performance-and-scaling.md)
- [System Architecture](../architecture/system-architecture.md)
