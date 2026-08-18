# ADR 0008: ChatRoom 생성 의도의 Transactional Outbox

> BobFull Backend의 전체 ADR 19개 중 포트폴리오에서 보여줄 대표 의사결정을 요약한 문서입니다. 상세 원본은 [Backend ADR](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/adr/0008-chat-room-transactional-outbox.md)을 기준으로 합니다.

## 문제

최초 예약 결제 확정 뒤 ChatRoom 생성은 핵심 결제·예약 트랜잭션과 분리되어야 합니다. 단순 `AFTER_COMMIT` 메모리 리스너만 사용하면 DB 커밋 직후 프로세스가 종료될 때 재시작 후 복구할 영속 근거가 없습니다.

## 결정

예약 확정 트랜잭션에서 `OutboxEvent(CHAT_ROOM_CREATION_REQUESTED, PENDING)`를 핵심 데이터와 함께 저장합니다.

- 커밋 후 즉시 signal은 빠른 처리 경로
- scheduler가 due `PENDING`과 stale `PROCESSING`을 복구
- Processor는 짧은 트랜잭션에서 claim만 수행
- ChatRoom 생성은 잠금 밖 별도 트랜잭션에서 `createIfAbsent(reservationId)`로 실행
- 5·10·20·40·80초 backoff 후 최종 실패는 `FAILED`
- `chat_room.reservation_id` UNIQUE와 `createIfAbsent`로 at-least-once 중복 부작용 방지

같은 Outbox 기반을 이메일 후속 처리에도 재사용하되, 이벤트 타입별 Processor 책임은 분리합니다.

## 왜 이 선택을 했나

핵심 Payment·Reservation 상태와 후속 작업 의도를 원자적으로 남기면서도 ChatRoom 저장 실패가 결제 트랜잭션을 되돌리지 않게 하기 위해서입니다.

## 트레이드오프와 검증

Scheduler polling과 재시도 상태 관리가 추가됩니다. 대신 프로세스 재시작, 단일 claim, stale 회수, 재시도 후 FAILED, 중복 생성 방지를 자동 테스트와 Evidence로 검증했습니다.
