# [인프라] AWS 인프라 발전 과정

BobFull의 인프라는 처음부터 복잡한 구성을 만들기보다, 실제 운영에서 확인한 문제를 기준으로 단계적으로 확장했습니다.

## 1. V1 — 서비스가 실제로 동작하는 최소 운영 환경

초기 목표는 프론트엔드와 백엔드를 실제 AWS 환경에 올리고, 사용자가 화면에서 API를 호출할 수 있는 최소 운영 구조를 만드는 것이었습니다.

```text
Browser
→ Frontend S3
→ App EC2 / Spring Boot
→ RDS MySQL
```

이 단계에서 함께 적용한 항목은 다음과 같습니다.

- Spring Profile 기반 로컬/운영 설정 분리
- Parameter Store 기반 Secret 관리
- Docker 기반 백엔드 실행
- RDS MySQL 분리
- S3 이미지 저장
- CloudWatch Logs 수집
- GitHub Actions 기반 배포 자동화

관련 원본 기록:
- [배포 준비 - V1 범위 정리](https://velog.io/@gpekd5/%EC%B5%9C%EC%A2%85-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%EB%B0%B0%ED%8F%AC-%EC%A4%80%EB%B9%84-V1-%EB%B2%94%EC%9C%84-%EC%A0%95)
- [배포 #2 - 로컬·운영 환경 설정 분리](https://velog.io/@gpekd5/%EC%B5%9C%EC%A2%85-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%EB%B0%B0%ED%8F%AC-1-%EB%A1%9C%EC%BB%AC%EC%9A%B4%EC%98%81-%ED%99%98%EA%B2%BD-%EC%84%A4%EC%A0%95-%EB%B6%84%EB%A6%AC)

## 2. V2 — 운영 관측과 외부 노출 구조 개선

기능이 늘어나면서 단순히 서버가 켜져 있는지만 확인해서는 운영 상태를 판단하기 어려워졌습니다.

그래서 다음 영역을 분리했습니다.

```text
Application EC2
├─ Spring Boot
└─ Redis

Monitoring EC2
├─ Prometheus
└─ Grafana

RDS MySQL
S3 + Lambda
CloudWatch Logs
```

또한 초기에는 App EC2의 8080 포트를 외부에 직접 노출했지만, 이후 ALB와 HTTPS를 적용해 외부 진입점을 분리했습니다.

```text
이전
Browser → App EC2 :8080

개선
Browser → HTTPS :443 → ALB → App EC2 :8080
```

이 단계에서 프론트엔드는 CloudFront를 통해 전달하고, 이미지는 Presigned URL로 Browser가 S3에 직접 업로드한 뒤 Lambda에서 검증하는 구조로 확장했습니다.

관련 원본 기록:
- [V2 인프라 구성 정리 및 후속 개선 계획](https://velog.io/@gpekd5/%EC%B5%9C%EC%A2%85-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-V2-%EC%9D%B8%ED%94%84%EB%9D%BC-%EA%B5%AC%EC%84%B1-%EC%A0%95%EB%A6%AC-%EB%B0%8F-%ED%9B%84%EC%86%8D-%EA%B0%9C%EC%84%A0-%EA%B3%84%ED%9A%8D)
- [배포 #10 - 백엔드 진입 구조 및 HTTPS 개선 검토](https://velog.io/@gpekd5/%EC%B5%9C%EC%A2%85-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%EB%B0%B0%ED%8F%AC-10-%EB%B0%B1%EC%97%94%EB%93%9C-%EC%A7%84%EC%9E%85-%EA%B5%AC%EC%A1%B0-%EB%B0%8F-HTTPS-%EA%B0%9C%EC%84%A0-%EA%B2%80%ED%86%A0)

## 3. 단일 App EC2의 한계

운영 중 단일 App EC2에서 Spring Boot, Redis, Kafka가 함께 자원을 사용하던 시점에 메모리 경쟁으로 애플리케이션이 불안정해지는 문제가 발생했습니다.

또 App EC2를 여러 대로 늘리려면 Redis와 Kafka를 App 내부에 함께 두는 구조도 문제가 됐습니다.

```text
App #1 내부 Redis
App #2 내부 Redis

→ Refresh Token / Blacklist / Cache / Pub/Sub 상태가 분리될 수 있음
```

따라서 공유 자원을 애플리케이션 서버 밖으로 분리했습니다.

- Redis → Amazon ElastiCache for Valkey
- Kafka → 전용 Kafka EC2
- App EC2 → 애플리케이션 실행 책임에 집중

## 4. 최종 — Application Layer Multi-AZ + Blue-Green

최종 운영 구조에서는 App 서버를 `ap-northeast-2a`, `ap-northeast-2c`에 분산하고 Blue/Green 환경을 각각 2대로 구성했습니다.

평시에는 Active 환경의 App EC2 2대만 트래픽을 처리하고, 배포할 때 Inactive 환경을 기동해 검증한 뒤 ALB 트래픽을 전환합니다.

<img width="1642" height="952" alt="BobFull System Architecture" src="https://github.com/user-attachments/assets/5a1371a7-7486-4fca-8a8a-43f8f1c44995" />

핵심 구성은 다음과 같습니다.

- Frontend: Route 53 → CloudFront → S3
- Backend: Route 53 → ALB → Blue/Green Target Group → Active App EC2 2대
- Data: RDS MySQL + ElastiCache for Valkey
- Messaging: 전용 Kafka EC2, Single KRaft Broker
- Monitoring: 별도 Monitoring EC2의 Prometheus + Grafana
- Logs: CloudWatch Logs
- CI/CD: GitHub Actions → ECR → SSM → App EC2

## 5. 고가용성 범위는 애플리케이션 계층까지

최종 구조를 전체 시스템 HA로 표현하지 않습니다.

현재 한계는 다음과 같습니다.

- RDS: Single-AZ
- ElastiCache: 단일 노드 구성
- Kafka: Single KRaft Broker

따라서 이번 프로젝트에서 실제로 검증한 고가용성 범위는 **ALB 뒤 Active App EC2 2대와 Blue-Green 트래픽 전환을 포함한 애플리케이션 계층**입니다.

## 관련 문서

- [System Architecture](../architecture/system-architecture.md)
- [Blue-Green ADR](../adr/0013-blue-green-deployment.md)
- [Kafka 전용 EC2 선택 ADR](../adr/0018-kafka-dedicated-ec2-over-msk.md)
- [[인프라] 최종 운영 구조와 남은 고가용성 과제](./final-infrastructure-retrospective.md)
