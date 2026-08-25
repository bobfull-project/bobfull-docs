# [실시간] 다중 App 채팅과 Redis Pub/Sub

## 1. 단일 App에서는 문제가 없었던 구조

초기 채팅은 WebSocket/STOMP와 Spring Simple Broker를 이용해 하나의 App 인스턴스 안에서 처리했습니다.

```text
사용자 A
→ App EC2
→ STOMP Broker
→ 사용자 B
```

단일 App 환경에서는 같은 JVM 안에 연결된 세션을 알고 있기 때문에 이 구조로 충분했습니다.

## 2. App이 두 대가 되자 생긴 문제

ALB 뒤 App EC2를 두 대로 늘리면 WebSocket 연결도 서로 다른 인스턴스로 나뉠 수 있습니다.

```text
사용자 A → App #1
사용자 B → App #2
```

App #1의 로컬 STOMP Broker는 App #2의 WebSocket 세션을 알 수 없습니다.

따라서 App #1에서 메시지를 받아 로컬 세션에만 전송하면 같은 채팅방의 사용자가 App #2에 연결된 경우 메시지를 받지 못할 수 있습니다.

## 3. 선택 — Redis Pub/Sub으로 인스턴스 간 전파

최종 구조는 다음과 같습니다.

```text
사용자 A
→ App #1
→ DB ChatMessage 저장
→ Redis Pub/Sub publish
          ↓
      App #1 / App #2 subscribe
          ↓
각 App의 local STOMP 세션으로 전달
          ↓
사용자 A / 사용자 B
```

핵심은 Redis를 채팅의 영속 저장소로 사용하지 않았다는 점입니다.

- DB: ChatMessage의 기준 저장소
- Redis Pub/Sub: 현재 연결된 여러 App 인스턴스에 실시간 전파
- STOMP: 각 App 내부 WebSocket 세션 전달

## 4. Redis Pub/Sub과 Kafka를 분리해서 사용한 이유

프로젝트에는 Redis Pub/Sub과 Kafka가 모두 존재하지만 서로 대체 관계로 사용하지 않았습니다.

| 기술 | 사용 목적 |
|---|---|
| Redis Pub/Sub | 현재 접속한 사용자에게 빠르게 실시간 전파 |
| Kafka | AI 검수·피드백 분석 같은 후속 작업의 독립 소비, Retry/DLT, 재처리 경계 |

채팅 전달 자체를 Kafka에 의존시키지 않았고, AI 처리 때문에 사용자 채팅 저장·전달이 늦어지지 않도록 경계를 분리했습니다.

## 5. Pub/Sub이 놓친 메시지는 DB에서 복구

Redis Pub/Sub은 구독자가 잠시 끊겨 있으면 그동안 발행된 메시지를 나중에 다시 전달해 주는 영속 큐가 아닙니다.

그래서 채팅 메시지는 먼저 DB에 저장하고, 재접속 시 cursor 기반 조회로 누락 메시지를 다시 읽을 수 있게 했습니다.

```text
실시간
DB 저장 → Redis Pub/Sub → STOMP

재접속 / 누락 복구
DB cursor 조회
```

이 구조로 실시간 전파와 영속 복구의 책임을 분리했습니다.

## 6. 검증 범위

다중 App 환경에서 다음을 확인했습니다.

- App A → App B 사용자 메시지 전달
- App B → App A 사용자 메시지 전달
- ChatMessage 단일 저장
- 방별 local STOMP 격리
- DB 기반 누락 메시지 복구 경계

## 관련 문서

- [ADR 0011 - Chat Redis Pub/Sub](../adr/0011-chat-redis-pubsub.md)
- [System Architecture](../architecture/system-architecture.md)
- [[인프라] 단일 EC2 메모리 장애와 자원 분리](./resource-separation.md)
