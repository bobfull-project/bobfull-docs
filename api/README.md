# BobFull API

BobFull의 HTTP API 구성과 권한 경계를 한눈에 확인하기 위한 포트폴리오용 요약 문서입니다.

실제 Request/Response DTO, Validation, ErrorCode 등 상세 계약은 Backend 저장소의 API 명세를 Source of Truth로 관리합니다.

- Application HTTP API: **71개**
- Actuator Endpoint: **2개**
- 전체 문서화 대상: **73개**
- 권한 구분: `PUBLIC` / `AUTHENTICATED` / `OWNER` / `ADMIN`

## API 구성

| 도메인 | API 수 | 주요 기능 |
|---|---:|---|
| 인증 | 5 | 회원가입, 로그인, 로그아웃, 토큰 재발급 |
| 회원 | 2 | 내 정보 조회·수정 |
| 식당 | 9 | 식당 CRUD, 검색, 이미지 업로드, 피드백 Insight |
| 합석 테이블 / 회차 | 12 | 테이블·회차 등록/조회/수정/삭제 |
| 예약 | 10 | 예약 가능 여부, 결제 준비, 참여 검색·취소, 사장님 예약 관리 |
| 결제 / 환불 / 정산 | 8 | 결제 완료 검증, 결제·환불 조회, 지급 예정 금액 조회 |
| 노쇼 | 5 | 노쇼 대상 조회, 처리·해제, 이력 조회 |
| 채팅 / 신고 | 3 | 채팅방·메시지 조회, 참여자 신고 |
| 관리자 / Moderation | 16 | 운영 조회, 통계, AI Moderation·신고 관리 |
| 운영 / Webhook | 3 | Health, Prometheus, PortOne Webhook |

## 주요 API

### 인증

```text
POST /api/auth/signup/users
POST /api/auth/signup/owners
POST /api/auth/login
POST /api/auth/logout
POST /api/auth/reissue
```

### 식당 / 회차

```text
GET  /api/restaurants
GET  /api/restaurants/{restaurantId}
GET  /api/restaurants/{restaurantId}/dining-sessions
POST /api/owner/restaurants
POST /api/owner/restaurants/{restaurantId}/tables
POST /api/owner/tables/{tableId}/dining-sessions
```

### 예약

```text
GET  /api/reservations/availability
POST /api/reservations/prepare
GET  /api/reservations/search
GET  /api/members/me/reservations
POST /api/reservations/{reservationId}/participations/me/cancel
POST /api/owner/reservations/{reservationId}/cancel
```

### 결제 / 환불 / 정산

```text
POST /api/payments/{paymentId}/complete
GET  /api/members/me/payments
GET  /api/payments/{paymentId}
GET  /api/members/me/refunds
GET  /api/refunds/{refundId}
GET  /api/owner/restaurants/{restaurantId}/settlements/expected
GET  /api/owner/restaurants/{restaurantId}/settlements/reservations
GET  /api/owner/settlements/reservations/{reservationId}
```

### 채팅 / 신고

```text
GET  /api/reservations/{reservationId}/chat-room
GET  /api/chat/rooms/{chatRoomId}/messages
POST /api/chat-rooms/{chatRoomId}/members/{reportedMemberId}/reports
```

> 실시간 채팅 송수신은 HTTP API가 아니라 WebSocket/STOMP `/ws` 경계에서 처리합니다.

### 관리자 / AI Moderation

```text
GET   /api/admin/members
GET   /api/admin/restaurants
GET   /api/admin/reservations
GET   /api/admin/payments
GET   /api/admin/refunds
GET   /api/admin/moderation/members
GET   /api/admin/moderation/reports
PATCH /api/admin/moderation/reports/{reportId}/review
```

### 운영 / 외부 연동

```text
GET  /actuator/health
GET  /actuator/prometheus
POST /api/webhooks/portone
```

## 상세 API 명세

전체 Endpoint와 Request/Response/Error 계약은 Backend 저장소에서 관리합니다.

- [전체 API 통합 요약](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/BOBFULL_API_SPEC_COMPLETE.md)
- [API 상세 명세 목차](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/api/API.md)

도메인별 상세 명세:

- [Auth](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/api/auth-api.md)
- [Member](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/api/member-api.md)
- [Restaurant](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/api/restaurant-api.md)
- [Table / Session](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/api/table-session-api.md)
- [Reservation](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/api/reservation-api.md)
- [Payment / Refund / Settlement](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/api/payment-api.md)
- [No-show](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/api/no-show-api.md)
- [Chat](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/api/chat-api.md)
- [Admin / Moderation](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/api/admin-api.md)
- [Operations / Webhook](https://github.com/bobfull-project/bobfull-backend/blob/develop/docs/api/operations-api.md)
