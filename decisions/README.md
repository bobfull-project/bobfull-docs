# 💡 Technical Decisions

정식 Architecture Decision Record까지는 아니지만, 구현 과정에서 여러 선택지를 비교하고 **왜 현재 방식을 선택했는지** 남긴 기록입니다.

`ADR`이 시스템 수준의 공식 결정이라면 이 디렉터리는 도메인·정책·구현 단위의 판단을 보존합니다.

## 문서

| 문서 | 핵심 결정 |
|---|---|
| [결제↔예약 도메인의 순환 의존을 어떻게 풀었나](./payment-reservation-dependency-boundary.md) | 새 구조를 추가하기보다 기존 Port/Adapter 경계를 재사용 |
| [예약 운영 정책과 인증 무효화 설계](./reservation-auth-operational-decisions.md) | 노쇼·취소·스케줄러·Blacklist 5개 의사결정 |

## 관리 기준

- 기술을 선택한 이유뿐 아니라 **선택하지 않은 대안과 현재 한계**를 함께 적습니다.
- 이미 공식 Backend ADR이 있는 결정은 [ADR](../adr/README.md)에서 관리하고 중복 작성하지 않습니다.
- 관련 문제의 실제 실패·복구 과정은 [Troubleshooting](../troubleshooting/README.md)으로 연결합니다.
