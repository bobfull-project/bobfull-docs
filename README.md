# 🍚 BobFull Technical Docs

합석형 좌석 예약 플랫폼 **BobFull(밥풀)**의 기술 문서·포트폴리오 허브입니다.

[🏠 Project Home](https://github.com/bobfull-project) · [⚙️ Backend](https://github.com/bobfull-project/bobfull-backend) · [🖥️ Frontend](https://github.com/bobfull-project/bobfull-frontend) · [🔬 Flow Lab](https://bobfull-project.github.io/bobfull-docs/flow-lab/v3/operations-flow-lab/)

> 처음 보는 분께는 **[System Architecture](./architecture/system-architecture.md) → [Case Studies](./case-studies/README.md) → [ADR](./adr/README.md) → [Troubleshooting](./troubleshooting/README.md) → [Performance](./performance/README.md) → [Engineering Records](./engineering-records/README.md) → [Flow Lab](https://bobfull-project.github.io/bobfull-docs/flow-lab/v3/operations-flow-lab/)** 순서를 권장합니다.

## 1. System Architecture

BobFull의 실제 운영 구성과 책임 경계를 먼저 확인합니다.

<img width="1642" height="952" alt="BobFull System Architecture" src="https://github.com/user-attachments/assets/5a1371a7-7486-4fca-8a8a-43f8f1c44995" />

> 평시에는 Blue/Green 중 **Active App EC2 2대만 서비스**하며, 배포 시 Inactive 환경을 기동해 동일 이미지를 배포·검증한 뒤 ALB Weight를 전환합니다.  
> RDS는 현재 **Single-AZ**, Kafka는 **단일 KRaft Broker**이므로 해당 계층의 HA까지 주장하지 않습니다.

**[▶ System Architecture 상세 보기](./architecture/system-architecture.md)**

## 2. Case Studies

프로젝트에서 여러 실험·트러블슈팅·기술 판단을 거쳐 최종 발표용으로 정제한 대표 사례입니다.

| ID | 영역 | Case Study | 핵심 |
|---|---|---|---|
| **CS-01** | 고가용성 | [단일 EC2 SPOF → Multi-AZ Blue-Green](./case-studies/cs-01-spof-to-multi-az-blue-green.md) | 자원 분리부터 무중단 Traffic Switch까지 |
| **CS-02** | 성능 | [조회 인덱스 부재 → 정산 조회 병목 해소](./case-studies/cs-02-query-index-to-settlement-optimization.md) | Query·Index·Cache·Pool·CPU를 단계적으로 분리 |
| **CS-03** | 거래/이벤트 | [핵심 거래와 후속 작업의 실패 경계](./case-studies/cs-03-transaction-and-followup-failure-boundary.md) | AFTER_COMMIT → Outbox → Async |
| **CS-04** | AI | [AI 검수 호출 최적화와 우회 검수 보완](./case-studies/cs-04-ai-moderation-optimization.md) | Rule Fast Path·DB Context·Prompt Injection |
| **CS-05** | Kafka | [Outbox + Async vs Kafka](./case-studies/cs-05-outbox-async-vs-kafka.md) | 성능이 아니라 Consumer 격리 기준으로 선택 |
| **CS-06** | 설계 | [AFTER_COMMIT·Outbox·Kafka를 나눈 기준](./case-studies/cs-06-post-payment-processing-strategy.md) | 후속 작업마다 필요한 보장을 구분 |

**[▶ Case Studies 전체 보기](./case-studies/README.md)**

## 3. Documentation Map

| 문서 | ID 규칙 | 역할 |
|---|---|---|
| [Architecture](./architecture/system-architecture.md) | - | 최종 운영 구조와 시스템 책임 경계 |
| [Case Studies](./case-studies/README.md) | `CS-xx` | 최종 정제 대표 사례 |
| [ADR](./adr/README.md) | `ADR-xxxx` / `P-ADR-xxx` | 공식 아키텍처 의사결정과 Project ADR |
| [Technical Decisions](./decisions/README.md) | `TD-xx` | 도메인·정책·구현 단위 기술 판단 |
| [Troubleshooting](./troubleshooting/README.md) | `TS-xx` | 개별 문제의 원인·해결·검증 기록 |
| [Performance](./performance/README.md) | `PF-xx` | 실제 측정 기반 성능 결과와 기술 비교 |
| [Engineering Records](./engineering-records/README.md) | `ER-xx` | 프로젝트가 어떻게 발전했는지 보여주는 기술 기록 |
| [API](./api/README.md) | - | 도메인별 API와 권한 경계 |
| [ERD](./database/erd.md) | - | 핵심 데이터 관계 |
| [Flow Lab](https://bobfull-project.github.io/bobfull-docs/flow-lab/v3/operations-flow-lab/) | - | 실제 코드/Evidence 기반 백엔드 흐름 시뮬레이션 |

## 4. 문서 ID 규칙

문서 종류마다 하나의 번호 체계만 사용합니다.

```text
CS-01      Case Study
TD-01      Technical Decision
TS-01      Troubleshooting
PF-01      Performance
ER-01      Engineering Record
ADR-0001   Backend 공식 ADR
P-ADR-001  Project ADR
```

- 새로 만드는 `CS / TD / TS / P-ADR` 문서는 **파일명과 H1 제목까지 같은 ID**를 사용합니다.
- 기존 외부 링크가 있을 수 있는 `PF / ER` 문서는 경로를 유지하고 README의 탐색 ID만 붙입니다.
- Backend 공식 ADR 번호는 임의로 다시 매기지 않습니다.

## 5. 문서 역할을 나눈 기준

### Case Study
여러 문제·실험·의사결정을 하나의 이야기로 연결한 **최종 포트폴리오 문서**입니다.

### Troubleshooting
하나의 구체적인 실패·버그·병목을 다룹니다. 같은 내용이 Case Study에 포함돼도 상세 근거 문서로 연결합니다.

### ADR / Technical Decisions
“무슨 문제가 났는가”보다 **왜 이 구조를 선택했는가**에 집중합니다.

### Performance
측정 수치·조건·Trade-off에 집중합니다. Raw 결과와 재현 절차는 Backend Evidence를 우선합니다.

### Engineering Records
시간순 일지를 복사하지 않고 인프라·배포·모니터링·실시간·성능이 **어떻게 발전했는지**를 주제별로 재구성합니다.

## 6. Source of Truth

- 상세 구현 계약, 최신 코드, 공식 번호형 ADR, Raw Evidence의 Source of Truth는 [bobfull-backend](https://github.com/bobfull-project/bobfull-backend)입니다.
- 같은 내용을 여러 폴더에 복사하지 않고 한 문서를 기준으로 다른 문서에서 링크합니다.
- 측정하지 않은 개선 수치, 검증하지 않은 HA 범위, 외부 시스템의 보장 범위를 확대해서 쓰지 않습니다.
- Notion의 만료형 첨부 URL은 장기 문서 링크로 사용하지 않고 Backend Evidence 또는 안정적인 Repository 자산을 우선합니다.

## 7. Templates

새 기록은 아래 양식을 기준으로 작성합니다.

- [Troubleshooting Template](./_templates/troubleshooting-template.md)
- [Decision Template](./_templates/decision-template.md)
