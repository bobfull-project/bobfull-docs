# 🍚 BobFull Technical Docs

합석형 좌석 예약 플랫폼 **BobFull(밥풀)**의 기술 문서·포트폴리오 자료 허브입니다.

[🏠 Project Home](https://github.com/bobfull-project) · [🔬 Flow Lab](https://bobfull-project.github.io/bobfull-docs/flow-lab/v3/operations-flow-lab/) · [⚙️ Backend](https://github.com/bobfull-project/bobfull-backend) · [🖥️ Frontend](https://github.com/bobfull-project/bobfull-frontend)

> 처음 보는 분께는 **[System Architecture](./architecture/system-architecture.md) → [Engineering Records](./engineering-records/README.md) → [대표 ADR 10선](./adr/README.md) → [Performance](./performance/README.md) → [Flow Lab](https://bobfull-project.github.io/bobfull-docs/flow-lab/v3/operations-flow-lab/)** 순서를 권장합니다.

## System Architecture

BobFull의 실제 운영 구성을 한눈에 볼 수 있도록 **Frontend 전달 경로, Blue-Green App, 데이터 저장소, Kafka, 모니터링, CI/CD, 외부 서비스**를 함께 표시합니다.

<img width="1642" height="952" alt="image" src="https://github.com/user-attachments/assets/5a1371a7-7486-4fca-8a8a-43f8f1c44995" />

> 평시에는 Blue/Green 중 **Active App EC2 2대만 서비스**하며, 배포 시 Inactive 환경을 기동해 동일 이미지를 배포·검증한 뒤 ALB Weight를 전환합니다.  
> RDS는 현재 **Single-AZ**, Kafka는 **단일 KRaft Broker**로 구성되어 있어 해당 계층의 HA까지 주장하지 않습니다.

**[▶ 상세 System Architecture와 책임 경계 보기](./architecture/system-architecture.md)**

## Documentation

| 문서 | 내용 |
|---|---|
| [System Architecture](./architecture/system-architecture.md) | 운영 기준 전체 시스템 구성 |
| [Engineering Records](./engineering-records/README.md) | 프로젝트 진행 기록을 문제·판단·적용·검증 흐름으로 재구성한 기술 기록 |
| [API](./api/README.md) | 도메인별 API와 권한 경계 요약 |
| [ERD](./database/erd.md) | 핵심 데이터 관계 구조 |
| [ADR](./adr/README.md) | 대표 기술 의사결정 10선 |
| [Performance](./performance/README.md) | 실제 측정 기반 주요 성능 개선·기술 비교 |
| [Troubleshooting](./troubleshooting/README.md) | 도메인별 문제 분석·해결 기록 |

## Engineering Records

시간순 작업 일지를 그대로 복사하지 않고 여러 기록에 흩어진 내용을 주제별로 묶었습니다. 인프라·배포·모니터링·실시간 처리·성능 확장 과정과 실제 검증 캡처를 함께 확인할 수 있습니다.

**[▶ Engineering Records 보기](./engineering-records/README.md)**

## Flow Lab

실제 코드와 Evidence를 기반으로 BobFull의 주요 백엔드 흐름을 단계별로 확인할 수 있는 인터랙티브 시뮬레이션입니다.

**[▶ Flow Lab V3 실행하기](https://bobfull-project.github.io/bobfull-docs/flow-lab/v3/operations-flow-lab/)**

## 문서 관리 기준

- 상세 구현 계약과 Evidence의 Source of Truth는 [Backend](https://github.com/bobfull-project/bobfull-backend) 저장소입니다.
- 이 저장소는 Architecture · Engineering Records · API · ERD · ADR · Performance · Troubleshooting을 포트폴리오 관점에서 읽기 쉽게 정리합니다.
- Engineering Records는 Velog 작업 기록을 그대로 복제하지 않고, 최종 구현과 Evidence를 기준으로 주제별 흐름을 재구성합니다.
- Performance는 핵심 결과와 판단만 요약하며, 측정 조건·Raw 결과·재현 방법은 Backend Evidence를 연결합니다.
- Flow Lab은 Backend의 `docs/flow-lab`을 기준으로 동기화해 GitHub Pages로 공개합니다.
- Troubleshooting은 각 담당자가 자신의 도메인 사례를 직접 보완합니다.
