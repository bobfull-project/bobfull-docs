# ADR 0018: Kafka Broker를 MSK 대신 전용 EC2로 운영

> BobFull Backend의 전체 ADR 19개 중 포트폴리오에서 보여줄 대표 의사결정을 요약한 문서입니다. 상세 원본은 [Backend ADR](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/adr/0018-kafka-dedicated-ec2-over-msk.md)을 기준으로 합니다.

## 문제

다중 App EC2 구조에서는 App마다 로컬 Kafka Broker를 둘 수 없으므로 공용 Broker가 필요합니다. 동시에 현재 Kafka 사용 범위에 비해 관리형 MSK 비용과 운영 편의성을 감수할 가치가 있는지 판단해야 했습니다.

## 고려한 대안

- App EC2 내부 Kafka 유지
- Amazon MSK Provisioned
- Amazon MSK Serverless
- Kafka 전용 EC2

## 결정

현재 프로젝트에서는 **MSK를 도입하지 않고 Kafka 전용 EC2의 단일 KRaft Broker를 운영**합니다.

```text
Blue / Green App EC2
        ↓
 Kafka 전용 EC2
 single KRaft broker
```

## 왜 이 선택을 했나

App과 Broker의 메모리·배포 생명주기를 분리하면서, 현재 제한된 Kafka 사용 범위에서는 관리형 Kafka의 상시 비용보다 직접 운영을 감수하는 편이 적합하다고 판단했습니다.

이 결정은 Kafka 자체의 채택 이유를 다루지 않습니다. Outbox + Kafka를 사용하는 애플리케이션 경계는 ADR 0010이 담당하고, 이 문서는 Broker를 **어디에서 운영할지**에 대한 인프라 결정입니다.

## 트레이드오프

- Kafka 설치·설정·패치·장애 복구를 직접 운영해야 함
- 현재 단일 EC2·단일 Broker라 Kafka HA를 보장하지 않음
- EC2 STOP 중에는 Kafka 기반 Consumer 처리가 진행되지 않음
- MSK 대비 성능 우위를 주장하지 않음

Kafka 의존 기능과 트래픽이 증가하거나 Broker 장애에 대한 가용성 요구가 커질 때 MSK 또는 다중 Broker 구성을 다시 비교합니다.
