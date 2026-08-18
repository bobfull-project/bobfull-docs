# ADR 0011: 다중 인스턴스 채팅 실시간 전파에 Redis Pub/Sub 사용

> BobFull Backend의 전체 ADR 19개 중 포트폴리오에서 보여줄 대표 의사결정을 요약한 문서입니다. 상세 원본은 [Backend ADR](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/adr/0011-chat-redis-pubsub.md)을 기준으로 합니다.

## 문제

Spring STOMP Simple Broker는 현재 App 인스턴스의 WebSocket 세션만 알고 있습니다. ALB 뒤 다중 EC2에서 같은 채팅방 사용자가 서로 다른 인스턴스에 붙으면 로컬 STOMP 발행만으로는 상대 인스턴스 세션에 메시지를 전달할 수 없습니다.

## 결정

DB에 `ChatMessage`를 저장한 뒤 커밋 후 Redis Pub/Sub 채널에 한 번 발행하고, 각 App 인스턴스의 subscriber가 자신의 로컬 STOMP 세션으로 fan-out합니다.

- Controller의 직접 STOMP 발행 제거
- Redis Pub/Sub은 다중 App 간 실시간 fan-out만 담당
- DB가 메시지의 최종 저장소
- Redis 발행·구독 실패는 저장된 메시지를 롤백하지 않음
- 누락 메시지는 DB cursor 조회로 복구

## 왜 이 선택을 했나

Sticky Session만으로는 서로 다른 사용자가 다른 App 인스턴스에 연결되는 문제를 해결할 수 없습니다. Redis Streams·Kafka처럼 재생 가능한 메시징은 현재 실시간 fan-out 요구에 비해 복잡도가 높았습니다.

## 트레이드오프와 검증

Redis Pub/Sub은 best-effort이므로 연결 단절 중 메시지 재생을 보장하지 않습니다. 이 한계를 DB cursor 복구 경로로 보완합니다.

실제 다중 App EC2 + 공용 ElastiCache 환경에서 서로 다른 EC2 사이 publish/subscriber 전달과 양방향 실시간 채팅을 확인했습니다.
