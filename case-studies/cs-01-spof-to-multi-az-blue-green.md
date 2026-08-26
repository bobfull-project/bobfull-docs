# CS-01 — 단일 EC2 SPOF에서 Multi-AZ Blue-Green까지

> 원본 분류: `[발표]` 트러블슈팅 · 작성자: 김홍기

## 1. 문제 발견 — App EC2 한 대가 전체 서비스의 단일 장애 지점이었다

초기 운영 구조는 ALB를 사용하고 있었지만 실제 요청을 처리하는 App EC2는 한 대뿐이었고, 같은 인스턴스에서 Spring Boot·Redis·Kafka를 함께 실행했다.

실제 장애 당시 메모리 사용률은 약 **92.76%**까지 올라갔고 ALB Health Check와 Prometheus 요청까지 Timeout이 발생했다. 단일 EC2 교체 배포에서도 신규 컨테이너가 준비되는 동안 최종 재측정 기준 약 **40.25초**의 접근 불가 구간이 관측됐다.

문제는 단순히 인스턴스 사양이 작은 것이 아니라 **App 한 대의 장애와 배포가 서비스 전체 중단으로 연결되는 구조**였다.

## 2. App EC2만 그대로 복제할 수는 없었다

기존 구성을 그대로 두 대로 복제하면 App마다 Redis와 Kafka가 따로 생긴다.

```text
EC2 #1                  EC2 #2
├─ Spring Boot          ├─ Spring Boot
├─ Redis #1             ├─ Redis #2
└─ Kafka #1             └─ Kafka #2
```

Redis에는 Refresh Token, Access Token Blacklist, 검색 Cache, Redis Pub/Sub처럼 여러 App이 함께 사용해야 하는 상태가 있었다. Kafka도 App과 같은 EC2에 두면 CPU·Memory를 공유하고 App 장애·교체가 Broker에 영향을 준다.

따라서 App을 늘리기 전에 공유 자원을 분리했다.

- Redis → **Amazon ElastiCache for Valkey**
- Kafka → **Kafka 전용 EC2**

### Redis는 왜 ElastiCache + Valkey인가

App 내부 Redis는 인스턴스별 상태 분절 문제가 있고, Redis 전용 EC2는 패치·모니터링·장애 대응을 직접 운영해야 한다. 여러 App이 하나의 상태를 공유하면서 관리 범위를 줄이기 위해 ElastiCache를 선택했다.

엔진은 기존 Redis 기반 인증·Cache·Pub/Sub 코드를 크게 바꾸지 않고 사용할 수 있고, BSD-3-Clause 오픈소스이며, 당시 서울 리전 `cache.t4g.micro` 공개 요금 기준 Redis OSS보다 약 **20% 낮은 비용**으로 계산된 Valkey를 선택했다.

| 항목 | Redis OSS | Valkey |
|---|---:|---:|
| 시간당 비용 | $0.0240 | $0.0192 |
| 월 730시간 가정 | $17.52 | $14.02 |
| 연 8,760시간 가정 | $210.24 | $168.19 |

> 이 표는 당시 공개 요금 기준 추정치이며 실제 청구 금액을 의미하지 않는다.

### Kafka는 왜 MSK가 아니라 전용 EC2인가

Kafka는 App과 자원을 분리할 필요가 있었지만 현재 규모에서 관리형 다중 Broker 비용을 바로 감수할 근거는 부족했다.

| 구성 | 최소 구성 | 월 예상 컴퓨팅 비용 | 판단 |
|---|---|---:|---|
| Kafka 전용 EC2 | `t3.small` 1대 | 약 $18.69 + EBS | 선택 |
| MSK Provisioned | `kafka.t3.small` 2 Broker | 약 $83.07 + Storage | 약 4.4배 |
| MSK Provisioned | `kafka.t3.small` 3 Broker | 약 $124.61 + Storage | 약 6.7배 |

현재 필요한 Producer/Consumer, Consumer Group, Retry/DLT, 중복 처리 방어는 단일 Broker에서도 검증할 수 있었기 때문에 **App과 Broker의 자원 경계를 먼저 분리**했다. 대신 단일 KRaft Broker는 Kafka 계층의 HA가 아니라는 한계를 명시한다.

## 3. 최종 방향 — Multi-AZ Blue-Green

App은 `ap-northeast-2a / 2c`에 분산하고 Blue/Green 각각 두 대를 둘 수 있는 구조로 확장했다. 평시에는 Active 환경의 App 2대만 트래픽을 받고, 배포 시 Inactive 환경을 기동해 동일 이미지를 배포·Readiness 검증한 뒤 ALB Weight를 전환한다.

핵심은 **새 버전을 먼저 검증하고 트래픽을 바꾸는 것**이다.

```text
Inactive 배포
→ Readiness 확인
→ Target Group Healthy
→ 사전 검증
→ ALB Traffic Switch
→ 전환 후 검증
→ 문제 시 Rollback
```

Blue-Green 전환 검증에서는 **2,787건 연속 요청이 모두 HTTP 200**, 관측 다운타임 **0초**였다. 이 결과는 App 배포 전환 범위의 실측이며 RDS·Redis·Kafka 전체 계층의 무중단을 의미하지 않는다.

## 4. 현재 한계

- RDS는 현재 Single-AZ이므로 DB 계층 Failover까지 검증하지 않았다.
- ElastiCache는 공유 자원 분리까지 적용했으며 Replica/Automatic Failover는 후속 과제다.
- Kafka는 단일 KRaft Broker이므로 Broker 장애 시 HA를 보장하지 않는다.
- Auto Scaling은 부하 측정 결과를 근거로 당장 도입하지 않았다.

## 관련 문서

- [System Architecture](../architecture/system-architecture.md)
- [단일 EC2 메모리 장애와 자원 분리](../engineering-records/resource-separation.md)
- [Blue-Green 무중단 배포와 롤백](../engineering-records/blue-green-deployment.md)
- [최종 운영 구조와 남은 고가용성 과제](../engineering-records/final-infrastructure-retrospective.md)
- [ADR-0013 — Blue-Green Deployment](../adr/0013-blue-green-deployment.md)
- [ADR-0018 — Kafka Dedicated EC2 over MSK](../adr/0018-kafka-dedicated-ec2-over-msk.md)
