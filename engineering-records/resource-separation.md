# [인프라] 단일 EC2 메모리 장애와 자원 분리

## 1. 문제 상황

초기 운영에서는 하나의 App EC2에서 Spring Boot, Redis, Kafka를 함께 실행했습니다.

```text
App EC2
├─ Spring Boot
├─ Redis
└─ Kafka
```

운영 중 ALB Health Check와 Prometheus 수집이 동시에 실패하고, SSM 응답도 지연되는 장애가 발생했습니다.

당시 주요 상태는 다음과 같았습니다.

- 메모리 사용률 약 `92.76%`
- 가용 메모리 약 `6MB`
- CPU idle 약 `81%`
- EBS 사용률 약 `19%`
- 1분 Load Average 약 `4.10`
- Spring Boot / Kafka 컨테이너 `Exited(255)`
- Redis 컨테이너는 실행 상태 유지

CPU가 먼저 포화된 상황이 아니라 메모리 여유가 거의 사라진 상태에서 여러 프로세스가 같은 EC2 자원을 경쟁하고 있었습니다.

### 장애 당시 관측 화면

**Grafana Memory 약 92.76%**

![장애 당시 Grafana Memory 92.76%](https://velog.velcdn.com/images/gpekd5/post/1507c5e1-9366-4ee0-8f44-5a760c0b7ec7/image.png)

**ALB Health Check Timeout / 장애 알림**

![ALB Health Check Timeout 또는 Slack 장애 알림](https://velog.velcdn.com/images/gpekd5/post/9038065c-38d6-46d1-b052-842ee7bbdeb9/image.png)

배포 측면에서도 단일 App EC2에서 기존 컨테이너를 교체하는 동안 대표적으로 약 `40.25초`의 서비스 접근 불가 구간을 관측했습니다.

![단일 EC2 배포 접근 불가 구간 측정](https://velog.velcdn.com/images/gpekd5/post/189f510b-00d8-4d42-9d38-9dbbddcba728/image.png)

> 이 값과 이후 Blue-Green의 관측 다운타임 0초는 측정 위치와 조건이 완전히 동일하지 않아 단순 개선율로 계산하지 않습니다.

## 2. 장애를 통해 확인한 구조적 문제

단일 서버는 단순했지만 Spring Boot, Redis, Kafka의 장애 경계와 자원 경계가 같았습니다.

```text
Spring Boot 메모리 증가
Kafka 메모리 사용
Redis 메모리 사용
       ↓
하나의 EC2 메모리 경쟁
       ↓
App 장애가 전체 API 중단으로 연결
```

또 App EC2를 여러 대로 늘릴 경우 더 큰 문제가 생겼습니다.

App 내부 Redis를 그대로 복제하면 다음 상태가 서버마다 달라질 수 있습니다.

- Refresh Token
- Access Token Blacklist
- 검색 Cache
- 채팅 Pub/Sub

Kafka 역시 App 서버마다 별도 Broker가 생기는 구조가 되어 같은 이벤트 파이프라인을 공유하기 어려워집니다.

## 3. 개선 — 공유 자원을 App 밖으로 분리

### Redis → ElastiCache for Valkey

Redis 역할을 App EC2 밖의 공용 저장소로 분리했습니다.

```text
App EC2 #1 ─┐
            ├→ ElastiCache for Valkey
App EC2 #2 ─┘
```

Valkey는 다음 역할을 공용으로 처리합니다.

- 인증 토큰 상태
- 검색 Cache
- 다중 App 채팅 Redis Pub/Sub

### Kafka → 전용 EC2

Kafka도 App EC2에서 분리해 전용 EC2의 KRaft Broker로 운영했습니다.

```text
App EC2 #1 ─┐
            ├→ Kafka EC2
App EC2 #2 ─┘
```

MSK는 관리 편의성과 고가용성 측면의 장점이 있지만 프로젝트 규모에서는 비용 부담이 커, 현재는 전용 EC2의 단일 Broker를 선택했습니다.

### 구조 변화

![단일 App EC2에서 공유 자원 분리와 Multi-AZ App 구성으로 변경](https://velog.velcdn.com/images/gpekd5/post/b9144db7-d58e-4263-aada-1f4cd053e4b7/image.png)

## 4. App EC2 이중화 후 장애 우회 검증

공유 자원을 분리한 뒤 App EC2 2대를 서로 다른 가용 영역에 배치했습니다.

- `ap-northeast-2a` → App EC2 #1
- `ap-northeast-2c` → App EC2 #2

두 Target이 정상 상태인지 먼저 확인했습니다.

![Target Group App EC2 2대 Healthy](https://velog.velcdn.com/images/gpekd5/post/a73b70c4-4a56-48b5-bc0c-03e25d2111b8/image.png)

이후 App #1의 Spring Boot 컨테이너를 직접 중지해 장애를 재현했습니다.

```text
App EC2 #1 → Unhealthy
App EC2 #2 → Healthy
```

![App #1 중지 후 Unhealthy Healthy 상태](https://velog.velcdn.com/images/gpekd5/post/40920431-d7fe-462a-bce2-f3bf4a2e68fb/image.png)

이 상태에서 외부 API를 10회 연속 호출했고 `10 / 10` 모두 HTTP 200으로 처리됐습니다.

![App 한 대 장애 상태의 외부 API 10회 HTTP 200](https://velog.velcdn.com/images/gpekd5/post/262d1090-bc1f-47e7-b85d-174d75fa6f9b/image.png)

이 검증에서 확인한 범위는 **App 한 대가 비정상일 때 ALB가 해당 Target을 제외하고 정상 App으로 요청을 우회할 수 있음**입니다.

## 5. 개선의 의미

이번 분리의 목적은 단순히 서버를 더 쓰는 것이 아니었습니다.

- App 서버가 애플리케이션 처리에 집중
- App 인스턴스가 늘어나도 Redis 상태를 공유
- Kafka Broker 생명주기를 App 배포와 분리
- App 장애가 Redis/Kafka 프로세스와 같은 호스트 자원 경쟁으로 직접 연결되는 범위 축소
- App 한 대 장애 시 다른 App으로 요청 우회 가능

이 분리가 선행되어야 App EC2를 여러 대로 늘리는 구조가 자연스러워졌습니다.

## 6. 남은 한계

자원을 분리했다고 전체 시스템이 고가용성이 된 것은 아닙니다.

현재 최종 범위에서도:

- RDS는 Single-AZ
- ElastiCache는 단일 노드
- Kafka는 Single KRaft Broker

이므로 각 계층은 별도의 장애 지점이 남아 있습니다.

따라서 이번 프로젝트에서는 **애플리케이션 계층의 장애 우회와 배포 안정성은 실제 검증했지만, 데이터·메시징 계층 HA까지 보장한다고 표현하지 않습니다.**

## 관련 문서

- [[인프라] AWS 인프라 발전 과정](./infrastructure-evolution.md)
- [Kafka 전용 EC2 선택 ADR](../adr/0018-kafka-dedicated-ec2-over-msk.md)
- [[인프라] 최종 운영 구조와 남은 고가용성 과제](./final-infrastructure-retrospective.md)
