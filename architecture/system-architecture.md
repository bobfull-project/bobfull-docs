# System Architecture

BobFull의 **운영 기준 전체 시스템 구성**을 포트폴리오 관점에서 한눈에 볼 수 있도록 정리한 문서입니다.

<img width="1642" height="952" alt="BobFull System Architecture" src="https://github.com/user-attachments/assets/5a1371a7-7486-4fca-8a8a-43f8f1c44995" />

> 평시에는 Blue/Green 중 **Active App EC2 2대만 서비스**하며, 배포 시 Inactive 환경을 기동해 동일 이미지를 배포·검증한 뒤 ALB Weight를 전환합니다.  
> RDS는 현재 **Single-AZ**, Kafka는 **단일 KRaft Broker**이므로 해당 계층의 HA까지 주장하지 않습니다.

## 핵심 구조

### Blue-Green 애플리케이션 배포

- ALB 뒤에 Blue / Green Target Group을 두고 각 환경은 App EC2 2대로 구성합니다.
- 평상시에는 Active 환경만 서비스하고, 배포 시 Inactive 환경을 기동해 동일 이미지를 배포합니다.
- Target Group Health Check와 외부 검증을 통과한 뒤 Listener Weight를 전환합니다.
- 전환 후 rollback window 동안 이전 Active 환경을 유지하고, Listener 상태를 다시 확인한 뒤 이전 환경을 중지합니다.

### 데이터 저장소와 메시징

- **RDS MySQL**: 예약, 결제, 환불, 채팅 메시지 등 영속 데이터의 기준 저장소
- **ElastiCache for Valkey**: Redis-compatible 공용 저장소로 인증 토큰 상태, 식당 검색 Cache, 다중 App 인스턴스 채팅 실시간 Pub/Sub fan-out에 사용
- **Kafka 전용 EC2**: AI Moderation과 Restaurant Feedback Insight의 비동기 후속 처리 경계. 현재 단일 KRaft Broker이므로 Kafka 계층 HA까지 보장하지 않음
- Redis Pub/Sub은 실시간 전달에 사용하고, 단절 중 놓친 채팅 메시지는 DB cursor 조회로 복구합니다.

### 외부 시스템

- **PortOne**: 결제 상태·금액 검증과 Webhook 처리
- **OpenAI**: AI Moderation 및 식당 피드백 분석 Provider
- **SMTP**: 예약 모집 결과 등 이메일 알림
- **S3 + Lambda**: 식당 이미지를 Presigned URL로 직접 업로드하고, ObjectCreated 이벤트 기반으로 파일을 검증한 뒤 최종 경로로 승격

### CI/CD와 운영

- GitHub Actions에서 검증 및 Docker Image Build 후 ECR에 Commit SHA 태그로 Push합니다.
- 운영 배포는 SSM Run Command와 Parameter Store를 이용해 App EC2에 동일 이미지를 배포합니다.
- Prometheus와 Grafana는 App EC2와 분리된 Monitoring EC2에서 동작하며 `/actuator/prometheus`를 수집합니다.
- Blue-Green 전환 후 Prometheus scrape target도 새 Active EC2 기준으로 갱신합니다.

---

### 더 자세히 보기

- [ADR](../adr/README.md)
- [Flow Lab에서 시스템 흐름 직접 실행하기](https://bobfull-project.github.io/bobfull-docs/flow-lab/v3/operations-flow-lab/)
- [Backend 상세 Architecture](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/ARCHITECTURE.md)
- [Backend AWS 배포 기준](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/deployment/aws-v1-backend.md)
