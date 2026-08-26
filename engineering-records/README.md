# 📝 Engineering Records

BobFull 프로젝트를 진행하며 남긴 작업 기록과 최종 Evidence를 바탕으로 **문제 → 판단 → 적용 → 검증 → 남은 한계** 흐름이 보이도록 재구성한 기술 기록입니다.

이 디렉터리의 역할은 개별 버그나 공식 의사결정을 다시 복사하는 것이 아니라 **프로젝트 구조가 어떻게 발전했는지** 보여주는 것입니다.

## 문서 목록

| ID | 구분 | 문서 | 핵심 내용 |
|---|---|---|---|
| **ER-01** | 인프라 | [[인프라] AWS 인프라 발전 과정](./infrastructure-evolution.md) | V1 단일 EC2에서 최종 Blue-Green App 구조까지 |
| **ER-02** | 인프라 | [[인프라] Presigned URL과 Lambda 이미지 검증 파이프라인](./image-upload-pipeline.md) | Browser → S3 직접 업로드와 Lambda 검증 책임 분리 |
| **ER-03** | 장애대응 | [[인프라] 단일 EC2 메모리 장애와 자원 분리](./resource-separation.md) | 메모리 경쟁 장애, Redis·Kafka 분리, App 장애 우회 |
| **ER-04** | 배포 | [[배포] 운영 환경 분리와 HTTPS 진입 구조](./environment-and-https.md) | Spring Profile, Parameter Store, SSM, ALB/HTTPS |
| **ER-05** | 배포 | [[배포] GitHub Actions CI/CD 구축과 배포 과정 최적화](./cicd-evolution.md) | 수동 배포 자동화, 중복 Build·고정 대기·Layer 개선 |
| **ER-06** | 배포 | [[배포] Blue-Green 무중단 배포와 롤백](./blue-green-deployment.md) | 비활성 환경 검증, Traffic Switch, Rollback, Connection 절감 |
| **ER-07** | 모니터링 | [[모니터링] 운영 관측 체계 구축](./monitoring-observability.md) | CloudWatch + Prometheus + Grafana + Slack 역할 분리 |
| **ER-08** | 실시간 | [[실시간] 다중 App 채팅과 Redis Pub/Sub](./realtime-multi-app-chat.md) | WebSocket/STOMP 다중 인스턴스 전달과 DB 복구 경계 |
| **ER-09** | 성능 | [[성능] Query·Cache·Hikari 병목과 확장 판단](./performance-and-scaling.md) | 쿼리·인덱스·캐시 실측과 확장 판단 |
| **ER-10** | 최종정리 | [[인프라] 최종 운영 구조와 남은 고가용성 과제](./final-infrastructure-retrospective.md) | 최종 운영 범위, 검증 캡처, RDS·Redis·Kafka TODO |

> 기존 `engineering-records/` 파일 경로는 외부 링크 호환을 위해 유지하고, 탐색용 ID만 `ER-xx`로 통일합니다.

## 다른 문서와의 경계

- 발표용으로 최종 정제한 대표 스토리 → [Case Studies](../case-studies/README.md)
- 하나의 구체적인 장애·버그·병목 → [Troubleshooting](../troubleshooting/README.md)
- 공식/프로젝트 기술 결정 → [ADR](../adr/README.md), [Technical Decisions](../decisions/README.md)
- 측정 수치와 실험 조건 → [Performance](../performance/README.md)
- 전체 운영 구조 → [System Architecture](../architecture/system-architecture.md)

같은 사건이 여러 관점에 등장하더라도 본문을 중복 복사하지 않고 **역할에 맞는 문서로 링크**합니다.

## 대표 연결

- ER-03 단일 EC2 장애와 자원 분리 → [CS-01 — SPOF → Multi-AZ Blue-Green](../case-studies/cs-01-spof-to-multi-az-blue-green.md)
- ER-09 Query/Cache/Pool 최적화 → [CS-02 — 검색·정산 성능](../case-studies/cs-02-query-index-to-settlement-optimization.md)
- ER-06 Blue-Green → [ADR-0013](../adr/0013-blue-green-deployment.md)
- ER-02 이미지 Pipeline → [ADR-0007](../adr/0007-s3-presigned-restaurant-image-validation.md)

## 관리 기준

- Velog/Notion의 시간순 기록을 그대로 복제하지 않습니다.
- 최신 수치와 최종 상태는 `bobfull-backend/docs/evidence/v3`와 운영 문서를 우선합니다.
- 당시 캡처가 필요한 경우 장기적으로 유지 가능한 Repository 자산 또는 공개 Evidence 링크를 사용합니다.
- RDS Single-AZ, Kafka 단일 Broker 등 현재 한계를 그대로 적고 검증하지 않은 HA를 주장하지 않습니다.
