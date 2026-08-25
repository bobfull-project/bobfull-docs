# 🧭 기술 기록

BobFull 프로젝트를 진행하며 남긴 Velog 기록과 최종 Evidence를 바탕으로, **문제 → 판단 → 적용 → 검증 → 남은 한계** 흐름이 보이도록 다시 정리한 기술 기록입니다.

Velog 글을 시간순으로 그대로 복사하지 않고, 여러 글에 흩어진 내용을 하나의 기술 주제로 합쳤습니다. 수치와 최종 상태가 바뀐 항목은 `bobfull-backend/docs/evidence/v3`와 현재 운영 문서를 우선 기준으로 정리합니다.

## 문서 목록

| 구분 | 문서 | 핵심 내용 |
|---|---|---|
| 인프라 | [[인프라] AWS 인프라 발전 과정](./infrastructure-evolution.md) | V1 단일 EC2에서 최종 Blue-Green App 2대 구조까지 |
| 배포 | [[배포] Blue-Green 무중단 배포와 롤백](./blue-green-deployment.md) | 단일 배포 중단 문제, 비활성 환경 검증, 트래픽 전환과 롤백 |
| 운영설정 | [[배포] 운영 환경 분리와 HTTPS 진입 구조](./environment-and-https.md) | Spring Profile, Parameter Store, SSM, ALB/HTTPS |
| 모니터링 | [[모니터링] 운영 관측 체계 구축](./monitoring-observability.md) | CloudWatch Logs + Prometheus + Grafana + Slack 역할 분리 |
| 장애대응 | [[인프라] 단일 EC2 메모리 장애와 자원 분리](./resource-separation.md) | 메모리 경쟁 장애, Redis·Kafka 분리, 공유 상태 문제 |
| 실시간 | [[실시간] 다중 App 채팅과 Redis Pub/Sub](./realtime-multi-app-chat.md) | WebSocket/STOMP 다중 인스턴스 전달과 DB 복구 경계 |
| 성능 | [[성능] Query·Cache·Hikari 병목과 확장 판단](./performance-and-scaling.md) | 쿼리·인덱스·캐시 실측과 Auto Scaling 미도입 근거 |
| 최종정리 | [[인프라] 최종 운영 구조와 남은 고가용성 과제](./final-infrastructure-retrospective.md) | 최종 운영 범위, 검증 결과, RDS·Redis·Kafka TODO |

## 읽는 기준

- 이 문서는 **포트폴리오용 서술 기록**입니다.
- 상세 구현 계약과 원본 Evidence는 [Backend Evidence](https://github.com/bobfull-project/bobfull-backend/tree/develop/docs/evidence/v3)를 기준으로 합니다.
- ADR 자체가 필요한 결정은 [ADR](../adr/README.md), 성능 수치 중심 문서는 [Performance](../performance/README.md)에서 별도로 확인할 수 있습니다.
- 전체 운영 구조는 [System Architecture](../architecture/system-architecture.md)를 기준으로 합니다.

## 원본 기록

- [Velog - gpekd5](https://velog.io/@gpekd5/posts)
- [최종 기록 - 단일 EC2 SPOF에서 Multi-AZ Blue-Green까지](https://velog.io/@gpekd5/%EC%B5%9C%EC%A2%85-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8%ED%8A%B8%EB%9F%AC%EB%B8%94%EC%8A%88%ED%8C%85-%EB%8B%A8%EC%9D%BC-EC2-SPOF%EC%97%90%EC%84%9C-Multi-AZ-Blue-Green%EA%B9%8C%EC%A7%80)
